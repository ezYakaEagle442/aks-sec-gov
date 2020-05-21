# Setup Key-Vault

Generate & save nodes [SSH keys](https://docs.microsoft.com/en-us/azure/aks/ssh) to Azure Key-Vault is a [Best-practice](https://github.com/Azure/k8s-best-practices/blob/master/Security_securing_a_cluster.md#securing-host-access)


See also :
- [https://docs.microsoft.com/en-us/azure/aks/developer-best-practices-pod-security#use-pod-managed-identities](https://docs.microsoft.com/en-us/azure/aks/developer-best-practices-pod-security#use-pod-managed-identities)
- [https://docs.microsoft.com/en-us/azure/key-vault/key-vault-soft-delete-cli](https://docs.microsoft.com/en-us/azure/key-vault/key-vault-soft-delete-cli)
- [https://docs.microsoft.com/en-us/azure/key-vault/private-link-service#establish-a-private-link-connection-to-key-vault-using-cli](https://docs.microsoft.com/en-us/azure/key-vault/private-link-service#establish-a-private-link-connection-to-key-vault-using-cli)
- [https://docs.microsoft.com/en-us/azure/private-link/create-private-link-service-cli](https://docs.microsoft.com/en-us/azure/private-link/create-private-link-service-cli)
- The KeyVault FlexVol solution is now deprecated in favor of [https://github.com/kubernetes-sigs/secrets-store-csi-driver](https://github.com/kubernetes-sigs/secrets-store-csi-driver)
- [https://github.com/Azure/secrets-store-csi-driver-provider-azure](https://github.com/Azure/secrets-store-csi-driver-provider-azure)

## Pre-requisites

[https://docs.microsoft.com/en-us/azure/key-vault/general/private-link-service#prerequisites](https://docs.microsoft.com/en-us/azure/key-vault/general/private-link-service#prerequisites)
To integrate a key vault with Azure Private Link, you will need the following:
- A key vault.
- An Azure virtual network.
- A subnet in the virtual network.
- Owner or contributor permissions for both the key vault and the virtual network.

```sh

az provider register -n Microsoft.KeyVault

# https://docs.microsoft.com/en-us/azure/key-vault/key-vault-soft-delete-cli
# az keyvault list-deleted
# az keyvault purge --name $vault_name --location $location

az keyvault create --name $vault_name --enable-soft-delete true --location $location -g $rg_name
az keyvault show --name $vault_name 
az keyvault update --name $vault_name --default-action deny -g $rg_name 

kv_id=$(az keyvault show --name $vault_name -g $rg_name --query "id" --output tsv)
echo "KeyVault ID :" $kv_id

```

## Setup DNS

```sh
az keyvault private-link-resource list --vault-name $vault_name -g $rg_name
az network private-dns zone create --name privatelink.vaultcore.azure.net -g $rg_name 

# AKS
az network private-dns link vnet create --name $kv_private_dns_link_name --virtual-network $vnet_name --zone-name privatelink.vaultcore.azure.net --registration-enabled false -g $rg_name

private_dns_link_id=$(az network private-dns link vnet show --name $kv_private_dns_link_name --zone-name "privatelink.vaultcore.azure.net" -g $rg_name --query "id" --output tsv)
echo "Private-Link DNS ID :" $private_dns_link_id

# Bastion
az network private-dns link vnet create --name $kv_bastion_private_dns_link_name --virtual-network $bastion_vnet_id --zone-name privatelink.vaultcore.azure.net --registration-enabled false -g $rg_name

bastion_private_dns_link_id=$(az network private-dns link vnet show --name $kv_bastion_private_dns_link_name --zone-name "privatelink.vaultcore.azure.net" -g $rg_name --query "id" --output tsv)
echo "Private-Link DNS ID :" $bastion_private_dns_link_id

nslookup $vault_name.vault.azure.net

```


## Create a Private Endpoint for AKS

```sh

az network private-endpoint create --name $kv_private_endpoint_name --subnet $subnet_id --manual-request --request-message "Please allow this AKS $appName to access KeyVault $vault_name" --private-connection-resource-id $kv_id --group-ids vault --connection-name $kv_private_endpoint_svc_con_name --location $location -g $rg_name

# Look at the PENDING status in the portal : https://docs.microsoft.com/en-us/azure/key-vault/general/private-link-service#how-to-manage-a-private-endpoint-connection-to-key-vault-using-the-azure-portal
az keyvault private-endpoint-connection approve -n $kv_private_endpoint_svc_con_name --description "AKS $appName request to access KeyVault $vault_name ALLOWED by CLI" --vault-name $vault_name -g $rg_name 

# Validate private link connection
az keyvault private-endpoint-connection show --name $kv_private_endpoint_svc_con_name --vault-name $vault_name -g $rg_name

kv_private_endpoint_id=$(az network private-endpoint show --name $kv_private_endpoint_name -g $rg_name --query id -o tsv)
echo "ACR private-endpoint ID :" $kv_private_endpoint_id

network_interface_id=$(az network private-endpoint show --name $kv_private_endpoint_name -g $rg_name --query 'networkInterfaces[0].id' -o tsv)
echo "KeyVault Network Interface ID :" $network_interface_id

kv_network_interface_private_ip=$(az resource show --ids $network_interface_id \
  --api-version 2019-04-01 --query 'properties.ipConfigurations[0].properties.privateIPAddress' \
  --output tsv)
echo "KeyVault Network Interface private IP :" $kv_network_interface_private_ip

az keyvault private-link-resource list --vault-name $vault_name -g $rg_name

```

## Update DNS records

```sh

# https://docs.microsoft.com/en-us/azure/dns/private-dns-getstarted-cli#create-an-additional-dns-record
az network private-dns zone list -g $rg_name
az network private-dns record-set a add-record -g $rg_name -z "privatelink.vaultcore.azure.net" -n $vault_name -a $kv_network_interface_private_ip
az network private-dns record-set list -g $rg_name -z "privatelink.vaultcore.azure.net"

# From home/public network, you wil get a public IP. If inside a vnet with private zone, then nslookup will resolve to the private ip.
nslookup $vault_name.vault.azure.net
nslookup $vault_name.privatelink.vaultcore.azure.net

```

## Update network rules
```sh
# https://docs.microsoft.com/en-us/azure/key-vault/general/network-security
az keyvault network-rule list --name $vault_name -g $rg_name
az network vnet list-endpoint-services -l $location

# Allow AKS cluster
az network vnet subnet update -g $rg_name --vnet-name $vnet_name --name $subnet_name --service-endpoints "Microsoft.KeyVault"
az keyvault network-rule add --name $vault_name -g $rg_name --subnet $subnet_id
# az keyvault network-rule add --name $vault_name -g $rg_name --ip-address "172.16.5.0/24"

az network vnet subnet update -g $rg_name --vnet-name $vnet_name --name $new_node_pool_subnet_name --service-endpoints "Microsoft.KeyVault"
az keyvault network-rule add --name $vault_name -g $rg_name --subnet $new_node_pool_subnet_id

# Test from bastion
az network vnet subnet update -g $rg_bastion_name --vnet-name $vnet_bastion_name --name $subnet_bastion_name --service-endpoints "Microsoft.KeyVault"
az keyvault network-rule add --name $vault_name -g $rg_name --subnet $bastion_subnet_id

az keyvault network-rule list --name $vault_name -g $rg_name
```

## Check private-link from Bastion
```sh
# From home/public network, you wil get a public IP. If inside a vnet with private zone, then nslookup will resolve to the private ip.
nslookup $vault_name.vault.azure.net
nslookup $vault_name.privatelink.vaultcore.azure.net
```

## Create Secrets

```sh
# https://docs.microsoft.com/en-us/cli/azure/keyvault/secret?view=azure-cli-latest#az-keyvault-secret-set

# sudo apt-key adv --keyserver packages.microsoft.com --recv-keys EB3E94ADBE1229CF


# Test ...
az vm create --name "kv-test-vm" \
    --image UbuntuLTS \
    --admin-username "kv_admin" \
    --ssh-key-values ~/.ssh/$ssh_key.pub \
    --resource-group $rg_name \
    --vnet-name $vnet_name \
    --subnet $subnet_name \
    --nsg "nsg-management" \
    --size Standard_D1_v2 \
    --zone 1 \
    --location $location
    # --enabled-for-disk-encryption True

# flush DNS : /etc/dbus-1/system.d/dnsmasq.conf
# sudo systemd-resolve --flush-caches

vm_id=$(az vm show --name kv-test-vm -g $rg_name --query id)

az network private-endpoint create --name "test private endpoint" --subnet $subnet_id --manual-request --request-message "Please allow this test VM to access KeyVault $vault_name" --private-connection-resource-id $vm_id --group-ids vault --connection-name "kv_private_endpoint_svc_con_test" --location $location -g $rg_name

# Look at the PENDING status in the portal : https://docs.microsoft.com/en-us/azure/key-vault/general/private-link-service#how-to-manage-a-private-endpoint-connection-to-key-vault-using-the-azure-portal
az keyvault private-endpoint-connection approve -n $kv_private_endpoint_svc_con_name --description "$appName request to access KeyVault $vault_name ALLOWED by CLI" --vault-name $vault_name -g $rg_name 

# Validate private link connection
az keyvault private-endpoint-connection show --name $kv_private_endpoint_svc_con_name --vault-name $vault_name -g $rg_name

az keyvault secret set --name $vault_secret_name --value $vault_secret --description "AKS ${appName} Secret" --vault-name $vault_name
az keyvault secret list --vault-name $vault_name
az keyvault secret show --vault-name $vault_name --name $vault_secret_name --output tsv

# ls -al ~/.ssh/
# mv ~/.ssh/id_rsa ~/.ssh/id_rsa_kv_vm
# mv ~/.ssh/id_rsa.pub ~/.ssh/id_rsa_kv_vm.pub
# ls -al ~/.ssh/

```

