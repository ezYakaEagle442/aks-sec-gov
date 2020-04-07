# Setup Azure firewall

see [https://docs.microsoft.com/en-us/azure/firewall/deploy-cli](https://docs.microsoft.com/en-us/azure/firewall/deploy-cli)

## Create Azure firewall
```sh

# https://docs.microsoft.com/en-us/cli/azure/network/public-ip?view=azure-cli-latest
az network public-ip create -g $rg_fw_name -n $firewall_public_IP_name -l $location --sku "Standard"
public_ip_id=$(az network public-ip show -n $firewall_public_IP_name --subscription $subId --resource-group $rg_fw_name --query "ipAddress" -o tsv)
echo $public_ip_id

# TO be FIXED : az network public-ip show --ids $public_ip_id # --subscription $subId --resource-group $managed_rg

# https://github.com/Azure/azure-quickstart-templates/tree/master/101-azurefirewall-create-with-ipgroups-and-linux-jumpbox
# https://github.com/Azure/azure-quickstart-templates/tree/master/101-azurefirewall-sandbox-linux
# https://github.com/Azure/azure-quickstart-templates/tree/master/101-azurefirewall-with-zones-sandbox

# https://docs.microsoft.com/en-us/azure/firewall/overview
# https://docs.microsoft.com/en-us/cli/azure/ext/azure-firewall/network/firewall?view=azure-cli-latest#ext-azure-firewall-az-network-firewall-create
az extension add --name azure-firewall
az network firewall create -n $firewall_name --zones 1 2 3 --location $location -g $rg_fw_name --sku AZFW_VNet

# check
az network firewall list --resource-group $rg_fw_name
az network firewall show --name $firewall_name --resource-group $rg_fw_name

# Configure Firewall IP Config
# /!\ This command will take a few mins.
az network firewall ip-config create -g $rg_fw_name -f $firewall_name -n $firewall_ip_config_name --public-ip-address $firewall_public_IP_name --vnet-name $firewall_vnet_name
# Capture Firewall IP Address for Later Use
fw_pub_IP=$(az network public-ip show -g $rg_fw_name -n $firewall_public_IP_name --query "ipAddress" -o tsv)
fw_prv_IP=$(az network firewall show -g $rg_fw_name -n $firewall_name --query "ipConfigurations[0].privateIpAddress" -o tsv)

echo "Firewall Public IP: " $fw_pub_IP
echo "Firewall Private IP: " $fw_prv_IP

# create a user-defined route (UDR) to force all egress traffic, make sure you create an appropriate DNAT rule in Firewall to correctly allow ingress traffic.
# https://docs.microsoft.com/en-us/cli/azure/network/route-table?view=azure-cli-latest#az-network-route-table-create
az network route-table list
az network route-table create -n $fw_route_table_name -g $rg_name

# https://docs.microsoft.com/en-us/cli/azure/network/route-table/route?view=azure-cli-latest#az-network-route-table-route-create
az network route-table route create -n $fw_route --route-table-name $fw_route_table_name -g $rg_name  \
    --next-hop-type VirtualAppliance --address-prefix 0.0.0.0/0 --next-hop-ip-address $fw_prv_IP

# Associate AKS Subnet to FW
az network vnet subnet update --name $subnet_name --vnet-name $vnet_name --route-table $fw_route_table_name -g $rg_name 

```

