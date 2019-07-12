---
title: 'Getting Started with Kubernetes (at home) - Part 1'
date: '27-04-2019 00:00'
canonical_url: https://medium.com/@just_insane/getting-started-with-kubernetes-at-home-part-1-5324b6654bb5
categories:
  - 'Blog'
  - 'DevOps'
tag:
  - 'Kubernetes'
  - 'Homelab'
series: Getting Started with Kubernetes (at home)
---

When you think about Kubernetes, you probably think AWS or GCP, a nice managed service where you can easily spin up resources and build applications on top of them. This is great, and honestly the best way to experience Kubernetes. However, if all you need is a lab to mess around in and experiment, or learn new things in, this can be very cost inefficient. That is why we are going to look at setting up Kubernetes ourselves.

In this post, we are going to look at the initial deployment of Kubernetes, from creating our nodes (in this case CentOS 7 VMs) to getting a cluster up and running.

## Part 1: Installing our VMs

The first step is to create some VMs. I use a custom vCenter template in my lab, but if you do not have one of those, you can follow these simple steps.

You will need to complete these steps on at least 1 machine, however more is certainly better to get the full benefit of Kubernetes.

Your machine/VM should have at least 1 core and 3Gb of RAM.

1. Install CentOS 7 from the USB ISO image, a basic install is fine
2. Create a user for Ansible access. This user should be part of the sudo users group, and ideally have passwordless SSH authentication
3. Assign static IP Addresses to your hosts. Optionally set a hostname.
4. I hate to say it, but the official docs say to disable the firewall between the nodes, and I was unable to find documentation on which ports are needed.
5. Optionally, add your hosts to DNS.

## Part 2: Kubernetes Installation

We are going to be using [Kubespray](https://github.com/kubernetes-sigs/kubespray) for our cluster, as it makes creating and updating a Kubernetes cluster very simple and straightforward.

You can find the official docs [here](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/getting-started.md).

The quick guide is as follows:

1. Install the dependencies

    `sudo pip install -r requirements.txt`

2. Copy the example Inventory

    `cp -rfp inventory/sample inventory/mycluster`

3. Build the inventory, you can use the built in builder, or take a look [here](https://gitlab.com/just.insane/kubernetes/blob/master/src/installation/hosts.ini) for an example. It is fine to have a single master, but the `kube-master` and `etcd` sections should be the same.

4. Deploy the cluster

    `ansible-playbook -i inventory/mycluster/hosts.yml --become --become-user=root cluster.yml`

Then all you have to do is wait while Kubespray deploys your cluster automatically. On my 6 node cluster, it usually takes about 10–15 minutes for the cluster to be completely setup and running.

Note that in the Kubespray inventory there are a couple of options which are useful to enable. First, in the `addons.yaml` file, it is a good idea to enable Helm and the Kubernetes Dashboard automatic deployments. It may also be beneficial to enable `kube_basic_auth` in the `k8s-cluster.yaml` file, if you are having issues with the default token based authentication. If you decide to do this later, you can simply make the change and then re-run the deployment with the command in step 4 above.

## Part 3: Testing the Cluster

You can test that your cluster is up and running with the following commands:

1. `kubectl cluster-info` which should return something like:

    `Kubernetes master is running at https://10.0.40.245:6444`

2. `kubectl get nodes` which displays the state of all of your nodes.

You can find more information about how I have setup Kubernetes at my [Gitlab repo](https://gitlab.com/just.insane/kubernetes), which has helpful code snippets, full configuration files, as well as expanded documentation.