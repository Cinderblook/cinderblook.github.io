# Personal Site
cinderblook.github.io & www.cinderblook.com

# How to install Hugo conditioned for POSTCSS
1. Commands to Setup npm versioning (Should be V.14)
   1.  ``` bash
       sudo apt update
       sudo apt install nodejs npm
       curl -sL https://deb.nodesource.com/setup_14.x | sudo -E bash -
       sudo apt install nodejs
       ```
2. Setup Hugo 
   1. ```
      wget https://github.com/gohugoio/hugo/releases/download/v0.98.0/hugo_extended_0.98.0_Linux-64bit.deb
      dpkg --install hugo_extended_0.98.0_Linux-64bit.deb
    
3. Build site using Blist
   1. ``` bash
      hugo new site test-blist
      cd test-blist
      git clone https://github.com/apvarun/blist-hugo-theme.git themes/blist
      cp themes/blist/package.json ./package.json
      cp themes/blist/package-lock.json ./package-lock.json
      npm install
      npm i -g postcss-cli
      echo "theme = \"blist\"" >> config.toml
      hugo serve
      ```