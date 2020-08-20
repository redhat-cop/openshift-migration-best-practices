# openshift-migration-best-practices

The OpenShift Best Practices Guide is meant to assist users who are migrating from OpenShift 3 to OpenShift 4. It is broken down into 5 sections:

 1. **Discovery Phase** - Initial considerations and strategies that can be used while migrating.
 2. **OpenShift 3 and 4 Cluster Health Checks** - Helpful metrics to check prior to migrating.
 3. **Migration Toolkit for Containers pre-migration testing** - How to test the Migration Toolkit for Containers to establish confidence.
 4. **Executing the Migration** - Executing migrations with the Migration Toolkit for Containers.
 5. **Troubleshooting** - Common troubleshooting tasks.

## Discovery Phase

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

### Migration Strategies

#### Stateless Apps - Big Bang migration


![BigBang](https://github.com/redhat-cop/openshift-migration-best-practices/raw/master/images/stateless-bigbang.png)

## OCP3 and OCP4 Cluster Health Checks


## Migration Toolkit for Containers pre-migration testing


## Executing the migration (best practices)


## Troubleshooting
