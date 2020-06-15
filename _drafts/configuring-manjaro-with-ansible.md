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
packages and settings present in one or more machines. A common application is
for system administrators to manage a cluster of servers from a center control
server.

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

Ansible uses the "push" model of configuring management, meaning that a central
server pushes configuration settings to various clients. It is also agentless,
which means that Ansible code runs on the server only, not on the clients. The
server sends all necessary commands over the network via SSH. This is convenient
since client machines simply need a network connection to start being
configured.

Our goal here is to orchestrate the installation of a Linux environment on a
single machine, so the client and server exist on the same machine. This is a
perfectly valid architecture for Ansible to operate on. While there are a number
of ways to specify the local machine as the Ansible host, I used the snippet
below:

{% highlight yaml %}
---
- name: <play_name>
  hosts: localhost
  connection: local
  tasks:
    ...
{% endhighlight %}

### Playbooks

The functionality of Ansible is realized through the use of playbooks. Each
playbook is a YAML file which coordinates a set of actions. Since playbooks can
effectively call other playbooks like they are functions, I divided up the
installation task among multiple playbooks, with one main playbook that calls
all the rest.

### Installing Packages

For my systems, there are types of package to install: Linux packages, Atom text
editor packages, and Ruby gems. Thankfully, Ansible has modules for all three
of these packaging systems. The specific modules are `pacman` (since this is
the package manager Manjaro uses), `apm` the Atom package manager, and `gem`.

Each of these package managers is executed in its own playbook and references
its own list of packages. The abstraction of each package manager via an Ansible
module allows one to easily iterate over lists in this manner.

<span style="background-color: #FFFF00">Mention differences for the different
package managers.</span>

### Installing Config Files



### User and SSH Settings

My Github repo featuring all of this Ansible functionality is located
[here](https://github.com/jdtaylor7/workflow), under the `ansible` directory.
