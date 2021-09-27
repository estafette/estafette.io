---
title: "Fixing vulnerabilities"
description: "Fixing vulnerabilities in your applications"
weight: 3
---

## Detect vulnerabilities at build time

When building a container with the `extensions/docker` image it checks for vulnerabilities automatically and fails on ones with `CRITICAL` severity.

### Upgrade your base image

Usually your best of upgrading the image you build `FROM` to the latest one available. Those are most likely to have their vulnerabilities addressed. And it likely does so in a smaller image.

However that's not always the case - that the latest versions have no vulnerabilities - or this forces you to simultaneously upgrade your programming language SDK which you might not want at this time or slows you down in addressing vulnerabilities in your already running containers.

### Update packages in the current image

What you can try to do if the above doesn't work or isn't an option for you is to upgrade the installed applications with vulnerabilities in your container image. Below you can see how to do that for different operating systems:

#### Debian / Ubuntu

Either you can upgrade specific packages with:

```
RUN apt-get update && \
    apt-get upgrade -y \
      libc-bin \
      libc6 \
      multiarch-support && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

You can also try - but your mileage may vary - to upgrade all applications that have security updates with the following step:

```
RUN apt-get update && \
    cat /etc/apt/sources.list | grep security > /etc/apt/sources.security.only.list && \
    apt-get upgrade -y -o Dir::Etc::SourceList=/etc/apt/sources.security.only.list && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

Unfortunately this last approach doesn't always seem to apply all the required updates.

Be aware the upgrading packages in your image can also grow the size of the image quite a bit, so if you can upgrade to a newer base image.

#### Alpine

For Alpine you'll have to upgrade individual packages with:

```
RUN apk add --update --upgrade --no-cache \
      openssl \
    && rm -rf /var/cache/apk/*
```

Again, upgrading packages grows the size of your image so is possible upgrade the base image instead.

## Detect vulnerabilities at runtime

Estafette's [vulnerability scanner](https://github.com/estafette/estafette-vulnerability-scanner) can be dropped in your Kubernetes cluster to scan all the images referenced by _deployments_, _statefulsets_, _daemonsets_, _jobs_, _cronjobs_ and stand-alone _pods_. 

This will help you become aware of vulnerabilities discovered after you've built and deployed your applications so you can address in your application and release the fix.