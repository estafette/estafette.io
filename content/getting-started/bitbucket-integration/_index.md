---
title: "Bitbucket integration"
description: "Setting up Estafette CI to build Bitbucket repositories"
weight: 6
---

In order to receive Bitbucket events in your Estafette CI installation you need to create Bitbucket app and install it in your workspace.

Ensure you have either [Github login]({{< relref "../github-login" >}}) or [Google login]({{< relref "../google-login" >}}) configured and set yourself as administrator.

## Configure Estafette for App Descriptor

In order for Estafette's Bitbucket App Descriptor endpoint to work ensure the Helm installation has the following `values.yaml` values file:

```yaml
api:
  ...
  config:
    files: |
      config.yaml: |
        integrations:
          bitbucket:
            enable: true

        apiServer:
          baseURL: https://<(private) host for the web gui>
          integrationsHost: https://<public host to receive webhooks>
```

or override the default config with environment variables like:

```yaml
api:
  baseHost: '<(private) host for the web gui>'
  integrationsHost: '<public host to receive webhooks>'
  deployment:
    extraEnv:
    - name: ESCI_INTEGRATIONS_BITBUCKET_ENABLE
      value: 'true'
```

With this in place run

```
helm upgrade --install estafette-ci estafette/estafette-ci -n estafette-ci --create-namespace --timeout 600s --values values.yaml
```

## Register App

Navigate to `Develop Apps` within your Bitbucket workspace at `https://bitbucket.org/<account>/workspace/settings/applications`. Once there click `Register app` and enter `https://<integrations host>/api/integrations/bitbucket/descriptor?key=<(private) host for the web gui>` for the _Descriptor URL`. This registers the application with Bitbucket.

## Installing the App

First go to the `Installed apps` page and check the `Enable development mode` checkbox. Then go back to `Develop apps` and click the _installation url_ as shown in the previously registered app and select the workspace you want to authorize it for.

## Viewing integrations

As an Estafette administrator go to the Admin > Integrations page to see all configured integrations.