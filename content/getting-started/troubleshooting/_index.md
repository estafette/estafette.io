---
title: "Troubleshooting"
description: "Deal with some possible issues you can run into"
weight: 10
---

# CockroachDB underprovisioning

At some point you might run into the database being underprovisioned. Make sure to set resources on all components and follow other [Production / high availability]({{< relref "../production-high-availability" >}}) steps.

However whenever the database's pods start crashlooping it might have trouble recovering and you're often best to shrink the _statefulset_ to 0 instances and then back to the original amount with

```
kubectl scale statefulset estafette-ci-db -n estafette-ci --replicas=0
# wait for all pods to terminate
kubectl scale statefulset estafette-ci-db -n estafette-ci --replicas=3 # or whatever size you had set it to before
```

Now the cluster usually restarts and manages to perform the necessary leader election.

If it still keeps failing it's often that it has too little cpu. Below is an example of how to specify the resources via the Helm values:

```yaml
db:
  statefulset:
    resources:
      requests:
        cpu: 2000m
        memory: 12Gi
      limits:
        memory: 42Gi
```

Best to keep memory request and limit the same for _guaranteed quality of service (QoS)_.

For more _CockroachDB_ related troubleshooting tips check https://www.cockroachlabs.com/docs/stable/troubleshooting-overview.html.

# Stage has trouble with connectivity to external systems

When running on Google Cloud or within a _software defined network (SDN)_ with a smaller than default _MTU_ of 1500 you can run into trouble connecting to external systems from your build/release stages, due to the way stages are executed using _Docker inside Docker_. _Estafette CI_ defaults the MTU to 1460 so it works for Google Cloud, but you might have an SDN that uses an even smaller MTU. In that case you have to update the `config.yaml` with the following Helm values:

```yaml
api:
  config:
    files:
      config.yaml: |
        apiServer:
          baseURL: https://{{ $.Values.baseHost }}
          integrationsURL: https://{{ $.Values.integrationsHost }}
          serviceURL: http://{{ include "estafette-ci-api.fullname" . }}.{{ .Release.Namespace }}.svc.cluster.local
          dockerConfigPerOperatingSystem:
            linux:
              runType: dind
              mtu: <MTU size of your SDN>

        jobs:
          namespace: {{ .Release.Namespace }}-jobs

        auth:
          jwt:
            domain: {{ $.Values.baseHost }}
```

Most of this `config.yaml`'s content is the default values used in the Helm chart and needs to be repeated here in order to make sure your Estafette CI installation is configured correctly. See https://github.com/estafette/estafette-ci/blob/main/helm/estafette-ci/charts/estafette-ci-api/values.yaml#L270-L281 for the defaults.