---
layout: post
title: "Configuring Manjaro with Ansible"
description: "Configuring Manjaro with Ansible"
tags: [devops, linux]
---

I have been an avid Linux user for the past five years and have run Arch Linux
installations alongside my Windows installations on most of my systems.
Recently, I've decided to try out Manjaro since it has most of the feature set
of Arch but with an extremely quick and convenient installation procedure.
Instead of just installing it manually like with Arch in the past, though, I
decided to apply some DevOps techniques to build up my installation.

### Configuration Management

In DevOps, configuration management is the concept of organizing the software
packages and settings present in one or more machines.

For my use case, I can use configuration management to install a complete list
of packages and install many "dotfiles", the configuration files which define
the behavior of bash, vim, atom, etc.

### Ansible

There are four configuration management tools which are the most popular right
now: Ansible, Chef, Puppet, and SaltStack. Since I have not dabbled with
configuration management before, I decided to use Ansible since it is quite
easy to set up and has a large following.

{% include image.html path="ansible-logo-wide.png"
path-detail="ansible-logo-wide.png" alt="Chalk intro" %}

Server/clients

Ansible uses the "push" model of configuring management, meaning that a central
server pushes configuration settings to various clients. It is also agentless,
which means that Ansible code is run on the clients. The server sends all
necessary commands over the network via SSH. This is convenient since client
machines simply need a network connection to start being configured.

### Playbooks

### Installing Packages

### Installing Config Files

### User and SSH Settings

My Github repo featuring all of this Ansible functionality is located
[here](https://github.com/jdtaylor7/workflow), under the `ansible` directory.
