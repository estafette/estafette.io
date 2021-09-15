---
title: Installation
description: Installing Estafette in your Kubernetes cluster
weight: 1
---

### Installing Estafette

_Estafette CI_ can easily be installed using [Helm](https://helm.sh/).


First add the `estafette` helm repository with

```
helm repo add estafette https://helm.estafette.io
```

Although Estafette aims to have as little configuration as possible by using sane defaults the Helm chart still needs a couple of values to be set. To do so create a `values.yaml` file with the following content:

```yaml
api:
  baseHost: '<(private) host for the web gui>'
  integrationsHost: '<public host to receive webhooks>'
```

Then install the `estafette-ci` chart with

```
helm upgrade --install estafette-ci estafette/estafette-ci -n estafette-ci --create-namespace --values values.yaml --timeout 600s
```

This should get all parts up and running, you can check with:

```
watch kubectl get svc,ing,deploy,sts,po -n estafette-ci
```

From here you need to set up either [Github login]({{< relref "github-login" >}}) or [Google login]({{< relref "google-login" >}}). And in order to receive webhooks for git pushes set up [Github integration]({{< relref "github-integration" >}}) and/or [Bitbucket integration]({{< relref "bitbucket-integration" >}}).