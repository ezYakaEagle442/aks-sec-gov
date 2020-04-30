# Azure Arc

See also :
- [https://azure.microsoft.com/en-us/resources/videos/kubernetes-app-management-with-azure-arc](https://azure.microsoft.com/en-us/resources/videos/kubernetes-app-management-with-azure-arc)
- []()
- []()
- []()

```sh
# https://github.com/Azure/azure-arc-kubernetes-preview



```

##  Enable resource providers
See :
- [https://github.com/Azure/azure-arc-kubernetes-preview/blob/master/docs/enable-providers.md](https://github.com/Azure/azure-arc-kubernetes-preview/blob/master/docs/enable-providers.md)

```sh

az feature register --namespace Microsoft.Kubernetes --name previewAccess
az feature register --namespace Microsoft.KubernetesConfiguration --name sourceControlConfiguration

az feature list -o table | grep Kubernetes

az provider register --namespace Microsoft.Kubernetes
az provider register --namespace Microsoft.KubernetesConfiguration
az provider show -n Microsoft.Kubernetes -o table
az provider show -n Microsoft.KubernetesConfiguration -o table

```

## Install CLI extension

See :
- [https://github.com/Azure/azure-arc-kubernetes-preview/blob/master/docs/install-cli-extension.md](https://github.com/Azure/azure-arc-kubernetes-preview/blob/master/docs/install-cli-extension.md)

```sh
az extension add --source ./extensions/connectedk8s-0.1.3-py2.py3-none-any.whl --yes
az extension add --source ./extensions/k8sconfiguration-0.1.6-py2.py3-none-any.whl --yes
az extension list -o table

```

## Create a test-cluster using the aks-engine
See also:
- [https://github.com/Azure/azure-arc-kubernetes-preview/blob/master/docs/create-cluster-with-AKS-engine.md#create-a-test-cluster-using-the-aks-engine](https://github.com/Azure/azure-arc-kubernetes-preview/blob/master/docs/create-cluster-with-AKS-engine.md#create-a-test-cluster-using-the-aks-engine)

```sh

# Set the Azure location for the cluster. Supported locations during preview are (eastus, westeurope).
arc_location="eastus"
echo "Azure Arc location " $arc_location

azurearc_appname="azurearc-${appName}"
echo "Azure Arc App. name " $azurearc_appname

# Set the name for the resource group that will be created by the AKS Engine. The name must be unique within the subscription.
azurearc_rg_name="rg-${azurearc_appname}-${arc_location}"
echo "Azure Arc RG name " $azurearc_rg_name

dnsPrefix="${azurearc_appname}-${arc_location}"
echo "Azure Arc DNS prefix " $dnsPrefix

spnAppPassword=$(az ad sp create-for-rbac --name $azurearc_appname --role contributor --query password --output tsv)
echo "Azure Arc Service Principal Secret " $spnAppPassword 

spnAppId=$(az ad sp list --show-mine --query "[?appDisplayName=='${azurearc_appname}'].{appId:appId}" --output tsv)
echo "Azure Arc Service Principal ID:" $spnAppId  



# Set the local absolute file path of the API model (download this from https://github.com/Azure/aks-engine/blob/master/examples/kubernetes.json)
# For example, "C:\Users\<you>\aks-engine-v0.48.0-windows-amd64\kubernetes.json"
apimodel="./cnf/azure-arc-api-model.json"
echo "API Model file location " $apimodel

# You can edit this file to modify the Kubernetes version size of VMs and the number of worker nodes (see example file below).

# Create the cluster
aks-engine deploy --subscription-id $subId -g $azurearc_rg_name --client-id $spnAppId  --client-secret $spnAppPassword --dns-prefix $dnsPrefix --location $arc_location --api-model $apimodel --force-overwrite

$env:KUBECONFIG=~/.kubeconfig
kubectl config view
kubectl cluster-info


```

```sh

```