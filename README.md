# GithubActions: How to create .NET 8 Web API Docker image and Upload it to Azure Container Registry (ACR)

## 1. Create a .NET 8 Web API in Visual Studio Community Edition

Do not forget to Enable Docker support for automatically create the **Dockerfile** when creating the application

**Dockerfile**

```dockerfile
#See https://aka.ms/customizecontainer to learn how to customize your debug container and how Visual Studio uses this Dockerfile to build your images for faster debugging.

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
USER app
WORKDIR /app
EXPOSE 8080
EXPOSE 8081

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src
COPY ["WebApplication1.csproj", "."]
RUN dotnet restore "./././WebApplication1.csproj"
COPY . .
WORKDIR "/src/."
RUN dotnet build "./WebApplication1.csproj" -c $BUILD_CONFIGURATION -o /app/build

FROM build AS publish
ARG BUILD_CONFIGURATION=Release
RUN dotnet publish "./WebApplication1.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "WebApplication1.dll"]
```

Do not forget to add the Kubernetes manifest files in your application: **deployment.yml** and **service.yml** 

**deployment.yml**

```yaml
  selector:
    matchLabels:
      app: webapplication1
  template:
    metadata:
      labels:
        app: webapplication1
    spec:
      containers:
      - name: webapplication1
        image: mycontainerazure1974.azurecr.io/webapplication1:latest  # Replace with your image path
        ports:
        - containerPort: 8080
```

**sevice.yml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapplication1-service
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: webapplication1
```

## 2. Create a Github repository in Visual Studio 2022 an upload the .NET 8 Web API source code


## 3. Create Azure Container Registry ACR service for storing your Docker image

First, with Azure CLI, we create a new ResourceGroup in France Central region 

```
az group create --name myRG --location francecentral
```

Then we create the new Azure Container Registry ACR named "mycontainerazure1974"
```
az acr create --name mycontainerazure1974 --resource-group myRG --sku Basic --location francecentral
```

We set Admin User in our new Azure ACR

![image](https://github.com/luiscoco/GithubActions_Create_DockerImage_Upload_to_Azure_ACR_dotNET8WebAPI/assets/32194879/3f050394-d530-4cac-a5c1-dda0867d115a)

We copy the username and the password to store both values in Gihub repository secrets.

## 4. Create the Github repository secrets

We nagivate to the Setting menu option in our Github repository

We select Settings->Secrets and variables->Actions->New repository secret and we create the secrets for storing the Azure ACR username and password

![image](https://github.com/luiscoco/GithubActions_Create_DockerImage_Upload_to_Azure_ACR_dotNET8WebAPI/assets/32194879/61703f61-08b0-4249-8dc4-1af251b797a4)

![image](https://github.com/luiscoco/GithubActions_Create_DockerImage_Upload_to_Azure_ACR_dotNET8WebAPI/assets/32194879/4d5ec931-4914-447a-acc2-44e4f7b470fe)

![image](https://github.com/luiscoco/GithubActions_Create_DockerImage_Upload_to_Azure_ACR_dotNET8WebAPI/assets/32194879/f572a38e-f267-48f1-99bf-b9ac2950dbae)

These are the two secrets stored in Github

![image](https://github.com/luiscoco/GithubActions_Create_DockerImage_Upload_to_Azure_ACR_dotNET8WebAPI/assets/32194879/b7a88367-755e-4d98-a428-fb6ccb1a00e2)

## 4. Create the Github Actions Workflow

We input the workflow **main.yml** file source code 

```yaml
name: Build and Push Docker Image

on:
  push:
    branches: [ master ]  # or any other branch you want to trigger on
  pull_request:
    branches: [ master ]

env:
  REGISTRY: mycontainerazure1974.azurecr.io  # Your Azure Container Registry
  IMAGE_NAME: webapplication1  # Replace with your image name

jobs:
  build_and_push:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v1

    - name: Login to Azure Container Registry
      uses: docker/login-action@v1
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ secrets.AZURE_REGISTRY_USERNAME }}  # Set in GitHub secrets
        password: ${{ secrets.AZURE_REGISTRY_PASSWORD }}  # Set in GitHub secrets

    - name: Build and Push Image
      uses: docker/build-push-action@v2
      with:
        context: .
        file: ./Dockerfile  # Path to your Dockerfile
        push: true
        tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest

    # Additional steps for deployment or other actions can be added here.
```

![image](https://github.com/luiscoco/GithubActions_Create_DockerImage_Upload_to_Azure_ACR_dotNET8WebAPI/assets/32194879/7e0ef95a-f1c0-4619-8da1-336277632607)

We verify the workflow is running ok

![image](https://github.com/luiscoco/GithubActions_Create_DockerImage_Upload_to_Azure_ACR_dotNET8WebAPI/assets/32194879/0b31b30f-a818-4868-9508-639f923b2fdf)


## 5. Verify in Azure Portal the docker image uploaded to Azure ACR

We log in Azure Portal and navigate to our new Azure ACR.

We select the Metrics option and we confirm we uploaded the docker image a few seconds ago

![image](https://github.com/luiscoco/GithubActions_Create_DockerImage_Upload_to_Azure_ACR_dotNET8WebAPI/assets/32194879/4342fd4c-a2ed-4891-a72b-987b95b78a67)

## 6. Deploy your application docker image in Azure Kubernetes AKS

We create a new Azure Kubernetes Cluster AKS with the following Azure CLI command:

``
az aks create --resource-group myRG --name mydotnet8webapiakscluster --location francecentral --node-count 1 --generate-ssh-keys
``

Then we attach my Container Registry ACR (called "mycontainerazure1974") to my Kubernetes Cluster AKS (called "mydotnet8webapiakscluster") 

```
az aks update -n mydotnet8webapiakscluster -g myRG --attach-acr mycontainerazure1974
```

In I did not find any permission proble running the "az aks update" command.

If you don't have the necessary permissions, you might need to ask your Azure administrator to grant you the required roles. 

For instance, being assigned the "**Contributor**" role at the resource group or resource level would typically suffice for these operations.

We go to Visual Studio and we right click on the project and select the option **Open in Terminal**

![image](https://github.com/luiscoco/GithubActions_Create_DockerImage_Upload_to_Azure_ACR_dotNET8WebAPI/assets/32194879/78743392-8971-4bdb-93c6-61581a8d3d32)

In the Terminal window we confirm we are in the folder where are located the Kubernetes manifest files (deployment.yml and service.yml)

![image](https://github.com/luiscoco/GithubActions_Create_DockerImage_Upload_to_Azure_ACR_dotNET8WebAPI/assets/32194879/801364dc-aa3f-4b3e-8f94-642d61f68067)

Then we open in another window Azure Portal and navigate to Azure AKS service and select the **Overview->Connect** menu option

![image](https://github.com/luiscoco/GithubActions_Create_DockerImage_Upload_to_Azure_ACR_dotNET8WebAPI/assets/32194879/a71c04df-ad5a-4c07-9cba-d84ce8fe08d3)

Then we copy the commands and login in our Azure AKS:

```
az account set --subscription XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

```
az aks get-credentials --resource-group myRG --name mydotnet8webapiakscluster
```

![image](https://github.com/luiscoco/GithubActions_Create_DockerImage_Upload_to_Azure_ACR_dotNET8WebAPI/assets/32194879/11179677-6898-4c4d-b071-d7e6c9d6fe62)

Now we apply the deployment to our AKS Cluster 

```
kubectl apply -f deployment.yml
```

```
kubectl apply -f service.yml
```

![image](https://github.com/luiscoco/GithubActions_Create_DockerImage_Upload_to_Azure_ACR_dotNET8WebAPI/assets/32194879/4e1e3ee7-dbe2-4a9f-8c70-4939c74adfc8)

We can see the Load Balancer is exposing the port 80 and the **external IP address is 20.199.6.236**

![image](https://github.com/luiscoco/GithubActions_Create_DockerImage_Upload_to_Azure_ACR_dotNET8WebAPI/assets/32194879/d9716635-f06a-4c77-ad32-9ca5d70ed1c8)

We can also confirm in Azure Portal the Load Balancer external IP

Navigate to the AKS and select the menu option **Services and Ingresses**  

![image](https://github.com/luiscoco/GithubActions_Create_DockerImage_Upload_to_Azure_ACR_dotNET8WebAPI/assets/32194879/37a605bf-c4df-4523-92f1-70fa1939ab4d)

## 7. Automate the deployment of your Docker container to an Azure Kubernetes Service (AKS) cluster using GitHub Actions

We have to add the following source code to our **main.yml** file given in section 4. in this document:

```yaml
- name: Install Azure CLI
      uses: azure/CLI@v1

    - name: Get AKS Credentials
      run: |
        az aks get-credentials --resource-group ${{ env.RESOURCE_GROUP }} --name ${{ env.AKS_CLUSTER_NAME }} --overwrite-existing

    - name: Deploy to AKS
      run: |
        kubectl apply -f deployment.yml
        kubectl apply -f service.yml
```

The resulting main.yml file would be:

```yaml
name: Build and Push Docker Image

on:
  push:
    branches: [ master ]
  pull_request:
    branches: [ master ]

env:
  REGISTRY: mycontainerazure1974.azurecr.io
  IMAGE_NAME: webapplication1
  RESOURCE_GROUP: myRG  # Replace with your Azure Resource Group name
  AKS_CLUSTER_NAME: mydotnet8webapiakscluster  # Replace with your AKS Cluster name

jobs:
  build_and_push:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v1

    - name: Login to Azure Container Registry
      uses: docker/login-action@v1
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ secrets.AZURE_REGISTRY_USERNAME }}
        password: ${{ secrets.AZURE_REGISTRY_PASSWORD }}

    - name: Build and Push Image
      uses: docker/build-push-action@v2
      with:
        context: .
        file: ./Dockerfile
        push: true
        tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest

    - name: Install Azure CLI
      uses: azure/CLI@v1

    - name: Get AKS Credentials
      run: |
        az aks get-credentials --resource-group ${{ env.RESOURCE_GROUP }} --name ${{ env.AKS_CLUSTER_NAME }} --overwrite-existing

    - name: Deploy to AKS
      run: |
        kubectl apply -f deployment.yml
        kubectl apply -f service.yml
```







