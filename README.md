# Bring Your Own OpenBSD VM to Azure
This is a step-by-step guide on how to get your own OpenBSD VM up and running on Microsoft Azure. 

Standing on the shoulders of giants: All of the instructions here are based on the fantastic work one by [https://github.com/reyk/](https://github.com/reyk/), and [https://github.com/jonathangray/ports-azure](https://github.com/jonathangray/ports-azure). Thank you both!

## Requirements

* shell access to OpenBSD 6.1.
* minimum 3GB free space of /tmp.
* doas configured; for building as a root the "permit nopass keepenv root as root" in /etc/doas.conf is enough.
* ports-azure tools (azure-cli, azure-vhd-utils). For a step-by-step procedure please follow my instructions at [https://github.com/dcasati/ports-azure](https://github.com/dcasati/ports-azure)
* The cloud-openbsd agent (instructions below)

Reference: [https://github.com/reyk/cloud-openbsd](https://github.com/reyk/cloud-openbsd)

### Verify that you have all of the required tools

In a shell running OpenBSD, run `pkg_info` to check that you have the required packages installed. DO NOT proceed if that's not the case. 

If you have to install the packages, please follow the set of instructions here first: [https://github.com/dcasati/ports-azure](https://github.com/dcasati/ports-azure) 

```bash
$ pkg_info | grep azure 
azure-cli-2.0.0p0   command line interface for Azure
azure-vhd-utils-20170104 Azure VHD utilities
py-azure-2.0.0rc6   Azure Client Libraries for Python
py-azure-batch-1.1.0 Azure Batch Client Library for Python
py-azure-common-1.1.4 Azure Client Library for Python (Common)
py-azure-graphrbac-0.30.0rc6 Azure Graph RBAC Resource Management Client Library
py-azure-keyvault-0.1.0 Azure KeyVault Client Library for Python
py-azure-mgmt-0.30.0rc6 Azure Resource Management Client Libraries
py-azure-mgmt-authorization-0.30.0rc6 Azure Authorization Resource Management Client Library
py-azure-mgmt-batch-2.0.0 Azure Batch Management Client Library for Python
py-azure-mgmt-compute-0.33.1rc1 Azure Compute Resource Management Client Library for Python
py-azure-mgmt-containerregistry-0.1.1 Azure Container Registry Client Library
py-azure-mgmt-dns-1.0.0 Azure DNS Library for Python
py-azure-mgmt-documentdb-0.1.0 Azure DocumentDB Management Client Library for Python
py-azure-mgmt-iothub-0.2.1 Azure IoTHub Management Client Library for Python
py-azure-mgmt-keyvault-0.30.0 Azure KeyVault Apps Resource Management Client Library
py-azure-mgmt-network-0.30.0 Azure Network Resource Management Client Library
py-azure-mgmt-nspkg-1.0.0 Azure Resource Management Namespace Package
py-azure-mgmt-redis-1.0.0 Azure Redis Cache Resource Management Client Library
py-azure-mgmt-resource-0.30.2 Azure Resource Management Client Library
py-azure-mgmt-sql-0.2.0 Azure SQL Management Client Library for Python
py-azure-mgmt-storage-0.31.0 Azure Storage Resource Management Client Library
py-azure-mgmt-trafficmanager-0.30.0rc6 Azure Traffic Manager Client Library
py-azure-mgmt-web-0.31.0 Azure Web Apps Resource Management Client Library
py-azure-nspkg-1.0.0 Azure Namespace Package
py-azure-servicebus-0.21.0 Azure Service Bus Client Library for Python
py-azure-servicemanagement-legacy-0.20.5 Azure Legacy Service Management Client Library for Python
py-azure-storage-0.33.0 Azure Storage Client Library
py-msrestazure-0.4.7 AutoRest: Python Client Runtime - Azure Module
$ 
```

## Install the cloud-openbsd utils

From an OpenBSD system, clone the `cloud-openbsd` utilities found at [https://github.com/reyk/cloud-openbsd](https://github.com/reyk/cloud-openbsd)

```bash
git clone https://github.com/reyk/cloud-openbsd.git
Cloning into 'cloud-openbsd'...
remote: Counting objects: 588, done.
remote: Compressing objects: 100% (4/4), done.
remote: Total 588 (delta 0), reused 2 (delta 0), pack-reused 584
Receiving objects: 100% (588/588), 146.03 KiB | 277.00 KiB/s, done.
Resolving deltas: 100% (249/249), done.
```

Verify that you have all ofthe tools installed. You should have 

With the repo locally available on your system and all of the pre-requisites in place we need to specify a few parameters before creating and pushing our new OpenBSD image to Azure. 

## Custom build the VM 

Here's a list of parameters that you can change: 


|Parameter   | Default Value  | Notes   
|---|---|---
|LOCATION | westeurope | Region where the image will be uploaded
|IMGSIZE |  30 | Image size. Value in GB   
|VM_SSHUSER | azure-user  | Default user
|VM_SKU | Standard_LRS | Storage SKU
|VM_SIZE | Standard_DS2_v2 | VM size
|MIRROR | - | This is the location where the script will fetch the OpenBSD install files.   

You can modify these and other options by changing the `create-az.sh` script. 

```bash
################################################################################
NAME=openbsd                            # Azure has some naming restrictions
LOCATION=westeurope                     # This is where we are!
IMGSIZE=30                              # Azure: >=30G for public images
#RELEASE=                               # eg. 6.1, comment out for snapshots
TIMESTAMP=$(date "+%Y%m%d%H%M%S")

AZ_RG=${NAME}${TIMESTAMP}               # Azure: resource group
AZ_SA=${AZ_RG}s                         # Azure: storage account
AZ_CN=${AZ_RG}c                         # Azure: container name
AZ_VM=${AZ_RG}vm                        # Azure: VM name
AZ_BLOB=${AZ_RG}.vhd                    # Azure: BLOB name

VM_SSHUSER=azure-user                   # These VM settings are only examples:
VM_SSHKEY=${HOME}/.ssh/id_rsa.pub       # Azure only supports RSA public keys
VM_SKU=Standard_LRS                     # Premium_LRS, Standard_LRS
VM_SIZE=Standard_DS2_v2                 # Standard_A2 etc.

ARCH=$(uname -m)
MIRROR=${MIRROR:=https://mirror.leaseweb.net/pub/OpenBSD}
AGENTURL=https://github.com/reyk/cloud-agent/releases/download/v0.1
RC_CLOUD=$PWD/data/rc.cloud
PKG_DEPS="azure-cli azure-vhd-utils qemu"
################################################################################
```

## Create the image

On an OpenBSD machine, open a shell and run the following:

`doas create-az.sh -r 6.1`

> For other sizes refer to the [General purpose virtual machine sizes](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/sizes-general).

## Create the VM

Before we create our VM on Azure, we have a few options: we can add a NIC to an existing VM or we can just simply create a VM with a single NIC (for example to be used as a jumphost/bastion), we can create a VM with two NICs. Let's see each example.

References:

* [Microsoft Azure - Create, change, or delete a network interface](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-network-interface)
* [How to create a Linux virtual machine in Azure with multiple network interface cards](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/multiple-nics?toc=%2fazure%2fvirtual-network%2ftoc.json)

## Option 1: Create a VM with a single NIC

The option of having a secure bastion/jumphost running OpenBSD is very tempting. Here's an example of how it would look like: 

```
+--------+
|  PIP   +----------------+
+--------+                |
                          v   +---------+
+----------------+     +------+         +
|MANAGEMENT NET  +-----+ hvn0 | OpenBSD |
+----------------+     +------+         |
                              +---------+
```

Run the following command:

```bash
az vm create -g openbsd20171106212621 \
    -n openbsd20171106212621vm \
    --image https://openbsd20171106212621s.blob.core.windows.net/openbsd20171106212621c/openbsd20171106212621.vhd\
    --ssh-key-value ~/.ssh/id_rsa.pub \ 
    --authentication-type ssh --admin-username azure-user \
    --public-ip-address-dns-name openbsd20171106212621vm \
    --os-type linux \
    --nsg-rule SSH \
    --storage-account openbsd20171106212621s \
    --storage-container-name openbsd20171106212621c \
    --storage-sku Standard_LRS \
    --use-unmanaged-disk \
    --size Standard_B2s
```

That's it! 

## Option 2: Create a VM with two NICs and IP Forwarding enabled

For this option we will create a VM with two NICs and will also enable IP Forwarding to you can move packets between your OpenBSD NICs. This will allow for things such as firewalling with PF, ipsec tunnels and more. This setup will be based on the following diagram:


```
                              +---------+
+----------------+     +------+         +------+       +-----------------+
|EXTERNAL SUBNET |+----+ hvn0 | OpenBSD | hvn1 +-------+ INTERNAL SUBNET |
+----------------+     +------+         +------+       +-----------------+
                              +---------+
```

For the sake of completion, we will go through the following steps: 

* Create a Resource Group
* Create a NSG
* Create a VNET
* Create a subnet

> NOTE: Bear in mind that you can use existing resources by adjusting your commands below. 

### Create a Resource Group

```bash
#Define a Resource Group. This parameter will be used along the way.
export AZURE_OPENBSD_RG=myOpenBSD_RG
az group create --name ${AZURE_OPENBSD_RG} --location westus2
```

### Create the VNet and the first subnet (External)

```bash 
az network vnet create \
    --resource-group ${AZURE_OPENBSD_RG} \
    --name ${AZURE_OPENBSD_RG}-VNet \
    --address-prefix 10.0.0.0/16 \
    --subnet-name ${AZURE_OPENBSD_RG}-external-subnet \
    --subnet-prefix 10.0.0.128/25
```

### Create the second VNet
```bash 
az network vnet subnet create \
    --resource-group ${AZURE_OPENBSD_RG} \
    --vnet-name ${AZURE_OPENBSD_RG}-VNet \
    --name ${AZURE_OPENBSD_RG}-internal-subnet \
    --address-prefix 10.0.1.0/24
```

### Create the Network Security Groups (NSG)

Create the external NSG 
```bash 
az network nsg create \
    --resource-group ${AZURE_OPENBSD_RG} \
    --name ${AZURE_OPENBSD_RG}-externalNSG
```

Create the internal NSG
```bash
az network nsg create \
    --resource-group ${AZURE_OPENBSD_RG} \
    --name ${AZURE_OPENBSD_RG}-internalNSG
```

### Add a rule to allow SSH to the external NSG

```bash
az network nsg rule create \
    --name allow-ssh \
    --nsg-name ${AZURE_OPENBSD_RG}-externalNSG \
    --resource-group ${AZURE_OPENBSD_RG} \
    --priority 100 \
    --protocol Tcp \
    --destination-port-ranges 22
```

### Create two NICs

Create the first NIC for the external subnet:

```bash
az network nic create \
    --resource-group ${AZURE_OPENBSD_RG} \
    --name externalNIC \
    --vnet-name ${AZURE_OPENBSD_RG}-VNet \
    --subnet ${AZURE_OPENBSD_RG}-external-subnet \
    --network-security-group ${AZURE_OPENBSD_RG}-externalNSG \
    --ip-forwarding
```

Create the second NIC for the internal subnet:

```bash
az network nic create \
    --resource-group ${AZURE_OPENBSD_RG} \
    --name internalNIC \
    --vnet-name ${AZURE_OPENBSD_RG}-VNet \
    --subnet ${AZURE_OPENBSD_RG}-internal-subnet \
    --network-security-group ${AZURE_OPENBSD_RG}-internalNSG \
    --ip-forwarding
```

### Create the VM with the two NICs attached

To create the VM we will need the information about the storage account, the container and the name of the VHD blob that was uploaded to Azure. You can retrieve this information from the output of the `create-az.sh` script.

```bash
export AZURE_STORAGE_ACCOUNT
export AZURE_STORAGE_BASE=${AZURE_STORAGE_ACCOUNT%?}

az vm create \
    --resource-group ${AZURE_OPENBSD_RG} \
    --name openbsdDualNIC \
    --image https://${AZURE_STORAGE_BASE}s.blob.core.windows.net/${AZURE_STORAGE_BASE}c/${AZURE_STORAGE_BASE}.vhd \
    --ssh-key-value ~/.ssh/id_rsa.pub \
    --authentication-type ssh \
    --admin-username azure-user \
    --public-ip-address-dns-name openbsdDualNIC \
    --os-type linux \
    --nsg-rule SSH  \
    --storage-account ${AZURE_STORAGE_BASE}s    \
    --storage-container-name ${AZURE_STORAGE_BASE}c \
    --storage-sku Standard_LRS  \
    --use-unmanaged-disk    \
    --size Standard_DS2_v2  \
    --os-disk-name "" \
    --nics externalNIC internalNIC
```

## Option 3: Add an extra NIC to an existing VM

### Create the VM on Azure
```bash
az vm create \
    --resource-group openbsd20171030222627 \
    --name openbsd20171030222627vm \
    --image https://openbsd20171030222627s.blob.core.windows.net/openbsd20171030222627c/openbsd20171030222627.vhd \
    --ssh-key-value ~/.ssh/id_rsa.pub \
    --authentication-type ssh   \ 
    --admin-username azure-user \
    --public-ip-address-dns-name openbsd20171030222627vm \
    --os-type linux \
    --nsg-rule SSH  \
    --storage-account openbsd20171030222627s    \
    --storage-container-name openbsd20171030222627c \
    --storage-sku Standard_LRS  \
    --use-unmanaged-disk    \
    --size Standard_DS2_v2  \
    --os-disk-name ""
```
### Add the extra NIC

This procedure will add a second NIC to an existing VM and it will enable IP Forwarding.

> NOTE: This will shutdown the VM so be cautious about using this on a running system.

Create the NIC
```bash
az network nic create \
    --resource-group openbsd20171030222627 \
    --name internalNIC \
    --vnet-name ${AZURE_OPENBSD_RG}-VNet \
    --subnet ${AZURE_OPENBSD_RG}-internal-subnet \
    --ip-forwarding
```
Deallocate the VM
```bash
az vm deallocate --resource-group ${AZURE_OPENBSD_RG} --name ${MY_OPENBSD_VM}
```
> Add the name of the OpenBSD VM to the ${MY_OPENBSD_VM} variable.

Add the extra NIC to the VM

```bash
az vm nic add  -g ${AZURE_OPENBSD_RG}  --vm-name ${MY_OPENBSD_VM} --nics internalNIC
```

Restart the VM

```bash
az vm start -g ${AZURE_OPENBSD_RG} --name ${MY_OPENBSD_VM} 
```

