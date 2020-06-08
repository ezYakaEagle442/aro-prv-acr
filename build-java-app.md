# Build Java SpringBoot App

See :
- [https://docs.openshift.com/aro/4/architecture/understanding-development.html](https://docs.openshift.com/aro/4/architecture/understanding-development.html)
- [https://docs.openshift.com/aro/4/applications/projects/working-with-projects.html](https://docs.openshift.com/aro/4/applications/projects/working-with-projects.html)
- [https://docs.openshift.com/aro/4/openshift_images/images-understand.html](https://docs.openshift.com/aro/4/openshift_images/images-understand.html)
- [https://docs.openshift.com/aro/4/builds/understanding-image-builds.html](https://docs.openshift.com/aro/4/builds/understanding-image-builds.html)
- [https://docs.openshift.com/aro/4/applications/application_life_cycle_management/creating-applications-using-cli.html](https://docs.openshift.com/aro/4/applications/application_life_cycle_management/creating-applications-using-cli.html)
- [https://docs.openshift.com/aro/4/applications/application_life_cycle_management/odc-deleting-applications.html](https://docs.openshift.com/aro/4/applications/application_life_cycle_management/odc-deleting-applications.html)
- [https://docs.openshift.com/aro/4/applications/deployments/what-deployments-are.html](https://docs.openshift.com/aro/4/applications/deployments/what-deployments-are.html)
- [https://aka.ms/aroworkshop-devops](https://aka.ms/aroworkshop-devops)

## Pre-req

```sh
oc config current-context
oc status
oc projects
oc new-project $appName --description="On-Prem Jenkins to build and push Docker image to ARO Built-in Registry configured with a NEW BLOB Stoarge+Private-Endpoint" --display-name="ARO PoC"
oc get ns $appName
oc get project $appName
oc describe project $appName

```

## Build using Source 2 Image (S2I)

See :
-  [https://docs.openshift.com/aro/4/builds/build-strategies.html#images-create-s2i_build-strategies](https://docs.openshift.com/aro/4/builds/build-strategies.html#images-create-s2i_build-strategies)
- [Language Detection](https://docs.openshift.com/aro/4/applications/application_life_cycle_management/creating-applications-using-cli.html#language-detection)

```sh
# https://catalog.redhat.com/software/containers/search?q=tomcat
# https://hub.docker.com/r/jboss/wildfly/dockerfile
# https://hub.docker.com/r/jboss/base/dockerfile ==> yum install curl wget
oc new-app registry.access.redhat.com/jboss-webserver-3/webserver31-tomcat8-openshift:1.4-20~$git_url_springboot --context-dir="/" --strategy=source
# oc new-app registry.access.redhat.com/jboss-webserver-3/webserver31-tomcat8-openshift:latest~$git_url_springboot --context-dir=/petclinic-s2i --strategy=source
oc status --suggest
# oc set probe dc/spring-petclinic --readiness xxxxx
oc set probe dc/spring-petclinic --liveness --get-url=http://:8080/manage
# oc set probe dc/spring-petclinic --remove --readiness --liveness

oc logs -f bc/spring-petclinic

# Check buildConfigs (bc)
oc get bc
oc describe bc spring-petclinic
# oc edit bc spring-petclinic
# oc start-build spring-petclinic

oc get imagestream
oc describe imagestream spring-petclinic

oc get imagestreamtags
oc describe imagestreamtag webserver31-tomcat8-openshift:1.4-20

oc get dc # deploymentconfig
oc describe dc spring-petclinic


for pod in $(oc get pods -n $appName -o custom-columns=:metadata.name)
do
    # oc describe pod $pod -n $appName # | grep -i "Error"
	oc logs $pod -n $appName | grep -i "Error"
    # oc exec $pod -n $appName -- wget http://localhost:8080/manage/health
    # oc exec $pod -n $appName -- wget http://localhost:8080/manage/info
    # k exec $pod -n $appName -it -- /bin/sh #  yum install curl wget
done



oc get svc
# Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
# # cannot use --load-balancer-ip='' with --generator=route/v1  # --external-ip=''
oc expose svc/spring-petclinic --name=spring-petclinic-pub --generator="service/v2" --type=LoadBalancer --load-balancer-ip='' 
oc expose svc/spring-petclinic --name=spring-petclinic-route
oc describe svc spring-petclinic-pub
slb_pub_ip=$(oc get svc spring-petclinic-pub -o jsonpath="{.status.loadBalancer.ingress[0].ip}")
echo "Service Load Balancer Public IP  : " $slb_pub_ip

oc get routes
oc describe route spring-petclinic-route
oc get ep
oc describe ep spring-petclinic

app_route_url=$(oc get route spring-petclinic-route -o jsonpath="{.spec.host}")
echo "App Route URL  : " $app_route_url

oc get images | grep -i "image-registry.openshift-image-registry.svc:5000/$appName/spring-petclinic"

# Check the Content of the BLOB storage /Container :
az storage blob list \
    --account-name $aro_registry_blob_str_name \
    --container-name $aro_registry_container_name \
    --output table \
    --auth-mode login

# Test 
docker login -u pinpin@ms.grd -p $token_secret_value $aro_reg_default_route
docker pull spring-petclinic:latest
```


## xxx

```sh

```

## xxx

```sh

```

## xxx

```sh

```

## xxx

```sh

```

# Build & Push Image to ACR

## Create Docker Image
```sh
# https://docs.microsoft.com/en-us/azure/container-registry/container-registry-quickstart-task-cli
git clone $git_url_springboot 
cd spring-petclinic

# build app
sudo apt install maven --yes
mvn -version
mvn package # -DskipTests

# Test the App
# mvn spring-boot:run

artifact="spring-petclinic-2.2.0.BUILD-SNAPSHOT.jar"
echo -e "FROM mcr.microsoft.com/java/jre:11u6-zulu-alpine\n"\
"VOLUME /tmp \n"\
"ADD target/${artifact} app.jar \n"\
"RUN touch /app.jar \n"\
"EXPOSE 8080 \n"\
"ENTRYPOINT [\""java\"", \""-Djava.security.egd=file:/dev/./urandom\"", \""-jar\"", \""/app.jar\""] \n"\
> Dockerfile

nslookup $acr_registry_name.azurecr.io
docker_server=$(az acr show --name $acr_registry_name --resource-group $rg_name --query "loginServer" --output tsv)
echo "Docker server :" $docker_server

docker_private_server="${acr_registry_name}.privatelink.azurecr.io"
echo "Docker private server :" $docker_private_server


oc -n $target_namespace create secret docker-registry acr-auth \
       --docker-server=$docker_server \
        --docker-username=$aro_spn \
        --docker-email="youremail@groland.grd" \
        --docker-password=$sp_password # Missing !

oc get secrets -n $target_namespace

az acr build -t "${docker_server}/spring-petclinic:{{.Run.ID}}" -r $acr_registry_name -g $rg_name --file Dockerfile .
az acr repository list --name $acr_registry_name


build_id=$(az acr task list-runs --registry $acr_registry_name -o json --query [0].name )
build_id=$(echo $build_id | tr -d '"')
echo "Successfully pushed image with ID " $build_id

az acr task logs --registry $acr_registry_name --run-id  $build_id
```

## Create Kubernetes deployment & Test container
```sh
mkdir deploy
export CONTAINER_REGISTRY=$acr_registry_name
export IMAGE_TAG=$build_id
envsubst < petclinic-deployment.yaml > deploy/petclinic-deployment.yaml 
cat deploy/petclinic-deployment.yaml
kubectl apply -f deploy/petclinic-deployment.yaml -n $target_namespace
kubectl get deployments -n $target_namespace
kubectl get deployment petclinic -n $target_namespace 
kubectl get pods -l app=petclinic -o wide -n $target_namespace
kubectl get pods -l app=petclinic -o yaml -n $target_namespace | grep podIP

# check eventual errors:
k get events -n $target_namespace | grep -i "Error"

```