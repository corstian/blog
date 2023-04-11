---
title: "How to set up Traefik on Kubernetes?"
slug: "configuring-traefik-on-kubernetes"
date: "2021-03-17"
summary: "Recently I started playing around with Traefik on Kubernetes and wanted to request wildcard LetsEncrypt certificates."
---

#software-development #devops

Recently I started playing around with Traefik on Kubernetes. Though I started my cluster with Nginx as load-balancer handling Kubernetes' ingresses, I quickly switched this one out with Traefik as I have a need for wildcard LetsEncrypt certificates. Requesting those with cert-manager is more difficult, and given Traefik comes with a long list of supported vendors for DNS validation, it was a fairly easy choice to go with them.

The switch from Nginx, with which I've been working for several years to now, to Traefik, from which I've only heard, has been fairly big. Combined with Kubernetes with which I'm totally unfamiliar, and it's been an incredibly steep learning curve over the past few days.


## Covered topics
There have been a few topics with which I have hit a few bumps in the road. Those were:

1. [Traefik configuration using Helm](#one)  
   1.1 [Persistence](#oneOne)  
   1.2 [Configuring an LetsEncrypt account](#oneTwo)  
   1.3 [Adding environment variables for DNS validation](#oneThree)  
   1.4 [Configuring TLS for the HTTPS endpoints](#oneFour)  
2. [Configuring an Ingress](#two)  
3. [Resources](#three)  

## <a id="one" href="#one">1.</a> Traefik configuration using Helm

First off, Helm is amazing to bootstrap multiple complex software packages in a relatively short time. As such I can get ElasticSearch, RabbitMQ, and SQL running within an incredibly short timeframe. Part of the power therein lies in a single way of configuring packages, and that exactly is what is so confusing about installing Traefik using Helm. As Traefik is built for use on multiple platforms, and natively uses YAML and TOML files for configuration. Practically this means you'll end up wondering about which conventions to use for configuration. Is it docker-compose, Kubernetes, Traefik, or maybe the Traefik Helm chart? There are subtle differences therein which are difficult to work with at first.

If you've installed Traefik using Helm, it's important to realize that the configuration options as represented in the Helm chart do not 1:1 represent those as shown within the yaml examples as available in the Traefik documentation. Instead, [the Helm chart's values](https://github.com/traefik/traefik-helm-chart/blob/master/traefik/values.yaml) are translated into several different Kubernetes resources.

As such;

- The Helm charts' values will be translated into a bunch of flags and resource descriptions, but not into config files.
- Some flags may be overwritten by the values as used by the Helm chart, or vice versa. Not sure about that.
- Traefik has Kubernetes CRD's for most operations. Though it's possible to configure middlewares in a config file, I do not really see why one would like to do so when using Traefik with Kubernetes.

Regarding my Helm configuration;

### <a id="oneOne" href="#oneOne">1.1</a> Persistence
Really the first thing I enabled just to ensure I do not hit the rate limits as imposed by LetsEncrypt. During installation of the Helm chart you can uncomment the following section. Ensure `enabled` is true.

```
persistence:
  enabled: true
  name: data
  accessMode: ReadWriteOnce
  size: 128Mi
  storageClass: ""
  path: /data
  annotations: {}
```

As part of this, ensure the `additionalArguments` list will have an entry like this;

```
- "--certificatesresolvers.le.acme.storage=/data/acme.json"
```

> Make sure that the certificate resolver (in this case `le`) has the same identifier as the information we're going to enter next.

**Resolving a permissions issue**
It's possible on several Kubernetes platforms (I'm using OVH, where I have this issue), that the certificate files cannot be accessed due to a file permission issue. The Traefik pod will log this as an error. To resolve this issue you can add the following to your configuration:

```
initContainers: []
  # The "volume-permissions" init container is required if you run into permission issues.
  # Related issue: https://github.com/traefik/traefik/issues/6972
  - name: volume-permissions
    image: busybox:1.31.1
    command: ["sh", "-c", "chmod -Rv 600 /data/*"]
    volumeMounts:
      - name: data
        mountPath: /data
```

[Find it here in the default Traefik Helm values.](https://github.com/traefik/traefik-helm-chart/blob/63181ebc50d4957ce113bcdf0d037ba3aeecd61c/traefik/values.yaml#L40)

### <a id="oneTwo" href="#oneTwo">1.2</a> Configuring an LetsEncrypt account
This account information is the only thing I have further configured within the `additionalArguments` list. 

```
- --certificatesresolvers.le.acme.dnschallenge.provider=transip
- --certificatesresolvers.le.acme.email=mail@example.com
- --certificatesresolvers.le.acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory
```

You can remove the `caserver` entry once everything works correctly. Only then you'll get certificates which can properly be validated by most devices.

### <a id="oneThree" href="#oneThree">1.3</a> Adding environment variables for DNS validation
As part of the account configured just now, there are some credentials which needs to be configured. [Here's a list of possible providers, and their required environment variables.](https://doc.traefik.io/traefik/https/acme/#providers). If you need to provide a file containing secrets, such as is the case with the `transip` provider, [check this post for more information about how to do so](/blog/2021-03-08/traefik-expose-secret-as-file).

These environment variables are configured as follows:

```
env:
- name: TRANSIP_ACCOUNT_NAME
  value: example-account-name
```

### <a id="oneFour" href="#oneFour">1.4</a> Configuring TLS for the HTTPS endpoints

The configuration beneath is fairly standard for a new Traefik installation.

```
ports:
  traefik:
    expose: false
    exposedPort: 9000
    port: 9000
    protocol: TCP
  web:
    expose: true
    exposedPort: 80
    port: 8000
    protocol: TCP
    redirectTo: websecure
  websecure:
    expose: true
    exposedPort: 443
    port: 8443
    protocol: TCP
    tls:
      certResolver: le
      domains:
      - main: example.com
        sans:
        - '*.example.com'
        - '*.beta.example.com'
      enabled: true
      options: ""
```

Some noteable things here:

**Redirection**

`ports.web.redirectTo: websecure` redirects traffic to the HTTPS section by default. Say goodbye to HTTP 🥳! This will translate to the following flags on the Traefik pod:

```
--entrypoints.web.http.redirections.entryPoint.to=:443
--entrypoints.web.http.redirections.entryPoint.scheme=https
```

Therefore, configuring those on the `additionalArguments` list will have the same effect.

**TLS Configuration**
The `ports.websecure.tls.certResolver` property will need the same identifier as the certificateResolver as configured before. In case of this post that is `le`. This certificate resolver will need to be a DNS resolver if you plan on requesting wildcard certificates as shown in the example.

> Remember that a wildcard certificate is only valid for the subdomains it is requested for. Therefore, `*.example.com` will work for `www.example.com`, but not for `example.com`.

Finally, be sure to enable it.

## <a id="two" href="#two">2.</a> Configuring an Ingress
As an Ingress is the basic unit which exposes Services from Kubernetes, I had been trying to configure wildcard certificates for those.

> **What NOT to do**
> When configuring tls with wildcard certificates, do not do so on an individual ingress, or in Traefik terms, routes. The following therefore DOES NOT WORK:
> 
> ```
> # traefik.ingress.kubernetes.io/router.tls: "true"
> # traefik.ingress.kubernetes.io/router.tls.certresolver: default
> ```
> 
> The result will be that individual certificates are requested for each domain you expose within your ingresses. In the example below it'd request two individual certificates.

As an example of something which might actually work, I'll show you this example;

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web,websecure
spec:
  rules:
  - host: api.beta.example.com
    http:
      paths:
        - path: /
          backend: 
            serviceName: example-api
            servicePort: 80
  - host: queue.beta.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: rabbitmq
          servicePort: 15672
```

As TLS is already configured on the gateway level within the Helm config, Traefik figure out which certificates to use by itself.

## <a id="three" href="#three">3.</a> Resources
- [**Lens**](https://k8slens.dev/): "The Kubernetes IDE" Required if you'd like to stay sane as a Kubernetes beginner.
- [**Install And Configure Traefik with Helm**](https://traefik.io/blog/install-and-configure-traefik-with-helm/): A post describing how to bootstrap a Traefik installation using Helm. Most notably it describes how to set up default http->https redirection, and configure LetsEncrypt wildcard certificates.
- [**Kubernetes Traefik Ingress Controller CRD**](https://github.com/sleighzy/k3s-traefik-v2-kubernetes-crd): Examples showing how to configure various aspects of Traefik using Kubernetes CRDs
- [**Treafik Helm Template**](https://github.com/traefik/traefik-helm-chart/tree/master/traefik/templates): These are the templates which resolve the Helm configuration into the resources as used by Kubernetes. Insightful if you're wondering how a certain configuration element is being resolved.
- [**Chart Templating Guide**](https://helm.sh/docs/chart_template_guide/getting_started/): If you're unsure how to interpret the template directives, check out this guide for more information.
- [**Default Helm values for Traefik**](https://github.com/traefik/traefik-helm-chart/blob/master/traefik/values.yaml): In case you're wondering about the defaults.

