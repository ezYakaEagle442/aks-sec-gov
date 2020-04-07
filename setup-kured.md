# setup 

You can use [Kured to Apply security and kernel updates to Linux nodes in AKS](https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-cluster-security#process-linux-node-updates-and-reboots-using-kured). 

Kured can now set a schedule: Kured [PR-66](https://github.com/weaveworks/kured/pull/66) merged in [Release 1.3.0](https://github.com/weaveworks/kured/releases/tag/1.3.0): cf [https://github.com/weaveworks/kured#setting-a-schedule](https://github.com/weaveworks/kured#setting-a-schedule) offering new arguments to set a schedule : --reboot-days, --start-time, --end-time, and --time-zone.


See also :
- [https://docs.microsoft.com/en-us/azure/aks/use-multiple-node-pools#upgrade-a-cluster-control-plane-with-multiple-node-pools](https://docs.microsoft.com/en-us/azure/aks/use-multiple-node-pools#upgrade-a-cluster-control-plane-with-multiple-node-pools)
- [https://docs.microsoft.com/en-us/azure/aks/node-updates-kured#update-cluster-nodes](https://docs.microsoft.com/en-us/azure/aks/node-updates-kured#update-cluster-nodes)


```sh
# 
cat /var/run/reboot-required
# *** System restart required ***
ls /var/run/reboot-required.pkgs

sudo apt-get update && sudo apt-get upgrade -y


```
