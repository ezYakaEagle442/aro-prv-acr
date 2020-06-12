
See :

- [https://docs.microsoft.com/fr-fr/cli/azure/network/local-gateway?view=azure-cli-latest#az-network-local-gateway-create]()
- [https://docs.microsoft.com/fr-fr/azure/storage/files/storage-files-configure-p2s-vpn-linux](https://docs.microsoft.com/fr-fr/azure/storage/files/storage-files-configure-p2s-vpn-linux)
- [https://docs.microsoft.com/en-us/azure/vpn-gateway/point-to-site-about](https://docs.microsoft.com/en-us/azure/vpn-gateway/point-to-site-about)
```sh

# "CN=mycompany.com,OU=IT,O=mycompany.com,L=Paris,ST=IDF,C=FR,emailAddress=DevOps-KissMyApp@groland.grd"
# "CN=John DOE,OU=IT,O=mycompany.com,STREET=1 London Street,L=777 London,ST=BK,C=UK,T=M,SURNAME=DOE,GIVENNAME=John,INITIALS=J.D,IP=MyIP,emailAddress=john.doe@mycompany.com"

# on WSL: sudo apt-get install strongswan
username="changeit"
password="changeit"
rootCertName="P2SRootCert"

mkdir temp
cd temp

sudo ipsec pki --gen --outform pem > rootKey.pem
sudo ipsec pki --self --in rootKey.pem --dn "CN=$rootCertName" --ca --outform pem > rootCert.pem

rootCertificate=$(openssl x509 -in rootCert.pem -outform der | base64 -w0 ; echo)

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

# --address-prefixes "172.16.0.0/24" is your on-prem IP or just set your own Home IP like 172.42.42.42
az network vnet-gateway create \
    --resource-group $rg_name \
    --name $vpnName \
    --vnet $vnet_name \
    --public-ip-addresses $publicIpAddress \
    --location $location \
    --sku "VpnGw1" \
    --gateway-typ "Vpn" \
    --vpn-type "RouteBased" \
    --address-prefixes "172.16.0.0/24" \
    --client-protocol "IkeV2" > /dev/null

az network vnet-gateway root-cert create \
    --resource-group $rg_name \
    --gateway-name $vpnName \
    --name $rootCertName \
    --public-cert-data $rootCertificate \
    --output none


# https://docs.microsoft.com/fr-fr/cli/azure/network/vnet-gateway/vpn-client?view=azure-cli-latest#az-network-vnet-gateway-vpn-client-generate
vpnClient=$(az network vnet-gateway vpn-client generate \
    --resource-group $rg_name \
    --name $vpnName \
    --authentication-method EAPTLS | tr -d '"')

curl $vpnClient --output vpnClient.zip
unzip vpnClient.zip

vpnServer=$(xmllint --xpath "string(/VpnProfile/VpnServer)" Generic/VpnSettings.xml)
vpnType=$(xmllint --xpath "string(/VpnProfile/VpnType)" Generic/VpnSettings.xml | tr '[:upper:]' '[:lower:]')
routes=$(xmllint --xpath "string(/VpnProfile/Routes)" Generic/VpnSettings.xml)

# az network local-gateway create -g MyResourceGroup -n MyLocalGateway \
#     --gateway-ip-address 23.99.221.164 --local-address-prefixes 10.0.0.0/24 20.0.0.0/24

```