# Chainlink node single VM deployment
## Preconditions
- Cloud infrastructure setup
- PostgreSQL (database setup)
- Applying security layers to your instance (2FA, SSH)
- Blockchain connection (via a third-party service provider or running an own full node)
## install docker

install docker and create a user with the permission to create containers

```bash
curl -sSL https://get.docker.com/ | sh
sudo systemctl start docker
sudo usermod -aG docker $USER
exit
# log in again
```
## create directory
The directory needs to be created as a hidden one in order to follow security best practices

```bash
mkdir .chainlink-kovan
cd .chainlink-kovan
```

## create Environmental file (for Kovan testnet)
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
ALLOW_ORIGINS=*" > ~/.chainlink-kovan/.env
````
- `MIN_OUTGOING_CONFIRMATIONS` and `MIN_INCOMING_CONFIRMATIONS` are set to `0` to perform faster on testnets. If you're node's jobs trigger transactions of real value you can adjust that value for higher security.
- `MINIMUM_CONTRACT_PAYMENT` is set to `100000000000000000` (0.1 LINK), for on-chain verification on https://market.link it should be set to `1000000000000000` (0.001) or lower.
- `LINK_CONTRACT_ADDRESS` is the chainlink token adress of the Kovan network. Other chains and networks: https://docs.chain.link/docs/link-token-contracts
- `LOG_LEVEL` is `debug` to display every action and synced block. You can change this parameter to "info" in order to use less storage capacity
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

https://medium.com/fiews/chainlink-eth-node-failover-proxy-7d76cdea49f3
```bash
docker run --name eth-failover --restart unless-stopped --network kovan fiews/cl-eth-failover wss://cl-ropsten.fiews.io/v1/myApiKey ws://localhost:8546/
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
You need to change the credentials of your Postgres-access. Take a look here for the official chainlink documentation: https://docs.chain.link/docs/connecting-to-a-remote-database
## TLS certificate for https scheme
First of all you need to create a hidden directory for your certificates.
```bash
cd ~/.chainlink-kovan
mkdir .tls
```
Create the TLS certificate with crt and key.
```bash
cd ~/.chainlink-kovan/.tls && openssl req -x509 -out  ~/.chainlink-kovan/.tls/server.crt  -keyout ~/.chainlink-kovan/.tls/server.key -newkey rsa:2048 -nodes -sha256 -days 365 -subj '/CN=localhost' -extensions EXT -config <( printf "[dn]\nCN=localhost\n[req]\ndistinguished_name = dn\n[EXT]\nsubjectAltName=DNS:localhost\nkeyUsage=digitalSignature\nextendedKeyUsage=serverAuth")
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
## set secret & api for GUI login and wallet
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
First of all you have to run your chainlinknode without a deamonflag and also you need to configure directly your api login and password.

First initialisation:
```bash
cd ~/.chainlink-kovan && sudo docker run --name kovan-main --network kovan -p 6689:6689 -v ~/.chainlink-kovan:/chainlink -it --env-file=.env smartcontract/chainlink:<latest_image> local n
```
You need to define `<latest_image>` with the image you want to use, e.g. `0.10.3` Take a look here for the current images of the chainlink smartcontractkit: https://hub.docker.com/r/smartcontract/chainlink/tags

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
 
 - `-d` flag = start the container in detached mode
 - `-p` flag = maps your containers port to the host machine
 - `-v` flag = mounts the current working directory into the container
 - `-a` flag = attach inside of the container
 - `--restart unless-stopped` = Restart policy to apply when a container exits
 - `--name` = give the container a name
 
 To ensure updates and configuration without downtime you need to kill and restart the main and the backup-node during full lock on the database: https://docs.chain.link/docs/performing-system-maintenance
 ## security flags ##
 Here is a list of other security flags to ensure full protection:
 1) Run images with "no new priviledges" to prevent privilege escalation
 
 `--security-opt=no-new-privileges`
 
2) Restrict the container to read-only priviledge

 `--read-only`
 
3) Limit
 
 `--pids-limit 100`
 
 Attackers could launch a fork bomb with a single command inside the container. This fork bomb could crash the entire system and would require a restart of the host to make the system functional again. Using the PIDs cgroup parameter –pids-limit would prevent this kind of attack by restricting the number of forks that can happen inside a container within a specified time frame.

4) CPU & Memory capacity 
 
 `--cpus=1.5`

 `--memory=5g`
 
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
