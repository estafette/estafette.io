---
title: "Best practices"
description: "The best practices when using Estafette"
weight: 3
---

### Pin image tags

In order to keep your builds reproducible, even if you haven't touched them for a year it's best to pin the stage image tags as narrowly as possible (or at least major and minor to avoid adopting breaking changes). This will ensure you're not wasting a day figuring out why your build is suddenly broken, when it turns out it's the latest image tag of one of the used images is actually a different image now.

{{< dont >}}

```yaml
stages:
  build:
    image: golang
```

```yaml
stages:
  build:
    image: golang:latest
```

{{< /dont >}}

{{< do >}}

```yaml
stages:
  build:
    image: golang:1.11.2-alpine3.8
```

{{< /do >}}

### Use commands instead of build scripts

One of the strengths of _Estafette_'s manifest is that you can immediately see which commands have been issued and can try those on your own machine. This particularly comes in handy when your build breaks. If the actual commands are hidden in a build script you have to navigate from repository to repository to find which commands are actually executed and it means more work to fix things in that case.

{{< dont >}}

```yaml
stages:
  build:
    image: microsoft/dotnet:2.1-sdk
    commands:
    - build-script.sh
```

{{< /dont >}}

{{< do >}}

```yaml
stages:
    image: microsoft/dotnet:2.1-sdk
    commands:
    - dotnet build --configuration Release /p:Version=${ESTAFETTE_BUILD_VERSION} --no-restore
```

{{< /do >}}

### Share as little as possible between applications

Although it's tempting to build your own fat _builder_ images to run your steps (particularly to improve speed), before you know it a dozen applications rely on this particular image. When you by accident break the image all the builds using it can no longer build. Or you remove an installed depedency because your particular build no longer use it, turns out a bunch of other applications actually do use it.

{{< dont >}}

```yaml
stages:
  build:
    image: myrepo/myown-cloud-sdk
    commands:
    - ...
```

{{< /dont >}}

{{< do >}}

```yaml
stages:
    image: google/cloud-sdk:223.0.0-alpine
    commands:
    - apk add gettext
    - gcloud commands install kubectl
    - ...
```

{{< /do >}}

### Avoid using Estafette's dev or beta tags

_Estafette_ dogfoods it's own components, pushing new versions to the `dev` tag first. Once it's confirmed to be functional it gets promoted to beta, then to stable. Because the dev tag of any extension can be broken at any time avoid using it and use the `stable` tag instead.

{{< dont >}}

```yaml
stages:
  build:
    image: extensions/docker:dev
```

{{< /dont >}}

{{< do >}}

```yaml
stages:
  build:
    image: extensions/docker:stable
```

{{< /do >}}


### Avoid using Estafette's builder dev track

_Estafette_ dogfoods it's own components, pushing new versions to the `dev` track first. Once it's confirmed to be functional it gets promoted to beta, then to stable. Because the dev track of the builder itself can be broken at any time avoid using it and use the `stable` track instead.

{{< dont >}}

```yaml
builder:
  track: dev
```

{{< /dont >}}

{{< do >}}

```yaml
builder:
  track: stable
```

Or drop the `builder` section alltogether, it defaults to the stable track.

{{< /do >}}

### Use pipeline restricted secrets instead of global secrets

In Estafette you can create _global_ secrets which can be decrypted by any pipeline and _restricted_ secrets which can only be used in a single pipeline. In your `.estafette.yaml` manifest it's best to only use pipeline _restricted_ secrets.

You can distinguish between the two types by the number of dots in the `estafette.secret(...)` envelope. A _global_ secret use one dot to separate the nonce from the encrypted value; a _restricted_ secret has two dots separating the nonce, the encrypted value and the pipeline regex.

{{< dont >}}

```yaml
env:
  MY_GLOBAL_SECRET: estafette.secret(deFTz5Bdjg6SUe29.oPIkXbze5G9PNEWS2-ZnArl8BCqHnx4MdTdxHg37th9u)
```

Global secrets were originally the only type of secret, but now they should only be used in Estafette's centrally stored credentials, which can then be reused by any pipeline using trusted images that get those credentials injected.

{{< /dont >}}

{{< do >}}

```yaml
env:
  MY_PIPELINE_RESTRICTED_SECRET: estafette.secret(7pB-Znp16my5l-Gz.l--UakUaK5N8KYFt-sVNUaOY5uobSpWabJNVXYDEyDWT.hO6JcRARdtB-PY577NJeUrKMVOx-sjg617wTd8IkAh-PvIm9exuATeDeFiYaEr9eQtfreBQ=)
```

When replacing a _global_ secret with a _restricted_ secret make sure to rotate the encrypted value itself - i.e. database password, api key, etc - in order to prevent anyone from using the former _global_ secret to gain access to the protected system. The _restricted_ secret can be generated in the pipeline's secrets tab.

{{< /do >}}