---
title: "Installation"
description: "Get Estafette up and running on your Kubernetes cluster"
date: 2018-06-19T12:51:31+02:00
menu:
  main:
    parent: 'docs'
---

### Installing Estafette CI

Prerequisites:

- install https://github.com/estafette/estafette-cloudflare-dns
- install https://github.com/estafette/estafette-letsencrypt-certificate
- install https://github.com/estafette/k8s-cockroachdb

Configure cluster roles and binding:

```
curl https://raw.githubusercontent.com/estafette/estafette-ci-api/master/rbac.yaml -o rbac.yaml

export NAMESPACE=estafette
export APP_NAME=estafette-ci-api

kubectl apply -f rbac.yaml
```

Install the API:

```
curl https://raw.githubusercontent.com/estafette/estafette-ci-api/master/kubernetes.yaml -o kubernetes.yaml

export NAMESPACE=estafette
export APP_NAME=estafette-ci-api
export TEAM_NAME=tooling-team
export HOSTNAMES=ci.estafette.io
export CLOUDFLARE_IP_RANGES=103.21.244.0/22, 103.22.200.0/22, 103.31.4.0/22, 104.16.0.0/12, 108.162.192.0/18, 131.0.72.0/22, 141.101.64.0/18, 162.158.0.0/15, 172.64.0.0/13, 173.245.48.0/20, 188.114.96.0/20, 190.93.240.0/20, 197.234.240.0/22, 198.41.128.0/17
export GITHUB_APP_PRIVATE_KEY=
export VERSION=123456
export GO_PIPELINE_LABEL=latest
export CPU_REQUEST=10m
export MEMORY_REQUEST=15Mi
export CPU_LIMIT=200m
export MEMORY_LIMIT=512Mi
export GITHUB_APP_ID=1234
export GITHUB_APP_OAUTH_CLIENT_ID=ab3.myjk4935qfamx8sw
export GITHUB_APP_OAUTH_CLIENT_SECRET=***
export GITHUB_WEBHOOK_SECRET=***
export BITBUCKET_API_KEY=***
export BITBUCKET_APP_OAUTH_KEY=f7mFzNWdFbZ2m54aCb
export BITBUCKET_APP_OAUTH_SECRET=***
export ESTAFETTE_CI_API_KEY=***
export SLACK_APP_CLIENT_ID=1575687846.473375433834
export SLACK_APP_CLIENT_SECRET=***
export SLACK_APP_VERIFICATION_TOKEN=***
export SLACK_APP_OAUTH_ACCESS_TOKEN=***
export SECRET_DECRYPTION_KEY=***
export COCKROACH_DATABASE=estafette_ci_api
export COCKROACH_HOST=cockroachdb-public.estafette.svc.cluster.local
export COCKROACH_INSECURE=true
export COCKROACH_CERTS_DIR=
export COCKROACH_PORT=26257
export COCKROACH_USER=estafette_ci_api
export COCKROACH_PASSWORD=***
export ALLOW_CIDRS=allow 0.0.0.0/0;
export MIN_PODS=3
export MAX_PODS=10
export AUTOSCALE_CPU_TARGET_PERCENTAGE=80

cat kubernetes.yaml | envsubst | kubectl apply -f -
```