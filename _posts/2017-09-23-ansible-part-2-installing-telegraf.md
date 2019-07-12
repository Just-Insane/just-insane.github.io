---
title: 'Ansible Part 2: Installing Telegraf'
date: '23-09-2017 00:00'
categories:
  - 'Blog'
  - 'Configuration Management'
tag:
  - 'Ansible'
  - 'Dev'
---

In part two of this series, we will look at creating a playbook to deploy and configure Telegraf, the popular devops service for getting your system's metrics into a time series database, for use with tools like Grafana.

As always, a link to the completed playbook can be found ~~[here](https://git.justin-tech.com/snippets/13)~~.

```yaml
---
- hosts: all

  tasks:
  - name: Add telegraf repository
    yum_repository:
      name: influxdb
      description: InfluxDB Repository - RHEL $releasever
      baseurl: https://repos.influxdata.com/rhel/$releasever/$basearch/stable
      gpgcheck: yes
      gpgkey: https://repos.influxdata.com/influxdb.key

  - name: Install telegraf
    yum:
      name: telegraf
      state: latest

  - name: Copy config
    copy:
      src: ./telegraf.conf
      dest: /etc/telegraf/telegraf.conf
      owner: root
      group: root
      mode: 0644
      backup: yes
      force: no
      notify:
      - restart telegraf

  handlers:
  - name: restart telegraf
    service:
      name: telegraf
      state: restarted
```

This playbook is a bit more advanced then the last one we looked at, however, it gives you a good overview on the structure of a simple playbook, and how to define multiple tasks.

Let's break it down below.

! Note that I won't explain parts that are the same across this playbook and the previous, which can be found [here](https://homelab.blog/ansible-part-1-linux-host-updates).

First, we can see that there are three tasks, and a handler defined. Handlers are a way to tell Ansible to do `<something>`, but only when a task actually changes or updates something. In this playbook, the handler tells Ansible to restart telegraf if the final task in the playbook makes a change, otherwise telegraf won't be restarted.

Second, let's look at the first task. Called "Add telegraf repository", in this task, we will find a new component of Ansible, called "yum_repository", which will add a Yum repository to our system. Below that, we will see the vairables that are used for the repository. These are what gets added to the .repo file, and are based on the repository we are trying to add. In this file, you will see the information needed to install Telegraf.

Third, in the second task, you will see that we are again using the yum module to install a system service. For more information, please see the previous blog post in this series.

Fourth, we will find a task called "Copy config", and as it sounds, it will copy a pre-made telegraf.conf file into the correct location for use by telegraf. Let's take a further look at it.

This task is also using a new module called copy, and below that, we can see some of the available variables for the module. The first five lines are pretty basic, they define the source location for the file, the destination on the remote system, as well as the owner and permissons on the file. The next three lines are a bit more advanced.

`backup` tells Ansible to "Create a backup file including the timestamp information so you can get the original file back if you somehow clobbered it incorrectly", this is very important if you are overwriting configuration files for programs, in the case that something breaks, you can always revert to the default configuration file if one exists.

`force: no` tells Ansible to only copy the file to the remote system if it does not already exist. "The default is yes, which will replace the remote file when contents are different than the source. If no, the file will only be transferred if the destination does not exist." In my case, this file should not be changing once it's on the remote system, setting force to no saves me a couple of seconds off the total run time of the playbook, and if I do ever change the conf file, I can easily change or remove the force variable from the playbook.

`notify` This is an interesting argument, but one that is very important to creating a strong playbook in Ansible. As I mentioned previously, the notify command can be used to call handlers. Handlers can do many things, but in this playbook, we only use them to programatically tell Ansible if we want to restart the telegraf service. If you want to learn more about Handlers, please see the Ansible documentation [here](http://docs.ansible.com/ansible/latest/playbooks_intro.html#handlers-running-operations-on-change).

Now we will look at the handlers section of the playbook. As you can see, the handler's section looks very similar to a task, and that's because it is. They are tasks that only run when you call them when something is changed. In our example, it doesn't make sense to restart telegraf every single time the playbook is run, as this leads to downtime in our metrics, and that is generally not a good thing, and leads to a lot of alerts being sent out (you have alerts setup, right?).

This completes our post on using Ansible to setup Telegraf, as well as looking at a slightly more advanced playbook then we did previously.

P.S. If you want to see what you can do with Grafana and system metrics, try pressing the "Dashboard" button in the top left of the page, or going to ~~[https://dashboard.justin-tech.com](https://dashboard.justin-tech.com)~~
