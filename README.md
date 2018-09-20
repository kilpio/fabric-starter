# Starter Application for Hyperledger Fabric

Create a network to jump start development of your decentralized application.

The network can be deployed to multiple docker containers on one host for development or to multiple hosts for testing 
or production.

Scripts of this starter generate crypto material and config files, start the network and deploy your chaincodes. 
Developers can use admin web app of 
[REST API server](https://github.com/maxxx1313/fabric-rest/tree/master/server/www-admin) 
to invoke and query chaincodes, explore blocks and transactions.

What's left is to develop your chaincodes and place them into the [chaincode](./chaincode) folder, 
and user interface as a single page web app that you can serve by by placing the sources into the [www](./www) folder. 
You can take web app code or follow patterns of the 
[admin app](https://github.com/maxxx1313/fabric-rest/tree/master/server/www-admin) to enroll users, 
invoke chaincodes and subscribe to events.

Most of the plumbing work is taken care of by this starter.

# Install

Install prerequisites: docker. This example is for Ubuntu 18:
```bash
sudo apt update
sudo apt install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable"
sudo apt update
sudo apt install docker-ce docker-compose
# add yourself to the docker group and re-login
sudo usermod -aG docker ${USER}
```

# Create the network

## Orderer Organization

Define your project's DOMAIN: all of your component names will have this as a top level domain:
```bash
export DOMAIN=example.com
```

Run script on your host machine to generate crypto material and genesis block for the Orderer organization:
```bash
./generate-orderer.sh
```

Start the orderer
```bash
docker-compose -f docker-compose/docker-compose-orderer.yaml up
```

## Peer Organization 1

Open a separate console to allow for different environment variables and log output.

Define your project's DOMAIN and ORG, all other values will remain at defaults. COMPOSE_PROJECT_NAME needs to be redefined
since we may be reusing service names.
```bash
export DOMAIN=example.com ORG=org1 COMPOSE_PROJECT_NAME=org1
```

Generate crypto material for peer organization org1:
```bash
./generate-peer.sh
```

Define list of all organizations your org will be transacting with:
```bash
export ORGS='{"org1":"peer0.org1.example.com:7051","org2":"peer0.org2.example.com:7051"}'
```

Define your organization's certificate authority:
```bash
export ORGS='{"org1":"ca.org1.example.com:7054"}'
```

Start docker containers for org1
```bash
docker-compose -f docker-compose/docker-compose-peer.yaml up
```
## Peer Organization 2

Open a separate console.

Define your project's DOMAIN and ORG, and override defaults ports as these containers expose them to the same host as org1:
```bash
export DOMAIN=example.com ORG=org2 COMPOSE_PROJECT_NAME=org2 CA_PORT=8054 PEER0_PORT=8051 PEER0_EVENT_PORT=8053 PEER1_PORT=8056 PEER1_EVENT_PORT=8058 API_PORT=4001 WWW_PORT=8082
```

Generate crypto material for peer organization org2:
```bash
./generate-peer.sh
```

Inspect how environment variables fill in values for docker parameters (optional):
```bash
docker-compose -f docker-compose/docker-compose-peer.yaml config
```

Start docker containers for org2
```bash
docker-compose -f docker-compose/docker-compose-peer.yaml up
```

# Example with a network of 3 organizations

## Create organizations, channels and chaincodes

Remove all containers, delete local crypto material:
```bash
export DOMAIN=example.com
docker rm -f $(docker ps -aq)
docker volume prune -f
sudo rm -rf crypto-config
```

Generate and start the *orderer*:
```bash
./generate-orderer.sh
docker-compose -f docker-compose/docker-compose-orderer.yaml up
```

Generate and start *org1* in another console:
```bash
export ORG=org1 DOMAIN=example.com CRYPTO_CONFIG_DIR=./crypto-config ORGS='{"org1":"peer0.org1.example.com:7051","org2":"peer0.org2.example.com:7051","org3":"peer0.org3.example.com:7051"}' CAS='{"org1":"ca.org1.example.com:7054"}'
./generate-peer.sh
docker-compose -f docker-compose/docker-compose-peer.yaml up
```

Generate and start *org2* in another console. Note the ports open to host machine need to be redefined to avoid collision:
```bash
export COMPOSE_PROJECT_NAME=org2 ORG=org2 DOMAIN=example.com CRYPTO_CONFIG_DIR=./crypto-config ORGS='{"org1":"peer0.org1.example.com:7051","org2":"peer0.org2.example.com:7051","org3":"peer0.org3.example.com:7051"}' CAS='{"org2":"ca.org2.example.com:7054"}'
export CA_PORT=8054 PEER0_PORT=8051 PEER0_EVENT_PORT=8053 PEER1_PORT=8056 PEER1_EVENT_PORT=8058 API_PORT=3001 WWW_PORT=8082
./generate-peer.sh
docker-compose -f docker-compose/docker-compose-peer.yaml up
```

Generate and start *org3* in another console:
```bash
export COMPOSE_PROJECT_NAME=org3 ORG=org3 DOMAIN=example.com CRYPTO_CONFIG_DIR=./crypto-config ORGS='{"org1":"peer0.org1.example.com:7051","org2":"peer0.org2.example.com:7051","org3":"peer0.org3.example.com:7051"}' CAS='{"org3":"ca.org2.example.com:7054"}'
export CA_PORT=9054 PEER0_PORT=9051 PEER0_EVENT_PORT=9053 PEER1_PORT=9056 PEER1_EVENT_PORT=9058 API_PORT=3002 WWW_PORT=8083
./generate-peer.sh
docker-compose -f docker-compose/docker-compose-peer.yaml up
```

Now you should have 4 consoles open running with the orderer, org1, org2, org3.

Open another console where we'll become the Admin of the *Orderer* organization. We'll add orgs to the consortium:
```bash
export DOMAIN=example.com
./consortium-addorg.sh org1
./consortium-addorg.sh org2
./consortium-addorg.sh org3
``` 

Open another console where we'll become *org1* again. We'll create channel *common*, add other orgs to it, 
and join ourselves to it:
```bash
export ORG=org1 DOMAIN=example.com
./channel-create.sh common
./channel-add-org.sh org2 common
./channel-add-org.sh org3 common
./channel-join.sh org1 common
``` 

Let's create a bilateral channel between *org1* and *org2*:
```bash
./channel-create.sh org1-org2
./channel-add-org.sh org2 org1-org2
./channel-join.sh org1 org1-org2
```

Install and instantiate chaincode *reference* on channel *common*:
```bash
./chaincode-install.sh reference 1.0 /opt/chaincode/node/reference node
./chaincode-instantiate.sh common reference "{\"Args\":[\"init\",\"a\",\"10\",\"b\",\"0\"]}" 1.0
```
Install and instantiate chaincode *relationship* on channel *org1-org2*:
```bash
./chaincode-install.sh relationship 1.0 /opt/chaincode/node/relationship node
./chaincode-instantiate.sh org1-org2 relationship "{\"Args\":[\"init\",\"a\",\"10\",\"b\",\"0\"]}" 1.0
```

Open another console where we'll become *org2* to join channels *common* and *org1-org2*:
```bash
export COMPOSE_PROJECT_NAME=org2 ORG=org2 DOMAIN=example.com
./channel-join.sh org1 common
./channel-join.sh org1 org1-org2
``` 

Now become *org3* to join channel *common*:
```bash
export COMPOSE_PROJECT_NAME=org3 ORG=org3 DOMAIN=example.com
./channel-join.sh org1 common
``` 

## Use REST API servers to query and invoke chaincodes

Login into *org1* as `user1` and save returned token into env variable `JWT` which we'll use to identify our user 
in subsequent requests:
```bash
JWT=`(curl -d '{"login":"user1","password":"pass"}' --header "Content-Type: application/json" http://localhost:3000/users | tr -d '"')`
```

Query channels *org1* has joined
```bash
curl -H "Authorization: Bearer $JWT" http://localhost:3000/channels
```
returns
```json
[{"channel_id":"common"},{"channel_id":"org1-org2"}]
``` 

Query channels *org1* has joined
```bash
curl -H "Authorization: Bearer $JWT" http://localhost:3000/channels
```
returns
```json
[{"channel_id":"common"},{"channel_id":"org1-org2"}]
``` 

Query latest block, orgs, instantiated chaincodes and block 2 of channel *common*:
```bash
curl -H "Authorization: Bearer $JWT" http://localhost:3000/channels/common
curl -H "Authorization: Bearer $JWT" http://localhost:3000/channels/common/chaincodes
curl -H "Authorization: Bearer $JWT" http://localhost:3000/channels/common/orgs
curl -H "Authorization: Bearer $JWT" http://localhost:3000/channels/common/blocks/2
```

Invoke chaincode *reference* on channel *common*, it's implemented by chaincode_example_02 and moves 1 from a to b:
```bash
curl -H "Authorization: Bearer $JWT" --header "Content-Type: application/json" http://localhost:3000/channels/common/chaincodes/reference -d '{"fcn":"invoke","args":["a","b","1"]}'
```

Query chaincode *reference* on channel *common* for balances of `a` and `b`:
```bash
curl -H "Authorization: Bearer $JWT" --header "Content-Type: application/json" 'http://localhost:3000/channels/common/chaincodes/reference?fcn=query&args=a'
curl -H "Authorization: Bearer $JWT" --header "Content-Type: application/json" 'http://localhost:3000/channels/common/chaincodes/reference?fcn=query&args=b'
```