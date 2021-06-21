---
title: Platform setup
---

# Platform setup

The project contains contains components to bootstrap, configure and maintain a remote
deployment of an OGC API web-service stack using modern "DevOps" tooling.

## design principles

The main design principles are:

* any action on the server/VM host is performed from a client host
* i.e. no direct access/login to/on the server/VM, only maybe for problem solving
* remote actions can be performed manually or triggered by GitHub Workflows
* all credentials (passwords, SSH-keys, etc) are secured 
* operational stack instances for "production" (stable) and "sandbox" (playground)

## Components

The components used at the lowest level are:

* [Docker](https://www.docker.com/) *"...OS-level virtualization to deliver software in packages called containers..."* ([Wikipedia](https://en.wikipedia.org/wiki/Docker_(software)))
* [Docker Compose](https://docs.docker.com/compose) *"...a tool for defining and running multi-container Docker applications..."*
* [Ansible](https://www.ansible.com/) *"...an open-source software provisioning tool"* ([Wikipedia](https://en.wikipedia.org/wiki/Ansible_(software)))
* [GitHub Actions/Workflows](https://docs.github.com/en/actions) *"...Automate, customize, and execute software development workflows in a GitHub repository..."*

The Docker-components are used to run the operational stack, i.e. the OGC API web-services. Ansible is used to provision both the server OS-software
and the operational stack. Ansible is executed on a local client/desktop system to invoke operations on a remote server/VM.
These operations are bundled in so called Ansible Playbooks, YAML files that describe a desired server state.
GitHub Actions are used to construct Workflows. These Actions will invoke these Ansible Playbooks, effectively configuring
and provisioning the operational stack on a remote server/VM. 
                    
Security is guaranteed by the use of [Ansible-Vault](https://docs.ansible.com/ansible/latest/user_guide/vault.html) 
and [GitHub Encrypted Secrets](https://docs.github.com/en/actions/reference/encrypted-secrets).

The operational stack is composed with the following components:

* [Traefik](https://traefik.io/) a frontend proxy/load-balancer and SSL (HTTPS) endpoint.
* [mkdocs](https://www.mkdocs.org/) for live documentation
* [GeoHealthCheck]() is a component to monitor the availability and complience of the implementations 
* [portainer]() provides a mechanism to monitor running containers
* [PostGreSQL / pgadmin]()

## Production and Sandbox Instance

Two separate server/CM-instances are managed to provide stable/production and 
sandbox/playground environments. As to control changes these instances are mapped to two GitHub branches:

* `main` for the stable/production instance
* `sandbox` for the playground

[GitHub Protected Branches](https://docs.github.com/en/github/administering-a-repository/defining-the-mergeability-of-pull-requests/about-protected-branches) are
used to provide for selective access and deployment.


## Selective Redeploy
When changes are pushed to this repo only the affected services are redeployed.
This is effected by a combination of GitHub Actions and Ansible Playbooks as follows:

* each Service has a dedicated GitHub Action "deploy" file, e.g. [deploy.pygeoapi.yml](.github/workflows/deploy.pygeoapi.yml)
* the GitHub Action "deploy" file contains a trigger for a `push` with a `paths` constraint, in this example:
```  
    on:
      push:
        paths:
          - 'services/pygeoapi/**'
```   
* the GH Action then calls the Ansible Playbook [deploy.yml](ansible/deploy.yml) with a `--tags` option related to the Service, e.g. `--tags pygeoapi`
* the [deploy.yml](ansible/deploy.yml) will always update the GH repo on the server VM via the `pre_tasks`
* the Ansible task indicated by the `tags` is then executed
       
TODO: this will be extended with 
[GitHub Protected Branches](https://docs.github.com/en/github/administering-a-repository/defining-the-mergeability-of-pull-requests/about-protected-branches)
to map the branch pushed to a server (host) instance via Ansible Inventory settings.

## Steps and Workflows

These can be used to setup a running server from zero.

### Step 0 - Obtain access to server/VM
This implies acquiring a server/VM instance from a hosting provider.
Main requirements are that server/VM runs an LTS Ubuntu (20.4 or better) and that SSL-keys are available for root access 
(or an admin user account with sudo-rights).

### Step 1 - Clone this repo

`git clone https://github.com/Geonovum/ogc-api-testbed.git`.

### Step 2 - Bootstrap the server/VM
"Bootstrap" here implies the complete provisioning of a remote server/VM that runs the operational service stack.
This is a one-time manual action, but can be executed at any time as Ansible actions are idempotent.
By its nature, Ansible tasks will only change the system if there is something to do.

Startpoint is a fresh Ubuntu-server or VM with root access via SSH-keys (no passwords).
The Ansible playbook [bootstrap.yml](ansible/bootstrap.yml) installs the neccessary software, and hardens
the server security, e.g. using [fail2ban](https://www.fail2ban.org/).
In this step Docker and Docker Compose are installed and a Linux [systemd](https://en.wikipedia.org/wiki/Systemd) service is run
that automatically starts/stops the operational stack, also on reboots.
The software for the operational stack, i.e. from this repo, is cloned on the server as well.

### Step 3 - Maintain the server/VM
This step is the daily operational maintenance. 
The basic substeps are:

* make a change, e.g. add a data Collection to an OGC API OAFeat service
* commit/push the change to GitHub
* watch the triggered GitHub Actions, check for any errors
* observe changes via website