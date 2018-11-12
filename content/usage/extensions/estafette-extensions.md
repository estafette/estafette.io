---
title: "Estafette CI extensions"
description: "Estafette comes bundled with a number of extensions that can handle complex logic on your behalf"
weight: 1
---

### extensions/git-clone

```yaml
git-clone:
  image: extensions/git-clone:stable
  shallow: < boolean | true >
```

The `git-clone` stage is automatically injected if not present in your manifest for the build stages; in the release stages it's not injected. In case you don't need anything from your repository to release this speeds things up.

The shallow clone - enabled by default - checks out the latest 50 commits for the particular branch, then selects the correct revision. When rebuilding an older version or releasing an older version this might not be enough, in that case explicitly set `shallow: false`.

### extensions/github-status

```yaml
set-build-status:
  image: extensions/github-status:stable
  status: < string | ESTAFETTE_BUILD_STATUS >
```

The `github-status` extension takes the build status from the `ESTAFETTE_BUILD_STATUS` environment variable, but this can be overridden with the `status` tag. Depending on the source of the git push - github or bitbucket - this extension is automatically injected at the beginning of the build stages and at the end in order to set the build status in the respective source code hosting system:

```yaml
set-pending-build-status:
  image: extensions/github-status:stable
  status: pending

...

set-build-status:
  image: extensions/github-status:stable
  when:
    status == 'succeeded' || 
    status == 'failed'
```

### extensions/bitbucket-status

```yaml
set-build-status:
  image: extensions/bitbucket-status:stable
  status: < string | ESTAFETTE_BUILD_STATUS >
```

The `bitbucket-status` extension takes the build status from the `ESTAFETTE_BUILD_STATUS` environment variable, but this can be overridden with the `status` tag. Depending on the source of the git push - github or bitbucket - this extension is automatically injected at the beginning of the build stages and at the end in order to set the build status in the respective source code hosting system:

```yaml
set-pending-build-status:
  image: extensions/bitbucket-status:stable
  status: pending

...

set-build-status:
  image: extensions/bitbucket-status:stable
  when:
    status == 'succeeded' || 
    status == 'failed'
```

### extensions/slack-build-status

```yaml
slack-notify:
  image: extensions/slack-build-status:stable
  webhook: estafette.secret(***)
  channels:
  - '#mychannel'
  when:
    status == 'succeeded' || 
    status == 'failed'
```

In order to send build notifications to one or more Slack channels the `slack-build-status` extension automatically sends a messages depending on the build status into the `channels` passed to this extension. The `webhook` needs a Slack webhook url in order to send the message.

Notes:
* This extension will be updated in the future to make use of the _credentials and trusted images_ configuration in the Estafette CI server so there's no need to embed a webhook in the manifest.

### extensions/docker

The `docker` extension supports the following actions: *build*, *push*, *tag*. For pushing and tagging containers it uses the _credentials and trusted images_ configuration in the Estafette CI server to get access to Docker registry credentials automatically.

##### build

```yaml
bake:
  image: extensions/docker:stable
  action: build
  container: < string | ESTAFETTE_LABEL_APP >
  repositories:
  - estafette
  path: < string | . >
  dockerfile: < string | Dockerfile >
  copy:
  - < string | copies Dockerfile by default >
  - /etc/ssl/certs/ca-certificates.crt
```

A minimal version when using all defaults looks like:

```yaml
bake:
  image: extensions/docker:stable
  action: build
  repositories:
  - estafette
```

##### push

```yaml
bake:
  image: extensions/docker:stable
  action: push
  container: < string | ESTAFETTE_LABEL_APP >
  repositories:
  - estafette
  tags:
  - < string | tags with ESTAFETTE_BUILD_VERSION by default and is always pushed >
  - dev
```

A minimal version when using all defaults looks like:

```yaml
bake:
  image: extensions/docker:stable
  action: push
  repositories:
  - estafette
```

##### tag

To later on tag a specific version with another tag - for example to promote a dev version to stable you can use the `docker` extension to tag that version with other tags:

```yaml
tag-container-image:
  image: extensions/docker:dev
  action: tag
  container: < string | ESTAFETTE_LABEL_APP >
  repositories:
  - estafette
  tags:
  - stable
  - latest
```

```yaml
bake:
  image: extensions/docker:stable
  action: tag
  repositories:
  - estafette
  tags:
  - latest
```


### extensions/gke

The `gke` extension is used to deploy an application to Kubernetes Engine. It generates a very opiniated deployment with a sidecar handling incoming traffic and forwarding requests; it asumes _horizontal pod autoscaling_, it injects secrets and configs in a standard way, and uses a lot of other sensible defaults to be able to use it with a minimum number of parameters specified.

```yaml
deploy:
  image: extensions/gke:stable
  credentials: < string | gke-ESTAFETTE_RELEASE_NAME >
  app: < string | ESTAFETTE_LABEL_APP >
  namespace: < string | default namespace from credentials config >
  visibility: < string | private >
  container:
    repository: < string >
    name: < string | app >
    tag: < string | ESTAFETTE_BUILD_VERSION >
    port: < integer | 5000 >
    env:
      MY_CUSTOM_ENV: value1
      MY_OTHER_CUSTOM_ENV: value2
    cpu:
      request: < string | 100m >
      limit: < string | 125m >
    memory:
      request: < string | 128Mi >
      limit: < string | 128Mi >
    liveness:
      path: < string | /liveness >
      delay: < integer | 30 >
      timeout: < integer | 1 >
    readiness:
      path: < string | /readiness >
      delay: < integer | 0 >
      timeout: < integer | 1 >
    metrics:
      scrape: < boolean | true >
      path: < string | /metrics >
      port: < integer | .container.port >
  sidecar:
    type: < string | openresty >
    image: < string | estafette/openresty-sidecar:1.13.6.1-alpine >
    env:
      CORS_ALLOWED_ORIGINS: "*"
      CORS_MAX_AGE: "86400"
    cpu:
      request: < string | 10m >
      limit: < string | 50m >
    memory:
      request: < string | 10Mi >
      limit: < string | 50Mi >
  autoscale:
    min: < integer | 3 >
    max: < integer | 100 >
    cpu: < integer | 80 >
  secrets:
    keys:
      secret-file-1.json: c29tZSBzZWNyZXQgdmFsdWU=
      secret-file-2.yaml: YW5vdGhlciBzZWNyZXQgdmFsdWU=
    mountpath: /secrets
  configs:
    files:
    - config/config.json
    - config/anotherconfig.yaml
    data:
      property1: value 1
      property2: value 2
    mountpath: /configs
  manifests:
    files:
    - override/service.yaml
    data:
      property3: value 3
  hosts:
  - gke.estafette.io
  - gke-deploy.estafette.io
  basepath: < string | / >
  enablePayloadLogging: < boolean | false >
  chaosproof: < boolean | false >
  rollingupdate:
    maxsurge: < string | 25% >
    maxunavailable: < string | 25% >
    timeout: < string | 5m >
  trustedips:
  - 103.21.244.0/22
  - 103.22.200.0/22
  - 103.31.4.0/22
  dryrun: < boolean | false >
```

A very minimal version when using all the defaults looks like:

```yaml
production:
  stages:
    deploy:
      image: extensions/gke:stable
      visibility: public
      container:
        repository: estafette
      hosts:
      - ci.estafette.io
```