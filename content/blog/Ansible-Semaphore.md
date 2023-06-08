---
author: "Austin Barnes"
title: "Ansible Semaphore - Automating Updates"
description: "Using Ansible Semaphore, a free and open source UI for managing Ansible within my homelab"
tags: [
    "automation",
    "networking",
    "github",
    "ansible",
    "homelab",
]
date: 2023-06-08
thumbnail: /covers/Ansible-Semaphore.png
---
# Overview

In this blog post, I'd like to introduce you to Semaphore, an open-source Ansible management project. With Semaphore, you can automate and keep track of all your Ansible tasks in your homelab. Check out my current Ansible Playbooks in this [GitHub repository](https://github.com/Cinderblook/tacklebox/Network-Automation/Ansible/Playbooks).

## Prerequisites 

Before we dive into Semaphore, let's ensure you meet the following prerequisites:

1. Some prior experience with Ansible and understanding of playbook formatting.
2. Ideally, some Docker experience as we'll be using Docker-Compose for the setup.
3. Something in your homelab that you want to manage!

## Getting the Docker Container Setup

Setting up the Docker container for Semaphore is straightforward. You'll need to configure two primary files.

1. The first file is `docker-compose.yml`, which contains all the configuration for the container. You can find the file in this [repository](https://github.com/Cinderblook/tacklebox/tree/main/Docker/ansiblesemaphore).

```yml
services:
  mysql:
    restart: unless-stopped
    ports:
      - 3306:3306
    image: mysql:8.0
    hostname: mysql-semaphore
    volumes:
      - semaphore-mysql:/var/lib/mysql
    environment:
      MYSQL_RANDOM_ROOT_PASSWORD: 'yes'
      MYSQL_DATABASE: semaphore
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASS}
    networks:
      - proxy
  semaphore:
    restart: unless-stopped
    ports:
      - 3000:3000
    image: semaphoreui/semaphore:latest
    environment:
      SEMAPHORE_DB_USER: ${MYSQL_USER}
      SEMAPHORE_DB_PASS: ${MYSQL_PASS}
      SEMAPHORE_DB_HOST: mysql-semaphore
      SEMAPHORE_DB_PORT: 3306 
      SEMAPHORE_DB_DIALECT: mysql
      SEMAPHORE_DB: semaphore
      SEMAPHORE_PLAYBOOK_PATH: /tmp/semaphore/
      SEMAPHORE_ADMIN_PASSWORD: ${SEMA_ADMIN_PASS}
      SEMAPHORE_ADMIN_NAME: ${SEMA_ADMIN_USER}
      SEMAPHORE_ADMIN_EMAIL: ${SEMA_ADMIN_EMAIL}
      SEMAPHORE_ADMIN: ${SEMA_ADMIN_USER}
      SEMAPHORE_ACCESS_KEY_ENCRYPTION: ${SEMA_ACCESS_KEY} # Generate using command 'head -c32 /dev/urandom | base64'
      #SEMAPHORE_LDAP_ACTIVATED: 'no' 
      #SEMAPHORE_LDAP_HOST: dc01.local.example.com
      #SEMAPHORE_LDAP_PORT: '636'
      #SEMAPHORE_LDAP_NEEDTLS: 'yes'
      #SEMAPHORE_LDAP_DN_BIND: 'uid=bind_user,cn=users,cn=accounts,dc=local,dc=shiftsystems,dc=net'
      #SEMAPHORE_LDAP_PASSWORD: 'ldap_bind_account_password'
      #SEMAPHORE_LDAP_DN_SEARCH: 'dc=local,dc=example,dc=com'
      #SEMAPHORE_LDAP_SEARCH_FILTER: "(\u0026(uid=%s)(memberOf=cn=ipausers,cn=groups,cn=accounts,dc=local,dc=example,dc=com))"
    depends_on:
      - mysql 
    networks:
      - proxy
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.semaphore.entrypoints=http"
      - "traefik.http.routers.semaphore.rule=Host(`${DNS_HOSTNAME_CLIENT}`)"
      - "traefik.http.middlewares.semaphore-https-redirect.redirectscheme.scheme=https"
      - "traefik.http.routers.semaphore.middlewares=semaphore-https-redirect"
      - "traefik.http.routers.semaphore-secure.entrypoints=https"
      - "traefik.http.routers.semaphore-secure.rule=Host(`${DNS_HOSTNAME_CLIENT}`)"
      - "traefik.http.routers.semaphore-secure.tls=true"
      - "traefik.http.routers.semaphore-secure.service=semaphore"
      - "traefik.http.services.semaphore.loadbalancer.server.port=3000"
      - "traefik.docker.network=proxy"

networks:
  proxy:
    external: true

volumes:
  semaphore-mysql: 
```

*If you do not use Traefik for proxies, on the same host, go ahead and remove / comment out that part of the configuration).*

2. The second file is a `.env` file that stores the credentials required for the Docker-Compose file. Fill in your valuable super-secret credentials in this file before spinning up the container.

```env
MYSQL_PASS = 
MYSQL_USER = 
SEMA_ACCESS_KEY = 
SEMA_ADMIN_EMAIL = 
SEMA_ADMIN_PASS = 
SEMA_ADMIN_USER = 
DNS_HOSTNAME_CLIENT =
```

## Semaphore Configuration to Get Started

Before you can start deploying Ansible tasks with Semaphore, there are a few necessary setup steps. In this tutorial, I will demonstrate the setup using my [GitHub repository](https://github.com/Cinderblook/tacklebox).

1. Set up the 'Key Store' profiles:
   - Once you're logged into the Ansible Semaphore site, navigate to the 'Key Store' section.
   - Create three keys:
     - Key titled 'None' with Type 'None'. This will be used for GitHub repository access (since it's public).
     - Key titled 'SSH-Key' with Type 'SSH Key'. This will be used for Ansible to run without sudo.
     - Key titled 'SSH-Pass' with Type 'Login with password'. This will be used for sudo-required Ansible tasks.

![semaphore-01](/examples/ansible-semaphore-01.png 'semaphore-01')

2. Set up the GitHub repository:
   - Use the URL of the repository.
   - Set the branch name (e.g., 'main').
   - If it's a public repository, set 'Access Key' to None. If it's an SSH private connection, create a key under 'Key Store' for that key and select it here.

![semaphore-02](/examples/ansible-semaphore-02.png 'semaphore-02') 

3. Configure the inventory:
   - Under 'Inventory', create a 'New Inventory'.
   - Provide a name and set the credentials to be used for non-sudo and sudo tasks (created earlier).
   - For the Type, select File if you have a local file containing the inventory, or use Static if you want to manage the inventory within Semaphore.

![semaphore-03](/examples/ansible-semaphore-03.png 'semaphore-03') 

4. Create an environment file:
   - Under 'Environment', create a 'New Environment' named 'default'.
   - Leave the extra variables section as '{}'.

![semaphore-04](/examples/ansible-semaphore-04.png 'semaphore-04')  

Now that we have the initial required configuration done, let's set up our playbooks.

## Semaphore Task Templates / Playbooks

Navigate to the 'Task Templates' tab, and we will set up our first Task playbook.

1. Create a 'New Template' and provide the following parameters. Leave the Template as a 'Task':
   - Name: `<name>`
   - Description: `<description>`
   - Playbook Filename: `<file/location/within/repo>`
   - Inventory: `<created-inv>`
   - Repository: `<created-repo>`
   - Environment: `<created-env>`

![semaphore-05](/examples/ansible-semaphore-05.png 'semaphore-05')  

2. Now that the playbook is created, navigate into it, and hit 'Run'
    - You can use different features while executing the task typically foundational of Ansible such as, 'Debug','Dry Run', 'Diff', etc.

![semaphore-06](/examples/ansible-semaphore-06.png 'semaphore-06')  

From here, you're all set! Start automating your Ansible tasks with Semaphore.

## Useful Resources

Here are some useful resources for further exploration:

* [Ansible Semaphore Site](https://www.ansible-semaphore.com/)
* [Ansible Documentation](https://docs.ansible.com/)
* [Docker Documentation](https://docs.docker.com/compose/)
* [My GitHub Repository](https://github.com/Cinderblook/tacklebox/tree/)
