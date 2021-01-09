## Plan IP addressing for your cluster

See  :

- [Public IP sku comparison](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-ip-addresses-overview-arm#sku)

- [https://docs.microsoft.com/en-us/azure/aks/configure-azure-cni#plan-ip-addressing-for-your-cluster](https://docs.microsoft.com/en-us/azure/aks/configure-azure-cni#plan-ip-addressing-for-your-cluster)

- For Advanced networking options, see [https://docs.microsoft.com/en-us/azure/aks/configure-azure-cni](https://docs.microsoft.com/en-us/azure/aks/configure-azure-cni)

```sh
# https://www.ipaddressguide.com/cidr
# https://github.com/Azure/azure-quickstart-templates/tree/master/101-aks-advanced-networking-aad
# https://github.com/Azure/azure-quickstart-templates/tree/master/101-aks-advanced-networking


``` 

## Create Networks

```sh
# AKS nodes VNet & Subnet
az network vnet create --name $vnet_name --resource-group $rg_name --address-prefixes 172.16.0.0/16 --location $location
az network vnet subnet create --name $subnet_name --address-prefixes 172.16.1.0/24 --vnet-name $vnet_name --resource-group $rg_name 
# extra subnet for new node pools : see https://github.com/Azure/AKS/issues/1338
az network vnet subnet create --name $new_node_pool_subnet_name --address-prefixes 172.16.5.0/24 --vnet-name $vnet_name -g $rg_name 

vnet_id=$(az network vnet show --resource-group $rg_name --name $vnet_name --query id -o tsv)
echo "VNet Id :" $vnet_id	

subnet_id=$(az network vnet subnet show --resource-group $rg_name --vnet-name $vnet_name --name $subnet_name --query id -o tsv)
echo "Subnet Id :" $subnet_id	

new_node_pool_subnet_id=$(az network vnet subnet show -g $rg_name --vnet-name $vnet_name --name $new_node_pool_subnet_name --query id -o tsv)
echo "New Node-Pool Subnet Id :" $new_node_pool_subnet_id	

# https://docs.microsoft.com/en-us/azure/private-link/create-private-link-service-cli#disable-private-link-service-network-policies-on-subnet
az network vnet subnet update --name $subnet_name --vnet-name $vnet_name --disable-private-link-service-network-policies true -g $rg_name

# /!\ **IMPORTANT** : https://docs.microsoft.com/en-us/azure/aks/configure-azure-cni?view=azure-cli-latest#prerequisites
# https://docs.microsoft.com/en-us/azure/aks/configure-kubenet?view=azure-cli-latest#create-a-service-principal-and-assign-permissions

# https://docs.microsoft.com/en-us/azure/role-based-access-control/custom-roles-cli
# az role definition create --role-definition <role_definition>
# https://docs.microsoft.com/en-us/azure/aks/load-balancer-standard#before-you-begin ==>  Network contributor
az role assignment list --assignee $sp_id 
az role assignment create --assignee $sp_id --scope $vnet_id --role Contributor
az role assignment create --assignee $sp_id --scope $subnet_id --role "Network contributor"

# When using AGIC , App Gateway runs in AKS VNet
az network vnet subnet create --name $agic_subnet_name --address-prefixes 172.16.2.0/24 --vnet-name $vnet_name --resource-group $rg_name

agic_subnet_id=$(az network vnet subnet show --resource-group $rg_name --vnet-name $vnet_name --name $agic_subnet_name --query id -o tsv)
echo "AGIC Subnet Id :" $agic_subnet_id	

# This is the subnet that will be used for Kubernetes Services that are exposed via an Internal Load Balancer (ILB). This mean the ILB internal IP will be from this subnet address space. By doing it this way we do not take away from the existing IP Address space in the AKS subnet that is used for Nodes and Pods.
az network vnet subnet create --name $ilb_subnet_name --address-prefixes 172.16.42.0/24 --vnet-name $vnet_name -g $rg_name 
ilb_subnet_id=$(az network vnet subnet show --resource-group $rg_name --vnet-name  $vnet_name --name $ilb_subnet_name --query id -o tsv)
echo "Internal Load BalancerLB Subnet Id :" $ilb_subnet_id	

# Firewall
az group create --name $rg_fw_name --location $location

az network vnet create --name $firewall_vnet_name --resource-group $rg_fw_name --address-prefixes 172.21.0.0/24 --location $location
# AzureFirewallSubnet must be at least /26 for deploying an Azure Firewall.
az network vnet subnet create --name $firewall_subnet_name --address-prefixes 172.21.0.0/26 --vnet-name $firewall_vnet_name -g $rg_fw_name

fw_vnet_id=$(az network vnet show --resource-group $rg_fw_name --name $firewall_vnet_name --query id -o tsv)
echo "Firewall VNet Id :" $fw_vnet_id	

fw_subnet_id=$(az network vnet subnet show --resource-group $rg_fw_name --vnet-name $firewall_vnet_name --name $firewall_subnet_name --query id -o tsv)
echo "Firewall Subnet Id :" $fw_subnet_id	

# App Gateway(not AGIC) runs in its own VNet
az network vnet create --name $vnet_appgw -g $rg_name --address-prefixes 172.44.0.0/24 --location $location
az network vnet subnet create --name $appgw_subnet_name --address-prefixes 172.44.0.0/27 --vnet-name $vnet_appgw -g $rg_name 

appgw_vnet_id=$(az network vnet show --resource-group $rg_name --name $vnet_appgw --query id -o tsv)
echo "App Gateway VNet Id :" $appgw_vnet_id	

appgw_subnet_id=$(az network vnet subnet show --resource-group $rg_name --vnet-name $vnet_appgw --name $appgw_subnet_name --query id -o tsv)
echo "App Gateway Subnet Id :" $appgw_subnet_id	


# ACR - PrivateLink
# az group create --name $rg_acr_name --location $location
az network vnet create --name $acr_vnet_name -g $rg_name --address-prefixes 172.42.42.0/24 --location $location
az network vnet subnet create --name $acr_subnet_name --address-prefixes 172.42.42.0/27 --vnet-name $acr_vnet_name -g $rg_name 

acr_vnet_id=$(az network vnet show --resource-group $rg_name --name $acr_vnet_name --query id -o tsv)
echo "ACR VNet Id :" $acr_vnet_id	

acr_subnet_id=$(az network vnet subnet show --resource-group $rg_name --vnet-name $acr_vnet_name --name $acr_subnet_name --query id -o tsv)
echo "ACR Subnet Id :" $acr_subnet_id	

az network vnet subnet update --name $acr_subnet_name --disable-private-endpoint-network-policies --vnet-name $acr_vnet_name -g $rg_name

# KeyVault - PrivateLink
# az group create --name $rg_kv_name --location $location
az network vnet create --name $kv_vnet_name -g $rg_name --address-prefixes 172.12.0.0/24 --location $location
az network vnet subnet create --name $kv_subnet_name --address-prefixes 172.12.0.0/27 --vnet-name $kv_vnet_name -g $rg_name 

# az network vnet subnet update --name $kv_subnet_name -g $rg_name --vnet-name $kv_vnet_name --disable-private-endpoint-network-policies true
az network vnet subnet update --name $kv_subnet_name --vnet-name $kv_vnet_name --disable-private-link-service-network-policies true -g $rg_name

kv_vnet_id=$(az network vnet show --name $kv_vnet_name -g $rg_name --query id -o tsv)
echo "KeyVault VNet Id :" $kv_vnet_id	

kv_subnet_id=$(az network vnet subnet show --name $kv_subnet_name --vnet-name $kv_vnet_name -g $rg_name --query id -o tsv)
echo "KeyVault Subnet Id :" $kv_subnet_id	


```

## Setup Bastion


### Create RG

See also [https://docs.microsoft.com/en-us/azure/security/fundamentals/network-best-practices](https://docs.microsoft.com/en-us/azure/security/fundamentals/network-best-practices)

```sh
az group create --name $rg_bastion_name --location $location

```

### Prepare pre-requisites
[Azure Bastion](https://docs.microsoft.com/en-us/azure/bastion/bastion-overview) allows you to log into VMs in the virtual network through SSH or remote desktop protocol (RDP) without exposing the VMs directly to the internet. Use Bastion to manage the VMs in the virtual network.

RDP and SSH directly in Azure portal: You can directly get to the RDP and SSH session directly in the Azure portal using a single click seamless experience.

Remote Session over SSL and firewall traversal for RDP/SSH: Azure Bastion uses an HTML5 based web client that is automatically streamed to your local device, so that you get your RDP/SSH session over SSL on port 443 enabling you to traverse corporate firewalls securely.

No hassle of managing NSGs: Azure Bastion is a fully managed platform PaaS service from Azure that is hardened internally to provide you secure RDP/SSH connectivity. You don't need to apply any NSGs on Azure Bastion subnet.

[AKS Best practice guidance](https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-network#securely-connect-to-nodes-through-a-bastion-host) - Don't expose remote connectivity to your AKS nodes. Create a bastion host, or jump box, in a management virtual network. Use the bastion host to securely route traffic into your AKS cluster to remote management tasks.
Most operations in AKS can be completed using the Azure management tools or through the Kubernetes API server. AKS nodes aren't connected to the public internet, and are only available on a private network. To connect to nodes and perform maintenance or troubleshoot issues, route your connections through a bastion host, or jump box. This host should be in a separate management virtual network that is securely peered to the AKS cluster virtual network.

[https://docs.microsoft.com/en-us/cli/azure/network/bastion?view=azure-cli-latest#az-network-bastion-create](https://docs.microsoft.com/en-us/cli/azure/network/bastion?view=azure-cli-latest#az-network-bastion-create)
The SKU of the public IP must be Standard.
Name of the virtual network. **It must have a subnet called AzureBastionSubnet.**

```sh



# The gateway provides connectivity between the routers in the on-premises network and the virtual network. The gateway is placed in its own subnet. Azure only allows VPN and ExpressRoute gateways to be deployed into these subnets.
# https://docs.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-vpn-faq#do-i-need-a-gatewaysubnet
az network vnet create --name $vnet_bastion_name -g $rg_bastion_name --address-prefixes 192.168.0.0/21 --location $location
az network vnet subnet create --name $subnet_bastion_name --address-prefixes 192.168.1.0/24 --vnet-name $vnet_bastion_name -g $rg_bastion_name
az network vnet subnet create --name ManagementSubnet --address-prefixes 192.168.2.0/24 --vnet-name $vnet_bastion_name -g $rg_bastion_name
az network vnet subnet create --name GatewaySubnet --address-prefixes 192.168.3.0/24 --vnet-name $vnet_bastion_name -g $rg_bastion_name

bastion_vnet_id=$(az network vnet show --name $vnet_bastion_name -g $rg_bastion_name --query id -o tsv)
echo "Bastion VNet Id :" $bastion_vnet_id	

bastion_subnet_id=$(az network vnet subnet show --name $subnet_bastion_name --vnet-name $vnet_bastion_name -g $rg_bastion_name --query id -o tsv)
echo "Bastion Subnet Id :" $bastion_subnet_id	

# https://github.com/Azure/azure-quickstart-templates/tree/master/101-azure-bastion

```

### Setup NSG 
```sh

# https://github.com/Azure/azure-quickstart-templates/tree/master/101-azure-bastion-nsg
# https://docs.microsoft.com/en-us/azure/bastion/bastion-nsg
# NSG sample : https://user-images.githubusercontent.com/47132998/69514141-4f55d380-0f70-11ea-980e-2094bd57de20.png
# https://github.com/Azure/azure-quickstart-templates/blob/master/101-azure-bastion-nsg/azuredeploy.json

b_nsg="bastion-nsg-management"
az network nsg create --name $b_nsg -g $rg_bastion_name --location $location

az network nsg rule create --access Allow --destination-port-range 22 --source-address-prefixes Internet --name "Allow SSH from Internet" --nsg-name $b_nsg -g $rg_bastion_name --priority 100

az network nsg rule create --access Allow --destination-port-range 3389 --source-address-prefixes Internet --name "Allow RDP from Internet" --nsg-name $b_nsg -g $rg_bastion_name --priority 110

az network vnet subnet update --name ManagementSubnet --network-security-group $b_nsg --vnet-name $vnet_bastion_name -g $rg_bastion_name

```

Now, choose either to use [Azure Bastion](#create-azure-bastion) or to create a [JumpOff VM](#optionally-create-a-jumpoff-vm)

### Create Azure Bastion

Azure Bastion is a VMSS which can scale according to the number of connections.
It is now available in France Central, CLI 2.2.0 is required

```sh

# Create the resource-id of the public ip: https://docs.microsoft.com/en-us/cli/azure/network/public-ip?view=azure-cli-latest#az-network-public-ip-create
az network public-ip create --name $bastion_IP \
                            --location $location \
                            --allocation-method Static \
                            --dns-name $bastion_DNS_name \
                            -g $rg_bastion_name \
                            --subscription $subId \
                            --sku Standard \
                            --version IPv4
                            

public_ip_id=$(az network public-ip show --name $bastion_IP --query id -o tsv --subscription $subId -g $rg_bastion_name)
echo $public_ip_id

bastion_public_ip_address=$(az network public-ip show --name $bastion_IP --query ipAddress -o tsv --subscription $subId -g $rg_bastion_name)
echo $bastion_public_ip_address

# https://azure.microsoft.com/en-us/services/azure-bastion/#faqs ==> is now available in France Central, CLI 2.2.0 is required
az network bastion create --name $bastion_name --vnet-name $vnet_bastion_name --public-ip-address $public_ip_id -g $rg_bastion_name --location $location --subscription $subId
az network bastion list -g $rg_bastion_name --subscription $subId

# https://docs.microsoft.com/en-us/azure/bastion/bastion-connect-vm-ssh

```

Now go to the section [§Setup VNet peering](#setup-VNet-peering)
### Optionally Create a JumpOff VM

See
- [https://docs.microsoft.com/en-us/azure/virtual-machines/windows/sizes](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/sizes)
- [https://docs.microsoft.com/en-us/azure/virtual-machines/linux/sizes](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/sizes)

[Your SSH keys should have been generated at pre-req step](./setup-prereq-rg-spn#generates-your-ssh-keys)


```sh
#az network nsg create --name nsg-management -g $rg_bastion_name --location $location 
#az network nsg rule create --access Allow -destination-port-range 3389 -source-address-prefixes Internet --name "Allow RDP from Internet" --nsg-name nsg-management -g $rg_bastion_name --priority 100
#az network nsg rule create --access Allow -destination-port-range 22-source-address-prefixes Internet --name "Allow SSH from Internet" --nsg-name nsg-management -g $rg_bastion_name --priority 110
#az network vnet subnet update --network-security-group nsg-management --name ManagementSubnet --vnet-name $vnet_bastion_name -g $rg_bastion_name

# az vm list-sizes --location $location --output table
# az vm image list-publishers --location $location --output table
# az vm image list-offers --publisher MicrosoftWindowsServer --location $location --output table
# az vm image list --publisher MicrosoftWindowsServer --offer WindowsServer --location $location --output table

# az vm image list-publishers --location $location --output table | grep -i Canonical
# az vm image list-offers --publisher Canonical --location $location --output table
# az vm image list --publisher Canonical --offer UbuntuServer --location $location --output table

# --size Standard_D1_v2 or Standard_B1s
az vm create --name $bastion_name \
    --image UbuntuLTS \
    --admin-username $bastion_admin_username \
    --resource-group $rg_bastion_name \
    --vnet-name $vnet_bastion_name \
    --subnet ManagementSubnet \
    --nsg $b_nsg \
    --size Standard_B1s \
    --zone 1 \
    --location $location \
    --ssh-key-values ~/.ssh/$ssh_key.pub
    # --generate-ssh-keys

network_interface_id=$(az vm show --name $bastion_name -g $rg_bastion_name --query 'networkProfile.networkInterfaces[0].id' -o tsv)
echo "Bastion VM Network Interface ID :" $network_interface_id

network_interface_private_ip=$(az resource show --ids $network_interface_id \
  --api-version 2019-04-01 --query 'properties.ipConfigurations[0].properties.privateIPAddress' -o tsv)
echo "Network Interface private IP :" $network_interface_private_ip

network_interface_pub_ip_id=$(az resource show --ids $network_interface_id \
  --api-version 2019-04-01 --query 'properties.ipConfigurations[0].properties.publicIPAddress.id' -o tsv)

network_interface_pub_ip=$(az network public-ip show -g $rg_name --id $network_interface_pub_ip_id --query "ipAddress" -o tsv)
echo "Network Interface public  IP :" $network_interface_pub_ip

# test
ssh -i ~/.ssh/$ssh_key $bastion_admin_username@$network_interface_pub_ip

# Save budget ! Stop will  power off the specified VM & it will continue to be billed.
# https://docs.microsoft.com/en-us/azure/virtual-machines/linux/states-lifecycle
# az vm stop --name $bastion_name -g $rg_bastion_name
# az vm deallocate --name $bastion_name -g $rg_bastion_name
# az vm delete --name $bastion_name -g $rg_bastion_name

```

<span style="color:red">/!\ IMPORTANT </span> : To successfully peer two virtual networks this command must be called twice with the values for --vnet-name and --remote-vnet reversed.

see [https://docs.microsoft.com/en-us/azure/virtual-network/tutorial-connect-virtual-networks-cli#peer-virtual-networks](https://docs.microsoft.com/en-us/azure/virtual-network/tutorial-connect-virtual-networks-cli#peer-virtual-networks)

## Setup VNet peering

Bastion ==> AKS
[https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-manage-peering#requirements-and-constraints](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-manage-peering#requirements-and-constraints)
```sh
# https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-peering-overview
# https://docs.microsoft.com/en-us/cli/azure/network/vnet/peering?view=azure-cli-latest#az-network-vnet-peering-create
az network vnet peering create -n $vnet_peering_name_bastion_aks \
    -g $rg_bastion_name \
    --subscription $subId \
    --allow-vnet-access \
    --vnet-name $vnet_bastion_name \
    --remote-vnet $vnet_id # https://docs.microsoft.com/en-us/azure/networking/scripts/virtual-network-cli-sample-peer-two-virtual-networks ==> remote-vnet-id parameter has been deprecated and will be removed in version '3.0.0'.
    # --remote-vnet $vnet_name ==> does not work 
    # --allow-forwarded-traffic

# In the output returned after the previous command executes, you see that the peeringState is Initiated. The peering remains in the 
# Initiated state until you create the peering from myVirtualNetwork2 to myVirtualNetwork1. Create a peering from myVirtualNetwork2 to myVirtualNetwork1.

az network vnet peering show -g $rg_bastion_name -n $vnet_peering_name_bastion_aks --vnet-name $vnet_bastion_name --query peeringState

az network vnet peering create -n $vnet_peering_name_bastion_aks \
    -g $rg_name \
    --subscription $subId \
    --allow-vnet-access \
    --vnet-name $vnet_name \
    --remote-vnet $bastion_vnet_id

az network vnet peering list -g $rg_bastion_name --vnet-name $vnet_bastion_name  --subscription $subId
az network vnet peering show -g $rg_bastion_name -n $vnet_peering_name_bastion_aks --vnet-name $vnet_bastion_name --query peeringState
az network vnet peering show -g $rg_name -n $vnet_peering_name_bastion_aks --vnet-name $vnet_name --query peeringState

```

## Setup VNet peering : AKS ==> ACR
```sh
az network vnet peering create -n $acr_vnet_peering_name \
    -g $rg_name \
    --subscription $subId \
    --allow-vnet-access \
    --vnet-name $vnet_name \
    --remote-vnet $acr_vnet_id

az network vnet peering show -g $rg_name -n $acr_vnet_peering_name --vnet-name $vnet_name --query peeringState

az network vnet peering create -n $acr_vnet_peering_name \
    -g $rg_name \
    --subscription $subId \
    --allow-vnet-access \
    --vnet-name $acr_vnet_name\
    --remote-vnet $vnet_id

az network vnet peering list -g $rg_name --vnet-name $vnet_name  --subscription $subId
az network vnet peering show -g $rg_name -n $acr_vnet_peering_name --vnet-name $vnet_name --query peeringState
az network vnet peering show -g $rg_name -n $acr_vnet_peering_name --vnet-name $acr_vnet_name --query peeringState

```
## Setup VNet peering : AKS ==> KV

[https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-manage-peering#requirements-and-constraints](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-manage-peering#requirements-and-constraints)
```sh
az network vnet peering create -n $kv_vnet_peering_name \
    -g $rg_name \
    --subscription $subId \
    --allow-vnet-access \
    --vnet-name $vnet_name \
    --remote-vnet $kv_vnet_id

az network vnet peering show -g $rg_name -n $kv_vnet_peering_name --vnet-name $vnet_name --query peeringState

az network vnet peering create -n $kv_vnet_peering_name \
    -g $rg_name \
    --subscription $subId \
    --allow-vnet-access \
    --vnet-name $kv_vnet_name \
    --remote-vnet $vnet_id

az network vnet peering list -g $rg_name --vnet-name $kv_vnet_name  --subscription $subId
az network vnet peering show -g $rg_name -n $kv_vnet_peering_name --vnet-name $vnet_name --query peeringState
az network vnet peering show -g $rg_name -n $kv_vnet_peering_name --vnet-name $kv_vnet_name --query peeringState

```

## Setup VNet peering : AKS ==> Firewall

```sh
az network vnet peering create -n $vnet_peering_name_aks_fw \
    -g $rg_name \
    --subscription $subId \
    --allow-vnet-access \
    --vnet-name $vnet_name \
    --remote-vnet $fw_vnet_id

az network vnet peering show -g $rg_name -n $vnet_peering_name_aks_fw --vnet-name $vnet_name --query peeringState

az network vnet peering create -n $vnet_peering_name_aks_fw \
    -g $rg_fw_name \
    --subscription $subId \
    --allow-vnet-access \
    --vnet-name $firewall_vnet_name \
    --remote-vnet $vnet_id	

az network vnet peering list -g $rg_fw_name --vnet-name $firewall_vnet_name --subscription $subId
az network vnet peering show -g $rg_fw_name -n $vnet_peering_name_aks_fw --vnet-name $firewall_vnet_name --query peeringState
az network vnet peering show -g $rg_name -n $vnet_peering_name_aks_fw --vnet-name $vnet_name --query peeringState

```

## Setup extra VNet peering 

### Setup VNet peering : Bastion ==> KV
```sh
az network vnet peering create -n $vnet_peering_name_bastion_kv \
    -g $rg_bastion_name \
    --subscription $subId \
    --allow-vnet-access \
    --vnet-name $vnet_bastion_name \
    --remote-vnet $kv_vnet_id

az network vnet peering show -g $rg_bastion_name -n $vnet_peering_name_bastion_kv --vnet-name $vnet_bastion_name --query peeringState

az network vnet peering create -n $vnet_peering_name_bastion_kv \
    -g $rg_name \
    --subscription $subId \
    --allow-vnet-access \
    --vnet-name $kv_vnet_name \
    --remote-vnet $bastion_vnet_id

az network vnet peering list -g $rg_name --vnet-name $kv_vnet_name --subscription $subId
az network vnet peering show -g $rg_name -n $vnet_peering_name_bastion_kv --vnet-name $kv_vnet_name --query peeringState
az network vnet peering show -g $rg_bastion_name -n $vnet_peering_name_bastion_kv --vnet-name $vnet_bastion_name --query peeringState

```

### Setup VNet peering : Bastion ==> ACR

```sh
az network vnet peering create -n $vnet_peering_name_bastion_acr \
    -g $rg_bastion_name \
    --subscription $subId \
    --allow-vnet-access \
    --vnet-name $vnet_bastion_name \
    --remote-vnet $acr_vnet_id

az network vnet peering show -g $rg_bastion_name -n $vnet_peering_name_bastion_acr --vnet-name $vnet_bastion_name --query peeringState

az network vnet peering create -n $vnet_peering_name_bastion_acr \
    -g $rg_name \
    --subscription $subId \
    --allow-vnet-access \
    --vnet-name $acr_vnet_name \
    --remote-vnet $bastion_vnet_id

az network vnet peering list -g $rg_name --vnet-name $acr_vnet_name --subscription $subId
az network vnet peering show -g $rg_name -n $vnet_peering_name_bastion_acr --vnet-name $acr_vnet_name --query peeringState
az network vnet peering show -g $rg_bastion_name -n $vnet_peering_name_bastion_acr --vnet-name $vnet_bastion_name --query peeringState

```

## Setup VNet peering : App. Gateway ==> AKS
```sh
az network vnet peering create -n $appgw_vnet_peering_name \
    -g $rg_name \
    --subscription $subId \
    --allow-vnet-access \
    --vnet-name $vnet_name \
    --remote-vnet $appgw_vnet_id

az network vnet peering show -g $rg_name -n $appgw_vnet_peering_name --vnet-name $vnet_name --query peeringState

az network vnet peering create -n $appgw_vnet_peering_name \
    -g $rg_name \
    --subscription $subId \
    --allow-vnet-access \
    --vnet-name $vnet_appgw\
    --remote-vnet $vnet_id

az network vnet peering list -g $rg_name --vnet-name $vnet_name  --subscription $subId
az network vnet peering show -g $rg_name -n $appgw_vnet_peering_name --vnet-name $vnet_name --query peeringState
az network vnet peering show -g $rg_name -n $appgw_vnet_peering_name --vnet-name $vnet_appgw --query peeringState