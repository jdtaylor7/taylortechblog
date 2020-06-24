---
layout: post
title: "Configuring Manjaro with Ansible"
description: "Configuring Manjaro with Ansible"
tags: [devops, linux]
---

I have been an avid Linux user for the past five years and have run Arch Linux
installations alongside Windows installations on most of my personal systems.
Recently, I've decided to try out Manjaro since it has most of the feature set
of Arch but with an extremely quick and convenient installation procedure.
Instead of just installing it manually, I decided to apply some DevOps
techniques to build a complete installation procedure which is simple to use,
deploy, and update.

### Configuration Management

In DevOps, configuration management is the concept of organizing the software
packages and settings present in one or more machines. A common application is
for system administrators to manage a cluster of servers from a central control
server.

For my use case, I can employ configuration management to automatically install
lists of packages and configuration files for applications like Bash, Vim, Atom,
etc.

### Ansible

There are many configuration management tools out there right now, including
popular entries such as Ansible, Chef, Puppet, and SaltStack. Since I have not
dabbled with configuration management before, I decided to use Ansible because
it is quite easy to set up and has a large following.

{% include image.html path="ansible-logo-wide.png"
path-detail="ansible-logo-wide.png" alt="Chalk intro" %}

Ansible uses the "push" model of configuration management, meaning that a
central server pushes configuration settings to various clients. It is also
agentless, in that Ansible code runs on the server only, not on the clients. The
server sends all necessary commands over the network. This is convenient since
client machines simply need a network connection to start being configured.

Our goal here is to orchestrate the installation of a Linux environment on a
single machine which has just installed a fresh version of Manjaro, so the
client and server are the same machine. This is a simple and valid architecture
for Ansible to operate upon.

### Playbooks and Modules

The functionality of Ansible is realized through the use of "playbooks". Each
playbook is a [YAML](https://en.wikipedia.org/wiki/YAML) file which describes a
set of actions. Since playbooks can effectively call other playbooks like
functions, I split the installation process among multiple playbooks, with one
main file that calls the rest.

Within a playbook is a series of "plays", which are sets of instructions
executed with some environmental state. This state can include a list of
machines (on which the instructions are executed), variables, source files, etc.

Plays are further divided into "tasks", which are the individual instructions
themselves. In general, tasks utilize "modules" to actually define the job they
want to accomplish. Modules are abstractions of various software tools that
allow Ansible users to utilize such tools in a uniform way.

Here's a simple playbook:

{% highlight yaml linenos %}
{% raw %}
---
- name: Install config files
  hosts: localhost
  connection: local
  vars:
    - home_dir: "/home/user1"
  tasks:
    - name: "Install .bashrc"
      copy:
        src: ../config/.bashrc
        dest: "{{ home_dir }}"
    - name: "Install .vimrc"
      copy:
        src: ../config/.vimrc
        dest: "{{ home_dir }}"

{% endraw %}
{% endhighlight %}

Let's break down the structure of this playbook. First, the playbook file itself
begins with a line of three dashes. While not technically required here, this is
necessary when specifying YAML directives and in some other cases. Next, all the
code shown is part of a single play which is named "Install config files."

The "environmental state" of this play is all of the key-value pairs above the
"tasks" directive: the name of the play, the machine on which it will run, and a
variable accessible by each task in the play. As a side note, variables are
accessed by double bracing with quotes as shown on lines 11 and 15.

Lastly, the tasks themselves. They each have a name and a module, which fully
describes their operation. In this case, each task uses the `copy` module to
copy a single file to a destination directory. `src` and `dest` are parameters
consumed by the copy module which define where files/directories should be
copied to and from. Other parameters are available to the module and can be used
to achieve more complex results. The [Ansible documentation](
https://docs.ansible.com/) is excellent and describes all modules and their
parameters clearly.

### Installing Packages

For my system, there are three types of package to install: Linux packages, Atom
text editor packages, and Ruby gems. The package managers used for these package
types, pacman, apm, and gem, are each represented slightly differently in
Ansible. Both pacman and gem have an Ansible module abstraction (called `pacman`
and `gem`, respectively), while apm must be run as a shell command. Thankfully,
all three representations are straightforward to use. Let's look at pacman's
usage.

{% highlight yaml %}
{% raw %}
  tasks:
  - name: Install Linux packages with pacman
    pacman:
        name: "{{ pacman_packages }}"
        state: latest
{% endraw %}
{% endhighlight %}

In the above code snippet, we invoke the `pacman` module and simply hand it a
list of packages, named `pacman_packages`. We also specify that we want the
latest version of each package. `pacman_packages` is a YAML list variable
defined in a separate YAML file. This is the simplest module implementation for
a package manager because Ansible knows to loop over every item of the list
automatically.

Executing the other two package managers, as mentioned, works slightly
differently. The `gem` module does not support list inputs to the `name`
parameter. Instead, the list is looped over manually using the `with_items`
lookup plugin:

{% highlight yaml %}
{% raw %}
  - name: Install Ruby gems
    gem:
        name: "{{ item }}"
        state: latest
    with_items: "{{ ruby_gems }}"
{% endraw %}
{% endhighlight %}

The Atom package manager, apm, is not implemented as an Ansible module.
Therefore, it must be run as a shell command rather than a module.

{% highlight yaml %}
{% raw %}
  - name: Install Atom packages
    shell: apm install "{{ item }}"
    with_items: "{{ atom_packages }}"
{% endraw %}
{% endhighlight %}

### Installing Config Files

My setup includes configurations for Bash, Vim, etc. Copies of these config
files are maintained in the same repo as the Ansible code, so it is a simple
matter of copying these files into the user's home directory using the `copy`
module. The first Ansible code snippet earlier in this post shows an example of
using `copy` in this way.

### User and SSH Settings

User group settings, SSH keys, and Github SSH key integration can all be managed
through Ansible as well. While one manual step is required here, namely in
generating a [Github access token](
https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line)
and passing it to Ansible, this still requires less effort than uploading SSH
keys manually. These additional steps can be explained via command line print
statements by the Ansible playbook as it runs.

Editing the user setting is accomplished by using the `group`, `lineinfile`, and
`user` modules. These modules allow Ansible to add a "wheel" group, grant that
group sudo privileges by editing the sudo config file, and then add the current
user to that group. The wheel group is an administrative group often used to
grant sudo access.

Setting up Github SSH keys is performed via a HTTP POST method. This is the task
that requires the aforementioned Github access token, since Github has
disallowed authentication via username/password. The task is as follows:

{% highlight yaml %}
{% raw %}
  - name: Upload SSH keys to Github
    uri:
        url: https://api.github.com/user/keys
        user: "{{ github_username }}"
        password: "{{ github_token }}"
        method: POST
        body_format: json
        force_basic_auth: yes
        status_code: 201
        body:
            title: "{{ github_ssh_key_name }}"
            key: "{{ lookup('file', '{{ key_path }}.pub') }}"
{% endraw %}
{% endhighlight %}

All of the Ansible variables (`github_username`, `github_token`, etc.) are
collected from the command line when the Ansible script runs.

### Conclusion

While I only needed these features mentioned here to create my specific
installation procedure, Ansible supports many other tools and tricks. One can
manage databases, edit partitions, send notifications through applications like
Slack, and even run other configuration management tools such as Puppet.

The code featuring all of this Ansible functionality is located
[here](https://github.com/jdtaylor7/workflow), under the `ansible` directory.
