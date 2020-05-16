
# Setup Private  Application Gateway  with private IP for Ingress endpoint internal routing

See :
- [https://docs.microsoft.com/en-us/azure/application-gateway/features](https://docs.microsoft.com/en-us/azure/application-gateway/features)
- [https://github.com/Azure/azure-quickstart-templates/tree/master/101-application-gateway-waf](https://github.com/Azure/azure-quickstart-templates/tree/master/101-application-gateway-waf) _
- [https://docs.microsoft.com/en-us/cli/azure/network/application-gateway?view=azure-cli-latest#az-network-application-gateway-create](https://docs.microsoft.com/en-us/cli/azure/network/application-gateway?view=azure-cli-latest#az-network-application-gateway-create)
- [https://docs.microsoft.com/en-us/azure/application-gateway/create-ssl-portal](https://docs.microsoft.com/en-us/azure/application-gateway/create-ssl-portal)
- [https://docs.microsoft.com/en-us/azure/application-gateway/key-vault-certs](https://docs.microsoft.com/en-us/azure/application-gateway/key-vault-certs)
- [https://docs.microsoft.com/en-us/azure/application-gateway/application-gateway-autoscaling-zone-redundant#scaling-application-gateway-and-waf-v2](https://docs.microsoft.com/en-us/azure/application-gateway/application-gateway-autoscaling-zone-redundant#scaling-application-gateway-and-waf-v2)
- [Pricing](https://azure.microsoft.com/en-us/pricing/details/application-gateway)

Azure Application Gateway can be used as an internal application load balancer or as an internet-facing application load balancer. An internet-facing application gateway uses public IP addresses. The DNS name of an internet-facing application gateway is publicly resolvable to its public IP address. As a result, internet-facing application gateways can route client requests to the internet. \

Internal application gateways use only private IP addresses. If you are using a Custom or Private DNS zone, the domain name should be internally resolvable to the private IP address of the Application Gateway. Therefore, internal load-balancers can only route requests from clients with access to a virtual network for the application gateway.

**It can take up to 30 minutes for Azure to create the application gateway.**

***Note:***
Application Gateway V1 supports public, private and public+private configuration. \
Application Gateway V2 supports only public or public+private configuration. 
Customers can lock the public IP from external traffic by not creating listeners on public IP which keeps all ports close. 
The management/control plane ports are required to be open on the public IP and can be locked down by using GatewayManager service tag.

Only private IP configuration is in the roadmap , no ETA available at this time.


```sh

# az keyvault secret set --name $appgw_vault_secret_name --value $appgw_vault_secret --description "AKS App. Gateway ${appName} Secret" --vault-name $vault_name
# az keyvault secret list --vault-name $vault_name
# az keyvault secret show --vault-name $vault_name --name $appgw_vault_secret_name --output tsv

# Application Gateway without Public IP ==> Supported SKU tiers are Standard,WAF. ==> WAF_v2 NOT SUPPORTED
az network public-ip create --name $appgw_IP \
                            --location $location \
                            --allocation-method Static \
                            --dns-name $appgw_DNS_name \
                            -g $rg_name \
                            --subscription $subId \
                            --sku Standard \
                            --version IPv4

public_ip_id=$(az network public-ip show --name $appgw_IP --query id -o tsv --subscription $subId -g $rg_name)
echo $public_ip_id

appgw_public_ip_address=$(az network public-ip show --name $appgw_IP --query ipAddress -o tsv --subscription $subId -g $rg_name)
echo $appgw_public_ip_address

# Create the WAF Policy : https://docs.microsoft.com/en-us/azure/web-application-firewall/ag/create-waf-policy-ag#configure-waf-rules-optional
az network application-gateway waf-policy create \
  --name "${appName}-waf-policy" \
  --resource-group $rg_name

waf_policy_id=$(az network application-gateway waf-policy show --name "${appName}-waf-policy" -g $rg_name --query id -o tsv)
echo "WAF Policy id " $waf_policy_id

az network application-gateway waf-policy policy-setting update --policy-name "${appName}-waf-policy" --mode Detection --state Enabled -g  $rg_name

az network application-gateway create --name $appgw_name \
                                      --resource-group $rg_name \
                                      --capacity 1 --min-capacity 1 --max-capacity 3  \
                                      --connection-draining-timeout 1 \
                                      --http-settings-cookie-based-affinity Enabled \
                                      --http-settings-port 443 --frontend-port 80 \
                                      --http-settings-protocol Https \
                                      --http2 Enabled \
                                      --location $location \
                                      --private-ip-address 172.44.0.42 \
                                      --public-ip-address $appgw_IP \
                                      --routing-rule-type Basic \
                                      --sku WAF_v2 \
                                      --subnet $appgw_subnet_id \
                                      --subscription $subId \
                                      --waf-policy "${appName}-waf-policy"
                                      # --zones 1 2 3 \
                                      # --key-vault-secret-id  $appgw_vault_secret_name  # https://docs.microsoft.com/en-us/azure/application-gateway/key-vault-certs
                                      # [--identity]
                                      # --vnet-name $appgw_subnet_name 

```

```sh
# TODO get an IP from AKS internal Ingress Controller
az network application-gateway address-pool update \
 --servers $ing_ctl_ip \
 --gateway-name $appgw_name \
 -g $rg_name \
 -n appGatewayBackendPool

# update DNS: add a record set pointing to $ing_ctl_ip
az network dns zone list -g $rg_name
az network dns record-set a add-record -g $rg_name -z $custom_dns -n appgw -a $appgw_public_ip_address --ttl 300 # (300s = 5 minutes)
az network dns record-set list -g $rg_name -z $custom_dns
```

create an Internal Ingress Controller, see [./build_java_app.md#create-petclinic-internal-ingress](./build_java_app.md#create-petclinic-internal-ingress)

```sh
k create namespace internal-ingress

helm install internal-ingress stable/nginx-ingress \
    --namespace internal-ingress \
    -f internal-ingress.yaml \
    --set controller.replicaCount=2 \
    --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux \
    --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux

helm ls --namespace internal-ingress
k get services -n internal-ingress -o wide internal-ingress-nginx-ingress-controller -w
ing_ctl_ip=$(k get svc -n internal-ingress internal-ingress-nginx-ingress-controller -o jsonpath="{.status.loadBalancer.ingress[*].ip}")
```

```sh
# Test with an internal Ingress  
export ING_HOST="appgw."$custom_dns
echo "INGRESS HOST " $ING_HOST
envsubst < java-app/petclinic-internal-ingress.yaml > deploy/petclinic-internal-ingress.yaml 
cat deploy/petclinic-internal-ingress.yaml 
k apply -f deploy/petclinic-internal-ingress.yaml -n $target_namespace
k get ingresses --all-namespaces
k get ing internal-petclinic -n $target_namespace -o json
k describe ingress internal-petclinic -n $target_namespace
k get events -n $target_namespace | grep -i "Error"

# Test inside AKS
k run -it --rm aks-ingress-test --image=alpine --restart=Never --namespace internal-ingress
apk update && apk add curl
curl -L http://172.44.0.42
exit

#test from b browser at http://appgw.akshandsonlabs.com/

```