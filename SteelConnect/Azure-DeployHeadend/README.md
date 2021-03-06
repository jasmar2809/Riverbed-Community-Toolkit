# Cookbook - Deploy SteelConnect EX Standalone Headend in Azure

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Deployment](#deployment)
    - [1. Create Azure Image Resources](#1-create-azure-image-resources)
    - [2. Deploy using Terraform template](#2-deploy-using-terraform-template)
- [Connect to the appliances](#connect-to-the-appliances)

## Overview

This cookbook explains how to deploy a SteelConnect-EX Standalone Headend in Azure, in the location of your choice.

![cloud architecture](images/steelconnect-ex-headend-standalone-architecture.png)

The Cookbook uses the following parameters and default values.

| Parameters | Default Value |
| --- | --- |
| SD-WAN overlay network | 99.0.0.0 /8 |
| VNET | 10.100.0.0/16 |
| Azure VM Size | Standard_F8s_v2 |
| Azure Location* | *no default value* |
| SSH keypair | *generated in local files ssh-timestamp and ssh-timestamp.pub* |
| passphrase of SSH key | riverbed-community |

> The cookbook let you choose any **Azure Location** for the deployment, for example **West-US**, **West-Europe** or **Korea Central**. Thus in some location the default VM size is not available, for example Standard_F8s_v2 is currently not available in Switzerland North, in that case please refer to the SteelConnect deployment guide and adapt the cookbook script parameters to use an other VM size.

## Prerequisites

| Tasks | Description |
| --- | --- |
| 1. Connect to the Azure Portal and check there is enough vCPU available in the target location: navigate to your Subscription details, open Usage and Quota menu|<ul><li>Sign-in on [Azure portal](https://portal.azure.com)</li><li>At least **24 vCPU available for the Standard FsV2 Compute in the target Location**</li><li>Hit "Request Increase" button if you need more vCPU</li></ul>![quota](images/azure-westus-fsv2-quota.png)|
| 2. Create a resource group in the location where you will deploy SteelConnect EX appliances| <ul><li>Resource Group name: **Riverbed-Images**</li><li>Location: **Target location for appliances**</li></ul> |
| 3. Create a Storage Account resource  | <ul><li>Storage Account Name: *a unique name*</li><li>Location: **Target location for appliances**</li><li>Replication: Locally-redundant storage (LRS) is ok</li></ul> |
| 4. Create a Blob container in the Storage Account previously created | <ul><li>Container name: **images**</li></ul> |
| 5. Generate a temporary Shared Access Signature for this Storage Account/Container | <ul><li>End date: few days</li><li>Hit **Generate SAS and connection string**</li></ul>![azure-sa-sas-creation](images/azure-sa-sas-creation.png)|
| 6. Send a request to Riverbed Support with the **SAS and connecting string** generated previously and check you received images in your Blob Container: FlexVNF, Director and Analytics |<ul><li>Request to [Riverbed Support](https://support.riverbed.com/)</li></ul> |

## Deployment

### 1. Create Azure Image Resources

| Tasks | Description |
| --- | --- |
| 1. In the Azure Portal, navigate to the Resource group **Riverbed Images** |Go to [Azure portal](https://portal.azure.com)|
| 2. Add new resource, select Image, and hit Create new| ![add button](images/azure-resource-group-add-button.png) |
| 3. Fill parameters to create an Image resource for the **SteelConnect EX FlexVNF**| <ul><li>Name: **steelconnect-ex-flexvnf**</li><li>Location: **Target location** for appliances</li><li>OS disk type: **Linux**</li><li>Storage Blob: url of the **flexvnf vhd** in the storage account blobs container</li><li>Storage type: Premium SSD recommended</li></ul>|
| 4. Repeat **step 2.** and fill the parameters to create an Image resource for the **SteelConnect EX Director**| <ul><li>Name: **steelconnect-ex-director**</li><li>Location: **Target location** for appliances</li><li>OS disk type: **Linux**</li><li>Storage Blob: url of the **director vhd** in the storage account blobs container</li><li>Storage type: Premium SSD recommended</li></ul>|
| 5. Repeat **step 2.** and fill the parameters to create an Image resource for the **SteelConnect EX Analytics**| <ul><li>Name: **steelconnect-ex-analytics**</li><li>Location: **Target location** for appliances</li><li>OS disk type: **Linux**</li><li>Storage Blob: url of the **analytics vhd** in the storage account blobs container</li><li>Storage type: Premium SSD recommended</li></ul>|

#### Example

When the import is done, the resource group will contain a storage account and an image resource for each appliance. In the Azure portal, it should looks like this:

![resource group](./images/steelconnect-ex-import-vhd-images-resources.png)

### 2. Deploy using Terraform template

#### 1. Open Azure Cloud Shell and select PowerShell console

Launch Cloud Shell from Azure Portal, or [shell.azure.com](https://shell.azure.com), or by clicking [![Embed launch](https://shell.azure.com/images/launchcloudshell.png "Launch Azure Cloud Shell")](https://shell.azure.com)

#### 2. Get Riverbed Community Toolkit sources

The following PowerShell commands initialize the console and download the sources from Riverbed Community Toolkit git repository on GitHub.

```PowerShell
# Check the Azure context
# i.e check subscription and tenant id are correct
Get-AzContext

# Comment our the line below and replace {your subscription name} if you need to select a different subscription
# Set-AzContext -SubscriptionName "{your subscription name}"

# Get a local copy of the Riverbed Community Toolkit scripts from Github
git clone https://github.com/riverbed/Riverbed-Community-Toolkit.git
```

#### 3. Stage variables for Terraform

The following PowerShell commands prepare the parameters file *terraform.vartf* for Terraform and generate a keypair for SSH stored local files *ssh-timestamp* and *ssh-timestamp.pub*.

```PowerShell
Set-Location ./Riverbed-Community-Toolkit
Set-Location ./SteelConnect/Azure-DeployHeadend/scripts

./SteelConnect-EX_Stage-DefaultHeadhendStandalone.ps1
```

#### 4. Deploy Terraform

The following PowerShell commands launch the deployment using Terraform ((init, plan and apply)

```PowerShell
../../Azure-DeployHeadend/scripts/SteelConnect-EX_Deploy-Terraform.ps1
```

#### 5. Keep the output

After 3 to 5 minutes, the deployment finishes and the Terraform deployment output gives useful information such as WebConsole URL and Public IP for each appliance.

![terraform output](./images/steelconnect-ex-terraform-output.png)

In the Azure portal, the resource group contains all the resources.

![resource group](./images/steelconnect-ex-headend-resources.png)

## Connect to the appliances

Appliances can be accessed via Azure Serial Console, SSH or Webconsole.

- For example, connect to the Director VM with Azure Serial Console:

![resource group](images/steelconnect-ex-director-serial-console.png)

- For example, connect to the webconsole  of the Director: [https://{{your Director public IP}}]()

- For example, connect to the Directory VM with SSH using the generated keypair protected with a passphrase (see [cookbook default values](#overview)). 

```shell
# replace {{your sshkey-timestamp}} with the actual file name generated in the current directory
# replace {{your Director public IP}} with the actual IP of the Director, see terraform output
ssh -i {{your sshkey-timestamp}} Administrator@{{your Director public IP}}
```

## Copyright (c) 2020 Riverbed Technology