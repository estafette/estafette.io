## Estafette, resilient and scalable CI

To support large number of concurrents builds without running many static or slow starting elastic agents when there's nothing to do Estafette leverages Kubernetes to run jobs for each build.

To determine the exact dependencies for each application Estafette runs each step of your pipeline in any Docker container you like to use.

To be able to run your jobs on cheap cloud vms that can be killed at any time Estafette provides resilience for pipelines that fail because of this.

To have full traceability of changes to your CI configurations everything is version controlled.

To do what it preaches - Continuous Delivery - Estafette can upgrade with zero downtime and no loss of running pipelines.

These are the goals that Estafette has to deliver in order to be a success.

### Markdown

To build your application with Estafette add a .estafette.yaml file to your application repository.

```yaml
label:
  app: estafette-ci
  team: estafette-team
  language: golang
  
version:
  semver:
    major: 0
    minor: 0
    patch: $ESTAFETTE_BUILD_COUNTER

pipeline:
  build:
    image: golang:1.8.0-alpine
    commands:
    - go test -v ./...
    - CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o ./publish/$LABEL_APP .

  dockerize:
    image: estafette/dockerizer:latest
    directory: /publish
    repo: estafette/$LABEL_APP
    tags: [ latest, $VERSION ]
    when:
      branch: master
    dockerfile: |-
      FROM scratch

      COPY ca-certificates.crt /etc/ssl/certs/
      COPY $LABEL_APP /

      ENTRYPOINT ["/$LABEL_APP"]

  git-build-status:
    image: estafette/git-build-status:latest

  slack-build-status:
    image: estafette/slack-build-status:latest
    channel: estafette-team-build-status
    when:
      status: [ failure ]

deployment:
  env:
    LIVENESS_ENDPOINT: /liveness
    READINESS_ENDPOINT: /readiness
    REPLICAS: 3
    HPA_MAX_REPLICAS: 10

  staging:
    image: estafette/gke-deploy:latest
    project: estafette-staging
    cluster: staging
    namespace: estafette
    env:
      LOG_LEVEL: DEBUG

  production:
    image: estafette/gke-deploy:latest
    project: estafette-production
    cluster: production
    namespace: estafette
    env:
      LOG_LEVEL: INFO
      HPA_MAX_REPLICAS: 25
```
