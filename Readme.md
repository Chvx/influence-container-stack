# Setting up containerized Influence environment

The following steps will guide you through setting up your machine to run Influence locally, for development and test purposes, based on Starknet Sepolia. You can chose to run the following:

- client and server only: you will use static data from the restored database snapshot and receive no updates
- indexer: the indexer is used to update your local data from retrieving events from Starknet; you can either point it to a RPC provider (e.g. Infura or Alchemy) or to a local node
- Starknet local node: RPC provider rate-limiting is pretty low and free plans are likely not enough to run the indexer. Setting up a local node is pretty straightforward but will require at least 110GB of storage space.

This guide will **not** cover how to run an Etherum node (the starknet node is standalone and does not validate against L1).

This guide will **not** cover how to run a Starknet Devnet for contracts development (will be tackled later).

You can follow the steps on your own machine or use a dedicated machine on your network. However, if you use a dedicated machine, be aware that your wallet may not allow signing in over a plain http connection, so you will be stuck in spectate mode (I'm working on solving this by adding support for a https reverse proxy).

You do not need high-end hardware to run this container stack, I am running it all including the Starknet node on an old HP workstation (8vCores, 24GB RAM, dedicated 250GB SATA SSD, Ubuntu 24.04 LTS).

# Prerequisites and system configuration

## Install Docker

Follow the official doc for apt install: https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository

*Note: if you already have Docker installed, this guide uses `docker compose`, not `docker-compose`.*

Grant yourself permission on docker, then restart the machine:

```sh
sudo usermod -aG docker $USER
sudo reboot
```

## Prepare directories to persist container data across restarts

The guide assumes a dedicated SSD mounted as `/influencedata`.

Find your disk's name and edit `/etc/fstab` to mount it permanently:

```sh
lsblk
sudo nano /etc/fstab
```

Add the following line (replace `sdb1` with your disk's name) and save the file:

```sh
/dev/sdb1       /influencedata   ext4    defaults    0    0
```

After saving the file, mount the filesystem and that it is mounted correctly:

```sh
sudo mount -a
df -h | grep influencedata
```

Create the directories needed to persist the data for Mongo, ElasticSearch, and Redis. If you want to run a Starknet node, create a directory for it too:

```sh
sudo mkdir -p /influencedata/mongodb-data/db
sudo mkdir -p /influencedata/mongodb-data/config
sudo mkdir -p /influencedata/elasticsearch-data
sudo mkdir -p /influencedata/redis-data
sudo mkdir -p /influencedata/starknet-juno-data
```

ElasticSearch runs with its own user (ID 1000) that needs to be permitted on the new directory:

```sh
sudo chown -R 1000:1000 /influencedata/elasticsearch-data/
```

## Update system configuration:

Allow memory overcommit for redis to avoid background save failures:

```sh
sudo nano /etc/sysctl.conf
```

Add a new line `vm.overcommit_memory = 1`.

Load the new config:

```sh
sudo sysctl --system
```

## Open firewalls

If you are working from a different machine than the one running the containers and the firewall is enabled, you'll need to open firewalls to access:

- the Influence client (port 3000)
- the Influence server API (port 3001)
- the mongo-express frontend (port 8081, only needed for debug)

Run the following command to check the firewal status; if it is `inactive` then everything is open by default.

```sh
sudo ufw status
```

Otherwise, you can allow traffic from your local network with those commands (feel free to restrict further):

```sh
sudo ufw allow from 192.168.0.0/16 to any port 3000 proto tcp
sudo ufw allow from 192.168.0.0/16 to any port 3001 proto tcp
sudo ufw allow from 192.168.0.0/16 to any port 8081 proto tcp
```

# Setting up the Influence stack

## Create working directory

All commands past this point are written assuming that the current working directory is `~/Influence/`.

```sh
mkdir ~/Influence
cd ~/Influence
```

## Clone the AdaliaFoundation repositories

```sh
git clone https://github.com/adaliafoundation/influence-server.git
git clone https://github.com/adaliafoundation/influence-client.git
```

## Environment variables and config files

Every time there is `<InfluenceBoxIP>`, replace it with:

- `localhost` if you are running it on the machine you are using to access the client
- the IP of machine where Docker is running, if working on another machine

**compose.yaml**

Download the `compose.yaml` file from this repo:

```sh
curl https://raw.githubusercontent.com/Chvx/influence-container-stack/refs/heads/main/compose.yaml -o compose.yaml
```

Consider adjusting the following values in the file, especially if not on a trusted network:

- Admin username and password in the mongo service definition as well as in the connection in mongo-express (those credentials should match).
- User and password in mongo-express (this is the user you'll use in case you want to connect to the frontend).
- If you changed any of the persistence directories (e.g. `/influencedata` paths), update them.

**influence-client/.env:**

```sh
cat > ./influence-client/.env <<'EOF'
BUFFER_GLOBAL=1
SKIP_PREFLIGHT_CHECK=1

NODE_ENV=development
REACT_APP_CONFIG_ENV=prerelease
REACT_APP_APP_VERBOSELOGS=1

REACT_APP_API_INFLUENCE=http://<InfluenceBoxIP>:3001
REACT_APP_API_INFLUENCEIMAGE=http://<InfluenceBoxIP>:3001
EOF
```

**influence-client/Dockerfile:**

```sh
cat > ./influence-client/Dockerfile <<'EOF'
# syntax=docker/dockerfile:1

FROM node:20-alpine
WORKDIR /app
COPY package*.json .babelrc .npmrc .nvmrc .slug-post-clean ./
COPY * ./
COPY patches ./patches
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
EOF
```

**influence-client/.dockerignore:**

```sh
cat > ./influence-client/.dockerignore <<'EOF'
node_modules
npm-debug.log
yarn-error.log

.git
.gitignore

.env
.env.*

Dockerfile
.dockerignore

dist
coverage
EOF
```

**influence-server/.env:**

Use a random string for `JWT_SECRET`.

Configure `STARKNET_RPC_PROVIDER` depending on your environment:

- if not running the indexer, leave it blank (your game will not update)
- if using an RPC like Infura or Alchemy, point to their url using your api-key
- if running your local Starknet node, point to the Juno container

Update the `root:password` credentials in MONGO_URL in case you changed them earlier in `compose.yaml` (don't touch the `admin` at the end).

```sh
cat > ./influence-server/.env <<'EOF'
API_SERVER=1
# url as used in the user's browser, for CORS checks
CLIENT_URL=http://<InfluenceBoxIP>:3000
BRIDGE_CLIENT_URL=http://<InfluenceBoxIP>:4000
IMAGES_SERVER=1
IMAGES_SERVER_URL=http://influence-server:3001
MONGO_URL=mongodb://root:password@mongo:27017/influence?authSource=admin
ELASTICSEARCH_URL=http://elasticsearch:9200
REDIS_URL=redis://redis:6379
CLOUDINARY_URL=
NODE_ENV=development
JWT_SECRET=randomstring
ETHEREUM_PROVIDER=http://localhost:8545

#CONTRACT_PLANETS=
#CONTRACT_ASTEROID_TOKEN=
#CONTRACT_ASTEROID_FEATURES=
#CONTRACT_ASTEROID_SCANS=
#CONTRACT_ASTEROID_SALE=
#CONTRACT_ASTEROID_NAMES=
#CONTRACT_ARVAD_CREW_SALE=
#CONTRACT_CREW_TOKEN=
#CONTRACT_CREW_FEATURES=
#CONTRACT_CREW_NAMES=

#STARKNET_RPC_PROVIDER=
#STARKNET_RPC_PROVIDER=https://starknet-sepolia.infura.io/v3/<YourApiKey>
#STARKNET_RPC_PROVIDER=http://juno:6060/v0_7
STARKNET_EVENT_RETRIEVER_RUN_DELAY=2000
ELASTICSEARCH_INDEXER_RUN_DELAY=1000

LOG_LEVEL=debug
EOF
```

**influence-server/Dockerfile:**

```sh
cat > ./influence-server/Dockerfile <<'EOF'
# syntax=docker/dockerfile:1

# Base dependencies
FROM node:18-slim AS base
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .

# Runtime image
FROM node:18-slim as runtime
# Install mongo tools dependencies
RUN apt-get update && apt-get install -y curl gnupg \
  && curl -fsSL https://pgp.mongodb.com/server-6.0.asc | gpg --dearmor -o /usr/share/keyrings/mongodb.gpg \
  && echo "deb [ signed-by=/usr/share/keyrings/mongodb.gpg ] https://repo.mongodb.org/apt/debian bullseye/mongodb-org/6.0 main" \
     > /etc/apt/sources.list.d/mongodb-org.list \
  && apt-get update \
  && apt-get install -y mongodb-database-tools \
  && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY --from=base /app /app
EXPOSE 3001
CMD ["npm", "run", "watch"]

# Unit testing image
FROM node:18-slim AS test
# Mongodb-memory-server dependencies
RUN apt-get update \
 && apt-get install -y --no-install-recommends libcurl4 \
 && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY --from=base /app /app
CMD ["npm", "test"]

EOF
```

**influence-server/.dockerignore:**

```sh
cp ./influence-client/.dockerignore ./influence-server/
```

**influence-server/config/development.json:**

Copy `prerelease.json` to use the game's existing Sepolia contract configuration:

```sh
cp ./influence-server/config/prerelease.json ./influence-server/config/development.json
```

## Run and synchronise the Starknet node (optional)

Download the latest Sepolia snapshot (~55GB):

```sh
wget https://juno-snapshots.nethermind.io/files/sepolia/latest -O /tmp/juno_sepolia.tar
```

Extract the data, you can remove the temporary download afterwards:
```sh
sudo tar -xf /tmp/juno_sepolia.tar -C /influencedata/starknet-juno-data
```

Or, if you want to visualise progress (this is a big file and will take some time), you can install `pv`:

```sh
sudo apt install pv
pv /tmp/juno_sepolia.tar | sudo tar -x --zstd -C /influencedata/starknet-juno-data
```

The downloaded file is not needed anymore and can be removed:

```sh
rm /tmp/juno_sepolia.tar
```

Start the container:

```sh
docker compose up -d juno
```

## Restore influence database in mongo

Download the pre-release database dump (link below taken on 2026-03-18; ~500MB):

```sh
wget -c https://ipfs.io/ipfs/QmUVNASxHtXtS2yDPWDEwTnJRNSiK2kYJcXCNEcovN6Zrz -O influence_prerelease.archive
```

Start the mongo container:

```sh
docker compose up -d mongo
```

Copy the mongodump archive in the mongo container, and open a remote shell:

```sh
docker cp influence_prerelease.archive mongo:/tmp/influence_prerelease.archive
docker exec -it mongo sh
```

Use mongorestore to restore the mongodump. Replace the username in the command below, you will be prompted for the password:

```sh
mongorestore \
  --username <user> \
  --authenticationDatabase admin \
  --db pre-release \
  --archive=/tmp/influence_prerelease.archive \
  --gzip
```

Press `ctrl-D` to exit the shell. Delete the dump file from the container:

```sh
docker exec -it mongo rm /tmp/influence_prerelease.archive
```

## Initialize the ElasticSearch indices

Start the elasticsearch container:

```sh
docker compose up -d elasticsearch
```

Build the influence container images and run the initial setup script to create the index structure. Then, index the data available in the database.

```sh
docker compose build --no-cache
docker compose run --rm influence-tools ./bin/elasticsearch.js initialSetup -v 1
docker compose run --rm influence-tools ./bin/elasticsearch.js reIndex
```

## Run the Influence server and client

Server and client are the default running containers. They will start required dependencies. No connection is made to RPC providers from any of those services.

```sh
docker compose up -d
```

The client is accessed by connecting to `http://<InfluenceBoxIP>:3000`.

## Run the indexer services (optional)

The indexer is responsible for connecting to RPC providers and collecting the events representing game actions and updating the database. It does not need the server or client to be running and will start the required dependencies.

```sh
docker compose --profile indexer up -d
```

# A few helpful things

## Exporting your own mongodump archive

While the mongo container is running, open a remote shell

```sh
docker exec -it mongo sh
```

Dump the DB to an archive file (replace the username with your credential, password will be prompted afterwards):

```sh
mongodump \
  --username <user> \
  --authenticationDatabase admin \
  --db pre-release \
  --excludeCollection=apikeys \
  --archive=/tmp/influence_prerelease.archive \
  --gzip
```

Press `ctrl-D` to exit the shell. Copy the dump file to the host machine, and delete it from the container:

```sh
docker cp mongo:/tmp/influence_prerelease.archive influence_prerelease.archive
docker exec -it mongo rm /tmp/influence_prerelease.archive
```

## Running influence-server project unit tests

A dedicated container image is available that includes the mongo memory server needed to support unit tests and runs the tests on start up:

```sh
docker compose run --rm influence-test
```

## Recomputing packed cache data

Lot data used to display asteroid view is packed and cached in the database. In case you need to recompute this cache from the actual data (e.g. due to logic changes or manual updates), this is how to do it:

Full cache reset and recompute:

```sh
docker compose run --rm influence-tools ./bin/preloadLotData.js --initEmpty
```

For a single lot (lot_id * 2^32 + asteroid_id):

```sh
docker compose run --rm influence-tools ./bin/preloadLotData.js --lots 6910336091291649
```
