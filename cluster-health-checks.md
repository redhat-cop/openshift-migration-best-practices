[![Home](https://github.com/redhat-cop/openshift-migration-best-practices/raw/master/images/home.png)](./README.md) | [Planning <](./planning.md) Cluster health checks  [> Premigration testing](./premigration-testing.md)
---
# Cluster health checks

This section contains a list of checks to run on your OpenShift 3.9+ source and 4.x target clusters before migration. The purpose of these checks is to detect issues that might affect the migration process.

This list is not comprehensive and the verification of these checks does not guarantee a successful migration.  We recommend getting in contact with the support team before migrating a cluster from Openshift 3 to 4, especially if the cluster is in a production environment.

* **[General health checks](#general-health-checks)**
  * [Source cluster](#source-cluster)
  * [Target cluster](#target-cluster)
* **[Resource capacity](#resource-capacity)**
* **[Performance](#performance)**
* **[Additional checks](#additional-checks)**

## General health checks

### Source cluster

You can perform the following health checks on an OpenShift 3.9+ source cluster:

* Install and configure [Prometheus cluster monitoring](https://docs.openshift.com/container-platform/3.11/install_config/prometheus_cluster_monitoring.html). Prometheus provides a detailed view of the health of the cluster components.

* Check the node status to verify that all nodes are in a **Ready** state: 
  ```
  $ oc get nodes
  ```

* Check the persistent volumes (PVs):
  ```
  $ oc get pv
  ```
  * Mounted PVs
  * Unmounted PVs
  * Abnormal configurations
  * PVs stuck in terminating state

* Check pods for status that is not **Running** or **Completed**.  Use the following command because pods might not display an error state:
  ```
  $ oc get pods --all-namespaces|egrep -v 'Running | Completed'
  ```

* Check for pods with a high restart count. Even if they are in a **Running** state, a high restart count might indicate underlying problems.

* Check the [health of the **etcd** cluster](https://access.redhat.com/articles/3093761).

* Check the [network connectivity](https://docs.openshift.com/container-platform/3.11/day_two_guide/environment_health_checks.html#connectivity-on-master-hosts) between master hosts.

* Check the [API service status](https://docs.openshift.com/container-platform/3.11/day_two_guide/environment_health_checks.html#day-two-guide-api-service-status).

* Check that the cluster certificates are not close to expiration and will be valid for the duration of the migration process. You can use the [`easy-mode` ansible playbook](https://docs.openshift.com/container-platform/3.11/install_config/redeploying_certificates.html#install-config-cert-expiry) to check the certificates.

* Check for pending certificate signing requests:
  ```
  $ oc get crs
  ```

### Target cluster

You can perform the following health checks on an OpenShift 4.x target cluster:

* Check that the cluster has access to external services required by the applications by verifying network connectivity and proper permissions. 

  Examples of external services include databases, source code repositories, container image registries, and CI/CD tools.

* Check that external applications, services, and appliances that use services provided by the target cluster have access and proper permissions.

* Verify that all internal container image dependencies are met.

  If an application requires an image that is not in the application namespace, check that the image exists. For example, an application that uses the `php:7.1` base image from the PHP imagestream on an OpenShift 3.11 cluster will not work on an OpenShift 4.x cluster because that particular version is not included in the PHP imagestream for OpenShift 4. See [migration prerequisites](https://docs.openshift.com/container-platform/4.5/migration/migrating_3_4/migrating-application-workloads-3-4.html#migration-prerequisites_migrating-3-4) for a list of `imagestreamtags` that have been removed from OpenShift 4.2.

## Resource capacity

* The clusters require additional memory, CPUs, and storage in order to run a migration on top of normal workloads. Actual resource requirements depend on the number of Kubernetes resources being migrated in a single migration plan.

* Check that the OpenShift 3.9+ source cluster meets the [minimum hardware requirements](https://docs.openshift.com/container-platform/3.11/install/prerequisites.html#hardware) for an OpenShift installation.

* Check that the OpenShift 4.x target cluster meets the minimum hardware requirements for the specific platform and installation method. For example, a bare metal installation has [specific minimum resource requirements](https://docs.openshift.com/container-platform/4.5/installing/installing_bare_metal/installing-bare-metal.html#minimum-resource-requirements_installing-bare-metal).

* Check the available bandwidth between the source and target clusters. Less than 10 Gbps is not recommended.

* If you are migrating more than 20 TB, check that the target cluster and the replication repository have sufficient storage.

## Performance

* Check cluster compute and memory utilization: `$ oc adm top node`

* Check the average response time of API calls in the source cluster. Less than 50 ms is recommended.

## Additional checks

* Review the [migration considerations](https://access.redhat.com/documentation/en-us/openshift_container_platform/4.5/html-single/migration/index#migration-considerations).

* Check that the identity provider is working on the source and target clusters.
