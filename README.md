# Chainlink node single VM docker deployment
## Preconditions
- Cloud infrastructure setup
- PostgreSQL (database setup)
- Applying security layers to your instance (2FA, SSH)
- Blockchain connection (via a third-party service provider or running an own full node)
## Install docker

Install docker and create a user with the permission to create containers:

```bash
curl -sSL https://get.docker.com/ | sh
sudo systemctl start docker
sudo usermod -aG docker $USER
exit
# log in again
```
## Create directory
The directory needs to be created as a hidden one in order to follow security best practices:

```bash
mkdir .chainlink-kovan
cd .chainlink-kovan
```

## Create Environmental file (for Kovan testnet)
List of all variables: https://docs.chain.link/docs/configuration-variables
```bash
echo "ROOT=/chainlink
LOG_LEVEL=debug
ETH_CHAIN_ID=42
MIN_OUTGOING_CONFIRMATIONS=1
MIN_INCOMING_CONFIRMATIONS=1
MINIMUM_CONTRACT_PAYMENT=100000000000000000
LINK_CONTRACT_ADDRESS=0xa36085F69e2889c224210F603D836748e7dC0088
GAS_UPDATER_ENABLED=true
ALLOW_ORIGINS=*" > ~/.chainlink-kovan/.env
````
- `MIN_OUTGOING_CONFIRMATIONS` and `MIN_INCOMING_CONFIRMATIONS` are set to `1` to perform faster on testnets. If you're node's jobs trigger transactions of real value or you're node sends multiple requets you can adjust that value for higher security.
- `MINIMUM_CONTRACT_PAYMENT` is set to `100000000000000000` (0.1 LINK), for on-chain verification on https://market.link it should be set to `1000000000000000` (0.001) or lower.
- `LINK_CONTRACT_ADDRESS` is the chainlink token adress of the Kovan network. Other chains and networks: https://docs.chain.link/docs/link-token-contracts
- `LOG_LEVEL` is `debug` to display every action and synced block. You can change this parameter to "info" in order to use less storage capacity
## Chainlink ETH failover
First you need to create a network, which is nessacery to connect your container to the Chainlink node:
https://docs.docker.com/engine/tutorials/networkingcontainers/
```bash 
docker network create kovan
```
Load the image into your instance:
```bash
docker pull fiews/cl-eth-failover
```
Run command for the ETH proxy container:

https://medium.com/fiews/chainlink-eth-node-failover-proxy-7d76cdea49f3
```bash
docker run --name eth-failover --restart unless-stopped --network kovan fiews/cl-eth-failover wss://cl-ropsten.fiews.io/v1/myApiKey ws://localhost:8546/
```
You need to change the websocket adresses of your Ethereum connection (nodes of a NaaS Provider or an own ETH full node):
```bash
echo "ETH_URL=ws://eth-failover:4000/" >> ~/.chainlink-kovan/.env
```
## Setting up a remote database connection
```bash
cd ~/.chainlink-kovan
```
```bash
echo "DATABASE_URL=postgresql://$USERNAME:$PASSWORD@$SERVER:$PORT/$DATABASE
DATABASE_TIMEOUT=0" >> ~/.chainlink-kovan/.env
```
You need to change the credentials of your Postgres access. Have a look at the official Chainlink documentation: 
https://docs.chain.link/docs/connecting-to-a-remote-database
## TLS certificate for https scheme
Create a hidden directory for your certificates:
```bash
cd ~/.chainlink-kovan
mkdir .tls
```
Create the TLS certificate with crt and key:
```bash
cd ~/.chainlink-kovan/.tls && openssl req -x509 -out  ~/.chainlink-kovan/.tls/server.crt  -keyout ~/.chainlink-kovan/.tls/server.key -newkey rsa:2048 -nodes -sha256 -days 365 -subj '/CN=localhost' -extensions EXT -config <( printf "[dn]\nCN=localhost\n[req]\ndistinguished_name = dn\n[EXT]\nsubjectAltName=DNS:localhost\nkeyUsage=digitalSignature\nextendedKeyUsage=serverAuth")
```

Add the certificate to your environmental file:
```bash
echo "CHAINLINK_TLS_PORT=6689
TLS_CERT_PATH=/chainlink/.tls/server.crt
TLS_KEY_PATH=/chainlink/.tls/server.key" >> ~/.chainlink-kovan/.env
```
```bash
sed -i '/SECURE_COOKIES=false/d' ~/.chainlink-kovan/.env
```
## Set credentials for the API/web GUI and node wallets
```bash
mkdir ~/.chainlink-kovan/.psw
cd ~/.chainlink-kovan/.psw
```
```bash
echo "<user@example.com>" > .api
echo "<password>" >> .api
echo "<my_wallet_password>" > .password
```
## Initialize node and backup
First of all you need to run your Chainlink node without a deamon flag and to configure your API login and password

First initialisation:
```bash
cd ~/.chainlink-kovan && sudo docker run --name kovan-main --network kovan -p 6689:6689 -v ~/.chainlink-kovan:/chainlink -it --env-file=.env smartcontract/chainlink:<latest_image> local n
```
You need to set `<latest_image>` to the image you want to use, e.g. `0.10.3` Have a look there for the current images of the Chainlink smartcontractkit: https://hub.docker.com/r/smartcontract/chainlink/tags

After entering your password and your API credentials you can cancel and remove the container (kills it automatically)
```bash
STRG + C
```
```bash
docker rm kovan-main
```
Now you can execute the entire command and initialise the node in deamon mode to ensure a permanent uptime

Command main:
```bash
cd ~/.chainlink-kovan && sudo docker run --name kovan-main --network kovan --restart unless-stopped -d -p 6689:6689 -v ~/.chainlink-kovan:/chainlink -it --env-file=.env smartcontract/chainlink:<latest_image> local n -p /chainlink/.psw/.password -a /chainlink/.psw/.api 
```
 
Command backup:
 ```bash
cd ~/.chainlink-kovan && sudo docker run --name kovan-backup --network kovan --restart unless-stopped -d -p 6689:6689 -v ~/.chainlink-kovan:/chainlink -it --env-file=.env smartcontract/chainlink:<latest_image> local n -p /chainlink/.psw/.password -a /chainlink/.psw/.api 
 ```
 
 - `-d` flag = start the container in detached mode
 - `-p` flag = maps your containers port to the host machine
 - `-v` flag = mounts the current working directory into the container
 - `-a` flag = attach inside of the container
 - `--restart unless-stopped` = Restart policy to apply when a container exits
 - `--name` = give the container a name
 
 To ensure updates and configuration changes can be done without downtime you need to kill and restart the main and backup node with full lock on the database: https://docs.chain.link/docs/performing-system-maintenance
 ## security flags ##
 Here is a list of other security flags to ensure best protection:
 1) Run images with "no new priviledges" to prevent privilege escalation
 
 `--security-opt=no-new-privileges`
 
2) Restrict the container to read-only privilege

 `--read-only`
 
3) Limit
 
 `--pids-limit 100`
 
 Attackers could launch a fork bomb with a single command inside the container. This fork bomb could crash the entire system and would require a restart of the host to make the system functional again. Using the PIDs cgroup parameter –pids-limit would prevent this kind of attack by restricting the number of forks that can happen inside a container within a specified time frame.

4) CPU & memory capacity 
 
 `--cpus=1.5`

 `--memory=5g`
 
 ## Important commands ##
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
 current log files of a container
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
