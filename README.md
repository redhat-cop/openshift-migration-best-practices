# openshift-migration-best-practices

The OpenShift Best Practices Guide is meant to assist users who are migrating from OpenShift 3 to OpenShift 4. It is broken down into 5 sections:

 1. Discovery Phase
 2. Initiatl Considerations
 3. OpenShift 3 and 4 Cluster Health Checks
 4. Migration Toolkit for Container

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


## CAM pre-migration testing


## Executing the migration (best practices)


## Troubleshooting
