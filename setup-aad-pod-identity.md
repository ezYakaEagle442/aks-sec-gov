# setup aad-pod-identity

See :
- [https://github.com/Azure/aad-pod-identity](https://github.com/Azure/aad-pod-identity)
- [https://medium.com/microsoftazure/pod-identity-5bc0ffb7ebe7](https://medium.com/microsoftazure/pod-identity-5bc0ffb7ebe7)

By default aad-pod-identity creates the AzureAssignedIdentity in default namespace. 
However you can run aad-pod-identity in [force namespaced mode](https://github.com/Azure/aad-pod-identity/blob/master/docs/readmes/README.namespaced.md). 
When run in force namespaced mode, the AzureAssignedIdentity will be created in the same namespace as the AzureIdentity, AzureIdentityBinding` and pods.

## Pre-requisites

<span style="color:red">/!\ IMPORTANT </span> : For AKS clusters with limited egress-traffic, Please install pod-identity in kube-system namespace using the [helm charts](https://github.com/Azure/aad-pod-identity/tree/master/charts/aad-pod-identity).

```sh
helm install aad-pod-identity aad-pod-identity/aad-pod-identity --namespace $target_namespace --set azureIdentity.namespace=$target_namespace
helm ls --namespace $target_namespace 

# k apply -f https://raw.githubusercontent.com/Azure/aad-pod-identity/master/deploy/infra/deployment-rbac.yaml
k get crd -A
k api-resources
k describe ds aad-pod-identity-nmi -n $target_namespace 
k get deployments -n $target_namespace 
k get deployment aad-pod-identity-mic -n $target_namespace 
k describe deployment aad-pod-identity-mic -n $target_namespace 
k get pods -n $target_namespace 
for pod in $(k get pods -l app.kubernetes.io/component=mic -n $target_namespace -o custom-columns=:metadata.name)
do
	k describe pod $pod -n $target_namespace # | grep -i "Error"
	k logs $pod -n $target_namespace | grep -i "Error"
done

```


```sh
export SUBSCRIPTION_ID=$subId
export RESOURCE_GROUP=$rg_name
export IDENTITY_NAME="${appName}-pod-identity" #  must consist of lower case 

# https://github.com/Azure/aad-pod-identity/pull/48
# https://github.com/Azure/aad-pod-identity/issues/38
az identity create -g $rg_name -n $IDENTITY_NAME
export IDENTITY_CLIENT_ID="$(az identity show -g $IDENTITY_NAME -n $IDENTITY_NAME --query clientId -otsv)"
export IDENTITY_RESOURCE_ID="$(az identity show -g $rg_name -n $IDENTITY_NAME --query id -otsv)"
export IDENTITY_ASSIGNMENT_ID="$(az role assignment create --role Reader --assignee $IDENTITY_CLIENT_ID --scope /subscriptions/$subId/resourceGroups/$rg_name --query id -o tsv)"
az identity show --name $IDENTITY_NAME -g $rg_name

# Your cluster will need the correct role assignment configuration to perform Azure-related operations such as assigning and un-assigning the identity on the underlying VM/VMSS. Please refer to https://github.com/Azure/aad-pod-identity/blob/master/docs/readmes/README.role-assignment.md
aks_client_id=$(az aks show -g $rg_name -n $cluster_name --query identityProfile.kubeletidentity.clientId -o tsv)
echo "AKS Cluster Identity Client ID " $aks_client_id
az identity show --ids $PoolIdentityResourceID
# you can see this App. in the portal / then in the AKS MC_ RG
# https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#managed-identity-operator
# https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#virtual-machine-contributor
az role assignment create --role "Managed Identity Operator" --assignee $aks_client_id --scope /subscriptions/$subId/resourcegroups/$managed_rg
az role assignment create --role "Managed Identity Operator" --assignee $aks_client_id --scope /subscriptions/$subId/resourcegroups/$rg_name
az role assignment create --role "Virtual Machine Contributor" --assignee $aks_client_id --scope /subscriptions/$subId/resourcegroups/$managed_rg

# Set type: 0 for user-assigned MSI or type: 1 for Service Principal
cat <<EOF | k apply -n $target_namespace -f -
apiVersion: "aadpodidentity.k8s.io/v1"
kind: AzureIdentity
metadata:
  name: $IDENTITY_NAME
  annotations:
    aadpodidentity.k8s.io/Behavior: namespaced  
spec:
  type: 0
  resourceID: $IDENTITY_RESOURCE_ID
  clientID: $IDENTITY_CLIENT_ID
EOF

k get azureidentity -A

cat <<EOF | k apply -n $target_namespace -f -
apiVersion: "aadpodidentity.k8s.io/v1"
kind: AzureIdentityBinding
metadata:
  name: $IDENTITY_NAME-binding
spec:
  azureIdentity: $IDENTITY_NAME
  selector: $IDENTITY_NAME
EOF

k get azureidentity -A
k get azureidentitybindings -A
k get azureassignedidentities -A

k describe azureidentity $IDENTITY_NAME -n $target_namespace
k describe azureidentitybinding $IDENTITY_NAME-binding -n $target_namespace
k describe azureassignedidentities pod-identity-demo-$target_namespace-$IDENTITY_NAME

```

## Default demo
```sh

cat << EOF | k apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pod-identity-demo
  namespace: $target_namespace
  labels:
    aadpodidbinding: $IDENTITY_NAME
spec:
  containers:
  - name: demo
    image: mcr.microsoft.com/k8s/aad-pod-identity/demo:1.6.0
    args:
      - --subscriptionid=$SUBSCRIPTION_ID
      - --clientid=$IDENTITY_CLIENT_ID
      - --resourcegroup=$RESOURCE_GROUP
    env:
      - name: MY_POD_NAME
        valueFrom:
          fieldRef:
            fieldPath: metadata.name
      - name: MY_POD_NAMESPACE
        valueFrom:
          fieldRef:
            fieldPath: metadata.namespace
      - name: MY_POD_IP
        valueFrom:
          fieldRef:
            fieldPath: status.podIP
  nodeSelector:
    kubernetes.io/os: linux
EOF

k get pods -n $target_namespace
k get pod pod-identity-demo -n $target_namespace -o wide
podIP=$(k get pod pod-identity-demo -n $target_namespace -o jsonpath="{.status.podIP}")
echo "Pod IP" $podIP
k describe pod pod-identity-demo -n $target_namespace
k logs pod-identity-demo -n $target_namespace
k get events -n $target_namespace | grep -i "Error" 
k delete pod pod-identity-demo -n $target_namespace
 
```

## Clean-Up
```sh
k delete pod pod-identity-demo -n $target_namespace
k delete ds nmi
k get ds
k delete deployment mic
k get deployments
k delete sa aad-pod-id-mic-service-account
k delete sa aad-pod-id-nmi-service-account
k get sa

k delete ClusterRole aad-pod-id-mic-role
k delete ClusterRole aad-pod-id-nmi-role
k delete ClusterRoleBinding aad-pod-id-nmi-binding
k delete ClusterRoleBinding aad-pod-id-mic-binding

helm uninstall aad-pod-identity
```


```sh

```