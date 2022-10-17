---
author: "Austin Barnes"
title: "Hugo - Creating a free website"
description: "Using Hugo to create a static website for free in Github"
tags: [
    "hugo",
    "automation",
    "github",
    "website",
]
date: 2022-10-08
thumbnail: /covers/FreeHugoSiteGitPages.png
---

# Overview
Using Github pages, Github actions, and Hugo to create a static website for free. 

**Check out all of the configuration files on [GitHub](https://github.com/Cinderblook/cinderblook.github.io) at the repository.**

## Prerequisites 
- Have a Domain name purchased (Will be demonstrating CloudFlare in this guide)
- Have a GitHub Account

## Steps
1. Create a Public Repository
2. Change Repository settings
    1. Custom Domain Name (Optional)
3. Setup Hugo
4. Picking a Theme
5. Commit

## Create a Public Repository
Go ahead and create a new repository. It is important that it is named something like   `example.github.io`

Next, navigate into the .github/workflows/ folder created by default. Create a file named `example.yml` (*This should be the name as the  `example.github.io`*)

Add the following to the `example.yml` file
```yml
name: github pages

on:
  push:
    branches:
      - main  # Set a branch to deploy
  pull_request:

jobs:
  deploy:
    runs-on: ubuntu-20.04
    steps:
      - uses: actions/checkout@v2
        with:
          submodules: true  # Fetch Hugo themes (true OR recursive)
          fetch-depth: 0    # Fetch all history for .GitInfo and .Lastmod
          
      - name: Setup Node
        uses: actions/setup-node@v2
        with:
          node-version: '14'
        
      - name: Install dependencies
        run: |
          npm install postcss -D
          npm install -g postcss-cli
          npm install -g autoprefixer
          
      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'
          # extended: true

      - name: Build
        run: npm i && hugo -D --gc #hugo --minify

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        #if: github.ref == 'refs/heads/master'
        with:
          github_token: ${{ secrets.SECRET_HUGO }}
          publish_dir: ./public
```
This will setup the GitHub page to build an environment that will support Hugo rather than the default GitHub pages service used Jekyll. The lines that run npm commands are additional, although necessary for some themes in order to run specific cross site scripts.

The github_token: ${{ secrets.SECRET_HUGO }} will be set under the Secrets -> Actions tab

## Change Repository Settings

Go to the settings in the repository

1. Create a new branch - I use `gh-pages`
2. Under Pages tab
    - Under Build and Deployment -> Change source to Deploy from a branch and Branch to `gh-pages`
    - (Optional) Change custom domain name to your purchased domain name
    - ![Pages Settings](examples/gh-pagesexample1.png "GH-Page Example")

###  Custom Domain Name (Optional)
1. Setting up DNS for Domain to allow GitHub Pages
 - If using CloudFlare, login to your dashboard and select the Domain Name to use.
    - Go to DNS
    - Create an A record for each of the following IP address pointing back at your `example.com` domain:
    ``` IPv4s
    185.199.108.153
    185.199.109.153
    185.199.110.153
    185.199.111.153
    ```
    - ![Pages Settings](examples/gh-pagesexample2.png "GH-Page Example")

    - Do the same but for IPv6 by Creating AAAA records for the following:
    ``` IPv6s
    2606:50c0:8000::153
    2606:50c0:8001::153
    2606:50c0:8002::153
    2606:50c0:8003::153
    ```

2. Modify config.toml file (Generated later with Hugo Setup)
- Once you create a Hugo server, it will generate a plethora of files. One of these is a config file. Edit it and change the `baseurl` varaible to `https://example.com/`

3. Create Static site name file
- Again, once the Hugo site is created. There is a static folder. Navigate into it, create a file named CNAME. Add your Domain name to it and save it.

## Setup Hugo
I recommend to clone down the repository created earlier, and generate the Hugo site into that folder structure. This will make commiting changes easier, and much more straightforward for this section. 

Setting up Hugo (There are a few extra steps compared to normal, due to the Theme I will be demonstrating requiring POSTCSS)

*This example will cover conducting the setup in Linux*

1. Commands to Setup npm versioning (Should be V.14) - *Necessary for PostCSS*
   1.  ``` bash
       sudo apt update
       sudo apt install nodejs npm
       curl -sL https://deb.nodesource.com/setup_14.x | sudo -E bash -
       sudo apt install nodejs
       ```
2. Setup Hugo 
   1. ``` bash
      wget https://github.com/gohugoio/hugo/releases/download/v0.98.0/hugo_extended_0.98.0_Linux-64bit.deb
      dpkg --install hugo_extended_0.98.0_Linux-64bit.deb
    
3. Build site using Blist
   1. ``` bash
      hugo new site blist # Can rename blist as needed
      cd blist 
      git clone https://github.com/apvarun/blist-hugo-theme.git themes/blist
      cp themes/blist/package.json ./package.json
      cp themes/blist/package-lock.json ./package-lock.json
      npm install
      npm i -g postcss-cli # Necessary for PostCSS
      echo "theme = \"blist\"" >> config.toml # Rename blist if you did so earlier
      hugo serve # Starts the Hugo server based on Static information within directory ran
      ```  


## Picking a Theme
If you wish to use another theme, I recommend searching online for freely available themes. Generally there isn't too much change for setup in variation between theme to theme.

## Commit
Now everytime you commit into the repository, it should update GH-Pages and populate the site! 

## Useful Resources
* [Hugo](https://gohugo.io/)
* [Hugo Themes](https://hugothemesfree.com/)