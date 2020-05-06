# Setup Key-Vault

Generate & save nodes [SSH keys](https://docs.microsoft.com/en-us/azure/aks/ssh) to Azure Key-Vault is a [Best-practice](https://github.com/Azure/k8s-best-practices/blob/master/Security_securing_a_cluster.md#securing-host-access)


See also :
- [https://docs.microsoft.com/en-us/azure/aks/developer-best-practices-pod-security#use-pod-managed-identities](https://docs.microsoft.com/en-us/azure/aks/developer-best-practices-pod-security#use-pod-managed-identities)
- [https://docs.microsoft.com/en-us/azure/key-vault/key-vault-soft-delete-cli](https://docs.microsoft.com/en-us/azure/key-vault/key-vault-soft-delete-cli)
- [https://docs.microsoft.com/en-us/azure/key-vault/private-link-service#establish-a-private-link-connection-to-key-vault-using-cli](https://docs.microsoft.com/en-us/azure/key-vault/private-link-service#establish-a-private-link-connection-to-key-vault-using-cli)
- The KeyVault FlexVol solution is now deprecated in favor of [https://github.com/kubernetes-sigs/secrets-store-csi-driver](https://github.com/kubernetes-sigs/secrets-store-csi-driver)
- [https://github.com/Azure/secrets-store-csi-driver-provider-azure](https://github.com/Azure/secrets-store-csi-driver-provider-azure)


```sh

az provider register -n Microsoft.KeyVault

# https://docs.microsoft.com/en-us/azure/key-vault/key-vault-soft-delete-cli
# az keyvault list-deleted
# az keyvault purge --name $vault_name --location $location -g $rg_kv_name

az keyvault create --name $vault_name --enable-soft-delete true --location $location -g $rg_kv_name
az keyvault show --name $vault_name 
az keyvault update --name $vault_name --default-action deny -g $rg_kv_name 

kv_id=$(az keyvault show --name $vault_name -g $rg_kv_name --query "id" --output tsv)
echo "KeyVault ID :" $kv_id

az network private-endpoint create --name $kv_private_endpoint_name --subnet $kv_subnet_id --manual-request --private-connection-resource-id $kv_id --group-ids vault --connection-name $kv_private_endpoint_svc_con_name --location $location -g $rg_kv_name

az network private-endpoint show --name $kv_private_endpoint_name -g $rg_kv_name
az keyvault private-endpoint-connection approve -n $kv_private_endpoint_svc_con_name \
-g $rg_kv_name --vault-name $vault_name --description "${appName} CLI approval"

# Validate private link connection
az keyvault private-endpoint-connection show --name $kv_private_endpoint_svc_con_name --vault-name $vault_name  -g $rg_kv_name


kv_private_endpoint_id=$(az network private-endpoint show --name $kv_private_endpoint_name -g $rg_kv_name --query id -o tsv)
echo "ACR private-endpoint ID :" $kv_private_endpoint_id

network_interface_id=$(az network private-endpoint show --name $kv_private_endpoint_name -g $rg_kv_name --query 'networkInterfaces[0].id' -o tsv)
echo "KeyVault Network Interface ID :" $network_interface_id

kv_network_interface_private_ip=$(az resource show --ids $network_interface_id \
  --api-version 2019-04-01 --query 'properties.ipConfigurations[0].properties.privateIPAddress' \
  --output tsv)
echo "KeyVault Network Interface private IP :" $kv_network_interface_private_ip

```

## Setup DNS

```sh

az network private-dns zone create --name privatelink.vaultcore.azure.net -g $rg_kv_name 
az network private-dns link vnet create --name $kv_private_dns_link_name --zone-name privatelink.vaultcore.azure.net --registration-enabled true --virtual-network $kv_vnet_name -g $rg_kv_name

private_dns_link_id=$(az network private-dns link vnet show --name $kv_private_dns_link_name --zone-name "privatelink.vaultcore.azure.net" -g $rg_kv_name --query "id" --output tsv)
echo "Private-Link DNS ID :" $private_dns_link_id

# From home/public network, you wil get a public IP. If inside a vnet with private zone, then nslookup will resolve to the private ip.
# note: we will have a feature roll out to let you disable the public access, which means “nslookup” will fail outside of vnet  
nslookup $vault_name.vault.azure.net

```

## Create Secrets

```sh
# https://docs.microsoft.com/en-us/cli/azure/keyvault/secret?view=azure-cli-latest#az-keyvault-secret-set
az keyvault secret set --name $vault_secret_name --value $vault_secret --description "AKS ${appName} Secret" --vault-name $vault_name
az keyvault secret list --vault-name $vault_name
az keyvault secret show --vault-name $vault_name --name $vault_secret_name --output tsv

# Test ...
az vm create --name "kv-test-vm" \
    --image UbuntuLTS \
    --admin-username "kv_admin" \
    --ssh-key-values ~/.ssh/$ssh_key.pub \
    # --generate-ssh-keys
    --resource-group $rg_kv_name \
    --vnet-name $kv_vnet_name \
    --subnet $kv_subnet_name \
    --nsg "nsg-management" \
    --size Standard_D1_v2 \
    --zone 1 \
    --location $location
    # --enabled-for-disk-encryption True

ls -al ~/.ssh/
mv ~/.ssh/id_rsa ~/.ssh/id_rsa_kv_vm
mv ~/.ssh/id_rsa.pub ~/.ssh/id_rsa_kv_vm.pub
ls -al ~/.ssh/

```

