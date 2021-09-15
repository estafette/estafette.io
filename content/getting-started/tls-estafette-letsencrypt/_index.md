---
title: "TLS with estafette controller"
description: "Generate TLS certificates using estafette-letsencrypt-certificate"
weight: 3
---

For Estafette CI to work it needs to be secured using https, which requires a valid TLS certificate. This can be dealt with by using [estafette-letsencrypt-controller](https://github.com/estafette/estafette-letsencrypt-certificate).

To install it in your cluster run the following commands:

```
helm repo add estafette https://helm.estafette.io
helm upgrade --install estafette-letsencrypt-certificate estafette/estafette-letsencrypt-certificate -n estafette --create-namespace --set secret.cloudflareApiEmail=<cloudflare account email address> --set secret.cloudflareApiKey=<cloudflare api key>
```

To generate a tls secret used by the ingresses use the following `values.yaml`

```yaml
api:
  tls:
    enabled: true
    annotations:
      estafette.io/letsencrypt-certificate: true
      estafette.io/letsencrypt-certificate-hostnames: '{{ $.Values.baseHost }},{{ $.Values.integrationsHost }}'
```

and apply this with 

```
helm upgrade --install estafette-ci estafette/estafette-ci -n estafette-ci --create-namespace --timeout 600s --values values.yaml
```