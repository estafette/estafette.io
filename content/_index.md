---
title: "Estafette CI/CD"
subtitle: "The resilient and cloud-native CI/CD"
---

**Hi, welcome to the Estafette CI/CD site. Here you will find all documentation on how to get started with Estafette, how to use it for simple use cases and more advanced ones.**

## Goals

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

## Estafette manifest

```yaml
labels:
  app: estafette-ci-web
  team: estafette-team
  language: node

pipelines:
  restore:
    image: node:10.4.0-alpine
    commands:
    - npm install

  unit-test:
    image: node:10.4.0-alpine
    commands:
    - npm run unit

  build:
    image: node:10.4.0-alpine
    commands:
    - npm run build
```