# Linkriver-Chainlinknode-VM
## requirements/ preconditions
- Cloudsetup
- PostgreSQL
- Security Layers of your instance (2FA ssh ...)
- Blockchain connection via third-party-provider or an own Fullnode
## install Docker

install docker and create a User with rules for creation of containers.

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
mkdir .chainlink-kovan
cd .chainlink-kovan
```

## create Environmental Kovan
list of all variables: https://docs.chain.link/docs/configuration-variables
```bash
echo "ROOT=/chainlink
LOG_LEVEL=debug
ETH_CHAIN_ID=42
MIN_OUTGOING_CONFIRMATIONS=0
MIN_INCOMING_CONFIRMATIONS=0
MINIMUM_CONTRACT_PAYMENT=100000000000000000
LINK_CONTRACT_ADDRESS=0xa36085F69e2889c224210F603D836748e7dC0088
GAS_UPDATER_ENABLED=true
ALLOW_ORIGINS=*" > ~/.chainlink/.env
````
MIN_OUTGOING_CONFIRMATIONS and MIN_INCOMING_CONFIRMATIONS ist set to "0" to performe faster with your node an start a jobrun instantly when it appears on the blockchain.

MINIMUM_CONTRACT_PAYMENT is set to 0.1 LINK, which is the usual payment.

LINK_CONTRACT_ADDRESS is the chainlink token adress for the kovan network. Other chains and networks: https://docs.chain.link/docs/link-token-contracts

LOG_LEVEL is debug to visualize every steps and synced blocks. You can change this parameter by time to "info" , that your node uses less storage
## ChainlinkEthFailover
First you need to create a network, which is nessacery to combine your container with the Chainlinknode
https://docs.docker.com/engine/tutorials/networkingcontainers/
```bash 
docker network create kovan
```
Load the image to your instance:
```bash
docker pull fiews/cl-eth-failover
```
Run command of the eth proxy container:
```bash
docker run --name eth-failover fiews/cl-eth-failover wss://cl-ropsten.fiews.io/v1/myApiKey ws://localhost:8546/
```
You need to change the websocket-adresses to your Ethereum connection (nodes of a NaaS Provider or an own Eth Fullnode)
```bash
echo "ETH_URL=ws://eth-failover:4000/" >> ~/.chainlink-kovan/.env
```
## Setup remote Database connection
```bash
cd ~/.chainlink-kovan
```
```bash
echo "DATABASE_URL=postgresql://$USERNAME:$PASSWORD@$SERVER:$PORT/$DATABASE
DATABASE_TIMEOUT=0" >> ~/.chainlink-kovan/.env
```
## TLS certificate for https scheme
First of all you need to create a hidden directory for your certificates.
```bash
cd ~/.chainlink-kovan
```
```bash
mkdir .tls
cd .tls
```
Create the TLS certificate with crt and key.
```bash
openssl req -x509 -out  ~/.chainlink-kovan/.tls/server.crt  -keyout ~/.chainlink-kovan/.tls/server.key -newkey rsa:2048 -nodes -sha256 -days 365 -subj '/CN=localhost' -extensions EXT -config <( printf "[dn]\nCN=localhost\n[req]\ndistinguished_name = dn\n[EXT]\nsubjectAltName=DNS:localhost\nkeyUsage=digitalSignature\nextendedKeyUsage=serverAuth")
```

Implement the certificate inside of your environmental:
```bash
echo "CHAINLINK_TLS_PORT=6689
TLS_CERT_PATH=/chainlink/.tls/server.crt
TLS_KEY_PATH=/chainlink/.tls/server.key" >> ~/.chainlink-kovan/.env
```
```bash
sed -i '/SECURE_COOKIES=false/d' ~/.chainlink-kovan/.env
```
## set secret & api for GUI login and Wallet
```bash
mkdir ~/.chainlink-kovan/.psw
cd ~/.chainlink-kovan/.psw
```
```bash
echo "<user@example.com>" > .api
echo "<password>" >> .api
echo "<my_wallet_password>" > .password
```
## initialize node and backup
First of all you have to run your chainlinknode without a deamonflag and also you need to configure directly your api login and password. You need to define the latest image version on your command. You find the actual version here: https://hub.docker.com/r/smartcontract/chainlink/tags?page=1&ordering=last_updated

First initialisation:
```bash
cd ~/.chainlink-kovan && sudo docker run --name kovan-main --network kovan -p 6689:6689 -v ~/.chainlink-kovan:/chainlink -it --env-file=.env smartcontract/chainlink:<latest_image> local n
```
After entering your password and your API credentials you can cancel the container (kills it automatically) and remove the container 
```bash
STRG + C
```
```bash
docker rm kovan-main
```
Now you can run your full node command and initialise the it in deamonmode to ensure a permanent uptime.

main command:
```bash
cd ~/.chainlink-kovan && sudo docker run --name kovan-main --network kovan --restart unless-stopped -d -p 6689:6689 -v ~/.chainlink-kovan:/chainlink -it --env-file=.env smartcontract/chainlink:<latest_image> local n -p /chainlink/.psw/.password -a /chainlink/.psw/.api 
```
 
backup node:
 ```bash
cd ~/.chainlink-kovan && sudo docker run --name kovan-backup --network kovan --restart unless-stopped -d -p 6689:6689 -v ~/.chainlink-kovan:/chainlink -it --env-file=.env smartcontract/chainlink:<latest_image> local n -p /chainlink/.psw/.password -a /chainlink/.psw/.api 
 ```
 - d flag = start the container in detached mode
 - p flag = maps your containers port to the host machine
 - v flag = mounts the current working directory into the container
 - a flag = attach inside of the container
 - restart unless-stopped = Restart policy to apply when a container exits
 - name = give the container a name
 ## security flags ##
 Here is a list of other security flags to ensure full protection:
 1) Run images with "no new priviledges" to prevent privilege escalation
 ```bash
 --security-opt=no-new-privileges
 ```
2)
 ```bash
 --read-only
 ```
3)
 ```bash
 --pids-limit 100
 ```
4)
 ```bash
 --cpus=1.5
 ```
 ```bash
 --memory=5g
 ```
 ## important commands ##
 list all containers
 ```bash
 docker ps
 ```
 ```bash
 docker ps -a
 ```
 last 10 log files of a container
 ```bash
 docker logs -n 10 <container_name>
 ```
 actual log files of a container
 ```bash
 docker logs -f <container_name>
 ```
 kill a running container
 ```bash
 docker kill <container_name>
 ```
 remove a running container
 ```bash
 docker rm <container_name>
 ```
 connect to the operator GUI on your browser
 ```bash
 https://localhost:6689
 ```
 connect to the CLI
 ```bash
 docker exec -it <kovan-main> /bin/bash
 ```
 ```bash
 chainlink admin login
 ```
