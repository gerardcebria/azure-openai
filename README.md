# Azure-openai workshop
This workshop will show you how you can create such AI powered assistant that operates on your own business data without leaving your Azure account.

## Contents
* Documents (`data/transformers_docs_full`): We will use [documentation of transformers library](https://huggingface.co/docs/transformers/index) as example.
* Infastructure Code (`infra`): Deploys the necessary resources to Azure.
* Backend (`workshop_oai_qa`): Gradio App that provides a simple UI to interact with the model.

## Prerequisites
* Azure Subscription
* [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)

> If you are following the live hands-on workshop, you will be provided with a free Azure subscription.
> Make sure you send us your email address through [this form](#todo) so we can send you the invitation
> 
> If you are following the workshop on your own, you will need to have an Azure subscription. You can create a free Azure subscription [here](https://azure.microsoft.com/en-us/free/).

## Deploying Infrastructure

### What will be deployed?
* [Azure Open AI Service](https://azure.microsoft.com/en-us/products/ai-services/openai-service)
  * gpt-3.5-turbo-16k: Chat model
  * text-embedding-ada-002: Embeddings model
* [Azure Cognitive Search](https://azure.microsoft.com/en-us/services/search/)
* [Azure App Service](https://azure.microsoft.com/en-us/services/app-service/web/)

### Before you start
Write down the following variables:

**Switch to your project subscription ID**:
```bash
az account set --subscription <subscriptionId>
```

**Find your Principal ID**:
This will be the user that indexes the documents. In this case, we assume it's run locally, thus we use the signed in user.

```bash
az ad signed-in-user show --output=json | jq ".objectId"
```

**Save your environment ID**: To avoid collision with other participants, we will use the environment name as a suffix for the resource names. You can use your name or any other unique identifier.
For example your initials and a random number: `ed42`

### Create AD App Registration
```bash
az ad app create \
--display-name workshop-oai-<environmentId> \
--available-to-other-tenants false \
--query appId
```

Write down the `appId`.

### Deploying Infrastructure Resources

```bash
PROJECT=workshop-oai \
AZURE_SUBSCRIPTION_ID=<subscriptionId> \
AZURE_ENV_NAME=<environmentName> \
AZURE_LOCATION=eastus \
AZURE_PRINCIPAL_ID=<principalId> \
AZURE_AUTH_CLIENT_ID=<appId> \
bash scripts/deploy_infrastructure.sh
```

Write down the `resourceGroup` name.

### Get Infrastructure Outputs
Run the following commands to get the outputs of the infrastructure deployment. 
Paste them into `.env` file created from .env.example.

```bash
# Backend App URL:
az deployment group show -g <resourceGroup> -n web --query properties.outputs.uri.value

# Backend App Name:
az deployment group show -g <resourceGroup> -n web --query properties.outputs.name.value

# OpenAI Service Endpoint:
az deployment group show -g <resourceGroup> -n openai --query properties.outputs.endpoint.value

# OpenAI Service Key:
az deployment group show -g <resourceGroup> -n openai --query properties.outputs.key.value

# Cognitive Search Endpoint:
az deployment group show -g <resourceGroup> -n search-service --query properties.outputs.endpoint.value

# Cognitive Search API Key:
az deployment group show -g <resourceGroup> -n search-service --query properties.outputs.adminKey.value
```

### Update AD App Registration with the Backend App Url
Update app registration to allow the backend app to authenticate with Azure AD.

```bash
az ad app create \
--display-name workshop-oai-<environmentId> \
--available-to-other-tenants false \
--reply-urls <backendAppUrl>
```

### Deploying the App Code
Deploy the app code to Azure App Service.

```bash
az webapp up \
--runtime PYTHON:3.10 \
--sku B1 \
--name <appBackendName> \
--resource-group <resourceGroup> \
--subscription <subscriptionId>
```

## Index the documents
First install the dependencies:
```bash
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

Index the documents:
```bash
python scripts/index_documents.py
```

## Test the app
Open the app url in the browser and ask a question about transformers library.