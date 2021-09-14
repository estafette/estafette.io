---
title: "Github integration"
description: "Setting up Estafette CI to build Github repositories"
weight: 3
---

In order to receive Github events in your Estafette CI installation you need enable the Github integration in Estafette, then you can create a Github app and install it.

## Configure Estafette

In order for Estafette's Github integration to work ensure the Helm installation has the following `values.yaml` values file:

```yaml
api:
  ...
  config:
    files: |
      config.yaml: |
        integrations:
          github:
            enable: true

        apiServer:
          baseURL: https://<(private) host for the web gui>
          integrationsHost: https://<public host to receive webhooks>
```

or override the default config with environment variables like:

```yaml
api:
  baseHost: <(private) host for the web gui>
  integrationsHost: <public host to receive webhooks>
  deployment:
    extraEnv:
    - name: ESCI_INTEGRATIONS_GITHUB_ENABLE
      value: 'true'
```

With this in place run `helm upgrade --install estafette-ci estafette/estafette-ci -n estafette-ci --timeout 600s --values values.yaml`.

## Register App

As an Estafette administrator go to the Admin > Integrations page and click _Register Github App for a personal account_. Ensure you rename the app to something unique - the name is global to Github - and meaningful to you.

## Installing the App

Registering the app will take you back to the Admin > Integrations page. Click _Install Github App_ to install the app and have events get sent to Estafette.

## Viewing integrations

As an Estafette administrator go to the Admin > Integrations page to see all configured integrations.