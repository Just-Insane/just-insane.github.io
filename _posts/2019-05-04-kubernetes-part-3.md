---
title: 'Getting Started with Kubernetes (at home) — Part 3'
date: '04-05-2019 12:00'
categories:
  - 'Blog'
  - 'DevOps'
tag:
  - 'Kubernetes'
  - 'Homelab'
series: Getting Started with Kubernetes (at home)
---

In the first two parts of this series, we looked at setting up a production Kubernetes cluster in our labs. In part three of this series, we are going to deploy some services to our cluster such as [Guacamole](https://guacamole.apache.org/) and [Keycloak](https://www.keycloak.org/).

Step-by-step documentation and further service examples are [here](https://gitlab.com/just.insane/kubernetes/blob/master/docs/services/services.md).

## Guacamole

[Guacamole](https://guacamole.apache.org/) is a very useful piece of software that allows you to remotely connect to your devices via RDP, SSH, or other protocols. I use it extensively to access my lab resources, even when I am at home.

You can use this [Helm Chart](https://github.com/Just-Insane/apache-guacamole-helm-chart) to install Guacamole on your Kubernetes cluster. The steps are as follows:

1. Clone the Guacamole Helm Chart from [here](https://github.com/Just-Insane/apache-guacamole-helm-chart)
2. Apply any changes to [values.yaml](https://gitlab.com/just.insane/kubernetes/blob/master/src/services/guacamole/values.yaml) such as the annotations and ingress settings.
3. Deploy the Helm Chart from the `apache-guacamole-helm-chart` directory
    * `helm install . -f values.yaml --name=guacamole --namespace=guacamole`
4. An ingress is automatically created by the Helm Chart, and you can access it based on the `hosts:` section of [values.yaml](https://gitlab.com/just.insane/kubernetes/blob/master/src/services/guacamole/values.yaml)

Once you have deployed the Helm Chart, you should be able to access Guacamole at the ingress hostname specified in values.yaml.

## Keycloak

[Keycloak](https://www.keycloak.org/) is an open-source single sign-on solution that is similar to Microsoft's ADFS product. You can read more about Keycloak on my [blog](https://homelab.blog/blog/security/keycloak-part-1-what-is-sso/).

There is a stable Keycloak Helm Chart available in the default Helm repo, which we will be using to deploy Keycloak, you can find it [here](https://github.com/helm/charts/tree/master/stable/keycloak).

1. Apply any changes to [values.yaml](https://gitlab.com/just.insane/kubernetes/blob/master/src/services/keycloak/values.yaml)
2. Deploy the helm chart stable/keycloak with values
    * `helm install --name keycloak stable/keycloak --values values.yaml`
3. Create an [ingress](https://gitlab.com/just.insane/kubernetes/blob/master/src/services/keycloak/ingress.yaml)
    * `kubectl apply -f ingress.yaml`
4. Get the default password for the `keycloak` user.
    * `kubectl get secret --namespace default keycloak-http -o jsonpath="{.data.password}" | base64 --decode; echo`

Though this article is on the shorter side, hopefully it exemplifies how easy it can be to run services in Kubernetes. We mainly looked at pre-made Helm Charts in this article, however deploying a service without a Chart can also be just as easy. I prefer using charts as I find it easier to manage than straight Kubernetes manifest files.

You can checkout my public Kubernetes repo at [https://gitlab.com/just.insane/kubernetes/](https://gitlab.com/just.insane/kubernetes/) for further information and more service examples.