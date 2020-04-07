# Setup AAD Integration

See [https://docs.microsoft.com/en-us/azure/active-directory/fundamentals/active-directory-how-subscriptions-associated-directory](https://docs.microsoft.com/en-us/azure/active-directory/fundamentals/active-directory-how-subscriptions-associated-directory)

***If you are not tenant-admin, you need to [create your own tenant]**(https://docs.microsoft.com/en-us/azure/active-directory/fundamentals/active-directory-access-create-new-tenant)

See also 
- new feature (not yet GA) [Integrate Azure AD v2.0 in AKS](https://docs.microsoft.com/en-us/azure/aks/azure-ad-v2)
- [https://github.com/Azure/AKS/issues/1489](https://github.com/Azure/AKS/issues/1489 )

```sh
# TODO

```


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

