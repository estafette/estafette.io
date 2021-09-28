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

### Automatically generated secret keys

During the first install a secret named `estafette-ci-api` is created containing the `secretDecryptionKey` and `jwtKey`; they're initialized to random strings of 32 character, in order to use AES-256 for encrypting and decrypting _Estafette secrets_ and for encrypting the JSON Web Token used in the _Authorization_ header in communication between the various parts of the Estafette system.

You can see this mechanism at https://github.com/estafette/estafette-ci/blob/main/helm/estafette-ci/charts/estafette-ci-api/templates/secret.yaml. Through its use of the `lookup` function it's possible to leave those keys blank in your values file. However a `helm diff` doesn't always render this correctly, it sometimes misleads you into thinking that it will change those keys, while in reality it doesn't.

See these [instructions]({{< relref "production-high-availability/#back-up-decryption-key" >}}) on making sure you securely backup your `secretDecryptionKey` for [disaster recovery]({{< relref "disaster-recovery" >}}) purposes.

#### Difference between global and pipeline restricted secrets

Both the _global_ and _pipeline restricted_ form of Estafette secrets use the same `secretDecryptionKey`. The restriction itself is embedded into the secret inside the `estafette.secret(...)` envelope.