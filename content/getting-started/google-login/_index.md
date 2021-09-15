---
title: "Google login"
description: "Setting up Estafette CI to login with a Google account"
weight: 3
---

In order to be able to take manual actions in Estafette you need to protect it with a login.

## Configure Estafette

To log in to Estafette using Google login first set up the _OAuth consent screen_ in one of your Google Cloud projects at `https://console.cloud.google.com/apis/credentials/consent?project=<gcp project id>`.

Once that's done you can go to `https://console.cloud.google.com/apis/credentials?project=<gcp project id>` and click _Create credentials_ and select _OAuth client ID_. Fill in the form with the following values:

![Register OAuth client ID](/getting-started/google-login/create-oauth-client-id.png)

After creating the application you'll be able to see the _Client ID_ and _Client secret_. Armed with these details you can update the values used by the Helm installation by updating your `values.yaml` values file:

```yaml
api:
  ...
  config:
    files: |
      config.yaml: |
        apiServer:
          baseURL: https://<(private) host for the web gui>
          integrationsHost: https://<public host to receive webhooks>
        
        auth:
          google:
            clientID: <google oauth client id>
            clientSecret: <google oauth client secret>
            allowedIdentitiesRegex: <regex to restrict domain for the email address of the user; for example .+@estafette\.io>
```

or override the default config with environment variables like:

```yaml
api:
  baseHost: <(private) host for the web gui>
  integrationsHost: <public host to receive webhooks>
  deployment:
    extraEnv:
    - name: ESCI_AUTH_GOOGLE_CLIENTID
      value: '<google oauth client id>'
    - name: ESCI_AUTH_GOOGLE_CLIENTSECRET
      value: '<google oauth client secret>'
    - name: ESCI_AUTH_GOOGLE_ALLOWEDIDENTITIESREGEX
      value: '<regex to restrict domain for the email address of the user; for example .+@estafette\.io>'
```

With this in place run `helm upgrade --install estafette-ci estafette/estafette-ci -n estafette-ci --timeout 600s --values values.yaml`.

Now when navigating to your base host you should be able to see a Google login button and use it to log in to your Estafette setup.