---
title: "Setting up a basic Kubernetes cluster"
slug: "setting-up-a-basic-kubernetes-cluster"
date: "2021-03-04"
summary: ""
references: 
  - [[202103170000 configuring-traefik-on-kubernetes]]
---

#software-development #devops

Kubernetes is a technology I have wanted to dive into for a long time, yet pushed off for over a year due to the complexity involved. There's so much to learn about that I did not feel like doing so, up until now. In this post I'll describe the steps I have taken to get something up and running, not so much as a complete guide to Kubernetes, but rather a quick list of resources with which you can get started.

## What I'm doing
At the moment I'm working to migrate services, databases and more from three different separate servers into a single cluster, primarily for maintenance. My stack consists of:

- ElasticSearch
- A SQL Database
- A bunch of files
- RabbitMQ
- Nginx
- A bunch of .NET services

I'm running this stack on OVH's managed Kubernetes platform, for brevity.

## Tools of choice
To get a better insight into what the cluster is doing I have been using [Lens](https://k8slens.dev/). Lens is an open source IDE with which Kubernetes clusters can be managed.

Additionally one needs the `kubectl` utility, which I got with Kubernetes support as shipped with Docker for Windows.

## Setting it up
With OVH, having the Kubernetes controller is free, and one only pays for the nodes and other resources utilized by the cluster.

The first step therein is to download the kubeconfig file from the OVH dashboard, and load it into Lens. This way Lens can connect with the cluster. In order to enable Lens to show all kinds of interesting metrics for the cluster, you'd need to install the Prometheus stack, for which Lens provides a convenient button.

## Adding the first nodes
Though we can already install Prometheus, we'd need a few nodes to run it on. Node pools can be added from the OVH control panel. I opted for three of the cheapest nodes which provides me with 6 cores, 21Gb of RAM and 150Gb of storage per node.

## Using Helm
Personally I'm not exactly sure what Helm is, but it seems like a package manager for Kubernetes. It works for me, and I'm grateful for that! I have installed the following tools using Helm:

- ElasticSearch
- RabbitMQ
- Some SQL server
- Nginx
- cert-manager

The actual process of doing so is not really interesting. Just hit the install button with the defaults and it kinda works.

## Configuring Nginx
The installation and configuration of Nginx is one of the most important aspects to my Kubernetes cluster. the `ingress-nginx` helm package allows me to use nginx as ingress to my cluster. In case you're not familiar with the things an ingress does, it is basically the interface between the scary online world to your cluster. For my application I want to focus on the following aspects:

1. How do I create an ingress?
2. How do I apply TLS encryption on connections?
3. How can I secure certain endpoints from prying eyes?

*(I applied above steps in reverse order to prevent resources from leaking to the web)*

During this step I figured out the beauty of Kubernetes, as I figured out it's basically a bunch of configuration files you can apply to your cluster to define how it works. That's in fact what the `kubectl` utility is about.

### Using basic auth to secure endpoints
The beauty in basic auth lies in its simplicity. Though utterly unsecure when used without encryption, it prevents each and every app from having to roll their own authentication mechanism. The documentation of `ingress-nginx` describes the application of basic auth way better than I'll probably will, so check that out here: [https://kubernetes.github.io/ingress-nginx/examples/auth/basic/](https://kubernetes.github.io/ingress-nginx/examples/auth/basic/).

### Applying TLS
To prevent our precious basic credentials from leaking, we'll have to add encryption. There's [this amazing tutorial on the DigitalOcean community](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nginx-ingress-with-cert-manager-on-digitalocean-kubernetes#step-3-%E2%80%94-creating-the-ingress-resource) which perfectly describes how to do so. Assuming you've already installed `ingress-nginx` using the Helm chart you can start at step 3 and continue from there.

The steps you'll execute are as follows:

1. Create an ingress
2. Install cert-manager
3. Request the required certificates

> Note: regarding the `apiVersion` used for the ClusterIssuer, one can use v1 now instead of v1alpha2 as is used in said tutorial.

That's all there is to do to set up the basic Kubernetes infrastructure one requires to run an application.

## Deploying the application
This is where I'm at right now. As I'm writing this I'm looking into ways to deploy my application (which contains a bunch of independent services) from GitHub via Docker Hub to my Kubernetes cluster. If interesting I'll cover that sometime in the future.

