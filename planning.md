[![Home](https://github.com/redhat-cop/openshift-migration-best-practices/raw/master/images/home.png) <](./README.md) | [> Cluster health checks](./cluster-health-checks.md)
---
# Planning

This section focuses on considerations to take into account when you plan your migration.

* **[Migration tools](#migration-tools)**:
  * [Migration Toolkit for Containers](#migration-toolkit-for-containers)
  * [Upstream migration tools](#upstream-migration-tools)
  * [Upstream tools vs MTC](#upstream-tools-vs-mtc)
* **[Migration environment considerations](#migration-environment-considerations)**:
  * **[OpenShift 3](#openshift-3)**: Aspects of the OpenShift 3 source environment that might affect migration
  * **[OpenShift 4](#openshift-4)**: Aspects of the OpenShift 4 target environment that might affect migration
* **[Migration strategies](#migration-strategies)**: Strategies for migrating stateless applications

## Migration tools

### Migration Toolkit for Containers

The Migration Toolkit for Containers (MTC) migrates an application workload, including  Kubernetes resources, data, and images, from an OpenShift 3 source cluster to an OpenShift 4 target cluster.

MTC performs the migration in two stages:

1. The application workload is backed up from the source cluster to object storage.
2. The application workload is restored to the target cluster from object storage.

**Migrating Kubernetes resources**

MTC migrates all namespaced resources, including Custom Resources. MTC can dynamically discover all the API resources in each referenced namespace.

MTC migrates some cluster-scoped resources. If a namespaced resource references a cluster-scoped resource, it is migrated. Migratable resources include persistent volumes (PVs) bound to a persistent volume claim, cluster role bindings, and security context constraints.

**Migrating persistent volume data**

MTC has two options for migrating persistent volume data:

* **Move**: If the remote storage is accessible to the source and target clusters, you can ‘move’ the PV definition from the source cluster to target cluster. 
  
  This is the fastest method for migrating PV data. This method is suitable for NFS.

* **Copy**: MTC has two options for copying PV data:
  * **Snapshot**: If your storage provider supports snapshots and if you have configured a Velero VolumeSnapshotPlugin, you can create a snapshot of the volume. 
    
    Snapshot copying has specific requirements such as a common cloud vendor, common region, and common storage class. AWS, Google Cloud Provider, and Microsoft Azure support the snapshot copy method. 

  * **Filesystem**: For all other storage use cases, the data is migrated by copying the file system.
    
    This method allows you to change storage classes in a migration and provides the most flexibility.

**Migrating internal images**

Internal images created by S2I builds are migrated. Each ImageStream reference in a given namespace is copied to the registry of the target cluster.

**References**

* [MTC prerequisites](https://docs.openshift.com/container-platform/4.5/migration/migrating_3_4/migrating-application-workloads-3-4.html#migration-prerequisites_migrating-3-4)
* [About MTC](https://docs.openshift.com/container-platform/4.5/migration/migrating_3_4/migrating-application-workloads-3-4.html#migration-understanding-cam_migrating-3-4)
* [About data copy methods](https://docs.openshift.com/container-platform/4.5/migration/migrating_3_4/migrating-application-workloads-3-4.html#migration-understanding-data-copy-methods_migrating-3-4)

#### When to use MTC

Ideally, you could migrate an application from one cluster to another by redeploying the application from a pipeline and perhaps copying the persistent volume data.  

However, this might not be possible in the real world. A running application on the cluster might experience unforeseen changes and, over a period of time, drift away from the initial deploy. MTC can handle scenarios where you are not certain what your namespace contains and you want to migrate all its contents to a new cluster.

If you can redeploy your application from pipeline, that is the best option. If not, you should use MTC.

### Upstream migration tools

There are upstream tools that you can use for large-scale migrations of PVs or images:

* [`pvc-migrate`](https://github.com/konveyor/pvc-migrate)
* [`imagestream-migrate`](https://github.com/konveyor/imagestream-migrate)

The upstream tools have the advantages of being smaller, more focused tools. Because they use a combination of Ansible playbooks, Python code snippets, [Rsync](https://rsync.samba.org/), and [Skopeo](https://github.com/containers/skopeo), they are easier to configure and debug than the Golang-based Kubernetes controllers used by MTC.

You can combine the upstream tools and MTC for migration in a process that resembles the following procedure:

1. Configure MTC to omit PVs and images from the migration plan by setting the following parameters in the Migration Controller manifest:
  ```
  disable_image_migration: true
  disable_pv_migration: true
  ```
2. Migrate the application workload with MTC.

3. Run `pvc-migrate` to migrate PVs or `imagestream-migrate` images.

### Upstream tools vs MTC

Upstream tools can perform large-scale migrations much faster than MTC if the migration environment meets tool requirements.

The following environment is an example of a large-scale migration:

* ~50+ ImageStreams per namespace
* Multiple ~100GB+ persistent volumes

**Upstream tool requirements**

* There must be a direct network connection between the source and target clusters so that a process running on each node of the source cluster can connect to an exposed route on the target cluster.
* The host running `pvc-migrate` must have root access to each node of the source cluster.
* `pvc-migrate` can only migrate from OpenShift Container Storage 3 to 4. No other storage providers are implemented.

**Comparison**

| | MTC  | Upstream tools |
| ------------- | ------------- | ------------- |
| Processing  | Serial processing. MTC processes one migration plan at a time, one backup/restore operation at a time. (MTC plans to support parallel execution in the future. See [velero-487](https://github.com/vmware-tanzu/velero/issues/487).)  |Parallel processing. `imagestream-migrate` can perform parallel image migrations. |
| Copying  |Two copy processes: backup and restore  | Single copy process |
|Debugging   |Challenging to debug. A migration error can span different clusters, namespaces, and controller logs.   |Easier to debug because of Ansible. |
|Customization   |Difficult to customize. Requires updating and recompiling the Golang code and then updating the MTC Operator to deliver the new version.   |Ansible and Python are easier to customize.  |


## Migration environment considerations

### OpenShift 3

The following considerations apply to the OpenShift 3 source environment.

#### TLS

* Termination types
  * Passthrough
  * Edge
    * Minimal amount as this is managed by the cluster by default
  * Re-encryption
    * Where does the certificate originate?
    * Corporate CA
    * Self-signed certificates
  * Update to routes
* CA certificates
  * Reading PEM files
  * Embedded certificates

#### Routing

* Traffic traversal between clusters

#### External dependencies

* Ingress/Egress

#### Images

* Migrating the internal image registry
* Prune the image registry before migration
* TBD: unknown blob error in registry (perhaps related to NFS)

#### Storage

* An intermediate object storage is required to act as a replication repository for the CAM tool to migrate data
* Source and target clusters must have full access to the replication repository
* Create a migration plan to copy or move the data
* TBD: Velero does not over-write objects in source environment. Link to Velero documentation.

### OpenShift 4

The following considerations apply to the OpenShift 4 target environment:

* Creating namespaces before migration is problematic because it can cause quotas to change.

## Migration strategies

This section describes migration strategies for stateless applications.

### "Big Bang" migration

* Applications are deployed in the 4.x cluster.
* If necessary, the 4.x router default certificate includes the 3.x wildcard SAN.
* Each application adds an additional route with the 3.x hostname.
* At migration, the 3.x wildcard DNS record is changed to point to the 4.x router VIP.

![BigBang](https://github.com/redhat-cop/openshift-migration-best-practices/raw/master/images/migration-strategy-bigbang.png)

### Individual migration

* Applications are deployed in the 4.x cluster
* If necessary, the 4.x router default certificate includes the 3.x wildcard SAN.
* Each application adds an additional route with the 3.x hostname.
* Optional: The route with the 3.x hostname contains an appropriate certificate.

For each app, at migration time, a new record is created with the app 3.x fqdn/hostname pointing to the 4.x router VIP. This will take precedence over the 3.x wildcard DNS record.

![Individual](https://github.com/redhat-cop/openshift-migration-best-practices/raw/master/images/migration-strategy-individual.png)

### Individual canary-release-style migration

* Applications are deployed in the 4.x cluster
* If necessary, the 4.x router default certificate includes the 3.x wildcard SAN.
* Each application adds an additional route with the 3.x hostname.
* Optional: The route with the 3.x hostname contains an appropriate certificate.

A per app VIP/proxy is created with two backends: the 3.x router VIP and the 4.x router VIP.

For each app, at migration time, a new record is created with the app 3.x fqdn/hostname pointing to the VIP/proxy. This will take precedence over the 3.x wildcard DNS record.

The proxy entry for that app is configured to route X% of the traffic to the 3.x router VIP and (100-X)% of the traffic to 4.x VIP.

X is gradually moved from 100 to 0.

![Canary](https://github.com/redhat-cop/openshift-migration-best-practices/raw/master/images/migration-strategy-canary.png)

### Individual audience-based migration

* Applications are deployed in the 4.x cluster
* If necessary, the 4.x router default certificate includes the 3.x wildcard SAN.
* Each application adds an additional route with the 3.x hostname.
* Optional: The route with the 3.x hostname contains an appropriate certificate.

A per app VIP/proxy is created with two backends: the 3.x router VIP and the 4.x router VIP.

For each app, at migration time, a new record is created with the app 3.x fqdn/hostname pointing to the VIP/proxy. This will take precedence over the 3.x wildcard DNS record.

The proxy entry for that app is configured to route traffic matching a given header pattern (e.g.: test customers) of the traffic to the 4.x router VIP and the rest of the traffic to 3.x VIP. More and more cohorts of customers are moved to the 4.x VIP through waves, until all the customers are on the 4.x VIP.

![Audience](https://github.com/redhat-cop/openshift-migration-best-practices/raw/master/images/migration-strategy-audience.png)

