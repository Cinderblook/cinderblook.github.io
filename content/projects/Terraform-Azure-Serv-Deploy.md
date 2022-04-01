+++
title = "Terraform - Azure-Serv-Deploy"
description = "Terraofmr Automation"
type = ["posts","post"]
tags = [
    "terraform",
    "ansible",
    "automation",
    "azure",
    "windows-server",
]
date = "2022-02-28"
categories = [
    "Projects",
    "Terraform",
    "Ansible",
    "Azure",
    "Linux",
]
series = ["Terraform"]
[ author ]
  name = "Austin Barnes"
+++

# Overview

Deploy and Configure 4 Windows 2022 Datacenter Servers in Azure.
- Using Terraform in conjunction with Ansible: Create 4 Windows Servers
  - Configure them to be a Primary Domain Controller, Replica Domain Controller, DHCP server, and Fileshare server    
  - Automate intial setup of the 4 servers to accept Ansible configuration from a Linux VM in Azure created VIA the Terraform deployment

# Terraform
Main role: Deploy the Virtual Machines, setup Network environment, and provide intial parameters for both Windows and Linux environments running in the cloud
-   Setup the four Windows Servers (Primary Domain Controller, Replica Domain Controller, DHCP, Fileshare) in Azure
    - These will all be Windows 2022 Datacenter Servers running on Standard_DS1_V2 by default
-   Setup the one Linux server to deploy a pre-defined Ansible configuration across the Windows Environment for setting up Active Directory, DHCP, File shares, users, and groups.
    - This will all be an Ubuntu 18.04-LTS server running on Standard_B1s by default
    - It will use cloud-init to supply it the necessary setup at creation for Ansible and SSH connection VIA its public IP address.
-   Supply necessary networking variables (Network interfaces, Security Groups, IP Addressing)
-   Supply necessary files for automation of Windows & Linux environments (Cloud-Init & Windows Unattend files)

<br>

## Prerequisites
- Setup necessary Terraform [environment](https://learn.hashicorp.com/tutorials/terraform/install-cli)
- Install and setup [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) or preferred method of authentication to Azure
- Configure variables for desired outcomes (Outlined further down)
<br>

## Terraform process
- Using the Azure provider:
    - Login to Azure with `az connect`
- Once prepared with appropriate values and the networking is in place: 
    - Navigate to the Terraform directory and run these commands
    - `terraform init` Pull proper Terraform providers and modules used
    - `terraform validate` This will return whether the configuration is valid or not
    - `terraform apply` ... `yes` Actually apply the configuration

## Terraform File Structure
Create a new directory, and place the following files in it, with your own variables

###  *provider.tf* File
- Calls necessary providers and sets their versions to be used in the Terraform configuration/deployment. Informs Terraform which modules and providers to use.

``` tf
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = ">=2.91.0"
    }
  }
}
provider "azurerm" {
  features {}
}
```

###  *networking.tf* File
- Defines resources, security groups, security rules, network interfaces, subnets, and public IPs to be created in Azure. 
    - These variables are pulled from the VM creation resources
    - Managed with variables contained in terraform.tfvars file

networking.tf
```tf
# Create a resource group to maintain security settings along with network interfaces for VMs
resource "azurerm_resource_group" "east" {
  name     = "terra-resources"
  location = "East US"
}
# ASSIGN ADDRESS SPACE TO RESOURCE GROUP
resource "azurerm_virtual_network" "east" {
  name                = "east-network"
  address_space       = ["${var.east_address_spaces}"]
  location            = azurerm_resource_group.east.location
  resource_group_name = azurerm_resource_group.east.name
}
# ASSIGN SUBNET TO NETWORK ADDRESS SPACE
resource "azurerm_subnet" "subnet1" {
  name                 = "internal"
  resource_group_name  = azurerm_resource_group.east.name
  virtual_network_name = azurerm_virtual_network.east.name
  address_prefixes     = [var.east_subnets]
}
# Create public IP variable for Linux machine
resource "azurerm_public_ip" "linux_public" {
  name                = "PublicIp1"
  resource_group_name = azurerm_resource_group.east.name
  location            = azurerm_resource_group.east.location
  allocation_method   = "Static"

}
# Create public IP variable for Windows machine
resource "azurerm_public_ip" "win_public" {
  name                = "PublicIp2"
  resource_group_name = azurerm_resource_group.east.name
  location            = azurerm_resource_group.east.location
  allocation_method   = "Static"

}
# ASSIGN NETWORK INTERFACE PER VM WE WILL BE USING
resource "azurerm_network_interface" "linux1" {
  name                = "linux1-nic"
  location            = azurerm_resource_group.east.location
  resource_group_name = azurerm_resource_group.east.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.subnet1.id
    private_ip_address_allocation = "Static"
    private_ip_address            = var.linux1_priavte_ip    
    public_ip_address_id          = azurerm_public_ip.linux_public.id
  }
}
resource "azurerm_network_interface" "winserv1" {
  name                = "winserv1-nic"
  location            = azurerm_resource_group.east.location
  resource_group_name = azurerm_resource_group.east.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.subnet1.id
    private_ip_address_allocation = "Static"
    private_ip_address            = var.winserv1_private_ip
    public_ip_address_id          = azurerm_public_ip.win_public.id
  }
}
resource "azurerm_network_interface" "winserv2" {
  name                = "winserv2-nic"
  location            = azurerm_resource_group.east.location
  resource_group_name = azurerm_resource_group.east.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.subnet1.id
    private_ip_address_allocation = "Static"
    private_ip_address            = var.winserv2_private_ip
  }
}
resource "azurerm_network_interface" "winserv3" {
  name                = "winserv3-nic"
  location            = azurerm_resource_group.east.location
  resource_group_name = azurerm_resource_group.east.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.subnet1.id
    private_ip_address_allocation = "Static"
    private_ip_address            = var.winserv3_private_ip
  }
}
resource "azurerm_network_interface" "winserv4" {
  name                = "winserv4-nic"
  location            = azurerm_resource_group.east.location
  resource_group_name = azurerm_resource_group.east.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.subnet1.id
    private_ip_address_allocation = "Static"
    private_ip_address            = var.winserv4_private_ip
  }
}
# CREATE SECURITY GROUPs TO ALLOW SSH/RDP/ANSIBLE FOR VMs
resource "azurerm_network_security_group" "linux1" {
  name                = "Allow-SSH"
  location            = azurerm_resource_group.east.location
  resource_group_name = azurerm_resource_group.east.name

  security_rule {
    name                       = "SSH"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "22"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
}
resource "azurerm_network_security_group" "winserv" {
  name                = "Allow-RDP-SSH-ANS"
  location            = azurerm_resource_group.east.location
  resource_group_name = azurerm_resource_group.east.name
  security_rule {
    name                       = "RDP"
    priority                   = 101
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "3389"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
  security_rule {
    name                       = "SSH"
    priority                   = 102
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "22"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
   security_rule {
    name                       = "ANSIBLE"
    priority                   = 103
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "5985"
    source_address_prefix      = "${var.east_subnets}"
    destination_address_prefix = "*"
  }
}
# ASSIGN SECURITY GROUPS TO INTERFACES
# LINUX SSH
resource "azurerm_network_interface_security_group_association" "linux1" {
  network_interface_id      = azurerm_network_interface.linux1.id
  network_security_group_id = azurerm_network_security_group.linux1.id
}
# WINSERV RDP
resource "azurerm_network_interface_security_group_association" "winserv1" {
  network_interface_id      = azurerm_network_interface.winserv1.id
  network_security_group_id = azurerm_network_security_group.winserv.id
}
resource "azurerm_network_interface_security_group_association" "winserv2" {
  network_interface_id      = azurerm_network_interface.winserv2.id
  network_security_group_id = azurerm_network_security_group.winserv.id
}
resource "azurerm_network_interface_security_group_association" "winserv3" {
  network_interface_id      = azurerm_network_interface.winserv3.id
  network_security_group_id = azurerm_network_security_group.winserv.id
}
resource "azurerm_network_interface_security_group_association" "winserv4" {
  network_interface_id      = azurerm_network_interface.winserv4.id
  network_security_group_id = azurerm_network_security_group.winserv.id
}
```

###  *variables.tf, terraform.tfvars* Files
-  Alter variables within these files to ensure they meet your environment needs
    - *variables.tf*
        - Declare variables that will be used with the Terraform configuration (Delcared intially or explicitely here as `locals` variables)
        - Specific local variables used for Windows .xml files
            - *firsT_logon_commands* local variable points to a .xml file to configure first time logon in Windows. This enables each server to recieve Winrm data on port 5985 for Ansible configuration
            - *auto_logon_ runs a .xml configuration to log in once right after intial creation of VM. This allows *first_logon_commands* to execute automatically

variables.tf

 ```tf
 variable "winserv_vm_os_publisher" {}
 variable "winserv_vm_os_offer" {}
 variable "winserv_vm_os_sku" {}
 variable "winserv_vm_size" {}
 variable "winadmin_username" {}
 variable "winadmin_password" {}
 variable "winserv_license" {}
 variable "winserv_sa_type" {}
 variable "winserv_pdc" {}
 variable "winserv_rdc" {} 
 variable "winserv_dhcp" {} 
 variable "winserv_file" {}    
 variable "linux_server" {}
 variable "linux_vm_os_publisher" {}
 variable "linux_vm_os_offer" {}
 variable "linux_vm_os_sku" {}
 variable "linux_vm_size" {}
 variable "linux_ssh_key" {}
 variable "linux_sa_type" {}
 variable "linux_ssh_key_pv" {}
 variable "winserv_allocation_method" {}
 variable "east_address_spaces" {}
 variable "east_subnets" {}
 variable "winserv_public_ip_sku" {}
 variable "winserv1_private_ip" {}
 variable "winserv2_private_ip" {}
 variable "winserv3_private_ip" {}
 variable "winserv4_private_ip" {}
 variable "linux1_priavte_ip" {}
 
 locals{
     first_logon_commands        = file("${path.module}/          winfiles/FirstLogonCommands.xml")
     auto_logon                  =           "<AutoLogon><Password><Value>${var.winadmin_password}</          Value></Password><Enabled>true</Enabled><LogonCount>1</          LogonCount><Username>${var.winadmin_username}</          Username></AutoLogon>"
 }
 ```

  - *terraform.tfvars*
      - Assign variables values here. These will be used with the Terraform configuration. If left blank, you can assign the variable at the terminal level when running the `terraform apply` 
          - Alter Network values to desired IP addressing scheme
              -  **Ensure IP addressing matches that in the Ansible configuration inventory.yml**
          - Here you can alter azure values for _publisher_, _offer_, _sku_, _size_, _sa_, and _license_ information for the Windows/Linux VMs
          - Additionally, ensure `linux_ssh_key` point to your public Key `id_rsa.pubc` file
          - I recommend to change _winadmin_username_ & _winadmin_password_ variables to sensetive and blank so you can delcare them in preferrably Vaulty or via the CLI 
              - **_winadmin_username_ & _winadmin_password_ MUST MATCH WHAT IS IN ANSIBLE /group_vars/all.yml**

terraform.tfvars
```tf
# Azure Windows Server related params
winserv_vm_os_publisher     ="MicrosoftWindowsServer"
winserv_vm_os_offer         = "WindowsServer"
winserv_vm_os_sku           = "2022-Datacenter"
winserv_vm_size             = "Standard_DS1_V2"
winserv_license             = "Windows_Server"
winserv_allocation_method   = "Static"
winserv_public_ip_sku       = "Standard"
winserv_sa_type             = "Standard_LRS"

# Azure Linux Server related params
linux_vm_os_publisher = "Canonical"
linux_vm_os_offer     = "UbuntuServer"
linux_vm_os_sku       = "18.04-LTS"
linux_vm_size         = "Standard_B1s"
linux_ssh_key         ="Local-PUBLIC-SSH-KEY-Here"
linux_sa_type         = "Premium_LRS"
linux_ssh_key_pv      = "Local-PRIV-SSH-KEY-Here"

# Which Windows administrator password to setduring vm           customization
winadmin_username = "SuperAdministrator"
winadmin_password = "Password1234"

# Naming Schemes 
winserv_pdc    = "ajb-pdc"
winserv_rdc    = "ajb-rdc"
winserv_dhcp   = "ajb-dhcp"
winserv_file   = "ajb-file"
linux_server   = "ajb-operator"

# Networking Variables
winserv1_private_ip   = "10.0.1.10"
winserv2_private_ip   = "10.0.1.11"
winserv3_private_ip   = "10.0.1.12"
winserv4_private_ip   = "10.0.1.13"
linux1_priavte_ip     = "10.0.1.9"
east_address_spaces  = "10.0.0.0/16"
east_subnets         = "10.0.1.0/24"
```

### *01-LinuxClient.tf* & *02-WinServers.tf* Files
- Here the creation of the VMs occur. Resources pull data from _networking.tf_, _variables.tf_, _terraform.tfvars_ files.
    - Windows VMs are assigned unattend configurations for first time setup (/winfiles/FirstLogonCommands.xml && _auto_logon_ variable data)
    - Linux Machine is assigned a cloud-init file configuraiton for first time setup (/cloudinit/custom.yml) We will create this in the next step.

01-LinuxClient.tf
```tf
# This pulls a Ubuntu Datacenter from Microsoft's VM platform directly
resource "azurerm_linux_virtual_machine" "operator" {
  name                = var.linux_server
  resource_group_name = azurerm_resource_group.east.name
  location            = azurerm_resource_group.east.location
  size                = var.linux_vm_size
  admin_username      = var.winadmin_username
  network_interface_ids = [
    azurerm_network_interface.linux1.id
  ]

  admin_ssh_key {
    username   = var.winadmin_username
    public_key = file("${var.linux_ssh_key}")
  }

  # Cloud-Init passed here
  custom_data = data.template_cloudinit_config.config.rendered

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = var.linux_sa_type
  }

  source_image_reference {
    publisher = var.linux_vm_os_publisher
    offer     = var.linux_vm_os_offer
    sku       = var.linux_vm_os_sku
    version   = "latest"
  }

  depends_on = [azurerm_resource_group.east, azurerm_network_interface.linux1]
}

# Create cloud-init file to be passed into linux vm
data "template_file" "user_data" {
  template = file("./cloudinit/custom.yml")
}
 
# Render a multi-part cloud-init config making use of the part
# above, and other source files
data "template_cloudinit_config" "config" {
  gzip          = true
  base64_encode = true

  # Main cloud-config configuration file.
  part {
    filename     = "init.cfg"
    content_type = "text/cloud-config"
    content      = "${data.template_file.user_data.rendered}"
  }
}

  # Pass Ansible File into created Linux VM using SCP (SSH Port 22)
resource "null_resource" "copyansible"{ 
  connection {
    type        = "ssh"
    host        = azurerm_public_ip.linux_public.ip_address
    user        = var.winadmin_username
    private_key = file("${var.linux_ssh_key_pv}")
  }
  
  provisioner "file" {
    source = "${path.module}/Ansible"
    destination = "/tmp/" 
  }
  depends_on = [azurerm_linux_virtual_machine.operator]
}
```
02-WinServers.tf
```tf
# This pulls the latest Windows Server Datacenter from Microsoft's VM platform directly
resource "azurerm_windows_virtual_machine" "pdc" {
  name                = var.winserv_pdc
  resource_group_name = azurerm_resource_group.east.name
  location            = azurerm_resource_group.east.location
  size                = var.winserv_vm_size
  admin_username      = var.winadmin_username
  admin_password      = var.winadmin_password
  network_interface_ids = [
    azurerm_network_interface.winserv1.id
  ]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = var.winserv_sa_type
  }

  source_image_reference {
    publisher = var.winserv_vm_os_publisher
    offer     = var.winserv_vm_os_offer
    sku       = var.winserv_vm_os_sku
    version   = "latest"
  }

  additional_unattend_content {
    content = local.auto_logon
    setting = "AutoLogon"
  }

  additional_unattend_content {
    content = local.first_logon_commands
    setting = "FirstLogonCommands"
  }

  depends_on = [azurerm_resource_group.east, azurerm_network_interface.winserv1]
}
resource "azurerm_windows_virtual_machine" "rdc" {
  name                = var.winserv_rdc
  resource_group_name = azurerm_resource_group.east.name
  location            = azurerm_resource_group.east.location
  size                = var.winserv_vm_size
  admin_username      = var.winadmin_username
  admin_password      = var.winadmin_password
  network_interface_ids = [
    azurerm_network_interface.winserv2.id
  ]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = var.winserv_sa_type
  }

  source_image_reference {
    publisher = var.winserv_vm_os_publisher
    offer     = var.winserv_vm_os_offer
    sku       = var.winserv_vm_os_sku
    version   = "latest"
  }

  additional_unattend_content {
    content = local.auto_logon
    setting = "AutoLogon"
  }

  additional_unattend_content {
    content = local.first_logon_commands
    setting = "FirstLogonCommands"
  }

  depends_on = [azurerm_resource_group.east, azurerm_network_interface.winserv2]
}
resource "azurerm_windows_virtual_machine" "dhcp" {
  name                = var.winserv_dhcp
  resource_group_name = azurerm_resource_group.east.name
  location            = azurerm_resource_group.east.location
  size                = var.winserv_vm_size
  admin_username      = var.winadmin_username
  admin_password      = var.winadmin_password
  network_interface_ids = [
    azurerm_network_interface.winserv3.id
  ]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = var.winserv_sa_type
  }

  source_image_reference {
    publisher = var.winserv_vm_os_publisher
    offer     = var.winserv_vm_os_offer
    sku       = var.winserv_vm_os_sku
    version   = "latest"
  }

  additional_unattend_content {
    content = local.auto_logon
    setting = "AutoLogon"
  }

  additional_unattend_content {
    content = local.first_logon_commands
    setting = "FirstLogonCommands"
  }

  depends_on = [azurerm_resource_group.east, azurerm_network_interface.winserv3]
}
resource "azurerm_windows_virtual_machine" "file" {
  name                = var.winserv_file
  resource_group_name = azurerm_resource_group.east.name
  location            = azurerm_resource_group.east.location
  size                = var.winserv_vm_size
  admin_username      = var.winadmin_username
  admin_password      = var.winadmin_password
  network_interface_ids = [
    azurerm_network_interface.winserv4.id
  ]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = var.winserv_sa_type
  }

  source_image_reference {
    publisher = var.winserv_vm_os_publisher
    offer     = var.winserv_vm_os_offer
    sku       = var.winserv_vm_os_sku
    version   = "latest"
  }

  additional_unattend_content {
    content = local.auto_logon
    setting = "AutoLogon"
  }

  additional_unattend_content {
    content = local.first_logon_commands
    setting = "FirstLogonCommands"
  }

  depends_on = [azurerm_resource_group.east, azurerm_network_interface.winserv4]
}


```
### outputs.tf File
- Provides necessary ip information that is allocated to the VMs created.
- This information by default includes:
    - Private IPs for all 5 deployed VMs (Which we know will by based on variables.tf file data)
    - Public IP for Linux machine (Not known by default, will be used for SSH connection if needed).

outputs.tf
```tf
output "Public_IP_Linux" {
    value = azurerm_public_ip.linux_public.ip_address
}
output "Public_IP_Windows" {
    value = azurerm_public_ip.win_public.ip_address
}
output "Private_IP_Linux" {
    value = azurerm_network_interface.linux1.private_ip_address
}

output "Private_IP_WinServ" {
    value = [
        "PDC: ${azurerm_windows_virtual_machine.pdc.private_ip_address}",
        "RDC: ${azurerm_windows_virtual_machine.rdc.private_ip_address}",
        "DHCP: ${azurerm_windows_virtual_machine.dhcp.private_ip_address}",
        "FILE: ${azurerm_windows_virtual_machine.file.private_ip_address}"
        ]
}
```
## Useful Azure related functions
Finding variable information for VM Images variables:
- You can use this command in Azure CLI to find UbuntuServer data. Change the values in offer, publisher, location, and sku for various other images.
 ````Powershell
    az vm image list \
    --location westus \
    --publisher Canonical \  
    --offer UbuntuServer \    
    --sku 18.04-LTS \
    --all --output table
````
-  "Check out Microsoft's" [documentation](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/cli-ps-findimage) on finding VM information


# Ansible
Main role: Configure the deployed Virtual Machines.
<br>
Check out the repository on [GitHub](https://github.com/Cinderblook/Azure-Serv-Deploy) for the configuration files!
-   Setup Windows Server Feature: Domain
    - Primary Domain Controller 
    - Replica Domain Controller
    - Auto-Join the Virutal Machines to the respective Domain created
    - Create a few users and groups within Active Directory
-   Setup Windows Ssrver Feature: DHCP 
    - Setup DHCP Scope
    - Authorize it to the Domain.
-   Setup Windows Server Feature: File Sharing
    - Create two shares
        - An employee share and administrator share. These shares are assigned group permissions.
-   Common Configurations
    - Enable RDP and allow it through the firewall on all windows servers created at server level
## Ansible Variable files 
- *inventory.yml*
    - Modify hosts associated with the playbook. Assign the IP addressing
        - **MUST MATCH _terraform.tfvars_ VARIABLE IP ADDRESSING**
- *winlab.yml*
    - Associate 'roles' to the hosts identified in the inventory file. 
    - These 'roles' are folders within the directory containing a set of code to configure per host
- *ansible.cfg*
    - Tells ansible variable information. In this scenario, identifies to use inventory.yml file.
- *./group_vars/all.yml*
    - Contains specific variable information used within the ./roles/* Ansible code.
    - Alter user, password, port, connection, and cert variable information
    - Alter domain variables as well

## Running Ansible
This is taken care of with terraform cloud-init file along with the file provisioner. The alternative would be below.
- On Linux Machine,
    - Requires: Python-pip, ansible-galaxzy-azure.azure_preview_modules
    - To Run: Navigate to Ansible directory and type `ansible-playbook winlab.yml`

# Useful Resources 
## Terraform Resources
- Terraform [Documentation](https://www.terraform.io/docs)
- Azure [Provider](https://registry.terraform.io/providers/hashicorp/azurerm/2.96.0) & [Modules](https://registry.terraform.io/modules/Azure/compute/azurerm/latest)
- Cloud-init [Documentation](https://cloudinit.readthedocs.io/en/latest/)
- [terraform-provider-azurerm](https://github.com/hashicorp/terraform-provider-azurerm) examples and documentation on GitHub
## Ansible Resources
- Ansible [Documentation](https://docs.ansible.com/)
  - [Windows-Modules](https://galaxy.ansible.com/ansible/windows?extIdCarryOver=true&sc_cid=701f2000001OH7YAAW)

