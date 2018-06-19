---
title: "Usage"
description: "Write the manifest to get your application built"
date: 2018-06-19T12:51:45+02:00
menu:
  main:
    parent: 'docs'
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