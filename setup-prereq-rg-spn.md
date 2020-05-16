# Create RG
```sh
az group create --name $rg_name --location $location

# az group create --name rg-cloudshell-$location --location $location

```

# Create Storage

This is not mandatory, you can create a storage account to play with CloudShell

```sh
# https://docs.microsoft.com/en-us/cli/azure/storage/account?view=azure-cli-latest#az-storage-account-create
# https://docs.microsoft.com/en-us/azure/storage/common/storage-introduction#types-of-storage-accounts
az storage account create --name $storage_name --kind StorageV2 --sku Standard_LRS --resource-group $rg_name --location $location --https-only true

# az storage account create --name stcloudshellus --kind StorageV2 --sku Standard_LRS -g rg-cloudshell-$location --location $location --https-only true

```

# Create Service Principal

You do not need to create SPN when enabling managed-identity on AKS cluster.

Read [https://docs.microsoft.com/en-us/azure/aks/kubernetes-service-principal](https://docs.microsoft.com/en-us/azure/aks/kubernetes-service-principal)

```sh
# https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal
# https://docs.microsoft.com/en-us/cli/azure/ad/sp?view=azure-cli-latest#az-ad-sp-create-for-rbac
# https://docs.microsoft.com/en-us/cli/azure/create-an-azure-service-principal-azure-cli?view=azure-cli-latest
# As of Azure CLI 2.0.68, the --password parameter to create a service principal with a user-defined password is no longer supported to prevent the accidental use of weak passwords.

# https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles
# https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#azure-kubernetes-service-cluster-admin-role
# https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#azure-kubernetes-service-cluster-user-role

sp_password=$(az ad sp create-for-rbac --name $appName --role contributor --query password --output tsv)
echo $sp_password > spp.txt
echo "Service Principal Password saved to ./spp.txt. IMPORTANT Keep your password ..." 
# sp_password=`cat spp.txt`
#sp_id=$(az ad sp show --id http://$appName --query objectId -o tsv)
#sp_id=$(az ad sp list --all --query "[?appDisplayName=='${appName}'].{appId:appId}" --output tsv)
sp_id=$(az ad sp list --show-mine --query "[?appDisplayName=='${appName}'].{appId:appId}" --output tsv)
echo "Service Principal ID:" $sp_id 
echo $sp_id > spid.txt
# sp_id=`cat spid.txt`
az ad sp show --id $sp_id

# az ad sp credential reset -n ecd34fca-9941-402a-9b16-caaac0a4eb20

```

# Generates your SSH keys

<span style="color:red">/!\ IMPORTANT </span> :  check & save your ssh_passphrase !!!

Generate & save nodes [SSH keys](https://docs.microsoft.com/en-us/azure/aks/ssh) to Azure Key-Vault is a [Best-practice](https://github.com/Azure/k8s-best-practices/blob/master/Security_securing_a_cluster.md#securing-host-access)

If you want to save your keys to keyVault, [KV must be created first](setup-kv.md)

```sh
ssh-keygen -t rsa -b 4096 -N $ssh_passphrase -f ~/.ssh/$ssh_key -C "youremail@groland.grd"

# https://www.ssh.com/ssh/keygen/
# -y Read a private OpenSSH format file and print an OpenSSH public key to stdout.
# ssh-keygen -y -f ~/.ssh/id_rsa > ~/.ssh/id_rsa.pub
# ssh-keygen -l -f ~/.ssh/id_rsa
# az keyvault key create --name $ssh_key --vault-name $vault_name --size 2048 --kty RSA
az keyvault key import --name $ssh_key --vault-name $vault_name --pem-file ~/.ssh/$ssh_key --pem-password $ssh_passphrase
az keyvault key list --vault-name $vault_name
az keyvault key show --name $ssh_key --vault-name $vault_name
az keyvault key download --name $ssh_key --vault-name $vault_name --encoding PEM --file key2
cat key1
ls -al key1
file key1
stat key1
ls -lApst key1
chmod go-rw key1
ssh-keygen -y -f key1.pem > key1.pub

```