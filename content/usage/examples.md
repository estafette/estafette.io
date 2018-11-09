---
title: "Examples"
description: "A number of manifest examples for various popular languages"
weight: 3
---


### golang

```yaml
labels:
  app: <APP NAME>
  language: golang

stages:
  build:
    image: golang:1.10.3-alpine3.8
    workDir: /go/src/github.com/estafette/${ESTAFETTE_LABEL_APP}
    commands:
    - go test `go list ./... | grep -v /vendor/`
    - CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -ldflags "-X main.version=${ESTAFETTE_BUILD_VERSION} -X main.revision=${ESTAFETTE_GIT_REVISION} -X main.branch=${ESTAFETTE_GIT_BRANCH} -X main.buildDate=${ESTAFETTE_BUILD_DATETIME}" -o ./publish/${ESTAFETTE_LABEL_APP} .
```

### csharp .net core

```yaml
labels:
  app: <APP NAME>
  language: dotnet-core

stages:
  restore:
    image: microsoft/dotnet:2.1-sdk
    commands:
    - dotnet restore --source https://www.nuget.org/api/v1 --source http://nuget-server.tooling/nuget --packages .nuget/packages

  build:
    image: microsoft/dotnet:2.1-sdk
    commands:
    - dotnet build --configuration Release --version-suffix ${ESTAFETTE_BUILD_VERSION_PATCH}

  unit-tests:
    image: microsoft/dotnet:2.1-sdk
    commands:
    - dotnet test --configuration Release --no-build test/<unit test project directory/<unit test project file>.csproj

  integration-tests:
    image: microsoft/dotnet:2.1-sdk
    commands:
    - dotnet test --configuration Release --no-build test/<integration test project directory>/<integration test project file>.csproj

  publish:
    image: microsoft/dotnet:2.1-sdk
    commands:
    - dotnet publish src/<publisheable project directory> --configuration Release --runtime debian.8-x64 --version-suffix ${ESTAFETTE_BUILD_VERSION_PATCH} --output ./publish
```

### python

```yaml
labels:
  app: <APP NAME>
  language: python

stages:
  build:
    image: python:3.7.0-alpine3.8
    commands:
    - python -m compileall
```

### java / maven

```yaml
labels:
  app: <APP NAME>
  language: java

stages:
  build:
    image: maven:3.5.4-jdk-10-slim
    commands:
    - mvn -B clean verify -Dmaven.repo.remote=https://<sonatype nexus server>/content/groups/public
```

### node js

```yaml
labels:
  app: <APP NAME>
  language: nodejs

stages:
  build:
    image: node:8.11.4-alpine
    commands:
    - npm install --verbose
```

### dockerize & push for master branch

```yaml
labels:
  app: <APP NAME>

stages:
  bake:
    image: docker:18.06.0-ce
    commands:
    - cp /etc/ssl/certs/ca-certificates.crt .
    - docker build -t estafette/${ESTAFETTE_LABEL_APP}:${ESTAFETTE_BUILD_VERSION} .

  push-to-docker-hub:
    image: docker:18.06.0-ce
    commands:
    - docker login --username=${ESTAFETTE_DOCKER_HUB_USERNAME} --password="${ESTAFETTE_DOCKER_HUB_PASSWORD}"
    - docker push estafette/${ESTAFETTE_LABEL_APP}:${ESTAFETTE_BUILD_VERSION}
    when:
      status == 'succeeded' &&
      branch == 'master'
```

### notification on failure

```yaml
labels:
  app: <APP NAME>

stages:
  slack-notify:
    image: docker:18.06.0-ce
    commands:
    - 'curl -X POST --data-urlencode ''payload={"channel": "#build-status", "username": "<DOCKER HUB USERNAME>", "text": "Build ''${ESTAFETTE_BUILD_VERSION}'' for ''${ESTAFETTE_LABEL_APP}'' has failed!"}'' ${ESTAFETTE_SLACK_WEBHOOK}'
    when:
      status == 'failed'
```