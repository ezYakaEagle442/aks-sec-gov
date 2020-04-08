# Private  Application Gateway & Application Gateway Ingress Controller with private IP for Ingress endpoint internal routing

Note: this should be soon useless as a new feature will enable AGIC to be installed easily as an AKS add-on at cluster creation time : \
    --enable-addons ingress-appgw \
    --appgw-name $appgw_agic_name \
    --appgw-subnet-id $agic_subnet_id \
    --appgw-watch-namespace $target_namespace


see [Issue #1441](https://github.com/Azure/AKS/issues/1441)



AGIC supports [private IP](https://docs.microsoft.com/en-us/azure/application-gateway/ingress-controller-private-ip) for internal routing for an Ingress endpoint, in such case the pre-requisites is Application Gateway with a [Private IP configuration](https://docs.microsoft.com/en-us/azure/application-gateway/configure-application-gateway-with-private-frontend-ip)

Azure Application Gateway can be configured with an Internet-facing VIP or with an internal endpoint that isn't exposed to the Internet. An internal endpoint uses a private IP address for the frontend, which is also known as an internal load balancer (ILB) endpoint. 

==> even with private IP , a public IP is provisioned ?!

Issue #[1441](https://github.com/Azure/AKS/issues/1441). See also this [session + slide deck from Ignite](https://myignite.techcommunity.microsoft.com/sessions/82945) .
With default settings, AGIC assumes [100% ownership of the App Gateway it is pointed to](https://github.com/Azure/application-gateway-kubernetes-ingress/blob/master/docs/setup/install-existing.md#multi-cluster--shared-app-gateway). AGIC overwrites all of App Gateway's configuration. If we were to manually create a listener for prod.contoso.com (on App Gateway), without defining it in the Kubernetes. See also [How-To Minimizing Downtime During Deployments](https://github.com/Azure/application-gateway-kubernetes-ingress/blob/master/docs/how-tos/minimize-downtime-during-deployments.md).


## Create Azure App Gateway v2
```sh
az network application-gateway waf-policy create \
  --name "${appName}-waf-policy" \
  --resource-group $rg_name

waf_policy_id=$(az network application-gateway waf-policy show --name "${appName}-waf-policy" -g $rg_name --query id -o tsv)
echo "WAF Policy id " $waf_policy_id

az network application-gateway waf-policy policy-setting update --policy-name "${appName}-waf-policy" --mode Detection --state Enabled -g  $rg_name

az network public-ip create --name $appgw_agic_IP \
                            --location $location \
                            --allocation-method Static \
                            --dns-name $appgw_agic_DNS_name \
                            -g $rg_name \
                            --subscription $subId \
                            --sku Standard \
                            --version IPv4

public_agic_ip_id=$(az network public-ip show --name $appgw_IP --query id -o tsv --subscription $subId -g $rg_name)
echo $public_agic_ip_id

appgw_agic_public_ip_address=$(az network public-ip show --name $appgw_IP --query ipAddress -o tsv --subscription $subId -g $rg_name)
echo $appgw_agic_public_ip_address

az network application-gateway create --name $appgw_agic_name \
                                      --resource-group $rg_name \
                                      --capacity 1 --min-capacity 1 --max-capacity 3  \
                                      --connection-draining-timeout 1 \
                                      --http-settings-cookie-based-affinity Enabled \
                                      --http-settings-port 443 --frontend-port 80 \
                                      --http-settings-protocol Https \
                                      --http2 Enabled \
                                      --location $location \
                                      --private-ip-address 172.16.2.42 \
                                      --public-ip-address $appgw_agic_IP \
                                      --routing-rule-type Basic \
                                      --sku WAF_v2 \
                                      --subnet $agic_subnet_id \
                                      --subscription $subId \
                                      --waf-policy "${appName}-waf-policy"
                                      # --zones 1 2 3 \

```

## Create AAD Pod Identity

See the docs below :
- [https://github.com/Azure/aad-pod-identity](https://github.com/Azure/aad-pod-identity)
- [https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-identity#use-pod-identities](https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-identity#use-pod-identities)

```sh




```

## setup AGIC 
```sh
# https://github.com/Azure/application-gateway-kubernetes-ingress/blob/master/docs/setup/install-new.md

helm repo add application-gateway-kubernetes-ingress https://appgwingress.blob.core.windows.net/ingress-azure-helm-package/
helm repo update

# Install and Setup Ingress
# Grant AAD Identity Access to App Gateway

APPGATEWAYSCOPEID=$(az network application-gateway show -g $rg_name -n $appgw_agic_name | jq .id | tr -d '"')
echo $APPGATEWAYSCOPEID
ROLEAGWCONTRIB=$(az role assignment create --role Contributor --assignee $ASSIGNEEID --scope $APPGATEWAYSCOPEID)
ROLEAGWREADER=$(az role assignment create --role Reader --assignee $ASSIGNEEID --scope "/subscriptions/${subId}/resourcegroups/${rg_name}")
ROLEAGWREADER2=$(az role assignment create --role Reader --assignee $ASSIGNEEID --scope $APPGATEWAYSCOPEID)
echo $ROLEAGWCONTRIB
echo $ROLEAGWREADER
echo $ROLEAGWREADER2

# Note: Update subscriptionId, resourceGroup, name, identityResourceID, identityClientID and apiServerAddress in agw-helm-config.yaml file with the following.
sed -i agw-helm-config.yaml -e "s/\(^.*subscriptionId: \).*/\1${subId}/gI"
sed -i agw-helm-config.yaml -e "s/\(^.*resourceGroup: \).*/\1${rg_name}/gI"
sed -i agw-helm-config.yaml -e "s/\(^.*name: \).*/\1${appgw_agic_name}/gI"
sed -i agw-helm-config.yaml -e 's@\(^.*identityResourceID: \).*@\1'"${SCOPEID}"'@gI'
sed -i agw-helm-config.yaml -e "s/\(^.*identityClientID: \).*/\1${ASSIGNEEID}/gI"

APISERVER=$(az aks show -n $cluster_name -g $rg_name --query 'fqdn' -o tsv)
sed -i agw-helm-config.yaml -e "s/\(^.*apiServerAddress: \).*/\1${APISERVER}/gI"

# Install App GW Ingress Controller with Helm
helm install --name $appgw_agic_name -f agw-helm-config.yaml application-gateway-kubernetes-ingress/ingress-azure

# Check created resources
k get po,svc,ingress,deploy,secrets
helm list

```