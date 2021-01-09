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

## Ensure you have the appropriate Roles to be allowed to create a Service Principal

[az ad group create CLI](https://docs.microsoft.com/en-us/cli/azure/ad/group?view=azure-cli-latest#az_ad_group_create)
[az ad user create CLI](https://docs.microsoft.com/en-us/cli/azure/ad/user?view=azure-cli-latest#az_ad_user_create)
[az ad group member add CLI](https://docs.microsoft.com/en-us/cli/azure/ad/group/member?view=azure-cli-latest#az_ad_group_member_add)
[az role assignment create CLI](https://docs.microsoft.com/en-us/cli/azure/role/assignment?view=azure-cli-latest#az_role_assignment_create)


Too much: [Application Administrator permission](https://docs.microsoft.com/en-us/azure/active-directory/roles/permissions-reference#application-administrator-permissions)
[Application Developer permission is enough to create a Service Principal](https://docs.microsoft.com/en-us/azure/active-directory/roles/permissions-reference#application-developer-permissions)
- microsoft.directory/servicePrincipals/create
- microsoft.directory/applications/create
- microsoft.directory/appRoleAssignments/create

You could add roles to a Group, this is in PREVIEW in the [Portal](https://docs.microsoft.com/en-us/azure/active-directory/roles/groups-concept#required-license-plan) but this feature requires you to have an available Azure AD Premium P1 license in your Azure AD organization.


## Create SP
You do not need to create SPN when enabling managed-identity on AKS cluster.

Read [https://docs.microsoft.com/en-us/azure/aks/kubernetes-service-principal](https://docs.microsoft.com/en-us/azure/aks/kubernetes-service-principal)
[Additional considerations](https://docs.microsoft.com/en-us/azure/aks/kubernetes-service-principal#additional-considerations)
On the agent node VMs in the Kubernetes cluster, the service principal credentials are stored in the file /etc/kubernetes/azure.json
When you use the az aks create command to generate the service principal automatically, the service principal credentials are written to the file /aksServicePrincipal.json on the machine used to run the command.

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

Read [this explanation about private keys management in KV](https://github.com/Azure/azure-sdk-for-js/issues/7647#issuecomment-594935307)
The KeyVault service stores both the public and the private parts of your certificate in a KeyVault secret, along with any other secret you might have created in that same KeyVault instance. With this separation comes considerable control of the level of access you can give to people in your organization regarding your certificates. The access control can be specified through the policy you pass in when creating a certificate. Knowing that the private key is stored in a KeyVault Secret, with the public certificate included, we can retrieve it by using the KeyVault-Secrets client.

See also :
- [About keys](https://docs.microsoft.com/en-us/azure/key-vault/certificates/about-certificates#composition-of-a-certificate)
- Composition of a Certificate](https://docs.microsoft.com/en-us/azure/key-vault/certificates/about-certificates#composition-of-a-certificate)
- [https://github.com/MicrosoftDocs/azure-docs/issues/55072](https://github.com/MicrosoftDocs/azure-docs/issues/55072)
- [https://github.com/Azure/azure-cli/issues/13547](https://github.com/Azure/azure-cli/issues/13547)
- [https://github.com/Azure/azure-cli/issues/13548](https://github.com/Azure/azure-cli/issues/13548)

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

KV stores both public part and private part of your key. Once you uploaded your key into KV, there is no way to show or download the private part: this is for security concern.
Once the key is imported in KV, then az keyvault key download returns the public key only. 

You can eventually save the private key key and also the passphrase as secrets in KV

See :
- this similar [topic & explanation regarding certificates](https://github.com/Azure/azure-sdk-for-js/issues/7647#issuecomment-594935307).
- [https://github.com/MicrosoftDocs/azure-docs/issues/55072](https://github.com/MicrosoftDocs/azure-docs/issues/55072)

```sh
az keyvault secret list --vault-name $vault_name
az keyvault secret show --name $vault_secret_name --vault-name $vault_name --output tsv

ssh_prv_key_val=`cat ~/.ssh/$ssh_key`

az keyvault secret set --name ssh-passphrase --value $ssh_passphrase --vault-name $vault_name --description "AKS ${appName} SSH Key Passphrase" 
az keyvault secret set --name ssh-key --value "$ssh_prv_key_val" --vault-name $vault_name --description "AKS ${appName} SSH Private Key value" 

az keyvault secret list --vault-name $vault_name
az keyvault secret show --vault-name $vault_name --name ssh-passphrase -o table
az keyvault secret show --vault-name $vault_name --name ssh-key -o table

az keyvault secret download --file myLostKey.txt --name ssh-key --vault-name $vault_name
az keyvault secret download --file myLostPassPhrase.txt --name ssh-passphrase --vault-name $vault_name
```