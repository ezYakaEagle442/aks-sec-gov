
See also :
- [https://github.com/Azure/secrets-store-csi-driver-provider-azure](https://github.com/Azure/secrets-store-csi-driver-provider-azure)
- [https://docs.microsoft.com/en-us/azure/aks/developer-best-practices-pod-security#limit-credential-exposure](https://docs.microsoft.com/en-us/azure/aks/developer-best-practices-pod-security#limit-credential-exposure)

# Install the Secrets Store CSI Driver and the Azure Keyvault Provider

[Installing the Chart](https://github.com/Azure/secrets-store-csi-driver-provider-azure/blob/master/charts/csi-secrets-store-provider-azure/README.md)
```sh
helm install csi-secrets-store-provider-azure/csi-secrets-store-provider-azure --generate-name -n $target_namespace
helm ls -n $target_namespace 

```

# Create key-Vault
```sh
az group create --name $rg_kv_name --location $location
az keyvault create --name $vault_name --enable-soft-delete true --location $location -g $rg_kv_name
az keyvault show --name $vault_name 
# az keyvault update --name $vault_name --default-action deny -g $rg_kv_name 

kv_id=$(az keyvault show --name $vault_name -g $rg_kv_name --query "id" --output tsv)
echo "KeyVault ID :" $kv_id
nslookup $vault_name.vault.azure.net

# https://docs.microsoft.com/en-us/cli/azure/keyvault/secret?view=azure-cli-latest#az-keyvault-secret-set
az keyvault secret set --name $vault_secret_name --value $vault_secret --description "CSI driver ${appName} Secret" --vault-name $vault_name
az keyvault secret list --vault-name $vault_name
az keyvault secret show --vault-name $vault_name --name $vault_secret_name --output tsv

```

# Create an Azure User Identity


```sh
export SUBSCRIPTION_ID=$subId
export RESOURCE_GROUP=$rg_name
export IDENTITY_NAME="${appName}-csi-demo-pod-identity" #  must consist of lower case 

# https://github.com/Azure/aad-pod-identity/pull/48
# https://github.com/Azure/aad-pod-identity/issues/38
az identity create -g $rg_name -n $IDENTITY_NAME
export IDENTITY_CLIENT_ID="$(az identity show -g $rg_name -n $IDENTITY_NAME --query clientId -otsv)"
export IDENTITY_RESOURCE_ID="$(az identity show -g $rg_name -n $IDENTITY_NAME --query id -otsv)"
export IDENTITY_ASSIGNMENT_ID="$(az role assignment create --role Reader --assignee $IDENTITY_CLIENT_ID --scope /subscriptions/$subId/resourceGroups/$rg_name --query id -o tsv)"

az role assignment create --role Reader --assignee $IDENTITY_CLIENT_ID --scope $kv_id

# set policy to access keys in your keyvault
az keyvault set-policy -n $vault_name --key-permissions get --spn $IDENTITY_CLIENT_ID
# set policy to access secrets in your keyvault
az keyvault set-policy -n $vault_name --secret-permissions get --spn $IDENTITY_CLIENT_ID
# set policy to access certs in your keyvault
az keyvault set-policy -n $vault_name --certificate-permissions get --spn $IDENTITY_CLIENT_ID

export TENANT_ID=$tenantId
export KV_NAME=$vault_name
export SECRET_NAME=$vault_secret_name
export POD_ID=xxxxxx

envsubst < ./cnf/secrets-store-csi-provider-class.yaml > deploy/secrets-store-csi-provider-class.yaml
cat deploy/secrets-store-csi-provider-class.yaml
k apply -f deploy/secrets-store-csi-provider-class.yaml -n $target_namespace
k get secretproviderclasses -n $target_namespace
k describe secretproviderclasses azure-kv-vsegov-xxxxxx -n $target_namespace

envsubst < ./cnf/csi-demo-pod-identity.yaml > deploy/csi-demo-pod-identity.yaml
cat deploy/csi-demo-pod-identity.yaml

k apply -f deploy/csi-demo-pod-identity.yaml -n $target_namespace
k get azureidentity -A
k get azureidentitybindings -A
k get azureassignedidentities -A

envsubst < ./cnf/secrets-store-csi-demo-pod.yaml > deploy/secrets-store-csi-demo-pod.yaml
cat deploy/secrets-store-csi-demo-pod.yaml
k apply -f deploy/secrets-store-csi-demo-pod.yaml -n $target_namespace
k get po -n $target_namespace
k describe pod nginx-secrets-store-inline -n $target_namespace
k logs nginx-secrets-store-inline -n $target_namespace
k get events -n $target_namespace | grep -i "Error" 

```

# Test

```sh
k exec -it nginx-secrets-store-inline-podid ls /mnt/secrets-store/
```