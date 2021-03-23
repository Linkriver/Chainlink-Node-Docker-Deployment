# Linkriver-Chainlinknode-VM
## install Docker

install docker and create a User with rules for creation of containers

```bash
curl -sSL https://get.docker.com/ | sh
sudo systemctl start docker
sudo usermod -aG docker $USER
exit
# log in again
```
## create Directory
The Directory needs to be created as a hidden for for BestSercurity Practices

```bash
mkdir .chainlink
cd .chainlink
```

## create Environmental Kovan

```bash
echo "ROOT=/chainlink
LOG_LEVEL=debug
ETH_CHAIN_ID=42
MIN_OUTGOING_CONFIRMATIONS=0
MIN_INCOMING_CONFIRMATIONS=0
LINK_CONTRACT_ADDRESS=0xa36085F69e2889c224210F603D836748e7dC0088
GAS_UPDATER_ENABLED=true
ALLOW_ORIGINS=*" > ~/.chainlink/.env
````
## ChainlinkEthFailover

```bash
docker pull fiews/cl-eth-failover
```
```bash
docker run fiews/cl-eth-failover wss://cl-ropsten.fiews.io/v1/myApiKey ws://localhost:8546/
```

## Setup remote Database connection

```bash
echo "DATABASE_URL=postgresql://$USERNAME:$PASSWORD@$SERVER:$PORT/$DATABASE
DATABASE_TIMEOUT=0" >> ~/.chainlink/.env
```
## TLS certificate for https scheme

## initialize node and backup
