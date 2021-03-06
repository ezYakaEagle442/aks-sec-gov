# setup 

You can use [Kured to Apply security and kernel updates to Linux nodes in AKS](https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-cluster-security#process-linux-node-updates-and-reboots-using-kured). 

Kured can now set a schedule: Kured [PR-66](https://github.com/weaveworks/kured/pull/66) merged in [Release 1.3.0](https://github.com/weaveworks/kured/releases/tag/1.3.0): cf [https://github.com/weaveworks/kured#setting-a-schedule](https://github.com/weaveworks/kured#setting-a-schedule) offering new arguments to set a schedule : --reboot-days, --start-time, --end-time, and --time-zone.


See also :
- [https://docs.microsoft.com/en-us/azure/aks/use-multiple-node-pools#upgrade-a-cluster-control-plane-with-multiple-node-pools](https://docs.microsoft.com/en-us/azure/aks/use-multiple-node-pools#upgrade-a-cluster-control-plane-with-multiple-node-pools)
- [https://github.com/weaveworks/kured#setting-a-schedule](https://github.com/weaveworks/kured#setting-a-schedule)
- [Reboot Sentinel File & Period](https://github.com/weaveworks/kured#reboot-sentinel-file--period) : By default kured checks for the existence of /var/run/reboot-required every sixty minutes; you can override these values with --reboot-sentinel and --period. Each replica of the daemon uses a random offset derived from the period on startup so that nodes don't all contend for the lock simultaneously.

```sh

# Update your local Helm chart repository cache
helm repo update

# Create a dedicated namespace where you would like to deploy kured into
k create namespace kured

# Install kured in that namespace with Helm 3 (only on Linux nodes, kured is not working on Windows nodes)
# https://github.com/weaveworks/kured#setting-a-schedule
# https://github.com/MicrosoftDocs/azure-docs/issues/45912
helm install kured stable/kured --namespace kured \
    --set nodeSelector."beta\.kubernetes\.io/os"=linux \
    --set extraArgs.reboot-days="mon\,tue\,wed\,thu\,fri" \
    --set extraArgs.start-time=9am \
    --set extraArgs.end-time=5pm \
    --set extraArgs.time-zone="Europe/Paris"

helm ls --namespace kured
helm status kured --namespace kured

```

## Manual SSH operation
- [https://docs.microsoft.com/en-us/azure/aks/node-updates-kured#update-cluster-nodes](https://docs.microsoft.com/en-us/azure/aks/node-updates-kured#update-cluster-nodes)

```sh
# 
cat /var/run/reboot-required
# *** System restart required ***
ls /var/run/reboot-required.pkgs

sudo apt-get update && sudo apt-get upgrade -y


```
