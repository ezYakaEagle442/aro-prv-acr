
See :

- [https://azure.microsoft.com/en-us/blog/vnet-peering-and-vpn-gateways](https://azure.microsoft.com/en-us/blog/vnet-peering-and-vpn-gateways)
- [https://docs.microsoft.com/fr-fr/cli/azure/network/local-gateway?view=azure-cli-latest#az-network-local-gateway-create]()
- [https://docs.microsoft.com/fr-fr/azure/storage/files/storage-files-configure-p2s-vpn-linux](https://docs.microsoft.com/fr-fr/azure/storage/files/storage-files-configure-p2s-vpn-linux)
- [https://docs.microsoft.com/en-us/azure/vpn-gateway/point-to-site-about](https://docs.microsoft.com/en-us/azure/vpn-gateway/point-to-site-about)
- [https://docs.microsoft.com/en-us/azure/vpn-gateway/point-to-site-about#authenticate-using-native-azure-certificate-authentication](https://docs.microsoft.com/en-us/azure/vpn-gateway/point-to-site-about#authenticate-using-native-azure-certificate-authentication)

- [https://www.digitalocean.com/community/tutorials/how-to-set-up-an-ikev2-vpn-server-with-strongswan-on-ubuntu-18-04-2](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-ikev2-vpn-server-with-strongswan-on-ubuntu-18-04-2)
- [https://kifarunix.com/how-to-configure-ipsec-vpn-using-strongswan-on-ubuntu-18-04](https://kifarunix.com/how-to-configure-ipsec-vpn-using-strongswan-on-ubuntu-18-04)
- [https://wiki.strongswan.org/projects/strongswan/wiki/UsableExamples](https://wiki.strongswan.org/projects/strongswan/wiki/UsableExamples)

When using the native Azure certificate authentication, a client certificate that is present on the device is used to authenticate the connecting user. Client certificates are generated from a trusted root certificate and then installed on each client computer. You can use a root certificate that was generated using an Enterprise solution, or you can generate a self-signed certificate.

The validation of the client certificate is performed by the VPN gateway and happens during establishment of the P2S VPN connection. The root certificate is required for the validation and must be uploaded to Azure.


WSL not able tun Run VPN, see [https://github.com/microsoft/WSL/issues/416](https://github.com/microsoft/WSL/issues/416)
```sh

# "CN=mycompany.com,OU=IT,O=mycompany.com,L=Paris,ST=IDF,C=FR,emailAddress=DevOps-KissMyApp@groland.grd"
# "CN=John DOE,OU=IT,O=mycompany.com,STREET=1 London Street,L=777 London,ST=BK,C=UK,T=M,SURNAME=DOE,GIVENNAME=John,INITIALS=J.D,IP=MyIP,emailAddress=john.doe@mycompany.com"

# on WSL: 
sudo apt install strongswan strongswan-pki libstrongswan-extra-plugins curl libxml2-utils cifs-utils libcharon-extra-plugins

installDir="/etc/"
username="changeit"
password="changeit"
rootCertName="P2SRootCert"

mkdir temp
cd temp

sudo ipsec pki --gen --outform pem > rootKey.pem
sudo ipsec pki --self --in rootKey.pem --dn "CN=$rootCertName" --ca --outform pem > rootCert.pem

rootCertificate=$(openssl x509 -in rootCert.pem -outform der | base64 -w0 ; echo)
openssl x509 -in rootCert.pem -text -noout

sudo ipsec pki --gen --size 4096 --outform pem > "clientKey.pem"
sudo ipsec pki --pub --in "clientKey.pem" | \
    sudo ipsec pki \
        --issue \
        --cacert rootCert.pem \
        --cakey rootKey.pem \
        --dn "CN=$username" \
        --san $username \
        --flag clientAuth \
        --outform pem > "clientCert.pem"

openssl x509 -in clientCert.pem -text -noout
openssl pkcs12 -in "clientCert.pem" -inkey "clientKey.pem" -certfile rootCert.pem -export -out "client.p12" -password "pass:$password"

vpnName="AlphaVpnGateway"
vpnPublicIpAddressName="$vpnName-PublicIP"

publicIpAddress=$(az network public-ip create \
    --resource-group $rg_name \
    --name $vpnPublicIpAddressName \
    --location $location \
    --sku "Basic" \
    --allocation-method "Dynamic" \
    --query "publicIp.id" | tr -d '"')

echo "publicIpAddress " $publicIpAddress

# GatewaySubnet in ARO VNet should be already created
# gway_subnet_id=$(az network vnet subnet create --name GatewaySubnet --address-prefixes 172.16.3.0/24 --vnet-name $vnet_name -g $rg_name  --query id -o tsv)
# echo "Gateway Subnet Id :" $gway_subnet_id	

# --address-prefixes "172.16.0.0/24" is your on-prem IP or just set your own Home IP like 172.42.42.42
# The Gateway creation can take 30 minutes
az network vnet-gateway create \
    --resource-group $rg_name \
    --name $vpnName \
    --vnet $vnet_name \
    --public-ip-addresses $publicIpAddress \
    --location $location \
    --sku "VpnGw1" \
    --gateway-typ "Vpn" \
    --vpn-type "RouteBased" \
    --client-protocol "IkeV2" > /dev/null \
    --address-prefixes "172.16.0.0/24" # The smallest address pool allowed on this gateway is size /29.
    

az network vnet-gateway root-cert create \
    --resource-group $rg_name \
    --gateway-name $vpnName \
    --name $rootCertName \
    --public-cert-data $rootCertificate \
    --output none

publicIp=$(az network public-ip show --name $vpnPublicIpAddressName -g $rg_name --query ipAddress)
echo "VPN Gateway publicIp " $publicIp 

# https://docs.microsoft.com/fr-fr/cli/azure/network/vnet-gateway/vpn-client?view=azure-cli-latest#az-network-vnet-gateway-vpn-client-generate
vpnClient=$(az network vnet-gateway vpn-client generate \
    --resource-group $rg_name \
    --name $vpnName \
    --authentication-method EAPTLS | tr -d '"')

apt-get install unzip
curl $vpnClient --output vpnClient.zip
unzip vpnClient.zip

vpnServer=$(xmllint --xpath "string(/VpnProfile/VpnServer)" Generic/VpnSettings.xml)
vpnType=$(xmllint --xpath "string(/VpnProfile/VpnType)" Generic/VpnSettings.xml | tr '[:upper:]' '[:lower:]')
routes=$(xmllint --xpath "string(/VpnProfile/Routes)" Generic/VpnSettings.xml)

echo "vpnServer " $vpnServer
echo "vpnType " $vpnType
echo "routes " $routes

sudo cp "${installDir}ipsec.conf" "${installDir}ipsec.conf.backup"
sudo cp "Generic/VpnServerRoot.cer" "${installDir}ipsec.d/cacerts"
sudo cp "client.p12" "${installDir}ipsec.d/private" 

echo -e "\nconn $vnet_name" | sudo tee -a "${installDir}ipsec.conf" > /dev/null
echo -e "\tkeyexchange=$vpnType" | sudo tee -a "${installDir}ipsec.conf" > /dev/null
echo -e "\ttype=tunnel" | sudo tee -a "${installDir}ipsec.conf" > /dev/null
echo -e "\tleftfirewall=yes" | sudo tee -a "${installDir}ipsec.conf" > /dev/null
echo -e "\tleft=%any" | sudo tee -a "${installDir}ipsec.conf" > /dev/null
echo -e "\tleftauth=eap-tls" | sudo tee -a "${installDir}ipsec.conf" > /dev/null
echo -e "\tleftid=%client" | sudo tee -a "${installDir}ipsec.conf" > /dev/null
echo -e "\tright=$vpnServer" | sudo tee -a "${installDir}ipsec.conf" > /dev/null
echo -e "\trightid=%$vpnServer" | sudo tee -a "${installDir}ipsec.conf" > /dev/null
echo -e "\trightsubnet=$routes" | sudo tee -a "${installDir}ipsec.conf" > /dev/null
echo -e "\tleftsourceip=%config" | sudo tee -a "${installDir}ipsec.conf" > /dev/null 
echo -e "\tauto=add" | sudo tee -a "${installDir}ipsec.conf" > /dev/null

echo ": P12 client.p12 '$password'" | sudo tee -a "${installDir}ipsec.secrets" > /dev/null

cat ${installDir}ipsec.conf
ls -al ${installDir}ipsec.d/cacerts
sudo cat ${installDir}ipsec.secrets
ipsec listcerts
sudo ls -al ${installDir}ipsec.d/private


# systemctl restart strongswan

sudo ipsec restart
sudo ipsec statusall
sudo ipsec up $vnet_name 

ls -al /sbin/iptables


sudo ufw status
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw allow 500/udp # Allows Internet Security Association and Key Management Protocol (ISAKMP) traffic to be forwarded
sudo ufw allow 4500/udp # Allows handling of IPsec between natted devices

sudo cp /etc/ufw/before.rules /etc/ufw/before.rules.BACKUP
sudo vim /etc/ufw/before.rules
```

Replace the IP pool and the default route interface accordingly
```sh
# Don't delete these required lines, otherwise there will be errors
-A POSTROUTING -s 172.16.0.0/24 -o enp0s3 -m policy --pol ipsec --dir out -j ACCEPT
-A POSTROUTING -s 172.16.0.0/24 -o enp0s3 -j MASQUERADE
COMMIT
-A FORWARD --match policy --pol ipsec --dir in -s 172.16.0.0/24 -o enp0s3 -p tcp -m tcp --tcp-flags SYN,RST SYN -m tcpmss --mss 1361:1536 -j TCPMSS --set-mss 1360
COMMIT
:ufw-before-input - [0:0]
:ufw-before-output - [0:0]
:ufw-before-forward - [0:0]
:ufw-not-local - [0:0]
# End required lines
-A ufw-before-forward --match policy --pol ipsec --dir in --proto esp -s 172.16.0.0/24 -j ACCEPT
-A ufw-before-forward --match policy --pol ipsec --dir out --proto esp -d 172.16.0.0/24 -j ACCEPT

sudo ufw disable
sudo ufw enable

sudo vim /etc/ufw/sysctl.conf
```

Uncomment the next line to enable packet forwarding for IPv4
```sh
net/ipv4/ip_forward=1
# Do not accept ICMP redirects (prevent MITM attacks)
net/ipv4/conf/all/accept_redirects = 0

# Do not send ICMP redirects (we are not a router)
net/ipv4/conf/all/send_redirects = 0

# Disble Path MTU discovery to prevent packet fragmentation by adding the line below
net/ipv4/ip_no_pmtu_disc=1

```

```sh

sudo ipsec restart
sudo ipsec statusall
sudo ipsec up $vnet_name 


ip route show default
ip route | grep default

# az network local-gateway create -g MyResourceGroup -n MyLocalGateway \
#     --gateway-ip-address 23.99.221.164 --local-address-prefixes 10.0.0.0/24 20.0.0.0/24

```