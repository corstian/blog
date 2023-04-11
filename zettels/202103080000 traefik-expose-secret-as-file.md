
---
title: "Expose a secret as file in Traefik via Helm to configure ACME DNS"
slug: "traefik-expose-secret-as-file"
date: "2021-03-08"
summary: ""
---

#software-development #devops

As I have only been working with Kubernetes for a few days, most things just like Helm are pretty new to me, and as such I get lost a lot. One of the things I had to find out the hard is the way a Helm chart and it's associated configuration options are resolved.

I won't go into the details, but instead provide a link to [the "Chart Template Guide" provided by Helm](https://helm.sh/docs/chart_template_guide/getting_started/). Diving into this, as well as the offending template I managed to solve my problems.

The problem I was trying to solve had to do with configuring an ACME DNS resolver on TransIP. The validation process itself happens via a library called Lego, and [the specific resolver for TransIP](https://go-acme.github.io/lego/dns/transip/) requires providing a key file.

The thing is, the Traefik Helm chart contains a "volumes" section, even though it does not exactly work like the volume configuration as known from Kubernetes itself. Instead I had to dig through [the chart template](https://github.com/traefik/traefik-helm-chart/blob/master/traefik/templates/_podtemplate.tpl) to figure out how it worked. In the end it's fairly simple;

```
volumes:
- name: name-of-your-secret
  type: secret
  mountPath: /etc/subdir-youd-like-to-use
```

This way you can access the elements of your secret within the `/etc/subdir-youd-like-to-use` directory. I had two elements: `user` and `key`, and as such those would be the text files in that aforementioned folder.

