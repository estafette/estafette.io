---
title: "Github login"
description: "Setting up Estafette CI to login with a Github account"
weight: 4
---

In order to be able to take manual actions in Estafette you need to protect it with a login.

NOTE: Github login and Github integration aren't the same thing. They use different types of Github apps behind the scenes. You can mix and match Github login with other types of integration and vice versa.

## Configure Estafette

To log in to Estafette using Github login first create a new OAuth application in Github at `https://github.com/settings/applications/new` for a personal account or at `https://github.com/organizations/<organization>/settings/applications/new` for an organizational account. Fill in the form with the following values:

![Register OAuth application](/getting-started/github-login/register-github-oauth-application.png)

After creating the application you'll be able to see the _Client ID_; you'll also need a _Client Secret_. Obtain one by clicking _Generate new client secret_. Armed with these details you can update the values used by the Helm installation by updating your `values.yaml` values file:

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
          github:
            clientID: '<github oauth app client id>'
            clientSecret: '<github oauth app client secret>'
            allowedIdentitiesRegex: '<regex to restrict domain for the email address of the user; for example .+@estafette\.io>'
          administrators:
          - admin1@estafette.io
          - admin2@estafette.io
```

or override the default config with environment variables like:

```yaml
api:
  baseHost: '<(private) host for the web gui>'
  integrationsHost: '<public host to receive webhooks>'
  deployment:
    extraEnv:
    - name: ESCI_AUTH_GITHUB_CLIENTID
      value: '<github oauth app client id>'
    - name: ESCI_AUTH_GITHUB_CLIENTSECRET
      value: '<github oauth app client secret>'
    - name: ESCI_AUTH_GITHUB_ALLOWEDIDENTITIESREGEX
      value: '<regex to restrict domain for the email address of the user; for example .+@estafette\.io>'
    - name: ESCI_AUTH_ADMINISTRATORS
      value: '<comma separate list of admin email addresses, for example admin1@estafette.io,admin2@estafette.io>'
```

With this in place run

```
helm upgrade --install estafette-ci estafette/estafette-ci -n estafette-ci --create-namespace --timeout 600s --values values.yaml
```

Now when navigating to your base host you should be able to see a Github login button and use it to log in to your Estafette setup.