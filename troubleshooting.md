[![Home](https://github.com/redhat-cop/openshift-migration-best-practices/raw/master/images/home.png)](./README.md) | [Running the migration <](./running-the-migration.md) Troubleshooting
---
# Troubleshooting

This section describes common troubleshooting procedures.

* **[Using `must-gather`](#using-must-gather)**
* **[Cleaning up a failed migration](#cleaning-up-a-failed-migration)**
* **[Deleting the MTC Operator and resources](#deleting-the-mtc-operator-and-resources)**

Upstream doc for improving debug experience
* https://github.com/konveyor/enhancements/tree/master/enhancements/debug

Debug flowchart (in progress)
* [MTC Debug flowchart](https://app.lucidchart.com/documents/view/d0907ce1-ccf1-4226-86eb-e5332f9d42a4/0_0)

# Using `must-gather`

You can use the `must-gather` tool to collect information for troubleshooting or for opening a customer support case on the [Red Hat Customer Portal](https://access.redhat.com/). The `openshift-migration-must-gather-rhel8` image collects migration-specific logs and Custom Resource data that are not collected by the default `must-gather` image.

Run the `must-gather` command on your cluster:
````
$ oc adm must-gather --image=registry.redhat.io/rhcam-1-2/openshift-migration-must-gather-rhel8
````

# Cleaning up a failed migration

## Failed migration, how to clean up and retry
* Ensure stage pods have been cleaned up.  If a migration fails during stage or copy, the 'stage' pods will be retained to allow debugging.  Before proceeding with a reattempt of the migration the stage pods need to be manually removed.

## Scale a quiesced application back

If the migrated source application was quiesced during your migration,
you will also need to scale it back to its initial replica count. This can be
done manually by editing the deployment primitive (Deployment, DeploymentConfig, etc.)
and setting the `spec.replicas` field back to the original, non-zero value:

```
$ oc edit deployment <deployment-name>
```

## Note on labels applied to help track what was migrated

When quiescing a source application, MTC will annotate the original replica
count on the deployment object for reference:

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
    migration.openshift.io/preQuiesceReplicas: "1"
```

# Deleting the MTC Operator and resources

The following procedure removes the MTC Operator and cluster-scoped resources:

1. Delete the Migration Controller and its resources:
    ```` 
    $ oc delete migrationcontroller <resource_name>
    ````
    Wait for the MTC Operator to finish deleting the resources.

2. Uninstall the MTC Operator:
    * OpenShift 4: Uninstall the Operator in the [web console](https://docs.openshift.com/container-platform/4.5/operators/olm-deleting-operators-from-cluster.html) or by running the following command: 
    ````
    $ oc delete ns openshift-migration
    ````
    * Openshift 3: Uninstall the operator by deleting it:
    ````
    $ oc delete -f operator.yml
    ````

4. Delete the cluster-scoped resources:
    * Migration custom resource definition:
    ````
    $ oc get crds | grep 'migration.openshift.io' | awk '{print $1}' | xargs -I{} oc delete crd {}
    ````  
    * Velero custom resource definition:
    ````
    $ oc get crds | grep velero | awk '{print $1}' | xargs -I{} oc delete crd {}
    ````  
    * Migration cluster role:
    ````
    $ oc get clusterroles | grep 'migration.openshift.io' | awk '{print $1}' | xargs -I{} oc delete clusterrole {}
    ````  
    * Velero cluster role:
    ````
    $ oc get clusterroles | grep velero | awk '{print $1}' | xargs -I{} oc delete clusterrole {}
    ````  
    * Migration cluster role bindings:
    ````
    $ oc get clusterrolebindings | grep 'migration.openshift.io' | awk '{print $1}' | xargs -I{} oc delete clusterrolebindings {}
    ````  
    * Velero cluster role bindings:
    ````
    $ oc get clusterrolebindings | grep velero | awk '{print $1}' | xargs -I{} oc delete clusterrolebindings {}
    ```` 

