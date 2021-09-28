---
title: "Development"
description: "Development & release management"
weight: 4
---

### Introduction

_Estafette CI_ installation is always done using the Helm chart. This also has consequences for how you can contribute and test your changes. And finally how new versions are released.

### Development

The following components are used from the _subcharts_ of the `estafette-ci` chart. Together they form the _Estafette CI_ platform.

| Component                          | Purpose                                                                                    | Repository                                                    |
| ---------------------------------- | ------------------------------------------------------------------------------------------ | ------------------------------------------------------------- |
| `estafette-ci`                     | Umbrella Helm chart to install all of the following resources                              | https://github.com/estafette/estafette-ci                     |
| `estafette-ci-api`                 | API to handle web component requests, github/bitbucket webhooks, spawn pipeline jobs, etc  | https://github.com/estafette/estafette-ci-api                 |
| `estafette-ci-web`                 | Web user interface to see pipeline builds, releases, logs and trigger manual releases      | https://github.com/estafette/estafette-ci-web                 |
| `estafette-ci-db-migrator`         | Performs database schema migrations                                                        | https://github.com/estafette/estafette-ci-db-migrator         |
| `estafette-ci-cron-event-sender`   | Provides a cron event in order to be able to start pipelines with cron triggers            | https://github.com/estafette/estafette-ci-cron-event-sender   |
| `estafette-ci-hanging-job-cleaner` | Cancels hanging jobs, removes orphaned Kubernetes resources related to builds and releases | https://github.com/estafette/estafette-ci-hanging-job-cleaner |

When making changes to any of the components references from the subcharts of `estafette-ci` you can test them in 2 ways:

#### Override the image tag in the Helm values

Let's say you creating a new build of the `estafette-ci-api` component. Once it's built succesfully the commit will be tagged with the version number also used for the Docker container. See https://github.com/estafette/estafette-ci-api/tags and https://hub.docker.com/r/estafette/estafette-ci-api/tags?page=1&ordering=last_updated. 

Now if you want to test this with an `estafette-ci` installation you are already running you can set the following values to start using this particular Docker container:

```yaml
api:
  image:
    tag: '<version number to test>'
```

Applying the changes will have your setup start using this particular version of said component.

#### Update the subchart version and build the estafette-ci Helm chart

One step further, but with more automated testing to check whether all components get installed correctly you can update the subchart version.

For the `estafette-ci-api` component you would update the subchart version at https://github.com/estafette/estafette-ci/blob/main/helm/estafette-ci/charts/estafette-ci-api/Chart.yaml#L18 to

```yaml
version: '<version number to test>'
```

If you commit and push your changes it will do a complete build and tag the repository with the correct build version number, as can be seen at https://github.com/estafette/estafette-ci/tags.

You can now install this version of the `estafette-ci` chart to test out your changes.

### Releases

When you following the second method of testing changes by updating the `version` in the subchart for that component, you'll have to do the following to create a new release:

* Create one or multiple issues at https://github.com/estafette/estafette-ci/issues and link them to a milestone with the next version number up for release. You can see the previous milestones/releases at https://github.com/estafette/estafette-ci/milestones?state=closed or https://github.com/estafette/estafette-ci/releases. When creating an issue also set an assignee.
* After having tested and verified all to be resolved issues close each of the issues that are going into the next release
* Create a release branch with the version number of the next milestone/release (for example `1.0.29`) by running `git checkout -b 1.0.29` and push with `git push origin 1.0.29`
* Once this has been built succesfully with the version number being equal to the release branch and the milestone (so no `-main-***` suffix) you can release the build to the `release` _release target_ in the `github.com/estafette/estafette-ci` pipeline. This will create the _Github release_ with all the closed issues in the release notes. And it will purge any prior pre-release Helm chart versions. If you're an external contributor ask one of the maintainers to create the release for you.
* To be ready for a next release it's good to bump the version number in `.estafette.yaml` immediately. Do this by switching to the main branch with `git checkout main`, updating the `version` in the `.estafette.yaml` manifest as shown below. And then commit and push with `git commit -am "bump version to 1.0.30" && git push`.  

```yaml
version:
  semver:
    major: 1
    minor: 0
    patch: 30
    labelTemplate: '{{branch}}-{{auto}}'
    releaseBranch: 1.0.30
```
