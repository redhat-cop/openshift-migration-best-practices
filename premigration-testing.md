[![Home](https://github.com/redhat-cop/openshift-migration-best-practices/raw/master/images/home.png)](./README.md) |  [Cluster health checks <](./cluster-health-checks.md) | [> Running the migration](./running-the-migration.md)
---
# Premigration testing

You are now ready to start using the migration tool to migrate your workloads from OCP3 to OCP4.   The Migration Toolkit for Containers (MTC) is a tool leveraging the Open Source projects Velero and Restic to help you migrate your workloads between clusters.   MTC is available as an Operator and can be easily installed following the steps described in the official OpenShift documentation under the migration section.  

https://docs.openshift.com/container-platform/4.5/migration/migrating_3_4/migrating-application-workloads-3-4.html

Please refer to the "Migration Tools prerequisite" section for more information on how to get started.

After completing the installation of MTC, you should end up with the following architecture.   A replication repository will be required between your clusters to backup and restore your data.   You will also have to configure the credential of your source and destination clusters.   

![Architecture](./images/mtc-architecture.png)


## Ensure same version of MTC on all clusters

Prior to proceeding with a migration ensure you are using the same version of MTC on all clusters.  It's possible over time that an OLM installed version of MTC may have been updated to pick up latest releases while a previously installed version on an OCP 3 cluster is lagging on an older release.   Ensuring all are on the same version of MTC will yield the best experience.

## Ensure 'OLM Managed' setting is correct for 'MigrationController' custom resource

* OCP-4 installs should leverage OLM and configure the Operator to defer to OLM for RBAC, this is set via the 'MigrationController' custom resource
```
olm_managed: true
```

* OCP-3 installs should not use OLM and will instruct the operator to handle RBAC concerns
```
olm_managed: false
```

## Executing migration pre-flight tests

When executing a migration using MTC, you are actually performing a backup automatically followed by a restore from this object storage.   All kubernetes objects, PVs and internal images associated with the namespace you are migrating will be copied to this repository.   There are many ways to copy data and your data can also be staged before performing a final migration.  Some best practices information about those options will be provided in the next section.


Your next step should be to validate that MTC is installed correctly by migrating a simple application using the most basic method available in MTC.   First, you should install an "hello world" application of your choice on your source cluster.   To keep it very simple, we recommend using an application without any PV.   Then, following the "Migration applications from the web console" section in the official documentation, select this application and click the migrate button.   You can skip the stage phase in this first step.   Once completed, validate that your application has been migrated from your source to destination cluster.   If you get any error, we recommend following the troubleshooting section available in this guide or the official documentation.

As a second validation step, you can test another application with a PV attached to it.   In this second testing phase, you could first "stage" your migration followed by a final migration.   Please note that staging can be run multiple times so that most of the data is copied to the target before the final migration.   Staging at least once will help reduce the migration time and application downtime during the migration in most cases.   Please refer to the next best practices section for more information on when to use stage.

It's also important to understand that your application should only be migrated once.    Your destination cluster should not have a namespace already created with the same name on your destination cluster before executing a migration.   When executing some tests, it's important to clean your destination cluster from that namespace before executing the same migration a second time.


Prev Section: [Cluster health checks](./cluster-health-checks.md)<br>
Next Section: [Running the migration](./running-the-migration.md)<br>
[Home](./README.md)
