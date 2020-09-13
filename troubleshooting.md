[![Home](https://github.com/redhat-cop/openshift-migration-best-practices/raw/master/images/home.png)](./README.md) | [Running the migration <](./running-the-migration.md) Troubleshooting
---
# Troubleshooting

Draft of upstream docs for improving debug experience:
* https://github.com/konveyor/enhancements/pull/2

Work In Progress for a flowchart
* [MTC Debug flowchart](https://app.lucidchart.com/documents/view/d0907ce1-ccf1-4226-86eb-e5332f9d42a4/0_0)

# Generating a must-gather
`oc adm must-gather --image=registry.redhat.io/rhcam-1-2/openshift-migration-must-gather-rhel8`

# How to Clean Up/Reset a migration
## Failed migration, how to clean up and retry
* Ensure stage pods have been cleaned up.  If a migration fails during stage or copy, the 'stage' pods will be retained to allow debugging.  Before proceeding with a reattempt of the migration the stage pods need to be manually removed.

## Scale a quiesced application back

## Note on labels applied to help track what was migrated

# How to remove MTC Operator and clean up cluster scoped resources
1. Remove the 'MigrationController' Custom Resource so the Operator cleans up resources it provisioned 
    * `oc delete migrationncontroller $resourcename`
      * Wait for the operator to finish cleaning up the resources it owns 
2. Remove the Migration Operator
    * For OLM installs
      1. Uninstall the operator via OLM
      2. `oc delete ns openshift-migration`
    * For OCP 3.x installs
      * Refer to the 'operator.yml' used to instantiate the operator
        * `oc delete -f operator.yml` 
          * Alternatively remove the resources inside of the 'openshift-migration' namespace
3. Remove cluster scopes resources
```
  oc get crds | grep 'migration.openshift.io' | awk '{print $1}' | xargs -I{} oc delete crd {}
  oc get crds | grep velero | awk '{print $1}' | xargs -I{} oc delete crd {}
  oc get clusterroles | grep 'migration.openshift.io' | awk '{print $1}' | xargs -I{} oc delete clusterrole {}
  oc get clusterroles | grep velero | awk '{print $1}' | xargs -I{} oc delete clusterrole {}
  oc get clusterrolebindings | grep 'migration.openshift.io' | awk '{print $1}' | xargs -I{} oc delete clusterrolebindings {}
  oc get clusterrolebindings | grep velero | awk '{print $1}' | xargs -I{} oc delete clusterrolebindings {}
```

Prev Section: [Running the migration](./running-the-migration.md)<br>
[Home](./README.md)
