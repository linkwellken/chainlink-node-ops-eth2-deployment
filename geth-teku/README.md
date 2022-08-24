# ETH 2.0 Full Node deployment using docker-compose for teku and geth
Short guide to deploy an ETH 2.0 full node with the geth execution client and teku consensus client on a single t3.xlarge EC2 instance in AWS using docker-compose. This guide is primarily for deploying full ETH 2.0 nodes for the use of Chainlink nodes, and does not include the validator client for staking.  You can optionally deploy fluentd to monitor your docker containers and send the logs to one or multiple logging destinations. Refer to [this guide on fluentd](https://github.com/linkwellken/fluentd-splunk-cw-logging) if you choose to do so.

### Resources
* https://docs.teku.consensys.net/en/latest/HowTo/Get-Started/Installation-Options/Run-Docker-Image/
* https://someresat.medium.com/guide-to-staking-on-ethereum-ubuntu-g%C3%B6erli-teku-6512b26f1372
* https://hackmd.io/bF0kygj4S92fmuZNRQXuSw?view

### Enable AWS EC2 Security Group Rules
```
Inbound
#SSH (Optional)
- 6001 from <your ip>

# EL P2P 
- 30303 tcp/udp from 0.0.0.0

# EL JSON-RPC 
- 8545 - http port / tcp from Chainlink security group and optionally Splunk security group
- 8546 - ws port / tcp from Chainlink security group and optionally Splunk security group

# EL Engine API (comms port between EC and CC) 
- 8551 http & ws port / tcp from local security group

# CL P2P 
- 9000 tcp/udp from 0.0.0.0

# Fluentd 
- 24224 tcp from local security group
```

### Create User
```
sudo su root
adduser <user>
visudo
<add your user and permissions to file>
su <user>
```

### Update instance
```
sudo yum update
```

### Modify default SSH port (Optional)
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
8. UUID=<volume id>       /lw/data        xfs     defaults,nofail  0  2
```

### geth pre-setup
```
sudo groupadd node
sudo useradd --no-create-home --shell /bin/false geth
sudo mkdir -p /lw/data/geth
sudo chown -R geth:geth /lw/data/geth
sudo gpasswd -a geth node
```

### Teku pre-setup
```
sudo useradd --no-create-home --shell /bin/false teku
sudo mkdir -p /lw/data/teku
sudo chown -R teku:teku /lw/data/teku
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
cd /lw/data
openssl rand -hex 32 > jwtsecret
sudo chown -R linkwelladmin:node jwtsecret
sudo chmod -R 440 jwtsecret
```

### Docker-compose
```
git clone https://github.com/linkwellken/ETH-Full-Node-Deployment.git
cd ETH-Full-Node-Deployment/geth-teku
cp docker-compose.yaml ../..
docker-compose up -d
```
