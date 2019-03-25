---
title: "Estafette extensions"
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

##### cloning another repository from the same owner

You can use the `git-clone` extension to clone another repository as well, as long as it's from the same owner. In order to make use of it you need to specify the repository name (without owner), and optionally the branch you want to clone (defaults to master) and the subdir to clone it into (defaults to the repository name).

```yaml
clone-another-repo:
  image: extensions/git-clone:stable
  shallow: < boolean | true >
  repo: < string >
  branch: < string | master >
  subdir: < string | repo >
```

Notes:

* Make sure to avoid the stage name `git-clone` if you're cloning another repository in one of your stages, because the regular repository will not be cloned automatically in that case.

### extensions/docker

The `docker` extension supports the following actions: *build*, *push*, *tag*. For pushing and tagging containers it uses the _[credentials and trusted images]({{< relref "/getting-started/configuration.md#credentials" >}})_ configuration in the Estafette server to get access to Docker registry credentials automatically.

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

If you'd like to avoid having a separate Dockerfile you can inline it as well. The benefit is that this automatically substitutes global or Estafette environment variables:

```yaml
  bake:
    image: extensions/docker:stable
    action: build
    inline: |
      FROM scratch
      COPY ca-certificates.crt /etc/ssl/certs/
      COPY ${ESTAFETTE_GIT_NAME} /
      ENTRYPOINT ["/${ESTAFETTE_GIT_NAME}"]
    repositories:
    - estafette
    path: ./publish
    copy:
    - /etc/ssl/certs/ca-certificates.crt
```

##### push

```yaml
push:
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
push:
  image: extensions/docker:stable
  action: push
  repositories:
  - estafette
```

##### tag

To later on tag a specific version with another tag - for example to promote a dev version to stable you can use the `docker` extension to tag that version with other tags:

```yaml
tag:
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
tag:
  image: extensions/docker:stable
  action: tag
  repositories:
  - estafette
  tags:
  - latest
```

### extensions/gke

The `gke` extension is used to deploy an application to Kubernetes Engine. It generates a very opinionated deployment with a sidecar. It asumes _horizontal pod autoscaling_, it injects secrets and configs in a standard way, and uses a lot of other sensible defaults to be able to use it with a minimum number of parameters specified.

By default it includes an [OpenResty](https://openresty.org/en/) sidecar handling incoming traffic and forwarding requests. ([This](https://github.com/estafette/openresty-sidecar) is the custom openresty image injected by default.) The sidecar configuration can be adjusted, and additional sidecars can be included by customizing the `sidecars` field. (The only other type currently supported is the `cloudsqlproxy`, which adds the proxy implementing secure communication with Google Cloud SQL databases.)  
If you don't want to use the default sidecar, you should add the `injecthttpproxysidecar: false` field to the deployment, then it will not be injected during the development.

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
  sidecars:
  - type: < string | openresty | cloudsqlproxy >
    image: < string | estafette/openresty-sidecar:1.13.6.2-alpine | gcr.io/cloudsql-docker/gce-proxy:1.13 >
    env:
      CORS_ALLOWED_ORIGINS: "*"
      CORS_MAX_AGE: "86400"
    cpu:
      request: < string | 10m >
      limit: < string | 50m >
    memory:
      request: < string | 10Mi >
      limit: < string | 50Mi >
    # Only relevant for cloudsqlproxy
    dbinstanceconnectionname: my-gcloud-project:europe-west1:my-database
    sqlproxyport: 5043
  # Additional sidecars can be listed here.
  - type: < string | openresty | cloudsqlproxy >
    image: < string | estafette/openresty-sidecar:1.13.6.2-alpine | gcr.io/cloudsql-docker/gce-proxy:1.13 >
    ...
  autoscale:
    min: < integer | 3 >
    max: < integer | 100 >
    cpu: < integer | 80 >
    safety:
      enabled: < bool | false >
      promquery: < string | sum(rate(nginx_http_requests_total{app='app'}[5m])) by (app) >
      ratio: < float | 1.0 >
      delta: < float | 0.0 >
      scaledownratio: < float | 1.0 >
  secrets:
    keys:
      secret-file-1.json: c29tZSBzZWNyZXQgdmFsdWU=
      secret-file-2.yaml: YW5vdGhlciBzZWNyZXQgdmFsdWU=
    mountpath: /secrets
  configs:
    # these are local files with golang template style placeholders, replaced with the values specified in the data section;
    # set clone: true on the release target to ensure you have access to these files stored in your repository
    files:
    - config/config.json
    - config/anotherconfig.yaml
    # these are the values for the placeholders specified as {{.property1}} to be replaced in the config files
    data:
      property1: value 1
      property2: value 2
    # if you want to avoid cloning your repository and just need to pass a very short config you can inline full files here
    inline:
      inline-config.properties: |
        enemies=aliens
        lives=3
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
      visibility: private
      container:
        repository: estafette
      hosts:
      - ci.estafette.io
```

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
  workspace: estafette
  channels:
  - '#mychannel'
  when:
    status == 'succeeded' || 
    status == 'failed'
```

In order to send build notifications to one or more Slack channels the `slack-build-status` extension automatically sends a messages depending on the build status into the `channels` passed to this extension for the `workspace` you've set.

Notes:

* This extension uses the _[credentials and trusted images]({{< relref "/getting-started/configuration.md#credentials" >}})_ configuration in the Estafette server to gain access to the slack webhooks configured there.
* If you only want to send a message on failure use

```
  when:
    status == 'failed'
```