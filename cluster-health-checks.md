[![Home](https://github.com/redhat-cop/openshift-migration-best-practices/raw/master/images/home.png)](./README.md) | [Planning <](./planning.md) Cluster health checks  [> Premigration testing](./premigration-testing.md)
---
# Cluster health checks
This section contains a list of checks to run on your source 3.9+ and target 4.x OpenShift clusters before proceeding with the actual migration process.  The goal of these checks is to detect potential issues that may impact the migration process. 

This list is not comprehensive and the verification of these checks does not guarantee a successful migration.  We recommend getting in contact with the support team before attempting a migration from an Openshift 3 to 4 cluster, especially in the case of a production environment.

## General health

### Checks on source 3.9+ cluster:

* Install and configure the [Prometheus Cluster Monitoring stack](https://docs.openshift.com/container-platform/3.11/install_config/prometheus_cluster_monitoring.html).  Prometheus provides a detailed view of the health of the cluster componentes, and will help in detecting problems in the cluster.

* Check the cluster nodes status, verify they are all in **Ready** state: 
```
 $ oc get nodes`
```

* Check the persistent volumes (PVs) on the source cluster: `$ oc get pv`
  * Mounted PVs
  * Unmounted PVs
  * Abnormal configurations
  * PVs stuck in terminating state

* Check pods with status different from **Running** or **Completed**.  However not all of the pods in the list will signal an error:
```
$ oc get pods --all-namespaces|egrep -v 'Running | Completed'
```

* Check for pods with a high restart count.  Even if they are in a status of **Running**, these may point to underlying problems.

* Check that the **etcd** cluster is [healthy](https://access.redhat.com/articles/3093761).

* Check the [network connectivity](https://docs.openshift.com/container-platform/3.11/day_two_guide/environment_health_checks.html#connectivity-on-master-hosts) between master hosts.

* Check the [API service status](https://docs.openshift.com/container-platform/3.11/day_two_guide/environment_health_checks.html#day-two-guide-api-service-status).

* Check that the cluster certificates are not close to expiration and will be valid for the duration of the migration process. The [easy-mode ansible playbook](https://docs.openshift.com/container-platform/3.11/install_config/redeploying_certificates.html#install-config-cert-expiry) is a useful tool to perform this check.

* Check for any pending certificate signing requests (csr):
```
$ oc get crs
```

### Checks on target 4.x cluster:

* Check that the cluster has access to any external services required by the applications, verify network connectivity and proper permissions.  Common examples of external services are: Databases; source code repositories, container image registries, CI/CD tools, etc.  

* Check that applications, services, appliances, etc. external to the cluster that use services provided by the Openshift cluster have access and proper permissions in the target cluster.

* Verify that all internal container images dependencies are met.
If an application depends on an image that must be present on the cluster's internal registry, and that image is not built in the application's own project, check that the image exists.  For example an application using the php:7.1 base image from the php image stream in an OpenShift 3.11 cluster will not work on an OpenShift 4.x cluster, because that particular version is not included in the php image stream.

## Resource capacity

### Source 3.9+ cluster. 

* Check that the nodes in the cluster meet the [minimum hardware requirements](https://docs.openshift.com/container-platform/3.11/install/prerequisites.html#hardware) for an OpenShift installation.  The cluster requires spare resources in terms of memory, CPU and storage to be able to run its usual workloads plus the Cluster Application Migration (**CAM**) tool.  The amount of spare resources required depends on the number of kubernetes resources being migrated on a single run of the **CAM** tool.

### Target 4.x cluster.

* Check that the cluster meets the minimum hardware requirements for the specific platform and method of installation used.  For example [this are the minimum resource requirements](https://docs.openshift.com/container-platform/4.5/installing/installing_bare_metal/installing-bare-metal.html#minimum-resource-requirements_installing-bare-metal) for a baremetal installation. The cluster requires spare resources in terms of memory, CPU and storage to be able to run its usual workloads plus the Cluster Application Migration (**CAM**) tool.  The amount of spare resources required depends on the number of kubernetes resources being migrated on a single run of the **CAM** tool.

* Available bandwidth between the source and target clusters:
  * Less than 10Gbps
  * 10 Gbps or more

* Amount of data to be migrated between the source and target clusters:
  * Less than 20 TB
  * 20 TB or more

## Performance

* Check cluster compute and memory utilization: `$ oc adm top node`

* Average response time of API calls in the source cluster:
  * Less than 50 ms
  * 50 ms or more

## Additional checks

* Review [migration considerations](https://access.redhat.com/documentation/en-us/openshift_container_platform/4.5/html-single/migration/index#migration-considerations)

* Check that the Identity provider is working on both source and destinations clusters
