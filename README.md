## Estafette - resilient and cloud native CI/CD

The goals of Estafette CI are to...

- support many concurrent builds without keeping static agents around at all times.
- control builds and deployments from one concise manifest file.
- provide resilience against failure during pipeline execution.
- have full traceability by building every commit.
- make deployments easy to roll back.
- have zero downtime upgrades and easy rollbacks for Estafette itself.
- give insight into build times, deployment times, failure rates and more.

### Usage in your application

To build your application with Estafette add an `.estafette.yaml` file to your application repository.

```yaml
labels:
  app: estafette-ci-builder
  team: estafette-team
  language: golang
  
version:
  semver:
    major: 0
    minor: 0
    patch: $ESTAFETTE_BUILD_COUNTER

pipelines:
  build:
    image: golang:1.8.0-alpine
    workDir: /go/src/github.com/estafette/${ESTAFETTE_LABEL_APP}
    commands:
    - go test `go list ./... | grep -v /vendor/`
    - CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o ./publish/${ESTAFETTE_LABEL_APP} .

  bake:
    image: docker:17.03.0-ce
    commands:
    - cp Dockerfile ./publish
    - cp /etc/ssl/certs/ca-certificates.crt ./publish   
    - docker build -t estafette/${ESTAFETTE_LABEL_APP}:${ESTAFETTE_BUILD_VERSION} ./publish
    when:
      branch: master
      
  push-to-docker-hub:
    image: docker:17.03.0-ce
    commands:
    - docker login --username=${ESTAFETTE_DOCKER_HUB_USERNAME} --password='${ESTAFETTE_DOCKER_HUB_PASSWORD}'
    - docker push estafette/${ESTAFETTE_LABEL_APP}:${ESTAFETTE_BUILD_VERSION}      
    when:
      branch: master

  github-notify:
    image: estafette/github-nofity:latest

  slack-notify:
    image: estafette/slack-nofity:latest
    team: estafette
    channel: build-status
    when:
      status: [ failure ]

deployments:
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
    labels:
      environment: staging

  production:
    image: estafette/gke-deploy:latest
    project: estafette-production
    cluster: production
    namespace: estafette
    env:
      LOG_LEVEL: INFO
      HPA_MAX_REPLICAS: 25
    labels:
      environment: production
```
