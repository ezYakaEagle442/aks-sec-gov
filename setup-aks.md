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
```sh

# small & cheap VM size : Basic_A1 or Standard_B1s or Standard_F2s_v2
az aks create --name $cluster_name \
    --resource-group $rg_name \
    --service-principal $sp_id \
    --client-secret $sp_password \
    --attach-acr $acr_registry_name \
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
    # requires Azure CLI, version 2.2.0 or later : https://docs.microsoft.com/en-us/azure/aks/use-managed-identity , and also  0.4.38 of the preview cli az extension update --name aks-preview
    # --no-wait ==> When --attach-acr and --enable-managed-identity are both specified, --no-wait is not allowed, please wait until the whole operation succeeds.
    --enable-aad --aad-admin-group-object-ids $AKSADM_GRP_ID --aad-tenant-id $tenantId
    # --aad-server-app-id $serverApplicationId \
    # --aad-server-app-secret $serverApplicationSecret \
    # --aad-client-app-id $clientApplicationId \
    # --aad-tenant-id $tenantId \


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

# Update firewall rules

Go to the portal and watch for the API server endpoint, get its URL

Apply [KubeCtl alias](./tools#kube-tools)

```sh

ls -al ~/.kube
rm  ~/.kube/config

az aks get-credentials --resource-group $rg_name --name $cluster_name --admin

aks_api_server_url=$(az aks show -n $cluster_name -g $rg_name --query 'privateFqdn' -o tsv)
echo "AKS API server URL: " $aks_api_server_url

az aks show -n $cluster_name -g $rg_name

```

<span style="color:red">/!\ IMPORTANT </span> : Follow [./setup-egress-lockdown.md#update-firewall-rule-after-aks-cluster-creation](./setup-egress-lockdown.md#update-firewall-rule-after-aks-cluster-creation)

# Vnet peering

```sh
# managed_rg="MC_$rg_name"_"$cluster_name""_""$location"
# echo $managed_rg

managed_rg=$(az aks show --resource-group $rg_name --name $cluster_name --query nodeResourceGroup -o tsv)
echo "CLUSTER_RESOURCE_GROUP:"$managed_rg


aks_node_rg_id=$(az group show --name $managed_rg --query id)
echo $aks_node_rg_id

# AKS_MC_RG=$(az group list --query "[?starts_with(name, 'MC_${rg_name}')].name | [0]" --output tsv)
# ROUTE_TABLE=$(az network route-table list -g ${AKS_MC_RG} --query "[].id | [0]" -o tsv)

# https://docs.microsoft.com/en-us/azure/aks/private-clusters#virtual-network-peering
# ex: aks-privat-rg-secgov-eastus-7b5f97-f49ffa15.713705ba-88d9-47cc-8d1b-ce03e4df9405.privatelink.eastus2.azmk8s.io
# 8833cc9a-99ec-4630-8b3c-786b59ef4507.privatelink.eastus2.azmk8s.io

# The snippet below is wrong, do it from the portal
az network private-dns link vnet list -g $rg_name --zone-name "<GUID>.privatelink.${location}.azmk8s.io

az network private-dns link vnet create \
  --resource-group $managed_rg \
  --zone-name "<GUID>.privatelink.${location}.azmk8s.io" \
  --name $aks_private_dns_link_name \
  --virtual-network $vnet_bastion_name \
  --registration-enabled false

private_dns_link_id=$(az network private-dns link vnet show --name $aks_private_dns_link_name --zone-name "<GUID>.privatelink.${location}.azmk8s.io" -g $managed_rg --query "id" --output tsv)
echo "AKS Private-Link DNS ID :" $private_dns_link_id

  
```


# Create JumpOff : not needed as you already have one in the Bastion RG/VNet which is peered with AKS VNet

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

# SSH Test

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

# try https://github.com/mohatb/kubectl-wls or https://github.com/kvaps/kubectl-node-shell 
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

kubectl run aks-ssh -it --image=debian --restart=Never -- /bin/sh
# Firewall must allow http://security.debian.org & http://deb.debian.org
apt-get update && apt-get install openssh-client -y

```

Open a new terminal window, not connected to your container, copy your private SSH key into the helper pod. This private key is used to create the SSH into the AKS node.
If needed, change ~/.ssh/id_rsa to location of your private SSH key:

```sh
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

```


# NTP test
```sh
nc -v -u -z -w 3 ntp.ubuntu.com 123
```


# Connect to the Cluster

```sh

az aks list -o table
az aks get-credentials --resource-group $rg_name --name $cluster_name --admin
kubectl cluster-info
#kubectl config view

# Get the id of the service principal configured for AKS
CLIENT_ID=$(az aks show --resource-group $rg_name --name $cluster_name --query "servicePrincipalProfile.clientId" --output tsv)
echo "CLIENT_ID:" $CLIENT_ID 
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
# https://docs.microsoft.com/en-us/azure/aks/azure-ad-rbac

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

kubectl apply -f role-dev-namespace.yaml
kubectl apply -f rolebinding-dev-namespace.yaml

kubectl apply -f role-sre-namespace.yaml
kubectl apply -f rolebinding-sre-namespace.yaml

# Reset the kubeconfig context using the az aks get-credentials command. In a previous section, you set the context using the cluster admin credentials. 
# The admin user bypasses Azure AD sign in prompts. Without the --admin parameter, the user context is applied that requires all requests to be authenticated using Azure AD.
az aks get-credentials --name $cluster_name -g $rg_name --overwrite-existing
kubectl run --generator=run-pod/v1 nginx-dev --image=nginx --namespace development
kubectl get pods --namespace development
kubectl get pods --all-namespaces

kubectl run --generator=run-pod/v1 nginx-dev --image=nginx --namespace sre
# Error from server (Forbidden): pods is forbidden: User "aksdev@contoso.com" cannot create resource "pods" in API group "" in the namespace "sre"

# Now Test the SRE access to the AKS cluster resources
az aks get-credentials --name $cluster_name -g $rg_name --overwrite-existing
kubectl run --generator=run-pod/v1 nginx-sre --image=nginx --namespace sre
kubectl get pods --namespace sre
kubectl get pods --all-namespaces
kubectl run --generator=run-pod/v1 nginx-sre --image=nginx --namespace dev

```



## Optionnal Play: what resources are in your cluster

```sh
kubectl get nodes

# https://docs.microsoft.com/en-us/azure/aks/availability-zones#verify-node-distribution-across-zones
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

analytics_workspace_id=$(az monitor log-analytics workspace show -n $analytics_workspace_name -g $rg_name --query id)
echo "analytics_workspace_id:" $analytics_workspace_id

# https://github.com/Azure/azure-cli/issues/9228
# https://github.com/Azure/azure-cli/blob/master/doc/use_cli_effectively.md#quoting-issues

az aks enable-addons --addons monitoring -g $rg_name --name $cluster_name --workspace-resource-id $analytics_workspace_id

```