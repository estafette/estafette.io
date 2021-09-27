---
title: "Security considerations"
description: "Things to take into account for keeping your platform secure"
weight: 10
---

## Privileged containers

_Estafette CI_ uses _privileged_ containers to run builds and releases. To ensure that it can't impact other workloads running in your Kubernetes cluster you're best of setting up dedicated node pools with taints and then using toleration and affinity in the Estafette CI config.

So let's say you've create a node pool with

```
gcloud container node-pools create pool-name \
  --cluster cluster-name \
  --node-taints role=privileged:NoSchedule
```

Then update the config to the following (this includes the required defaults plus specific items related to tolerations/affinity):


```yaml
api:
  config:
    files:
      config.yaml: |
        apiServer:
          baseURL: https://{{ $.Values.baseHost }}
          integrationsURL: https://{{ $.Values.integrationsHost }}
          serviceURL: http://{{ include "estafette-ci-api.fullname" . }}.{{ .Release.Namespace }}.svc.cluster.local

        jobs:
          namespace: {{ .Release.Namespace }}-jobs

          build:
            affinity:
              nodeAffinity:
                requiredDuringSchedulingIgnoredDuringExecution:
                  nodeSelectorTerms:
                  - matchExpressions:
                    - key: role
                      operator: In
                      values:
                      - privileged
            tolerations:
            - key: role
              operator: Equal
              value: privileged
              effect: NoSchedule

          release:
            affinity:
              nodeAffinity:
                requiredDuringSchedulingIgnoredDuringExecution:
                  nodeSelectorTerms:
                  - matchExpressions:
                    - key: role
                      operator: In
                      values:
                      - privileged
            tolerations:
            - key: role
              operator: Equal
              value: privileged
              effect: NoSchedule

          bot:
            affinity:
              nodeAffinity:
                requiredDuringSchedulingIgnoredDuringExecution:
                  nodeSelectorTerms:
                  - matchExpressions:
                    - key: role
                      operator: In
                      values:
                      - privileged
            tolerations:
            - key: role
              operator: Equal
              value: privileged
              effect: NoSchedule

        auth:
          jwt:
            domain: {{ $.Values.baseHost }}
```

This will run all of the build/release bot jobs on nodes with taint `role=privileged`.

## Estafette secrets

Estafette secrets are encrypted using AES-256 - as long as your `secretDecryptionKey` is 32 characters long - which should be safe enough against any attempts to brute force decrypt your secrets. However the `secretDecryptionKey` itself is stored in a Kubernetes secret. So the biggest attack vector is someone that manages to read the Kubernetes secret and hence obtains the decryption key. This will allow that person to decrypt any of the secrets used in your Estafette CI installation. Storing secrets in a real secrets management system will be much safer. You can try storing references to those in your manifests with https://github.com/variantdev/vals, but at this moment there is no native support in Estafette for using this system.