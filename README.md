# Deploy App Service in Kubernetes Cluster

How to deploy App Services (Web App , Function App , Logic App ) in Kubernetes Cluster (Azure Arc Enabled)

### Create a Kubernetes Cluster

create a Kubernetes Cluster and configure it to be an Azure Arc enabled Cluster

- aksResourceGroupName="rb-aks-rg"
- aksClusterName="rb-aks"

![image](https://user-images.githubusercontent.com/6815990/155522008-858622ea-8b0c-4279-bd64-6bfb202df3a9.png)

![image](https://user-images.githubusercontent.com/6815990/155522048-1597c223-040a-43bc-89f0-08562d81974d.png)

Connect to your cluster using command line tooling to interact directly with cluster using kubectl, the command line tool for Kubernetes. Kubectl is available within the Azure Cloud Shell by default and can also be installed locally.

![image](https://user-images.githubusercontent.com/6815990/155532930-9a0eb946-acd1-4e6e-88e3-fed2e6f45303.png)

You’ll also need to make sure that you have a number of Microsoft Resource Endpoints enabled -

```
az provider register --namespace Microsoft.ExtendedLocation --wait
az provider register --namespace Microsoft.Web --wait
az provider register --namespace Microsoft.KubernetesConfiguration --wait
```

### Setting up Azure Arc

Create resource group for Azure arc enabled Kubernetes

```
arcResourceGroupName="rb-arc-rg"
arcResourceGroupLocation="westeurope"
az group create --name $arcResourceGroupName --location $arcResourceGroupLocation --output table
```
Create Azure arc enabled Kubernetes

```
arcClusterName="rb-arc-aks"
az connectedk8s connect --name $arcClusterName --resource-group $arcResourceGroupName
```

![image](https://user-images.githubusercontent.com/6815990/155538643-7e5b983e-5a38-4fe4-9ca8-26c56cc9354d.png)



# Creating a Log Analytics Workspace
