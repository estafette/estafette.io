---
title: "Architecture"
description: "Of what parts does the Estafette system consist"
weight: 2
---

The high-level architecture of Estafette CI looks as follows:

![Architecture overview](/design/architecture.png)

## Components

#### API

In Estafette CI the api layer is at the heart of the system. It receives events from integrations from the build and release jobs and some other components. The API itself is stateless and runs multiple pods for high availability. It can be upgraded without downtime.

#### Web

The web frontend is a single page application written in VueJS interfacing with the API for it's content.

#### Build / release jobs

When a build or release is started due to an event being received by the API it is the API itself that creates a build job via the Kubernetes API. It uses the _Job_ resource type, so Kubernetes takes care of jobs failing due to nodes getting deleted. Inside the job the _estafette-ci-builder_ runs the stages defined in the `.estafette.yaml` manifest. Any failure in running the stages is reported to the API, but the Kubernetes job itself still succeeds so it doesn't get restarted.

The job can run to completion independently of API upgrades happening in the meantime making it a resilient system that can be upgraded at all times.

When the _estafette-ci-builder_ gets updated in the mean time it doesn't affect running builds, only new ones started after is has been updated.

#### Database

For being able to scale and perform well Estafette is built on top of the cloud-native distributed SQL database [CockroachDB](https://www.cockroachlabs.com/). It can scale along with the number of pipelines, build jobs and release jobs running on your setup, so it can easily handle hundreds of pipelines and tens of thousands of builds without slowing down.

It also makes it possible to upgrade the database itself without any downtime, which is an important feature for Estafette, to always be online.

Currently the most important data stored in the database are the autoincrementing build number per pipeline and all the build and release logs.

#### Queue

In order to decouple parts of the logic cloud-native queue [NATS](https://nats.io/) is used. It's a very performant system to route messages between various parts of Estafette CI. It's currently used as queue - while it also supports _publish-subscribe_ and other concepts - to handle various triggers for Estafette pipelines.

#### Cron timer

This component uses a Kubernetes CronJob to send a _cron event_ message through the queue towards the the API in order to handle _cron triggers_ while keeping the API layer stateless.

## Languages

All Estafette components - except for the web UI - are built in golang, to make use of it's low cpu and memory usage, it's speed of starting up new containers and it's ease of building it for many platforms.