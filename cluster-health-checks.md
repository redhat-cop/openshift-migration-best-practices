[![Home](https://github.com/redhat-cop/openshift-migration-best-practices/raw/master/images/home.png)](./README.md) | [Planning <](./planning.md) Cluster health checks  [> Premigration testing](./premigration-testing.md)
---
# Cluster health checks

## General health

* Install and configure the [Prometheus Cluster Monitoring stack](https://docs.openshift.com/container-platform/4.5/monitoring/cluster_monitoring/configuring-the-monitoring-stack.html).
* Check the cluster nodes: ` $ oc get nodes`
* Check the persistent volumes (PVs) on the source cluster: `$ oc get pv`
  * Mounted PVs
  * Unmounted PVs
  * Abnormal configurations
  * PVs stuck in terminating state

## Capacity

* Available bandwidth between the source and target clusters:
  * Less than 10Gbps
  * 10 Gbps or more

* Amount of data to be migrated between the source and target clusters:
  * Less than 20 TB
  * 20 TB or more

## Performance

* Check cluster compute and memory utilization: `$ oc adm top node`
* Average response time of API calls in the source cluster:
  * Less than 50 ms
  * 50 ms or more

## Additional checks

Do instance types selected for the infrastructure on the target cluster meet the performance recommendations? 



