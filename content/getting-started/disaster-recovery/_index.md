---
title: "Disaster recovery"
description: "Recover Estafette CI and its database after the unimaginable happened"
weight: 12
---

Estafette CI is a pretty resilient system, but of course an incident can happen. Like wiping out a complete namespace by accident. This will not only loose you all of the pipeline build version numbers and references to the logs, but also the master encryption/decryption key.

# Recreate/repair Helm release

If your Estafette CI release somehow managed to become corrupt or deleted you can reinstall it with the instructions below. Do ensure you have the original `values.yaml` file stored somewhere. And also have any of the secrets used during installation at hand.

```
helm repo add estafette https://helm.estafette.io
helm repo update

# check diff
helm diff upgrade estafette-ci estafette/estafette-ci -n estafette-ci --values local-values.yaml --set api.secret.secretDecryptionKey=<base64 encoded secret decryption key>

# apply changes
helm upgrade --install estafette-ci estafette/estafette-ci -n estafette-ci --create-namespace --values local-values.yaml --timeout 600s --set api.secret.secretDecryptionKey=<base64 encoded secret decryption key> --set db-migrator.enable=false
```

Any of the secret values you might use can be passed with the `--set` argument. Make sure you've stored these commands somewhere secure, like in a password manager, so you don't have to figure out all of the secrets when trying to restore functionality.

Note: the `db-migrator` component needs to be disabled if the database needs to be restored from a backup, otherwise it already creates database tables which makes it impossible for a backup to be restored.

## CockroachDB restore backup

If you've following the instructions in the [Production / high availability]({{< relref "../production-high-availability" >}}) section you've already set up a daily backup for Estafette's Cockroachdb database.

After having followed the instructions above you hopefully still have all of your data, but in case you threw away all the _Persistent Volume Claims_ for the database you will have to restore it from your backup.

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

1. Get a keyfile for the service account used for the scheduled backup
2. Execute the following query in the `estafette-ci-db-client`:

```sql
RESTORE
  FROM 'gs://{bucket name}/db-backups?AUTH=specified&CREDENTIALS={base64 encoded key}';
```

Once you're done best to disable the _db-client_ again by updating the values to

```yaml
db-client:
  enabled: false
```

and making sure to either pass `--set db-migrator.enable=true` to `helm upgrade` or skip overriding this value completely. This will let the `db-migrator` perform any schema changes that still need to happen, although with a daily backup the schema should be up to date.