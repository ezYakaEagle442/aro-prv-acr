# Create RG
```sh
az group create --name $rg_name --location $location
# az group create --name rg-cloudshell-$location --location $location

```

# Create Storage

This is not mandatory, you can create a storage account to play with CloudShell

```sh
# https://docs.microsoft.com/en-us/cli/azure/storage/account?view=azure-cli-latest#az-storage-account-create
# https://docs.microsoft.com/en-us/azure/storage/common/storage-introduction#types-of-storage-accounts
az storage account create --name stcloudshellwe --kind StorageV2 --sku Standard_LRS -g rg-cloudshell-$location --location $location --https-only true
# az storage account create --name $storage_name --kind StorageV2 --sku Standard_LRS --resource-group $rg_name --location $location --https-only true

```

# Get a Red Hat pull secret

See [Azure docs](https://docs.microsoft.com/en-us/azure/openshift/tutorial-create-cluster#get-a-red-hat-pull-secret-optional)
to connect to [Red Hat OpenShift cluster manager portal](https://cloud.redhat.com/openshift/install/azure/aro-provisioned)

Click Download pull secret from [https://cloud.redhat.com/openshift/install/azure/aro-provisioned/pull-secret](https://cloud.redhat.com/openshift/install/azure/aro-provisioned/pull-secret)
Keep the saved pull-secret.txt file somewhere safe - it will be used in each cluster creation.
When running the az aro create command, you can reference your pull secret using the --pull-secret @pull-secret.txt parameter. Execute az aro create from the directory where you stored your pull-secret.txt file. Otherwise, replace @pull-secret.txt with @<path-to-my-pull-secret-file>.

See also [https://github.com/stuartatmicrosoft/azure-aro#aro4-replace-pull-secretsh](https://github.com/stuartatmicrosoft/azure-aro#aro4-replace-pull-secretsh)


# Generates your SSH keys

<span style="color:red">/!\ IMPORTANT </span> :  check & save your ssh_passphrase !!!

Generate & save nodes [SSH keys](https://docs.microsoft.com/en-us/azure/aks/ssh) to Azure Key-Vault is a [Best-practice](https://github.com/Azure/k8s-best-practices/blob/master/Security_securing_a_cluster.md#securing-host-access)

If you want to save your keys to keyVault, [KV must be created first](setup-kv.md)

Read [this explanation about private keys management in KV](https://github.com/Azure/azure-sdk-for-js/issues/7647#issuecomment-594935307)
The KeyVault service stores both the public and the private parts of your certificate in a KeyVault secret, along with any other secret you might have created in that same KeyVault instance. With this separation comes considerable control of the level of access you can give to people in your organization regarding your certificates. The access control can be specified through the policy you pass in when creating a certificate. Knowing that the private key is stored in a KeyVault Secret, with the public certificate included, we can retrieve it by using the KeyVault-Secrets client.

See also :
- [About keys](https://docs.microsoft.com/en-us/azure/key-vault/certificates/about-certificates#composition-of-a-certificate)
- Composition of a Certificate](https://docs.microsoft.com/en-us/azure/key-vault/certificates/about-certificates#composition-of-a-certificate)
- [https://github.com/MicrosoftDocs/azure-docs/issues/55072](https://github.com/MicrosoftDocs/azure-docs/issues/55072)
- [https://github.com/Azure/azure-cli/issues/13547](https://github.com/Azure/azure-cli/issues/13547)
- [https://github.com/Azure/azure-cli/issues/13548](https://github.com/Azure/azure-cli/issues/13548)

```sh
ssh-keygen -t rsa -b 4096 -N $ssh_passphrase -f ~/.ssh/$ssh_key -C "youremail@groland.grd"
```