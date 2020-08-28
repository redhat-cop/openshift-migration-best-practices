# Pre-Migration Testing

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