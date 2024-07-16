# Salesforce Data Cloud Vector Database Azure Function App Notifier
Python code for an Azure function that monitors a storage account for changes and calls Salesforce Data Cloud when a new blob is added or removed to tell Salesforce Data Cloud to start indexing the content into the vector database. The `az` CLI commands can be used to configure required resources on Azure- 

## Prerequisites
* `az` CLI

## Configuration
* Create a keypair using the `openssl` commands below
* Create a Connected App in Salesforce uploading the certificate

``` bash
# create a keypair and export public key
openssl genrsa -out ./keypair.pem 4096
openssl req -new -x509 -nodes -sha256 -days 365 -key ./keypair.pem -out ./certificate.crt

# azure configuration (prefix prefixed with x as key vault name must start with a letter)
PREFIX=x`uuidgen | cut -d "-" -f 5 | tr "[:upper:]" "[:lower:]"`
SUBSCRIPTION=`az account show --query id -o tsv`
USER_ID=`az ad signed-in-user show --query id -o tsv`
LOCATION=swedencentral
RESOURCE_GROUP=${PREFIX}-datacloud-vectordb
STORAGE_ACCOUNT=${PREFIX}storageacc
STORAGE_CONTAINER=${PREFIX}-blob-container
TOPIC_NAME=${PREFIX}-eventgrid-systemtopic
KEY_VAULT_NAME=${PREFIX}-keyvault
FUNCTION_APP_NAME=${PREFIX}-functionapp
SUBSCRIPTION_NAME=${PREFIX}-functionapp-storage-event-subscription
LOG_ANALYTICS_WS=${PREFIX}-loganalyticsws
APP_INSIGHT_NAME=${PREFIX}-appinsights

# salesforce configuration
SALESFORCE_LOGIN_URL=https://login.salesforce.com
SALESFORCE_CLIENT_ID="<Salesforce clientId>"
SALESFORCE_USERNAME="<Salesforce username>"

# create resource group
az group create \
	--name $RESOURCE_GROUP \
	--location $LOCATION

# create storage account and a blob container
az storage account create \
	--resource-group $RESOURCE_GROUP \
	--name $STORAGE_ACCOUNT \
	--sku Standard_LRS \
	--location $LOCATION \
	--allow-blob-public-access=false \
	--enable-hierarchical-namespace \
	--kind StorageV2
az role assignment create \
	--role "Storage Blob Data Contributor" \
	--assignee $USER_ID \
	--scope "/subscriptions/$SUBSCRIPTION/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCOUNT"
az storage container create \
	--account-name $STORAGE_ACCOUNT \
	--name $STORAGE_CONTAINER \
	--auth-mode login

# ensure the provider for eventgrid is registered
az provider register --namespace Microsoft.EventGrid

# create system topic
az eventgrid system-topic create \
	--name $TOPIC_NAME \
	--resource-group ${RESOURCE_GROUP} \
	--source /subscriptions/${SUBSCRIPTION}/resourceGroups/${RESOURCE_GROUP}/providers/Microsoft.Storage/storageAccounts/${STORAGE_ACCOUNT} \
	--topic-type microsoft.storage.storageaccounts \
	--location ${LOCATION}

# create a keyvault and allow current user to access it
az provider register -n Microsoft.KeyVault
az keyvault create \
	--name $KEY_VAULT_NAME \
	--resource-group $RESOURCE_GROUP
az role assignment create \
	--assignee $USER_ID \
	--role "Key Vault Administrator" \
	--scope "/subscriptions/${SUBSCRIPTION}/resourcegroups/${RESOURCE_GROUP}/providers/Microsoft.KeyVault/vaults/${KEY_VAULT_NAME}"

# import private key in PEM format as a key
az keyvault key import \
	--vault-name $KEY_VAULT_NAME \
	--name RSA-PRIVATE-KEY \
	--pem-file ./keypair.pem

# set consumer key (clientId) from Salesforce as a secret
az keyvault secret set \
	--vault-name $KEY_VAULT_NAME \
	--name CONSUMER-KEY \
	--value $SALESFORCE_CLIENT_ID

# create application insights for function app
az extension add --name application-insights
az monitor log-analytics workspace create \
	--resource-group $RESOURCE_GROUP \
	--name $LOG_ANALYTICS_WS
az monitor app-insights component create \
	--resource-group $RESOURCE_GROUP \
	--app $APP_INSIGHT_NAME \
	--location $LOCATION \
	--workspace $LOG_ANALYTICS_WS

# create function app
az functionapp create \
	--resource-group ${RESOURCE_GROUP} \
	--name ${FUNCTION_APP_NAME} \
	--app-insights $APP_INSIGHT_NAME \
	--consumption-plan-location ${LOCATION} \
	--runtime python \
	--runtime-version 3.9 \
	--functions-version 4 \
	--os-type linux \
	--assign-identity "[system]" \
	--storage-account ${STORAGE_ACCOUNT} 
az functionapp deployment source config-zip \
	--resource-group $RESOURCE_GROUP \
	--name $FUNCTION_APP_NAME \
	--src ./function_app.zip \
	--build-remote true
az functionapp config appsettings set \
	--name $FUNCTION_APP_NAME \
	--resource-group $RESOURCE_GROUP \
	--settings SF_LOGIN_URL=$SALESFORCE_LOGIN_URL SF_USERNAME=$SALESFORCE_USERNAME KEY_VAULT_NAME=$KEY_VAULT_NAME

# get the userid of the function app and allow that id to be a user of secrets and keys in the key vault
APP_PRINCIPAL_ID="$(az functionapp show --name $FUNCTION_APP_NAME --resource-group $RESOURCE_GROUP --query identity.principalId -o tsv)"
KEYVAULT_SCOPE="/subscriptions/${SUBSCRIPTION}/resourcegroups/${RESOURCE_GROUP}/providers/Microsoft.KeyVault/vaults/${KEY_VAULT_NAME}"
az role assignment create \
	--assignee $APP_PRINCIPAL_ID \
	--role "Key Vault Crypto User" \
	--role "Key Vault Secrets User" \
	--role "Key Vault Reader" \
	--scope $KEYVAULT_SCOPE

# create an event subscription for the function app to events from the blob storage
az eventgrid system-topic event-subscription create \
	--name ${SUBSCRIPTION_NAME} \
	--resource-group ${RESOURCE_GROUP} \
	--system-topic-name $TOPIC_NAME \
	--endpoint /subscriptions/${SUBSCRIPTION}/resourceGroups/${RESOURCE_GROUP}/providers/Microsoft.Web/sites/${FUNCTION_APP_NAME}/functions/eventGridTrigger \
	--endpoint-type azurefunction \
	--event-delivery-schema eventgridschema \
	--included-event-types Microsoft.Storage.BlobCreated Microsoft.Storage.BlobDeleted

```