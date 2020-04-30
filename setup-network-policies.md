# Setup Network-Policies

See :
- [https://kubernetes.io/docs/concepts/services-networking/network-policies](https://kubernetes.io/docs/concepts/services-networking/network-policies)
- [https://kubernetes.io/docs/tasks/administer-cluster/declare-network-policy](https://kubernetes.io/docs/tasks/administer-cluster/declare-network-policy)
- [https://docs.microsoft.com/en-us/azure/aks/support-policies#aks-support-coverage](https://docs.microsoft.com/en-us/azure/aks/support-policies#aks-support-coverage)
- [https://docs.microsoft.com/en-us/azure/aks/use-network-policies#differences-between-azure-and-calico-policies-and-their-capabilities](https://docs.microsoft.com/en-us/azure/aks/use-network-policies#differences-between-azure-and-calico-policies-and-their-capabilities)
- [https://cloudblogs.microsoft.com/opensource/2019/10/17/tutorial-calico-network-policies-with-azure-kubernetes-service](https://cloudblogs.microsoft.com/opensource/2019/10/17/tutorial-calico-network-policies-with-azure-kubernetes-service)
- [https://docs.projectcalico.org/security/policy-rules-overview](https://docs.projectcalico.org/security/policy-rules-overview)
- [https://github.com/ahmetb/kubernetes-network-policy-recipes](https://github.com/ahmetb/kubernetes-network-policy-recipes)
- [https://github.com/palma21/secureaks/blob/master/np-web-allow-worker.yaml](https://github.com/palma21/secureaks/blob/master/np-web-allow-worker.yaml)

In Azure you can only use policy mode – not Calico networking for routing- . See [Calico docs](https://docs.projectcalico.org/reference/public-cloud/azure)
Calico can be configured with Azure CNI IPAM plug-in. In AKS, it is not possible [to set IP pools per-namespace or per-pod](https://docs.projectcalico.org/reference/cni-plugin/configuration#specifying-ip-pools-on-a-per-namespace-or-per-pod-basis) using the “cni.projectcalico.org/ipv4pools” Kubernetes annotation

About Network Policies: CNI & KubeNet are now both supported:
ex: az aks create -n calkubclu -g calkubrg -c 1 --network-plugin kubenet --network-policy calico



