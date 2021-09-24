---
title: "Production / High Availability"
description: "Set up Estafette CI in High Availability mode for production"
weight: 8
---

By default this Helm chart sets up _Estafette CI_ so that you can have a look at it, try it out, without requiring too much resources. However in order to run it in production you do want to tune some settings in order for all components to run in High Availability (HA) mode.

# Defaults

Having a look at the default values reveals how many replicas of each component is installed. It is notable that _Cockroachdb_ is the only component already running in HA mode. This is because it's much harder to change it into HA once it's initialized. For other components it's no problem to do the change at a later stage.

```yaml
api:
  deployment:
    replicaCount: 1
  autoscaling:
    enabled: false
web:
  deployment:
    replicaCount: 1
  autoscaling:
    enabled: false
db:
  statefulset:
    replicas: 3
metrics:
  server:
    replicaCount: 1
queue:
  cluster:
    enabled: false
```

# High Availability

In order to have all the components that are there to handle browser requests, web hooks, storage or internal communication run in _High Availability_ mode use the following values:

```yaml
api:
  deployment:
    replicaCount: 3
  autoscaling:
    enabled: true
web:
  deployment:
    replicaCount: 3
  autoscaling:
    enabled: true
db:
  statefulset:
    replicas: 3
metrics:
  server:
    replicaCount: 2
queue:
  cluster:
    enabled: true
    replicas: 3
```

# Resources

It also make sense to set resource requests and limits for each component so it gets the cpu and memory it needs:

```yaml
api:
  deployment:
    resources:
      requests:
        cpu: 100m
        memory: 256Mi
      limits:
        memory: 256Mi
web:
  deployment:
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        memory: 128Mi
db:
  statefulset:
    resources:
      requests:
        cpu: 2000m
        memory: 12Gi
      limits:
        memory: 12Gi
metrics:
  server:
    resources:
      requests:
        cpu: 1000m
        memory: 6Gi
      limits:
        memory: 6Gi
queue:
  nats:
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        memory: 128Mi
```

Do not these are just example values and you should tune resources over time and monitor them closely to see when you're either overprovisioning or underprovisiong each of the components.