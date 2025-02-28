---
layout: post
title: How to create a golden image of Ubuntu Pro with Packer for Azure
---

Based on the [tutorial](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/build-image-with-packer) from Microsoft.
Note: Packer version **1.8.1** or above is needed for creating images
from **Ubuntu** **22.04** - Jammy Jellyfish or newer.

For this tutorial I will be using Packer v**1.10.3**.

**Creating Azure Credentials**

Retrieve your Azure subscription ID:
```console
$ az account show --query "{ subscription_id: id }"
```
    

Create Azure credentials for Packer to authenticate with:
```console
$ az ad sp create-for-rbac --role Contributor --scopes /subscriptions/<subscription_id> --query "{ client_id: appId, client_secret: password, tenant_id: tenant }"
    .....
{
"client_id": "xxxxx",
"client_secret": "xxxxx",
"tenant_id": "xxxxx"
}
```

**Accept PRO terms**\
Accept the [Ubuntu Pro terms and conditions ](https://mpcprodsa.blob.core.windows.net/legalterms/3E5ED_legalterms_CANONICAL%253a240001%253a2DCOM%253a2DUBUNTU%253a2DPRO%253a2DJAMMY%253a24PRO%253a2D22%253a5F04%253a2DLTS%253a24TPNDKIR6GF6XF5BOJ5KAODEQKKA76P33RWIHBLYFOR2IZC4PDPCUNYW2MEZPJPHFNUOSCXVFMROYI53AA7MLHTRRCARG7OZMI2IXC7Y.txt) before proceeding:

```console
$ az vm image accept-terms --offer 0001-com-ubuntu-pro-jammy --plan pro-22_04-lts --publisher canonical
    {
      "accepted": true,
    }
```

**Prepare packer json**\
Save the example json for a VM based on Ubuntu PRO 22.04.\
Adjust according to your needs.\
Check the [documentation for arm builder ](https://developer.hashicorp.com/packer/integrations/hashicorp/azure/latest/components/builder/arm) for further details and available configuration parameters.\
ubuntupro.json:

    {
      "builders": [{
        "type": "azure-arm",

        "client_id": "xxxxx",
        "client_secret": "xxxxx",
        "tenant_id": "xxxxx",
        "subscription_id": "xxxxx",

        "managed_image_resource_group_name": "myResourceGroup",
        "managed_image_name": "myPackerImage",

        "os_type": "Linux",
        "image_publisher": "canonical",
        "image_offer": "0001-com-ubuntu-pro-jammy",
        "image_sku": "pro-22_04-lts",

        "azure_tags": {
         "dept": "Engineering",
         "task": "Image deployment"
        },

        "location": "East US",
        "plan_info": {
         "plan_publisher": "canonical",
         "plan_name": "pro-22_04-lts",
         "plan_product": "0001-com-ubuntu-pro-jammy"
        },
        "vm_size": "Standard_DS2_v2"
      }],
      "provisioners": [{
        "execute_command": "chmod +x {{ .Path }}; {{ .Vars }} sudo -E sh '{{ .Path }}'",
        "inline": [
          "apt-get update",
          "apt-get upgrade -y",
          "apt-get -y install nginx",

          "/usr/sbin/waagent -force -deprovision+user && export HISTSIZE=0 && sync"
        ],
        "inline_shebang": "/bin/sh -x",
        "type": "shell"
      }]
    }

**Build the image with Packer**

Finally launch Packer build:
```console
$ packer plugins install github.com/hashicorp/azure
    Installed plugin github.com/hashicorp/azure v2.1.3 in "/Users/lolwww/.config/packer/plugins/github.com/hashicorp/azure/packer-plugin-azure_v2.1.3_x5.0_darwin_arm64"

$ sudo packer build ubuntupro.json
    ....
    ==> Wait completed after 5 minutes 22 seconds

    ==> Builds finished. The artifacts of successful builds are:
    --> azure-arm: Azure.ResourceManagement.VMImage:

    OSType: Linux
    ManagedImageResourceGroupName: myResourceGroup
    ManagedImageName: myPackerImage
    ManagedImageId: /subscriptions/xxxxxxxxxxx/myResourceGroup/providers/Microsoft.Compute/images/myPackerImage
    ManagedImageLocation: East US
```

Check your newly created image:
```console
$ az image show --name myPackerImage --resource-group myresourcegroup
    {
      "hyperVGeneration": "V1",
      "id": "/subscriptions/xxxxx/resourceGroups/myresourcegroup/providers/Microsoft.Compute/images/myPackerImage",
      "location": "eastus",
      "name": "myPackerImage",
      "provisioningState": "Succeeded",
      "resourceGroup": "myresourcegroup",
      "sourceVirtualMachine": {
     "id": "/subscriptions/xxxxx/resourceGroups/pkr-Resource-Group-bnfu1xniud/providers/Microsoft.Compute/virtualMachines/pkrvmbnfu1xniud",
     "resourceGroup": "pkr-Resource-Group-bnfu1xniud"
      },
      "storageProfile": {
     "dataDisks": [],
     "osDisk": {
       "caching": "ReadWrite",
       "diskSizeGB": 30,
       "managedDisk": {
         "id": "/subscriptions/xxxxx/resourceGroups/pkr-Resource-Group-bnfu1xniud/providers/Microsoft.Compute/disks/xxxxx",
         "resourceGroup": "pkr-Resource-Group-xxxx"
       },
       "osState": "Generalized",
       "osType": "Linux",
       "storageAccountType": "Standard_LRS"
     },
     "zoneResilient": false
      },
      "tags": {
     "PlanInfo": "pro-22_04-lts",
     "PlanProduct": "0001-com-ubuntu-pro-jammy",
     "PlanPromotionCode": "",
     "PlanPublisher": "canonical",
     "dept": "Engineering",
     "task": "Image deployment"
      },
      "type": "Microsoft.Compute/images"
    }
```

**Create a VM based on the Image** \
The image was uploaded by Packer into Azure Images.\
Let’s create a VM based on it.\
Note: "Creating a virtual machine from a Marketplace image or a custom image sourced from a Marketplace image requires Plan information in the request.”

```console
$ az vm create \
--resource-group myResourceGroup \
--name testPackerVM \
--image myPackerImage \
--admin-username azureuser \
--plan-publisher canonical \
--plan-name pro-22_04-lts \
--plan-product 0001-com-ubuntu-pro-jammy \
--generate-ssh-keys

    {
      "fqdns": "",
      "id": "/subscriptions/xxxxx/resourceGroups/myResourceGroup/providers/Microsoft.Compute/virtualMachines/testPackerVM",
      "location": "eastus",
      "macAddress": "xxxxx",
      "powerState": "VM running",
      "privateIpAddress": "10.0.0.4",
      "publicIpAddress": "xxxxx",
      "resourceGroup": "myResourceGroup",
      "zones": ""
    }
```

SSH to the newly created VM and check PRO status:
```console
$ ssh azureuser@xxxxx

azureuser@testPackerVM:~$ sudo pro status
SERVICE      ENTITLED  STATUS   DESCRIPTION
anbox-cloud  yes   disabled Scalable Android in the cloud
esm-apps     yes   enabled  Expanded Security Maintenance for Applications
esm-infra    yes   enabled  Expanded Security Maintenance for Infrastructure
fips-preview yes   disabled Preview of FIPS crypto packages undergoing certification with NIST
fips-updates yes   disabled FIPS compliant crypto packages with stable security updates
livepatch    yes   enabled  Canonical Livepatch service
usg          yes   disabled Security compliance and audit tools

For a list of all Ubuntu Pro services, run 'pro status --all'
Enable services with: pro enable <service>

Account: xxxxx
Subscription: xxxxx
Valid until: Fri Dec 31 00:00:00 9999 UTC
Technical support level: essential
```
Try to enable a pro service (for example usg):

```console
azureuser@testPackerVM:~$ sudo pro enable usg

One moment, checking your subscription first
Updating Ubuntu Security Guide package lists
Ubuntu Security Guide enabled
Visit https://ubuntu.com/security/certifications/docs/usg for the next steps

azureuser@testPackerVM:~$ sudo pro status
SERVICE      ENTITLED  STATUS    DESCRIPTION
anbox-cloud  yes    disabled  Scalable Android in the cloud
esm-apps      yes    enabled  Expanded Security Maintenance for Applications
esm-infra    yes    enabled  Expanded Security Maintenance for Infrastructure
fips-preview  yes    disabled  Preview of FIPS crypto packages undergoing certification with NIST
fips-updates  yes    disabled  FIPS compliant crypto packages with stable security updates
livepatch    yes    enabled  Canonical Livepatch service
usg          yes    enabled  Security compliance and audit tools

For a list of all Ubuntu Pro services, run 'pro status --all'
Enable services with: pro enable <service>

Account: xxxxx
Subscription: xxxxx
Valid until: Fri Dec 31 00:00:00 9999 UTC
Technical support level: essential
```


**Build image and upload to Azure compute gallery**

As a prerequisite, create a compute gallery for packer images (powershell) :
```console
$Name = 'packerImageGallery'
$location = 'eastus'
$tags = @{'Environment' = 'Dev'; 'Description' = 'Resources for packer image builds' }

$resourceGroup = 'myResourceGroup'
$location = 'East US'
$gallery = New-AzGallery -GalleryName $Name -ResourceGroupName $resourceGroup -Location $location -Description 'Shared Image Gallery for packer builds' -Tag $tags -Verbose
```

Next an image definition is required, this will be referenced in the packer template to upload the image to the gallery (powershell): 

```console
$Name = 'packerImageGallery'
$location = 'eastus'
$imageName = 'ubuntupro22.04'
$resourceGroup = 'myResourceGroup'

$params = @{
GalleryName       = $Name
ResourceGroupName = $resourceGroup
Location          = $location
Name              = $imageName
OsState           = 'generalized'
OsType            = 'Linux'
Publisher         = 'canonical'
Offer             = '0001-com-ubuntu-pro-jammy'
Sku               = 'pro-22_04-lts'
PurchasePlanName    = 'pro-22_04-lts'
PurchasePlanProduct = '0001-com-ubuntu-pro-jammy'
PurchasePlanPublisher = 'canonical'
 
}

New-AzGalleryImageDefinition @params
```

The json template would have a new section - gallery details where to upload the image:\
&#x20;ubuntupro.json

    {
      "builders": [{
        "type": "azure-arm",

        "client_id": "xxxxx",
        "client_secret": "xxxxx",
        "tenant_id": "xxxxx",
        "subscription_id": "xxxxx",

        "managed_image_resource_group_name": "myResourceGroup",
        "managed_image_name": "myPackerImage2",

        "os_type": "Linux",
        "image_publisher": "canonical",
        "image_offer": "0001-com-ubuntu-pro-jammy",
        "image_sku": "pro-22_04-lts",

        "azure_tags": {
         "dept": "Engineering",
         "task": "Image deployment"
        },

        "location": "East US",
         "shared_image_gallery_destination": {
          "resource_group": "myResourceGroup",
           "gallery_name": "packerImageGallery",
           "image_name": "ubuntupro22.04",
           "image_version": "0.1.23",
           "replication_regions": "eastus"},
        "plan_info": {
         "plan_publisher": "canonical",
         "plan_name": "pro-22_04-lts",
         "plan_product": "0001-com-ubuntu-pro-jammy"
        },
        "vm_size": "Standard_DS2_v2"
      }],
      "provisioners": [{
        "execute_command": "chmod +x {{ .Path }}; {{ .Vars }} sudo -E sh '{{ .Path }}'",
        "inline": [
          "apt-get update",
          "apt-get upgrade -y",
          "apt-get -y install nginx",

          "/usr/sbin/waagent -force -deprovision+user && export HISTSIZE=0 && sync"
        ],
        "inline_shebang": "/bin/sh -x",
        "type": "shell"
      }]
    }

**Launch Packer build with updated json**
```console
$ sudo packer build ubuntupro.json
    ....
 ==> Wait completed after 14 minutes 17 seconds

==> Builds finished. The artifacts of successful builds are:
--> azure-arm: Azure.ResourceManagement.VMImage:

OSType: Linux
ManagedImageResourceGroupName: myResourceGroup
ManagedImageName: myPackerImage2
ManagedImageId: /subscriptions/xxxxx/resourceGroups/myResourceGroup/providers/Microsoft.Compute/images/myPackerImage2
ManagedImageLocation: East US
ManagedImageSharedImageGalleryId: /subscriptions/xxxxx/resourceGroups/myResourceGroup/providers/Microsoft.Compute/galleries/packerImageGallery/images/ubuntupro22.04/versions/0.1.23
SharedImageGalleryResourceGroup: myResourceGroup
SharedImageGalleryName: packerImageGallery
SharedImageGalleryImageName: ubuntupro22.04
SharedImageGalleryImageVersion: 0.1.23
SharedImageGalleryReplicatedRegions: eastus
```

**Create a VM based on the Image:** \
The image was uploaded by Packer to Azure Compute Gallery.\
Let’s create a VM based on it.

Note: "Creating a virtual machine from a Marketplace image or a custom image sourced from a Marketplace image requires Plan information in the request.”

```console
$ az vm create --resource-group myResourceGroup --name testPackerVM333 --admin-username azureuser --image "/subscriptions/xxxxx/resourceGroups/myResourceGroup/providers/Microsoft.Compute/galleries/packerImageGallery/images/ubuntupro22.04/versions/0.1.23" --generate-ssh-keys --plan-publisher canonical --plan-name pro-22_04-lts --plan-product 0001-com-ubuntu-pro-jammy

{
"fqdns": "",
"id": "/subscriptions/xxxxx/resourceGroups/myResourceGroup/providers/Microsoft.Compute/virtualMachines/testPackerVM333",
"location": "eastus",
"macAddress": "xxxxx",
"powerState": "VM running",
"privateIpAddress": "10.0.0.6",
"publicIpAddress": "xxxxx",
"resourceGroup": "myResourceGroup",
"zones": ""
}
```

SSH to the newly created VM and check PRO status:
```console
$ ssh azureuser@xxxxx

azureuser@testPackerVM:~$ sudo pro status
SERVICE      ENTITLED  STATUS   DESCRIPTION
anbox-cloud  yes   disabled Scalable Android in the cloud
esm-apps     yes   enabled  Expanded Security Maintenance for Applications
esm-infra    yes   enabled  Expanded Security Maintenance for Infrastructure
fips-preview yes   disabled Preview of FIPS crypto packages undergoing certification with NIST
fips-updates yes   disabled FIPS compliant crypto packages with stable security updates
livepatch    yes   enabled  Canonical Livepatch service
usg          yes   disabled Security compliance and audit tools

For a list of all Ubuntu Pro services, run 'pro status --all'
Enable services with: pro enable <service>
Account: xxxxx
Subscription: xxxxx
Valid until: Fri Dec 31 00:00:00 9999 UTC
Technical support level: essential
```
After that you can enable pro services as needed.

## **Conclusion**<a id="conclusion"></a>

We have learned how to build Ubuntu pro 22.04 based golden image with Packer and upload it to Azure Images and Azure Compute Gallery, then created VMs from the uploaded images.
