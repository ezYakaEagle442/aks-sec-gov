# Filter outbound flows

Read [https://docs.microsoft.com/en-us/azure/firewall/rule-processing](https://docs.microsoft.com/en-us/azure/firewall/rule-processing)

## setup egress traffic lockdown

- [https://docs.microsoft.com/en-us/azure/aks/limit-egress-traffic](https://docs.microsoft.com/en-us/azure/aks/limit-egress-traffic)
- [https://docs.microsoft.com/en-us/azure/aks/egress-outboundtype](https://docs.microsoft.com/en-us/azure/aks/egress-outboundtype)

MCR Endpoint evolution: 
- [https://aka.ms/acr/cmk](https://aka.ms/acr/cmk)
- [https://aka.ms/mcr/firewallrules](https://aka.ms/mcr/firewallrules)

The ACR team has changed their CDN endpoint from *.cdn.mscr.io to *.data.mcr.microsoft.com effective 3rd March 2020. The details are captured on [this GH page](https://github.com/microsoft/containerregistry/blob/master/client-firewall-rules.md). 
See also [https://github.com/Azure/AKS/issues/1476](https://github.com/Azure/AKS/issues/1476)

```sh

# Collections are created as part of the az network firewall application-rule create command.
# https://docs.microsoft.com/en-us/cli/azure/ext/azure-firewall/network/firewall/application-rule?view=azure-cli-latest#ext-azure-firewall-az-network-firewall-application-rule-create
az network firewall application-rule create --collection-name $fw_app_collection_name \
                                            --firewall-name $firewall_name \
                                            --name $fw_app_rule_name \
                                            --protocols "http=80" "https=443" \
                                            --resource-group $rg_fw_name \
                                            --source-addresses "*" \
                                            --target-fqdns "${acr_registry_name}.azurecr.io" \
                                            "*.tun.${location}.azmk8s.io" "*.hcp.${location}.azmk8s.io" \
                                            "*mcr.microsoft.com" "*.data.mcr.microsoft.com" "*.cdn.mscr.io" \
                                            "*.microsoftonline.com" "management.azure.com" \
                                            "*ubuntu.com" "*.hcp.${location}.azmk8s.io" \
                                            "*.gk.${location}.azmk8s.io" "gov-prod-policy-data.trafficmanager.net" \
                                            "raw.githubusercontent.com" "dc.services.visualstudio.com" \
                                            "*.ods.opinsights.azure.com" "*.oms.opinsights.azure.com" "*.monitoring.azure.com" \
                                            "cloudflare.docker.com" "storage.googleapis.com" \
                                            --action allow --priority 100

az network firewall application-rule collection list --firewall-name $firewall_name -g $rg_fw_name

# Add Network FW Rules
# IMPORTANT!!: Here I'm adding a simplified list to represent the France-Central DataCenter, for the full list please check
# https://aka.ms/azureipranges ==> https://www.microsoft.com/en-us/download/details.aspx?id=56519
# UDP : Port 53 is DNS
# Port 445 is for SMB needed later on for the integration with Azure Files. You don't need this if you have established a storage service endpoint

 # https://docs.microsoft.com/en-us/cli/azure/ext/azure-firewall/network/firewall/network-rule?view=azure-cli-latest#ext-azure-firewall-az-network-firewall-network-rule-create
az network firewall network-rule create -n "fw-net-rules-TCP" -f $firewall_name --collection-name "${fw_network_collection_name}-TCP" \
    --protocols 'TCP' --destination-ports 9000 22 445 --action allow --priority 100 -g $rg_fw_name \
    --source-addresses '*' \
    --destination-fqdns "tun.${location}.azmk8s.io" "hcp.${location}.azmk8s.io"

az network firewall network-rule create -n "fw-net-rules-UDP" -f $firewall_name -g $rg_fw_name \
    --collection-name "${fw_network_collection_name}-UDP" \
    --protocols "UDP" --destination-ports 1194 53 123 --action allow --priority 200 \
    --source-addresses "*" \
    --destination-fqdns "ntp.ubuntu.com" "hcp.${location}.azmk8s.io" "tun.${location}.azmk8s.io"

az network firewall network-rule collection list --firewall-name $firewall_name -g $rg_fw_name

```
## Update firewall rule after AKS cluster creation
 
```sh
# https://docs.microsoft.com/en-us/azure/aks/limit-egress-traffic#required-ports-and-addresses-for-aks-clusters
# TCP [IPAddrOfYourAPIServer]:443 is required if you have an app that needs to talk to the API server. This change can be set after the cluster is created.
az network firewall application-rule create -n "fw-app-rules-aks-api" -f $firewall_name --collection-name "${fw_app_collection_name}-aks-api"\
    --protocols "https=443" --action allow --priority 300 -g $rg_fw_name \
    --source-addresses "*" \
    --target-fqdns $aks_api_server_url

```

## Add Azure Arc requirements: https://github.com/Azure/azure-arc-kubernetes-preview/blob/master/docs/connect-a-cluster.md

```sh
az network firewall network-rule create -n "fw-net-rules-azure-arc-TCP" -f $firewall_name --collection-name "${fw_network_collection_name}-ARC" \
    --protocols "TCP" --destination-ports 9418 443 --action allow --priority 400 -g $rg_fw_name \
    --source-addresses "*" \
    --destination-fqdns "*.microsoftonline.com" "management.azure.com" \
    "azurearcfork8s.azurecr.io" \
    "eastus.dp.kubernetesconfiguration.azure.com" "${location}.dp.kubernetesconfiguration.azure.com" \
    "westeurope.dp.kubernetesconfiguration.azure.com" \
    "docker.io" "github.com"



```

# Test in AKS that the outbound traffic is blocked

```sh
exec $pod -n $target_namespace -it -- /bin/sh
wget https://www.qwant.com # https://azure.microsoft.com


```

