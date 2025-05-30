<b>NOTE:</b>
For anyone reading this, the documentation is probably quite outdated as I ended up using Immich due to file size limitation and preference / use case. The web deployment side of documentation might be still useful!

# 📖 About

Ente.io has recently open-sourced their server (museum) which is a backend for various Ente services such as Photos, Auth, etc. There's currently a lack of documentation for deployment of these services. The documentation that does exist is scattered all over the place. The goal of this guide and configuration is to make the deployment as painless as possible with all of the information needed in one place.

Ente server (museum) is deployed using the official docker image provided by Ente. It is configured to run under Traefik reverse proxy to automatically generate and renew certificates. This means no ports are exposed and all of the traffic flows through Traefik reverse proxy before reaching any of the services. 

Ente photos web is deployed using Cloudflare pages. This guide uses a slightly modified github workflow config from Ente's repository for production deployments. The idea is to replicate Ente's production environment as close as possible. 

**NOTE:** This repository is mostly meant for personal use and documentation. Some of configuration choices or methods may not be the best for your particular use case. However, I am open to any improvements or suggestions. Otherwise feel free to fork this as a starting point and modify it however you want. Give it a star if you found this useful! ❤️

**🏗️ Work in progress! 🏗️**
Some of the sections, especially Ente museum are still work in progress. Ente photos deployment to Cloudflare pages is fully finished. 

# 🧰 Getting Started
This guide assumes you have a basic knowledge of Linux, Docker and Docker Compose. You'll also need at least one external S3 storage provider (AWS, Backblaze, Wasabi, etc.). This guide and configuration doesn't include Minio and is rather aimed for production ready deployment with best practices in mind.

## Requirements
- Domain
- Docker and Docker Compose
- Cloudflare account
- S3 storage - (this is not a requirement to get it up and running, but it is required to be able to store photos)
- SMTP provider such as AWS SES (this is not a hard requirement but it's recommended)

## Repository overview
Quick overview of directory structure, services and the ideas behind why it is setup how it is. 

```
ente-selfhost
├── _base
│   ├── compose
│   │   ├── docker-compose.yml
│   │   └── .env
│   └── data
│       └──traefik
└── ente-server
│   ├── compose
│   │   ├── docker-compose.yml
│   │   └── .env
│   └── data
│       ├── museum
│       └── postgres
└── images
    └── ente-server-tools
```

Both `_base` and `ente-server` directories have `compose` and `data` folders. Folder `compose` has all of configuration and variables for deployment. Folder `data` has all of the data generated by services defined inside `compose`. Such a structure clearly separates service stacks which makes it extremely modular and portable. I am not following any standards here, just something I came up with which I personally use for everything Docker related. 

`_base` directory includes Traefik reverse proxy, the idea is to run services which affect the rest of other services in here. It also makes it more modular in case you're already running Traefik, which only requires you to add a docker network `ente` to your own Traefik container to connect rest of the services.

`ente-server` directory includes Ente server (museum). It's currently only running ente server itself and postgres database. Ente server has Traefik labels attached to place the service behind Traefik reverse proxy. Postgres runs locally and is only able to talk to Ente server locally. No ports are exposed on any of the services.

`images` directory includes all of docker images like `ente-server-tools` which simply generates encryption keys for Ente server museum without having to install go on your system.

`.github/workflows/` directory includes all of Github Actions workflows like Ente photos web application building and deployment.

# 🏗️ Installation

## Preparations
First we want to configure, setup domain records and prepare a few things before installation and deployment. 

### Clone repository
```sh
git clone https://github.com/EdyTheCow/ente-selfhost.git
```
### Set correct permissions
Traefik uses file `acme.json` to store certificates it generates for all of domains. Traefik will not start if incorrect file permissions are used, to fix that navigate to `_base/data/traefik` and run:
```sh
sudo chmod 600 acme.json
```
### Create docker network
Since we're running Traefik reverse proxy from directory `_base` we need to connect Ente server to Traefik using external docker network. You can simply create one by running:

```sh
docker network create ente
```
### DNS records
Create DNS record for `DOMAIN_MUSEUM` pointing to the server running `ente-server`. If you're using Cloudflare, it's recommended proxy the record through Cloudflare (orange cloud) and enable `Full (strict)` encryption mode found under `SSL/TLS > Full (strict)` in Cloudflare panel.
## Ente server (museum)

🏗️ Work in progress! 🏗️
### Configure environment variables
Navigate to `ente-selfhost/ente-server/compose/.env/` and edit variables. Only variables marked as required need to be changed, the rest can be left as is.

These variables are unofficial, created by me, and used for basic configuration of containers.

| Variable             |       Default       | Description                                                                                                                              | Required |
| :------------------- | :-----------------: | :--------------------------------------------------------------------------------------------------------------------------------------- | :------: |
| COMPOSE_PROJECT_NAME |        ente         | Prefix for all of containers when started from the compose file                                                                          |    No    |
| DATA_DIR             |       ../data       | Location where data from containers is stored                                                                                            |    No    |
| DOMAIN_MUSEUM        | api.your-domain.com | This is endpoint domain for Ente server which will be used later on to connect Ente services. You can choose whatever subdomain you want |   Yes    |

These variables are official Ente server variables and are used to override settings usually found in various configuration files. We're using variables to keep things tidy, basically all of configuration of ente-server is found in the single `.env` file. Not all of settings are used, you can find a full config file here: https://github.com/ente-io/ente/blob/main/server/configurations/local.yaml

| Variable              |    Default    | Description                                                                                  | Required |
| :-------------------- | :-----------: | :------------------------------------------------------------------------------------------- | :------: |
| ENTE_DB_USER          |     ente      | Database user                                                                                |    No    |
| ENTE_DB_PASSWORD      |       -       | Database password, generate a long and complex password!                                     |   Yes    |
| ENTE_DB_NAME          |    ente_db    | Database name                                                                                |    No    |
| ENTE_LOG-FILE         | set to volume | By default this is set to generate logs at `ente-server/data/museum/logs/`                   |    No    |
| ENTE_SMTP_HOST        |       -       | Host for email service (AWS SES, Gmail, etc.)                                                |    No    |
| ENTE_SMTP_PORT        |       -       | SMTP port commonly either 587 or 465                                                         |    No    |
| ENTE_SMTP_USERNAME    |       -       | Depending on provider you pick, it could be api (AWS SES) or actual account username (Gmail) |    No    |
| ENTE_SMTP_PASSWORD    |       -       | SMTP secret / password                                                                       |    No    |
| ENTE_KEY_ENCRYPTION   |       -       | Generated using instructions below                                                           |   Yes    |
| ENTE_KEY_HASH         |       -       | Generated using instructions below                                                           |   Yes    |
| ENTE_JWT_SECRET       |       -       | Generated using instructions below                                                           |   Yes    |
| ENTE_S3_B2-EU-CEN_KEY |               |                                                                                              |          |
|                       |               |                                                                                              |          |
### Generating jwt, hash and encryption keys
Ente server comes with keys already generated, but it is highly advised against using these keys in production, source: https://github.com/ente-io/ente/blob/main/server/configurations/local.yaml#L158. You can use the provided go command to generate the said keys, but that requires you to install go. I have spent more time than I am willing to admit to craft a one-liner command which generates these keys without ever installing anything on your system, other than Docker. 
The Dockerfile for the image can be found for inspection at `images/ente-server-tools/Dockerfile` and workflow action at `.github/workflows/ente-server-tools-docker-image.yml`. You can either build it yourself using docker build command or use already pre-built docker image from this repo. Dockerfile installs required system packages, clones Ente's repo and downloads required go packages. 

To generate keys simply run this command, which will output three randomly generated keys using tools provided by Ente.
```sh
docker run --rm ghcr.io/edythecow/ente-server-tools go run tools/gen-random-keys/main.go
```

Once you got the keys, copy them into corresponding variables mentioned earlier.
### Setting up SMTP
🏗️ Work in progress! 🏗️
### Setting up S3 storage
🏗️ Work in progress! 🏗️
## Ente photos web
We will be deploying Ente photos web to Cloudflare pages using Github Actions. This is almost exactly the same process as Ente's official production deployment.

The only modification are:
- Action is triggered on push anywhere in repository (since we don't have their branches)
- Checkout action specifies Ente's repository (since we're not running this from their repo)
- Addition of `API_ENDPOINT_URL` variable (so we can set our own endpoint, otherwise it's deployed with their hard coded endpoint) 

The rest of workflow config is unchanged, all of Ente's workflows can be found here: https://github.com/ente-io/ente/tree/main/.github/workflows. This kind of setup allows us to pull and deploy latest changes directly from Ente's repository automatically by simply triggering workflow action on our end. 
### Setup Github Actions workflow
The idea is to setup a workflow using Github Actions. You can either fork this repository or copy workflow config found at `.github/workflows/web-deploy-photos.yml` into your own repository. You should be good to go as long as repository you own has the workflow config in the same mentioned location. 
### Cloudflare credentials
We will need Cloudflare account ID and API token. You can follow Cloudflare's official guide to find these: https://developers.cloudflare.com/pages/how-to/use-direct-upload-with-continuous-integration/#get-credentials-from-cloudflare. Use the same exact permissions for API token as described in the guide.
### Configure Github Actions secrets
Navigate to your repository which has the workflow config from previous steps and navigate to `Settings > Secrets and Variables > Actions` and create three repository secrets. 

| Variable              | Description                                                                                                                                     |
| :-------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------- |
| CLOUDFLARE_ACCOUNT_ID | Your Cloudflare account ID                                                                                                                      |
| CLOUDFLARE_API_TOKEN  | Your Cloudflare API token you have generated                                                                                                    |
| API_ENDPOINT_URL      | Endpoint URL you have set as a DOMAIN_MUSEUM variable in previous steps. Make sure this is a full URL. <br>Example: https://api.your-domain.com |
### Cloudflare pages project
In order to push Ente photos web application to Cloudflare pages, we have to create a Cloudflare pages project. In Cloudflare navigate to `Workers & Pages > Overview > Create application > Pages`. Under `Create using direct upload` click on `Upload assets` then provide `Project name`, this has to match the project name defined in workflow config. By default use `ente-web-photos` as project name unless you have manually changed it in workflow config. Click `Create project`, done! You don't have to upload anything, we just needed to create a Cloudflare pages project so Github Actions workflow is able to use it to deploy ente photos web application. You can now either re-run Github Action or trigger it by pushing to the repository.
### Finishing steps
You should now be able to navigate to preview URL found under the newly created project on Cloudflare pages. If you see traffic on Ente server (museum) console, it means it has successfully connected and is working! If that's not the case, try restarting Ente server (museum) and see if that helps. When you have confirmed it working, you can set a custom domain by navigating to `Workers & Pages > Overview > ente-web-photos > Custom domains`. 

Congrats! You are now running fully automated deployment of Ente web photos! 
# 📱 Mobile
You can use Ente server (museum) endpoint we defined earlier to connect to your own instance by tapping 7 times on the logo while you're in onboarding screen. Source: https://help.ente.io/self-hosting/guides/custom-server/
# 🔧 Administration
🏗️ Work in progress! 🏗️
## Ente CLI
🏗️ Work in progress! 🏗️
# 📚 Resources
This guide is based on sources provided below.
- Ente web deployments https://github.com/ente-io/ente/blob/main/web/docs/deploy.md
- Ente's .env for web photos https://github.com/ente-io/ente/blob/main/web/apps/photos/.env
- Cloudflare pages deployment using Github Actions https://developers.cloudflare.com/pages/how-to/use-direct-upload-with-continuous-integration/
- Ente's Github Actions workflows  https://github.com/ente-io/ente/tree/main/.github/workflows
- Define custom server on mobile app https://help.ente.io/self-hosting/guides/custom-server/
- Ente-server full configuration file https://github.com/ente-io/ente/blob/main/server/configurations/local.yaml
- Docker setup from official Ente docs https://help.ente.io/self-hosting/guides/external-s3
# 📄 TO-DO
- Docker container as alternative for Ente web photos deployment
- Deployment of additional Ente services (E.g. Auth)
- Automated backups using Ente CLI for export of images and Restic for backup to AWS S3 Glacier
- Automated offsite backups for ente-server compose and database
- Addition of Prometheus for metrics 
- Ansible & Terraform alternatives for deployment
