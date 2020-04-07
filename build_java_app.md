# Build Spring Boot App

## Create Docker Image
```sh
# https://docs.microsoft.com/en-us/azure/container-registry/container-registry-quickstart-task-cli
git clone $git_url_springboot 
cd spring-petclinic

# build app
sudo apt install maven --yes
mvn package

# On Azure Zulu JRE located at : /usr/lib/jvm/zulu-8-azure-amd64/
# to check which process runs eventually already on port 8080 :  netstat -anp | grep 8080 
# lsof -i :8080 | grep LISTEN
# ps -ef | grep PID

# Test the App
mvn spring-boot:run

# https://docs.microsoft.com/en-us/java/azure/jdk/java-jdk-docker-images?view=azure-java-stable
# https://github.com/microsoft/java/blob/master/docker/alpine/Dockerfile.zulu-8u242-jre
# https://github.com/microsoft/java/blob/master/docker/alpine/Dockerfile.zulu-11u6-jre
# Java 8 image : mcr.microsoft.com/java/jdk:8u232-zulu-alpine
# Java 11 image :  mcr.microsoft.com/java/jre:11u6-zulu-alpine
artifact="spring-petclinic-2.2.0.BUILD-SNAPSHOT.jar"
echo -e "FROM mcr.microsoft.com/java/jre:11u6-zulu-alpine\n" \
"VOLUME /tmp \n" \
"ADD target/${artifact} app.jar \n" \
"RUN touch /app.jar \n" \
"EXPOSE 8080 \n" \
"ENTRYPOINT [ \""java\"", \""-Djava.security.egd=file:/dev/./urandom\"", \""-jar\"", \""/app.jar\"" ] \n"\
> Dockerfile

# other app snippet: https://github.com/microsoft/todo-app-java-on-azure/blob/master/deploy/aks/deployment.yml

docker_server=$(az acr show --name $acr_registry_name --resource-group $rg_acr_name --query "loginServer" --output tsv)
echo "Docker server :" $docker_server

docker_private_server="${acr_registry_name}.privatelink.azurecr.io"
echo "Docker private server :" $docker_private_server

kubectl -n $target_namespace create secret docker-registry acr-auth \
        --docker-server=$docker_server \
        --docker-username=$sp_id \
        --docker-email="youremail@groland.grd" \
        --docker-password=$sp_password

kubectl get secrets -n $target_namespace

# TODO : Integration with KeyVault, External authentication in an ACR task using an Azure-managed identity
# https://aka.ms/acr/tasks/task-create-managed-identity
# https://docs.microsoft.com/en-us/azure/container-registry/container-registry-tasks-authentication-key-vault

# Store Docker Hub user name
az keyvault secret set \
  --name DockerUserName \
  --value $sp_id \
  --vault-name $vault_name

# Store Docker Hub password
az keyvault secret set \
  --name DockerPassword \
  --value $sp_password \
  --vault-name $vault_name

ACRIdentityName=${appName}ACRTasksIdentity
echo "ACR Identity Name " :  $ACRIdentityName
az identity create -g $rg_acr_name --name $ACRIdentityName

# Get resource ID of the user-assigned identity
IdentityResourceID=$(az identity show -g $rg_acr_name --name  $ACRIdentityName --query id --output tsv)

# Get principal ID of the task's user-assigned identity
IdentityPrincipalID=$(az identity show -g $rg_acr_name --name  $ACRIdentityName --query principalId --output tsv)

# Get client ID of the user-assigned identity
IdentityClientID=$(az identity show -g $rg_acr_name --name  $ACRIdentityName --query clientId --output tsv)

echo "ACR Identity Resource ID " : $IdentityResourceID
echo "ACR Identity Principal ID " : $IdentityPrincipalID
echo "ACR Identity Client ID " : $IdentityClientID

# Create role assignment
az role assignment create --assignee $IdentityPrincipalID --role acrpull --scope $acr_registry_id

az acr task credential add --name helloworld \
    --registry $acr_registry_name \
    --login-server $docker_server \
    --use-identity $IdentityClientID

az acr task run

az acr task create \
  --name dockerhubtask \
  --registry $acr_registry_name \
  --context /dev/null \
  --file dockerhubtask.yaml \
  --assign-identity $IdentityResourceID

az acr build -t "${docker_server}/spring-petclinic:{{.Run.ID}}" -r $acr_registry_name -g $rg_acr_name --file Dockerfile .
az acr repository list --name $acr_registry_name

build_id=$(az acr task list-runs --registry $acr_registry_name -o json --query "[0].name" )
echo "Successfully pushed image with ID " $build_id

```

## Create Kubernetes deployment & Test container

<span style="color:red">/!\ IMPORTANT : the container image name is hardcoded and must be replaced, the Run ID was provided at the end of acr build command: 
${registryname}.azurecr.io/spring-petclinic:{{.Run.ID}}</span>

[https://docs.microsoft.com/en-us/azure/container-registry/container-registry-helm-repos](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-helm-repos)

```sh
#az acr run -r $acr_registry_name --cmd "${docker_server}/spring-petclinic:dd2" /dev/null
kubectl apply -f petclinic-deployment.yaml -n $target_namespace
kubectl get deployments -n $target_namespace
kubectl get deployment petclinic -n $target_namespace 
kubectl get pods -l app=petclinic -o wide -n $target_namespace
kubectl get pods -l app=petclinic -o yaml -n $target_namespace | grep podIP

# check eventual errors:
k get events -n $target_namespace | grep -i "Error"

```
If there were no errors, you can skip the snippet below. In case of error, use the snippet below to found the error
```sh
for pod in $(k get po -n $target_namespace -o=name)
do
	k describe $pod | grep -i "Error"
	k logs $pod | grep -i "Error"
    k exec $pod -n $target_namespace -- wget http://localhost:8080/manage/health
    k exec $pod -n $target_namespace -- wget http://localhost:8080/manage/info
    # k exec $pod -n $target_namespace -it -- /bin/sh
done

# kubectl describe pod petclinic-YOUR-POD-ID -n $target_namespace
# kubectl logs petclinic-YOUR-POD-ID -n $target_namespace
# kubectl  exec -it "POD-UID" -n $target_namespace -- /bin/sh

```

Now you have several option to expose your app :
- [PUBLIC service](#create-kubernetes-public-service) through a Standard Load Balancer & public IP
- [Internal service](#create-kubernetes-internal-service) + [Ingress](#Create-Petclinic-Ingress)
- [Use App. Gateway Ingress Controller](#AGIC)

## Create Kubernetes PUBLIC service
```sh
kubectl apply -f petclinic-service-lb.yaml -n $target_namespace
k get svc -n $target_namespace -o wide

# Standard load Balancer Use Case
# Use the command below to retrieve the External-IP of the Service. Make sure to allow a couple of minutes for the Azure Load Balancer to assign a public IP.
service_ip=$(kubectl get service petclinic-lb-service -n $target_namespace -o jsonpath="{.status.loadBalancer.ingress[*].ip}")
# All config proiperties ref: sur https://docs.spring.io/spring-boot/docs/current/reference/html/common-application-properties.html 
echo "Your service is now exposed through a Cluster IP at http://${service_ip}"
echo "Check Live Probe with Spring Actuator : http://${service_ip}/manage/health"
curl "http://${service_ip}/manage/health" -i -X GET
echo "\n"
# You should received an UP reply :
# {
#  "status" : "UP"
# }
echo "Check spring Management Info at http://${service_ip}/manage/info" -i -X GET
curl "http://${service_ip}/manage/info" -i -X GET
```


## Create Kubernetes INTERNAL service

```sh
kubectl apply -f petclinic-service-cluster-ip.yaml -n $target_namespace
k get svc -n $target_namespace -o wide

# Use the command below to retrieve the Cluster-IP of the Service.
service_ip=$(kubectl get service petclinic-internal-service -n $target_namespace -o jsonpath="{.spec.clusterIP}")
kubectl get endpoints petclinic-lb-service -n $target_namespace
```


### Create Petclinic Ingress

If not already done, apply [HELM Setup](setup-helm.md)
#### Use HELM to setup the Ingress Controller
```sh
kubectl create namespace ingress
# https://www.nginx.com/products/nginx/kubernetes-ingress-controller
helm install ingress stable/nginx-ingress --namespace ingress
helm upgrade --install ingress stable/nginx-ingress --namespace ingress
ing_ctl_ip=$(kubectl get svc -n ingress ingress-nginx-ingress-controller -o jsonpath="{.status.loadBalancer.ingress[*].ip}")
```

```sh
kubectl apply -f petclinic-ingress.yaml -n $target_namespace
kubectl get ingresses --all-namespaces
kubectl describe ingress petclinic -n $target_namespace

# All config proiperties ref: sur https://docs.spring.io/spring-boot/docs/current/reference/html/common-application-properties.html 
echo "Your service is now exposed through an Ingress Controller at http://${ing_ctl_ip}"
echo "Check Live Probe with Spring Actuator : http://petclinic.${ing_ctl_ip}.nip.io/manage/health"
curl "http://petclinic.${ing_ctl_ip}.nip.io/manage/health" -i -X GET
echo ""
echo ""
# You should received an UP reply :
# {
#  "status" : "UP"
# }
echo "Check spring Management Info at http://petclinic.${ing_ctl_ip}.nip.io/manage/info" -i -X GET
curl "http://petclinic.${ing_ctl_ip}.nip.io/manage/info" -i -X GET

```

## AGIC

Firstly, you need to setup AGIC [Setup AGIC](setup-agic.md)
```sh
# TODO

kubectl apply -f petclinic-ingress-agic.yaml -n $target_namespace
kubectl get ingresses --all-namespaces
kubectl describe ingress petclinic -n $target_namespace

# All config proiperties ref: sur https://docs.spring.io/spring-boot/docs/current/reference/html/common-application-properties.html 
echo "Your service is now exposed through an Ingress Controller at http://${ing_ctl_ip}"
echo "Check Live Probe with Spring Actuator : http://petclinic.${ing_ctl_ip}.nip.io/manage/health"
curl "http://petclinic.${ing_ctl_ip}.nip.io/manage/health" -i -X GET
echo ""
echo ""
# You should received an UP reply :
# {
#  "status" : "UP"
# }
echo "Check spring Management Info at http://petclinic.${ing_ctl_ip}.nip.io/manage/info" -i -X GET
curl "http://petclinic.${ing_ctl_ip}.nip.io/manage/info" -i -X GET


```

### Configure DNS

Free DNS: http://xip.io , https://nip.io, https://freedns.afraid.org, https://dyn.com/dns
```sh
$custom_dns
#service_ip=$(kubectl get service petclinic-lb-service -n $target_namespace -o jsonpath="{.status.loadBalancer.ingress[*].ip}")
service_ip=$(kubectl get svc -n ingress ingress-nginx-ingress-controller -o jsonpath="{.status.loadBalancer.ingress[*].ip}")
echo $service_ip

public_ip_id=$(az network public-ip list --subscription $subId --resource-group $managed_rg --query "[?ipAddress!=null]|[?contains(ipAddress, '$service_ip')].[id]" --output tsv)
echo $public_ip_id

In the Azure portal, go to All services / Public IP addresses / kubernetes-xxxx - Configuration ( the Ingress Controller IP) , then there is a field "DNS name label (optional)" ==> An "A record" that starts with the specified label and resolves to this public IP address will be registered with the Azure-provided DNS servers. Example: mylabel.westus.cloudapp.azure.com.

az network public-ip show --ids $public_ip_id --subscription $subId --resource-group $managed_rg

az network public-ip update --ids $public_ip_id --dns-name $dns_zone --subscription $subId --resource-group $managed_rg

#http://petclinic.${location}.cloudapp.azure.com
#http://petclinic.internal.cloudapp.net

#http://petclinic.kissmyapp.${location}.cloudapp.azure.com/ 
dns_zone="kissmyapp.${location}.cloudapp.azure.com" 
az network dns zone create -g $rg_name -n $dns_zone
az network dns zone list -g $rg_name
az network dns record-set a add-record -g $rg_name -z $dns_zone -n www -a ${service_ip}
az network dns record-set list -g $rg_name -z $dns_zone

az network dns record-set cname create -g $rg_name -z $dns_zone -n petclinic-ingress
az network dns record-set cname set-record -g $rg_name -z $dns_zone -n petclinic-ingress -c www.$dns_zone
az network dns record-set cname show -g $rg_name -z $dns_zone -n petclinic-ingress
http://petclinic-ingress.kissmyapp.${location}.cloudapp.azure.com/ 


