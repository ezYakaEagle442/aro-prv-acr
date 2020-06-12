# ARO Private cluster &amp; ACR with Private-Endpoint

## Introduction
This is an introduction to play with ARO Private cluster and :
- ARO Built-in integrated Registry using an [Azure BLOB Storage with a Private-Endpoint](https://docs.microsoft.com/en-us/azure/storage/common/storage-private-endpoints)
- ACR with [Private-Endpoint](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-private-link#create-a-private-registry-endpoint)


## **Disclaimer**

**The features described in this workshop might be not yet production-ready, we enable preview-features for the purpose of learning.**

See also :

- [Azure ARO docs](https://docs.microsoft.com/en-us/azure/openshift/tutorial-create-cluster)
- [ARO 4.x docs](https://docs.openshift.com/aro/4/registry/architecture-component-imageregistry.html)
- [http://aroworkshop.io](http://aroworkshop.io)
- [https://aka.ms/aroworkshop-devops](https://aka.ms/aroworkshop-devops)
- [https://github.com/akamenev/aro-private](https://github.com/akamenev/aro-private)
- [https://github.com/stuartatmicrosoft/azure-aro/blob/master/aro4-replace-pull-secret.sh](https://github.com/stuartatmicrosoft/azure-aro/blob/master/aro4-replace-pull-secret.sh)
- [https://blog.cloudtrooper.net/2020/06/02/a-day-in-the-life-of-a-packet-in-azure-redhat-openshift-part-5](https://blog.cloudtrooper.net/2020/06/02/a-day-in-the-life-of-a-packet-in-azure-redhat-openshift-part-5)

## ToC

1. Setup [Tools](tools.md)
1. Check [subscription](subscription.md)
1. Setup [environment variables](set-var.md)
1. Setup [pre-requisites](setup-prereq.md)
   1. Create RG
   1. Create Storage
   1. Get a Red Hat pull secret
   1. Setup [Network](setup-network.md)
   <!-- Create [SSH Keys](setup-prereq.md#generates-your-ssh-keys) -->
1. Setup [ARO cluster](setup-aro.md)
1. Setup [HELM](setup-helm.md)
1. Setup [ACR](setup-acr.md)