---
title: 'Configuring Istio with OIDC authentication'
date: '03-02-2020 12:00'
published: true
categories:
  - 'Blog'
  - 'DevOps'
tag:
  - 'Kubernetes'
  - 'Homelab'
---

In this blog post, we will look at the first part of my ideal setup, which is to secure inbound communication via an authenticating reverse proxy (OAuth2_Proxy), and Keycloak.

This setup will use the follow technologies:

1. Istio (ingress gateway)
   1. Certmanager (certificates) - not covered in this post
2. OAuth2_Proxy (controls the OIDC flow)
   1. Redis (session storage)
3. Keycloak (OIDC Provider)

## Istio

[Istio](https://istio.io/) is a service mesh that allows you to define and secure services in your Kubernetes cluster. In my lab, I use it as the ingress gateway for my cluster, and I am planning on using it to secure service-to-service communication using mutual-tls.

Istio will require a valid certificate for the gateway, you can either set this up via cert-manager, or by importing a certificate into your cluster manually.

Here are some example config files for setting up basic services in Istio:

Istio Gateway:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - '*'
    port:
      name: http
      number: 80
      protocol: HTTP
    tls:
      httpsRedirect: true
  - hosts:
    - '*'
    port:
      name: https
      number: 443
      protocol: HTTPS
    tls:
      credentialName: changeme
      httpsRedirect: false
      mode: SIMPLE
      privateKey: sds
      serverCertificate: sds
```

Destination Rule:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: oauth-corp-justin-tech-com
  namespace: istio-system
spec:
  host: oauth.corp.justin-tech.com
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
    portLevelSettings:
    - port:
        number: 443
      tls:
        mode: SIMPLE
        sni: oauth.corp.justin-tech.com
```

Virtual Service:

oauthproxy-service:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: oauthproxy-service
  namespace: default
spec:
  gateways:
  - gateway.istio-system
  hosts:
  - oauth.corp.justin-tech.com
  - oauth.justin-tech.com
  http:
  - route:
    - destination:
        host: oauthproxy-service
        port:
          number: 4180
      weight: 100
```

the virtual service for your application, I am using Rancher-Demo:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: rancher-demo
  namespace: default
spec:
  gateways:
  - gateway.istio-system
  hosts:
  - rancher-demo.corp.justin-tech.com
  http:
  - route:
    - destination:
        host: rancher-demo
        port:
          number: 80
      weight: 100
```

The Envoy Filter, which is the part that starts the redirection for unauthenticated users. This is similar to the auth-url function of Nginx-Ingress. This acts on your Ingress-Gateway workload.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: authn-filter
spec:
  workloadLabels:
    istio: ingressgateway
  filters:
  - filterConfig:
      http_service:
        server_uri:
          uri: http://oauthproxy-service.default.svc.cluster.local
          cluster: outbound|4180||oauthproxy-service.default.svc.cluster.local
          timeout: 1.5s
        authorizationRequest:
          allowedHeaders:
            patterns:
            - exact: "cookie"
            - exact: "x-forwarded-access-token"
            - exact: "x-forwarded-user"
            - exact: "x-forwarded-email"
            - exact: "authorization"
            - exact: "x-forwarded-proto"
            - exact: "proxy-authorization"
            - exact: "user-agent"
            - exact: "x-forwarded-host"
            - exact: "from"
            - exact: "x-forwarded-for"
            - exact: "accept"
            - prefix: "x-forwarded"
            - prefix: "x-auth-request"
        authorizationResponse:
          allowedClientHeaders:
            patterns:
            - exact: "location"
            - exact: "proxy-authenticate"
            - exact: "set-cookie"
            - exact: "authorization"
            - exact: "www-authenticate"
            - prefix: "x-forwarded"
            - prefix: "x-auth-request"
          allowedUpstreamHeaders:
            patterns:
            - exact: "location"
            - exact: "proxy-authenticate"
            - exact: "set-cookie"
            - exact: "authorization"
            - exact: "www-authenticate"
            - prefix: "x-forwarded"
            - prefix: "x-auth-request"
    filterName: envoy.ext_authz
    filterType: HTTP
    listenerMatch:
      portNumber: 443
      listenerType: GATEWAY
```

The important parts are to set the server_uri. The allowed lists of headers is probably more than what is needed, but it works for me.

This final part is optional, if you omit this part, you would be able to use the standard OAuth2_Proxy setup which is to send the cookies to the client directly, instead of using Redis as a session store. This configuration uses Istio's JWT authentication validation to ensure that every request to your service is authenticated by your issuer. You could expand on this by requiring specific groups per service, and by doing client certificate validation (which you could also couple with Keycloak's client certificate validation), for the best authentication of incoming requests.

This is set per service with the target definition.

```yaml
apiVersion: authentication.istio.io/v1alpha1
kind: Policy
metadata:
  name: ingress-auth-policy
spec:
  targets:
  - name: rancher-demo
  origins:
  - jwt:
      issuer: "https://auth.justin-tech.com/auth/realms/Justin-Tech"
      jwksUri: "https://auth.justin-tech.com/auth/realms/Justin-Tech/protocol/openid-connect/certs"
  principalBinding: USE_ORIGIN
```

## Oauth2_Proxy

[OAuth2_Proxy](https://pusher.github.io/oauth2_proxy) performs the OIDC flow for unauthenticated users. We will need to use the Redis session store, as having the ID Token sent to the browser results in cookies that are too long, and splitting the cookie into parts does not work with Istio (see https://pusher.github.io/oauth2_proxy/configuration#configuring-for-use-with-the-nginx-auth_request-directive for an example that spits the cookies with Nginx Ingress support).

Oauth2_Proxy Deployment example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: oauth2-proxy
  name: oauth2-proxy
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: oauth2-proxy
  template:
    metadata:
      labels:
        k8s-app: oauth2-proxy
    spec:
      containers:
        - args:
            - --provider=oidc
            - --provider-display-name="Keycloak"
            - --email-domain=corp.justin-tech.com
            - --email-domain=justin-tech.com
            - --pass-access-token=true
            - --pass-authorization-header=true
            - --set-authorization-header=true
            - --oidc-issuer-url=https://auth.justin-tech.com/auth/realms/Justin-Tech
            - --login-url=https://auth.justin-tech.com/auth/realms/Justin-Tech/protocol/openid-connect/auth
            - --redeem-url=https://auth.justin-tech.com/auth/realms/Justin-Tech/protocol/openid-connect/token
            - --validate-url=https://auth.justin-tech.com/auth/realms/Justin-Tech/protocol/openid-connect/userinfo
            - --http-address=0.0.0.0:4180
            - --cookie-expire=4h0m0s
            - --whitelist-domain=".corp.justin-tech.com"
            - --cookie-domain=.justin-tech.com
            - --standard-logging=true
            - --auth-logging=true
            - --request-logging=true
            - --skip-provider-button=true
            - --upstream=static://
            - --redis-connection-url=redis://redis-master.default.svc.cluster.local:6379
            - --session-store-type=redis
          env:
            - name: OAUTH2_PROXY_CLIENT_ID
              value: rancher-demo
            - name: OAUTH2_PROXY_CLIENT_SECRET
              value:
            # docker run -ti --rm python:3-alpine python -c 'import secrets,base64; print(base64.b64encode(base64.b64encode(secrets.token_bytes(16))));'
            - name: OAUTH2_PROXY_COOKIE_SECRET
              value:
            - name: OAUTH2_PROXY_COOKIE_DOMAIN
              value: .justin-tech.com
          image: quay.io/pusher/oauth2_proxy:v4.1.0
          imagePullPolicy: Always
          name: oauth2-proxy
          ports:
            - containerPort: 4180
              protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: oauth2-proxy
  name: oauthproxy-service
  namespace: default
spec:
  ports:
    - name: http
      port: 4180
      protocol: TCP
      targetPort: 4180
  selector:
    k8s-app: oauth2-proxy
---
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: oauth2-proxy
  name: oauthproxy-service
  namespace: default
spec:
  ports:
    - name: http
      port: 4180
      protocol: TCP
      targetPort: 4180
  selector:
    k8s-app: oauth2-proxy
```

The above config is what I am using to deploy OAuth2_Proxy, some of the configuration is probably unnecessary. Some important parts are the URLs for your OIDC Provider (Keycloak in my case), and the cookie domain, if you have a domain and subdomains that are being used.

### Redis

I am using the stable/redis helm chart, with minimal configuration explained below. Redis is needed in order to pass JWT tokens from Keycloak to Istio, otherwise the cookies are too large and get split (which is not supported easily in Istio). The downside is that currently OAuth2_Proxy does not support a password on the Redis connection.

The standard values.yaml from redis is fine to use, though you can change a few options:

1. Enable persistence for the Redis master and slaves
2. Enable metrics (if you want)

## Keycloak

i will not cover Keycloak too much in this post, as I have other posts available which cover setting up Keycloak. One thing to note is that you need to set the Redirect URL to either a wildcard, or by setting up one OAuth2_Proxy service per service, or have your services available at different URIs instead of subdomains. This could be a security issue as any redirect url could be set, please consider this before implementing.

Altogether, this setup allows you to use Istio as your ingress-gateway, which provides a lot of features if you are using the rest of their service mesh, including mutual TLS inside the cluster, and integration with Kiali, for example. It allows you to use a service for starting/handling the OIDC flow (if you are using third party applications, or just want an authenticating proxy in front of your applications), and allows you to use any OIDC provider to do it.
