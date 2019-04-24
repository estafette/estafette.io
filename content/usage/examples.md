---
title: "Examples"
description: "A number of manifest examples for various popular languages"
weight: 4
---

### golang

```yaml
labels:
  app: my-application
  language: golang

stages:
  build:
    image: golang:1.11.2-alpine3.8
    workDir: /go/src/github.com/estafette/${ESTAFETTE_GIT_NAME}
    env:
      CGO_ENABLED: 0
      GOOS: linux
    commands:
    - go test `go list ./... | grep -v /vendor/`
    - CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -ldflags "-X main.version=${ESTAFETTE_BUILD_VERSION} -X main.revision=${ESTAFETTE_GIT_REVISION} -X main.branch=${ESTAFETTE_GIT_BRANCH} -X main.buildDate=${ESTAFETTE_BUILD_DATETIME}" -o ./publish/${ESTAFETTE_GIT_NAME} .
```

### csharp .net core

```yaml
labels:
  app: my-application
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
  app: my-application
  language: python

stages:
  build:
    image: python:3.7.1-alpine3.8
    commands:
    - python -m compileall
```

### java / maven

```yaml
labels:
  app: my-application
  language: java

stages:
  build:
    image: maven:10.13.0-alpine
    commands:
    - mvn -B clean verify
```

### node js

```yaml
labels:
  app: my-application
  language: nodejs

stages:
  build:
    image: node:10.13.0-alpine
    commands:
    - npm install
    - npm run build
```

### dockerize & push

```yaml
labels:
  app: my-application
  language: docker

stages:
  bake:
    image: extensions/docker:stable
    action: build
    repositories:
    - estafette
    copy:
    - /etc/ssl/certs/ca-certificates.crt .

  push-to-docker-hub:
    image: extensions/docker:stable
    action: push
    repositories:
    - estafette
```

### deployment to kubernetes engine

```yaml
releases:
  production:
    stages:
      deploy:
        image: extensions/gke:stable
        namespace: estafette
        visibility: public
        container:
          repository: estafette
          port: 8080
          cpu:
            request: 10m
            limit: 200m
          memory:
            request: 15Mi
            limit: 512Mi
          liveness:
            path: /robots.txt
          readiness:
            path: /robots.txt
          metrics:
            scrape: false
        hosts:
        - ci.estafette.io
        chaosproof: true
```