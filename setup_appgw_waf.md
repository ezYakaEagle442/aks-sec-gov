
# Setup Private  Application Gateway  with private IP for Ingress endpoint internal routing

See :
- [https://docs.microsoft.com/en-us/azure/application-gateway/features](https://docs.microsoft.com/en-us/azure/application-gateway/features)
- [https://github.com/Azure/azure-quickstart-templates/tree/master/101-application-gateway-waf](https://github.com/Azure/azure-quickstart-templates/tree/master/101-application-gateway-waf) _
- [https://docs.microsoft.com/en-us/cli/azure/network/application-gateway?view=azure-cli-latest#az-network-application-gateway-create](https://docs.microsoft.com/en-us/cli/azure/network/application-gateway?view=azure-cli-latest#az-network-application-gateway-create)
- [https://docs.microsoft.com/en-us/azure/application-gateway/create-ssl-portal](https://docs.microsoft.com/en-us/azure/application-gateway/create-ssl-portal)
- [https://docs.microsoft.com/en-us/azure/application-gateway/key-vault-certs](https://docs.microsoft.com/en-us/azure/application-gateway/key-vault-certs)
- [https://docs.microsoft.com/en-us/azure/application-gateway/application-gateway-autoscaling-zone-redundant#scaling-application-gateway-and-waf-v2](https://docs.microsoft.com/en-us/azure/application-gateway/application-gateway-autoscaling-zone-redundant#scaling-application-gateway-and-waf-v2)

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
                                      --zones 1 2 3 \
                                      --private-ip-address 172.42.1.42 \
                                      --public-ip-address $appgw_IP \
                                      --routing-rule-type Basic \
                                      --sku WAF_v2 \
                                      --subnet $appgw_subnet_id \
                                      --subscription $subId \
                                      --waf-policy "${appName}-waf-policy"
                                      # --key-vault-secret-id  $appgw_vault_secret_name  # https://docs.microsoft.com/en-us/azure/application-gateway/key-vault-certs
                                      # [--identity]
                                      # --vnet-name $appgw_subnet_name 


# 
az network application-gateway address-pool update \
 --servers <EXTERNAL_IP> \ # TODO get an IP from AKS
 --gateway-name $appgw_name \
 -g $rg_name \
 -n appGatewayBackendPool

```