# Setup [Azure Policy](https://docs.microsoft.com/en-us/azure/governance/policy/overview) with [OPA](https://www.openpolicyagent.org/) / [GateKeeper V3](https://kubernetes.io/blog/2019/08/06/opa-gatekeeper-policy-and-governance-for-kubernetes/) to ensure all images come from a trusted registry

See also :
- [https://docs.microsoft.com/en-us/azure/governance/policy/concepts/rego-for-aks?](https://docs.microsoft.com/en-us/azure/governance/policy/concepts/rego-for-aks?)
- [https://docs.microsoft.com/en-us/azure/security-center/security-center-permissions](https://docs.microsoft.com/en-us/azure/security-center/security-center-permissions)
- [https://docs.microsoft.com/en-us/azure/governance/policy/how-to/programmatically-create](https://docs.microsoft.com/en-us/azure/governance/policy/how-to/programmatically-create)
- [https://docs.microsoft.com/en-us/azure/security-center/security-center-permissions](https://docs.microsoft.com/en-us/azure/security-center/security-center-permissions)
- [https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#security-admin](https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#security-admin)
- [https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#resource-policy-contributor](https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#resource-policy-contributor)
- [https://github.com/Azure/Community-Policy/tree/master/Policies/KubernetesService/append-aks-api-ip-restrictions](https://github.com/Azure/Community-Policy/tree/master/Policies/KubernetesService/append-aks-api-ip-restrictions)
- [https://docs.microsoft.com/en-us/azure/governance/policy/samples/built-in-policies#kubernetes](https://docs.microsoft.com/en-us/azure/governance/policy/samples/built-in-policies#kubernetes)


```sh
# https://github.com/Azure/sg-aks-workshop/blob/master/post-provisioning/README.md#opa-and-gatekeeper-policy-setup


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

```