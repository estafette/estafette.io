---
title: "Estafette CI extensions"
description: "Estafette comes bundled with a number of extensions that can handle complex logic on your behalf"
weight: 1
---

### extensions/git-clone

```yaml
git-clone:
  image: extensions/git-clone:stable
  shallow: < boolean | true >
```

The `git-clone` stage is automatically injected if not present in your manifest for the build stages; in the release stages it's not injected. In case you don't need anything from your repository to release this speeds things up.

The shallow clone - enabled by default - checks out the latest 50 commits for the particular branch, then selects the correct revision. When rebuilding an older version or releasing an older version this might not be enough, in that case explicitly set `shallow: false`.

### extensions/github-status

```yaml
set-build-status:
  image: extensions/github-status:stable
  status: < string | ESTAFETTE_BUILD_STATUS >
```

The `github-status` extension takes the build status from the `ESTAFETTE_BUILD_STATUS` environment variable, but this can be overridden with the `status` tag. Depending on the source of the git push - github or bitbucket - this extension is automatically injected at the beginning of the build stages and at the end in order to set the build status in the respective source code hosting system:

```yaml
set-pending-build-status:
  image: extensions/github-status:stable
  status: pending

...

set-build-status:
  image: extensions/github-status:stable
  when:
    status == 'succeeded' || 
    status == 'failed'
```

### extensions/bitbucket-status

```yaml
set-build-status:
  image: extensions/bitbucket-status:stable
  status: < string | ESTAFETTE_BUILD_STATUS >
```

The `bitbucket-status` extension takes the build status from the `ESTAFETTE_BUILD_STATUS` environment variable, but this can be overridden with the `status` tag. Depending on the source of the git push - github or bitbucket - this extension is automatically injected at the beginning of the build stages and at the end in order to set the build status in the respective source code hosting system:

```yaml
set-pending-build-status:
  image: extensions/bitbucket-status:stable
  status: pending

...

set-build-status:
  image: extensions/bitbucket-status:stable
  when:
    status == 'succeeded' || 
    status == 'failed'
```

### extensions/slack-build-status

```yaml
slack-notify:
  image: extensions/slack-build-status:stable
  webhook: estafette.secret(***)
  channels:
  - '#mychannel'
  when:
    status == 'succeeded' || 
    status == 'failed'
```

In order to send build notifications to one or more Slack channels the `slack-build-status` extension automatically sends a messages depending on the build status into the `channels` passed to this extension. The `webhook` needs a Slack webhook url in order to send the message.

Notes:
* This extension will be updated in the future to make use of the _credentials and trusted images_ configuration in the Estafette CI server so there's no need to embed a webhook in the manifest.