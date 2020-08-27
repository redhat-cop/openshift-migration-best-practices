# Best practices for migrating from OpenShift&nbsp;Container&nbsp;Platform&nbsp;3&nbsp;to&nbsp;4

This guide provides recommendations and best practices for migrating from OpenShift Container Platform 3.9+ to OpenShift 4.x with the Migration Tookit for Containers (MTC). The recommendations are organized into the following sections:

1. **[Planning](#planning)**: Considerations and strategies to review while planning the migration
2. **[Cluster health checks](#cluster-health-checks)**: Metrics to check before migration
3. **[Pre-migration testing](#pre-migration-testing)**: Testing the migration plan before running the actual migration
4. **[Running the migration](#running-the-migration)**: Best practices for running the migration
5. **[Troubleshooting](#troubleshooting)**: Common troubleshooting tasks

## Planning

This section focuses on considerations to take into account when you plan your migration.

### Source environment

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
* TBD: unknown blog error in registry

#### Storage

* An intermediate object storage is required to act as a replication repository for the CAM tool to migrate data
* Source and target clusters must have full access to the replication repository
* Create a migration plan to copy or move the data
* TBD: Velero does not over-write objects in source environment. Link to Velero documentation.

### Target environment

The following considerations apply to the OpenShift 4 target environment.

* Creating namespaces before migration is problematic because it can cause quotas to change.

### Migration strategies

#### Stateless Applications * Big Bang migration

* Applications are deployed in the 4.x cluster.
* If necessary, the 4.x router default certificate includes the 3.x wildcard SAN.
* Each application adds an additional route with the 3.x hostname.
* At migration, the 3.x wildcard DNS record is changed to point to the 4.x router VIP.

![BigBang](https://github.com/redhat-cop/openshift-migration-best-practices/raw/master/images/stateless-bigbang.png)

#### Stateless Applications * Individual migration

* Applications are deployed in the 4.x cluster
* If necessary, the 4.x router default certificate includes the 3.x wildcard SAN.
* Each application adds an additional route with the 3.x hostname.
* Optional: The route with the 3.x hostname contains an appropriate certificate.

For each app, at migration time, a new record is created with the app 3.x fqdn/hostname pointing to the 4.x router VIP. This will take precedence over the 3.x wildcard DNS record.

#### Stateless Applications * Individual canary-release-style migration

* Applications are deployed in the 4.x cluster
* If necessary, the 4.x router default certificate includes the 3.x wildcard SAN.
* Each application adds an additional route with the 3.x hostname.
* Optional: The route with the 3.x hostname contains an appropriate certificate.

A per app VIP/proxy is created with two backends: the 3.x router VIP and the 4.x router VIP.

For each app, at migration time, a new record is created with the app 3.x fqdn/hostname pointing to the VIP/proxy. This will take precedence over the 3.x wildcard DNS record.

The proxy entry for that app is configured to route X% of the traffic to the 3.x router VIP and (100-X)% of the traffic to 4.x VIP.

X is gradually moved from 100 to 0.

#### Stateless Applications * Individual audience-based migration

* Applications are deployed in the 4.x cluster
* If necessary, the 4.x router default certificate includes the 3.x wildcard SAN.
* Each application adds an additional route with the 3.x hostname.
* Optional: The route with the 3.x hostname contains an appropriate certificate.

A per app VIP/proxy is created with two backends: the 3.x router VIP and the 4.x router VIP.

For each app, at migration time, a new record is created with the app 3.x fqdn/hostname pointing to the VIP/proxy. This will take precedence over the 3.x wildcard DNS record.

The proxy entry for that app is configured to route traffic matching a given header pattern (e.g.: test customers) of the traffic to the 4.x router VIP and the rest of the traffic to 3.x VIP. More and more cohorts of customers are moved to the 4.x VIP through waves, until all the customers are on the 4.x VIP.

## Cluster health checks


## Pre-migration testing


## Running the migration


## Troubleshooting
