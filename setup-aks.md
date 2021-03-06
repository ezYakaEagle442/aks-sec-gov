# Create AKS Cluster

Regarding Private cluster, please read :
- [https://medium.com/@denniszielke/fully-private-aks-clusters-without-any-public-ips-finally-7f5688411184](https://medium.com/@denniszielke/fully-private-aks-clusters-without-any-public-ips-finally-7f5688411184)
- [https://docs.microsoft.com/en-us/azure/aks/private-clusters](https://docs.microsoft.com/en-us/azure/aks/private-clusters)


Regarding ACR :
- [https://docs.microsoft.com/en-us/azure/aks/cluster-container-registry-integration#create-a-new-aks-cluster-with-acr-integration](https://docs.microsoft.com/en-us/azure/aks/cluster-container-registry-integration#create-a-new-aks-cluster-with-acr-integration)
- [https://github.com/Azure/azure-quickstart-templates/tree/master/101-aks](https://github.com/Azure/azure-quickstart-templates/tree/master/101-aks)


## pre-requisites
```sh
az extension list-available
# az extension remove --name aks-preview
az extension add --name aks-preview
az extension update --name aks-preview
az feature register --name UseCustomizedUbuntuPreview --namespace Microsoft.ContainerService

# https://docs.microsoft.com/en-us/azure/governance/policy/concepts/rego-for-aks
# Provider register: Register the Azure Kubernetes Services provider
az provider register --namespace Microsoft.ContainerService

# Provider register: Register the Azure Policy provider
az provider register --namespace Microsoft.PolicyInsights

# Feature register: enables installing the add-on
az feature register --namespace Microsoft.ContainerService --name AKS-AzurePolicyAutoApprove

# Use the following to confirm the feature has registered
az feature list -o table --query "[?contains(name, 'Microsoft.ContainerService/AKS-AzurePolicyAutoApprove')].{Name:name,State:properties.state}"

# Once the above shows 'Registered' run the following to propagate the update
az provider register -n Microsoft.ContainerService

# Feature register: enables the add-on to call the Azure Policy resource provider
az feature register --namespace Microsoft.PolicyInsights --name AKS-DataPlaneAutoApprove

# Use the following to confirm the feature has registered
az feature list -o table --query "[?contains(name, 'Microsoft.PolicyInsights/AKS-DataPlaneAutoApprove')].{Name:name,State:properties.state}"

# Once the above shows 'Registered' run the following to propagate the update
az provider register -n Microsoft.PolicyInsights
# az provider unregister  -n Microsoft.PolicyInsights

# az feature register --name AKSPrivateLinkPreview --namespace Microsoft.ContainerService
# az feature list -o table --query "[?contains(name, 'Microsoft.ContainerService/AKSPrivateLinkPreview')].{Name:name,State:properties.state}"
# az provider register -n Microsoft.ContainerService 
# az provider register --namespace Microsoft.Network

az extension show --name aks-preview --query [version]

# ManagedIdentity requires aks-preview 0.4.38
az extension update --name aks-preview

```

### Prepare NSG to be able to access AKS from JumpOff

```sh
az network nsg create --name nsg-management -g $rg_name --location $location

az network nsg rule create --access Allow --destination-port-range 22 --source-address-prefixes Internet --name "Allow SSH from Internet" --nsg-name nsg-management -g $rg_name --priority 100

az network nsg rule create --access Allow --destination-port-range 22 --source-address-prefixes x.x.x.x --name "Allow SSH from Azure Bastion" --nsg-name nsg-management -g $rg_name --priority 101

az network nsg rule create --access Allow --destination-port-range 22 --source-address-prefixes 192.168.x.x --name "Allow SSH from Linux JumpOff" --nsg-name nsg-management -g $rg_name --priority 102

az network nsg rule create --access Allow --destination-port-range 3389 --source-address-prefixes Internet --name "Allow RDP from Internet" --nsg-name nsg-management -g $rg_name --priority 110

az network vnet subnet update --name $subnet_name --network-security-group nsg-management --vnet-name $vnet_name -g $rg_name

```

## Create AKS Cluster

To learn more about UDR, see [https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-udr-overview](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-udr-overview)


```sh

# small & cheap VM size : Basic_A1 or Standard_B1s or Standard_F2s_v2
az aks create --name $cluster_name \
    --resource-group $rg_name \
    --node-resource-group $ $cluster_rg_name \
    --aks-custom-headers CustomizedUbuntu=aks-ubuntu-1804 \
    --zones 1 2 3 \
    --enable-cluster-autoscaler \
    --min-count=1 \
    --max-count=3 \
    --node-vm-size Standard_F2s_v2 \
    --location $location \
    --vnet-subnet-id $subnet_id \
    --service-cidr 10.42.0.0/24 \
    --dns-service-ip 10.42.0.10 \
    --kubernetes-version $version \
    --network-plugin $network_plugin \
    --network-policy $network_policy \
    --nodepool-name $node_pool_name \
    --admin-username $aks_admin_username \
    --load-balancer-sku standard \
    --vm-set-type VirtualMachineScaleSets \
    --verbose \
    --ssh-key-value ~/.ssh/${ssh_key}.pub \
    # --generate-ssh-keys \
    # outboundType ==> userDefinedRouting|loadBalancer (public preview target on release) : Addresses SLB configuration to skip setup of public IP addresses and backend pools https://github.com/Azure/azure-rest-api-specs/blob/master/specification/containerservice/resource-manager/Microsoft.ContainerService/stable/2020-03-01/managedClusters.json#L1714
    # see issue #1385 https://github.com/Azure/AKS/issues/1385 & https://docs.microsoft.com/en-us/azure/aks/egress-outboundtype
    --outbound-type userDefinedRouting \
    --enable-private-cluster \
    --enable-addons azure-policy \
    --enable-managed-identity \
    #--service-principal $sp_id \ ==> you do not need it when enabling managed-identity
    #--client-secret $sp_password \ ==> you do not need it when enabling managed-identity
    # --attach-acr $acr_registry_name \
    # requires Azure CLI, version 2.2.0 or later : https://docs.microsoft.com/en-us/azure/aks/use-managed-identity , and also  0.4.38 of the preview cli az extension update --name aks-preview
    # --no-wait ==> When --attach-acr and --enable-managed-identity are both specified, --no-wait is not allowed, please wait until the whole operation succeeds.
    --enable-aad --aad-admin-group-object-ids $AKSADM_GRP_ID --aad-tenant-id $tenantId # requires Kubectl client 1.18.x
    #--aad-server-app-id $serverApplicationId \
    #--aad-server-app-secret $serverApplicationSecret \
    #--aad-client-app-id $clientApplicationId \
    #--aad-tenant-id $tenantId


    # below is not yet implemented, see https://github.com/Azure/AKS/issues/1441
    #--enable-addons ingress-appgw \
    #--appgw-name $appgw_agic_name \
    #--appgw-subnet-id $agic_subnet_id \
    #--appgw-watch-namespace $target_namespace \    
    #--enable-addons monitoring --workspace-resource-id $analytics_workspace_id \
    #--node-count 3 \
    # --api-server-authorized-ip-ranges 73.140.245.0/24 \
    # https://docs.microsoft.com/en-us/azure/aks/private-clusters#limitations ==> IP authorized ranges cannot be applied to the private api server endpoint, they only apply to the public API server
    # https://docs.microsoft.com/en-us/azure/aks/api-server-authorized-ip-ranges#create-an-aks-cluster-with-api-server-authorized-ip-ranges-enabled

```

<span style="color:red">/!\ IMPORTANT </span> : do not wait for the end of the cluster creation and continue to ne next sections below 


### Get AKS Credentials

Apply [KubeCtl alias](./tools#kube-tools)

```sh

ls -al ~/.kube
rm  ~/.kube/config

az aks get-credentials --resource-group $rg_name --name $cluster_name --admin

az aks show -n $cluster_name -g $rg_name

aks_api_server_url=$(az aks show -n $cluster_name -g $rg_name --query 'privateFqdn' -o tsv)
echo "AKS API server URL: " $aks_api_server_url

```
### Update firewall rules

Go to the portal and watch for the API server endpoint, get its URL
Go to the [Update firewall rules section](./setup-egress-lockdown.md#update-firewall-rule-after-aks-cluster-creation)


<span style="color:red">/!\ IMPORTANT </span> : Follow [./setup-egress-lockdown.md#update-firewall-rule-after-aks-cluster-creation](./setup-egress-lockdown.md#update-firewall-rule-after-aks-cluster-creation)

### Vnet-Link

```sh
# managed_rg="MC_$rg_name"_"$cluster_name""_""$location"
# echo $managed_rg

managed_rg=$(az aks show --resource-group $rg_name --name $cluster_name --query nodeResourceGroup -o tsv)
echo "CLUSTER_RESOURCE_GROUP:" $managed_rg

aks_node_rg_id=$(az group show --name $managed_rg --query id)
echo $aks_node_rg_id

# AKS_MC_RG=$(az group list --query "[?starts_with(name, 'MC_${rg_name}')].name | [0]" --output tsv)
# ROUTE_TABLE=$(az network route-table list -g ${AKS_MC_RG} --query "[].id | [0]" -o tsv)

# If the snippet below is wrong, do it from the portal: https://docs.microsoft.com/en-us/azure/aks/private-clusters#virtual-network-peering

aks_api_server_url_length=$(echo -n $aks_api_server_url | wc -c)
echo "API server URL length " $aks_api_server_url_length

prv_lnk_url_pattern_length=$(echo -n ".privatelink.${location}.azmk8s.io" | wc -c)
echo "Private link URL pattern length " $prv_lnk_url_pattern_length

url_begin_length=$(echo $(($aks_api_server_url_length - $prv_lnk_url_pattern_length)))
aks_api_server_url_begin="${aks_api_server_url:0:$url_begin_length}"
echo "AKS URL name begins with " $aks_api_server_url_begin

index=$(echo `expr index "$aks_api_server_url_begin" .`)
echo "Index " $index

prv_lnk_zone_guid="${aks_api_server_url_begin:$index:$url_begin_length}"
echo "Private link zone GUID " $prv_lnk_zone_guid

# Much simple and work with Mac : 
prv_lnk_zone_guid=$(echo $aks_api_server_url | cut -d . -f2)

az network private-dns link vnet list -g $managed_rg --zone-name "$prv_lnk_zone_guid.privatelink.${location}.azmk8s.io"

az network private-dns link vnet create \
  --resource-group $managed_rg \
  --zone-name "$prv_lnk_zone_guid.privatelink.${location}.azmk8s.io" \
  --name $aks_private_dns_link_name \
  --virtual-network $bastion_vnet_id \
  --registration-enabled false

private_dns_link_id=$(az network private-dns link vnet show --name $aks_private_dns_link_name --zone-name "$prv_lnk_zone_guid.privatelink.${location}.azmk8s.io" -g $managed_rg --query "id" --output tsv)
echo "AKS Private-Link DNS ID :" $private_dns_link_id

  
```


## Create JumpOff : not needed as you already have one in the Bastion RG/VNet which is peered with AKS VNet

[Your SSH keys should have been generated at pre-req step](./setup-prereq-rg-spn#generates-your-ssh-keys)

```sh

# --ssh-key-values : Space-separated list of SSH public keys or public key file paths.
az vm create --name "aks-jumpoff-vm" \
    --image UbuntuLTS \
    --admin-username "aks-adm" \
    --resource-group $rg_name \
    --vnet-name $vnet_name \
    --subnet $subnet_name \
    --nsg "nsg-management" \
    --size Standard_F2s_v2 \
    --zone 1 \
    --location $location \
    --ssh-key-values ~/.ssh/$ssh_key.pub
    # --generate-ssh-keys

ls -al ~/.ssh/
mv ~/.ssh/id_rsa ~/.ssh/$ssh_key
mv ~/.ssh/id_rsa.pub ~/.ssh/${ssh_key}.pub
ls -al ~/.ssh/

# Option 2 : create a Windows JumpOff
az vm create --name "aks-win-jumpoff" \
    --image Win2019Datacenter \
    --admin-username "aks-adm" \
    --admin-password "DWP4adm101202303404!" \
    --resource-group $rg_name \
    --vnet-name $vnet_name \
    --subnet $subnet_name \
    --nsg "nsg-management" \
    --size Standard_F2s_v2 \ # the sizing should be bigger for Windows
    --zone 1 \
    --location $location

```

## SSH Test

```sh
##############################################################################################################################
#
# Option : Play SSH connection to your node. You should be able to SSH connect into any node in your cluster and curl pod's IP
# https://docs.microsoft.com/en-us/azure/aks/ssh
#
##############################################################################################################################



vmss_name=$(az vmss list -g $managed_rg --query [0].name -o tsv)
echo "VMSS name: " $vmss_name

node0_name=$(az vmss list-instances --name $vmss_name -g $managed_rg --query [0].name -o tsv)
echo "Node0 VM name: " $node0_name

# https://docs.microsoft.com/en-us/cli/azure/vmss/nic?view=azure-cli-latest#az-vmss-nic-list
#node0_IP=$(az vmss nic list --vmss-name $vmss_name -g $managed_rg --query [].{ip:ipConfigurations[0].privateIpAddress})
node0_IP=$(az vmss nic list --vmss-name $vmss_name -g $managed_rg --query [0].ipConfigurations[0].privateIpAddress)
echo "Node0 IP : " $node0_IP

# if SSH to AKS is broken, use the tool below (see https://portal.microsofticm.com/imp/v3/incidents/details/180571943/home)

# try https://github.com/Azure/kubelogin#service-principal-login-flow-non-interactive (replaces https://github.com/mohatb/kubectl-wls or https://github.com/kvaps/kubectl-node-shell)
# instead use https://github.com/Azure/kubelogin
wget https://raw.githubusercontent.com/mohatb/kubectl-wls/master/kubectl-wls
chmod +x ./kubectl-wls
sudo mv ./kubectl-wls /usr/local/bin/kubectl-wls
kubectl-wls

# test
systemctl status kubelet
# http://man.openbsd.org/OpenBSD-current/man1/nc.1
nc -v -u -z -w 3 ntp.ubuntu.com 123
wget www.qwant.com


# https://github.com/MicrosoftDocs/azure-docs/issues/50180
# https://github.com/Azure/azure-cli/issues/12594
az vmss extension set \
    --resource-group $managed_rg \
    --vmss-name $vmss_name \
    --name VMAccessForLinux \
    --publisher Microsoft.OSTCExtensions \
    --protected-settings "{\"username\":\"${aks_admin_username}\", \"ssh_key\":\"$(cat ~/.ssh/${ssh_key}.pub)\"}" \
    --version 1.4 \
    --verbose
    # --debug

az vmss update-instances --instance-ids '*' \
    --resource-group $managed_rg \
    --name $vmss_name

# https://github.com/MicrosoftDocs/azure-docs/issues/50860
az vmss extension set \
    --resource-group $managed_rg \
    --vmss-name $vmss_name \
    --name AADLoginForLinux \
    --publisher Microsoft.Azure.ActiveDirectory.LinuxSSH \
    --verbose

# For Debian : Firewall must allow http://security.debian.org & http://deb.debian.org
kubectl run aks-ssh -it --image=debian --restart=Never -- /bin/sh
apt-get update && apt-get install openssh-client -y

# For Alpine: Firewall must allow http://dl-cdn.alpinelinux.org/alpine/v3.11/community , potentially http://dl-4.alpinelinux.org
kubectl run aks-ssh -it --rm --image=alpine --restart=Never -- /bin/sh
apk update
apk add --no-cache openssh


```

Open a new terminal window, not connected to your container, copy your private SSH key into the helper pod. This private key is used to create the SSH into the AKS node.
If needed, change ~/.ssh/id_rsa to location of your private SSH key:

```sh
cd ~/.ssh
tar -zcvf ssh_keys.tar.gz $ssh_key $ssh_key.pub

scp -i ~/.ssh/$ssh_key ssh_keys.tar.gz $bastion_admin_username@$bastion_public_ip_address:~/.ssh
ssh -i~/.ssh/$ssh_key $bastion_admin_username@$bastion_public_ip_address
tar -zxvf ~/.ssh/ssh_keys.tar.gz

kubectl cp ~/.ssh/${ssh_key} $(kubectl get pod -l run=aks-ssh -o jsonpath='{.items[0].metadata.name}'):/${ssh_key}
```

Return to the terminal session to your container, update the permissions on the copied id_rsa private SSH key so that it is user read-only:
```sh
chmod 0600 ${ssh_key}
```


Return to the terminal session to your Bastion/JumpOff
```sh
node0_name=$(kubectl get nodes -o jsonpath="{.items[0].metadata.name"})
echo "Node0 HostName : " $node0_name

node0_IP=$(kubectl get nodes -o jsonpath="{.items[0].status.addresses[1].address"})
echo "Node0 IP : " $node0_IP

kubectl get nodes -o wide
ssh -i ~/.ssh/${ssh_key} $aks_admin_username@$node0_IP

systemctl status kubelet
# systemctl start kubelet

cat /var/log/azure/cluster-provision.log
sudo cat /etc/kubernetes/azure.json

# TEST
az vmss run-command invoke -g $managed_rg  \
  -n $vmss_name \
  --instance 0 \
  --scripts "hostname && date && cat /etc/kubernetes/azure.json" --command-id RunShellScript


```


## NTP test
```sh
nc -v -u -z -w 3 ntp.ubuntu.com 123
```

## Attach ACR

See [./setup-acr#create-role-assignment](./setup-acr#create-role-assignment)

```sh
# Once the acrpull role is assigned to the AKS idenity , now finally attach ACR to AKS
az aks update -n $cluster_name -g $rg_name --attach-acr $acr_registry_id
# az aks show -g $rg_name -n $cluster_name

```

## Connect to the Cluster

Connect to your bastion / JumpOff, then :
- install the [tools](tools.md)
- reinit your variables

```sh

az aks list -o table
az aks get-credentials --resource-group $rg_name --name $cluster_name --admin
kubectl cluster-info
#kubectl config view

# below is N/A with ManagedIdentities
# Get the id of the service principal configured for AKS
# CLIENT_ID=$(az aks show --resource-group $rg_name --name $cluster_name --query "servicePrincipalProfile.clientId" --output tsv)
# echo "CLIENT_ID:" $CLIENT_ID 


```

## Create Namespaces
```sh
kubectl create namespace development
kubectl label namespace/development purpose=development

kubectl create namespace staging
kubectl label namespace/staging purpose=staging

kubectl create namespace production
kubectl label namespace/production purpose=production

kubectl create namespace sre
kubectl label namespace/sre purpose=sre

kubectl get namespaces
kubectl describe namespace production
kubectl describe namespace sre
```

## Control access to cluster resources using RBAC and Azure Active Directory identities in AKS
```sh

# https://github.com/Azure/AKS/issues/1557 : If the custom vnet resides outside of MC_ resource group, you must manually grant needed permission to the system assigned identity associated with the cluster. That’s because AKS resource provider can’t grant any permission outside of MC_ resource group.
# https://docs.microsoft.com/en-us/azure/aks/use-managed-identity#create-an-aks-cluster-with-managed-identities

PoolIdentityName=$cluster_name"-agentpool"
echo "AKS Agent Pool Identity Name " : $PoolIdentityName

PoolIdentityResourceID=$(az identity show -g $managed_rg --name  $PoolIdentityName --query id --output tsv)
PoolIdentityPrincipalID=$(az identity show -g $managed_rg --name  $PoolIdentityName --query principalId --output tsv)
PoolIdentityClientID=$(az identity show -g $managed_rg --name  $PoolIdentityName --query clientId --output tsv)

echo "AKS Agent Pool Identity Resource ID " : $PoolIdentityResourceID
echo "AKS Agent Pool Identity Principal ID " : $PoolIdentityPrincipalID
echo "AKS Agent Pool Identity Client ID " : $PoolIdentityClientID

az role assignment create --assignee $PoolIdentityPrincipalID --scope $vnet_id --role "Network contributor"
az role assignment create --assignee $PoolIdentityClientID --scope $vnet_id --role "Network contributor"

AzurePolicyIdentityName=azurepolicy-${cluster_name}
echo "AKS Azure Policy Identity Name " : $AzurePolicyIdentityName

AzurePolicyIdentityResourceID=$(az identity show -g $managed_rg --name  $AzurePolicyIdentityName --query id --output tsv)
AzurePolicyIdentityPrincipalID=$(az identity show -g $managed_rg --name  $AzurePolicyIdentityName --query principalId --output tsv)
AzurePolicyIdentityClientID=$(az identity show -g $managed_rg --name  $AzurePolicyIdentityName --query clientId --output tsv)

echo "AKS Azure Policy Identity Resource ID " : $AzurePolicyIdentityResourceID
echo "AKS Azure Policy Identity Principal ID " : $AzurePolicyIdentityPrincipalID
echo "AKS Azure Policy Identity Client ID " : $AzurePolicyIdentityClientID


# https://docs.microsoft.com/en-us/azure/aks/azure-ad-rbac
# https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-identity#use-role-based-access-controls-rbac 

aks_cluster_id=$(az aks show -n $cluster_name -g $rg_name --query id -o tsv)
echo "AKS cluster ID : " $aks_cluster_id

# Create the first example group in Azure AD for the application developers
APPDEV_ID=$(az ad group create --display-name appdev-${appName} --mail-nickname appdev --query objectId -o tsv)
echo "APPDEV GROUP ID: " $APPDEV_ID
# APPDEV_ID=$(az ad group show --group appdev --query objectId -o tsv)
az role assignment create --assignee $APPDEV_ID --role "Azure Kubernetes Service Cluster User Role" --scope $aks_cluster_id

# Create a second example group, this one for SREs named opssre
OPSSRE_ID=$(az ad group create --display-name opssre-${appName} --mail-nickname opssre-${appName} --query objectId -o tsv)
echo "OPSSRE GROUP ID: " $OPSSRE_ID
# OPSSRE_ID=$(az ad group show --group opssre --query objectId -o tsv)
az role assignment create --assignee $OPSSRE_ID --role "Azure Kubernetes Service Cluster User Role" --scope $aks_cluster_id

# Create a user for the Dev role
AKSDEV_ID=$(az ad user create --display-name "AKS Dev ${appName}" --user-principal-name "aksdev@groland.grd" --password P@ssw0rd1 --query objectId -o tsv)
echo "AKS DEV USER ID: " $AKSDEV_ID
# Add the user to the appdev Azure AD group
az ad group member add --member-id $AKSDEV_ID --group appdev-${appName} 

# Create a user for the SRE role
AKSSRE_ID=$(az ad user create --display-name "AKS SRE ${appName}" --user-principal-name "akssre@groland.grd" --password P@ssw0rd1 --query objectId -o tsv)
echo "AKS SRE USER ID: " $AKSSRE_ID
# Add the user to the opssre Azure AD group
az ad group member add --member-id $AKSSRE_ID --group opssre-${appName} 

# https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-groups
# https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/

kubectl apply -f ./cnf/role-dev-namespace.yaml
export DEV_GROUP_OBECT_ID=$APPDEV_ID
envsubst < ./cnf/rolebinding-dev-namespace.yaml > deploy/rolebinding-dev-namespace.yaml
kubectl apply -f deploy/rolebinding-dev-namespace.yaml
k describe role dev-user-full-access -n development
k describe rolebindings dev-user-access -n development

kubectl apply -f ./cnf/role-sre-namespace.yaml
export SRE_GROUP_OBECT_ID=$OPSSRE_ID
envsubst < ./cnf/rolebinding-sre-namespace.yaml > deploy/rolebinding-sre-namespace.yaml
kubectl apply -f deploy/rolebinding-sre-namespace.yaml

export AKS_ADM_GROUP_OBECT_ID=$AKSADM_GRP_ID
envsubst < ./cnf/azure-ad-binding.yaml > deploy/azure-ad-binding.yaml
kubectl apply -f deploy/azure-ad-binding.yaml

k get clusterrolebindings -A
k describe clusterrolebindings owner-cluster-admin
k describe clusterrolebindings aks-cluster-admin-binding
k describe clusterrolebindings aks-cluster-admin-binding-aad
k get clusterroles -A
k describe clusterrole admin
k describe clusterrole cluster-admin
k describe clusterrole policy-agent

# Reset the kubeconfig context using the az aks get-credentials command. In a previous section, you set the context using the cluster admin credentials. 
# The admin user bypasses Azure AD sign in prompts. Without the --admin parameter, the user context is applied that requires all requests to be authenticated using Azure AD.
az aks get-credentials --name $cluster_name -g $rg_name --overwrite-existing
kubectl run --restart=Never nginx-dev --image=nginx --namespace development
kubectl get pods --namespace development
kubectl get pods --all-namespaces

kubectl run --restart=Never nginx-dev --image=nginx --namespace sre
# Error from server (Forbidden): pods is forbidden: User "aksdev@contoso.com" cannot create resource "pods" in API group "" in the namespace "sre"

# Now Test the SRE access to the AKS cluster resources
az aks get-credentials --name $cluster_name -g $rg_name --overwrite-existing
kubectl run --restart=Never nginx-sre --image=nginx --namespace sre
kubectl get pods --namespace sre
kubectl get pods --all-namespaces
kubectl run --restart=Never nginx-sre --image=nginx --namespace development

```



## Optionnal Play: what resources are in your cluster

```sh
kubectl get nodes

# https://docs.microsoft.com/en-us/azure/aks/availability-zones#verify-node-distribution-across-zones
# https://docs.microsoft.com/en-us/azure/aks/availability-zones#verify-pod-distribution-across-zones
kubectl describe nodes | grep -e "Name:" -e "failure-domain.beta.kubernetes.io/zone"

kubectl get pods
kubectl top node
kubectl api-resources --namespaced=true
kubectl api-resources --namespaced=false

kubectl get roles --all-namespaces
kubectl get serviceaccounts --all-namespaces
kubectl get rolebindings --all-namespaces
kubectl get ingresses  --all-namespaces
```

## Create Analytics Workspace


If you get an error like "The workspace name 'log-secgov-analytics-wks' is not unique" ==>  the workspace is in soft-delete state and its name remains reserved for 14d. We added the option to delete workspace permanently, but this is supported via API currently. Learn more in [this article](https://docs.microsoft.com/en-us/azure/azure-monitor/platform/delete-workspace)

See also
- [https://github.com/Azure/azure-cli/issues/11916](https://github.com/Azure/azure-cli/issues/11916)
- [DELETE API](https://docs.microsoft.com/en-us/rest/api/loganalytics/workspaces/delete)

```sh

nameX=$(az group deployment list -g $rg_name --verbose --query [].name)
echo "nameX:" $nameX | grep -i $analytics_workspace_name

az deployment group show -n $analytics_workspace_name -g $rg_name --verbose

DELETE https://management.azure.com/subscriptions/<subscription-id>/resourcegroups/<resource-group-name>/providers/Microsoft.OperationalInsights/workspaces/<workspace-name>?api-version=2015-11-01-preview&force=true
Authorization: Bearer eyJ0eXAiOiJKV1Qi….

```

```sh
# https://docs.microsoft.com/en-us/azure/azure-monitor/learn/quick-create-workspace-cli
# /!\ ATTENTION : check & modify location in the JSON template from https://docs.microsoft.com/en-us/azure/azure-monitor/learn/quick-create-workspace-cli#create-and-deploy-template
# https://docs.microsoft.com/en-us/cli/azure/monitor/log-analytics/workspace?view=azure-cli-latest#az-monitor-log-analytics-workspace-create

# You can use `VIM <file you want to edit>` in Azure Cloud Shell to open the built-in text editor.
# You can upload files to the Azure Cloud Shell by dragging and dropping them
# You can also do a `curl -o filename.ext https://file-url/filename.ext` to download a file from the internet.

# az deployment group create --name $analytics_workspace_name --parameters=workspaceName=$analytics_workspace_name -g $rg_name --template-file $analytics_workspace_template 

az monitor log-analytics workspace create -n $analytics_workspace_name --location $location -g $rg_name --verbose
az monitor log-analytics workspace list
az monitor log-analytics workspace show -n $analytics_workspace_name -g $rg_name --verbose

# -o tsv to manage quotes issues
analytics_workspace_id=$(az monitor log-analytics workspace show -n $analytics_workspace_name -g $rg_name --query id -o tsv)
echo "analytics_workspace_id:" $analytics_workspace_id

# https://github.com/Azure/azure-cli/issues/9228
# https://github.com/Azure/azure-cli/blob/master/doc/use_cli_effectively.md#quoting-issues

az aks enable-addons --addons monitoring -g $rg_name --name $cluster_name --workspace-resource-id $analytics_workspace_id

```