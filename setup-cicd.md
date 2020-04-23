# Setup CI/CD

## When AAD-RBAC is enabled
See also :
- [https://devopscube.com/kubernetes-api-access-service-account/](https://devopscube.com/kubernetes-api-access-service-account/)
- [https://github.com/Azure/AKS/issues/600](https://github.com/Azure/AKS/issues/600)
- [https://feedback.azure.com/forums/914020-azure-kubernetes-service-aks/suggestions/35146387-support-non-interactive-login-for-aad-integrated-c](https://feedback.azure.com/forums/914020-azure-kubernetes-service-aks/suggestions/35146387-support-non-interactive-login-for-aad-integrated-c)
- [https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#config](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#config)

```sh

kubectl create clusterrolebinding owner-cluster-admin-binding \
    --clusterrole cluster-admin \
    --user $USR_OBJECT_ID

kubectl create serviceaccount api-service-account

kubectl apply -f clusterRole.yaml

sa_secret_name=$(kubectl get serviceaccount api-service-account  -o json | jq -Mr '.secrets[].name')
echo "SA secret name " $sa_secret_name

sa_secret_value=$(kubectl get secrets  $sa_secret_name -o json | jq -Mr '.data.token' | base64 -d)
echo "SA secret  " $sa_secret_value

kube_url=$(kubectl get endpoints -o jsonpath='{.items[0].subsets[0].addresses[0].ip}')
echo "Kube URL " $kube_url

curl -k  https://$kube_url/api/v1/namespaces -H "Authorization: Bearer $sa_secret_value"

```

## TODO

```sh

# https://stackoverflow.com/questions/60083889/kubectl-token-token-doesnt-run-with-the-permissions-of-the-token
kubectl config set-credentials $USER_NAME --token=$sa_secret_value

```