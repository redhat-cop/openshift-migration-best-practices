[![Home](https://github.com/redhat-cop/openshift-migration-best-practices/raw/master/images/home.png)](./README.md) | [> Cluster health checks](./cluster-health-checks.md)
---
# Planning

This section focuses on considerations to take into account when you plan your migration.

* **[Migration tools](#migration-tools)**:
  * [Migration Toolkit for Containers](#migration-toolkit-for-containers)
  * [Upstream migration tools](#upstream-migration-tools)
  * When to consider using alternative tools vs MTC
* **[Migration environment considerations](#migration-environment-considerations)**:
  * **[OpenShift 3](#openshift-3)**: Aspects of the OpenShift 3 source environment that might affect migration
  * **[OpenShift 4](#openshift-4)**: Aspects of the OpenShift 4 target environment that might affect migration
* **[Migration strategies](#migration-strategies)**: Strategies for migrating stateless applications

## Migration tools

### Migration Toolkit for Containers

The Migration Toolkit for Containers (MTC) migrates an application workload, including  Kubernetes resources, data, and images, from an OpenShift 3 source cluster to an OpenShift 4 target cluster.

The migration has two stages:

1. The application workload is backed up from the source cluster to object storage.
2. The application workload is restored to the target cluster from object storage.

**Migrating Kubernetes resources**

MTC migrates all namespaced resources, including Custom Resources. MTC can dynamically discover all the API resources in each referenced namespace.

MTC migrates some cluster-scoped resources. If a namespaced resource references a cluster-scoped resource, it is migrated. Migratable resources include persistent volumes (PVs) bound to a persistent volume claim, cluster role bindings, and security context constraints.

**Migrating persistent volume data**

MTC has two options for migrating persistent volume data:

* **Move**: If the remote storage is accessible to the source and target clusters, you can ‘move’ the PV definition from the source cluster to target cluster. This is the fastest method for migrating PV data. This method is suitable for NFS.
* **Copy**: MTC has two options for copying PV data:
  * **Snapshot**: If your storage provider supports snapshots and if you have configured a Velero VolumeSnapshotPlugin, you can create a snapshot of the volume. Snapshot copying has specific requirements such as a common cloud vendor, common region, and common storage class. AWS, Google Cloud Provider, and Microsoft Azure support the snapshot copy method. 
  * **Filesystem**: For all other storage use cases, the data is migrated by copying the file system. This mechanism supports changing storage classes in a migration and has the most flexibility.

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

* `pvc-migrate`: https://github.com/konveyor/pvc-migrate
* `imagestream-migrate`: https://github.com/konveyor/imagestream-migrate

The upstream tools have the advantages of being smaller, more focused tools. Because they use a combination of Ansible playbooks, Python code snippets, [Rsync](https://rsync.samba.org/), and [Skopeo](https://github.com/containers/skopeo), they are easier to configure and debug than the Golang-based Kubernetes controllers used by MTC.

It is possible to use the upstream tools with MTC:

1. Configure MTC to omit PVs and images from the migration plan by setting the following parameters in the Migration Controller manifest:
  ```
  disable_image_migration: true
  disable_pv_migration: true
  ```
2. Migrate the application workload with MTC.

3. Run `pvc-migrate` or `imagestream-migrate` to migrate the PVs or images.

### When to consider using alternative tools vs MTC

 * Large scale migrations
  * ~50+ ImageStreams per Namespace
  * Multiple ~100GB+ Persistent Volumes to be copied

 * Requirements for alternative tools
  * Direction connection available between source and target cluster, i.e. a process on each Node of the source cluster is able to connect to an exposed Route on the target cluster.
  * The host executing pvc-migrate has root ssh access to each Node of the source cluster
  * For pvc-migrate, OCS 3 -> OCS 4 is the only supported path.  No other storage providers are implemented.

 * Considerations of MTC vs alternative tools
    * Serial processing: MTC processes 1 Plan at a time and 1 Backup/Restore operation at a time
        * MTC plans to address this in future to allow parallel execution
            * Related to future contributions in Velero via [velero-#487](https://github.com/vmware-tanzu/velero/issues/487) 
        * Parallel Image migrations are possible using imagestream-migrate
    * Double copy:  Each MTC migration consists of at least 2 copy operations:  Backup copy and Restore copy
        * Alternative tools are a direct single copy from source to target, they don't need to do the 2-step backup/restore process.
    * Ease of debugging/customizing behavior
        * MTC is a collection of golang Kubernetes controllers working together to orchestrate Velero to perform a series of Backup and Restore operations.  Debugging a failure may be challenging, it can span different clusters, namespaces, and controller logs.  In addition, customizing MTC behavior requires golang updates and recompiling code and updating the operator to deliver.
        * Alternative tools leverage Ansible, a small amount of python snippets when essential, and standard tools of rsync and skopeo.  Users are more likely to have a familiarity with Ansible and be able to debug problems and/or customize behavior as they need.
            * An example we've seen in past has been when doing large migrations at scale and running into filesystem corruption issues.  Leveraging `pvc-migrate` with rsync, it is fairly easy to see the rsync error from a corrupted source file and reason through the fix, then address on source volume and re-migrate that PV.  

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

