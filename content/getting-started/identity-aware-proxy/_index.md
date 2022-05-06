---
title: "Identity Aware Proxy (IAP)"
description: "Configure the web interface to run behind IAP"
weight: 8
---

For using _Estafette CI_ at your company you probably want to keep the web interface to be private and only accessible to your employees. One way to do that is to run it behind _Identity Aware Proxy_ (IAP) if you're on Google Cloud. This makes it accessible based on the user's identity, instead of relying on a less secure or less friendly option like VPN.

## Creating OAuth credentials

You might have already followed the instructions on how to set up [Google login]({{< relref "../google-login" >}}), otherwise do that first. Once done go back to the credentials page in the Google Cloud Console.

There you have to add an additional _Authorized redirect URI_ of the form:

```
https://iap.googleapis.com/v1/oauth/clientIds/<client id>:handleRedirect
```

Where you'll use the _Client ID_ visible on the same page for the value of `<client id>`.

## Configure Helm values

```yaml
api:
  iap:
    enabled: true
    clientId: '***.apps.googleusercontent.com'
    clientSecret: '***'
api:
  web:
    enabled: true
    clientId: '***.apps.googleusercontent.com'
    clientSecret: '***'
```

## Permissions for IAP

After applying the new values you'll have to set up the access rule for the _Identity Aware Proxy_; You can do so in the _Google Cloud Console_ at https://console.cloud.google.com/security/iap.

First select the `estafette-ci/estafette-ci-api` backend service and click _Add principal_ in the right info pane. You want to add either your organization or groups, rather than individuals to automatically have new joiners be able to access the system. Add them in the _new principals_ field and then select role _Cloud IAP > IAP-Secured Web App User_. Repeat this for the `estafette-ci/estafette-ci-web` backend service. Now it's a matter of minutes before the base host address of the _Estafette CI_ installation is accessible through IAP.

## Cockroachdb Admin UI

In order to be able to access the the CockroachDB admin interface you might want to set up IAP as well. Best to create a dedicated set of OAuth credentials for this, because you'll access cockroachdb on another host and want a more restricted audience to be able to connect through IAP.

Create the credentials at `https://console.cloud.google.com/apis/credentials?project=<gcp project id>` and click _Create credentials_ and select _OAuth client ID_. Fill in the form with the following values:

* Application Type - Web application
* Name - hostname for cockroachdb admin interface, for example `cockroachdb-estafette.mydomain.com`
* Authorized Javascript origins, for example `https://cockroachdb-estafette.mydomain.com`

After creating it it will have a _Client ID_ and you can add

* Authorized redirect URI - `https://iap.googleapis.com/v1/oauth/clientIds/<client id>:handleRedirect`

With this in place you can update the values for the Helm chart:

```yaml
db:
  ingress:
    enabled: true
    annotations: {}
      # cert-manager.io/cluster-issuer: nameOfClusterIssuer
    paths: ['/*']
    hosts:
    - cockroachdb-estafette.mydomain.com
    tls:
    - hosts: [cockroachdb-estafette.mydomain.com]
      secretName: estafette-ci-db.tls
  iap:
    enabled: true
    clientId: '***.apps.googleusercontent.com'
    clientSecret: '***'
```

You'll have to specify the correct annotation depending on which certificate provisioning method you are using.

After applying the changes you'll have to go into the IAP console page and set permissions for backend service `estafette-ci/estafette-ci-db-public`.
