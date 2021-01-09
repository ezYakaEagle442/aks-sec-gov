# Setup AAD Integration

See [https://docs.microsoft.com/en-us/azure/active-directory/fundamentals/active-directory-how-subscriptions-associated-directory](https://docs.microsoft.com/en-us/azure/active-directory/fundamentals/active-directory-how-subscriptions-associated-directory)

***If you are not tenant-admin, you need to [create your own tenant]**(https://docs.microsoft.com/en-us/azure/active-directory/fundamentals/active-directory-access-create-new-tenant)

See also 
- new feature (not yet GA) [Integrate Azure AD v2.0 in AKS](https://docs.microsoft.com/en-us/azure/aks/managed-aad)
- [https://github.com/Azure/AKS/issues/1489](https://github.com/Azure/AKS/issues/1489 )
- [https://docs.microsoft.com/en-us/azure/aks/concepts-identity#azure-active-directory-integration]
- [https://docs.microsoft.com/en-us/azure/aks/azure-ad-integration-cli](https://docs.microsoft.com/en-us/azure/aks/azure-ad-integration-cli)
- [https://docs.microsoft.com/en-us/azure/aks/azure-ad-rbac](https://docs.microsoft.com/en-us/azure/aks/azure-ad-rbac)
- [https://kubernetes.io/docs/reference/access-authn-authz/authentication/#webhook-token-authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#webhook-token-authentication)

Choose AAD Integration [V1](#aad-integration-v1) or [V2](#aad-integration-v2)
Azure AD integration with AKS v2 is designed to simplify the Azure AD integration with AKS v1 experience, where users were required to create a client app, a server app, and required the **Azure AD tenant to grant Directory Read permissions**. In the new version, the AKS resource provider manages the client and server apps for you.


See also[Kube Login](https://github.com/Azure/kubelogin), a client-go credential plugin implementing Azure authentication, supported only with Managed AAD (V2). 
This plugin provides features that are not available in kubectl :
- non-interactive Service Principal login
- non-interactive Managed Service identity login
- AAD token will be cached locally for renewal in device code login and user principal login (ropc) flow. By default, it is saved in ~/.kube/cache/kubelogin


## AAD Integration V2 

(Currently in Preview)
For AADv2 you [do not need to be AAD tenant admin](https://github.com/MicrosoftDocs/azure-docs/issues/53378).

[Pre-requistes](https://docs.microsoft.com/en-us/azure/aks/managed-aad#before-you-begin) :
- The Azure CLI, version 2.2.0 or later
- The aks-preview 0.4.38 extension
- Kubectl with a minimum version of **1.18** beta

```sh
# TODO
az feature register --name AAD-V2 --namespace Microsoft.ContainerService
az feature list -o table --query "[?contains(name, 'Microsoft.ContainerService/AAD-V2')].{Name:name,State:properties.state}"

# Test for CLI
#cli_sp_password=$(az ad sp create-for-rbac --name cli-$appName --role contributor --query password --output tsv)
#echo $cli_sp_password
#appName="cli-${appName}"
#cli_sp_id=$(az ad sp list --show-mine --query "[?appDisplayName=='${appName}'].{appId:appId}" --output tsv)
#echo "CLI Service Principal ID:" $cli_sp_id 
#az login --service-principal --username $cli_sp_id --password  $cli_sp_password --tenant $tenantId

# https://docs.microsoft.com/en-us/azure/active-directory/fundamentals/active-directory-groups-create-azure-portal
# Create the AKS Admin group in Azure AD
AKSADM_GRP_ID=$(az ad group create --display-name aks-adm-${appName} --mail-nickname aks-adm-${appName} --query objectId -o tsv)
echo "AKS ADMIN GROUP ID: " $AKSADM_GRP_ID
az ad group show --group $AKSADM_GRP_ID

# https://docs.microsoft.com/en-us/azure/active-directory/fundamentals/add-users-azure-active-directory
# Create AKS Admin user. The user principal name (someuser@contoso.com) must contain one of the verified domains for the tenant.
AKSADM_USR_ID=$(az ad user create --display-name "AKS Admin user ${appName}" --user-principal-name "aksadm@groland.grd" --password "P@ssw0rd1" --query objectId -o tsv)
echo "AKS ADMIN USER ID: " $AKSADM_USR_ID
az ad user show --id $AKSADM_USR_ID

# Add the user to the appdev Azure AD group
az ad group member add --member-id $AKSADM_USR_ID --group  $AKSADM_GRP_ID

# https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#azure-kubernetes-service-cluster-admin-role
# az role assignment create --assignee $AKSADM_GRP_ID --role "Azure Kubernetes Service Cluster Admin Role" --scope $aks_cluster_id

```

## AAD Integration V1
```sh
# https://docs.microsoft.com/en-us/azure/aks/azure-ad-integration-cli
# Create the Azure AD application
serverApplicationId=$(az ad app create \
    --display-name "aks-${appName}" \
    --identifier-uris "https://aks-${appName}-Server" \
    --query appId -o tsv)

echo "serverApplicationId" : $serverApplicationId
az ad app list --app-id $serverApplicationId # --show-mine

# Update the application group memebership claims
# In the Azure portal, you can update this field in : App registrations / aks-secgov | Manifest ==> "groupMembershipClaims": "All",
az ad app update --id $serverApplicationId --set groupMembershipClaims=All
az ad app list --app-id $serverApplicationId | grep -i groupMembershipClaims

# Create a service principal for the Azure AD application
# https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal
az ad sp create --id $serverApplicationId

# Get the service principal secret
serverApplicationSecret=$(az ad sp credential reset \
    --name $serverApplicationId \
    --credential-description "AKSPassword" \
    --query password -o tsv)

echo $serverApplicationSecret > serverApplicationSecret.txt
echo "Server Application Secret saved to ./serverApplicationSecret.txt IMPORTANT Keep your secret safe ..." 


```


The Azure AD needs permissions to perform the following actions:
- Read directory data
- Sign in and read user profile

Finally, grant the permissions assigned in the previous step for the server application using the az ad app permission grant command. **This step fails if the current account is not a tenant admin**. You also need to add permissions for Azure AD application to request information that may otherwise require administrative consent using the az ad app permission admin-consent:

```sh
# https://docs.microsoft.com/en-us/cli/azure/ad/app/permission?view=azure-cli-latest
az ad app permission add \
    --id $serverApplicationId \
    --api 00000003-0000-0000-c000-000000000000 \
    --api-permissions e1fe6dd8-ba31-4d61-89e7-88639da4683d=Scope 06da0dbc-49e2-44d2-8312-53f166ab848a=Scope 7ab1d382-f21e-4acd-a863-ba3e13f7da61=Role

# https://stackoverflow.com/questions/56907292/az-ad-app-permission-list-grants-doesnt-match-what-is-listed-for-the-app-in-t
# If you also want to get the application permissions granted to the service principal, currently it is not supported by the Azure CLI and Az powershell module, you need to use AzureAD powershell module.

az ad app permission grant --id $serverApplicationId --api 00000003-0000-0000-c000-000000000000
az ad app permission admin-consent --id $serverApplicationId # requires CLI version min of 2.0.67 and max of 2.1.0.
az ad app permission list-grants --id $serverApplicationId --show-resource-name
az ad app permission list --id $serverApplicationId

clientApplicationId=$(az ad app create \
    --display-name "${appName}Client" \
    --native-app \
    --reply-urls "https://aks-${appName}Client" \
    --query appId -o tsv)

echo "clientApplicationId" : $clientApplicationId
az ad app list --app-id $clientApplicationId # --show-mine

az ad sp create --id $clientApplicationId
oAuthPermissionId=$(az ad app show --id $serverApplicationId --query "oauth2Permissions[0].id" -o tsv)
echo "oAuthPermissionId" : ${oAuthPermissionId}

az ad app permission add --id $clientApplicationId --api $serverApplicationId --api-permissions ${oAuthPermissionId}=Scope
az ad app permission grant --id $clientApplicationId --api $serverApplicationId
az ad app permission list-grants --id $clientApplicationId --show-resource-name
az ad app permission list --id $clientApplicationId

tenantId=$(az account show --query tenantId -o tsv)
echo $tenantId > tenantId.txt
echo $serverApplicationId > serverApplicationId.txt
echo $serverApplicationSecret > serverApplicationSecret.txt
echo $clientApplicationId > clientApplicationId.txt

```

**If you use 1 subscription to be tenant-admin and another one to run AKS, now logout and login to the second one** \

az account list

az account set --subscription $subId

az account show

az login

