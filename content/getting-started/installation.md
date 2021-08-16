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

Then install the `estafette-ci` chart with

```
helm upgrade --install estafette-ci estafette/estafette-ci -n estafette-ci --timeout 600s
```

With Estafette CI making use of CockroachDB in **secure mode** you need to approve one or more _certificate signing requests_ by doing the following while the installation runs (and before it times out):

```
kubectl get csr
```

You will see one or more csr's and will have to approve this with (for example):

```
kubectl certificate approve estafette-ci.client.root
kubectl certificate approve estafette-ci.node.estafette-ci-db-0
kubectl certificate approve estafette-ci.node.estafette-ci-db-1
kubectl certificate approve estafette-ci.node.estafette-ci-db-2
```