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

```
appsvcWorkspaceName="$arcClusterName-appsvc-logs"

az monitor log-analytics workspace create --resource-group $arcResourceGroupName --workspace-name $appsvcWorkspaceName

logAnalyticsWorkspaceId=$(az monitor log-analytics workspace show --resource-group $arcResourceGroupName --workspace-name $appsvcWorkspaceName --query customerId --output tsv)
logAnalyticsWorkspaceIdEnc=$(printf %s $logAnalyticsWorkspaceId | base64) # Needed for the next step
logAnalyticsKey=$(az monitor log-analytics workspace get-shared-keys --resource-group $arcResourceGroupName --workspace-name $appsvcWorkspaceName --query primarySharedKey --output tsv)
logAnalyticsKeyEncWithSpace=$(printf %s $logAnalyticsKey | base64)
logAnalyticsKeyEnc=$(echo -n "${logAnalyticsKeyEncWithSpace//[[:space:]]/}") # Needed for the next step

```
### Configuration

```
aksResourceGroupName="rb-aks-rg"
aksClusterName="rb-aks"
appsvcPipName="$aksClusterName-appsvc-pip"

# The below is not needed if you are not using AKS. You will need to populate this in another way, it is effectively being used to obtain a static IP to be used by the App Service on Kubernetes deployment.
aksComponentsResourceGroupName=$(az aks show --resource-group $aksResourceGroupName --name $aksClusterName --output tsv --query nodeResourceGroup)

# Create a Public IP address to be used as the Public Static IP for our App Service Kubernetes environment. We'll assign that to a variable called staticIp.
az network public-ip create --resource-group $aksComponentsResourceGroupName --name $appsvcPipName --sku STANDARD

```

### Install the App Service Extension to your Azure Arc enabled Kubernetes cluster

```
extensionName="$arcClusterName-appsvc" # Name of the App Service extension
namespace="appservice" # Namespace in your cluster to install the extension and provision resources
kubeEnvironmentName="$arcClusterName-appsvc" # Name of the App Service Kubernetes environment resource


# Explain that this step configures the extension for Azure Arc.
az k8s-extension create --resource-group $arcResourceGroupName --name $extensionName --cluster-type connectedClusters --cluster-name $arcClusterName --extension-type 'Microsoft.Web.Appservice' --release-train stable --auto-upgrade-minor-version true --scope cluster --release-namespace $namespace --configuration-settings "Microsoft.CustomLocation.ServiceAccount=default" --configuration-settings "appsNamespace=${namespace}" --configuration-settings "clusterName=${kubeEnvironmentName}" --configuration-settings "loadBalancerIp=${staticIp}"     --configuration-settings "keda.enabled=true" --configuration-settings "buildService.storageClassName=default" --configuration-settings "buildService.storageAccessMode=ReadWriteOnce" --configuration-settings "customConfigMap=${namespace}/kube-environment-config" --configuration-settings "envoy.annotations.service.beta.kubernetes.io/azure-load-balancer-resource-group=${aksResourceGroupName}" --configuration-settings "logProcessor.appLogs.destination=log-analytics" --configuration-protected-settings "logProcessor.appLogs.logAnalyticsConfig.customerId=${logAnalyticsWorkspaceIdEnc}" --configuration-protected-settings "logProcessor.appLogs.logAnalyticsConfig.sharedKey=${logAnalyticsKeyEnc}"

extensionId=$(az k8s-extension show --cluster-type connectedClusters --cluster-name $arcClusterName --resource-group $arcResourceGroupName --name $extensionName --query id --output tsv)


```

![image](https://user-images.githubusercontent.com/6815990/155550583-e3bace8e-b285-4197-ac18-fcc6a74f6629.png)



Create a Custom Location

```
extensionId=$(az k8s-extension show --cluster-type connectedClusters --cluster-name $arcClusterName --resource-group $arcResourceGroupName --name $extensionName --query id --output tsv)

customLocationName="$arcClusterName-appsvc" # Name of the custom location

# Obtain the Azure Arc enabled Kubernetes Cluster Resource ID. We'll need this for later Azure CLI commands.
connectedClusterId=$(az connectedk8s show --resource-group $arcResourceGroupName --name $arcClusterName --query id --output tsv)

# Now create a custom location based upon the information we've been gathering over the course of this post
az customlocation create --resource-group $arcResourceGroupName --name $customLocationName --host-resource-id $connectedClusterId --namespace $namespace --cluster-extension-ids $extensionId

# The above resource should be created quite quickly. We'll need the Custom Location Resource ID for a later step, so let's go 
# ahead and assign it to a variable.
customLocationId=$(az customlocation show --resource-group $arcResourceGroupName --name $customLocationName --query id --output tsv)

# Let's double check that the variable was appropriately assigned and isn't empty.
echo $customLocationId
```

### Create the App Service Kubernetes environment

we now need to create an App Service Kubernetes Environment to map the Custom Location and The Static IP that we setup earlier
