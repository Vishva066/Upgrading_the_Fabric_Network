
# Upgrading the Fabric Version

This is a lab manual to explain how to upgrade the fabric version from v2.2 to v2.5.4. This mannual can also be used to upgrade to your prefered version by just mentioning the version number. 



## Introduction

To upgrade the fabric network to the latest version, let’s follow these 3 steps:

**Upgrading your components** 

**Updating the capability level of a channel**

**Enabling the new chaincode lifecycle**

## Upgrading your components

At a high level, upgrading the binary level of your nodes is a two step process:

- Backup the ledger and MSPs.
- Upgrade binaries to the latest version.


## Upgrading ordering nodes

Orderer containers should be upgraded in a rolling fashion (one at a time). At a high level, the ordering node upgrade process goes as follows:

- Stop the ordering node.
- Back up the ordering node’s ledger and MSP.
- Remove the ordering node container.
- Launch a new ordering node container using the relevant image tag.

Repeat this process for each node in your ordering service until the entire ordering service has been upgraded.

## Manual

Here I am using Fabric version 2.2.2 and CA version 1.4.9. Now I am going to upgrade to 2.5.4 Version and CA version 1.5.7

Download fabric-samples

```bash
  curl -sSL https://bit.ly/2ysbOFE | bash -s -- 2.2.2 1.4.9
```

Copy the fabric binaries

```bash
sudo cp fabric-samples/bin/* /usr/local/bin
```

Test network commands:

```bash
cd fabric-samples/test-network/
```

Bring up the network:

```bash
./network.sh -h
```

Create the channel and bring up the database:

```bash
./network.sh up createChannel -ca -s couchdb
```

Checnk whether all the container are running:

```bash
docker ps -a
```

Deploy the chaincode to the channel

```bash
./network.sh deployCC -ccn basic -ccp ../asset-transfer-basic/chaincode-javascript/ -ccl javascript
```

Set the environment varibales:

```bash
export FABRIC_CFG_PATH=$PWD/../config/

export CORE_PEER_TLS_ENABLED=true

export CORE_PEER_LOCALMSPID="Org1MSP"

export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt

export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp

export CORE_PEER_ADDRESS=localhost:7051
```

Invoke the chaincode:

```bash
peer chaincode invoke -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --tls --cafile "${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem" -C mychannel -n basic --peerAddresses localhost:7051 --tlsRootCertFiles "${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt" --peerAddresses localhost:9051 --tlsRootCertFiles "${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt" -c '{"function":"InitLedger","Args":[]}'
```

Query the chaincode:

```bash
peer chaincode query -C mychannel -n basic -c '{"Args":["GetAllAssets"]}'
```

**Backing up the orderer**

Export the orderer container

```bash
export ORDERER_CONTAINER=2e1a3fac8b40 //Replace with your orderer container

export ORDERER_NAME=orderer.example.com
```

Create the backup folder

```bash
mkdir backup

export LEDGERS_BACKUP=backup

export IMAGE_TAG=2.5.4
```

Create a new file called **envorderer.list**

Stop the existing docker container

```bash
docker stop $ORDERER_CONTAINER
```

Copy the stopped docker containers to the backup folder

```bash
docker cp $ORDERER_CONTAINER:/var/hyperledger/ ./$LEDGERS_BACKUP/$ORDERER_CONTAINER

docker cp $ORDERER_CONTAINER:/etc/hyperledger/fabric ./$LEDGERS_BACKUP/Identity
```
Remove the old docker container

```bash
docker rm -f $ORDERER_CONTAINER
```

Replace the latest image and mount the backed up data

```bash
docker run -d -v ./backup/$ORDERER_CONTAINER/:/var/hyperledger/             -v ./backup/Identity:/etc/hyperledger/fabric/             --env-file ./envorderer.list             --name $ORDERER_NAME    -p 7050:7050 -p 7053:7053 -p 9443:9443   --network fabric_test         hyperledger/fabric-orderer:$IMAGE_TAG orderer
```

Check whehter the new docker container is running or not using this command: 

```bash
docker ps -a
```

## Upgrading peers

Peers should, like the ordering nodes, be upgraded in a rolling fashion (one at a time). As mentioned during the ordering node upgrade, ordering nodes and peers may be upgraded in parallel, but for the purposes of this tutorial we’ve separated the processes out. At a high level, we will perform the following steps:

- Stop the peer.
- Back up the ordering node’s ledger and MSP
- Remove chaincode containers and images.
- Remove the peer container.
- Launch a new peer container using the relevant image tag.

Repeat this process for each node in your ordering service until the entire ordering service has been upgraded.


**Backing up the peers**


**PEER1**

Export the peer containers

```bash
export PEER_CONTAINER=8a95cb9f7d10 //Replace with your peer container

export PEER1_NAME=peer0.org1.example.com
```

Create the backup folder

```bash
mkdir peer1_backup

export LEDGERS_BACKUP=peer1_backup

```
Create a new file called **envpeer1.list**

Copy the  docker containers to the backup folder

```bash
docker cp $PEER_CONTAINER:/var/hyperledger/ ./$LEDGERS_BACKUP/$PEER_CONTAINER

docker cp $PEER_CONTAINER:/etc/hyperledger/fabric/ ./$LEDGERS_BACKUP/Identity
```

Stop the existing peer1 container

```bash
docker stop $PEER_CONTAINER
```


Remove the old docker container

```bash
docker rm -f $PEER_CONTAINER
```

Replace the latest image and mount the backed up data

```bash
docker run -d -v ./peer1_backup/$PEER_CONTAINER/production/:/var/hyperledger/production             -v ./peer1_backup/Identity/:/etc/hyperledger/fabric  -v /var/run/docker.sock:/var/run/docker.sock           --env-file ./envpeer1.list             --name $PEER1_NAME   -p 7051:7051 -p 9444:9444  --network fabric_test          hyperledger/fabric-peer:$IMAGE_TAG peer node start
```

Check whehter the new docker container is running or not using this command: 

```bash
docker ps -a
```

**PEER2**

Export the peer containers

```bash
export PEER_CONTAINER=dd9e7b7244d3 //Replace with your peer container

export PEER2_NAME=peer0.org2.example.com
```

Create the backup folder

```bash
mkdir peer2_backup

export LEDGERS_BACKUP=peer2_backup

```
Create a new file called **envpeer2.list**

Copy the  docker containers to the backup folder

```bash
docker cp $PEER_CONTAINER:/var/hyperledger/ ./$LEDGERS_BACKUP/$PEER_CONTAINER

docker cp $PEER_CONTAINER:/etc/hyperledger/fabric/ ./$LEDGERS_BACKUP/Identity
```

Stop the existing peer1 container

```bash
docker stop $PEER_CONTAINER
```

Remove the old docker container

```bash
docker rm -f $PEER_CONTAINER
```

Replace the latest image and mount the backed up data

```bash
docker run -d -v ./peer2_backup/$PEER_CONTAINER/production/:/var/hyperledger/production             -v ./peer2_backup/Identity/:/etc/hyperledger/fabric  -v /var/run/docker.sock:/var/run/docker.sock           --env-file ./envpeer2.list             --name $PEER2_NAME   -p 9051:9051 -p 19051:19051  --network fabric_test          hyperledger/fabric-peer:$IMAGE_TAG peer node start
```

Check whehter the new docker container is running or not using this command: 

```bash
docker ps -a
```

## Upgrading the CA

To upgrade a single instance of Fabric CA server:

- Stop the fabric-ca-server process.
- Ensure the current database is backed up.
- Replace previous fabric-ca-server binary with the upgraded version.
- Launch the fabric-ca-server process.

Repeat this process for each CA until all the CA are updated.

**CA-ORDERER**

```bash
export CA_ORDERER_CONTAINER=7af1328915cc // Replace with your docker container

export CA_ORDERER_NAME=ca_orderer
```

Create the ca-orderer backup folder

```bash
mkdir ca_orderer_backup

export LEDGERS_BACKUP=ca_orderer_backup
```

Backup the contents

```bash
docker cp $CA_ORDERER_CONTAINER:/etc/hyperledger/fabric-ca-server/ ./$LEDGERS_BACKUP/ca
```

Stop the docker container and remove the old container

```bash
docker stop $CA_ORDERER_CONTAINER

docker rm -f $CA_ORDERER_NAME
```

Run the docker ca with upgraded version

```bash
docker run -d -v ./organizations/fabric-ca/ordererOrg:/etc/hyperledger/fabric-ca-server --env-file ./caorderer_backup.list --name $CA_ORDERER_NAME -p 9054:9054 -p 19054:19054 --network fabric_test  hyperledger/fabric-ca:1.5.7 sh -c 'fabric-ca-server start -b admin:adminpw -d'
```

**CA-ORG1**

```bash
export CA_ORG1_CONTAINER=ff6c47da1c5d //Replace with your ca container

export CA_ORG1_NAME=ca_org1 //Replace with your org1 name
```

Create the ca-org1 backup folder

```bash
mkdir ca_org1_backup

export LEDGERS_BACKUP=ca_org1_backup
```

Backup the contents

```bash
docker cp $CA_ORG1_CONTAINER:/etc/hyperledger/fabric-ca-server/ ./$LEDGERS_BACKUP/ca
```

Stop the docker container and remove the old container

```bash
docker stop $CA_ORG1_CONTAINER

docker rm -f $CA_ORG1_NAME
```

Run the docker ca with upgraded version

```bash
docker run -d -v ./organizations/fabric-ca/org1:/etc/hyperledger/fabric-ca-server --env-file ./caorg1_backup.list --name $CA_ORG1_NAME -p 7054:7054 -p 17054:17054 --network fabric_test  hyperledger/fabric-ca:1.5.7 sh -c 'fabric-ca-server start -b admin:adminpw -d'
```

**CA-ORG2**

```bash
export CA_ORG2_CONTAINER=1e0387f15ecb // Replace with your docker container

export CA_ORG2_NAME=ca_org2
```

Create the ca-org2 backup folder

```bash
mkdir ca_org2_backup

export LEDGERS_BACKUP=ca_org2_backup
```

Backup the contents

```bash
docker cp $CA_ORG2_CONTAINER:/etc/hyperledger/fabric-ca-server/ ./$LEDGERS_BACKUP/ca
```

Stop the docker container and remove the old container

```bash
docker stop $CA_ORG2_CONTAINER

docker rm -f $CA_ORG2_NAME
```

Run the docker ca with upgraded version

```bash
docker run -d -v ./organizations/fabric-ca/org2:/etc/hyperledger/fabric-ca-server --env-file ./caorg2_backup.list --name $CA_ORG2_NAME -p 8054:8054 -p 18054:18054 --network fabric_test  hyperledger/fabric-ca:1.5.7 sh -c 'fabric-ca-server start -b admin:adminpw -d'
```

## Upgrade Node SDK clients:

Use NPM to upgrade any Node.js client by executing these commands in the root directory of your application:

```bash
npm install fabric-client@latest

npm install fabric-ca-client@latest
```


## Upgrading the couchdb:

To upgrade CouchDB:

- Stop CouchDB.
- Backup CouchDB data directory.
- Install the latest CouchDB binaries or update deployment scripts to use a new Docker image.
- Restart CouchDB.


**Couchdb0**

```bash
export COUCHDB0_CONTAINER=a5525c4424c6 //Replace with your couchdb container

export COUCHDB0_NAME=couchdb0
```

Create the couchdb backup folder

```bash
mkdir couchdb0_backup

export LEDGERS_BACKUP=couchdb0_backup
```

Backup the contents

```bash
docker cp $COUCHDB0_CONTAINER:/opt/couchdb/data/ ./$LEDGERS_BACKUP/
```

Stop and remove the existing docker container 

```bash
docker stop $COUCHDB0_CONTAINER

docker rm -f $COUCHDB0_NAME
```

Run the couchdb with upgraded docker version

```bash
docker run -d -v ./$LEDGERS_BACKUP/data/:/opt/couchdb/data/ --env-file ./couchdb0_backup.list --name $COUCHDB0_NAME -p 5984:5984 --network fabric_test  couchdb:3.3.2
```

**Couchdb1**

```bash
export COUCHDB1_CONTAINER=583c1c1e0ce2 //Replace this container ID with your own container ID

export COUCHDB1_NAME=couchdb1
```

Create the couchdb backup folder

```bash
mkdir couchdb1_backup

export LEDGERS_BACKUP=couchdb1_backup
```

Backup the contents

```bash
docker cp $COUCHDB1_CONTAINER:/opt/couchdb/data/ ./$LEDGERS_BACKUP/
```

Stop and remove the existing docker container 

```bash
docker stop $COUCHDB1_CONTAINER

docker rm -f $COUCHDB1_NAME
```

Run the couchdb with upgraded docker version

```bash
docker run -d -v ./$LEDGERS_BACKUP/data/:/opt/couchdb/data/ --env-file ./couchdb1_backup.list --name $COUCHDB1_NAME -p 7984:5984 --network fabric_test  couchdb:3.3.2
```

## Upgrade Node chain code shim:

To move to the new version of the Node chaincode shim a developer would need to:

- Change the level of fabric-shim in their chaincode package.json from their old level to the new one.
- Repackage this new chaincode package and install it on all the endorsing peers in the channel.
- Perform an upgrade to this new chaincode. To see how to do this, check out Peer chaincode commands.

```bash
peer chaincode upgrade -o orderer.example.com:7050 --tls --cafile $ORDERER_CA -C mychannel -n mycc -v 1.2 -c '{"Args":["init","a","100","b","200","c","300"]}' -P "AND ('Org1MSP.peer','Org2MSP.peer')"
```

//Replace the necessary arguments as this is just an example.

## Upgrading the capabilities:(When using system channel)

As with any channel configuration update, updating capabilities is, at a high level, a three step process (for each channel):

- Get the latest channel config
- Create a modified channel config
- Create a config update transaction

We will enable these capabilities in the following order:

- Orderer system channel
  - Orderer group
  - Channel group

- Application channels
  - Orderer group
  - Channel group
  - Application group


Set up the environment variables.

```bash
export CHANNEL_NAME=mychannel //Replace the name of the channel with your own channel

export CORE_PEER_LOCALMSPID=Org1MSP

export CORE_PEER_TLS_ENABLED=true

export CORE_PEER_TLS_ROOTCERT_FILE=/home/vishva/Upgrading_version/fabric-samples/test-network/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt

export CORE_PEER_MSPCONFIGPATH=/home/vishva/Upgrading_version/fabric-samples/test-network/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/

export CORE_PEER_ADDRESS=localhost:7051

export ORDERER_CA=/home/vishva/Upgrading_version/fabric-samples/test-network/organizations/ordererOrganizations/example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```

Fetch the latest configuration

```bash
peer channel fetch config channel-artifacts/config_block.pb -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com -c $CHANNEL_NAME --tls --cafile $ORDERER_CA
```

Create the capabilities.json file

Go inside the channel artifacts folder

```bash
cd channel-artifacts/
```


```bash
configtxlator proto_decode --input config_block.pb --type common.Block --output config_block.json

jq .data.data[0].payload.data.config config_block.json > config.json

cp config.json modified_config.json

```

Add the capabilities.json file to the configuration

```bash
jq -s '.[0] * {"channel_group":{"groups":{"Application": {"values": {"Capabilities": .[1].application}}}}}' config.json ../capabilities.json > modified_config.json
```

```bash
configtxlator proto_encode --input config.json --type common.Config --output config.pb

configtxlator proto_encode --input modified_config.json --type common.Config --output modified_config.pb

configtxlator compute_update --channel_id $CHANNEL_NAME --original config.pb --updated modified_config.pb --output config_update.pb

configtxlator proto_decode --input config_update.pb --type common.ConfigUpdate --output config_update.json

echo '{"payload":{"header":{"channel_header":{"channel_id":"'$CHANNEL_NAME'", "type":2}},"data":{"config_update":'$(cat config_update.json)'}}}' | jq . > config_update_in_envelope.json

configtxlator proto_encode --input config_update_in_envelope.json --type common.Envelope --output config_update_in_envelope.pb

```

Sign the channel update configuration  using the peer

```bash
cd ..

peer channel signconfigtx -f channel-artifacts/config_update_in_envelope.pb 
```

Now do the same for Org2

```bash
export CORE_PEER_TLS_ENABLED=true

export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
export CORE_PEER_ADDRESS=localhost:9051

peer channel update -f channel-artifacts/config_update_in_envelope.pb -c $CHANNEL_NAME -o $localhost:7050 --ordererTLSHostnameOverride orderer.example.com --tls --cafile $ORDERER_CA

```