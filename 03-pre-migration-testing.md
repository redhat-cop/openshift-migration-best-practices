---
title: Premigration testing
layout: default
---

# Premigration testing

## Installing MTC

Install MTC on your source and target clusters:

- [Prerequisites](https://access.redhat.com/articles/5064151)
- [About MTC on OpenShift 3](https://access.redhat.com/articles/5064151#installing-the-mtc-with-the-web-console-and-the-migration-controller-on-a-4x-remote-cluster-2)
- [About MTC on OpenShift 4](https://docs.openshift.com/container-platform/4.6/migration-toolkit-for-containers/installing-mtc.html)
- [About data copy methods](https://docs.openshift.com/container-platform/4.6/migration-toolkit-for-containers/about-mtc.html#migration-understanding-data-copy-methods_about-mtc)

The following diagram describes how MTC uses Velero and Restic to back up data from the source cluster to the replication repository and then restores data from the replication repository to the target cluster:

![MTC Architecture](./images/mtc-architecture.png)

## Ensuring same MTC versions

The same MTC z-stream version must be installed on the source and target clusters.

Download and re-install the MTC Operator on the OpenShift 3 cluster just before you run a migration to ensure that you have the latest version.

The Operator Lifecycle Manager (OLM) pushes MTC Operator updates to the OpenShift 4 cluster automatically. On the OpenShift 3 cluster, however, the MTC Operator is installed manually and is not updated automatically.

## Checking 'OLM Managed' setting

In the web console, check the 'OLM Managed' setting in the 'MigrationController' manifest of each cluster:

- OpenShift 4 uses OLM:
  ```yaml
  olm_managed: true
  ```
- OpenShift 3 does not use OLM:
  ```yaml
  olm_managed: false
  ```

## Migrating a simple application

Migrate a simple application without a persistent volume (PV):

1. Install a simple application without a PV on the source cluster.
2. [Migrate the application](https://docs.openshift.com/container-platform/4.4/migration/migrating_3_4/about-migration.html) to the target cluster. You do not need to stage the migration.
3. Validate the application on the target cluster.

## Migrating an application with a persistent volume

Migrate an application with a PV:

1. Install an application with an associated PV on the source cluster.
2. Stage the migration one or more times. Staging reduces migration time and application downtime during migration.
3. Migrate the application to the target cluster.
4. Validate the application on the target cluster.

## Removing a migrated application namespace

If you are performing multiple test migrations, remove the migrated application namespace from the target cluster after each test.
