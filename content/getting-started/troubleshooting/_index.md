---
title: "Troubleshooting"
description: "Deal with some possible issues you can run into"
weight: 11
---

## Pipelines not triggered

First check the status of the various source code management (SCM) systems to see if they have an issue deliverying webhooks:

* https://www.githubstatus.com/
* https://bitbucket.status.atlassian.com/

If this is the case they'll eventually start delivering those webhooks and pipelines will trigger at that time.

Next thing to check is delivery of webhooks within the respective SCM:

* Github - https://github.com/organizations/<organization>/settings/apps/<estafette app name>/advanced
* Bitbucket - https://bitbucket.org/<workspace>/workspace/settings/applications, select the estafette app, check delivery

If you can see webhooks getting sent to Estafette but failing there it points you to where things might go wrong. Can be the ingress controller, SCM integration details in Estafette itself or something else. Check the logs for the `estafette-ci-api` component to see if there's any errors.

When nothing can be found it might have been a transient error that isn't automatically retried. You can then try to push a new empty commit just to trigger the pipeline and see if it helps:

```
git commit --allow-empty -m "trigger build" && git push
```

## CockroachDB underprovisioning

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

## Docker daemon failing to start or struggling to execute stages

The `estafette-ci-builder` component running the build/release stages inside a Kubernetes job utilizing _Docker inside Docker_ can put quite a lot of strain on the underlying Kubernetes nodes. On Google Cloud the default 100GB pd-standard boot disks seem to fall short a bit for Estafette CI to run really well. Upgrading those to 200GB pd-ssd boot disks should resolve any issues in this department.

## Connectivity failure to external systems

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

## Configuration errors

When your configuration file is invalid the `estafette-ci-api` component will fail and restart constantly, leading to a failure to upgrade the Helm release. Check the logs of the `estafette-ci-api` pods (with `kubectl logs` since `stern` will be too slow to catch those errors) to see which configuration item is incorrect and fix it accordingly.

## Config too large for configmap

When you continue to add _credentials_ to your config eventually you'll hit the maximum size of configmaps. What you can do then is store the config in a git repository and use a git-sync sidecar to fetch the config, instead of using a configmap for this.

The values you need for this is pretty big but will resolve this issue for you:

```yaml
api:
  config:
    enabled: false
  deployment:
    extraEnv:
      - name: CONFIG_FILES_PATH
        value: /configs/git
    extraContainers: |
      - name: estafette-ci-api-git-sync
        image: k8s.gcr.io/git-sync:v3.1.5
        env:
        - name: GIT_SYNC_REPO
          value: "<git repository url (ssh not https)>"
        - name: GIT_SYNC_BRANCH
          value: "<branch that has your config.yaml>"
        - name: GIT_SYNC_DEPTH
          value: "1"
        - name: GIT_SYNC_ROOT
          value: "/configs"
        - name: GIT_SYNC_DEST
          value: "git"
        - name: GIT_SYNC_SSH
          value: "true"
        - name: GIT_SSH_KEY_FILE
          value: "/ssh/git-sync-rsa"
        - name: GIT_KNOWN_HOSTS
          value: "false"
        - name: GIT_SYNC_MAX_SYNC_FAILURES
          value: "10"
        - name: GIT_SYNC_WAIT
          value: "60"
        securityContext:
          runAsUser: 65534 # git-sync user
        resources:
          requests:
            cpu: 10m
            memory: 100Mi
          limits:
            memory: 100Mi
        volumeMounts:
        - name: configs
          mountPath: /configs
        - name: secrets-ssh
          mountPath: /ssh
    extraVolumes: |
      - name: client-certificate
        projected:
          sources:
          - secret:
              name: estafette-ci-db-client-secret
              items:
              - key: ca.crt
                path: ca.crt
                mode: 256
              - key: tls.crt
                path: tls.crt
                mode: 256
              - key: tls.key
                path: tls.key
                mode: 256
      - name: configs
        emptyDir: {}
    extraVolumeMounts: |
      - name: client-certificate
        mountPath: /cockroach-certs
      - name: configs
        mountPath: /configs
  extraSecrets:
    - key: ssh
      mountPath: /ssh
      b64encoded: true
      data:
        git-sync-rsa: '<base64 encoded ssh key with read access to the git repository with config.yaml file>'
```

Do not that the `extraVolumes` and `extraVolumeMounts` for `client-certificate` aren't needed for this trick, but are default values already used in the Helm chart, but won't be included if you do not specify them again.

Note: drawback of having this 'runtime' configuration update method is that validation errors won't be detected during a rollout, but can happen when you update the configuration file(s) in the synced git repository. Again in this case check the logs of the `estafette-ci-api` pods (with `kubectl logs` since `stern` will be too slow to catch those errors) to see which configuration item is incorrect and fix it accordingly.