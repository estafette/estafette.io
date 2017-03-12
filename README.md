## Estafette - resilient and cloud native CI

The goals of Estafette CI are to...

- support large number of concurrents builds without keeping many static agents around when there's nothing to do.
- control builds and deployments from one concise manifest file.
- provide resilience against failure due to preemptible hardware.
- have full traceability by not bundling multiple commits in a single pipeline run.
- have zero downtime upgrades and easy rollbacks for Estafette itself.

### Usage in your application

To build your application with Estafette add an `.estafette.yaml` file to your application repository.

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
