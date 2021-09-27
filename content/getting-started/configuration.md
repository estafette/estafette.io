---
title: Other configuration
description: Configure Estafette to tune it to your needs
weight: 20
---

## Config.yaml

While _Estafette_'s goal is to keep as much control as possible with each individual application build pipeline through the `.estafette.yaml` manifest, unavoidably there's some configuration that needs to be done centrally. This configuration is stored in the `config.yaml` file. Below is a description of the various configuration sections.

## Integrations

For integrations with 3rd party services the `integrations` section provides a place for configuration for each individual service.

### Integrations > Github

See [Github integration]({{< relref "github-integration" >}}).

Through `config.yaml`:

```yaml
integrations:
  github:
    enable: true
```

Or via Helm `values.yaml`:

```yaml
api:
  deployment:
    extraEnv:
      - name: ESCI_INTEGRATIONS_GITHUB_ENABLE
        value: 'true'
```

### Integrations > Bitbucket

See [Bitbucket integration]({{< relref "bitbucket-integration" >}}).

Through `config.yaml`:

```yaml
integrations:
  bitbucket:
    enable: true
```

Or via Helm `values.yaml`:

```yaml
api:
  deployment:
    extraEnv:
      - name: ESCI_INTEGRATIONS_BITBUCKET_ENABLE
        value: 'true'
```

### Integrations > Slack

The Slack integration allows a Slash command to be configured in order to provide functionalities like encrypting a secret or triggering a release directly from Slack. For the Slash command to integrate with _Estafette_ the following configuration section is needed.

Through `config.yaml`:

```yaml
integrations:
  slack:
    enable: true
    clientID: '<slack client id>'
    clientSecret: '<slack client secret>'
    appVerificationToken: '<slack app verification token>'
    appOAuthAccessToken: '<slack app oauth access token>'
```

Or via Helm `values.yaml`:

```yaml
api:
  deployment:
    extraEnv:
      - name: ESCI_INTEGRATIONS_SLACK_ENABLE
        value: 'true'
      - name: ESCI_INTEGRATIONS_SLACK_CLIENTID
        value: '<slack client id>'
      - name: ESCI_INTEGRATIONS_SLACK_CLIENTSECRET
        value: '<slack client secret>'
      - name: ESCI_INTEGRATIONS_SLACK_APPVERIFICATIONTOKEN
        value: '<slack app verification token>'
      - name: ESCI_INTEGRATIONS_SLACK_APPOAUTHACCESSTOKEN
        value: '<slack app oauth access token>'
```

### Integrations > Pub/sub

In order for the Estafette CI api to create subscriptions to topics used in _pubsub_ triggers and validate incoming events the following config section needs to be added.

Through `config.yaml`:

```yaml
integrations:
  pubsub:
    enable: true
    defaultProject: '<project that runs estafette>'
    endpoint: 'https://<integrations host>/api/integrations/pubsub/events'
    audience: beautiful-butterfly
    serviceAccountEmail: '<email address for service account used by estafette-ci-api>'
    subscriptionNameSuffix: '~estafette-ci-pubsub-trigger'
    subscriptionIdleExpirationDays: 365
```

Or via Helm `values.yaml`:

```yaml
api:
  deployment:
    extraEnv:
      - name: ESCI_INTEGRATIONS_PUBSUB_ENABLE
        value: 'true'
      - name: ESCI_INTEGRATIONS_PUBSUB_DEFAULTPROJECT
        value: '<project that runs estafette>'
      - name: ESCI_INTEGRATIONS_PUBSUB_ENDPOINT
        value: 'https://<integrations host>/api/integrations/pubsub/events'
      - name: ESCI_INTEGRATIONS_PUBSUB_AUDIENCE
        value: 'beautiful-butterfly'
      - name: ESCI_INTEGRATIONS_PUBSUB_SERVICEACCOUNTEMAIL
        value: '<email address for service account used by estafette-ci-api>'
      - name: ESCI_INTEGRATIONS_PUBSUB_SUBSCRIPTIONNAMESUFFIX
        value: '~estafette-ci-pubsub-trigger'
      - name: ESCI_INTEGRATIONS_PUBSUB_SUBSCRIPTIONIDLEEXPIRATIONDAYS
        value: '365'
```

For all projects used in _pubsub_ triggers the _estafette-ci-api_ service account needs the following roles:

```yaml
- pubsub.subscriber
- pubsub.editor
```

### Integrations > Prometheus

Through `config.yaml`:

```yaml
integrations:
  prometheus:
    enable: true
    serverURL: http://estafette-ci-metrics-server
    scrapeIntervalSeconds: 5
```

Or via Helm `values.yaml`:

```yaml
api:
  deployment:
    extraEnv:
      - name: ESCI_INTEGRATIONS_PROMETHEUS_ENABLE
        value: 'true'
      - name: ESCI_INTEGRATIONS_PROMETHEUS_SERVERURL
        value: 'http://estafette-ci-metrics-server'
      - name: ESCI_INTEGRATIONS_PROMETHEUS_SCRAPEINTERVALSECONDS
        value: '5'
```

### Integrations > BigQuery

Through `config.yaml`:

```yaml
integrations:
  bigquery:
    enable: true
    projectID: '<google cloud project id with dataset>'
    dataset: estafette_ci
```

Or via Helm `values.yaml`:

```yaml
api:
  deployment:
    extraEnv:
      - name: ESCI_INTEGRATIONS_BIGQUERY_ENABLE
        value: 'true'
      - name: ESCI_INTEGRATIONS_BIGQUERY_PROJECTID
        value: '<google cloud project id with dataset>'
      - name: ESCI_INTEGRATIONS_BIGQUERY_DATASET
        value: 'estafette_ci'
```

### Integrations > CloudStorage

Through `config.yaml`:

```yaml
integrations:
  gcs:
    enable: true
    projectID: '<google cloud project id with gcs bucket>'
    bucket: '<google cloud bucket name for logs storage>'
    logsDir: logs
```

Or via Helm `values.yaml`:

```yaml
api:
  deployment:
    extraEnv:
      - name: ESCI_INTEGRATIONS_CLOUDSTORAGE_ENABLE
        value: 'true'
      - name: ESCI_INTEGRATIONS_CLOUDSTORAGE_PROJECTID
        value: '<google cloud project id with gcs bucket>'
      - name: ESCI_INTEGRATIONS_CLOUDSTORAGE_BUCKET
        value: '<google cloud bucket name for logs storage>'
      - name: ESCI_INTEGRATIONS_CLOUDSTORAGE_LOGSDIRECTORY
        value: 'logs'
```

### Integrations > CloudSource

Through `config.yaml`:

```yaml
integrations:
  cloudsource:
    enable: true
```

Or via Helm `values.yaml`:

```yaml
api:
  deployment:
    extraEnv:
      - name: ESCI_INTEGRATIONS_CLOUDSOURCE_ENABLE
        value: 'true'
```


## API Server

### Urls

In order to set correct links for build status integration and provide correct communication between the builder jobs and the api there's some minimal configuration for the API itself:

Through `config.yaml`:

```yaml
apiServer:
  baseURL: 'https://<(private) host for the web gui>'
  integrationsURL: 'https://<public host to receive webhooks>'
  serviceURL: 'http://estafette-ci-api.<namespace>.svc.cluster.local/'
```

Or via Helm `values.yaml`:

```yaml
api:
  # (private) host at which to access the web interface and api
  baseHost: estafette.mydomain.com

  # host to receive webhooks from 3rd party integrations like github, bitbucket
  integrationsHost: estafette-integrations.mydomain.com
```

These values are automatically used in the default `config.yaml` used in the `estafette-ci` Helm chart.

### Log reading/writing

See [full instructions]({{< relref "production-high-availability#store-logs-in-cloud-storage" >}}).

Through `config.yaml`:

```yaml
apiServer:
  logwriters:
  - cloudstorage
  logreader: cloudstorage
```

Or via Helm `values.yaml`:

```yaml
api:
  deployment:
    extraEnv:
      - name: ESCI_APISERVER_LOGWRITERS
        value: cloudstorage
      - name: ESCI_APISERVER_LOGREADER
        value: cloudstorage
```

### Injecting stages

Estafette allows you to globally inject stages into all pipelines. Both _before_ or _after_ all the stages defined in the `.estafette.yaml` manifest have been executed.

Through `config.yaml`:

```yaml
apiServer:
  injectStagesPerOperatingSystem:
    linux:
      build:
        before:
        - name: envvars
          image: extensions/envvars:stable
        after: []
      release:
        before:
        - name: envvars
          image: extensions/envvars:stable
        after: []
      bot:
        before:
        - name: envvars
          image: extensions/envvars:stable
        after: []
```

### Docker config for pipeline jobs

To specify some details on how the `estafette-ci-builder` runs the Docker daemon and configures one another.

Through `config.yaml`:

```yaml
apiServer:
  dockerConfigPerOperatingSystem:
    linux:
      runType: dind
      mtu: 1460
      bip: 192.168.1.1/24
      networks:
      - name: estafette
        driver: default
        subnet: 192.168.2.1/24
        gateway: 192.168.2.1
        durable: false
      registryMirror: https://mirror.gcr.io
```

## Authentication

### Securing the API through JSON Web Tokens (JWT)

The communication between the different _Estafette CI_ components is secured using JSON Web Tokens. This is configured out of the box in the `estafette-ci` Helm chart, but for completeness it has the following config properties.

Through `config.yaml`:

```yaml
auth:
  jwt:
    domain: <private hostname for estafette>
```

Or via Helm `values.yaml`:

```yaml
api:
  deployment:
    extraEnv:
      - name: ESCI_AUTH_JWT_DOMAIN
        value: '<private hostname for estafette>'
```

### Administrators

To gain access to the _Admin_ sections in the Estafette web ui and be allowed to perform any actions there you need to be set up as administrator. You can do so in the following manner.

Through `config.yaml`:

```yaml
auth:
  administrators:
  - <email admin 1>
  - <email admin 2>
```

Or via Helm `values.yaml`:

```yaml
api:
  deployment:
    extraEnv:
      - name: ESCI_AUTH_ADMINISTRATORS
        value: '<email admin 1>,<email admin 2>'
```

Once applied log out and log in again and you should be able to see the _Admin_ section.

### Google login

See [Google login]({{< relref "google-login" >}}).

### Github login

See [Github login]({{< relref "github-login" >}}).

## Database

_Estafette_ supports Postgresql; this allows it to use CockroachDB as it's database. Below is the required configuration to connect to the database.

Through `config.yaml`:

```yaml
database:
  databaseName: defaultdb
  host: estafette-ci-db-public
  insecure: false
  sslMode: verify-full
  certificateAuthorityPath: /cockroach-certs/ca.crt
  certificatePath: /cockroach-certs/tls.crt
  certificateKeyPath: /cockroach-certs/tls.key
  port: 26257
  user: root
  password: ''
	maxOpenConnections: 0
  maxIdleConnections: 2
  connectionMaxLifetimeMinutes: 0
```

Or via Helm `values.yaml`:

```yaml
api:
  deployment:
    extraEnv:
      - name: ESCI_DATABASE_DATABASENAME
        value: defaultdb
      - name: ESCI_DATABASE_HOST
        value: estafette-ci-db-public
      - name: ESCI_DATABASE_INSECURE
        value: 'false'
      - name: ESCI_DATABASE_SSLMODE
        value: verify-full
      - name: ESCI_DATABASE_CERTIFICATEAUTHORITYPATH
        value: /cockroach-certs/ca.crt
      - name: ESCI_DATABASE_CERTIFICATEPATH
        value: /cockroach-certs/tls.crt
      - name: ESCI_DATABASE_CERTIFICATEKEYPATH
        value: /cockroach-certs/tls.key
      - name: ESCI_DATABASE_PORT
        value: '26257'
      - name: ESCI_DATABASE_USER
        value: root
      - name: ESCI_DATABASE_PASSWORD
        value: ''
      - name: ESCI_DATABASE_MAXOPENCONNS
        value: '0'
      - name: ESCI_DATABASE_MAXIDLECONNS
        value: '2'
      - name: ESCI_DATABASE_CONNMAXLIFETIMEMINUTES
        value: '0'
```

## Credentials

In order to centrally manage credentials used by various _Estafette_ extensions or your own custom extensions. The only required fields are `name` and `type`, other fields are fully up to the consumer of the credentials. This allows _Estafette_ to be easily extended with new types of credentials used by _trusted images_ (see section below).

Types supported by _Estafette_'s various official extensions are `container-registry`, `kubernetes-engine` and `slack-webhook` amongst others.

```yaml
credentials:
- name: 'container-registry-estafette'
  type: 'container-registry'
  repository: 'estafette'
  private: false
  username: 'estafette.secret(793-0QLfe5QN3n1k.mDk5fL-_urybZ3T9Z3famkSZR68d-SrfqA==)'
  password: 'estafette.secret(3p0Geq5U12j98HXV.Eaupah0YHK6nQkMgT4WYzC5R8FRQbDk5H6aTo1saw35de2KQ)'
- name: 'container-registry-extensions'
  type: 'container-registry'
  repository: 'extensions'
  private: false
  username: 'estafette.secret(793-0QLfe5QN3n1k.mDk5fL-_urybZ3T9Z3famkSZR68d-SrfqA==)'
  password: 'estafette.secret(3p0Geq5U12j98HXV.Eaupah0YHK6nQkMgT4WYzC5R8FRQbDk5H6aTo1saw35de2KQ)'
- name: 'gke-production'
  type: 'kubernetes-engine'
  project: estafette-production
  cluster: production-europe-west4
  region: europe-west4
  zone: europe-west4-d
  serviceAccountKeyfile: 'estafette.secret(***)'
  defaults:
    namespace: production
    autoscale:
      min: 3
      max: 500
    hosts:
    - ${ESTAFETTE_GIT_NAME}.estafette.io
- name: slack-webhook-estafette
  type: slack-webhook
  workspace: estafette
  webhook: estafette.secret(***)
```

Notes:

* You can best see how much flexibility the credential type allows in the `kubernetes-engine` example where `defaults` for the `extensions/gke` image are provided. This allows you for example to override some of the extensions defaults for specific environments in case you want to deviate from those.

## Trusted images

In order to gain access to the centrally stored credentials and optionally gain additional elevated permissions _trusted images_ can be configured. For a trusted image the `path` provides the full container name except for it's tag. To automatically get access all credentials of one or multiple types each type of those credentials can be listed in the `injectedCredentialTypes` array.

Those injected credentials are then automatically mounted into the stage container(s) that use the trusted image. It mounts credentials at `/credentials/<credential type in lower snake case>.json` (for example `/credentials/kubernetes_engine.json`) in JSON format.

A typical configuration looks like:

```yaml
trustedImages:
- path: extensions/git-clone
  injectedCredentialTypes:
  - bitbucket-api-token
  - github-api-token
- path: extensions/docker
  runDocker: true
  injectedCredentialTypes:
  - container-registry
- path: extensions/gke
  injectedCredentialTypes:
  - kubernetes-engine
- path: extensions/bitbucket-status
  injectedCredentialTypes:
  - bitbucket-api-token
- path: extensions/github-status
  injectedCredentialTypes:
  - github-api-token
- path: extensions/slack-build-status
  injectedCredentialTypes:
  - slack-webhook
- path: estafette/estafette-ci-builder
  runPrivileged: true
```

Notes:

* The `bitbucket-api-token` or `github-api-token` credential types are set on the fly when a build/release job is started for either a Bitbucket or Github repository. This allows a repository to be cloned without needing to set up deploy keys or other forms of ssh authentication.