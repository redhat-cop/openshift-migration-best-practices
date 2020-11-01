[![Home](./images/home.png)](./README.md) |  [Premigration testing <](./premigration-testing.md) Running the migration [> Troubleshooting](./troubleshooting.md)
---
# Running the migration

This section focuses on considerations associated with creating and running a migration plan.

* **[Before creating a migration plan](#before-creating-a-migration-plan)**
  * [Migration environment](#migration-environment)
  * [Increasing migration plan limits](#increasing-migration-plan-limits)
  * [Resource quotas](#resource-quotas)
  * [Internal images](#internal-images)
  * [Route host names](#route-host-names)
  * [Pod UIDs](#pod-uids)
* **[Creating a migration plan](#creating-a-migration-plan)**
  * [Migrate an application with its dependencies](#migrate-an-application-with-its-dependencies)
  * [Create one migration plan at a time](#create-one-migration-plan-at-a-time)
  * [PV move](#pv-move)
  * [PV copy](#pv-copy)
  * [Deprecated APIs](#deprecated-apis)
* **[Running a migration plan](#running-a-migration-plan)**
  * [Run one migration plan at a time](#run-one-migration-plan-at-a-time)
  * [Run stage migrations to reduce downtime](#run-stage-migrations-to-reduce-downtime)
  * [Quiescing applications](#quiescing-applications)

## Before creating a migration plan

This section describes considerations to review before you create a migration plan.

### Migration environment

Prepare your migration environment by checking the following:

* Network:
  * Ensure that your clusters have adequate network bandwidth, especially if you are copying PVs. 
  * Ensure that [DNS records](https://docs.openshift.com/container-platform/4.5/installing/installing_bare_metal/installing-bare-metal.html#installation-infrastructure-user-infra_installing-bare-metal) for your application exist on the target cluster.
  * Certificates: Ensure that certificates used by your application exist on the target cluster.
  * Ensure that the appropriate [firewall rules](https://docs.openshift.com/container-platform/4.5/installing/install_config/configuring-firewall.html) are configured on the target cluster.
  * Ensure that load balancing is correctly configured on the target cluster.
* Resources:
  * Check whether your application uses a service network or an external route to communicate with services.
  * If your application uses non-namespaced resources, you must re-create them on the target cluster.
  * You must migrate images from the internal image registry if you are not using an external image registry. If they cannot be migrated, you must re-create them manually on the target cluster.
  * Increase the [CPU and memory limits of the Migration Controller](https://docs.openshift.com/container-platform/4.5/migration/migrating_3_4/migrating-applications-with-cam-3-4.html#migration-changing-migration-plan-limits_migrating-3-4) for large migrations.
  * Use the [`prune` command](https://docs.openshift.com/container-platform/4.5/applications/pruning-objects.html) to remove old builds, deployments, and images from each namespace being migrated.
  * [Exclude PVs, imagestreams, and other resources](https://docs.openshift.com/container-platform/4.5/migration/migrating_3_4/migrating-applications-with-cam-3-4.html#migration-excluding-resources_migrating-3-4) if you do not want to migrate them with MTC.
* Namespaces:
  * Check the namespaces on the target cluster to ensure that they do not duplicate namespaces being migrated.
  * Do not create namespaces for your application on the target cluster before migration because this can cause quotas to change.
* Storage:
  * Ensure that the object storage (replication repository) has sufficient room for the PV data and images being migrated.
  * If PV migrations are slow, check that your storage nodes have adequate input/output, CPUs, and memory.
  * Back up PV data in case an application displays unexpected behavior after migration and corrupts the data.

### Increasing migration plan limits

You can [increase the limits](https://docs.openshift.com/container-platform/4.5/migration/migrating_3_4/migrating-applications-with-cam-3-4.html#migration-changing-migration-plan-limits_migrating-3-4) for a single migration plan, but such changes must be *approached with caution and thoroughly tested*.

MTC has the following migration plan defaults to encourage smaller migrations:
* 100 PVs
* 100 Pods
* 10 namespaces

These limits should only be increased if a large plan is required for migrating a namespace that contains many resources or migrating several namespaces together because of dependency issues.

### Resource quotas

If you are using resource quotas, the following considerations might apply:
* Source cluster: If you are migrating PVs, you might have to [increase the Pod limit](https://docs.openshift.com/container-platform/4.5/applications/quotas/quotas-setting-per-project.html#quotas-creating-a-quota_quotas-setting-per-project) on the source cluster. The migration process creates a temporary 'stage' Pod on the source cluster when a PV is copied. See [MIG-217 - Catch error conditions of not enough quota when attempting to do a stage/migrate](https://issues.redhat.com/browse/MIG-217).
* Target cluster: The resource quotas on the target cluster must be sufficient for the namespace that you are migrating. See [MIG-203 - Handle ResourceQuotas when changing storage classes, allow for possibility of keys changing](https://issues.redhat.com/browse/MIG-203).

### Internal images

If your application uses images from the `openshift` namespace, you must ensure that the required versions of these images are present on the target cluster. MTC processes each imagestream in a referenced namespace and migrates the images. Images that are not included in the imagestream of a referenced namespace are not copied.

If the required images are not present, you must update the `imagestreamtags` references to use an available version that is compatible with your application.

If the `imagestreamtags` cannot be updated, you can manually upload equivalent images to the application namespace and update the application to reference them. [Certain `imagestreamtags`](https://docs.openshift.com/container-platform/4.5/migration/migrating_3_4/migrating-application-workloads-3-4.html#migration-prerequisites_migrating-3-4) have been removed from OpenShift 4.

### Route host names

If an application uses an OpenShift route, the resource is migrated to the target cluster.

The `openshift.io/host.generated` annotation determines whether the host name is updated:
* If `openshift.io/host.generated` is set, the [OpenShift Velero route plugin](https://github.com/konveyor/openshift-velero-plugin/blob/master/velero-plugins/route/restore.go) strips the source host name from the route and updates the route with the target cluster host name.
* If `openshift.io/host.generated` is _not_ set, the route is migrated as is from the source cluster. The host name of the route **is not** updated.

### Pod UIDs

MTC preserves Pod UIDs during the migration process by migrating the following namespace annotations:
* `openshift.io/sa.scc.mcs`
* `openshift.io/sa.scc.supplemental-groups`
* `openshift.io/sa.scc.uid-range`

These annotations preserve the UID range, ensuring that the containers retain their file system permissions on the target cluster.

Migrated UIDs might duplicate UIDs in a current or future namespace on the target cluster. See [BZ#1748531 - Handling UID range namespace annotations when migrating an application from source to destination](https://bugzilla.redhat.com/show_bug.cgi?id=1748531).

## Creating a migration plan

This section describes considerations to review when you create a migration plan.

### Migrate an application with its dependencies

Check whether your application uses services, images, or resources in other namespaces.

If the application has dependencies, you should migrate the application and the dependency namespaces in the same migration plan.

### Create one migration plan at a time

Create and save *one* migration plan at a time. Do not create and save multiple migration plans.

Currently, the Migration Controller watches all the namespaces and associated resources in a migration plan. Multiple migration plans result in an additional load on the source cluster and the MTC host cluster. This may change in the future: [MIG-319 - Allow migration plans to be precreated without adding extra load to source or host cluster](https://issues.redhat.com/browse/MIG-319).

### PV move

*PV move* is faster than *PV copy*.

MTC moves a PV by re-creating the PV definition on the target cluster. The actual data remains untouched. PV move is suitable for NFS.

The remote storage must be accessible to both the source and target clusters.

### PV copy

MTC copies a snapshot or the file system of the PV.

The following table compares the *snapshot* and *file system* copy options.

| | Snapshot copy  | File system copy |
| ------------- | ------------- | ------------- |
|**Speed**  |Fast   |Slow   |
|**Provider support**  |AWS, Google Cloud Provider, Microsoft Azure<sup>1</sup>   |All providers   |
|**Storage class changes**   |No   |Yes<sup>2</sup>  |
|**Regional limitations**   |Yes<sup>3</sup>   |No   |

<sup>1</sup> The source and target PVs must have the same cloud storage provider.  
<sup>2</sup> The source and target PVs can have different storage classes.  
<sup>3</sup> The source and target PVs must be in the same geographic region.

### Deprecated APIs

In OpenShift 4.x, some API GroupVersionKinds (GVKs) that are used by OpenShift 3.x are [deprecated](https://kubernetes.io/blog/2019/07/18/api-deprecations-in-1-16/). If your source cluster contains deprecated APIs, the following error is displayed in the MTC console when you create a migration plan: `Some namespaces contain GVKs incompatible with destination cluster`. You can run the migration plan but the affected resources will not be migrated.

You can [migrate the excluded resources manually](https://docs.openshift.com/container-platform/4.5/migration/migrating_3_4/troubleshooting-3-4.html#migration-gvk-incompatibility_migrating-3-4) after the migration.

## Running migration plans

This section describes considerations to review when you run a migration plan.

### Run one migration plan at a time

Run *one* plan at a time. Do not try to run multiple migration plans simultaneously.

Although the MTC Migration Controller can run multiple migration plans, Velero, which performs the backup/restore operations, currently supports only single threading.

If you need to run different migrations, create and run the plans sequentially.

### Run stage migrations to reduce downtime

Users will experience application downtime during the migration. The length of downtime depends on the size of the migration, including the number of namespaces, PVs, images, and resources in each namespace.

Stage migrations reduce the duration of the final migration because they perform incremental backup/restore operations on PV data and images.

You can reduce the downtime by performing stage migrations several days before the final migration:

1. Schedule downtime for the final migration.
2. Several days before the final migration, run stage migrations so that most of the PV and image data is copied to the target cluster.
3. Just before the final migration, run a stage migration to copy the most recent changes to the target cluster.
4. Quiesce the application and then run the final migration. 

Stage migrations normally reduce the duration of the final migration. However, if an application writes very large files, incremental backup/restore might not reduce the final migration time significantly.

**Comparison of stage migration and final migration**

The following table compares stage migration and final migration:

| | Stage migration  | Final migration |
| ------------- | ------------- | ------------- |
|**Kubernetes resources**   | No<sup>1</sup>  | Yes  |
|**PV move**   |No   |Yes  |
|**PV copy**   |Yes  |Yes  |
|**Images**   |Yes  |Yes  |
|**Application state**   |Running   |Quiesced<sup>2</sup>  |

<sup>1</sup> Exceptions: PV copy and images.  
<sup>2</sup> All pods are scaled to `0` for a consistent view of the data.

### Quiescing applications

Quiescing brings an application to a consistent state so that its file system can be copied without causing data corruption.

Deployments, deployment configuration, jobs, stateful sets, and other resources are scaled to `0` if you select the `Quiesce` option before running a migration plan.

In the future, MTC might support quiescence options such as putting an application into `read-only` mode.

  
