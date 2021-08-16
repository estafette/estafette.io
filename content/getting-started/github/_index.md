---
title: "Github integration"
description: "Setting up Estafette CI to build Github repositories"
weight: 2
---

In order to receive Github events in your Estafette CI installation you need to create a Github App and Installation.

## Register a Github App

Depending on whether you're using a _personal_ Github account or an _organizational_ account navigate to `https://github.com/settings/apps/new` for a personal account or `https://github.com/organizations/<organization>/settings/apps/new` for your organization to register a Github App for Estafette CI.

Set the following values while registering the Github App:

* _Github App Name_ - Set to the hostname you're planning to host Estafette CI at. 
* _Homepage Url_ - The same hostname, but ensure it's prefixed by `https://`. 
* _Webhook URL_ - Has to be set to `https://<webhook host>/api/integrations/github/events`. This separate hostname is open to the world for receiving webhooks, while your regular host is preferably behind an identity-aware access solutino.
* _Webhook secret_ - Create a secret following these [instructions](https://docs.github.com/en/developers/webhooks-and-events/webhooks/securing-your-webhooks). Store it's value in a secure location like a password manager. You'll need it to configure Estafette.
* _Where can this GitHub App be installed?_ - Only on this account

### Repository permissions

To ensure the Github API can be used to retrieve necessary information for build & releases to work ensure the following permissions are set:

* _Contents_ - read-only
* _Metadata_ - read-only

### Subscribe to events

And to be informed about events to start builds and initiate other important actions subscribe to the following events:

* Repository
* Push

## Update the Github App

After creating the Github App you want to go through some extra steps to ensure Estafette can communicate with the Github API.

- _Generate a new client secret_ - Copy the value and store it in a secure location like a password manager. You'll need it to configure Estafette.
- _Generate a private key_ - Used to sign access token requests. Store the _pem_ file in a secure location like a password manager. Again needed for configuring Estafette.

## Install the Github App

Go to the _Install App_ tab and install it in the visible account (organization or personal). Select _All repositories_ to make it possible for anyone creating a new reposity to be able to create a pipeline in the Estafette installation.

## Configure Estafette

Update the Helm installation with the following `estafette-ci.yaml` values file by running `helm upgrade --install estafette-ci estafette/estafette-ci -n estafette-ci --timeout 600s --values estafette.ci-yaml`:

```yaml
api:
  secret:
    files:
      private-key.pem: |
        -----BEGIN RSA PRIVATE KEY-----
        <the github private key downloaded before>
        -----END RSA PRIVATE KEY-----

  config:
    files: |
      config.yaml: |
        integrations:
          github:
            enable: true
            appID: <App ID>
            clientID: <Client ID>
            clientSecret: <Client secret recorded before>
            webhookSecret: <Webhook secret recorded before>
```
