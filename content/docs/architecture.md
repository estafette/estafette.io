---
title: "Architecture"
description: "Learn how Estafette is decomposed into small service and how they interact"
date: 2018-06-19T12:51:51+02:00
menu:
  main:
    parent: 'docs'
---

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