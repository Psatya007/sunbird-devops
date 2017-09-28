# Pre-requisites

You will need servers with the following minimum system requirements:
- Operating System: Ubuntu 16.04 LTS
- RAM: 7GB
- CPU: 2 core, >2 GHz
- root access (should be able to sudo)

# Variables relevant to deployment
- **implementation-name** - Name of your sunbird implementation. Let's say for the sake of this document, it is `ntp`. As you may know, National Teacher Platform aka Diksha is also a Sunbird implementation.
- **environment-name** - Name of the environment you are deploying. Typically, it is one of development, test, staging, production, etc. For this document, lets say we are setting up a `production` environment.

# Step 1: Provisioning your servers
For a non production setup, you could skip the automation and proceed to the manual steps. If however, you are setting up Sunbird and are not sure if you are setting up the infrastructure correctly, or if you plan to roll out your implementation to serious users, automation can help you setup your environment the same way we set it up.
## Automated
The following set of scripts create the network and servers needed to run Sunbird. With the default configuration, you will be creating 3 servers, with the above mentioned min. requirement. A little knowledge about Azure: VNet, Resource Group, etc would help but is not necessary.
### Automation for Azure 
**Run Time**: 30 mins first time. Scripts can be re-tried and will not create a new set of servers every time. Some configurations cannot be changed, for instance, the server type. However, you can add/reduce the number of servers and re-run if you want to scale up or down.

Run the following steps from a machine which is connected to the internet:
- Clone the sunbird-devops repo using `git clone https://github.com/project-sunbird/sunbird-devops.git`
- Run `./sunbird-devops/deploy/generate-config.sh ntp production provision` This will create config files for you in `./ntp-devops/test/azure`. Here, `ntp` is the **implementation-name** and `production` is the **environment-name**.
- Edit BOTH the new config files `azuredeploy.parameters.json` and `env.sh` as per your requirements. 
- Run `export DEPLOYMENT_JSON_PATH=<absolute path of azuredeploy.parameters.json>`. For instance, on my laptop it is `export DEPLOYMENT_JSON_PATH=/Users/shashankt/code2/sunbird/flamingo-devops/test/azure/`
- Run `cd sunbird-devops/deploy`
- Run `./provision-servers.sh`
- Login to Azure when CLI instructs
- Wait for deployment to complete
- Check on Azure portal: Resource Group -> Deployments -> Click on deployment to see deployment details. 
- Try to SSH. If your `masterFQDN` from deployment details was `production-1a.centralindia.cloudapp.azure.com` you can ssh using `ssh -A ops@production-1a.centralindia.cloudapp.azure.com`
- If you could SSH, you have successfully created the server platform.

### Others
Not automated as of now but you are free to contribute back! Send in a PR.
## Manual
Get 2 servers and prepare to get your hands dirty when needed. 1st server would serve as the DB server and the 2nd, the application server plus the administration server. Note that the default automation creates 3 servers because it separates the application and the administration server.

# Step 2: Setup your DBs
You are free to either use existing DBs, create DBs manually or run the following automation scripts to create them. The DBs Sunbird uses are:
- Cassandra
- Postgres
- Mongo
- Elasticsearch

## Preparation
Run the following steps starting from your local machine:
- SSH into the `db-server`. If you have not edited the default configuration, then the name of the DB VM would be `db-1`. Automated setup does not expose the DB to the Internet, so to SSH into the DB, you will need to SSH to `vm-1` (check out `masterFQDN` above) with `ssh -A` (key forwarding) and then SSH to `db-1`.
- Clone the sunbird-devops repo using `git clone https://github.com/project-sunbird/sunbird-devops.git`
- Run `./sunbird-devops/deploy/generate-config.sh <implementation-name> <environment-name>`. Example `./sunbird-devops/deploy/generate-config.sh ntp production db`. This creates `ntp-devops` directory with *incomplete* configurations. You **WILL** need to supply missing configuration.
- Modify all the configurations under `# DB CONFIGURATION` block in `<implementation-name>-devops/ansible/inventories/<environment-name>/group_vars/<environment-name>`

## DB creation
### Via automation
**Run Time**: 15-30 mins to prepare and 30 mins to complete.
Following is a set of scripts which install the DBs into the `db-server` and copy over `master` data.
- Run `cd sunbird-devops/deploy`
- Run `sudo ./install-dbs.sh <implementation-name>-devops/ansible/inventories/<environment-name>`. This script takes roughly 10-15 mins (in an environment with fast internet) and will install the databases.
### Manual
Refer to DB user guides.

# Step 3: Initialize DBs 
- Run `sudo ./init-dbs.sh <implementation-name>-devops/ansible/inventories/<environment-name>` to initialize the DB.

# Step 4: Setup Application and Core services
- SSH into `admin-server`. If you have used automated scripts used here, then this server would be `vm-1`.
- Clone the sunbird-devops repo using `git clone https://github.com/project-sunbird/sunbird-devops.git`
- Copy over the configuration directory from the DB server(`<implementation-name>-devops`) to this machine
- Modify all the configurations under `# APPLICATION CONFIGURATION` block
- The automated setup also creates a proxy server and like all proxy servers, it will require a SSL certificate. Details of the certificates have to added in the configuration, please see [this wiki](https://github.com/project-sunbird/sunbird-commons/wiki/Updating-SSL-certificates-in-Sunbird-Proxy-service) for details on how to do this. Note: If you don't have SSL certificates and want to get started you could generate and use [self-signed certificates](https://en.wikipedia.org/wiki/Self-signed_certificate), steps for this are detailed in [this wiki](https://github.com/project-sunbird/sunbird-commons/wiki/Generating-a-self-signed-certificate)
- Run `cd sunbird-devops/deploy`
- Run `sudo ./install-deps.sh`. This will install dependencies.
- Run `sudo ./deploy-apis.sh <implementation-name>-devops/ansible/inventories/<environment-name>`. This will onboard various APIs and consumer groups.

**Note:** Next 2 steps are necessary only when the application is being deployed for the first time and could be skipped for subsequent deploys.

- deploy-apis.sh script will print a JWT token that needs to be updated in the application configuration. To find the token search the script output to look for "JWT token for player is :", copy the corresponding token. Example output below, token is highlighted in italics:

  > changed: [localhost] => {"changed": true, "cmd": "python /tmp/kong-api-scripts/kong_consumers.py 
  /tmp/kong_consumers.json ....... "**JWT token for player is :**
  *eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJlMzU3YWZlOTRmMjA0YjQxODZjNzNmYzQyMTZmZDExZSJ9.L1nIxwur1a6xVmoJZT7Yc0Ywzlo4v-pBVmrdWhJaZro*", "Updating rate_limit for consumer player for API cr......"]}

- Update `sunbird_api_auth_token` in you configuration with the above copied token. 
- Run `sudo ./deploy-core.sh <implementation-name>-devops/ansible/inventories/<environment-name>`. This will setup all the sunbird core services. 
- Run `sudo ./deploy-proxy.sh <implementation-name>-devops/ansible/inventories/<environment-name>`. This will setup sunbird proxy services. 

# Step 4: Check Installation
**TODO** Need link to functional documentation to perform just enough user flows to ensure Sunbird implementation is functional

# Step 5: Upgrade with a new version of Sunbird
**TODO** This section details how to take a new release version of Sunbird.

# Step 6: Customize assets
**TODO** This section will explain how to make cosmetic changed to Sunbird to give a custom look and feel.

# Step 7: Customize sunbird
**TODO** You can also build software extensions to Sunbird custom built to your requirements. Look forward to more detail here.