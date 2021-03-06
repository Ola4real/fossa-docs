# Installing FOSSA On-Prem

**This doc only applies for FOSSA on-prem users!  Ignore if you use app.fossa.io**

This guide will help you set up your machine's environment to install FOSSA on `Ubuntu 14.04 LTS`. Make sure you have the necessary [prerequisites](../advance-prerequisites) before beginning. 

**Important! Run this guide as a superuser via `sudo -s`.**

There are 2 ways in set your environment up for Fossa:

1. Using the automated setup script: `setup.sh` **(preferred)**
2. Manually

## Automated Setup

Execute the following command and follow the prompt on the screen:

```bash
# Download and run the environment setup script
mkdir -p ~/fossa && curl -L https://github.com/fossas/fossa-installer/archive/v0.0.12.tar.gz | tar -zxv -C ~/fossa --strip-components=1 && chmod a+x ~/fossa/boot.sh && sudo ln -sf ~/fossa/boot.sh /usr/local/bin/fossa && pushd fossa && sudo ~/fossa/setup.sh && popd
```

This script should walk you through setting up your environment and configuring FOSSA for first-time boot, creating a root directory at `{HOME}/fossa` and a CLI you can run using `fossa {command}`.  

Once finished, you're ready to download and run the appliance. You will be prompted for a `username, password and email` which you should have received after purchasing FOSSA.  Contact `support@fossa.io` if you haven't already been given those credentials.

```bash
# Download and install FOSSA (this may take a while)
fossa init

# Run FOSSA 
fossa start 4

# Note: '4' refers to the number of analysis agents to launch with FOSSA.  
# The more agents you run, the faster & greater your analysis load.
# Reccomended max agents = GB Avail. Mem/2, rounded down (i.e. 32GB RAM/2 = 16 agents)
```

## Manual Setup

### 1. Set up the environment

```bash
# Update the registry
apt-get update
apt-get -y install apt-transport-https ca-certificates
apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D

# Download and install docker, postgres
# NOTE: Do not use Docker 1.9.1 because of: https://github.com/docker/docker/issues/18180
echo deb https://apt.dockerproject.org/repo ubuntu-trusty main >> /etc/apt/sources.list.d/docker.list
apt-get update
apt-get purge lxc-docker
apt-get install -y docker-engine postgresql-9.3 postgresql-contrib-9.3 postgresql-server-dev-9.3 curl tar default-jdk

# Replace "ubuntu" with your username, if it's different
usermod -aG docker ubuntu

# Edit docker config to use "devicemapper" over "aufs" due to issues with aufs on Ubuntu
echo "DOCKER_OPTS=\"--storage-driver=devicemapper\" --storage-opt dm.basesize=20G" >> /etc/default/docker

# Configure forwarding
sudo ufw disable

# Find the line net.ipv6.conf.default.forwarding=1 and uncomment it (or add it) in the file underneath:
vi /etc/sysctl.conf
sudo sysctl -p /etc/sysctl.conf

service docker restart
```
### 2. Set up the Postgres database

In the machine that's running postgres (could be the same), run the following:

```bash
mkdir -p ~/pg_fossa && curl -L https://github.com/fossas/pg_fossa/archive/v1.1.tar.gz | tar -zxv -C ~/pg_fossa --strip-components=1 && sudo cp -R ~/pg_fossa/* $( pg_config | grep SHAREDIR | awk '{print $3}' )/extension/

sudo -u postgres psql -c "CREATE DATABASE fossa"
sudo -u postgres psql -c "CREATE DATABASE rubygems"

# replace the default 'fossa123' password with what you have in config.env
sudo -u postgres psql -c "CREATE USER fossa WITH PASSWORD 'fossa123' WITH CREATEUSER;"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE fossa TO fossa;"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE rubygems TO fossa;"

# Install trigram extension
sudo -u postgres psql fossa -c "CREATE EXTENSION IF NOT EXISTS pg_trgm"

# Install pg_fossa extension
sudo -u postgres psql fossa -c "CREATE EXTENSION IF NOT EXISTS pg_fossa"

# In the file below, find the IPv4 host configuration and make sure it looks like this:
# host    all             all             0.0.0.0/0            md5
sudo -u postgres vi /etc/postgresql/9.3/main/pg_hba.conf

# In this file, find listen_addresses and set it to '0.0.0.0'
sudo -u postgres vi /etc/postgresql/9.3/main/postgresql.conf

sudo service postgresql restart
```

If you have a separate postgres instance, make sure you add those connection details to `config.env`.

### 3. (Optional) Prepare Integrations

#### Bitbucket/Stash:

If you'd like FOSSA to integrate with Bitbucket Server/Stash, make sure you prepare the following:

1. Location of Bitbucket Server on your network
2. A bitbucket login for FOSSA (default username/password `fossabot`/`fossa123`) with...
  - **Global read access** (i.e. to clone repos behind your firewall) 
  - **Ability to create application links** via `Admin > Application Links` 

Once FOSSA is running, `{FOSSA_HOST}/docs/integrations/bitbucket-server-(stash)` will have futher instructions on setup.

#### JIRA Issue Tracker:

To run the Atlassian JIRA setup, make sure you create a `fossabot` account with global access to create/edit tickets as well as an admin account to install webhooks. 

Once FOSSA is running, `{FOSSA_HOST}/docs/integrations/jira-issue-tracker` will have futher instructions on setup.

#### Github: 

TBA

#### Cocoapods API:

Make sure that you have the fossa-cocoapods-api container IP mapped to what is listed as your `cocoapods_api__hostname` config in `config.env`. The Cocoapods-api migration will create a public/private key pair if one does not exist on your host machine at `/root/.ssh/id_rsa'.

### 4. Run the FOSSA installer

As part of the installer, you will be prompted for a `username, password and email`.  Contact `support@fossa.io` if you haven't already been given those credentials.

```bash
# Download and run the installer
mkdir -p ~/fossa && curl -L https://github.com/fossas/fossa-installer/archive/v0.0.12.tar.gz | tar -zxv -C ~/fossa --strip-components=1 && chmod a+x ~/fossa/boot.sh && sudo ln -sf ~/fossa/boot.sh /usr/local/bin/fossa && fossa init

# Configure FOSSA first-time
vi ~/fossa/config.env

# Run FOSSA 
fossa start 4

# Note: '4' refers to the number of analysis agents to launch with FOSSA.  
# The more agents you run, the faster & greater your analysis load.
# Reccomended max agents = GB Avail. Mem/2, rounded down (i.e. 32GB RAM/2 = 16 agents)
```

## Updating FOSSA

See the [FOSSA CLI commands](../managing-instance) for more details on how to manage and update the appliance.
