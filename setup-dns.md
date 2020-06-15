
See : 
- [https://docs.microsoft.com/en-us/azure/private-link/private-endpoint-dns#on-premises-workloads-using-a-dns-forwarder](https://docs.microsoft.com/en-us/azure/private-link/private-endpoint-dns#on-premises-workloads-using-a-dns-forwarder)
- [https://www.digitalocean.com/community/tutorials/how-to-configure-bind-as-a-private-network-dns-server-on-ubuntu-18-04](https://www.digitalocean.com/community/tutorials/how-to-configure-bind-as-a-private-network-dns-server-on-ubuntu-18-04)
- [https://docs.microsoft.com/en-us/azure/virtual-network/what-is-ip-address-168-63-129-16](https://docs.microsoft.com/en-us/azure/virtual-network/what-is-ip-address-168-63-129-16)
- [https://docs.microsoft.com/en-us/azure/aks/private-clusters#hub-and-spoke-with-custom-dns](https://docs.microsoft.com/en-us/azure/aks/private-clusters#hub-and-spoke-with-custom-dns)
- [https://docs.microsoft.com/en-us/azure/storage/files/storage-files-networking-dns#manually-configuring-dns-forwarding](https://docs.microsoft.com/en-us/azure/storage/files/storage-files-networking-dns#manually-configuring-dns-forwarding)
- []()

Every storage account has a fully qualified domain name (FQDN). For the public cloud regions, this FQDN follows the pattern storageaccount.blob.core.windows.net where storageaccount is the name of the storage account. When you make requests against this name, your OS performs a DNS lookup to resolve the fully qualified domain name to an IP address which it can use to send the requests to.

By default, storageaccount.blob.core.windows.net resolves to the public endpoint's IP address. The public endpoint for a storage account is hosted on an Azure storage cluster which hosts many other storage accounts' public endpoints. When you create a Private-Endpoint, a Private-DNS zone is linked to the VNet it was added to, with a CNAME record mapping storageaccount.blob.core.windows.net to an A record entry for the private IP address of your storage account's private endpoint. This enables you to use storageaccount.blob.core.windows.net FQDN within the VNet and have it resolve to the private endpoint's IP address.

Since our ultimate objective is to access the Azure Storage hosted within the storage account from on-premises using a network tunnel such as a VPN or ExpressRoute connection, you must configure your on-premises DNS servers to forward requests made to the Azure Storage service to the Azure private DNS service. To accomplish this, you need to set up conditional forwarding of *.core.windows.net (or the appropriate storage endpoint suffix for the US Government, Germany, or China national clouds) to a DNS server hosted within your Azure VNet. This DNS server will then recursively forward the request on to Azure's private DNS service that will resolve the fully qualified domain name of the storage account to the appropriate private IP address.

Configuring DNS forwarding for Azure Storage will require running a virtual machine to host a DNS server to forward the requests, however this is a one time step for all the Azure Storage hosted within your VNet. Additionally, this is not an exclusive requirement to use Azure Stoarge - any Azure service that supports Private-Endpoints that you want to access from on-premises can make use of the DNS forwarding you will configure in this guide: Azure File  storage, Azure Blob storage, SQL Azure, Cosmos DB, etc.

This guide shows the steps for configuring DNS forwarding for the Azure storage endpoint, so in addition to Azure Files, DNS name resolution requests for all of the other Azure storage services (Azure Blob storage, Azure Table storage, Azure Queue storage, etc.) will be forwarded to Azure's private DNS service. Additional endpoints for other Azure services can also be added if desired. DNS forwarding back to your on-premises DNS servers will also be configured, enabling cloud resources within your VNet (such as a DFS-N server) to resolve on-premises machine names.

 
 The conditional forwarding must be made to the recommended public DNS zone forwarder. For example: database.windows.net instead of privatelink.database.windows.net.


In [this sample](https://docs.microsoft.com/en-us/azure/storage/files/storage-files-networking-dns#manually-configuring-dns-forwarding) , The [PowerShell snippet uses CommandLet from dnsserver Module](https://docs.microsoft.com/en-us/powershell/module/dnsserver/?view=win10-ps) which is not avialble neither on WSL neither on Windows 10, so then you have 2 options :
- Setup a VM with --image Win2019Datacenter to create a DNS server in the ARO VNet (where there is the Storage PE) as describe in the docs
- setup CoreDNS on an Ubuntu VM bind to the VNet IP of the VM
- [https://coredns.io/plugins/forward](https://coredns.io/plugins/forward)
. {
    bind 172.16.2.5
    forward . 168.63.129.16
    log
    errors
    cache
}




### Setup NSG 

The ARO RG is locked, it is not possible to modify resources in it ...
```sh

worker_nsg_id=$(az network vnet subnet show --name $worker_subnet_name --vnet-name $vnet_name -g $rg_name --query networkSecurityGroup.id -o tsv)
echo "Worker NSG Id :" $worker_nsg_id	

# az network nsg list -g $rg_name
az network nsg show --id $worker_nsg_id -g $rg_name

worker_nsg_name=$(az network nsg show --id $worker_nsg_id  --query 'name' -o tsv)
echo "Worker NSG Name :" $worker_nsg_name	

# az network nsg update --id $worker_nsg_id -g $rg_name

# https://github.com/Azure/azure-quickstart-templates/tree/master/101-azure-bastion-nsg
# https://docs.microsoft.com/en-us/azure/bastion/bastion-nsg
# NSG sample : https://user-images.githubusercontent.com/47132998/69514141-4f55d380-0f70-11ea-980e-2094bd57de20.png
# https://github.com/Azure/azure-quickstart-templates/blob/master/101-azure-bastion-nsg/azuredeploy.json

dnsf_nsg="dnsf-nsg-management"
az network nsg create --name $dnsf_nsg -g $rg_name --location $location

az network nsg rule create --access Allow --destination-port-range 22 --source-address-prefixes Internet --name "Allow SSH from Internet" --nsg-name $dnsf_nsg -g $rg_name --priority 100

az network nsg rule create --access Allow --destination-port-range 3389 --source-address-prefixes Internet --name "Allow RDP from Internet" --nsg-name $dnsf_nsg -g $rg_name --priority 110

az network nsg rule create --access Allow --destination-port-range 1053 --source-address-prefixes Internet --name "Allow CoreDNS" --nsg-name $dnsf_nsg -g $rg_name --priority 120

az network vnet subnet update --name $worker_subnet_name --network-security-group $dnsf_nsg --vnet-name $vnet_name -g $rg_name

```

### Create a JumpBox VM

See
- [https://docs.microsoft.com/en-us/azure/virtual-machines/windows/sizes](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/sizes)
- [https://docs.microsoft.com/en-us/azure/virtual-machines/linux/sizes](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/sizes)

[Your SSH keys should have been generated at pre-req step](./setup-prereq#generates-your-ssh-keys)


<span style="color:red">/!\ IMPORTANT </span> : If you create a Linux JumpOff you will not have access to the ARO console, you may pefer to use a Windows JumpOff instead.

```sh
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
az vm create --name $dnsf_name \
  --image Win2019Datacenter \
  --admin-username $dnsf_admin_username \
  --admin-password $dnsf_vm_admin_pwd \
  --resource-group $rg_name \
  --vnet-name $vnet_name \
  --subnet $worker_subnet_name \
  --nsg $dnsf_nsg \
  --size Standard_B2s \
  --location $location \
  --output table

az vm create --name $dnsf_name \
    --image UbuntuLTS \
    --admin-username $dnsf_admin_username \
    --resource-group $rg_name \
    --vnet-name $vnet_name \
    --subnet $worker_subnet_name \
    --nsg $dnsf_nsg \
    --size Standard_B1s \
    --zone 1 \
    --location $location \
    --ssh-key-values ~/.ssh/$ssh_key.pub
    # --generate-ssh-keys

network_interface_id=$(az vm show --name $dnsf_name -g $rg_name --query 'networkProfile.networkInterfaces[0].id' -o tsv)
echo "Bastion VM Network Interface ID :" $network_interface_id

network_interface_private_ip=$(az resource show --ids $network_interface_id \
  --api-version 2019-04-01 --query 'properties.ipConfigurations[0].properties.privateIPAddress' -o tsv)
echo "Network Interface private IP :" $network_interface_private_ip

network_interface_pub_ip_id=$(az resource show --ids $network_interface_id \
  --api-version 2019-04-01 --query 'properties.ipConfigurations[0].properties.publicIPAddress.id' -o tsv)

network_interface_pub_ip=$(az network public-ip show -g $rg_name --id $network_interface_pub_ip_id --query "ipAddress" -o tsv)
echo "Network Interface public  IP :" $network_interface_pub_ip

# test from JumpOff
ssh -i ~/.ssh/$ssh_key $dnsf_admin_username@$network_interface_pub_ip


https://coredns.io/manual/toc
https://github.com/coredns/coredns/blob/master/corefile.5.md
https://coredns.io/2017/07/23/corefile-explained/
https://wiki.archlinux.org/index.php/CoreDNS


 /etc/coredns/Corefile ,


wget https://github.com/coredns/coredns/releases/download/v1.6.9/coredns_1.6.9_linux_amd64.tgz
tar zxvf coredns_1.6.9_linux_amd64.tgz
ls -al
chmod +x coredns
./coredns -version

vim Corefile

.:53 {
    bind 172.16.2.5
    forward . 168.63.129.16
    log
    errors
    cache
}

sudo ./coredns -dns.port=53

# Thus most users use the Corefile to configure CoreDNS. When CoreDNS starts, and the -conf flag is not given, it will look for a file named Corefile in the current directory.
# https://coredns.io/manual/toc/#installation : Server blocks can optionally specify a port number to listen on. This defaults to port 53 (the standard port for DNS). Specifying a port is done by listing the port after the zone separated by a colon. This Corefile instructs CoreDNS to create a Server that listens on port 1053:

./coredns -dns.port=1053




Then to modify DNS on your Windows 10 client :

- [https://www.lifewire.com/how-to-change-dns-servers-in-windows-2626242](https://www.lifewire.com/how-to-change-dns-servers-in-windows-2626242)

From CMD as admin
netsh and press Enter
netsh> prompt, type interface ip show config, then press Enter
interface ip set dns "Ethernet0" static 8.8.8.8 and press Enter. Replace Ethernet0 with the name of your connection and 8.8.8.8 with the DNS server you want to use.

- [https://pureinfotech.com/change-dns-windows-10/](https://pureinfotech.com/change-dns-windows-10/)
Get-NetIPConfiguration
Set-DnsClientServerAddress -InterfaceIndex 8 -ServerAddresses 192.168.1.254, 8.8.8.8

```sh
sudo apt-get update


sudo nano /etc/default/bind9
# OPTIONS="-u bind -4"

# sudo systemctl restart bind9
sudo service bind9 restart
sudo vim /etc/bind/named.conf.options
options {
        directory "/var/cache/bind";
        forwarders {
                8.8.8.8;
                168.63.129.16;
        };
        dnssec-validation auto;
        auth-nxdomain no;    # conform to RFC1035
        listen-on-v6 { any; };
};

# Get the IP
# ip addr show eth3 | grep -i inet
# ifconfig -a
# hostname -I
# host myip.opendns.com resolver1.opendns.com | grep "myip.opendns.com has address"
# myip=$(dig +short myip.opendns.com @resolver1.opendns.com)


sudo vim /etc/bind/named.conf.local

zone "privatelink.blob.core.windows.net" {
    type master;
    file "/etc/bind/zones/privatelink.blob.core.windows.net"; # zone file path
    allow-transfer {192.168.1.39; }; # ns2 private IP address - secondary
};

sudo service bind9 restart

sudo dnsmasq --clear-on-reload
nslookup ${aro_registry_blob_str_name}.blob.core.windows.net

# Define your own IP range
#AUTHORIZED_IP_RANGE="176.134.171.0-176.134.171.255" # 176.134.171.0/24 | 172.16.2.0/24 | 192.168.1.0/24 | 192.168.0.0-192.168.7.255
AUTHORIZED_IP_RANGE=176.134.171.0/24 # 176.134.171.92
az storage account network-rule add --action allow --ip-address $AUTHORIZED_IP_RANGE --account-name $aro_registry_blob_str_name -g $rg_name
az storage account network-rule list --account-name $aro_registry_blob_str_name 


```
