# Use Azure role-based access controls to define access to the Kubernetes configuration file in AKS

```sh
# https://docs.microsoft.com/en-us/azure/aks/control-kubeconfig-access# Get the resource ID of your AKS cluster
# https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/

AKS_CLUSTER=$(az aks show --resource-group $rg_name --name $cluster_name --query id -o tsv)

# Get the account credentials for the logged in user
ACCOUNT_UPN=$(az account show --query user.name -o tsv)
ACCOUNT_ID=$(az ad user show --id $ACCOUNT_UPN --query objectId -o tsv)

# Assign the 'Cluster Admin' role to the user
az role assignment create \
    --assignee $ACCOUNT_ID \
    --scope $AKS_CLUSTER \
    --role "Azure Kubernetes Service Cluster Admin Role"

az aks get-credentials -g $rg_name --name $cluster_name --admin
kubectl config view
az role assignment delete --assignee $ACCOUNT_ID --scope $cluster_name

```
