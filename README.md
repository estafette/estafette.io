## Estafette - resilient and cloud-native CI/CD

The goals of Estafette CI are to...

- control builds and deployments from one concise manifest file.
- move control over build dependencies to the manifest file.
- control builds and deployments via cli, slack and web interface.
- give insight into build times, deployment times, failure rates and more.
- allow development teams to build their own extensions.
- support many concurrent builds by leveraging kubernetes jobs.
- provide resilience against failure during job execution.
- dogfood its own components by providing different tracks for each (dev, beta, stable/latest).
- allow itself to upgrade without downtime.

### Usage in your application

To build your application with Estafette add an `.estafette.yaml` file to your application repository.

```yaml
# these labels turn into ESTAFETTE_LABEL_... envvars, automatically injected into all pipeline steps
labels:
  app: estafette-ci-builder
  team: estafette-team
  language: golang

# semver version generates a 0.0.x version (or 0.0.x-<branch> for other branches than the release branch)
# made available as ESTAFETTE_BUILD_VERSION envvar, automatically injected into all pipeline steps
version:
  semver:
    major: 0
    minor: 0
    patch: '{{auto}}'
    labelTemplate: '{{branch}}'
    releaseBranch: master

# global environments variables that are automatically injected into all pipeline steps and can be
# overridden by defining an envvar in a pipeline step with the same name
env:
  VAR1: somevalue
  VAR2: another value

# pipeline steps are executed sequentially;
# a step uses a public container to mount the working directory into and execute commands in;
# extensions are containerized applications that execute based on injected environment variables or
# custom properties injected as ESTAFETTE_EXTENSION_<PROPERTY> envvars
# executing or skipping a step is controlled by the 'when' property
pipelines:
  set-pending-build-status:
    image: extensions/github-status:stable
    status: pending

  build:
    image: golang:1.9.2-alpine3.6
    workDir: /go/src/github.com/estafette/${ESTAFETTE_LABEL_APP}
    commands:
    - apk --update add git
    - go test `go list ./... | grep -v /vendor/`
    - CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -ldflags "-X main.version=${ESTAFETTE_BUILD_VERSION} -X main.revision=${ESTAFETTE_GIT_REVISION} -X main.branch=${ESTAFETTE_GIT_BRANCH} -X main.buildDate=${ESTAFETTE_BUILD_DATETIME}" -o ./publish/${ESTAFETTE_LABEL_APP} .

  bake-estafette:
    image: docker:17.09.0-ce
    commands:
    - cp Dockerfile ./publish
    - docker build -f ./publish/Dockerfile -t estafette/${ESTAFETTE_LABEL_APP}:${ESTAFETTE_BUILD_VERSION} ./publish

  push-to-docker-hub:
    image: docker:17.09.0-ce
    env:
      DOCKER_HUB_USERNAME: estafette.secret(T2YbJ4sAMgJMpZ.YS4vLTh8kdNQ5cBrU2AGztThmzJ7aD8w5a5W_-xY1Cw==)
      DOCKER_HUB_PASSWORD: estafette.secret(_kZ8EraN784wEF.GZEpqauwaEb33PmV7_-_fYfszjbKBEhgEfxvdLvZUE9aXq63tB)
    commands:
    - docker login --username=${DOCKER_HUB_USERNAME} --password="${DOCKER_HUB_PASSWORD}"
    - docker push estafette/${ESTAFETTE_LABEL_APP}:${ESTAFETTE_BUILD_VERSION}

  set-build-status:
    image: extensions/github-status:stable

  slack-notify:
    image: extensions/slack-build-status:stable
    webhook: estafette.secret(VQhyeydGURZSFLmh.LVhcvAuxx5EsgBqRJ9kvxXsvpZdSkP9pQ6AzUU5rrsFeKQAAAf8fGtQ6VhXL5qThw5m8hrsBASWQKaaYsGn5ZmC5jTUKpLfgg7YR)
    name: ${ESTAFETTE_LABEL_APP}
    channels:
    - '#build-status'
    when:
      status == 'failed'

```


### Installing Estafette CI


Prerequisites:

- install https://github.com/estafette/estafette-cloudflare-dns
- install https://github.com/estafette/estafette-letsencrypt-certificate
- install https://github.com/estafette/k8s-cockroachdb

Configure cluster roles and binding:

```
curl https://raw.githubusercontent.com/estafette/estafette-ci-api/master/rbac.yaml -o rbac.yaml

export NAMESPACE=estafette
export APP_NAME=estafette-ci-api

kubectl apply -f rbac.yaml
```

Install the API:

```
curl https://raw.githubusercontent.com/estafette/estafette-ci-api/master/kubernetes.yaml -o kubernetes.yaml

export NAMESPACE=estafette
export APP_NAME=estafette-ci-api
export TEAM_NAME=tooling-team
export HOSTNAMES=ci.estafette.io
export CLOUDFLARE_IP_RANGES=103.21.244.0/22, 103.22.200.0/22, 103.31.4.0/22, 104.16.0.0/12, 108.162.192.0/18, 131.0.72.0/22, 141.101.64.0/18, 162.158.0.0/15, 172.64.0.0/13, 173.245.48.0/20, 188.114.96.0/20, 190.93.240.0/20, 197.234.240.0/22, 198.41.128.0/17
export GITHUB_APP_PRIVATE_KEY=
export VERSION=123456
export GO_PIPELINE_LABEL=latest
export CPU_REQUEST=10m
export MEMORY_REQUEST=15Mi
export CPU_LIMIT=200m
export MEMORY_LIMIT=512Mi
export GITHUB_APP_ID=1234
export GITHUB_APP_OAUTH_CLIENT_ID=ab3.myjk4935qfamx8sw
export GITHUB_APP_OAUTH_CLIENT_SECRET=***
export GITHUB_WEBHOOK_SECRET=***
export BITBUCKET_API_KEY=***
export BITBUCKET_APP_OAUTH_KEY=f7mFzNWdFbZ2m54aCb
export BITBUCKET_APP_OAUTH_SECRET=***
export ESTAFETTE_CI_API_KEY=***
export SLACK_APP_CLIENT_ID=1575687846.473375433834
export SLACK_APP_CLIENT_SECRET=***
export SLACK_APP_VERIFICATION_TOKEN=***
export SLACK_APP_OAUTH_ACCESS_TOKEN=***
export SECRET_DECRYPTION_KEY=***
export COCKROACH_DATABASE=estafette_ci_api
export COCKROACH_HOST=cockroachdb-public.estafette.svc.cluster.local
export COCKROACH_INSECURE=true
export COCKROACH_CERTS_DIR=
export COCKROACH_PORT=26257
export COCKROACH_USER=estafette_ci_api
export COCKROACH_PASSWORD=***
export ALLOW_CIDRS=allow 0.0.0.0/0;
export MIN_PODS=3
export MAX_PODS=10
export AUTOSCALE_CPU_TARGET_PERCENTAGE=80

cat kubernetes.yaml | envsubst | kubectl apply -f -
```