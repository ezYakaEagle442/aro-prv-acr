## Plan IP addressing for your cluster

See  :

- [Public IP sku comparison](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-ip-addresses-overview-arm#sku)
- [https://docs.openshift.com/aro/4/networking/understanding-networking.html](https://docs.openshift.com/aro/4/networking/understanding-networking.html)
- [https://docs.microsoft.com/fr-fr/cli/azure/aro?view=azure-cli-latest](https://docs.microsoft.com/fr-fr/cli/azure/aro?view=azure-cli-latest)
- [https://docs.openshift.com/aro/4/networking/openshift_sdn/configuring-egress-firewall.html](https://docs.openshift.com/aro/4/networking/openshift_sdn/configuring-egress-firewall.html)
- [https://docs.openshift.com/aro/4/networking/configuring_ingress_cluster_traffic/overview-traffic.html](https://docs.openshift.com/aro/4/networking/configuring_ingress_cluster_traffic/overview-traffic.html)

```sh
# https://www.ipaddressguide.com/cidr


``` 

## Create Networks

```sh
# ARO nodes VNet & Subnet
az network vnet create --name $vnet_name --resource-group $rg_name --address-prefixes 172.16.0.0/21 --location $location
az network vnet subnet create --name $master_subnet_name --address-prefixes 172.16.1.0/24 --vnet-name $vnet_name --resource-group $rg_name --service-endpoints Microsoft.ContainerRegistry
az network vnet subnet create --name $worker_subnet_name --address-prefixes 172.16.2.0/24 --vnet-name $vnet_name -g $rg_name --service-endpoints Microsoft.ContainerRegistry

vnet_id=$(az network vnet show --resource-group $rg_name --name $vnet_name --query id -o tsv)
echo "VNet Id :" $vnet_id	

master_subnet_id=$(az network vnet subnet show --name $master_subnet_name --vnet-name $vnet_name  -g $rg_name --query id -o tsv)
echo "Master Subnet Id :" $master_subnet_id	

worker_subnet_id=$(az network vnet subnet show --name $worker_subnet_name --vnet-name $vnet_name -g $rg_name --query id -o tsv)
echo "Worker Subnet Id :" $worker_subnet_id	

gway_subnet_id=$(az network vnet subnet create --name GatewaySubnet --address-prefixes 172.16.3.0/24 --vnet-name $vnet_name -g $rg_name  --query id -o tsv)
echo "Gateway Subnet Id :" $gway_subnet_id	

# management_subnet_id=$(az network vnet subnet create --name ManagementSubnet --address-prefixes 172.16.4.0/24 --vnet-name $vnet_name -g $rg_name  --query id -o tsv)
# echo "Management Subnet Id :" $management_subnet_id	

# https://docs.microsoft.com/en-us/azure/private-link/create-private-link-service-cli#disable-private-link-service-network-policies-on-subnet
az network vnet subnet update --name $master_subnet_name --vnet-name $vnet_name --disable-private-link-service-network-policies true -g $rg_name

# https://docs.microsoft.com/en-us/azure/container-registry/container-registry-private-link#disable-network-policies-in-subnet
az network vnet subnet update --name $worker_subnet_name --vnet-name $vnet_name --disable-private-endpoint-network-policies true -g $rg_name

# This is the subnet that will be used for Services that are exposed via an Internal Load Balancer (ILB). This mean the ILB internal IP will be from this subnet address space. By doing it this way we do not take away from the existing IP Address space in the AKS subnet that is used for Nodes and Pods.
az network vnet subnet create --name $ilb_subnet_name --address-prefixes 172.16.7.0/24 --vnet-name $vnet_name -g $rg_name 
ilb_subnet_id=$(az network vnet subnet show --resource-group $rg_name --vnet-name  $vnet_name --name $ilb_subnet_name --query id -o tsv)
echo "Internal Load BalancerLB Subnet Id :" $ilb_subnet_id	


```

## Setup Bastion


### Create RG
```sh
az group create --name $rg_bastion_name --location $location
```

### Prepare pre-requisites
[Azure Bastion](https://docs.microsoft.com/en-us/azure/bastion/bastion-overview) allows you to log into VMs in the virtual network through SSH or remote desktop protocol (RDP) without exposing the VMs directly to the internet. Use Bastion to manage the VMs in the virtual network.

RDP and SSH directly in Azure portal: You can directly get to the RDP and SSH session directly in the Azure portal using a single click seamless experience.

Remote Session over SSL and firewall traversal for RDP/SSH: Azure Bastion uses an HTML5 based web client that is automatically streamed to your local device, so that you get your RDP/SSH session over SSL on port 443 enabling you to traverse corporate firewalls securely.

No hassle of managing NSGs: Azure Bastion is a fully managed platform PaaS service from Azure that is hardened internally to provide you secure RDP/SSH connectivity. You don't need to apply any NSGs on Azure Bastion subnet.

[https://docs.microsoft.com/en-us/cli/azure/network/bastion?view=azure-cli-latest#az-network-bastion-create](https://docs.microsoft.com/en-us/cli/azure/network/bastion?view=azure-cli-latest#az-network-bastion-create)
The SKU of the public IP must be Standard.

Subnet with name 'AzureBastionSubnet' **can be used only for the Azure Bastion resource**.
If you use Azure Bastion,  **VNet must have a subnet called AzureBastionSubnet.**

```sh
# The gateway provides connectivity between the routers in the on-premises network and the virtual network. The gateway is placed in its own subnet. Azure only allows VPN and ExpressRoute gateways to be deployed into these subnets.
# https://docs.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-vpn-faq#do-i-need-a-gatewaysubnet
az network vnet create --name $vnet_bastion_name -g $rg_bastion_name --address-prefixes 192.168.0.0/21 --location $location
az network vnet subnet create --name $subnet_bastion_name --address-prefixes 192.168.1.0/24 --vnet-name $vnet_bastion_name -g $rg_bastion_name
az network vnet subnet create --name ManagementSubnet --address-prefixes 192.168.2.0/24 --vnet-name $vnet_bastion_name -g $rg_bastion_name
az network vnet subnet create --name GatewaySubnet --address-prefixes 192.168.3.0/24 --vnet-name $vnet_bastion_name -g $rg_bastion_name

bastion_vnet_id=$(az network vnet show --name $vnet_bastion_name -g $rg_bastion_name --query id -o tsv)
echo "Bastion VNet Id :" $bastion_vnet_id	

bastion_subnet_id=$(az network vnet subnet show --name $subnet_bastion_name --vnet-name $vnet_bastion_name -g $rg_bastion_name --query id -o tsv)
echo "Bastion Subnet Id :" $bastion_subnet_id	

# https://github.com/Azure/azure-quickstart-templates/tree/master/101-azure-bastion

```


### Setup NSG 
```sh

# https://github.com/Azure/azure-quickstart-templates/tree/master/101-azure-bastion-nsg
# https://docs.microsoft.com/en-us/azure/bastion/bastion-nsg
# NSG sample : https://user-images.githubusercontent.com/47132998/69514141-4f55d380-0f70-11ea-980e-2094bd57de20.png
# https://github.com/Azure/azure-quickstart-templates/blob/master/101-azure-bastion-nsg/azuredeploy.json

b_nsg="bastion-nsg-management"
az network nsg create --name $b_nsg -g $rg_bastion_name --location $location

az network nsg rule create --access Allow --destination-port-range 22 --source-address-prefixes Internet --name "Allow SSH from Internet" --nsg-name $b_nsg -g $rg_bastion_name --priority 100

az network nsg rule create --access Allow --destination-port-range 3389 --source-address-prefixes Internet --name "Allow RDP from Internet" --nsg-name $b_nsg -g $rg_bastion_name --priority 110

az network vnet subnet update --name ManagementSubnet --network-security-group $b_nsg --vnet-name $vnet_bastion_name -g $rg_bastion_name

```

### Optionally Create a JumpBox VM

See
- [https://docs.microsoft.com/en-us/azure/virtual-machines/windows/sizes](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/sizes)
- [https://docs.microsoft.com/en-us/azure/virtual-machines/linux/sizes](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/sizes)

[Your SSH keys should have been generated at pre-req step](./setup-prereq#generates-your-ssh-keys)


<span style="color:red">/!\ IMPORTANT </span> : If you create a Linux JumpOff you will not have access to the ARO console, you may pefer to use a Windows JumpOff instead.

```sh
#az network nsg create --name nsg-management -g $rg_bastion_name --location $location 
#az network nsg rule create --access Allow -destination-port-range 3389 -source-address-prefixes Internet --name "Allow RDP from Internet" --nsg-name nsg-management -g $rg_bastion_name --priority 100
#az network nsg rule create --access Allow -destination-port-range 22-source-address-prefixes Internet --name "Allow SSH from Internet" --nsg-name nsg-management -g $rg_bastion_name --priority 110
#az network vnet subnet update --network-security-group nsg-management --name ManagementSubnet --vnet-name $vnet_bastion_name -g $rg_bastion_name

# az vm list-sizes --location $location --output table
# az vm image list-publishers --location $location --output table
az vm image list-offers --publisher MicrosoftWindowsServer --location $location --output table
az vm image list --publisher MicrosoftWindowsServer --offer WindowsServer --location $location --output table
az vm image list-offers --publisher MicrosoftWindowsDesktop --location $location --output table
az vm image list --publisher MicrosoftWindowsDesktop --offer Windows-10 --location $location --output table

# az vm image list-publishers --location $location --output table | grep -i Canonical
# az vm image list-offers --publisher Canonical --location $location --output table
# az vm image list --publisher Canonical --offer UbuntuServer --location $location --output table

# --size Standard_D3_v2 --image Win2019Datacenter
az vm create --name $jumpoff_name \
  --image MicrosoftWindowsDesktop:Windows-10:20h1-entn-g2:19041.264.2005110456 \
  --admin-username $bastion_admin_username \
  --admin-password $win_vm_admin_pwd \
  --resource-group $rg_bastion_name \
  --vnet-name $vnet_bastion_name \
  --subnet ManagementSubnet \
  --nsg $b_nsg \
  --size Standard_B2s \
  --location $location \
  --output table

az vm create --name $bastion_name \
    --image UbuntuLTS \
    --admin-username $bastion_admin_username \
    --resource-group $rg_bastion_name \
    --vnet-name $vnet_bastion_name \
    --subnet ManagementSubnet \
    --nsg $b_nsg \
    --size Standard_B1s \
    --zone 1 \
    --location $location \
    --ssh-key-values ~/.ssh/$ssh_key.pub
    # --generate-ssh-keys

network_interface_id=$(az vm show --name $bastion_name -g $rg_bastion_name --query 'networkProfile.networkInterfaces[0].id' -o tsv)
echo "Bastion VM Network Interface ID :" $network_interface_id

network_interface_private_ip=$(az resource show --ids $network_interface_id \
  --api-version 2019-04-01 --query 'properties.ipConfigurations[0].properties.privateIPAddress' -o tsv)
echo "Network Interface private IP :" $network_interface_private_ip

network_interface_pub_ip_id=$(az resource show --ids $network_interface_id \
  --api-version 2019-04-01 --query 'properties.ipConfigurations[0].properties.publicIPAddress.id' -o tsv)

network_interface_pub_ip=$(az network public-ip show -g $rg_name --id $network_interface_pub_ip_id --query "ipAddress" -o tsv)
echo "Network Interface public  IP :" $network_interface_pub_ip

# test
ssh -i ~/.ssh/$ssh_key $bastion_admin_username@$network_interface_pub_ip

```

<span style="color:red">/!\ IMPORTANT </span> : To successfully peer two virtual networks this command must be called twice with the values for --vnet-name and --remote-vnet reversed.

see [https://docs.microsoft.com/en-us/azure/virtual-network/tutorial-connect-virtual-networks-cli#peer-virtual-networks](https://docs.microsoft.com/en-us/azure/virtual-network/tutorial-connect-virtual-networks-cli#peer-virtual-networks)

## Setup VNet peering

Bastion ==> ARO
[https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-manage-peering#requirements-and-constraints](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-manage-peering#requirements-and-constraints)
```sh
# https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-peering-overview
# https://docs.microsoft.com/en-us/cli/azure/network/vnet/peering?view=azure-cli-latest#az-network-vnet-peering-create
az network vnet peering create -n $vnet_peering_name_bastion_aro \
    -g $rg_bastion_name \
    --subscription $subId \
    --allow-vnet-access \
    --vnet-name $vnet_bastion_name \
    --remote-vnet $vnet_id

az network vnet peering show -g $rg_bastion_name -n $vnet_peering_name_bastion_aro --vnet-name $vnet_bastion_name --query peeringState

az network vnet peering create -n $vnet_peering_name_bastion_aro \
    -g $rg_name \
    --subscription $subId \
    --allow-vnet-access \
    --vnet-name $vnet_name \
    --remote-vnet $bastion_vnet_id

az network vnet peering list -g $rg_bastion_name --vnet-name $vnet_bastion_name  --subscription $subId
az network vnet peering show -g $rg_bastion_name -n $vnet_peering_name_bastion_aro --vnet-name $vnet_bastion_name --query peeringState
az network vnet peering show -g $rg_name -n $vnet_peering_name_bastion_aro --vnet-name $vnet_name --query peeringState

```
