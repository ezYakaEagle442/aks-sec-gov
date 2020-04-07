#  Clean-Up

```sh

# Workspaces
az monitor log-analytics workspace delete --workspace-name $acr_analytics_workspace -g $rg_acr_name
az monitor log-analytics workspace delete --workspace-name $analytics_workspace_name -g $rg_name

az group deployment delete -n $analytics_workspace_name -g $rg_name

# AKS
az aks nodepool delete --name $poc_node_pool_name --cluster-name $cluster_name -g $rg_name
az aks nodepool delete --name $spotpool_name_max  --cluster-name $cluster_name -g $rg_name
az aks nodepool delete --name $spotpool_name_min --cluster-name $cluster_name -g $rg_name

az aks delete --name $cluster_name --resource-group $rg_name -y


# ACR
az acr delete -g $rg_acr_name --name $acr_registry_name -y
az network vnet peering delete -n $acr_vnet_peering_name --vnet-name $vnet_name -g $rg_name --subscription $subId

# KV
az keyvault delete --name $vault_name --resource-group $rg_name
az network vnet peering delete -n $kv_vnet_peering_name --vnet-name $vnet_name -g $rg_name --subscription $subId

# Networks
az network vnet subnet delete --name $subnet_name --vnet-name $vnet_name --resource-group $rg_name 
az network vnet subnet delete --name $appgw_subnet_name --vnet-name $vnet_name --resource-group $rg_name 
az network vnet subnet delete --name $ilb_subnet_name --vnet-name $vnet_name --resource-group $rg_name 
az network vnet subnet delete --name $appgw_subnet_id --vnet-name $vnet_appgw --resource-group $rg_name 

az network vnet delete --name $vnet_name --resource-group $rg_name 
az network vnet delete --name $vnet_appgw --resource-group $rg_name

az network vnet subnet delete --name acr_subnet_name --vnet-name $acr_vnet_name --resource-group $rg_acr_name 
az network vnet delete --name  $acr_vnet_name --resource-group $rg_acr_name 
az network private-dns link vnet delete --name $acr_private_dns_link_name -g $rg_acr_name --zone-name "privatelink.azurecr.io" -y
az network private-endpoint delete --name $acr_private_endpoint_name -g $rg_acr_name # --ids $acr_subnet_id 

az network private-dns record-set a delete --name $acr_registry_name --zone-name privatelink.azurecr.io -g $rg_acr_name -y
az network private-dns record-set a delete --name ${acr_registry_name}.${location}.data --zone-name privatelink.azurecr.io -g $rg_acr_name -y

# Bastion
az network vnet delete --name $vnet_bastion_name --resource-group $rg_bastion_name
az network vnet subnet delete --name $subnet_bastion_name --vnet-name $vnet_bastion_name --resource-group $rg_bastion_name
az network vnet subnet delete --name ManagementSubnet --vnet-name $vnet_bastion_name --resource-group $rg_bastion_name
az network vnet subnet delete --name GatewaySubnet --vnet-name $vnet_bastion_name --resource-group $rg_bastion_name
az network vnet subnet delete --name AzureBastionSubnet --vnet-name $vnet_bastion_name --resource-group $rg_bastion_name

az network bastion delete --name $bastion_name -g $rg_bastion_name --subscription $subId
az network public-ip delete --name $bastion_IP --subscription $subId # --ids $public_ip_id
az network nsg delete --name nsg-management -g $rg_bastion_name --subscription $subId
az vm delete --name $bastion_name -g $rg_bastion_name --subscription $subId --yes
az network vnet peering delete -n $vnet_peering_name_bastion_aks --vnet-name $vnet_bastion_name -g $rg_name --subscription $subId

# DNS
az network dns record-set a delete -g $rg_name -z $dnz_zone -n www 
az network dns zone delete -g $rg_name -n $dnz_zone
az network dns zone list -g $rg_name
az group delete --name $rg_name

# App. gateway
az network application-gateway delete -g $rg_name -n $appgw_name
az network public-ip delete --name $appgw_IP --subscription $subId -g $rg_name

az network application-gateway delete -g $rg_name -n $appgw_agic_name

# Firewall

az network route-table route delete -n $fw_route --route-table-name $fw_route_table_name -g $rg_name 
az network route-table delete -n $fw_route_table_name -g $rg_name 

az network firewall network-rule collection delete --collection-name $fw_network_collection_name --firewall-name $firewall_name -g $rg_fw_name

az network firewall application-rule collection delete --collection-name $fw_network_collection_name --firewall-name $firewall_name -g $rg_fw_name

az network firewall application-rule delete -n $fw_app_rule_name -g $rg_fw_name -f $firewall_name --collection-name $fw_network_collection_name
az network firewall network-rule delete -n "fw-net-rules-TCP" -g $rg_fw_name -f $firewall_name --collection-name $fw_network_collection_name
az network firewall network-rule delete -n "fw-net-rules-UDP" -g $rg_fw_name -f $firewall_name --collection-name $fw_network_collection_name
az network firewall network-rule delete -n "fw-net-rules-aks-api" -g $rg_fw_name -f $firewall_name --collection-name $fw_network_collection_name
az network firewall network-rule delete -n "fw-net-rules-azure-arc-TCP" -g $rg_fw_name -f $firewall_name --collection-name $fw_network_collection_name

az network firewall delete -n $firewall_name -g $rg_fw_name
az network public-ip delete --name $firewall_public_IP_name --subscription $subId  -g $rg_fw_name

az network vnet peering delete -n $vnet_peering_name_aks_fw --vnet-name $vnet_name -g $rg_fw_name --subscription $subId

az network vnet subnet delete --name $firewall_subnet_name --vnet-name $firewall_vnet_name -g $rg_fw_name
az network vnet delete --name $firewall_vnet_name -g $rg_fw_name

# AAD
az ad user delete --upn-or-object-id $AKSDEV_ID
az ad user delete --upn-or-object-id $AKSSRE_ID

az ad group delete --upn-or-object-id $APPDEV_ID
az ad group delete --upn-or-object-id $OPSSRE_ID
```

## Service Principal
<span style="color:red">**/!\ IMPORTANT** </span> : Decide to keep or delete the Service Principal used to create the AKS cluster
```sh
az ad sp delete --id $sp_id
rm spp.txt
rm spid.txt
```

## Files
```sh
# TODO

```

## RG
az storage account delete --name $storage_name  --resource-group $rg_name --subscription $subId --yes
az group delete --name $rg_name --subscription $subId --yes 
az group delete --name $rg_acr_name --subscription $subId --yes
az group delete --name $rg_kv_name --subscription $subId --yes 
az group delete --name $rg_fw_name --subscription $subId --yes 
az group delete --name $rg_bastion_name --subscription $subId --yes 

## CleanUp AAD
```sh
# TODO
rm tenantId.txt
rm serverApplicationId.txt
rm serverApplicationSecret.txt
rm clientApplicationId.txt

```