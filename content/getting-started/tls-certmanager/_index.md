---
title: "TLS with certmanager"
description: "Generate TLS certificates using certmanager"
weight: 2
---

For Estafette CI to work it needs to be secured using https, which requires a valid TLS certificate. The most popular way to handle this is by running [cert-manager](https://cert-manager.io/).

To install it in your cluster run the following commands:

```
helm repo add jetstack https://charts.jetstack.io
helm upgrade --install cert-manager jetstack/cert-manager -n cert-manager --create-namespace --set installCRDs=true
```

To configure an issuer create `issuer.yaml`:

```yaml
kind: Secret
apiVersion: v1
metadata:
  name: cloudflare-api-key-secret
  namespace: cert-manager
data:
  api-key: <base64 encoded cloudflare api key>
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
  namespace: cert-manager
spec:
  acme:
    # You must replace this email address with your own.
    # Let's Encrypt will use this to contact you about expiring
    # certificates, and issues related to your account.
    email: <an email address you own to use as a letsencrypt account>
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      # Secret resource that will be used to store the account's private key.
      name: letsencrypt-issuer-account-key
    # Add a single challenge solver, HTTP01 using nginx
    solvers:
    - dns01:
        cloudflare:
          email: <an email address for a cloudflare account to create an api key>
          # !! Remember to create a k8s secret before
          # kubectl create secret generic cloudflare-api-key-secret
          apiKeySecretRef:
            name: cloudflare-api-key-secret
            key: api-key
```

Make sure to set the _Cloudflare account email_, _Cloudflare api key_ and an email address to act as an identifier for a letsencrypt account.

```
kubectl apply -f issuer.yaml
```

To make use of this issuer to generate a tls secret used by the ingresses use the following `values.yaml`

```yaml
api:
  tls:
    enabled: true
    certManager:
      enabled: true
      issuer: letsencrypt-prod
```

and apply this with 

```
helm upgrade --install estafette-ci estafette/estafette-ci -n estafette-ci --create-namespace --timeout 600s --values values.yaml
```