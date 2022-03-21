# Customizations

This file explains how to configure CARTO Self-Hosted to meet your needs.

## Production Ready

By default, the Helm configuration provided by CARTO works out of the box, but it's **not production ready**.
These are the steps to prepare the installation to a production ready environment.

It should be configured:
- **MANDATORY** [Configure the domain to be used](#configure-the-domain-of-your-self-hosted).
- [Allow the external traffic](#access-to-carto-from-outside-the-cluster) to access the app.
- [Configure DBs to be production ready](#use-external-databases). Our recomendation is to use managed DBs with backups and so on.

Configuring it is optional:
- Configure scale of the components (staticaly or with auto-scaling).
- Use your own bucket to store the data (By default, GCP CARTO buckets are used)

## Architecture diagram

<!--
We should add an arquitectural diagram to make it easier for customers to understand the parts and the relationship between them.
-->

## How to define customizations

There are two ways to configure or customize the deployment:
- [**RECOMMENDED**] Create a dedicated [yaml](https://yaml.org/) file. For example, you can create a file with the next content:
  ```yaml
  customConfigValues:
    selfHostedDomain: "my.domain.com"
  ```
  And add the following at the end of ALL the install or upgrade command:
  ```bash
  ... -f <my_customization_file>.yaml
  ```
- Use the inline set. For example, add the following at the end of ALL the install or upgrade command:
  ```bash
  ... --set customConfigValues.selfHostedDomain=my.domain.com
  ```


## Configure the domain of your self-hosted

The most important step to have your CARTO self-hosted ready to be used is to configure the domain to be used.

⚠️ CARTO self-hosted is not designed to be used in the path of a URL, it needs a full domain or subdomain. ⚠️

To do this you need to [add the following customization](#how-to-define-customizations):
```yaml
customConfigValues:
  selfHostedDomain: "my.domain.com"
```

Don't forget to upgrade your chart after the change.


## Access to CARTO from outside the cluster
By default, the `router` Service of CARTO self-hosted is configured in mode `ClusterIP` so it's only usable from inside the cluster.
To access to it you can exec locally a [kubectl port-forward](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/) but this only make it accessible to your machine.

To made it accessible from the internet (or outside the Kubernetes network) we recommended you to properly [configure the `router` Service](#service-as-loadbalancer).

Additionally, depending of the way to reach your self-hosted from internet, you would need to [configure the HTTPS/TLS](#configure-tls).

### Router general notes
<!--
TODO: Document timeout increment and disable internal TLS and so on
TODO: Talk about static IP
-->

### Service as LoadBalancer

That's the easiest way of open your CARTO self-hosted to the world.
You need to change the `router` Service type to `LoadBalancer`.

You can check an example [here](service_loadBalancer/config.yaml) (keep in mind the [general notes](#router-general-notes) before to proceed).

<!--
TODO: We need to talk about TLS and so on...
-->

We have examples for multiple cloud providers:
- [AWS EKS](service_loadBalancer/aws_eks/config.yaml)
<!--
TODO: Add the other providers
-->

<!--
### Ingress
-->
<!--
TODO: Document Ingress
-->

### Configure TLS
By default, the package generate a self-signed certificate with a validity of 365 days.
Some times you need to use a valid certificate or need to totally disable it to leave the management to an external proxy.

⚠️ CARTO self-hosted only works if the final client use HTTPS protocol. ⚠️

#### Disable internal HTTPS
<!--
TODO: Document and add the ability to do it
-->

#### Configure your own certificates in CARTO

- Create a kubernetes secret with following content:
  ```bash
  kubectl create secret tls \
    -n <namespace> \
    <your_own_installation_name|carto>-tls-certificate \
    --cert=path/to/cert/file \
    --key=path/to/key/file
  ```

- [Add the following customization](#how-to-define-customizations) lines:

  ```yaml
  tlsCerts:
    autoGenerate: false
    existingSecret:
      name: "<your_own_installation_name|carto>-tls-certificate"
      keyKey: "tls.key"
      certKey: "tls.crt"
  ```

## Use external databases
This package comes with an internal Postgres and Redis but it is not recommended for production. It does not have any logic for backups or any other monitoring.

So we recommend to use external databases, preferible managed database by your provider, with backups, high availability, etc.

### Configure your own postgres
CARTO self-hosted require a Postges (version 13+) to work.
In that Postgres, CARTO stores some metadata and also the credentials of the external connections configured by the CARTO self-hosted users.

⚠️ That Postgres has nothing to do with the connections that the user configures in the CARTO workspace since it stores the metadata of the entire CARTO self-hosted. ⚠️

To proceed you need:
- Create one secret in kubernetes with the Postgres passwords:
  ```bash
  kubectl create secret generic \
    -n <namespace> \
    <your_own_installation_name|carto>-postgres-secret \
    --from-literal=carto-password=<password> \
    --from-literal=admin-password=<password>
  ```
- [Add the following customization](#how-to-define-customizations) lines:
  ```yaml
  postgresql:
    # With that config, we disable the internal Postgres provided by the package
    enabled: false
  externalDatabase:
    host: <Postgres IP/Hostname>
    user: "carto"
    # password: ""
    adminUser: "postgres"
    # adminPassword: ""
    existingSecret: "<your_own_installation_name|carto>-postgres-secret"
    existingSecretPasswordKey: "carto-password"
    existingSecretAdminPasswordKey: "admin-password"
    database: "workspace_db"
    port: "5432"
  ```
- Note: `externalDatabase.user` and `externalDatabase.databse` inside the Postgres instance are going to be created automatically during the installation process, they do not need to be pre-created.

### Configure your own redis
CARTO self-hosted require a Redis (version 5+) to work.
That Redis is mainly used as a cache for the postgres.

To proceed you need:
- Create one secret in kubernetes with the Redis AUTH string:
  ```bash
  kubectl create secret generic \
    -n <namespace> \
    <your_own_installation_name|carto>-redis-secret \
    --from-literal=password=<AUTH string password>
  ```
- [Add the following customization](#how-to-define-customizations) lines:
  ```yaml
  redis:
    # With that config, we disable the internal Redis provided by the package
    enabled: false
  externalRedis:
    host: <Redis IP/Hostname>
    port: "6379"
    # password: ""
    existingSecret: "<your_own_installation_name|carto>-redis-secret"
    existingSecretPasswordKey: "password"
  ```
## Advanced configuration

If you need a more advanced configuration you can check the [full chart documentation](../chart/README.md) or contact [support@carto.com](mailto:support@carto.com)