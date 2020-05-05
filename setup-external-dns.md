# Setup External-DNS


See also :
- [https://github.com/kubernetes-sigs/external-dns/blob/master/docs/faq.md](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/faq.md)
- [https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/azure.md#azure-managed-service-identity-msi](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/azure.md#azure-managed-service-identity-msi)
- [https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/azure-private-dns.md](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/azure-private-dns.md)
- [https://github.com/kubernetes-sigs/external-dns/issues/1456](https://github.com/kubernetes-sigs/external-dns/issues/1456)
- [https://github.com/kubernetes-sigs/external-dns/issues/1510](https://github.com/kubernetes-sigs/external-dns/issues/1510)
- [https://github.com/bitnami/charts/issues/2311](https://github.com/bitnami/charts/issues/2311)
- [https://github.com/helm/charts/tree/master/stable/external-dns](https://github.com/helm/charts/tree/master/stable/external-dns)

## Pre-requisites
```sh
echo -e "\""tenantId\"": \""$tenantId\""\n"\
"\""subscriptionId\"": \""$subId\""\n"\
"\""resourceGroup\"": \""$rg_name\""\n"\
"\""useManagedIdentityExtension\"": \""true\""\n"\
> external-dns-mi-azure.json

cat external-dns-mi-azure.json
k create secret generic ext-dns-cnf --from-file=external-dns-mi-azure.json -n $target_namespace
k get secrets -n $target_namespace
k describe secret ext-dns-cnf  -n $target_namespace

# Configure role assigment
rg_id=$(az group show --name $rg_name --query id)
echo "RG ID : "  $rg_id

CLIENT_ID=$(az aks show --resource-group $rg_name --name $cluster_name --query "servicePrincipalProfile.clientId" --output tsv)
echo "CLIENT_ID:" $CLIENT_ID 

PoolIdentityName=$cluster_name"-agentpool"
echo "AKS Agent Pool Identity Name " : $PoolIdentityName

PoolIdentityResourceID=$(az identity show -g $managed_rg --name  $PoolIdentityName --query id --output tsv)
PoolIdentityPrincipalID=$(az identity show -g $managed_rg --name  $PoolIdentityName --query principalId --output tsv)
PoolIdentityClientID=$(az identity show -g $managed_rg --name  $PoolIdentityName --query clientId --output tsv)

echo "AKS Agent Pool Identity Resource ID " : $PoolIdentityResourceID
echo "AKS Agent Pool Identity Principal ID " : $PoolIdentityPrincipalID
echo "AKS Agent Pool Identity Client ID " : $PoolIdentityClientID

#az role assignment create --assignee $PoolIdentityPrincipalID --scope $rg_id --role "Reader"
#az role assignment create --assignee $PoolIdentityClientID --scope $rg_id --role "Reader"

az role assignment create --assignee $PoolIdentityClientID --scope $subnet_id --role "Reader"

# 2. as a contributor to DNS Zone itself
dns_zone_id=$(az network dns zone show --name $app_dns_zone -g $rg_name --query id --output tsv)
echo "DNS Zone ID" $dns_zone_id
az role assignment create --role "Contributor" --assignee $PoolIdentityClientID --scope $dns_zone_id

dns_zone_id=$(az network dns zone show --name $custom_dns -g $rg_name --query id --output tsv)
echo "DNS Zone ID" $dns_zone_id
az role assignment create --role "Contributor" --assignee $PoolIdentityClientID --scope $dns_zone_id

```

## Create Pod identity

See [https://github.com/kubernetes-sigs/external-dns/issues/1456](https://github.com/kubernetes-sigs/external-dns/issues/1456)

```sh
export IDENTITY_NAME="ext-dns-pod-identity" #  must consist of lower case 

# https://github.com/Azure/aad-pod-identity/pull/48
# https://github.com/Azure/aad-pod-identity/issues/38
az identity create -n $IDENTITY_NAME -g $rg_name
export IDENTITY_CLIENT_ID="$(az identity show -g $rg_name -n $IDENTITY_NAME --query clientId -otsv)"
export IDENTITY_RESOURCE_ID="$(az identity show -g $rg_name -n $IDENTITY_NAME --query id -otsv)"
export IDENTITY_ASSIGNMENT_ID="$(az role assignment create --role Reader --assignee $IDENTITY_CLIENT_ID --scope /subscriptions/$subId/resourceGroups/$rg_name --query id -o tsv)"

export ResourceID=$IDENTITY_RESOURCE_ID
export ClientID=$IDENTITY_CLIENT_ID
envsubst < ./cnf/external-dns-pod-identity.yaml > deploy/external-dns-pod-identity.yaml

k apply -f external-dns-pod-identity.yaml -n $target_namespace
k get azureidentitybindings -A
k get azureassignedidentities -A
```


## Use Manifest 
for clusters with [RBAC enabled, namespace access](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/azure.md#manifest-for-clusters-with-rbac-enabled-namespace-access)

```sh
export DOMAIN_FITER=$custom_dns
export EXT_DNS_RG=$rg_name
export ResourceID=$IDENTITY_RESOURCE_ID
export ClientID=$IDENTITY_CLIENT_ID
envsubst < ./cnf/external-dns.yaml > deploy/external-dns.yaml
k create -f deploy/external-dns.yaml -n $target_namespace

# k delete deployment external-dns -n $target_namespace
# k delete sa external-dns -n $target_namespace
# k delete role external-dns -n $target_namespace
# k delete rolebinding external-dns -n $target_namespace

k get rolebindings -n $target_namespace
k get roles -n $target_namespace
k get sa -n $target_namespace
k get deployments -n $target_namespace
k get deployment external-dns -n $target_namespace
k describe deployment external-dns -n $target_namespace
k get pods -n $target_namespace
for pod in $(k get pods -n $target_namespace -l app=external-dns -o custom-columns=:metadata.name)
do
	k describe pod $pod -n $target_namespace # | grep -i "Error"
	k logs $pod -n $target_namespace | grep -i "Error"
done

k get events -n $target_namespace | grep -i "Error" 
```

## Create Ingress 

See :
1. [build_java_app.md](./build_java_app.md#create-kubernetes-internal-service)

## Verifying Azure DNS records
```sh
az network dns zone list -g $rg_name
az network dns record-set list -g $rg_name -z $custom_dns
az network dns record-set a list -g $rg_name -z $custom_dns
az network dns record-set list -g $rg_name -z $app_dns_zone
az network dns record-set a list -g $rg_name -z $app_dns_zone

```