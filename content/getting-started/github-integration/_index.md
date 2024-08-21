---
title: "GitHub integration"
description: "Setting up Estafette CI to build GitHub repositories"
weight: 7
---

In order to receive GitHub events in your Estafette CI installation you need enable the GitHub integration in Estafette, then you can create a GitHub app and install it.

Ensure you have either [GitHub login]({{< relref "../github-login" >}}) or [Google login]({{< relref "../google-login" >}}) configured and set yourself as administrator.

NOTE: GitHub login and GitHub integration aren't the same thing. They use different types of GitHub apps behind the scenes. You can mix and match GitHub login with other types of integration and vice versa.

## Configure Estafette

In order for Estafette's GitHub integration to work ensure the Helm installation has the following `values.yaml` values file:

```yaml
api:
  ...
  config:
    files:
      config.yaml: |
        integrations:
          github:
            enable: true
        apiServer:
          baseURL: 'https://<(private) host for the web gui>'
          integrationsHost: 'https://<public host to receive webhooks>'
```

or override the default config with environment variables like:

```yaml
api:
  baseHost: '<(private) host for the web gui>'
  integrationsHost: '<public host to receive webhooks>'
  deployment:
    extraEnv:
    - name: ESCI_INTEGRATIONS_GITHUB_ENABLE
      value: 'true'
```

With this in place run

```
helm upgrade --install estafette-ci estafette/estafette-ci -n estafette-ci --create-namespace --timeout 600s --values values.yaml
```

## Register App

As an Estafette administrator go to the Admin > Integrations page and click _Register GitHub App for a personal account_. Ensure you rename the app to something unique - the name is global to GitHub - and meaningful to you.

The _Webhook URL_ must be a public URL that GitHub can reach. For testing purposes, you can use [webhookrelay](https://docs.webhookrelay.com/) to forward requests to internal. Please refer [delivering-webhooks-to-private-systems](https://docs.github.com/en/webhooks/using-webhooks/delivering-webhooks-to-private-systems) for more solutions.

## Installing the App

Registering the app will take you back to the Admin > Integrations page. Click _Install GitHub App_ to install the app and have events get sent to Estafette.

## Viewing integrations

As an Estafette administrator go to the Admin > Integrations page to see all configured integrations.

## Stored details

The details for the integrations are stored in the `estafette-ci-api.github` configmap. They do not get backed up in any way, so after disaster recovery you will have to reconfigure integrations.
