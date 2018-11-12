---
title: "Goals"
description: "Overarching goals of the Estafette CI system"
weight: 1
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