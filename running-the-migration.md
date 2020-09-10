[![Home](https://github.com/redhat-cop/openshift-migration-best-practices/raw/master/images/home.png)](./README.md) |  [Premigration testing <](./premigration-testing.md) | [> Troubleshooting](./troubleshooting.md)
---
# Running the migration

## Preliminary considerations

### Resource requirements
* Be aware of how much room is required for PV data and Images, if copying to Object Storage ensure sufficient room
* Depending on the number of resources in each namespace you may need to increase the cpu and memory limits for the migration controller.  Settings are adjusted via the 'MigrationController' custom resource on the host cluster
    ```
        mig_controller_limits_cpu: "1"
        mig_controller_limits_memory: "10Gi"
    ```
* Available bandwidth for migration to proceed
    * Migrations, especially of PV data are bandwidth intensive.  Prior to a migration look at the network bandwidth available of the cluster and ensure there is sufficient capacity available for your needs.  This is highly dependent on the scenario/environment.  Basic guidance is that if you see PV migrations proceeding slowly ensure your storage nodes have sufficient resources and the migration is not being starved for network i/o, cpu, or memory.


### Pruning

* Run `oc adm prune` prior to a migration to remove unneeded 'builds', 'depoyments', and 'images' in each source namespace
    * https://docs.openshift.com/container-platform/3.11/admin_guide/pruning_resources.html

### Skip processing of specific resource types

For some scenarios it may be desired to instruct MTC to skip processing a specific kind of resource type, for example Images, PVs, or even a specific Group,Version,Kind.  The below shows how to configure this global setting for all Plans processed by MTC.

* 'disable_pv_migration':  skips PV migration when `true`
* 'disable_image_migration': skips Image migration when `true`
* 'excluded_resources': a list of Group,Version,Kinds to skip, each entry is a specific GVK to skip processing

An example:
```
$ oc edit migrationcontroller -n openshift-migration

disable_image_migration: true
disable_pv_migration: true
excluded_resources:
- imagetags
- templateinstances

```

### Plan limits
 * MTC suggests a series of limits on PVs, Pods, and Namespaces per Plan to encourage smaller migrations.  These limits may be altered with the understanding any changes to limits should be approached with caution.  
    * Recommendation is to create smaller Plans when possible, would consider bumping limits only for cases when a large Plan is required perhaps because of a large single namespace or need to group several namespaces together for dependencies.
        ```
        $ oc edit migrationcontroller -n openshift-migration
        mig_pv_limit: 100
        mig_pod_limit: 100
        mig_namespace_limit: 10
        ```

### Quotas

When Quotas are in place on source/target cluster the below should be considered:
* Migrations on the source cluster will create temporary 'Stage' pods per PV being copied, you may need to increase quota to allow these to succeed.
    * Related: [MIG-217 - Catch error conditions of not enough quota when attempting to do a stage/migrate](https://issues.redhat.com/browse/MIG-217)
* Migrations to the target cluster will need sufficient quota to at least create the needs from the source namespace.
* There is a known issue when a ResourceQuota is used and the related storageclass name changes from source to target:  
    * Related: [MIG-203 - Handle ResourceQuotas when changing storageclasses, allow for possibility of keys changing](https://issues.redhat.com/browse/MIG-203)


### Deprecated APIs

Be aware that when migrating applications between clusters that are different versions or even just have different capabilities, it is possible to run into situations where the source cluster has an API being used which is not present in the target cluster.  MTC will attempt to perform a Group,Version,Kind comparison of source and target cluster and warn of potential issues.  If problems are found, those impacted resources will be noted on the MigPlan custom resource and will be excluded from the migration.

Manual intervention is recommended post migration to handle those specific resources.

### Internal Images

MTC will process each ImageStream in a referenced namespace and copy the underlying images.  Note that some applications may be leveraging images in another namespace such as `openshift`.  Those images which are not directly refered to by ImageStreams in the referenced namespace are not copied.  


### Migrating Routes

If an application is using an OpenShift Route the resource will be migrated to the target cluster.  Based on how the Route is created it may or may not auto update the associated hostname of the route.

If the Route has the annotation `openshift.io/host.generated` set then the [openshift-velero-plugin/route plugin](https://github.com/konveyor/openshift-velero-plugin/blob/master/velero-plugins/route/restore.go) will strip the host field so it is recreated using the target clusters configuration.   

For other cases, where `openshift.io/host.generated` is _not_ set the Route will be migrated as-is using whatever information was used from the source cluster.

### Namespaces on target cluster

For each namespace you are migrating from the source cluster it is expected that namespace does _not_ exist on the destination cluster.

### Preserving UID/SELinux from source to target

When a pod is running on a source cluster and is then migrated to a target cluster we want to preserve the same UID it was running as so persistent volume data retains expected privileges when remounted and accessed on the target.

MTC approaches this by ensuring the annotations in the namespace are recreated from source to target
```
openshift.io/sa.scc.mcs
openshift.io/sa.scc.supplemental-groups
openshift.io/sa.scc.uid-range
```

These annotations preserve the UID range, ensuring that the containers retain their file system permissions on the target cluster. There is a risk that the migrated UIDs could duplicate UIDs within an existing or future namespace on the target cluster.

Related:  [Bug 1748531 - Handling UID range namespace annotations when migrating an application from source to destination](https://bugzilla.redhat.com/show_bug.cgi?id=1748531)


### Quiesce

Quiescing is the act of bringing your application to a consistent state so that it's filesystem may be copied without worries of corrupted or inconstent data.  

MTC offers a simplified version of a quiesce which is to scale an application to '0', i.e., deployments, deploymentconfigs, jobs, statefulsets, etc will be scaled to '0' if quiesce is selected when running a migration.

Further improvements will likely allow more custom approaches to quiescing by leveraging application specific features such as putting the application into a read-only mode.


### Migration Window - Downtime

Migrations with MTC will have downtime.  The downtime is directly dependent on the size of the migration, including the number of namespaces, PVs, Images, and resources in each namespace.  

Migrations are a 2 step process of Backup + Restore.

The migration window may be shrunk through a process of 'staging' PV and Image data to the target cluster.   This is achieved by performing one or more 'Stage' migrations followed by the single 'Final Migration' to do the cutover.   This will shorten the migration window by potentially making the 'Final Migration' shorter through benefits of incremental backup/restore.

The general recommendation is:
1. Stage your data ahead of time perhaps several days before the migration window run a stage to copy the bulk of PVs and Images to the target.
1. Closer to your migration window perform a subsequent stage to pickup any deltas
1. Take an outage and begin the final migration, quiesce the application to scale it to zero and have a consistent view of the data, the migration will perform another backup/restore to pick up changes.   The previous stages may make this operation quicker, depending on the application characteristics and if incremental backup/restore helped.  Some applications may have significant churn or write very large files, so even small changes end up avoiding a savings from incremental backup/restore.

## Creating Migration Plans

### Check application dependencies

Prior to migrating an application you should consider if that namespace has dependencies on other services on the cluster.  Ideally you would migrate related services during the same migration window.  

Recommendation is:
1. Identify dependent namespaces
2. Group dependent namespaces in the same migration plan

### Limit precreated Migration Plans

Precreating Migration Plans adds an additional load to both the source and host cluster as the migration controller running on the host will begin to watch all namespaces and associated resources in each Migration Plan.  

Recommendation is to only create a single plan at a time.

Related to: [MIG-319 - Allow migration plans to be precreated without adding extra load to source or host cluster](https://issues.redhat.com/browse/MIG-319)


## Execute the Migration

### Migrate 1 Plan at a time

Do not attempt to migrate multiple plans at the same time.  While the migration controller is written to support multiple processing of plans, the underlying Velero component is single threaded at present.  

Recommendation is:
1. Create a single plan a a time
1. Migrate that single plan
1. When the migration is complete, create the next plan and migrate that and so forth


### Stage or Final Migration

* Stage migrations will _only_ copy Images and process PVs to be copied.  
    * Kubernetes resources will not be processed beyond the essentials to support PV + Image copying
    * Persistent Volumes to be moved will not yet be processed and will be skipped in a stage.

* Final migration is the migration that will perform a full backup/restore of the namespaces processing all entities
    * Kubernetes resources will be backed up from the source and restored to the target via Velero
    * Persistent volume data will be copied and/or moved
    * Internal Images will be processed

Note, 'Final Migrations' may be quiesced where the source namespace has all managed pods scaled to zero to bring the application 'down' for a consistent view of data.

### Persistent Volume Move or Copy
* Move:  'moves' the persistent volume definition only, actual data remains untouched in the remote volume
    * Only available for remote storage options which are reachable by both the source and target cluster.
    * Involves recreating the Persistent Volume YAML definition on the target cluster, essentially doing an unmount on source and mount on target.
    * Quickest option for PV migrations, data remains on the remote volume and is not processed.
    * Recommendation:  Ensure you have a backup of data prior to migrating as if your application misbehaves after migration the PV data may be corrupted.
* Copy:  Snapshot or Filesystem copy of persistent volume data
    * Snapshot
        * For storage providers which have a configured velero volume snapshotter plugin a snapshot may be possible.
        * Generally cloud storage such as AWS, Azure, Google
            * Be aware of further requirements per cloud provider such as each cluster existing in same region or perhaps even data center.
    * Filesystem copy:
        * A general purpose means of leveraging Velero's Restic integration to perform a filesystem level copy of data from a source persistent volume to object storage, then later copying from object storage to destination cluster.


Prev Section: [Pre-migration testing](./pre-migration-testing.md)<br>
Next Section: [Troubleshooting](./troubleshooting.md)<br>
[Home](./README.md)
  
