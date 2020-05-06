# Create NodePools

[Doc Reference](https://docs.microsoft.com/en-us/azure/aks/use-multiple-node-pools)

- The name of a node pool may only contain lowercase alphanumeric characters and must begin with a lowercase letter. 
- For Linux node pools the length must be between 1 and 12 characters, for Windows node pools the length must be between 1 and 6 characters.
- All node pools must reside in the same virtual network and subnet.
- When creating multiple node pools at cluster create time, all Kubernetes versions used by node pools must match the version set for the control plane. This version can be updated after the cluster has been provisioned by using per node pool operations.
- [Kubernetes Taints & Tolerations docs](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration)
- [CLI Reference](https://docs.microsoft.com/en-us/cli/azure/aks/nodepool?view=azure-cli-latest#az-aks-nodepool-add)


See also [AKS creation of NodePools in different Subnets](https://docs.microsoft.com/en-us/azure/aks/use-multiple-node-pools#add-a-node-pool-with-a-unique-subnet-preview)
- [https://aka.ms/node-labels](https://aka.ms/node-labels)

[Issue #1338](https://github.com/Azure/AKS/issues/1338) 

```sh

# https://github.com/Azure/AKS/issues/1338

az aks nodepool list --cluster-name $cluster_name -g $rg_name

# https://github.com/Azure/azure-cli/issues/8572
# az extension remove --name aks-preview

# Standard_D1_v2 or Standard_B1s or Standard_F2s_v2
az aks nodepool add -n $poc_node_pool_name --cluster-name $cluster_name \
    --labels env=poc team=gbb dept=wcb \
    --resource-group $rg_name \
    --vnet-subnet-id $new_node_pool_subnet_id \
    --zones 1 2 3 \
    --enable-cluster-autoscaler \
    --min-count=1 \
    --max-count=3 \
    --kubernetes-version $version \
    --node-vm-size Standard_B1s \
    --node-taints env=poc:NoSchedule \
    --os-type Linux # Windows

kubectl get nodes

```

## Create spot instance node pools.

To know before you start:

- Spot pools cannot be primary pool
- Spot needs to be VMSS based
- Spot pools cannot be upgraded
- Spot pools cannot change ScaleSetPriority after creation - recreation is needed
- Spot pools has no SLA and they can be evicted at any time. See [VMSS eviction policy](https://docs.microsoft.com/en-us/azure/virtual-machine-scale-sets/use-spot#eviction-policy)
- Spot Nodes will have label: kubernetes.azure.com/scalesetpriority:spot  and System Pod's having anti-affinity to it.
- Spot Nodes will have taint: kubernetes.azure.com/scalesetpriority=spot:NoSchedule
- Remember to add tolerate to put designated workloads on Spot pool
- Spot node pools require [user node pools](https://aka.ms/aks/nodepool/mode). AKS has now introduced a new Mode property for nodepools. This will allow you to set nodepools as System or User nodepools. System nodepools will have additional validations and will be preferred by system pods, while User pool will have more lax validations and can perform additional operations like **scale to 0 nodes or be removed from the cluster**. Each cluster needs at least one system pool.

```sh

# **Note** Using '--spot-max-price -1' will give you the following:
#
# 1) The default price will be up-to on-demand.
# 2) The instance won't be evicted based on price.
# 3) The price for the instance will be the current price for Spot or the price for a standard instance, which ever is less, as long as there is capacity and quota available.

# if the value is set to $0.846 it would be a max price of $0.846 USD per hour.
az extension update --name aks-preview
az feature register --namespace Microsoft.ContainerService --name SpotPoolPreview
az provider register -n Microsoft.ContainerService

az aks nodepool add -g $rg_name -n $spotpool_name_max --cluster-name $cluster_name --mode user \
    --priority Spot \
    --spot-max-price -1 \
    --eviction-policy Delete \
    --labels env=poc team=gbb dept=wcb price=max \
    --resource-group $rg_name \
    --vnet-subnet-id $new_node_pool_subnet_id \
    --zones 1 2 3 \
    --enable-cluster-autoscaler \
    --min-count=1 \
    --max-count=3 \
    --kubernetes-version $version \
    --node-vm-size Standard_F2s_v2 \
    --node-taints spot=max:NoSchedule \
    --os-type Linux # Windows

# https://azure.microsoft.com/en-us/pricing/calculator/?service=virtual-machines
# https://azure.microsoft.com/en-us/pricing/details/virtual-machines/linux/#f-series
# https://github.com/Azure/azure-cli/blob/06cce5b0bf3fe807280b59600001c8213390745c/src/azure-cli/azure/cli/command_modules/acs/custom.py#L3140
az aks nodepool add -g $rg_name -n $spotpool_name_min --cluster-name $cluster_name --mode user \
    --priority Spot \
    --spot-max-price 0.016 \
    --eviction-policy Delete \
    --labels env=poc team=gbb dept=wcb price=min \
    --resource-group $rg_name \
    --vnet-subnet-id $new_node_pool_subnet_id \
    --zones 1 2 3 \
    --enable-cluster-autoscaler \
    --min-count=1 \
    --max-count=3 \
    --kubernetes-version $version \
    --node-vm-size Standard_F2s_v2 \
    --node-taints env=poc:NoSchedule \
    --os-type Linux # Windows

kubectl get nodes --show-labels


```

## Upgrades

[https://docs.microsoft.com/en-us/azure/aks/use-multiple-node-pools#upgrade-a-node-pool](https://docs.microsoft.com/en-us/azure/aks/use-multiple-node-pools#upgrade-a-node-pool)
[https://docs.microsoft.com/en-us/azure/aks/use-multiple-node-pools#upgrade-a-cluster-control-plane-with-multiple-node-pools](https://docs.microsoft.com/en-us/azure/aks/use-multiple-node-pools#upgrade-a-cluster-control-plane-with-multiple-node-pools)
```sh
az aks nodepool list --cluster-name $cluster_name -g $rg_name

az aks get-upgrades -g $rg_name --name $cluster_name

az aks upgrade --control-plane-only \
    --name $cluster_name \
    --resource-group $rg_name \
    --kubernetes-version 1.15.10

az aks nodepool upgrade \
    --resource-group $rg_name \
    --cluster-name $cluster_name \
    --name $poc_node_pool_name \
    --kubernetes-version 1.15.10


```
