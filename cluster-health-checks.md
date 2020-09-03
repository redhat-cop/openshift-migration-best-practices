[![Home](https://github.com/redhat-cop/openshift-migration-best-practices/raw/master/images/home.png)](./README.md) [Planning <](./planning.md) |  [> Premigration testing](./premigration-testing.md)
---
# Cluster health checks


## Capacity



## Performance

## Additional checks



What is the compute and memory utilization of the cluster?
Command: `oc adm top node`






What is the current node health?
Command: `oc get nodes`






What is the average response time from API calls in the source clusters?

<50ms
>50ms


Have you checked that the instance types picked for the infrastructure on the target OpenShift Cluster meet the performance recommendations? 

Have you checked the persistent volumes across the OpenShift 3 cluster to determine if they are mounted, not mounted, or have any abnormal configurations?
Command: `oc get pv`

PVs mounted
no abnormal configurations
PVs unmounted
PVs stuck in terminating
What is the available bandwidth between source and destination clusters?

10Gbps or greater
Less than 10Gbps
How much data is being migrated between the source and destination cluster? 

Less than 20TB
Great than 20TB
