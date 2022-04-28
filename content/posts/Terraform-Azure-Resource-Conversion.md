+++
title = "Terraform - Converting Azure Resources"
description = "Converting Azure resources to Terraform resources"
type = ["posts","post"]
tags = [
    "terraform",
    "aztfy",
    "automation",
    "azure",
    "windows-server",
]
date = "2022-04-27"
categories = [
    "Projects",
    "Terraform",
    "aztfy",
    "Azure",
    "Linux",
]
series = ["Terraform"]
[ author ]
  name = "Austin Barnes"
+++

# Overview

Convert existing resources in Azure to Terraform files (Terrafy/aztfy)
- Microsoft Recently released a sweet tool called aztfy. This enables you to 'terrafy' existing Azure resources. This effectively allows for total conversions of existing data in Azure to be modified and updated/managed using Terraform. Whether it be for backup reasons, future management of resource changes, or cloud implmentation purposes, this could prove extremely useful.


# Prerequisites
- Have Go installed in your dev environment
  - For Windows, go to [this site](https://go.dev/doc/install) and install Go
- Go to Microsoft's Azure Aztfy [Github Page](https://github.com/Azure/aztfy)
  - Pull the repository to your local machine and install the go module they provide for aztfy
    - ``` 
      git clone https://github.com/Azure/aztfy
      cd "Directory-You-Cloned-To"
      go install 
      ```

# Using Aztfy
Once you have a resource group in Azure, you can run the following command to pull specific resources from that group
* `aztfy Azure-Resource-Group-Here`

From here, you can navigate each individual resource within that group and import them individually from Azure to your local machine. It generates the .tf files, terraform state files, and all necessary JSON components. 

* Example of Aztfy Result:
![JRCustomHomes](/aztfy-example1.png 'aztfy-example1') 

Navigate into directory you cloned aztfy to, and double check the clone worked by attempting a `terraform apply` and if it says no changes needed, you are solid!
# Useful Resources 
## Terraform Resources
- Microsoft [Blog Post](https://techcommunity.microsoft.com/t5/azure-tools-blog/announcing-azure-terrafy-and-azapi-terraform-provider-previews/ba-p/3270937)
- Azure [Documentation for AzAPI](https://docs.microsoft.com/en-us/azure/developer/terraform/overview-azapi-provider)


