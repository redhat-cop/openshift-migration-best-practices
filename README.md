# openshift-migration-best-practices

The OpenShift Best Practices Guide is meant to assist users who are migrating from OpenShift 3 to OpenShift 4. It is broken down into 5 sections:

 1. **Discovery Phase** - Initial considerations and strategies that can be used while migrating.
 2. **OpenShift 3 and 4 Cluster Health Checks** - Helpful metrics to check prior to migrating.
 3. **Migration Toolkit for Containers pre-migration testing** - How to test the Migration Toolkit for Containers to establish confidence.
 4. **Executing the Migration** - Executing migrations with the Migration Toolkit for Containers.
 5. **Troubleshooting** - Common troubleshooting tasks.

## Discovery Phase

The discovery phase section focuses on considerations that should be taken into account when planning a migration including security, routing, and image registry migration. It also explains several of the common strategies and scenarios for migration.

### Initial Considerations

#### SSL Considerations

- Termination Type
  - Passthrough
  - Edge
    - Minimal amount as this is managed by the cluster by default
  - Reencrypt
    - Where is the certificate originating?
     - Corporate CA
     - Self-Signed
  - Update to routes
- Certificate Consumption
  - Reading PEM files
  - Embedded Certificates

#### Routing
  - Traffic traversal between clusters

#### External Dependencies
  - Ingress/Egress

#### Images
  - Migration of OpenShift's Internal Registry

#### Storage

- An intermediate object storage is required to act as a replication repository for the CAM tool to migrate data

- Source and target clusters must have full access to the replication repository

- Create a migration plan to either copy or move the data


### Migration Strategies

#### Stateless Apps - Big Bang migration

Apps are deployed in the 4.x cluster
(if needed) 4.x router default certificate contains also the 3.x  wildcard SAN
Each application adds an additional route with the 3.x hostname
At migration time the 3.x wildcard DNS record is changed to point to the 4.x router VIP


![BigBang](https://github.com/redhat-cop/openshift-migration-best-practices/raw/master/images/stateless-bigbang.png)

#### Stateless Apps - Individual migration

Description:
Apps are deployed in the 4.x cluster
(if needed) 4.x router default certificate contains also the 3.x  wildcard SAN
Each application adds an additional route with the 3.x hostname

(optionally) the route with the 3.x hostname contains an appropriate certificate

For each app, at migration time, a new record is created with the app 3.x fqdn/hostname pointing to the 4.x router VIP. This will take precedence over the 3.x wildcard DNS record.

#### Stateless Apps - Individual canary-release-style migration

Apps are deployed in the 4.x cluster
(if needed) 4.x router default certificate contains also the 3.x  wildcard SAN
Each application adds an additional route with the 3.x hostname

(optionally) the route with the 3.x hostname contains an appropriate certificate.

A per app VIP/proxy is created with two backends: the 3.x router VIP and the 4.x router VIP.

For each app, at migration time, a new record is created with the app 3.x fqdn/hostname pointing to the VIP/proxy. This will take precedence over the 3.x wildcard DNS record.

The proxy entry for that app is configured to route X% of the traffic to the 3.x router VIP and (100-X)% of the traffic to 4.x VIP.

X is gradually moved from 100 to 0.

#### Stateless Apps - Individual audience-based migration

Apps are deployed in the 4.x cluster
(if needed) 4.x router default certificate contains also the 3.x  wildcard SAN
Each application adds an additional route with the 3.x hostname

(optionally) the route with the 3.x hostname contains an appropriate certificate.

A per app VIP/proxy is created with two backends: the 3.x router VIP and the 4.x router VIP.

For each app, at migration time, a new record is created with the app 3.x fqdn/hostname pointing to the VIP/proxy. This will take precedence over the 3.x wildcard DNS record.

The proxy entry for that app is configured to route traffic matching a given header pattern (e.g.: test customers) of the traffic to the 4.x router VIP and the rest of the traffic to 3.x VIP. More and more cohorts of customers are moved to the 4.x VIP through waves, until all the customers are on the 4.x VIP.



## OCP3 and OCP4 Cluster Health Checks



## Migration Toolkit for Containers pre-migration testing


## Executing the migration (best practices)


## Troubleshooting
