# ETH-Full-Node-Deployment
Short guide to deploy an ETH 2.0 full node with the execution client and consensus clients on a single m6i.xlarge EC2 instance in AWS.

### Resources
* https://docs.teku.consensys.net/en/latest/HowTo/Get-Started/Installation-Options/Run-Docker-Image/
* https://someresat.medium.com/guide-to-staking-on-ethereum-ubuntu-g%C3%B6erli-teku-6512b26f1372
* https://hackmd.io/bF0kygj4S92fmuZNRQXuSw?view

### Enable Firewall Rules
```
Inbound
#SSH 10.0.0.0/16
- 6001 

# EC P2P 0.0.0.0
- 30303 tcp/udp

# EC JSON-RPC Inbound from Chainlink security group and possibly Splunk security group
- 8545 - http port / tcp
- 8546 - ws port / tcp

# CC P2P 0.0.0.0
- 9000 tcp/udp

# EC Engine API (comms port between EC and CC) <node sg>
- 8551 http & ws port / tcp

# Fluentd <node sg>
- 24224 tcp
```

### Create User
```
sudo su root
adduser linkwelladmin
usermod -aG sudo <yourusername>
su linkwelladmin
```

### Update instance
```
sudo yum update
```

### Modify default SSH port
```
sudo nano /etc/ssh/sshd_config
Port 6001
sudo systemctl restart ssh
```

### Modify swap space
```
#Create swap space
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
# Modify to persist after reboot
sudo cp /etc/fstab /etc/fstab.bak
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
# Configure swap space
sudo sysctl vm.swappiness=10
sudo sysctl vm.vfs_cache_pressure=50
# Modify swap file
sudo nano /etc/sysctl.conf
# Add
vm.swappiness=10
vm.vfs_cache_pressure = 50
```

### Mount volume 
```
1. sudo file -s /dev/<volume>
2. sudo mkfs -t xfs /dev/xvdf
3. sudo lsblk -f
4. sudo mount /dev/sdb /lw/data
## Persist volume
5. sudo cp /etc/fstab /etc/fstab.orig
6. sudo blkid
7. sudo vim /etc/fstab
8. UUID=5ab375d1-334f-4ed9-bfcc-a628eb3ff14c       /lw/data        xfs     defaults,nofail  0  2
```

### Besu pre-setup
```
sudo groupadd node
sudo useradd --no-create-home --shell /bin/false besu
sudo mkdir -p /lw/data/besu
sudo chown -R besu:besu /lw/data/besu
sudo gpasswd -a besu node
```

### Teku pre-setup
```
1. sudo useradd --no-create-home --shell /bin/false teku
2. sudo mkdir -p /lw/data/teku
3. sudo chown -R teku:teku /lw/data/teku
sudo gpasswd -a teku node
```

### Install docker and docker-compose
```
sudo amazon-linux-extras install -y docker
sudo systemctl start docker
sudo gpasswd -a $USER docker
exit
# log in again
sudo curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
sudo cp /usr/local/bin/docker-compose /usr/bin
sudo chmod +x /usr/bin/docker-compose
```

### Create JWT
```
sudo mkdir -p /lw/data/jwtsecret
openssl rand -hex 32 | sudo tee /lw/data/jwtsecret/jwt.hex > /dev/null
sudo chown -R linkwelladmin:node /lw/data/jwtsecret
sudo chmod -R 770 /lw/data/jwtsecret
```

### Docker-compose
```
---
version: '3.4'
x-logging:
  &default-logging
  driver: "fluentd"
x-log-opts:
  &log-opts
  fluentd-address: "localhost:24224"
  tag: "{{.Name}}-{{.ID}}"
services:
  besu_node:
    image: hyperledger/besu:latest
    container_name: besu
    user: 1003:1003
    restart: always
    command: ["--network=goerli",
              "--data-storage-format=BONSAI",
              "--data-path=/lw/data/besu",
              "--host-allowlist=*",
              "--sync-mode=X_SNAP",
              "--engine-rpc-enabled=true",
              "--engine-host-allowlist=localhost",
              "--engine-rpc-port=8551",
              "--engine-jwt-secret=/lw/data/jwtsecret/jwt.hex",
              "--rpc-http-enabled",
              "--rpc-ws-enabled",
              "--rpc-ws-port=8546",
              "--rpc-ws-host=0.0.0.0",
              "--rpc-http-port=8545",
              "--rpc-http-host=0.0.0.0"]
    volumes:
      - ./besu:/lw/data/besu
    ports:
      # Map the p2p port(30303) and RPC HTTP port(8545)
      - "8545:8545"
      - "8546:8546"
      - "8551:8551"
      - "30303:30303/tcp"
      - "30303:30303/udp"

  teku_node:
    environment:
      - "JAVA_OPTS=-Xmx4g"
      - "TEKU_OPTS=-XX:-HeapDumpOnOutOfMemoryError"
    image: consensys/teku:latest
    container_name: teku
    user: 1004:1004
    restart: always
    command: ["--network=goerli",
              "--data-path=/lw/data/teku",
              "--ee-endpoint=http://localhost:8551",
              "--initial-state=https://goerli.checkpoint-sync.ethdevops.io/eth/v2/debug/beacon/states/finalized",
              "--ee-jwt-secret-file=/lw/data/jwtsecret/jwt.hex",
              "--p2p-port=9000"]
    depends_on:
      - besu_node
    volumes:
      - ./teku:/lw/data/teku
    ports:
      # Map the p2p port(9000) and REST API port(5051)
      - "9000:9000/tcp"
      - "9000:9000/udp"
```
