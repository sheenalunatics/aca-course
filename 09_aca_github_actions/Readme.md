# Creating a CI/CD pipeline for Azure Container Apps

A lab for showing how to create CI/CD pipelines using Github Actions to deploy an app into Azure Container Apps.

![](images/architecture.png)

Create a Container Apps

```powershell
$RESOURCE_GROUP="rg-containerapps-github-actions"
$LOCATION="southeastasia"
$CONTAINERAPPS_ENVIRONMENT="managedEnvironment-rgcontainerapps-be89"
$CONTAINERAPPS_APP="album-backend-api"

az group create `
         --name $RESOURCE_GROUP `
         --location $LOCATION

az containerapp env create `
                --name $CONTAINERAPPS_ENVIRONMENT `
                --resource-group $RESOURCE_GROUP `
                --location $LOCATION

az containerapp create `
                --name $CONTAINERAPPS_APP `
                --resource-group $RESOURCE_GROUP `
                --environment $CONTAINERAPPS_ENVIRONMENT `
                --image mcr.microsoft.com/azuredocs/containerapps-helloworld:latest `
                --target-port 80 `
                --ingress 'external'

az containerapp show `
                --name $CONTAINERAPPS_APP `
                --resource-group $RESOURCE_GROUP `
                --query properties.configuration.ingress.fqdn
```

We should have these resources created.

![](images/resources.png)

Create an Azure Service Principal (SPN) to be used by Github Actions to authenticate to Azure.

```powershell
$SUBSCRIPTION_ID=$(az account list --query [?isDefault].id -o tsv)

echo $SUBSCRIPTION_ID
# e987901a-710f-4edd-b4c6-7c98572d04d5

az ad sp create-for-rbac -n "spn-aca-github" --role Contributor --scope /subscriptions/$SUBSCRIPTION_ID --sdk-auth
# {
#  "clientId": "a174a817-bd80-4978-99b6-b799cf450707",
#  "clientSecret": "Fp48Q~qOk8N6k.X3W_D3zRlXmw2eEO3IPO94ZbPm",
#  "subscriptionId": "e987901a-710f-4edd-b4c6-7c98572d04d5",
#  "tenantId": "b96cc57b-d146-48f5-a381-7cf474c23a9e",
#  "activeDirectoryEndpointUrl": "https://login.microsoftonline.com",
#  "resourceManagerEndpointUrl": "https://management.azure.com/",
#  "activeDirectoryGraphResourceId": "https://graph.windows.net/",
#  "sqlManagementEndpointUrl": "https://management.core.windows.net:8443/",
#  "galleryEndpointUrl": "https://gallery.azure.com/",
#  "managementEndpointUrl": "https://management.core.windows.net/"
#}

```

In Github, create new secret named `AZURE_CREDENTIALS` and save the SPN.

![](images/gh-secret-az-creds.png)

Then create a new Github workflow using the following YAML. Make sure to change the variables accordingly.

```yaml
# src: .github/workflows/
name: Build and deploy a container to an Azure Container Apps - ACA

env:
  IMAGE_NAME: houssemdellai/containerapps-album-api
  CONTAINERAPPS_APP: album-backend-api
  CONTAINERAPPS_ENVIRONMENT: aca-environment
  RESOURCE_GROUP: rg-containerapps-github-actions

on:
  push:
    branches: [ "main" ]
    paths:
      - '.github/workflows/**'
  workflow_dispatch:

permissions:
  contents: read
  packages: write

jobs:
  build-push-container-image:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v1

      - name: Log in to GitHub container registry
        uses: docker/login-action@v1.10.0
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ github.token }}

      - name: Build and push container image to registry
        uses: docker/build-push-action@v2
        with:
          push: true
          tags: ghcr.io/${{ env.IMAGE_NAME }}:${{ github.run_number }}
          file: ./backend_api/backend_api_csharp/Dockerfile
          context: ./backend_api/backend_api_csharp

  deploy-to-container-apps:
    permissions:
      contents: none
    runs-on: ubuntu-latest
    needs: build-push-container-image

    steps:

    - uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}

    - name: Build and deploy Container App
      uses: azure/container-apps-deploy-action@v1
      with:
        registryUrl: ghcr.io
        imageToDeploy: ghcr.io/${{ env.IMAGE_NAME }}:${{ github.run_number }}
        targetPort: 3500
        ingress: external
        containerAppName: ${{ env.CONTAINERAPPS_APP }}
        containerAppEnvironment: ${{ env.CONTAINERAPPS_ENVIRONMENT }}
        resourceGroup: ${{ env.RESOURCE_GROUP }}
```

By pushing the workflow file to Github, a new action will start running.

![](images/github-actions-run.png)

Check the created container image.

![](images/ghcr-image.png)

And check the deployed Container Apps using a new revision.

![](images/aca-revision.png)

ANd check the web app running.

![](images/webapp.png)

## More resources

https://github.com/marketplace/actions/azure-container-apps-build-and-deploy
