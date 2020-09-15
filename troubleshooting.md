[![Home](https://github.com/redhat-cop/openshift-migration-best-practices/raw/master/images/home.png)](./README.md) | [Running the migration <](./running-the-migration.md) Troubleshooting
---
# Troubleshooting

Upstream doc for improving debug experience
* https://github.com/konveyor/enhancements/tree/master/enhancements/debug

Work In Progress for a flowchart
* [MTC Debug flowchart](https://app.lucidchart.com/documents/view/d0907ce1-ccf1-4226-86eb-e5332f9d42a4/0_0)

# Using `must-gather`

You can use the `must-gather` tool to collect information for troubleshooting or for opening a customer support case on the [Red Hat Customer Portal](https://access.redhat.com/).

The `openshift-migration-must-gather-rhel8` image collects migration-specific logs and Custom Resource data that are not collected by the default `must-gather` image.

Run the `must-gather` command on your cluster:
````
$ oc adm must-gather --image=registry.redhat.io/rhcam-1-2/openshift-migration-must-gather-rhel8
````

# How to Clean Up/Reset a migration

## Failed migration, how to clean up and retry
* Ensure stage pods have been cleaned up.  If a migration fails during stage or copy, the 'stage' pods will be retained to allow debugging.  Before proceeding with a reattempt of the migration the stage pods need to be manually removed.

## Scale a quiesced application back

TBD

## Note on labels applied to help track what was migrated

TBD

# Removing the MTC Operator and cluster-scoped resources

The following procedure removes the MTC Operator and cluster-scoped resources:

1. Delete the Migration Controller and its resources:
```` 
$ oc delete migrationcontroller <resource_name>
````
2. Wait for the MTC Operator to finish deleting the resources.
3. Uninstall the MTC Operator:
  * On the OpenShift 4 cluster, you can uninstall the Operator by using the [web console](https://docs.openshift.com/container-platform/4.5/operators/olm-deleting-operators-from-cluster.html) or by running the following command: 
  ````
  $ oc delete ns openshift-migration
  ````
  * On the Openshift 3 cluster, you can uninstall the operator by deleting it:
  ````
  $ oc delete -f operator.yml
  ````
4. Delete the cluster-scoped resources:

  * Migration Custom Resource Definition (CRD):
  ````
  $ oc get crds | grep 'migration.openshift.io' | awk '{print $1}' | xargs -I{} oc delete crd {}
  ````  
  * Velero CRD:
  ````
  $ oc get crds | grep velero | awk '{print $1}' | xargs -I{} oc delete crd {}
  ````  
  * Migration ClusterRole:
  ````
  $ oc get clusterroles | grep 'migration.openshift.io' | awk '{print $1}' | xargs -I{} oc delete clusterrole {}
  ````  
  * Velero ClusterRole:
  ````
  $ oc get clusterroles | grep velero | awk '{print $1}' | xargs -I{} oc delete clusterrole {}
  ````  
  * Migration ClusterRoleBindings:
  ````
  $ oc get clusterrolebindings | grep 'migration.openshift.io' | awk '{print $1}' | xargs -I{} oc delete clusterrolebindings {}
  ````  
  * Velero ClusterRoleBindings:
  ````
  $ oc get clusterrolebindings | grep velero | awk '{print $1}' | xargs -I{} oc delete clusterrolebindings {}
  ```` 

