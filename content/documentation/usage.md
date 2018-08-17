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

# pipeline stages are executed sequentially;
# a step uses a public container to mount the working directory into and execute commands in;
# extensions are containerized applications that execute based on injected environment variables or
# custom properties injected as ESTAFETTE_EXTENSION_<PROPERTY> envvars
# executing or skipping a step is controlled by the 'when' property
stages:
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

The **CI builder** is the component that interprets the `.estafette.yaml` file and executes the pipeline stages specified in there.

At this time the yaml file consist of two top-level sections, _labels_ and _stages_.

## Labels

_Labels_ are not required, but are useful to keep pipelines slightly less application-specific by using the labels as variables. In the future labels will be the mechanism to easily filter application pipelines by label values.

Any of the labels can be used in all the pipeline steps with the environment variable `ESTAFETTE_LABEL_<LABEL NAME>` in snake-casing.

## Stages

The _stages_ section allows you to define multiple steps to run within public Docker containers. Within each step you specify the public Docker image to use and the commands to execute within the Docker container. Your cloned repository is mounted within the docker container at `/estafette-work` by default, but you can override it with the `workDir` setting in case you need your code to live in a certain directory structure like with certain golang projects.

All files in the working directory are passed from step to step. Any files stored outside the working directory are not passed on. This means that - for example - when fetching NuGet packages with .NET core you specify the packages directory to be stored within the working directory, so it's available in the following steps.

For each step you can use a different Docker image, but it's best practice to minimize the number of different images / versions in order to save time on downloading all the Docker images.

Pipeline steps run in sequence. With the `when` property you can specify a logical expression - in golang format - to determine whether the step should run or be skipped. This is useful to, for example, only push docker containers to a registry for certain branches. Or to fire a notification if any of the previous steps has failed. The default `when` condition is `status == 'succeeded'`.

## Releasing from Estafette CI

In order to release your applications, whether it's a Maven package to push to a repository, a Docker container to push to a registry or a deployment to Kubernetes, you can do this with Estafette CI by defining release targets in the `.estafette.yaml` file:

```yaml
...

releases:
  latest:
    stages:
      tag-container-image-as-latest:
        image: docker:18.03.1-ce
        env:
          DOCKER_HUB_USERNAME: estafette.secret(***)
          DOCKER_HUB_PASSWORD: estafette.secret(***)
        commands:
        - docker login --username=${DOCKER_HUB_USERNAME} --password="${DOCKER_HUB_PASSWORD}"
        - docker pull estafette/estafette-ci-api:${ESTAFETTE_BUILD_VERSION}
        - docker tag estafette/estafette-ci-api:${ESTAFETTE_BUILD_VERSION} estafette/estafette-ci-api:latest
        - docker push estafette/estafette-ci-api:${ESTAFETTE_BUILD_VERSION}
```

Currently these release targets can be triggered via the Slack integration; web gui and cli support will follow later. To trigger a release from Slack you use the following Slash command:

```
/estafette release <repo name> <release target> <version>
```

An example with the release sample as above would be:

```
/estafette release estafette-ci-api latest 0.0.21
```

If the repository name is not unique within your Estafette CI database you'll have to specificy the full repo path like:

```
/estafette release github.com/estafette/estafette-ci-api latest 0.0.21
```

### Use files from the git repository

By default - to speed things up - the release targets do not clone the git repository. If you need to have access to the files in your git repository you can do so by using the `git-clone` extension as the first stage:


```yaml
...

releases:
  latest:
    stages:
      git-clone:
        image: extensions/git-clone:dev

      tag-container-image-as-latest:
        ...
```

This brings in the last 50 revisions of the particular branch the version was built from.

If you need to be able to release older versions you can do a full clone by specifying property `shallow: false` on the git-clone stage:

```yaml
...

releases:
  latest:
    stages:
      git-clone:
        image: extensions/git-clone:dev
        shallow: false

      tag-container-image-as-latest:
        ...
```

### Rolling back a release

Currently there's no dedicated way to roll back a release except for just releasing the previous version again.

## Examples

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