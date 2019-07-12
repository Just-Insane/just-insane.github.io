---
title: 'Ansible Part 1: Linux host Updates'
date: '20-09-2017 00:00'
categories:
  - 'Blog'
  - 'Configuration Management'
tag:
  - 'Ansible'
  - 'Dev'
---

This Post will serve as an introduction to Ansible and give to you a taste of what it can do.

Ansible is a configuration management, application deployment, task automation, and IT orchestration tool. In short, it allows you to have a source of truth for your system's configuration, deploy system level and web level applications (from nginx, to your company's distributed high-availability web app), automate common, repetitive tasks, and do all of this in sequence to create a chain of events that can completely rebuild your infrastructure.

As an example, let's look at deploying a website, with database servers, application servers, web servers, and a load balancer. Normally, these services would each be configured by hand, one after another, and it could take anywhere from an hour to days to get everything sorted out and working correctly.

With Ansible, you can create a playbook (a set of tasks that you want done on your systems), which will automatically install your database servers, setup your web and application servers, and configure your load balancer, and it will do it the same way, every single time, not to mention that it is also faster than doing it by hand. On top of that, it can also ensure all of these services are running the correct version, not only of your site, but also of the programs you are using in the background to run your site.

Interested? Awesome! Let's look at installing Ansible and creating our first playbook.

## Step 1: Installation

Note: I am assuming you are using CentOS 7, if you are using another OS, you can find Ansible's installation documentation [here](http://docs.ansible.com/ansible/latest/intro_installation.html).

On your Ansible host (which can be a dedicated VM, or your laptop), you can either install via YUM (if you have the [Extras Channel](https://access.redhat.com/solutions/912213)), or build the RPM yourself, or use Python's pip package manager. I will show you how to install via pip.

Ensure you have python installed via your distributions guidelines, then do:

```sh
$ sudo pip install ansible
```

and you are all set to get started with Ansible.

## Step 2: Writing your first Playbook

Note: Ansible uses YAML for it's files, as such, white space is very important for syntax. Always uses spaces, not tabs, when working with Ansible.

Note: I keep my Ansible playbooks in Git, the corresponding repo for this post can be found ~~[here](https://git.justin-tech.com/ansible/yum_update)~~.

1. Add some hosts to your host file:

The global hosts file is located at `/etc/ansible/hosts`, open it with your favourite text editor and add an IP or hostname of a machine or machines you want to manage

2. Create a new file, generally in your home directory or similar. It can be called whatever you want, as long as the extension is .yml

The outline will be as follows:

```yaml
---
- hosts:

  tasks:
  - name:
```

Here is an example:

```yaml
---
- hosts: all

  tasks:
  - name: install updates
    yum: name=* state=latest
    become: yes
```

Let's break that down a bit.

First, the `---` is very important, as it needs to be the first line of every YAML page you create for use with Ansible.

Second, `- hosts: all`, this specifies that you want to run the playbook against every host in your Ansible hosts file. This can additionally be set when you run the playbook, and therefore is not manditory.

Third, `tasks:`, this is a heading, where you will put all the tasks you want done in the playbook. It is one of a few headings at this level, and Ansible uses these to know what it is supposed to be doing at any particular time while running the playbook.

Fourth, `- name: install updates`, this is the name of the task, it's also how you define multiple tasks, as each will have it's own name.

Fifth, `yum: name=* state=latest`, Ansible is able to use Yum and Apt package managers to install system level packages onto managed computers, this line will run yum, look for any packages installed on the system (`name=*`, where * is a wildcard), and ensure that the state is the latest. In other words, it ensures that every package on the system is on the latest version, the same as running yum update directly on the system.

Sixth, `become: yes`, tells Ansible that it needs to become an admin level user when it runs the task (same as when running yum or apt update).

Now that we have written our first playbook, it's time to run it!

## Running a playbook

Running a playbook against a remote system requires a few prerequisites, outlined below:

1. Must be able to login to the remote system using SSH (this is easiest to use SSH Keys)
2. Must have permissions to run as an admin, especially if installing system level packages. You need the same permissions as if you were running commands directly against the remote system

Once you meet these pre-requisites, you can run the following command:

```sh
$ ansible-playbook <name_of_your_file>.yml
```

Assuming you added hosts at the top of the file, you should see output showing that Ansible is collecting data from the remote computer(s), and then installing updates. After it goes through these steps, you will see a nice output showing an overview of any issues or changed remote computers. Note that in this case, changed means that one or more packages was updated. Error messages are fairly verbose by default, and if you run into any issues, it will tell you exactly what's wrong.
