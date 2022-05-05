---
author: "Austin Barnes"
title: "Terraform - Spotify Automation"
description: "Using Terraform to customize and manage playlist in Spotify"
tags: [
    "terraform",
    "automation",
    "spotfy",
]
date: 2022-05-02
thumbnail: /covers/SpotifyAutomation.png
---

# Overview
Manipulating spotify playlist using Terraform, because why not? 

**Check out all of the configuration files on [GitHub](https://github.com/Cinderblook/tacklebox/tree/main/Terraform/Spotify) at the repository. I have various examples of Terraform using the Spotify Provider within as well!**

## Understanding the provider
For all Spotify API usage with Terraform, You must either run a oauth server locally and just provide the API key, or use something online and provide all 4 necessary features (api, token_id, user, and oauth_url). You will need to create a Spotify Developer account [here](https://developer.spotify.com/)
<br>

*If you are unfamiliar with running an oauth server, I highly recommend to use this oauth server provided by the creator of the Terraform Spotify Provider! Check it out on his [Github page](https://github.com/conradludgate/terraform-provider-spotify). All the necessary instructions to get started with it will be on there.*

--------

# Multi-Artist Playlist Creation
The following code will represent how to use Terraform to create a Multi-artist playlist based on artist names


## Creating the Provider.tf file
In here, I defined the version to use for the provider, along with setting variables for `auth_server`, `api_key`, `username`, and `token_id`

``` tf
terraform {
  required_providers {
    spotify = {
      source = "conradludgate/spotify"
      version = "0.2.7"
    }
  }
}

provider "spotify" {
  # Refernced in tfvariables.tfvars
  auth_server = "${var.spotify_oauth_url}"
  api_key = "${var.spotify_api_key}"
  username = "${var.spotify_user}"
  token_id = "${var.spotify_token_id}"
}
```

## Creating the playlist.tf file
This will be where the core of the Terraform structure is provided. It will use variables defined in the other files to loop through and generate the desired playlist. 
```tf
# Create the playlist
resource "spotify_playlist" "custom" {
  depends_on = [
      data.spotify_search_track.custom_count,
    ]
  name        = var.playlist_name
  description = var.playlist_desc
  public = true
  tracks = flatten([
      local.all_track_ids 
  ])
  }
# Find ids for songs
data "spotify_search_track" "custom_count" {
  count = length(var.artist_list)
  artist = "${var.artist_list[count.index]}"
  limit = 10
  explicit = true
}
# Local Variables to concat id lists into  one
locals {
  depends_on = [data.spotify_search_track.custom_count]
  all_tracks = concat(data.spotify_search_track.custom_count[*])
  all_track_ids = flatten(local.all_tracks[*].tracks[*].id)
  all_track_names = flatten(local.all_tracks[*].tracks[*].name)
}

output "list" {
  value = local.all_track_names
}
```

## Create your varialbes in a variables.tf
``` tf
variable "spotify_api_key" {
  type        = string
  description = "Set this as the APIKey that the authorization proxy server outputs"
}
variable "spotify_token_id" {
  type        = string
  description = "Token ID provided from oauth"
}
variable "spotify_user" {
  type        = string
  description = "username for oauth"
}
variable "spotify_oauth_url" {
  type        = string
  description = "url path of oauth server"
}
variable "playlist_name" {
  type        = string
  description = "name of playlist"
}
variable "playlist_desc" {
  type        = string
  description = "desc of playlist"
}
variable "artist_list" {
  type = list
  description = "Spotify Artists"
}
```

## Define the variables.auto.tfvars file
Ensure to change the information within this file to fit your needs <br>

In this file, reference your `spotify_api_key`, `spotify_token_id`, `spotify_oauth_url`, and `spotify_user`

``` tf
spotify_api_key = "Spotify API key here"
spotify_user = "oauth User"
spotify_token_id = "oauth Token Here"
spotify_oauth_url = "oauth site  here"
playlist_name = "name"
playlist_desc ="description"
artist_list = ["Artist", "Go", "Here"]
```

## Creating the playlist
After the aformentioned files are created and customized to your liking, just execute the terraform commands:
``` tf
terraform init
terraform apply --auto-approve
```
Open your spotify and see the newly genereated playlist, and delete the newly generated list with:
``` tf
terraform destroy --auto-approve
```
Since Terraform is a stateful deployment, as long as you have control of the state file that gets created alongside Terraform, you can manage and alter the created playlist as you wish.

## Useful Resources
Refer to this official spotify guide for creating a self-hosted oauth server, or using the free open source one the creater made: [Github](https://github.com/conradludgate/terraform-provider-spotify)

[Spotify Provider](https://registry.terraform.io/providers/conradludgate/spotify/latest/docs)on Terraform Provider list
