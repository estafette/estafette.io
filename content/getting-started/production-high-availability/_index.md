---
title: "Production / High Availability"
description: "Set up Estafette CI in High Availability mode for production"
weight: 8
---

By default this Helm chart sets up _Estafette CI_ so that you can have a look at it, try it out, without requiring too much resources. However in order to run it in production you do want to tune some settings in order for all components to run in High Availability (HA) mode.

## Default values

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

## High Availability

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

## Resources

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

## Avoid Docker Hub rate limits

Since Docker Hub introduced lowered rate limits for anonymous pulls - see https://www.docker.com/increase-rate-limits - it's good practice to use _image pull secrets_ for all of the containers pulled from Docker Hub. You can do so by creating a Docker Hub account and generate a token. Once you have those apply the following values:

```yaml
api:
  image:
    credentials:
      registry: docker.io
      username: '<docker hub user>'
      password: '<docker hub token>'
web:
  image:
    credentials:
      registry: docker.io
      username: '<docker hub user>'
      password: '<docker hub token>'
db:
  image:
    credentials:
      registry: docker.io
      username: '<docker hub user>'
      password: '<docker hub token>'
cron-event-sender:
  image:
    credentials:
      registry: docker.io
      username: '<docker hub user>'
      password: '<docker hub token>'
hanging-job-cleaner:
  image:
    credentials:
      registry: docker.io
      username: '<docker hub user>'
      password: '<docker hub token>'
db-migrator:
  image:
    credentials:
      registry: docker.io
      username: '<docker hub user>'
      password: '<docker hub token>'
db-client:
  image:
    credentials:
      registry: docker.io
      username: '<docker hub user>'
      password: '<docker hub token>'
metrics:
  imagePullSecrets:
  - name: estafette-ci-api.registry
queue:
  imagePullSecrets:
  - name: estafette-ci-api.registry
```

In order to ensure builds also use credentials in order to elevate _docker pull_ quota add a credentials of type `container-registry-pull` in the following way:

```yaml
config:
  files:
    credentials.yaml: |
      credentials:
      - name: 'docker-hub-pull'
        type: 'container-registry-pull'
        username: '<docker hub user>'
        password: '<docker hub token>'
```

These will be used both as an _image pull secret_ for the build/release jobs itself and for pulling stage container images inside of each build/release.

## CockroachDB backup

The most critical part of _Estafette CI_ to safeguard for disaster discovery is the data stored in the database. The default database used by Estafette is CockroachDB. You'll have to set up a _backup schedule_ as documented at https://www.cockroachlabs.com/docs/stable/manage-a-backup-schedule.html in order to have daily backups of the database.

In order to connect to the Cockroachdb database to perform queries, the Helm chart has a `db-client` subchart, that's disabled by default. You can enabled it by setting values:

```yaml
db-client:
  enabled: true
```

This will spin up a pod named `estafette-ci-db-client` which you can 'log in to' with the following command:

```
kubectl exec -it estafette-ci-db-client -n estafette-ci \
-- ./cockroach sql \
--certs-dir=/cockroach-certs \
--host=estafette-ci-db-public
```

From here on you can follow the instructions as listed by the documentation of Cockroachdb.

For example in order to create a scheduled backup to Google Cloud Storage

1. Create a cloud storage bucket
2. Create a service account
3. Give the service account read/write permissions on the storage bucket
4. Get a keyfile for the service account
5. Execute the following query in the `estafette-ci-db-client`:

```sql
CREATE SCHEDULE schedule_label
  FOR BACKUP INTO 'gs://{bucket name}/backups/daily?AUTH=specified&CREDENTIALS={base64 encoded key}'
    WITH revision_history
    RECURRING '@daily'
    WITH SCHEDULE OPTIONS first_run = 'now';
```

Once you're done best to disable the _db-client_ again by updating the values to

```yaml
db-client:
  enabled: false
```

## Safely back up encryption/decryption key

In case you're already using _Estafette secrets_ in build manifests and centrally configured _credentials_ you'll need a backup of the encryption/decryption key. You can get it's value with

```
kubectl get secret --namespace estafette-ci estafette-ci-api -o jsonpath="{.data.secretDecryptionKey}" | base64 --decode ; echo
```

Best store this securely in a password manager. This is the most sensitive piece of data used by Estafette. Once it leaks everyone can decrypt any of the _Estafette secrets_ used by the system.