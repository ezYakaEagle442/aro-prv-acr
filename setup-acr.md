# Create Azure Container Registry

See also :
- [Notary v2 project - cross-industry initiative](https://github.com/notaryproject/requirements)
- ACR supports [Content-Trust](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-content-trust)

## Create Azure Container Registry
Note: 
- Premium sku is a requirement to enable replication, and also to leverage ACR Firewall
- ACR [Private Link support](https://aka.ms/acr/privatelink): This feature is available in the **Premium** container registry service tier
- To use the Azure CLI for ACR Private Link, Azure CLI version 2.2.0 or later is recommended


```sh

# Optionnal Play: Design your Application for HA to support multiple  regions deployment & Enable geo-replication for container images
# Note: Premium sku is a requirement to enable replication
# Configure https://docs.microsoft.com/en-us/azure/container-registry/container-registry-geo-replication#configure-geo-replication
# https://docs.microsoft.com/en-us/cli/azure/acr/replication?view=azure-cli-latest
# location from az account list-locations : francecentral | northeurope | westeurope 

# https://github.com/Azure/azure-quickstart-templates/tree/master/101-container-registry-geo-replication
# az acr create --resource-group $rg_name --name $acr_registry_name --sku Premium --location $location
# az acr replication create --location northeurope --registry $acr_registry_name --resource-group $rg_name

# classic registry without replication, private link neither: az acr create --name $acr_registry_name --sku standard --location $location --resource-group $rg_name 

az provider register --namespace Microsoft.ContainerRegistry

#az monitor log-analytics workspace create --workspace-name $acr_analytics_workspace --location $location -g $rg_name
#acr_analytics_workspace_id=$(az monitor log-analytics workspace show --workspace-name $acr_analytics_workspace -g $rg_name --query "id" --output tsv)
#echo $acr_analytics_workspace_id

# Use Premium sku to enable Private Link & ACR Firewall
az acr create --name $acr_registry_name --sku Premium --location $location -g $rg_name # --workspace $acr_analytics_workspace_id 

# Get the ACR registry resource id
acr_registry_id=$(az acr show --name $acr_registry_name --resource-group $rg_name --query "id" --output tsv)
echo "ACR registry ID :" $acr_registry_id

# run sudo on WSL, otherwise you will get the error below : An error occurred: DOCKER_COMMAND_ERROR
#Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running? Please refer to https://aka.ms/acr/errors#docker_command_error for more information.
# sudo nohup docker daemon -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock &
# sudo dockerd
# sudo service docker stop
# sudo service docker start
az acr repository list --name $acr_registry_name
az acr check-health --yes -n $acr_registry_name 

# https://aka.ms/acr/installaad/bash
```

# Create role assignment

See :
- [https://docs.microsoft.com/en-us/azure/container-registry/container-registry-auth-service-principal](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-auth-service-principal)
- [https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#acrpull](https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#acrpull)
- [https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#acrpush](https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#acrpush)
- [https://github.com/MicrosoftDocs/azure-docs/issues/51672](https://github.com/MicrosoftDocs/azure-docs/issues/51672#issuecomment-630652951)

```sh
az role assignment create --assignee $aro_spn --role acrpull --scope $acr_registry_id
az role assignment create --assignee $aro_spn --role acrpush --scope $acr_registry_id
```


## Setup Private-Link
```sh
az network private-dns zone create --name "privatelink.azurecr.io" -g  $rg_name

# Create an association link: https://docs.microsoft.com/en-us/azure/container-registry/container-registry-private-link#create-an-association-link
# virtual-network is the consumer VNet, so it will be the AKS VNet $vnet_name
az network private-dns link vnet create \
  --resource-group $rg_name \
  --zone-name "privatelink.azurecr.io" \
  --name $acr_private_dns_link_name \
  --virtual-network $vnet_name \
  --registration-enabled false

private_dns_link_id=$(az network private-dns link vnet show --name $acr_private_dns_link_name --zone-name "privatelink.azurecr.io" -g $rg_name --query "id" --output tsv)
echo "Private-Link DNS ID :" $private_dns_link_id

# do the same for the Bastion ...
az network private-dns link vnet create --name $acr_bastion_private_dns_link_name --virtual-network $bastion_vnet_id --zone-name privatelink.azurecr.io --registration-enabled false -g $rg_name

bastion_private_dns_link_id=$(az network private-dns link vnet show --name $acr_bastion_private_dns_link_name --zone-name "privatelink.azurecr.io" -g $rg_name --query "id" --output tsv)
echo "Bastion Private-Link DNS ID :" $bastion_private_dns_link_id

# The private-endpoint must be created in the Consumer VNet/Subnet, so it will be the ARO Workers $worker_subnet_id
az network private-endpoint create \
    --name $acr_private_endpoint_name \
    --resource-group $rg_name \
    --subnet $worker_subnet_id \
    --private-connection-resource-id $acr_registry_id \
    --group-ids registry \
    --location $location \
    --connection-name $acr_private_endpoint_svc_con_name

acr_private_endpoint_id=$(az network private-endpoint show --name $acr_private_endpoint_name -g $rg_name --query id -o tsv)
echo "ACR private-endpoint ID :" $acr_private_endpoint_id

network_interface_id=$(az network private-endpoint show --name $acr_private_endpoint_name -g $rg_name --query 'networkInterfaces[0].id' -o tsv)
echo "ACR Network Interface ID :" $network_interface_id

acr_network_interface_private_ip=$(az resource show --ids $network_interface_id \
  --api-version 2019-04-01 --query 'properties.ipConfigurations[1].properties.privateIPAddress' \
  --output tsv)
echo "ACR Network Interface private IP :" $acr_network_interface_private_ip

acr_data_endpoint_private_ip=$(az resource show --ids $network_interface_id \
  --api-version 2019-04-01 \
  --query 'properties.ipConfigurations[0].properties.privateIPAddress' \
  --output tsv)
echo "ACR Data Endpoint private IP :" $acr_data_endpoint_private_ip


```

## Setup DNS

```sh
az network private-dns record-set a create --name $acr_registry_name --zone-name privatelink.azurecr.io -g $rg_name

# Specify registry region in data endpoint name
az network private-dns record-set a create --name ${acr_registry_name}.${location}.data --zone-name privatelink.azurecr.io -g $rg_name

az network private-dns record-set a add-record -g $rg_name \
  --record-set-name $acr_registry_name \
  --zone-name privatelink.azurecr.io \
  --ipv4-address $acr_network_interface_private_ip

# Specify registry region in data endpoint name
az network private-dns record-set a add-record -g $rg_name \
  --record-set-name ${acr_registry_name}.${location}.data \
  --zone-name privatelink.azurecr.io \
  --ipv4-address $acr_data_endpoint_private_ip

# Validate private link connection
# az acr private-endpoint-connection list --registry-name $acr_registry_name
# From home/public network, you wil get a public IP. If inside a vnet with private zone, then nslookup will resolve to the private ip.
# note: we will have a feature roll out to let you disable the public access, which means “nslookup” will fail outside of vnet  
acr_public=$(az acr show --name $acr_registry_name --resource-group $rg_name --query "loginServer" --output tsv)
echo "ACR Public :" $acr_public

acr_private="${acr_registry_name}.privatelink.azurecr.io"
echo "ACR private :" $acr_private

nslookup $acr_registry_name.azurecr.io
nslookup $acr_private

```

## Setup ACR Firewall
By default, an Azure container registry allows connections from hosts on any network. To limit access to a selected network, change the default action to deny access. Add a network rule to your registry that allows access from the VM's subnet.

[https://docs.microsoft.com/en-us/azure/container-registry/container-registry-vnet](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-vnet)

Preview [Limitations](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-vnet#preview-limitations) :
- Only a Premium container registry can be configured with network access rules.
- Only an ARO cluster or Azure virtual machine can be used as a host to access a container registry in a VNet
- ACR Tasks operations aren't currently supported in a container registry accessed in a virtual network.

```sh

az acr update --name $acr_registry_name --default-action Deny
az acr network-rule add --name $acr_registry_name --subnet $worker_subnet_id

# myip=$(dig +short myip.opendns.com @resolver1.opendns.com)
# export AUTHORIZED_IP_RANGE="176.134.171.0/24"
# echo "IP range allowed to call ACR : " $AUTHORIZED_IP_RANGE
# az acr network-rule add --name $acr_registry_name --ip-address $AUTHORIZED_IP_RANGE
az acr update --name $acr_registry_name --public-network-enabled false
az acr network-rule list --name $acr_registry_name

# Verify access to the registry
az acr login --name  $acr_registry_name 
# az acr login -n $acr_registry_name --expose-token
docker pull $acr_registry_name.azurecr.io/hello-world:v1

```