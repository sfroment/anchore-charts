# Anchore Engine OSS Helm Chart

[Instructions for migrating deployments from helm/stable to charts.anchore.io](#migrating-to-the-new-anchore-charts-repository)

This chart deploys the Anchore Engine docker container image analysis system. Anchore Engine requires a PostgreSQL database (>=9.6) which may be handled by the chart or supplied externally, and executes in a service based architecture utilizing the following Anchore Engine services: External API, SimpleQueue, Catalog, Policy Engine, and Analyzer.

Each of these services can be scaled and configured independently.

See [Anchore Engine](https://github.com/anchore/anchore-engine) for more project details.

## Chart Details

The chart is split into global and service specific configurations for the open source Anchore Engine.

  * The `anchoreGlobal` section is for configuration values required by all Anchore Engine components.
  * Service specific configuration values allow customization for each individual service.

For a description of each component, view the official documentation at: [Anchore Service Overview](https://docs.anchore.com/current/docs/overview/architecture/)

## Installing the Anchore Engine OSS Helm Chart
### TL;DR
```
helm repo add anchore https://charts.anchore.io
helm install my-release anchore/engine
```

Anchore Engine will take approximately 3 minutes to bootstrap. After the initial bootstrap period, Anchore Engine will begin a vulnerability feed sync. During this time, image analysis will show zero vulnerabilities until the sync is completed. This sync can take multiple hours depending on which feeds are enabled. The following anchore-cli command is available to poll the system and report back when the engine is bootstrapped and the vulnerability feeds are all synced up. `anchore-cli system wait`

The recommended way to install the Anchore Engine Helm Chart is with a customized values file and a custom release name. It is highly recommended to set non-default passwords when deploying, all passwords are set to defaults specified in the chart. It is also recommended to utilize an external database, rather then using the included postgresql chart.

Create a new file named `anchore_values.yaml` and add all desired custom values (examples below); then run the following command:

  #### Helm v3 installation
  `helm repo add anchore https://charts.anchore.io`

  `helm install <release_name> -f anchore_values.yaml anchore/anchore-engine`

##### Example anchore_values.yaml - using chart managed PostgreSQL service with custom passwords.
*Note: Installs with chart managed PostgreSQL database. This is not a guaranteed production ready config.*
```
## anchore_values.yaml

postgresql:
  postgresPassword: <PASSWORD>
  persistence:
    size: 50Gi

anchoreGlobal:
  defaultAdminPassword: <PASSWORD>
  defaultAdminEmail: <EMAIL>
```

## Installing on OpenShift
As of chart version 1.3.1 deployments to OpenShift are fully supported. Due to permission constraints when utilizing OpenShift, the official RHEL postgresql image must be utilized, which requires custom environment variables to be configured for compatibility with this chart.

#### Example anchore_values.yaml - deploying on OpenShift
*Note: Installs with chart managed PostgreSQL database. This is not a guaranteed production ready config.*
```
## anchore_values.yaml

postgresql:
  image: registry.access.redhat.com/rhscl/postgresql-96-rhel7
  imageTag: latest
  extraEnv:
  - name: POSTGRESQL_USER
    value: anchoreengine
  - name: POSTGRESQL_PASSWORD
    value: anchore-postgres,123
  - name: POSTGRESQL_DATABASE
    value: anchore
  - name: PGUSER
    value: postgres
  - name: LD_LIBRARY_PATH
    value: /opt/rh/rh-postgresql96/root/usr/lib64
  - name: PATH
     value: /opt/rh/rh-postgresql96/root/usr/bin:/opt/app-root/src/bin:/opt/app-root/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
  postgresPassword: <PASSWORD>
  persistence:
    size: 50Gi

anchoreGlobal:
  defaultAdminPassword: <PASSWORD>
  defaultAdminEmail: <EMAIL>
  openShiftDeployment: True
```

# Chart Updates
See the anchore-engine [CHANGELOG](https://github.com/anchore/anchore-engine/blob/master/CHANGELOG.md) for updates to anchore engine.

## Upgrading from previous chart versions
A Helm post-upgrade hook job will shut down all previously running Anchore services and perform the Anchore DB upgrade process using a kubernetes job. The upgrade will only be considered successful when this job completes successfully. Performing an upgrade will cause the Helm client to block until the upgrade job completes and the new Anchore service pods are started. To view progress of the upgrade process, tail the logs of the upgrade jobs `anchore-engine-upgrade`. These job resources are safe delete after a successful helm upgrade completes.

# Configuration

All configurations should be appended to your custom `anchore_values.yaml` file and utilized when installing the chart. While the configuration options of Anchore Engine are extensive, the options provided by the chart are:

## Exposing the service outside the cluster:

#### Using Ingress

This configuration allows SSL termination using your chosen ingress controller.

##### NGINX Ingress Controller
```
ingress:
  enabled: true
```

##### ALB Ingress Controller
```
  ingress:
    enabled: true
    annotations:
      kubernetes.io/ingress.class: alb
      alb.ingress.kubernetes.io/scheme: internet-facing
    apiPath: /v1/*
    apiHosts:
      - anchore-api.example.com

  anchoreApi:
    service:
      type: NodePort
```

##### GCE Ingress Controller
  ```
  ingress:
    enabled: true
    annotations:
      kubernetes.io/ingress.class: gce
    apiPath: /v1/*
    apiHosts:
      - anchore-api.example.com

  anchoreApi:
    service:
      type: NodePort
  ```

#### Using Service Type
  ```
  anchoreApi:
    service:
      type: LoadBalancer
  ```

### Utilize an Existing Secret
Can be used to override the default secrets.yaml provided
```
anchoreGlobal:
  existingSecret: "foo-bar"
```

### Install using an existing/external PostgreSQL instance
*Note: it is recommended to use an external Postgresql instance for production installs*

  ```
  postgresql:
    postgresPassword: <PASSWORD>
    postgresUser: <USER>
    postgresDatabase: <DATABASE>
    enabled: false
    externalEndpoint: <HOSTNAME:5432>

  anchoreGlobal:
    dbConfig:
      ssl: true
  ```

### Install using Google CloudSQL
  ```
  ## anchore_values.yaml
  postgresql:
    enabled: false
    postgresPassword: <CLOUDSQL-PASSWORD>
    postgresUser: <CLOUDSQL-USER>
    postgresDatabase: <CLOUDSQL-DATABASE>

  cloudsql:
    enabled: true
    instance: "project:zone:cloudsqlinstancename"
    # Optional existing service account secret to use.
    useExistingServiceAcc: true
    serviceAccSecretName: my_service_acc
    serviceAccJsonName: for_cloudsql.json
    image:
      repository: gcr.io/cloudsql-docker/gce-proxy
      tag: 1.12
      pullPolicy: IfNotPresent
  ```

### Archive Driver
*Note: it is recommended to use an external archive driver for production installs.*

The archive subsystem of Anchore Engine is what stores large json documents and can consume quite a lot of storage if
you analyze a lot of images. A general rule for storage provisioning is 10MB per image analyzed, so with thousands of
analyzed images, you may need many gigabytes of storage. The Archive drivers now support other backends than just postgresql,
so you can leverage external and scalable storage systems and keep the postgresql storage usage to a much lower level.

##### Configuring Compression:

The archive system has compression available to help reduce size of objects and storage consumed in exchange for slightly
slower performance and more cpu usage. There are two config values:

To toggle on/off (default is True), and set a minimum size for compression to be used (to avoid compressing things too small to be of much benefit, the default is 100):

  ```
  anchoreCatalog:
    archive:
      compression:
        enabled=True
        min_size_kbytes=100
  ```

##### The supported archive drivers are:

* S3 - Any AWS s3-api compatible system (e.g. minio, scality, etc)
* OpenStack Swift
* Local FS - A local filesystem on the core pod. Does not handle sharding or replication, so generally only for testing.
* DB - the default postgresql backend

#### S3:
  ```
  anchoreCatalog:
    archive:
      storage_driver:
        name: 's3'
        config:
          access_key: 'MY_ACCESS_KEY'
          secret_key: 'MY_SECRET_KEY'
          #iamauto: True
          url: 'https://S3-end-point.example.com'
          region: null
          bucket: 'anchorearchive'
          create_bucket: True
      compression:
      ... # Compression config here
  ```

#### Using Swift:

The swift configuration is basically a pass-thru to the underlying pythonswiftclient so it can take quite a few different
options depending on your swift deployment and config. The best way to configure the swift driver is by using a custom values.yaml

The Swift driver supports three authentication methods:

* Keystone V3
* Keystone V2
* Legacy (username / password)

##### Keystone V3:
  ```
  anchoreCatalog:
    archive:
      storage_driver:
        name: swift
        config:
          auth_version: '3'
          os_username: 'myusername'
          os_password: 'mypassword'
          os_project_name: myproject
          os_project_domain_name: example.com
          os_auth_url: 'foo.example.com:8000/auth/etc'
          container: 'anchorearchive'
          # Optionally
          create_container: True
      compression:
      ... # Compression config here
  ```

##### Keystone V2:
  ```
  anchoreCatalog:
    archive:
      storage_driver:    
        name: swift
        config:
          auth_version: '2'
          os_username: 'myusername'
          os_password: 'mypassword'
          os_tenant_name: 'mytenant'
          os_auth_url: 'foo.example.com:8000/auth/etc'
          container: 'anchorearchive'
          # Optionally
          create_container: True
      compression:
      ... # Compression config here
  ```

##### Legacy username/password:
  ```
  anchoreCatalog:
    archive:
      storage_driver:
        name: swift
        config:
          user: 'user:password'
          auth: 'http://swift.example.com:8080/auth/v1.0'
          key:  'anchore'
          container: 'anchorearchive'
          # Optionally
          create_container: True
      compression:
      ... # Compression config here
  ```

#### Postgresql:

This is the default archive driver and requires no additional configuration.

### Prometheus Metrics

Anchore Engine supports exporting prometheus metrics form each container. To enable metrics:
  ```
  anchoreGlobal:
    enableMetrics: True
  ```

When enabled, each service provides the metrics over the existing service port so your prometheus deployment will need to
know about each pod and the ports it provides to scrape the metrics.

### Using custom certificates
A secret needs to be created in the same namespace as the anchore-engine chart installation. This secret should contain all custom certs, including CA certs & any certs used for internal TLS communication. 
This secret will be mounted to all anchore-engine pods at /home/anchore/certs to be utilized by the system.

### Event Notifications

Anchore Engine in v0.2.3 introduces a new events subsystem that exposes system-wide events via both a REST api as well
as via webhooks. The webhooks support filtering to ensure only certain event classes result in webhook calls to help limit
the volume of calls if you desire. Events, and all webhooks, are emitted from the core components, so configuration is
done in the coreConfig.

To configure the events:
  ```
  anchoreCatalog:
    events:
      notification:
        enabled:true
      level=error
  ```

### Scaling Individual Components

As of Chart version 0.9.0, all services can now be scaled-out by increasing the replica counts. The chart now supports
this configuration.

To set a specific number of service containers:
  ```
  anchoreAnalyzer:
    replicaCount: 5

  anchorePolicyEngine:
    replicaCount: 3
  ```

To update the number in a running configuration:

`helm upgrade --set anchoreAnalyzer.replicaCount=2 <releasename> anchore/anchore-engine -f anchore_values.yaml`