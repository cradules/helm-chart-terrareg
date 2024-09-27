# Terrareg Helm Chart

This Helm chart is designed to deploy the Terrareg application on a Kubernetes cluster. It includes configurations for persistent storage and secure handling of sensitive environment variables.

## Prerequisites

- Helm 3.x
- Kubernetes 1.16+
- PersistentVolume provisioner support in the underlying infrastructure (optional, for persistent storage)

## Installation

To install the chart with the release name `terrareg`:
```shell
helm install terrareg ./terrareg
```

Or, to specify custom values using the `values.yaml` file:
```shell
helm install terrareg ./terrareg -f values.yaml
```

## Values

The following table lists the configurable parameters of the `terrareg` chart and their default values.

| Parameter                                    | Description                                                                                                                                       | Default                        |
|----------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------|
| `replicaCount`                               | Number of replicas of the application pod                                                                                                         | `1`                            |
| `image.repository`                           | Application image repository                                                                                                                      | `ghcr.io/matthewjohn/terrareg` |
| `image.tag`                                  | Application image tag                                                                                                                             | `v3.11.0`                      |
| `image.pullPolicy`                           | Image pull policy                                                                                                                                 | `IfNotPresent`                 |
| `service.port`                               | Service port exposed by the application                                                                                                           | `80`                           |
| `persistence.enabled`                        | Enable persistent storage                                                                                                                         | `false`                        |
| `persistence.size`                           | Size of persistent volume                                                                                                                         | `10Gi`                         |
| `persistence.storageClass`                   | Storage class to be used for persistent volume                                                                                                    | `standard`                     |
| `persistence.accessModes`                    | Access modes for the persistent volume                                                                                                            | `ReadWriteOnce`                |
| `resources.limits.cpu`                       | CPU limit for the application pod                                                                                                                 | `100m`                         |
| `resources.limits.memory`                    | Memory limit for the application pod                                                                                                              | `128Mi`                        |
| `resources.requests.cpu`                     | CPU request for the application pod                                                                                                               | `50m`                          |
| `resources.requests.memory`                  | Memory request for the application pod                                                                                                            | `64Mi`                         |
| `livenessProbe`                              | Liveness probe configuration                                                                                                                      | `{}`                           |
| `readinessProbe`                             | Readiness probe configuration                                                                                                                     | `{}`                           |
| `autoscaling.enabled`                        | Enable horizontal pod autoscaler                                                                                                                  | `false`                        |
| `autoscaling.minReplicas`                    | Minimum number of replicas when autoscaling                                                                                                       | `1`                            |
| `autoscaling.maxReplicas`                    | Maximum number of replicas when autoscaling                                                                                                       | `10`                           |
| `autoscaling.targetCPUUtilizationPercentage` | Target CPU utilization percentage for autoscaling                                                                                                 | `80`                           |
| `environment.PUBLIC_URL`                     | The Public URL of the application. If left empty, it defaults to HTTPS and port 443.                                                              | `https://chart-example.local`  |
| `environment.MIGRATE_DATABASE`               | Whether to run database migrations on startup.                                                                                                    | `True`                         |
| `environment.LISTEN_PORT`                    | The port on which the application listens.                                                                                                        | `80`                           |
| `environment.SERVER`                         | The server used for running the application (`builtin` or `waitress`).                                                                            | `waitress`                     |
| `existingSecret`                             | Existing Kubernetes secret to use for environment variables.                                                                                      | `terrareg-secret`              |
| `mariadb.enabled`                            | Enable or disable MariaDB as the database backend.                                                                                                | `true`                         |
| `mariadb.architecture`                       | The architecture of MariaDB (`standalone` or `replication`).                                                                                      | `standalone`                   |
| `mariadb.auth.existingSecret`                | Use an existing secret for database credentials (requires keys: `mariadb-root-password`, `mariadb-replication-password`, and `mariadb-password`). | `terrareg-db-secret`           |
| `mariadb.auth.username`                      | The username for the MariaDB user.                                                                                                                | `terrareg`                     |
| `mariadb.auth.database`                      | The name of the MariaDB database.                                                                                                                 | `terrareg`                     |

## Storage
If persistence.enabled is set to true, a PersistentVolumeClaim (PVC) will be created for the application, and the storage will be mounted at /app/data. 
The size, storage class, and access modes can be configured through the values.yaml file.

If you don’t need persistent storage, leave persistence.enabled set to false.

## Probes
This chart allows you to configure livenessProbe and readinessProbe to ensure the application is running and ready to accept traffic. These configurations can be customized by modifying the values.yaml.

## Autoscaling
You can enable autoscaling by setting autoscaling.enabled to true. 
The chart supports Horizontal Pod Autoscaler (HPA) configuration, and you can adjust the minimum and maximum number of replicas as well as the target CPU utilization percentage.

## Security
Environment variables, including sensitive data such as SECRET_KEY and DATABASE_URL, are managed using Kubernetes secrets. The application requires the following secrets to be provided in the cluster:

terrareg-secret: A secret containing the SECRET_KEY, ADMIN_AUTHENTICATION_TOKEN, and DATABASE_URL.

## Uninstalling the Chart
```shell
helm uninstall terrareg
```
This command removes all the Kubernetes components associated with the chart and deletes the release.

## Upgrading the Chart
```shell
helm upgrade terrareg ./terrareg -f values.yaml
```


## Database Configuration

### MariaDB as a Dependency

This Helm chart has MariaDB as a dependency for the application when persistence is required. MariaDB is deployed using the Bitnami MariaDB chart, which allows the application to use a fully managed, persistent database solution.

To enable MariaDB as the database:

1. Set `mariadb.enabled` to `true` in your `values.yaml`.
2. Ensure that `persistence.enabled` is set to `true` for both MariaDB and the application if you need data to be stored persistently.

Example configuration in `values.yaml`:

```yaml
mariadb:
  enabled: true

persistence:
  enabled: true
```
When MariaDB is enabled, the chart automatically builds the SQL connection string using the MariaDB credentials and exposes it to the application through a Kubernetes secret.

## Secrets Management

The secrets for the MariaDB database, such as DATABASE_UR, are automatically generated by the chart. These secrets are created and injected into the application environment securely using Kubernetes Secrets.
The chart ensures that these secrets are properly cleaned up when the Helm release is deleted, so no sensitive information is left behind.

## Automatic SQL Connection String
```shell
mysql+mysqlconnector://<user>:<password>@<host>[:<port>]/<database>
```
This connection string is injected into the application as the DATABASE_URL environment variable, ensuring the app connects to MariaDB correctly without manual configuration.

## Init Container for Database Readiness
The chart includes an initContainer that checks for database readiness before the main application container starts. 
This ensures that the application does not attempt to start before the MariaDB database is fully initialized and ready to accept connections.

The initContainer will:
- Continuously poll the MariaDB service until it is reachable.
- Ensure there is no race condition where the application pod starts before MariaDB is available.

## SQLite Option (No Persistence)

The application also has an option to run with an embedded SQLite database, which requires no additional database configuration. However, SQLite does not provide persistent storage, meaning all data is lost when the pod restarts.

To enable SQLite, simply set mariadb.enabled to false. By default, when MariaDB is disabled, the application will use SQLite. If you choose this option, the application can run without any external dependencies, but there will be no persistence.

Example SQLite configuration in values.yaml:
```yaml
mariadb:
  enabled: false
```

## Persistence Requirements
- SQLite: No persistence available. Data is ephemeral and will be lost when the pod restarts.
- MariaDB: If persistence is required, enable both MariaDB and persistence in the values.yaml configuration. This will ensure that the database and the application both store data persistently across pod restarts.

Example persistence configuration:
```yaml
persistence:
  enabled: true
  size: "10Gi"
  storageClass: "standard"
```

By enabling persistence for the application and the database, you ensure that both application data and database data are safely stored in a persistent volume.


## Summary of Configuration
- MariaDB: Enable for a fully persistent database solution.
- SQLite: Use for a lightweight, non-persistent database (data is not retained after restarts).
- Secrets: Automatically generated and cleaned up on deletion.
- Init Container: Ensures that the application only starts after the MariaDB database is ready.
- Persistence: Required for MariaDB, optional for SQLite. Persistent volumes can be managed via values.yaml.


### Key Points:

1. **MariaDB as a Dependency**: You can choose to enable it for persistence; otherwise, the application will default to SQLite without persistence.
2. **Secrets Management**: Secrets like app secrets or db secrets are automatically created and deleted.
3. **SQL Connection String**: Automatically constructed by the chart and injected into the application.
4. **Init Container**: Ensures database readiness before the application starts.
5. **SQLite**: Offers no persistence; use MariaDB for persistence if needed.


## Sources
[mariaDB chart](https://artifacthub.io/packages/helm/bitnami/mariadb)