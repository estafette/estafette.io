---
title: "Bitbucket integration"
description: "Setting up Estafette CI to build Bitbucket repositories"
weight: 3
---

In order to receive Bitbucket events in your Estafette CI installation you need to create Bitbucket app and install it in your workspace.

## Configure Estafette for App Descriptor

In order for Estafette's Bitbucket App Descriptor endpoint to work ensure the Helm installation has the following `estafette-ci.yaml` values file:

```yaml
api:
  config:
    files: |
      config.yaml: |
        integrations:
          bitbucket:
            enable: true
            key: <unique text string to identify the app>
            name: <text string with the name of the app for display>

        apiServer:
          integrationsURL: https://<integrations host>
```

With this in place run `helm upgrade --install estafette-ci estafette/estafette-ci -n estafette-ci --timeout 600s --values estafette.ci-yaml`.

## Register App

Navigate to `Develop Apps` within your Bitbucket workspace at https://bitbucket.org/<account>/workspace/settings/applications. Once there click `Register app` and enter `https://<integrations host>/api/integrations/bitbucket/descriptor` for the _Descriptor URL`. This registers the application with Bitbucket.

## Installing the App

Click the _installation url_ as shown in the registered app and select the workspace you want to authorize it for.

## Configure Estafette for registered App

Now that the App has been registered with Bitbucket update the Estafette config by setting `appClientID` and `appClientSecret` from the values shown in Bitbucket.


```yaml
api:
  config:
    files: |
      config.yaml: |
        integrations:
          bitbucket:
            enable: true
            key: <unique text string to identify the app>
            name: <text string with the name of the app for display>
            appClientID: <registered app client id>
            appClientSecret: <registered app client secret>

        apiServer:
          integrationsURL: https://<integrations host>
```