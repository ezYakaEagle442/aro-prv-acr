

```sh

pull_secret=`cat pull-secret.txt`

az provider show -n  Microsoft.RedHatOpenShift --query  "resourceTypes[?resourceType == 'OpenShiftClusters']".locations 

az aro create \
  --name $cluster_name \
  --vnet $vnet_name \
  --master-subnet $master_subnet_id	\
  --worker-subnet $worker_subnet_id \
  --apiserver-visibility $apiserver_visibility \
  --ingress-visibility  $ingress_visibility \
  --location $location \
  --pod-cidr $pod_cidr \
  --service-cidr $svc_cidr \
  --pull-secret @pull-secret.txt \
  --worker-count 3 \
  --resource-group $rg_name 

az aro list -g $rg_name
az aro show -n $cluster_name -g $rg_name

aro_api_server_url=$(az aro show -n $cluster_name -g $rg_name --query 'apiserverProfile.url' -o tsv)
echo "ARO API server URL: " $aro_api_server_url

aro_version=$(az aro show -n $cluster_name -g $rg_name --query 'clusterProfile.version' -o tsv)
echo "ARO version : " $aro_version

aro_console_url=$(az aro show -n $cluster_name -g $rg_name --query 'consoleProfile.url' -o tsv)
echo "ARO console URL: " $aro_console_url

ing_ctl_ip=$(az aro show -n $cluster_name -g $rg_name --query 'ingressProfiles[0].ip' -o tsv)
echo "ARO Ingress Controller IP: " $ing_ctl_ip

aro_spn=$(az aro show -n $cluster_name -g $rg_name --query 'servicePrincipalProfile.clientId' -o tsv)
echo "ARO Service Principal Name: " $aro_spn

managed_rg=$(az aro show -n $cluster_name -g $rg_name --query 'clusterProfile.resourceGroupId' -o tsv)
echo "ARO Managed Resource Group : " $managed_rg

managed_rg_name=`echo -e $managed_rg | cut -d  "/" -f5`
echo "ARO RG Name" $managed_rg_name

cat ~/.azure/accessTokens.json
# You can have a look at the App. Registrations in the portal at https://ms.portal.azure.com/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/RegisteredApps

```
## Connect to the Cluster

See [https://docs.microsoft.com/en-us/azure/openshift/tutorial-connect-cluster#connect-to-the-cluster](https://docs.microsoft.com/en-us/azure/openshift/tutorial-connect-cluster#connect-to-the-cluster)

```sh
az aro list-credentials -n $cluster_name -g $rg_name
aro_usr=$(az aro list-credentials -n $cluster_name -g $rg_name | jq -r '.kubeadminUsername')
aro_pwd=$(az aro list-credentials -n $cluster_name -g $rg_name | jq -r '.kubeadminPassword')

# Launch the console URL in a browser and login using the kubeadmin credentials.

```

## Install the OpenShift CLI

See [https://docs.microsoft.com/en-us/azure/openshift/tutorial-connect-cluster#install-the-openshift-cli](https://docs.microsoft.com/en-us/azure/openshift/tutorial-connect-cluster#install-the-openshift-cli)
```sh
cd ~

aro_download_url=${aro_console_url/console/downloads}
echo "aro_download_url" $aro_download_url

wget $aro_download_url/amd64/linux/oc.tar

mkdir openshift
tar -xvf oc.tar -C openshift
echo 'export PATH=$PATH:~/openshift' >> ~/.bashrc && source ~/.bashrc
oc version

source <(oc completion bash)
echo "source <(oc completion bash)" >> ~/.bashrc 

oc login $aro_api_server_url -u $aro_usr -p $aro_pwd
oc whoami
oc cluster-info

```

## Create Namespaces
```sh
oc create namespace development
oc label namespace/development purpose=development

oc create namespace staging
oc label namespace/staging purpose=staging

oc create namespace production
oc label namespace/production purpose=production

oc create namespace sre
oc label namespace/sre purpose=sre

oc get namespaces
oc describe namespace production
oc describe namespace sre
```

## Optionnal Play: what resources are in your cluster

```sh
oc get nodes

# https://docs.microsoft.com/en-us/azure/aks/availability-zones#verify-node-distribution-across-zones
oc describe nodes | grep -e "Name:" -e "failure-domain.beta.kubernetes.io/zone"

oc get pods
oc top node
oc api-resources --namespaced=true
oc api-resources --namespaced=false
oc get crds

oc get serviceaccounts --all-namespaces
oc get roles --all-namespaces
oc get rolebindings --all-namespaces
oc get ingresses  --all-namespaces

oc create serviceaccount api-service-account
oc apply -f ./cnf/clusterRole.yaml

sa_secret_name=$(oc get serviceaccount api-service-account  -o json | jq -Mr '.secrets[].name')
echo "SA secret name " $sa_secret_name

token_secret_value=$(oc get secrets  $sa_secret_name -o json | jq -Mr '.items[0].data.token' | base64 -d)
echo "SA secret  " $token_secret_value

# kube_url=$(oc get endpoints -o jsonpath='{.items[0].subsets[0].addresses[0].ip}')
# echo "Kube URL " $kube_url

curl -k $aro_api_server_url/api/v1/namespaces -H "Authorization: Bearer $token_secret_value" -H 'Accept: application/json'
curl -k $aro_api_server_url/apis/user.openshift.io/v1/users/~ -H "Authorization: Bearer $token_secret_value" -H 'Accept: application/json'

```


## [Configuring the registry for Azure storage](https://docs.openshift.com/container-platform/4.4/registry/configuring_registry_storage/configuring-registry-storage-azure-user-infrastructure.html)


<span style="color:red">/!\ IMPORTANT </span> : You are not allowed to create resources in the ARO Managed RG. You not allowed neither to use the Azure Storage Account created by default in ARO for the built-in Registry, so you have to create a Azure Storage Account if you want to create a Private-Endpoint for the Storage.

See :
- [ARO doc "Configuring the registry for Azure user-provisioned infrastructure"](https://docs.openshift.com/container-platform/4.4/registry/configuring_registry_storage/configuring-registry-storage-azure-user-infrastructure.html)
- [Azure doc "Authorize requests to Azure Storage"](https://docs.microsoft.com/en-us/rest/api/storageservices/authorize-requests-to-azure-storage)
- [Azure doc "Shared Access Signature](https://docs.microsoft.com/en-us/azure/storage/common/storage-sas-overview)
- [Azure doc "SAS CLI sample"](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-blob-user-delegation-sas-create-cli)
- [https://docs.microsoft.com/en-us/cli/azure/storage/container?view=azure-cli-latest#az-storage-container-generate-sas](https://docs.microsoft.com/en-us/cli/azure/storage/container?view=azure-cli-latest#az-storage-container-generate-sas)
- When you create a storage account, [Azure generates two 512-bit storage account access keys](https://docs.microsoft.com/en-us/azure/storage/common/storage-account-keys-manage). These keys can be used to authorize access to data in your storage account via Shared Key authorization. Microsoft recommends that you use Azure Key Vault to manage your access keys, and that you regularly rotate and regenerate your keys. Using Azure Key Vault makes it easy to rotate your keys without interruption to your applications.

### Create Storage

```sh
aro_registry_blob_str_name="straroreg$(uuidgen | cut -d '-' -f5 | tr '[A-Z]' '[a-z]')"
aro_registry_container_name="aro-registry-container"

az storage account create --name $aro_registry_blob_str_name --kind StorageV2 --sku Standard_ZRS --encryption-services blob --location $location -g $rg_name 
aro_reg_str_acc_id=$(az storage account show --name $aro_registry_blob_str_name --resource-group $rg_name --query "id" --output tsv)
echo "ARO Registry Storage Account ID :" $aro_reg_str_acc_id

az storage account list -g $rg_name
az storage container create --name $aro_registry_container_name --account-name $aro_registry_blob_str_name --auth-mode login # --public-access off
az storage container set-permission --name $aro_registry_container_name  --account-name $aro_registry_blob_str_name  --public-access off

```

```sh

az role assignment create \
    --role "Storage Blob Data Contributor" \
    --scope "/subscriptions/$subId/resourceGroups/$rg_name/providers/Microsoft.Storage/storageAccounts/$aro_registry_blob_str_name" \
    --assignee pinpin@ms.grd # put your <email>

str_acc_access_key1=$(az storage account keys list -g $rg_name --account-name $aro_registry_blob_str_name --query [0].value -o tsv)
str_acc_access_key2=$(az storage account keys list -g $rg_name --account-name $aro_registry_blob_str_name --query [1].value -o tsv)

# tzselect
# end_date=`date -u -d "30 minutes" '+%Y-%m-%dT%H:%M:%SZ'` # 2020-06-03T20:19Z
# end_date=`date --date 'TZ="Europe/Paris" 2020-07-01T17:00:00'` 
end_date=`date --iso-8601=seconds --date "2020-07-01T17:00:00"` # +%FT%T%z

# Define your own IP range
AUTHORIZED_IP_RANGE="192.168.0.0-192.168.7.255" # 176.134.171.0/24 | 172.16.2.0/24 | 192.168.1.0/24

# expiry : specifies the UTC datetime (Y-m-d'T'H:M'Z')
# Because the maximum interval over which the user delegation key is valid is 7 days from the start date, you should specify an expiry time for the SAS that is within 7 days of the start time. The SAS is invalid after the user delegation key expires, so a SAS with an expiry time of greater than 7 days will still only be valid for 7 days.

aro_registry_container_sas=$(az storage container generate-sas \
    --account-name $aro_registry_blob_str_name \
    --name $aro_registry_container_name \
    --account-key $str_acc_access_key1 \
    --permissions acdlrw \
    --expiry "2020-06-10T17:00Z" \
    --auth-mode login \
    --https-only \
    --ip $AUTHORIZED_IP_RANGE)
    # --as-user

echo "aro_registry_container_sas : " $aro_registry_container_sas

```

### Setup Private-Endpoint

```sh
az network private-dns zone create --name "privatelink.blob.core.windows.net" -g $rg_name

# Create an association link: 
# virtual-network is the consumer VNet, so it will be the ARO VNet $vnet_name
az network private-dns link vnet create \
  --resource-group $rg_name \
  --zone-name "privatelink.blob.core.windows.net" \
  --name $storage_private_dns_link_name \
  --virtual-network $vnet_name \
  --registration-enabled false

private_dns_link_id=$(az network private-dns link vnet show --name $storage_private_dns_link_name --zone-name "privatelink.blob.core.windows.net" -g $rg_name --query "id" --output tsv)
echo "Private-Link DNS ID :" $private_dns_link_id

# do the same for the Bastion ...
az network private-dns link vnet create --name $storage_bastion_private_dns_link_name --virtual-network $bastion_vnet_id --zone-name "privatelink.blob.core.windows.net" --registration-enabled false -g $rg_name

bastion_private_dns_link_id=$(az network private-dns link vnet show --name $storage_bastion_private_dns_link_name --zone-name "privatelink.blob.core.windows.net" -g $rg_name --query "id" --output tsv)
echo "Bastion Private-Link DNS ID :" $bastion_private_dns_link_id

# az network private-link-resource list --id $aro_reg_str_acc_id

# The private-endpoint must be created in the Consumer VNet/Subnet, so it will be the ARO Workers $worker_subnet_id
az network private-endpoint create \
    --name $storage_private_endpoint_name \
    --resource-group $rg_name \
    --subnet $worker_subnet_id \
    --private-connection-resource-id $aro_reg_str_acc_id \
    --group-id blob \
    --location $location \
    --connection-name $storage_private_endpoint_svc_con_name

storage_private_endpoint_id=$(az network private-endpoint show --name $storage_private_endpoint_name -g $rg_name --query id -o tsv)
echo "Storage private-endpoint ID :" $storage_private_endpoint_id

network_interface_id=$(az network private-endpoint show --name $storage_private_endpoint_name -g $rg_name --query 'networkInterfaces[0].id' -o tsv)
echo "Storage Network Interface ID :" $network_interface_id

storage_network_interface_private_ip=$(az resource show --ids $network_interface_id \
  --api-version 2019-04-01 --query 'properties.ipConfigurations[0].properties.privateIPAddress' --output tsv)
echo "Storage Network Interface private IP :" $storage_network_interface_private_ip

# az storage account private-endpoint-connection show --name $storage_private_endpoint_name --account-name $blob_str_name -g $rg_name
# az storage account private-endpoint-connection approve --name $storage_private_endpoint_name --account-name $blob_str_name -g $rg_name

# The private-endpoint must be created in the Consumer VNet/Subnet, so it will be also in the Bastion $bastion_subnet_id
az network vnet subnet update --name $subnet_bastion_name --vnet-name $vnet_bastion_name --disable-private-endpoint-network-policies true -g $rg_bastion_name

az network private-endpoint create \
    --name $storage_bastion_private_endpoint_name \
    --resource-group $rg_name \
    --subnet $bastion_subnet_id \
    --private-connection-resource-id $aro_reg_str_acc_id \
    --group-id blob \
    --location $location \
    --connection-name $storage_bastion_private_endpoint_svc_con_name

storage_private_endpoint_id=$(az network private-endpoint show --name $storage_bastion_private_endpoint_name -g $rg_name --query id -o tsv)
echo "Storage private-endpoint ID :" $storage_private_endpoint_id

network_interface_id=$(az network private-endpoint show --name $storage_bastion_private_endpoint_name -g $rg_name --query 'networkInterfaces[0].id' -o tsv)
echo "Storage Network Interface ID :" $network_interface_id

storage_network_interface_private_ip=$(az resource show --ids $network_interface_id \
  --api-version 2019-04-01 --query 'properties.ipConfigurations[0].properties.privateIPAddress' --output tsv)
echo "Storage Network Interface private IP :" $storage_network_interface_private_ip
```

### Setup DNS

```sh
az network private-dns record-set a create --name $aro_registry_blob_str_name --zone-name privatelink.blob.core.windows.net -g $rg_name

az network private-dns record-set a add-record -g $rg_name \
  --record-set-name $aro_registry_blob_str_name \
  --zone-name privatelink.blob.core.windows.net \
  --ipv4-address $storage_network_interface_private_ip

storage_private="${aro_registry_blob_str_name}.privatelink.blob.core.windows.net"
echo "Storage private :" $storage_private

nslookup ${aro_registry_blob_str_name}.blob.core.windows.net
nslookup $storage_private
```

### Update ARO Cluster Config
```sh
oc create secret generic image-registry-private-configuration-user --from-literal=REGISTRY_STORAGE_AZURE_ACCOUNTKEY=$str_acc_access_key1 --namespace openshift-image-registry
oc get secret image-registry-private-configuration-user -n openshift-image-registry
oc describe secret image-registry-private-configuration-user -n openshift-image-registry

# Get the Azure Storage Account created by default in ARO
oc get config cluster
storage_acct_name=$(oc describe config cluster | grep -i "Account Name:")
aro_azure_storage_acct=`echo -e $storage_acct_name | cut -d  ":" -f2 | cut -d  " " -f2`
echo "ARO default Azure Storage Account Name " $aro_azure_storage_acct

aro_azure_storage_acct_id=$(az storage account show --name $aro_azure_storage_acct -g $managed_rg_name --query "id" --output tsv)
echo "ARO default Azure Storage Account ID :" $aro_azure_storage_acct_id

# Now Update ARO Cluster Config, see https://docs.openshift.com/container-platform/4.4/registry/configuring_registry_storage/configuring-registry-storage-azure-user-infrastructure.html
oc edit configs.imageregistry.operator.openshift.io/cluster

# You will see fields like this :
storage:
  azure:
    accountName: <account-name> # ==> replace with NEW $aro_registry_blob_str_name
    container: <container-name> # ==> replace with NEW $aro_registry_container_name

```

### Setup Storage Firewall

- [https://docs.microsoft.com/en-us/azure/storage/common/storage-network-security#grant-access-from-an-internet-ip-range](https://docs.microsoft.com/en-us/azure/storage/common/storage-network-security#grant-access-from-an-internet-ip-range)

IP network rules are only allowed for public internet IP addresses. IP address ranges reserved for private networks (as defined in RFC 1918) aren't allowed in IP rules. Private networks include addresses that start with 10.*, 172.16.* - 172.31.*, and 192.168.*.

```sh
az storage account update --name $aro_registry_blob_str_name --default-action deny
az storage account network-rule list --account-name $aro_registry_blob_str_name 
az storage account network-rule add --action allow --account-name $aro_registry_blob_str_name --subnet $worker_subnet_id -g $rg_name

jumpoff_management_subnet_id=$(az network vnet subnet show --name ManagementSubnet --vnet-name $vnet_bastion_name -g $rg_bastion_name --query id -o tsv)
echo "Bastion Subnet Id :" $jumpoff_management_subnet_id	
az storage account network-rule add --action allow --account-name $aro_registry_blob_str_name --subnet $jumpoff_management_subnet_id  -g $rg_name

# Define your own IP range
#AUTHORIZED_IP_RANGE="176.134.171.0-176.134.171.255" # 176.134.171.0/24 | 172.16.2.0/24 | 192.168.1.0/24 | 192.168.0.0-192.168.7.255
#az storage account network-rule add --action allow --ip-address $AUTHORIZED_IP_RANGE --account-name $aro_registry_blob_str_name -g $rg_name
az storage account network-rule list --account-name $aro_registry_blob_str_name 

```


### Test from Bastion

```sh
AUTHORIZED_IP_RANGE="42.42.42.0/24" # Set your own PUBLIC range
az storage account update --name $blob_str_name --default-action deny
az storage account network-rule list --account-name $blob_str_name 
az storage account network-rule add --action allow --account-name $blob_str_name --subnet $bastion_subnet_id -g $rg_name
# az storage account network-rule add --action allow --account-name $blob_str_name --ip-address $AUTHORIZED_IP_RANGE -g $rg_name

# https://docs.microsoft.com/en-us/azure/storage/common/storage-auth-aad?toc=/azure/storage/blobs/toc.json

file_to_upload="helloworldtest_in.txt"
file_to_download="helloworldtest_out.txt" #~/destination/path/to/outputfile

echo -e "Hello Pinpin from PE" > $file_to_upload

az storage blob upload \
    --account-name $aro_registry_blob_str_name \
    --container-name $aro_registry_container_name \
    --name helloworldtest \
    --file $file_to_upload \
    --auth-mode login

az storage blob list \
    --account-name $aro_registry_blob_str_name \
    --container-name $aro_registry_container_name \
    --output table \
    --auth-mode login

az storage blob download \
    --account-name $aro_registry_blob_str_name \
    --container-name $aro_registry_container_name \
    --name helloworldtest \
    --file $file_to_download \
    --auth-mode login

```


## Expose the built-in integrated OpenShift container registry with a Route

- [ARO docs "Exposing the registry"](https://docs.openshift.com/aro/4/registry/securing-exposing-registry.html) states: Unlike previous versions of OpenShift Container Platform, the registry is not exposed outside of the cluster at the time of installation.
- You need to have configured an identity provider (like Azure AD) and [assign user access](https://docs.openshift.com/aro/4/registry/accessing-the-registry.html). 

### Configure AAD as the Identity Provider (IDP)

I have automated a snippet inspired by [Stuart](https://github.com/stuartatmicrosoft/azure-aro/blob/master/aro4-aad-connect.sh)
```sh
bash ./cnf/setup-aro-idp-aad.sh $cluster_name $rg_name
```
You can then check in the Portal / [App Registration](https://portal.azure.com/#blade/Microsoft_AAD_RegisteredApps/ApplicationsListBlade)

### ARO Config

```sh

oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge
aro_reg_default_route=$(oc get route default-route -n openshift-image-registry -o json | jq -Mr '.status.ingress[0].host')
echo "ARO Registry default route : " $aro_reg_default_route

oc policy add-role-to-user registry-viewer <user_name> # To pull images
oc policy add-role-to-user registry-editor <user_name> # To Push images
oc login $aro_api_server_url -u $aro_usr -p $aro_pwd
docker login -u $aro_usr -p $(oc whoami -t) https://image-registry.openshift-image-registry.svc:5000
docker login -u pinpin@ms.grd -p $token_secret_value $aro_reg_default_route

# https://docs.openshift.com/aro/4/registry/accessing-the-registry.html#registry-accessing-metrics_accessing-the-registry
curl --insecure -s -u $aro_usr -p $(oc whoami -t) https://image-registry.openshift-image-registry.svc:5000/extensions/v2/metrics | grep imageregistry | head -n 20
curl --insecure -s -u pinpin@ms.grd -p $token_secret_value https://$aro_reg_default_route/extensions/v2/metrics | grep imageregistry | head -n 20
```

### Sample SpringBoot App Build Test 

Go to [build-java-app.md](build-java-app.md)