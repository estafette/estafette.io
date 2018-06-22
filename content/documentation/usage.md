---
title: "Usage"
weight: 2
description: "Learn how to use Estafette to build your application"
date: 2018-06-19T12:51:45+02:00
menu:
  main:
    parent: 'documentation'
---

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

## Running a build

The **CI builder** is the component that interprets the `.estafette.yaml` file and executes the pipelines specified in there.

At this time the yaml file consist of two top-level sections, _labels_ and _pipelines_.

## Labels

_Labels_ are not required, but are useful to keep pipelines slightly less application-specific by using the labels as variables. In the future labels will be the mechanism to easily filter application pipelines by label values.

Any of the labels can be used in all the pipeline steps with the environment variable `ESTAFETTE_LABEL_<LABEL NAME>` in snake-casing.

## Pipelines

The _pipelines_ section allows you to define multiple steps to run within public Docker containers. Within each step you specify the public Docker image to use and the commands to execute within the Docker container. Your cloned repository is mounted within the docker container at `/estafette-work` by default, but you can override it with the `workDir` setting in case you need your code to live in a certain directory structure like with certain golang projects.

All files in the working directory are passed from step to step. Any files stored outside the working directory are not passed on. This means that - for example - when fetching NuGet packages with .NET core you specify the packages directory to be stored within the working directory, so it's available in the following steps.

For each step you can use a different Docker image, but it's best practice to minimize the number of different images / versions in order to save time on downloading all the Docker images.

Pipeline steps run in sequence. With the `when` property you can specify a logical expression - in golang format - to determine whether the step should run or be skipped. This is useful to, for example, only push docker containers to a registry for certain branches. Or to fire a notification if any of the previous steps has failed. The default `when` condition is `status == 'succeeded'`.

## Examples

### golang

```yaml
labels:
  app: <APP NAME>
  language: golang

pipelines:
  build:
    image: golang:1.8.1-alpine
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

pipelines:
  restore:
    image: microsoft/dotnet:1.1.1-sdk
    commands:
    - dotnet restore --source https://www.nuget.org/api/v1 --source http://nuget-server.tooling/nuget --packages .nuget/packages

  build:
    image: microsoft/dotnet:1.1.1-sdk
    commands:
    - dotnet build --configuration Release --version-suffix ${ESTAFETTE_BUILD_VERSION_PATCH}

  unit-tests:
    image: microsoft/dotnet:1.1.1-sdk
    commands:
    - dotnet test --configuration Release --no-build test/<unit test project directory/<unit test project file>.csproj

  integration-tests:
    image: microsoft/dotnet:1.1.1-sdk
    commands:
    - dotnet test --configuration Release --no-build test/<integration test project directory>/<integration test project file>.csproj

  publish:
    image: microsoft/dotnet:1.1.1-sdk
    commands:
    - dotnet publish src/<publisheable project directory> --configuration Release --runtime debian.8-x64 --version-suffix ${ESTAFETTE_BUILD_VERSION_PATCH} --output ./publish
```

### python

```yaml
labels:
  app: <APP NAME>
  language: python

pipelines:
  build:
    image: python:3.4.6-alpine
    commands:
    - python -m compileall
```

### java / maven

```yaml
labels:
  app: <APP NAME>
  language: java

pipelines:
  build:
    image: maven:3.3.9-jdk-8-alpine
    commands:
    - mvn -B clean verify -Dmaven.repo.remote=https://<sonatype nexus server>/content/groups/public
```

### node js

```yaml
labels:
  app: <APP NAME>
  language: nodejs

pipelines:
  build:
    image: node:7.8.0-alpine
    commands:
    - npm install --verbose
```

### dockerize & push for master branch

```yaml
labels:
  app: <APP NAME>

pipelines:
  bake:
    image: docker:17.04.0-ce
    commands:
    - cp /etc/ssl/certs/ca-certificates.crt .
    - docker build -t estafette/${ESTAFETTE_LABEL_APP}:${ESTAFETTE_BUILD_VERSION} .

  push-to-docker-hub:
    image: docker:17.04.0-ce
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

pipelines:
  slack-notify:
    image: docker:17.04.0-ce
    commands:
    - 'curl -X POST --data-urlencode ''payload={"channel": "#build-status", "username": "<DOCKER HUB USERNAME>", "text": "Build ''${ESTAFETTE_BUILD_VERSION}'' for ''${ESTAFETTE_LABEL_APP}'' has failed!"}'' ${ESTAFETTE_SLACK_WEBHOOK}'
    when:
      status == 'failed'
```