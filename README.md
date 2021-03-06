# IO (Digital Citizenship) infrastructure Terraform modules

The repository contains the Terraform modules used to provision and maintain the IO infrastructure. This should be considered a library of modules to be called from the [Terraform live infrastructure repository](https://github.com/teamdigitale/io-infrastructure-live).

## What is IO?

More informations about the IO can be found on the [Digital Transformation Team website](https://teamdigitale.governo.it/en/projects/digital-citizenship.htm)

## What tools should I know to work in this repo?

* [Terraform](https://www.terraform.io/)

* [Terragrunt](https://github.com/gruntwork-io/terragrunt)

>NOTE: The project is currently still using Terraform 0.11.x and -consequentially- compatible Terragrunt versions (0.18.x)

## Repository directories and files structure

The "live" Terraform component in this repository have been structured in a hierarchical way, following the best-practices described by [Terragrunt](https://github.com/gruntwork-io/terragrunt).

```
Environment
    |_ Deployment area
        |_ Module
```

The root folder contains one or more *deployment environments*, for example *dev* or *prod*.
There's an extra -service- environment, called *infra*, in which the *io.italia.it* DNS zone and the resources used by Terraform itself live. While the *infra* resource group and the resources needed by Terraform (such as a storage account and a dedicated Azure keyvault) are pre-created with a [bash az-init.sh script](utils/README.md), Terraform manages the DNS zone and its records.

Each *environment* contains one or more *deployment geographical areas*, for example *westeurope*.

Each deployment area contains one or more *live components* that have a one to one correspondence with a Terragrunt module. The Terragrunt modules are possibly versioned and maintained in a [separate repository](https://github.com/teamdigitale/pdnd-infra-tf-modules).

Variable values -passed to the modules in input- and the main Terragrunt configuration files are stored in *terraform.tfvars* files located in each live component folder.

Modules can also optionally inherit shared variables from higher level folders. These variables may be stored at each level of the hierarchy in the *vars.tfvars* files. For example, the *westeurope* folder under each environment contains the variable `location = "westeurope"`, that is inherited by all the underlying live components.

There's a *naming convention* around folders and files. First of all, they should all be lowercase. The first part of the folder name usually describes what the component does (for example, if it's a kubernetes cluster, a function or a virtual network); the second part reflects the unique name of the specific resource on Azure. Underscores are used to separate words; dashes separate words referring to the Azure resource name, which uses by convention dashes as well. For example, if a Kubernetes cluster on Azure is named *k8s-01*, the corresponding live TF component folder could be named as *kubernetes_cluster_k8s-01*.

## Prerequisites

The following softwares need to be installed on your machine before moving forward:

* Python 3 - used by the Azure CLI

* [Azure CLI](https://docs.microsoft.com/it-it/cli/azure/install-azure-cli?view=azure-cli-latest)

* [Terraform v0.11.14](https://www.terraform.io/)

* [Terragrunt v0.18.2](https://github.com/gruntwork-io/terragrunt)

>NOTE: it's strongly suggested to install both the tools manually copying their binary in the local path.

>NOTE: Terraform 0.12 and related Terragrunt versions are still not supported. Make sure to use compatible versions, and possibly to use the ones indicated above. Brew or other automated tools try to install the latest version available, which is not alwasy necessarily compatible.

Additional requirements:

* Ask to another Administrator to invite you on the Azure subscription you've to run the scripts aginst, and to assign you a *Contributor role*.

* Ask to another Administrator to set (at least) a *Reader access policy* on the Keyvauly *io-infra-keyvault* in the resource group *io-infra-rg*

## Authentication and environment variables export

Before being able to apply Terragrunt configurations, users need to authenticate on Azure and export some environment variables, needed by Terraform to operate.

To authenticate on Azure, use the Azure CLI you've downloaded, running:

```shell
az login
```

A web page will show up. When the authentication process is complete, you'll be able to go back to the Terminal.

Copy the *.env.example* file located in the *utils* directory to a *.env* file (keep it in the same location) and fill in the values required. You can get the missing values from the Azure GUI (*portal.azure.com*) or from a colleague. Alternatively, you can directly ask to another Administrator for a copy of the final *.env* file.

Finally, source the *az-export.sh* script in the *utils* folder (while you are in the utils folder):

```shell
source az-export.sh
```

You can now apply the Terragrunt configurations.

More info about the *utils scripts* can be found in the [README](utils/README.md) located in the *utils* folder.

## Terragrunt usage overview

Each live component applied along the hierarchy will also inherit the variable values from the top *vars.tfvars* and *terraform.tfvars* files. Variables re-declared at lower level of the hierarchy can also override the same variables, declared at higher levels of the hierarchy.

To provision a specific component, go in the live component folder and run

```shell
terragrunt apply
```

If you have not committed your Terragrunt modules yet, and you'd like to test the scripts using a local copy of the modules run:

```shell
terragrunt apply --terragrunt-source PATH_TO_YOUR_THE_LOCAL_MODULE_DIR
```

>HINT: Substitute apply with destroy or plan to destroy a resource or preview the effects of its creation.

Sometimes, you may want to provision the entire environment. To do so, go into the environment folder (for example *dev* or *dev/westeurope*) and run

```shell
terragrunt apply-all
```

Terragrunt will try to apply all the components under the folder where the Terragrunt command is applied, understanding what module should be applied first and what should be applied later, depending on the dependencies defined in the components' *terraform.tfvars* files.

>NOTE: Substitute apply-all with destroy-all or plan-all to destroy a set of resources or preview the effects of their creation.

## How to build a new infrastructure from scratch

The paragraph describes the action to perform and the order in which they should be applied, in order to bring up a new IO infrastructure from scratch.

### Notes about secrets and certificates creation/import

While some of the secrets and certificates needed by some components below are automatically generated and uploaded during the creation of other resources, some secrets and certificates need to be uploaded manually. This may not necessarily be an easy operation as it would seems. Here are few tips that may help:

* The [secrets-copier.sh utility script](utils/secrets-copier.sh) automates copy operation of secrets between two keyvaults

* Certificates should be generally uploaded manually in the *certificates* section of the keyvault. Uploading certificates in PEM format may be error prone, since the private key and the public certificate should be manually combined in a unique file. We found out that the best way to upload certificates is to previously convert them in pfx format with *openssl*. For example, `openssl pkcs12 -inkey your_private.key -in your_public.crt -export -out your_pfx.pfx`

* Some secrets contain the keyword certificate, even if they do not represent a private/public key-pair, but only a public certificate. This is the reason why they have been uploaded as secrets and not certificates. This is for example the case of *k8s-pagopa-proxy-test-secrets-pagopa-ca-chain-certs*, which only represents the PagoPA public certificate. Do not try to upload it as certificate. It simply won't work!

### Step by step

The commands assume you're trying to provision the *dev* environment and that you've already initialized once the Azure infrastructure with the [az-init.sh script](utils/README.md) and that you already have a working [.env file](utils/README.md).

```shell
cd io-infrastructure-live

# Export env vars
cd utils
source az-export.sh

# Optionally set TERRAGRUNT_SOURCE_UPDATE to true
# to download the latest Terraform module each time
# an apply is executed
export TERRAGRUNT_SOURCE_UPDATE=true

# Deploy the components of the dev environment, under the West Europe Azure DC location
cd ../dev/westeurope

# Resource group
cd resource_group && terragrunt apply

# Networking: vnet and subnets
cd ../vnet_common && terragrunt apply
cd ../subnet_redis && terragrunt apply
cd ../subnet_function_app_app && terragrunt apply
cd ../subnet_function_app_admin && terragrunt apply
cd ../subnet_function_app_services && terragrunt apply
cd ../subnet_function_app_public && terragrunt apply
cd ../subnet_apim && terragrunt apply
cd ../subnet_ag-frontend && terragrunt apply
cd ../subnet_ag-to-k8s-01 && terragrunt apply
cd ../subnet_k8s-01 && terragrunt apply

# Private DNS zone
cd ../dns_zone_private_common && terragrunt apply

# Keyvault
cd ../key_vault && terragrunt apply

# CosmosDB: account, sql-database, sql-containers
cd ../cosmosdb_account_01 && terragrunt apply
cd ../cosmosdb_sql_database_db-01 && terragrunt apply
cd ../cosmosdb_container_message-status && terragrunt apply
cd ../cosmosdb_container_messages && terragrunt apply
cd ../cosmosdb_container_notification-status && terragrunt apply
cd ../cosmosdb_container_notifications && terragrunt apply
cd ../cosmosdb_container_profiles && terragrunt apply
cd ../cosmosdb_container_sender-services && terragrunt apply
cd ../cosmosdb_container_services && terragrunt apply

# Eventhub
cd ../eventhub_apim && terragrunt apply
```

**Manual operations required**

The following secrets need to be manually inserted in the Azure Keyvault:

| Secret  | Description    |
| ------- | -------------- |
|`notification-hub-01-bundle-id` | *Also called the "App ID." This is the bundle identifier for the application.* |
|`notification-hub-01-gc-m-key` | *The api key for Google Cloud Messaging.* |
|`notification-hub-01-key-id` | *Identifier of the private key generated in the Apple Developer portal.* |
|`notification-hub-01-team-id` | *Also called the "Prefix" or "App Prefix." This is the identifier for the organization in the Apple Developer portal.* |
|`notification-hub-01-token` | *Also called the "Key" or "Private Key." This is obtained from the .p8 file generated on the Apple Developer portal. The key must have APNS enabled (which is selected on the Apple Developer portal when generating the key). The value must have the PEM header/footer stripped from it when you supply it to the NH Portal/API.* |

```shell
# Notificationhub
cd ../notification_hub && terragrunt apply

# Storage
cd ../storage_account_fn2admin && terragrunt apply
cd ../storage_account_fn2app && terragrunt apply
cd ../storage_account_fn2services && terragrunt apply
cd ../storage_account_fn2public && terragrunt apply
cd ../storage_account_servicedata && terragrunt apply
cd ../storage_account_appdata && terragrunt apply
cd ../storage_container_cached && terragrunt apply
cd ../storage_container_message-content && terragrunt apply
cd ../storage_queue_createdmessages && terragrunt apply
cd ../storage_queue_emailnotifications && terragrunt apply
cd ../storage_queue_profileevents && terragrunt apply
cd ../storage_queue_webhooknotifications && terragrunt apply
cd ../storage_table_subscriptionsfeedbyday && terragrunt apply
cd ../storage_table_validationtokens && terragrunt apply

# Redis cache
cd ../redis_cache-01 && terragrunt apply

# App Insights
cd ../app_insights && terragrunt apply
```

**Manual operations required**

The following secrets need to be manually inserted in the Azure Keyvault:

| Secret  | Description    |
| ------- | -------------- |
|`fn2-commons-mailup-secret` | *The secret used to connect to the MailUP service.* |
|`fn2-commons-mailup-username` | *The username used to connect to the MailUP service.* |
|`fn2-app-public-api-key` | *TBD* |
|`fn2-services-webhook-channel-url` | *The IO app backend callback URL, formatted as https://APP_BACKEND_ADDR/api/v1/notify?token=APP_BACKEND_PRE_SHARED_KEY where the APP_BACKEND_PRE_SHARED_KEY is a lower-case, alpha-numeric 30 chars string randomly generated* |

```shell
# Function-app admin
cd ../function_app_service_plan_fn2admin && terragrunt apply
cd ../function_app_admin && terragrunt apply
cd ../function_app_admin_config && terragrunt apply

# Function-app app
cd ../function_app_service_plan_fn2app && terragrunt apply
cd ../function_app_app && terragrunt apply
cd ../function_app_app_config && terragrunt apply

# Function-app services
cd ../function_app_service_plan_fn2services && terragrunt apply
cd ../function_app_services && terragrunt apply
cd ../function_app_services_config && terragrunt apply

# Function-app public
cd ../function_app_service_plan_fn2public && terragrunt apply
cd ../function_app_public && terragrunt apply
cd ../function_app_public_config && terragrunt apply
```

**Manual operations required**

* Navigate to the [Azure GUI](https://portal.azure.com)

* Go to Function App
 
* Navigate to io-dev-fn-2-admin -> Function app settings
 
* Add new host key: as name, input *ApimHostKey*; leave the value empty to auto-generate the host key. Copy it somewhere. You'll need it soon

* Go back to Function App
 
* Navigate to io-dev-fn-2-services -> Function app settings
 
* Add new host key: as name, input *ApimHostKey*; leave the value empty to auto-generate the host key. Copy it somewhere. You'll need it soon

* The following secrets need to be manually inserted in the Azure Keyvault

| Secret  | Description    |
| ------- | -------------- |
|`apim-01-fn2-admin-host-key` | *The auto-generated host key of the admin function.* |
|`apim-01-fn2-services-host-key` | *The auto-generated host key of the services function.* |
|`apim-01-fn2-public-host-key` | *The auto-generated host key of the public function.* |
|`generated-cert` | *The application gateway certificate consumed by the APIM, to be loaded manually in pfx format.* |

```shell
# API Management (APIM)
cd ../api_management && terragrunt apply
cd ../api_management_properties && terragrunt apply
cd ../api_management_apis && terragrunt apply
cd ../api_management_products && terragrunt apply
```

**Manual operations required**

* Remove *Subscription required* from APIs to public functions: access the Azure portal -> go to the API Management section -> enter your APIM -> APIs -> IO API Public -> Settings -> Remove the flag from *Subscription required*

* The following secrets need to be manually inserted in the Azure Keyvault

| Secret  | Description    |
| ------- | -------------- |
|`application-gateway-to-apim-01-cert` | *The application gateway certificate, to be loaded manually in pfx format.* |

```shell
# Application gateways
cd ../public_ip_ag-to-apim-01 && terragrunt apply
cd ../public_ip_ag-to-k8s-01 && terragrunt apply
cd ../application_gateway_ag-to-apim-01 && terragrunt apply
cd ../application_gateway_ag-to-k8s-01 && terragrunt apply
```

**Manual operations required**

The following secrets need to be manually inserted in the Azure Keyvault:

| Secret  | Description    |
| ------- | -------------- |
|`k8s-app-backend-secrets` | *A JSON with backend secrets {"api-key": "XXX", "appinsights-instrumentationkey": "XXX", "azure-nh-endpoint": "XXX", "pre-shared-key": "XXX", "redis-password": "XXX"}* |
|"`k8s-io-onboarding-pa-api`" | *A JSON with io-onboarding-pa-api secrets {"email-password": "your_email_password", "postgresql-password": "your_postgresql_password_here", "postgresql-replication-password": "your_postgresql_replication_password_here", "arss-identity-otp-pwd": "your_aruba_identity_otp_pwd", "arss-identity-user-pwd": "your_aruba_ientity_user_pwd"}* |
|`k8s-developer-portal-backend-secrets` | *A JSON with portal backend secrets {"admin-api-key": "XXX”, "admin-api-url": "https://api.dev.io.italia.it", "appinsights-instrumentationkey": "XXX”, "arm-subscription-id": "XXX”, "arm-tenant-id": "XXX”, "client-id": "XXX”, "client-secret": "XXX”, "cookie-iv": "XXX”, "cookie-key": "XXX, "service-principal-client-id": "XXX”, "service-principal-secret": "XXX”, "service-principal-tenant-id": "XXX, "tenant-id": "XXX”}* |
|`k8s-pagopa-proxy-prod-secrets` | *a Json with the pagopa-proxy application secrets: {"pagopa-password": "XXX", "pagopa-id-psp": "XXX", "pagopa-id-int-psp": "XXX", "pagopa-id-canale": "XXX", "pagopa-id-canale-pagamento": "XXX", "redis-db-password": "XXX"}* |
|`k8s-pagopa-proxy-test-secrets` | *a JSON with the pagopa-proxy application secrets used to connect to the PagoPA test environment: {"pagopa-password": "XXX", "pagopa-id-psp": "XXX", "pagopa-id-int-psp": "XXX", "pagopa-id-canale": "XXX", "pagopa-id-canale-pagamento": "XXX", "redis-db-password": "XXX"}* |
|`k8s-pagopa-proxy-prod-secrets` | *a JSON with the pagopa-proxy application secrets used to connect to the PagoPA prod environment: {"pagopa-password": "XXX", "pagopa-id-psp": "XXX", "pagopa-id-int-psp": "XXX", "pagopa-id-canale": "XXX", "pagopa-id-canale-pagamento": "XXX", "redis-db-password": "XXX"}* |
|`k8s-pagopa-proxy-test-secrets-pagopa-ca-chain-certs` | *The full-chain certificates of the PagoPA test endpoint.* |
|`k8s-pagopa-proxy-prod-secrets-pagopa-ca-chain-certs` | *The full-chain certificates of the PagoPA prod endpoint.* |

* The following certificates need to be manually inserted in the Azure Keyvault

|  Certificate  |  Description  |
| ------------- | ------------- |
|`k8s-io-app-backend-secrets-spid-certs` | *The certificates used by IO app backend to authenticate and dialog with official SPID IDPs.* |
|`k8s-io-onboarding-pa-api-secrets-spid-certs` | *The certificates used by the onboarding portal to authenticate and dialog with official SPID IDPs.* |
|`k8s-pagopa-proxy-test-secrets-io-certs` | *The certificate in PFX format for the IO PagoPA test environment endpoint.* |
|`k8s-pagopa-proxy-prod-secrets-io-certs` | *The certificate in PFX format for the IO PagoPA prod environment endpoint.* |

```shell
# Kubernetes
cd ../key_vault_secret_ssh_keys_vm && terragrunt apply
cd ../service_principal_k8s-01 && terragrunt apply
cd ../service_principal_k8s-01-aad-server && terragrunt apply
```

**Manual operations required**

 * Navigate to https://portal.azure.com -> Azure Active Directory -> io-{dev|prod}-sp-k8s-01-aad-server

 * API permissions -> Grant admin consent -> yes

 * Expose an API -> /user_impersonation -> Admins only -> Save

```shell
cd ../service_principal_k8s-01-aad-client && terragrunt apply
```

**Manual operations required**

* Navigate to https://portal.azure.com -> Azure Active Directory -> io-{dev|prod}-sp-k8s-01-aad-client

* API permissions -> Grant admin consent -> yes

```shell
cd ../ad_group_k8s-admin && terragrunt apply
cd ../ad_group_k8s-onboarding && terragrunt apply
cd ../kubernetes_cluster_k8s-01 && terragrunt apply
cd ../public_ip_k8s-01 && terragrunt apply
```

**Manual operations required**

* Update with the group object_id output by the ad_group_k8s-admin live component the value in https://github.com/teamdigitale/io-infrastructure-post-config/blob/master/system/azure-aad-cluster-roles.yaml

* Update with the new IP the loadBalancerIP value in https://github.com/teamdigitale/io-infrastructure-post-config/blob/master/system/nginx-ingress-custom.yaml

```shell
# Public DNS zones and records
cd ../dns_zone_public_dev_io_italia_it && terragrunt apply
cd ../dns_zone_public_dev_io_italia_it_records && terragrunt apply

# Developer portal prerequisites
cd ../service_principal_developer-portal && terragrunt apply
```

**Manual operations required**

The following secrets need to be manually inserted in the Azure Keyvault:

| Secret  | Description    |
| ------- | -------------- |
|`app-insight-web-tests-ocp-apim-subscription-Key` | *The APIM secret used by the monitoring user (for automated monitoring activities). This is usually the user "IO Monitoring" user, listed under users in the APIM.* |

```shell
# Logs, passive and active monitoring
cd ../log_analytics_workspace && terragrunt apply
cd ../monitoring && terragrunt apply
cd ../monitoring_group_01 && terragrunt apply
cd ../app_insights_web_tests && python generate-terraform-tfvars.py && terragrunt apply
```

## All done. What's next?

It's now time to configure the single components you've just deployed:

* Deploy the *Azure functions* code - the following repositories contain the Azure functions code:

    * [io-functions-admin](https://github.com/teamdigitale/io-functions-admin)

    * [io-functions-app](https://github.com/teamdigitale/io-functions-app)

    * [io-functions-services](https://github.com/teamdigitale/io-functions-services)

* Configure *Kubernetes* - the IO Kubernetes helm-charts, configuration files and related docs are available at [https://github.com/teamdigitale/io-infrastructure-post-config](https://github.com/teamdigitale/io-infrastructure-post-config)

## How to contribute

Contributions are welcome! Feel free to open issues and submit a pull request at any time.
