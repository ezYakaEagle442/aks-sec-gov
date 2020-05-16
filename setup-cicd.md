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

## setup a Self-Hosted Azure DevOps agent

- [https://github.com/microsoft/azure-pipelines-agent/blob/master/docs/design/byos.md](https://github.com/microsoft/azure-pipelines-agent/blob/master/docs/design/byos.md) 
- [https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/agents?view=azure-devops&tabs=browser#install](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/agents?view=azure-devops&tabs=browser#install )
- [https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/hosted?view=azure-devops#agent-ip-ranges]
- [https://github.com/MicrosoftDocs/azure-docs/issues/49578](https://github.com/MicrosoftDocs/azure-docs/issues/49578 )
- [https://github.com/MicrosoftDocs/azure-docs/issues/45678](https://github.com/MicrosoftDocs/azure-docs/issues/45678)
- [https://github.com/actions/virtual-environments/issues/666](https://github.com/actions/virtual-environments/issues/666)
- [https://github.com/actions/virtual-environments/issues/196](https://github.com/actions/virtual-environments/issues/196)
- [http://echoazure.com/2020/03/17/use-azure-devops-to-automate-ci-cd-on-azure-kubernetes-service-aks-private-cluster](http://echoazure.com/2020/03/17/use-azure-devops-to-automate-ci-cd-on-azure-kubernetes-service-aks-private-cluster)


- If you use a self-hosted agent on VM , you will have to manage the VM, the patch management & the agent. 
- a better option would be to run the Azure Devops agent in [ACI linked to AKS](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-vnet)
see 
- [https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/docker?view=azure-devops](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/docker?view=azure-devops)
- [https://devblogs.microsoft.com/devops/azure-devops-agents-on-azure-container-instances-aci/](https://devblogs.microsoft.com/devops/azure-devops-agents-on-azure-container-instances-aci/)

[https://github.com/microsoft/azure-pipelines-image-generation](https://github.com/microsoft/azure-pipelines-image-generation) ==> Azure Pipelines Linux-based images are now in the virtual-environments repository

[https://github.com/actions/virtual-environments/blob/master/images.CI/azure-pipelines/ubuntu1804.yml]([https://github.com/actions/virtual-environments/blob/master/images.CI/azure-pipelines/ubuntu1804.yml)

[https://github.com/MicrosoftDocs/azure-docs/issues/54750](https://github.com/MicrosoftDocs/azure-docs/issues/54750)

```sh

# TODO

```