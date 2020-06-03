
See also :
- []()
- [https://docs.microsoft.com/en-us/azure/storage/blobs/storage-quickstart-blobs-cli](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-quickstart-blobs-cli)
- [https://docs.microsoft.com/en-us/cli/azure/storage/blob?view=azure-cli-latest](https://docs.microsoft.com/en-us/cli/azure/storage/blob?view=azure-cli-latest)

# Create Storage

```sh
blob_str_name="str""${appName,,}"
container_name="blob-container"

az storage account create --name $blob_str_name --kind StorageV2 --sku Standard_LRS --encryption-services blob --location $location -g $rg_name 
str_acc_id=$(az storage account show --name $blob_str_name --resource-group $rg_name --query "id" --output tsv)
echo "Storage Account ID :" $str_acc_id

az storage account list -g $rg_name

az storage container create --account-name $blob_str_name --name $container_name --auth-mode login --public-access off

```

## Setup Private-Endpoint

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

az network private-link-resource list --id $str_acc_id

# The private-endpoint must be created in the Consumer VNet/Subnet, so it will be the ARO Workers $worker_subnet_id
az network private-endpoint create \
    --name $storage_private_endpoint_name \
    --resource-group $rg_name \
    --subnet $worker_subnet_id \
    --private-connection-resource-id $str_acc_id \
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

```

## Setup DNS

```sh
# az network private-dns record-set a create --name $blob_str_name --zone-name privatelink.blob.core.windows.net -g $rg_name

az network private-dns record-set a add-record -g $rg_name \
  --record-set-name $blob_str_name \
  --zone-name privatelink.blob.core.windows.net \
  --ipv4-address $storage_network_interface_private_ip

# storage_public=$(az storage account show --name $blob_str_name -g $rg_name --query "?? IP ??" --output tsv)
# echo "Storage Public :" $storage_public

storage_private="${blob_str_name}.privatelink.blob.core.windows.net"
echo "Storage private :" $storage_private

nslookup ${blob_str_name}.blob.core.windows.net
nslookup $storage_private

```

## Test

```sh

# https://docs.microsoft.com/en-us/azure/storage/common/storage-auth-aad?toc=/azure/storage/blobs/toc.json

az storage blob upload \
    --account-name $blob_str_name \
    --container-name $container_name \
    --name helloworld \
    --file helloworld.txt \
    --auth-mode login

az storage blob list \
    --account-name $blob_str_name \
    --container-name $container_name \
    --output table \
    --auth-mode login

az storage blob download \
    --account-name $blob_str_name \
    --container-name $container_name \
    --name helloworld \
    --file ~/destination/path/for/file \
    --auth-mode login

```