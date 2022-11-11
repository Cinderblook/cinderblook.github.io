---
author: "Austin Barnes"
title: "OpenVPN Site-to-Site - HomeLab"
description: "VPN Solution without having a Public IP address"
tags: [
    "openvpn",
    "vpn",
    "terraform",
    "azure",
    "site-to-site",
]
date: 2022-11-10
thumbnail: /covers/openvpn_sitetosite.png
---

# Overview

A quick guide to setup and create an openvpn server that connects to a client to form a site-to-site. A solution to have a VPN into your home network without having a public IP available to you at home.


## Requirements
---
- Cloud VM with a public IP running an OpenVPN server
- Home VM/client running OpenVPN client

## Steps
---
1. Create Linux VM in chosen cloud provider
2. Setup Cloud VM OpenVPN Server
    1. Configure User Settings
    2. Configure VPN Settings
3. Setup Home OpenVPN client
    1. Setup Client to forward IPv4 traffic and use NAT
    2. Automate connection at boot up


## Create Linux VM in chosen cloud provider
---
It is important to remember to setup the firewall settings applied to the VPN to allow necessary ports. Port 22 for SSH and OpenVPN's server port 943 are necessary.

- Might add a Terraform file to set this up in Azure later for automation reasons.


## Setup Cloud VM OpenVPN Server
---
SSH into your cloud server using the public IP address. 


1. Follow this [guide for setting up OpenVPN server](https://openvpn.net/vpn-software-packages/ubuntu/)

###  Configure User Settings

1. Create a user under User Management -> User Profiles
2. Under User Management -> User Permissions, give the user Allow Auto-login rights.
    - ![User Permissions](/examples/openvpn_userlogin.png "User Permissions")
3. Under that user, hit More settings.
    - Change Access Control to Use Routing. Then enter the network you wish the user to be able to access while on the VPN.
    - Enable Access from both VPN clients and Server-side private subnets
    - Change VPN Gateway to 'Yes', and enter client side subnets you wish to access.
    - ![User Settings](/examples/openvpn_UserPermissiosnedit.png "User Settings")

### Configure VPN Settings
1. Under the Configuration -> VPN Settings tab.
    - Change Routing to 'Yes, using Routing. Enter private sub nets clients should be able to access.
    - Change 'Allow access form these private subnets to all VPN client IP addresses and subnets' to 'Yes'
    - Change 'Should clients be allowed to access network services on the VPN gateway IP address?' to 'Yes'
    - I recomeend enabling all traffic being sent over VPN, due to my slow connection at home I have it off.
    - DNS settings are preference.
    - ![VPN Settings](/examples/openvpn_VPNSettings.png "VPN Settings")

## Setup Home OpenVPN client
---
*For reference, this will be based on using an Ubuntu 20.04 VM.*
1. First install the client
    - `apt update -y && apt install openvpn -y` 
2. Download thee client.vpn file from your VPN server
    - Navigate to the URL and sign in as the VPN user going to authenticate the client
    - ![client.ovpn Download](/examples/openvpn_autologin.png "client.ovpn Download")
3. Upload the config file generated from the autologin profile of VPN user to the VM
    - SCP to move file to VM ex: `scp \local\file\location\client.ovpn user@server:\etc\openvpn`
    - Connect client to server by running `openvpn --config client.ovpn --daemon`

###  Setup Client to forward IPv4 traffic
We must enable IP forwarding along with NAT on the Ubuntu OpenVPN client in order for traffic to reach your internal services on the other end of the site-to-site VPN.

This is achieved with the following steps:

1. Enable IP forwarding 
    - Edit /etc/sysctl.conf file
        - `nano /etc/sysctl.conf` 
    - Uncomment the line 
        - `net.ipv4.ip_forward=1`
    - Restart sysctl to enable forwarding 
        - `sysctl -p`
2. Enable NAT
    - `iptables -t nat -A POSTROUTING -j MASQUERADE`
3. Keep settings persistent between boots
    - `apt install iptables-persistent`
    - `iptables-save > /etc/iptables/rules.v4`

### Automate connection at boot up
By altering the ovpn into a config file, and moving it into the openvpn directory /etc/openvpn, at bootup the client will autheitnciate with the server if it is able to.
Move client.ovpn file and rename it to client.conf

1. `mv /etc/openvpn/client.ovpn /etc/openvpn/client.conf` 

---

## Useful Resources
* [OpenVPN](https://openvpn.net/)
* [Hugo Themes](https://hugothemesfree.com/)