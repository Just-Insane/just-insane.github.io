---
title: 'Getting Started with Kubernetes (at home) — Part 2'
date: '27-04-2019 00:00'
canonical_url: https://medium.com/@just_insane/getting-started-with-kubernetes-at-home-part-2-ce59289ca8ff
categories:
  - 'Blog'
  - 'DevOps'
tag:
  - 'Kubernetes'
  - 'Homelab'
series: Getting Started with Kubernetes (at home)
---

In the first part of the series, we looked at installing a bare bones Kubernetes cluster in some CentOS 7 VMs. In this part, we are going to look at setting up some back-end services, like a load balancer and ingress.

Some important things needed to properly run a Kubernetes cluster are a [storage class](https://kubernetes.io/docs/concepts/storage/storage-classes/) or manual storage configuration via [volumes](https://kubernetes.io/docs/concepts/storage/volumes/), a [load balancer](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer) though not strictly needed makes accessing services much easier, and an [ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/).

[Step-by-step documentation is here](https://gitlab.com/just.insane/kubernetes/blob/master/docs/configuration/configuration.md).

#### Setting up Helm

[Helm](https://helm.sh) is known as the “package manager for Kubernetes”, it is a tool that is used to template Kubernetes manifest files in a way that makes it easy to install new applications. I use Helm for all my service installation on Kubernetes due to it’s ease, and simplicity when getting started.

If you did not enable the optional Helm installation via Kubespray, we should do that first.

#### 1. Download Helm

If you are on a Mac, you can install Helm via brew install kubernetes-helm. The Helm documentation has instructions for other methods of installation (unpacking a .zip and adding it to the $PATH).

#### 2. Setup a Tiller user with RBAC

You can use this [rbac-config.yaml](https://gitlab.com/just.insane/kubernetes/blob/master/src/configuration/rbac-config.yaml) file to setup the user, download it and run the following commands:

`kubectl create -f rbac-config.yaml`

#### 3. Initialize Helm

Now we have to initialize Helm by deploying Tiller. The simplest method is to run `helm init --service-account tiller`. We have to specify the service account for tiller to use due to RBAC.

#### Persistent Storage

Since Docker, and by extension Kubernetes requires storage for any persistent data, it is important to provide Kubernetes some storage. If you were using a cloud provider, this would be handled for you, and you would use the providers default storage solution. In our case, we need to tell Kubernetes what storage to use.

We are going to use NFS storage for our cluster, which requires you to have an existing NFS server.

Once we have an NFS server, we are going to use the `nfs-client-provisioner`, which is available via Helm, to access that storage via Kubernetes [storage class](https://kubernetes.io/docs/concepts/storage/storage-classes/).

Simply run `helm install stable/nfs-client-provisioner -name nfs-client -namespace kube-system -set nfs.server=10.0.40.5 -set nfs.path=/mnt/Shared/kubernetes`

Let’s break that down a bit, the above command does the following:

1. Tell Helm to install the Helm Chart `stable/nfs-client-provisioner`
2. Give the release a name `nfs-client`
3. Install `nfs-client` to the `kube-system` namespace
4. Set the NFS Server IP to 10.0.40.5 you should change this as necessary
5. Set the NFS Path to /mnt/Shared/kubernetes you should change this to the share path on your NFS server

We should now check the status of nfs-client to ensure it is running correctly, via helm status nfs-client which should show ready.

#### Install MetalLB

[MetalLB](https://metallb.universe.tf/) is a [load balancer](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer) for Kubernetes that allows Kubernetes services to retrieve IP addresses when needed that are on the LAN or WAN. This greatly simplifies network access to services in a bare-metal (or VM) based Kubernetes cluster.

MetalLB has some requirements that we must comply with, namely, we have to have a Kubernetes cluster, a supported cluster network configuration, some IPv4 addresses, and optionally hardware that supports BGP. The first two are automatically handled by Kubespray in section 1, and for the IP addresses, you can provide a block that is outside your DHCP server’s scope on the same network as the Kubernetes nodes. I have not used the BGP option, as I do not have BGP hardware available.

You can install MetalLB via Helm with the following command `helm install -name metallb -f values.yaml stable/metallb` an example values.yaml file can be found [here](https://gitlab.com/just.insane/kubernetes/blob/master/src/configuration/metallb/values.yaml). Notice the `configInline` section that defines the IP addresses MetalLB will hand out.

#### Install Nginx-Ingress

[Nginx-Ingress](https://github.com/kubernetes/ingress-nginx) is an [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) service for Kubernetes. Ingresses serve to manage external access to services in the cluster. It is important to understand how nginx-ingress works, so I recommend you read through the [GitHub documentation](https://github.com/helm/charts/tree/master/stable/nginx-ingress).

Once you have done that, we can install Nginx-Ingress via Helm. You can review the [values.yaml](https://gitlab.com/just.insane/kubernetes/blob/master/src/configuration/nginx-ingress/values.yaml)as an example, the main thing to change being the `default-ssl-certificate` that Nginx should use.

Once you have done that, you can install it via `helm install -name nginx-ingress -f values.yaml stable/nginx-ingress -namespace nginx-ingress` which does the following.

1. Tell Helm to install the Helm Chart `stable/nginx-ingress`
2. With the release name `nginx-ingress`
3. To the `nginx-ingress` namespace
4. With the values from `values.yaml`

Since the type is set to loadBalancer, MetalLB will automatically provide an external IP to the service from the provided IP addresses, which you can point your DNS or Port Forwards to in order to access your cluster’s services.

#### Install Cert-Manager

[Cert-Manager](https://docs.cert-manager.io/en/latest/) is a complex service that allows us to automatically retrieve and renew SSL certificates from certificate providers. It works with Nginx-Ingress to ensure that all of our services are available via a secured connection over HTTPS.

There are a number of steps required to setup Cert-Manager correctly. You should start by reviewing the Helm Chart documentation [here](https://github.com/jetstack/cert-manager/tree/master/deploy/charts/cert-manager).

_Prerequisites_

1. Install the CustomResourceDefinition resources separately

- `kubectl apply -f [https://raw.githubusercontent.com/jetstack/cert-manager/release-0.7/deploy/manifests/00-crds.yaml](https://raw.githubusercontent.com/jetstack/cert-manager/release-0.7/deploy/manifests/00-crds.yaml)`

2. Create the namespace for cert-manager

- `kubectl create namespace cert-manager`

3. Label the cert-manager namespace to disable resource validation

- `kubectl label namespace cert-manager certmanager.k8s.io/disable-validation=true`

4. Add the Jetstack Helm repository

- `helm repo add jetstack [https://charts.jetstack.io](https://charts.jetstack.io)`

5. Update your local Helm chart repository cache

- `helm repo update`

_Installation via Helm_

1. Make any changes to [values.yaml](https://gitlab.com/just.insane/kubernetes/blob/master/src/configuration/cert-manager/values.yaml) notably the `podDnsConfig/nameservers`
2. Deploy the Helm Chart

- `helm install -name cert-manager -namespace cert-manager -f values.yaml jetstack/cert-manager`

_Verify the Installation_

This is covered in the official documentation which you can find [here](https://docs.cert-manager.io/en/latest/getting-started/install.html#verifying-the-installation). It has you create a test issuer and get a self-signed certificate to ensure everything is working properly.

_Create the Production Issuer_

**Note** : Creation of a test issuer is omitted for brevity, but please ensure you try creating a test issuer first so you do not hit the Let’s Encrypt rate limits.

If you are using Let’s Encrypt and Cloudflare, you can use this [letsencrypt.yaml](https://gitlab.com/just.insane/kubernetes/blob/master/src/configuration/cert-manager/letsencrypt.yaml) file. Remember to change the values as needed, especially anything marked with changeme.

Once you are ready, run `kubectl apply -f letsencrypt.yaml` to create the issuer.

_Create Cloudflare API Key Secret_

Now we need to add our Cloudflare API key into a Kubernetes secret. You can use [cloudflare-secret.yaml](https://gitlab.com/just.insane/kubernetes/blob/master/src/configuration/cert-manager/cloudflare-secret.yaml) as an example.

Once you have made your changes, create the secret with `kubectl apply -f cloudflare-secret.yaml`.

_Create the Default Certificate_

We can now create the default certificate that Nginx-Ingress will use for serving our sites. See [certificate.yaml](https://gitlab.com/just.insane/kubernetes/blob/master/src/configuration/cert-manager/certificate.yaml) for an example.

Once you have made your changes to the file, with regard to the names and domains, you can create the certificate with `kubectl apply -f certificate.yaml`.

#### Create an Ingress

Now we can create an ingress for the Kubernetes dashboard, so instead of going to [https://masternode:6443/proxy/](https://masternode:6443/proxy/) ect, we can go directly to [https://kubernetes.domain.com](https://kubernetes.domain.com)

The first step is to create the dashboard ingress, which we can do with [ingress.yaml](https://gitlab.com/just.insane/kubernetes/blob/master/src/configuration/dashboard/ingress.yaml) once you have made your changes, you can create the ingress with `kubectl apply -f ingress.yaml`

You can now go to [https://kubernetes.domain.com](https://kubernetes.domain.com) to access your Kubernetes dashboard, after you add the domain to DNS via the nginx-ingress external IP.

You now have a completely setup Kubernetes cluster running at home that you can test your services on, or use as a homelab.