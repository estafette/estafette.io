---
title: "Manifest"
description: "At the heart of Estafette is the manifest for your application to define build stages and releases"
weight: 1
---

Estafette is manifest-controlled. That means each build and stage is controlled via the manifest. No more fiddling in a GUI to set up a pipeline.

## Labels

Your manifest usually starts with a set of labels; these labels are useful to quickly filter pipelines in the GUI or api, but it also helps for reusing the same application name in your build or release steps.

In the manifest labels can be specified within the `labels` section; you have full freedom to come up with additional keys for your labels:

```yaml
labels:
  app: estafette-ci-builder
  team: estafette-team
  language: golang
```

The labels are available as `ESTAFETTE_LABEL_<UPPER_SNAKE_CASE_OF_LABEL_VALUE>` in your build and release stages.

## Build stages

On every `git push` to repositories in associated hosted git services Estafette receives a signal that there's new commits and starts the build stages as defined in the `stages` section.

For each stage you can define a public container image to be used or when you've configured private repositories in the Estafette server you can use images from those as well.

```yaml
stages:
  restore:
    image: node:10.9.0-alpine
    commands:
    - npm install

  unit-test:
    image: node:10.9.0-alpine
    commands:
    - npm run unit

  build:
    image: node:10.9.0-alpine
    commands:
    - npm run build

  bake:
    image: extensions/docker:stable
    action: build
    repositories:
    - estafette
    path: ./publish
    copy:
    - dist
    - favicon.ico
    - nginx.vh.default.conf

  push-to-docker-hub:
    image: extensions/docker:stable
    action: push
    repositories:
    - estafette
```

Notes:

* Estafette automatically injects a stage to clone the git repository.
* It's best practice to pin the versions of each image, instead of using `latest` to ensure your build still runs when you haven't touched in a long time.

## Releases

In Estafette release are triggered by a manual action via either the GUI or via Slack integration. Releases aren't just for deploying code, but can be used for any release action like pushing a Maven or Nuget package, tagging a docker container and more.

Within the `releases` section you can define one or multiple _release targets_ - for example development, staging and production - within which you can define stages in a similar fashion as in the build `stages` section.

###### A release to trigger a deployment

```yaml
releases:
  tooling:
    stages:
      deploy:
        image: extensions/gke:stable
        namespace: estafette
        visibility: public
        container:
          repository: estafette
        hosts:
        - ci.estafette.io
```

###### A release to promote a docker container to 'beta' or 'stable'

```yaml
releases:
  beta:
    stages:
      tag-container-image:
        image: extensions/docker:stable
        action: tag
        container: gke
        repositories:
        - extensions
        tags:
        - beta

  stable:
    stages:
      tag-container-image:
        image: extensions/docker:stable
        action: tag
        container: gke
        repositories:
        - extensions
        tags:
        - stable
        - latest
```

###### Release actions

For a slightly more advanced workflow you can define _actions_ on a release target, which can be separately triggered from the GUI for that particular _release target_. The triggered action is passed down to the release stage containers as environment variable `ESTAFETTE_RELEASE_ACTION`. The Estafette server itself is unaware of the meaning of the action at this moment, but the `extensions/gke` image is one of the extension that makes use of the following actions and exhibits different behaviour dependening on what action is triggered.

```yaml
releases:
  production:
    actions:
    - name: deploy-canary
    - name: deploy-stable
    - name: rollback-canary
    stages:
      deploy:
        image: extensions/gke:stable
```

Notes:

* There's currently no order in actions or dependencies between them so it's up to the user to trigger them in the correct order
* Unlike the build stages git clone doesn't happen by default in order to speed releases that don't require it up

To have the git-clone stage get automatically injected set the `clone: true` property on the stage target:

```yaml
releases:
  tooling:
    clone: true
    stages:
      deploy:
        image: extensions/gke:stable
        namespace: estafette
        visibility: public
        container:
          repository: estafette
        hosts:
        - ci.estafette.io
```

## A stage in detail

The individual stages of a build or release share nothing except for the cloned repository mounted into each stage container. Passing on information can be done via this working directory.

###### Image

With the `image` tag you can define which Docker container is used to execute this stage. You can use any public container or if you define private registries in the Estafette server those are available to use as well.

Notes:

* One of the goals of Estafette is to decouple individual application builds as much as possible by sharing nothing; no build agents with shared dependencies. So be cautious when creating builder images that are shared between many applications, because you might lose the advantage of being able to upgrade one application at a time to the latest and greatest version of your language of choice.

###### Shell

By default the shell used to execute `commands` is `/bin/sh`. With the `shell` tag you can override this to for example `/bin/bash`.

###### Working directory

The cloned git repository is mounted into the `/estafette-work` directory by default. In some cases you want to be able to override this. For example complex golang applications with nested packages need to be build within a certain directory structure.

You can set an override like this:

```yaml
workDir: /go/src/github.com/estafette/estafette-ci-api
```

Or by reusing a possible `app` (or other) label you've defined in the `labels` section this looks like:

```yaml
workDir: /go/src/github.com/estafette/${ESTAFETTE_LABEL_APP}
```

###### Commands

Unless you make use of extensions it's likely that you'll pass `commands` to the stage image to execute certain behaviour. You can define one or more commands which are executed sequentially:

```yaml
restore:
  image: microsoft/dotnet:2.1-sdk
  commands:
  - dotnet restore --packages .nuget/packages

build:
  image: microsoft/dotnet:2.1-sdk
  commands:
  - dotnet build --version-suffix ${ESTAFETTE_BUILD_VERSION_PATCH}
```

You could also chose to combine these stages into a single stage like this:

```yaml
restore:
  image: microsoft/dotnet:2.1-sdk
  commands:
  - dotnet restore
  - dotnet build --version-suffix ${ESTAFETTE_BUILD_VERSION_PATCH}
```

A compelling reason to split things up into multiple stages is to get individual timings on the various stages which you can use to find the best opportunity to improve performance in your build pipeline.

###### Environment variables

To pass environment variables into a build/release stage you can use the `env` tag to specify one or more environment variables in the image. 

```yaml
build:
  image: golang:1.11.1-alpine3.8
  workDir: /go/src/github.com/estafette/${ESTAFETTE_LABEL_APP}
  env:
    CGO_ENABLED: 0
    GOOS: linux
  commands:
  - go test `go list ./... | grep -v /vendor/`
```

Note:
* The globally set Estafette environment variables are automatically injected into each stage container, so you don't have to do that explicitly.

###### When clause

To control execution of each individual step you can use complex conditions - in golang notation - to control whether the stage gets executed or not.

By default a stage only executes if all stages before it have been executed succesfully. The implicit value of the `when` tag looks like this:

```yaml
when:
  status == 'succeeded'
```

To ensure that for example you only push Docker containers for a specific branch you can use the following condition:

```yaml
when:
  status == 'succeeded' &&
  branch == 'master'
```


Do note that you have to repeat the `status == 'succeeded'`; we don't prepend this by default to your when clause so you can actually trigger stages on failure as well; very useful for notifications in case of failure.

```yaml
when:
  status == 'failed'
```

###### Retries

If you have flaky stages you obviously want to improve what happens in that stage to be more reliable; but a quick stop-gap to at least retry the full stage in case of failure can be to set a number of retries before it's seen as a real failure with the `retries` tag.

```yaml
retries: 3
```

###### Custom properties

When you set unrecognized keywords - none of the above - in a stage Estafette automatically sees those as custom properties which are injected into the stages as environment variables; this is useful to create extensions with a friendlier set of parameters to control the behaviour.

In case a parameter is of type string, boolean, integer of an array of strings they are injected into the container as an environment variable with name `ESTAFETTE_EXTENSION_<UPPER_SNAKE_CASE_OF_THE_TAG_NAME>`.

All of the custom properties together are also json serialized and injected with environment variable name `ESTAFETTE_EXTENSION_CUSTOM_PROPERTIES`. This way you can use more complex and nested structures to deserialize within your extension.

## Environment variables

Besides the environment variables defined at each individual stage you can also set global environment variables with top-level section `env`:

```yaml
env:
  VAR_A: Greetings
  VAR_B: World
```

These are also automatically injected into each individual stage container; if the same environment variable name is used at the stage level it overrides the global one.

## Secret values

In order to be able to use secret values in your various steps the Estafette server is configured with an AES key of 32 bytes in order to use AES-256 to encrypt secrets.

Currently values can be encrypted via the Slack integration with Slash command `/estafette encrypt <your secret value>`.

This returns a string of the form `estafette.secret(***)` which can be used as value for your environment variables or custom properties.

## Versioning

By default Estafette uses semantic versioning with the following values:

```yaml
version:
  semver:
    major: 0
    minor: 0
    patch: '{{auto}}'
    labelTemplate: '{{branch}}'
    releaseBranch: master
```

The `patch` value is generated from a constantly increasing integer per repository, which increments across all branches, not per branch. This ensures it's unique. When the actual branch is different from the one specified in the `releaseBranch` tag the `labelTemplate` is appended to the build counter resulting in a version number like `0.0.531-my-pr-branch`. For the release branch this will result in a version number like `0.0.532`.

## Builder parameters

In order to dogfood Estafette's own components before they're promoted to stable version there's the possibility to set the track to `dev`, `beta` or the default value of `stable`. This controls which version of the builder container is used to execute build steps.

```
builder:
  track: stable
```

Notes:

* It's advised to always stay on the stable track to avoid flaky behaviour because a bad version made it into `dev`.

## Global Estafette environment variables

The following Estafette specific global variables are injected automatically into all stage containers to be used at your own disposal:

| Environment variable                  | Description |
|---------------------------------------|-------------|
| `ESTAFETTE_GIT_SOURCE`                  | Indicates the hosted source code system; github.com or bitbucket.org |
| `ESTAFETTE_GIT_OWNER`                   | The owner of the repository; in github.com/estafette/estafette-ci-api the middle value `estafette` is the owner |
| `ESTAFETTE_GIT_NAME`                    | The name of the repository;  in github.com/estafette/estafette-ci-api the last value `estafette-ci-api` is the name |
| `ESTAFETTE_GIT_FULLNAME`                | The full name is the owner plus repository name, for example `estafette/estafette-ci-api` |
| `ESTAFETTE_GIT_REVISION`                | The git sha-1 hash for a commit; it's the hash for the latest commit within a push |
| `ESTAFETTE_GIT_BRANCH`                  | The git branch being built or released |
| `ESTAFETTE_BUILD_DATETIME`              | This is the UTC time in RFC3339 format that the build started |
| `ESTAFETTE_BUILD_VERSION`               | The full version number generated by Estafette and the `version` section in the manifest |
| `ESTAFETTE_BUILD_VERSION_MAJOR`         | If semantic versioning is used this holds the major version |
| `ESTAFETTE_BUILD_VERSION_MINOR`         | If semantic versioning is used this holds the minor version |
| `ESTAFETTE_BUILD_VERSION_PATCH`         | If semantic versioning is used this has the patch version |
| `ESTAFETTE_BUILD_STATUS`                | Starts out as `succeeded` but changes to `failed` as soon as a build/release stage fails |
| `ESTAFETTE_LABEL_...`                   | All items in the `labels` section are turned into an environment variable of this form; the label key is turned into upper snake case |
| `ESTAFETTE_EXTENSION_...`               | All custom properties of type string, boolean, integer or array of strings are turned into an environment variable of this form; the tag is turned into upper snake case |
| `ESTAFETTE_EXTENSION_CUSTOM_PROPERTIES` | All custom properties of a stage are json serialized and injected into the stage container with this environment variable |

Notes:

* Don't set any variables starting with `ESTAFETTE_` yourself, they will be wiped or skipped.

## Other

To find out more about the Estafette extensions and how they can be used at the [extensions][] documentation.

[extensions]: /usage/extensions/
