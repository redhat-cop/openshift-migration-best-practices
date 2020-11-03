[![Home](./images/home.png) | <](./README.md) Planning [> Cluster health checks](./cluster-health-checks.md)
---
# Planning

This section focuses on considerations to review when you plan your migration.

* **[Migration tools](#migration-tools)**
  * [Migration Toolkit for Containers](#migration-toolkit-for-containers)
    - [When to use MTC](#when-to-use-mtc)
    - [MTC documentation](#mtc-documentation)
  * [Upstream migration tools](#upstream-migration-tools)
    - [Comparison of MTC and upstream tools](#comparison-of-mtc-and-upstream-tools)
    - [Combining MTC and upstream tools](#combining-mtc-and-upstream-tools)
* **[Migration environment considerations](#migration-environment-considerations)**
* **[Migration workflows](#migration-workflows)**
  - [MTC workflow](#mtc-workflow)
  - [CI/CD workflow](#cicd-workflow)
* **[Network traffic migration strategies](#network-traffic-migration-strategies)**
  - ["Big Bang" migration](#big-bang-migration)
  - [Individual applications](#individual-applications)
  - [Canary-style migration of individual applications](#canary-style-migration-of-individual-applications)
  - [Audience-based migration of individual applications](#audience-based-migration-of-individual-applications)

## Migration tools

### Migration Toolkit for Containers

The Migration Toolkit for Containers (MTC) migrates an application workload, including  Kubernetes resources, data, and images, from an OpenShift 3 source cluster to an OpenShift 4 target cluster.

MTC performs the migration in two stages:

1. The application workload is backed up from the source cluster to object storage.
2. The application workload is restored to the target cluster from object storage.

**Migrating Kubernetes resources**

MTC migrates all namespaced resources, including Custom Resources. MTC can dynamically discover all the API resources in each referenced namespace.

MTC migrates some cluster-scoped resources. If a namespaced resource references a cluster-scoped resource, it is migrated. Migratable resources include persistent volumes (PVs) bound to a persistent volume claim, cluster role bindings, and security context constraints (SCCs).

**Migrating persistent volume data**

MTC has two options for migrating persistent volume data:

* **Move**: The PV definition is moved from the source cluster to the target cluster without touching the data. This is the fastest option.

* **Copy**: MTC copies either a *snapshot* or the *file system* of the PV.
  
See [PV move](./running-the-migration.md#pv-move) and [PV copy](./running-the-migration.md#pv-move) for requirements and details.

**Migrating internal images**

Internal images created by S2I builds are migrated. Each ImageStream reference in a given namespace is copied to the internal registry of the target cluster.

#### When to use MTC

Ideally, you could migrate an application from one cluster to another by redeploying the application from a pipeline and perhaps copying the persistent volume data.  

However, this might not be possible in the real world. A running application on the cluster might experience unforeseen changes and, over a period of time, drift away from the initial deploy. MTC can handle scenarios where you are not certain what your namespace contains and you want to migrate all its contents to a new cluster.

If you can redeploy your application from pipeline, that is the best option. If not, you should use MTC.

#### MTC documentation

* [Prerequisites](https://docs.openshift.com/container-platform/4.6/migration/migrating_3_4/migrating-application-workloads-3-4.html#migration-prerequisites_migrating-3-4)
* [About MTC](https://docs.openshift.com/container-platform/4.6/migration/migrating_3_4/migrating-application-workloads-3-4.html#migration-understanding-cam_migrating-3-4)
* [About data copy methods](https://docs.openshift.com/container-platform/4.6/migration/migrating_3_4/migrating-application-workloads-3-4.html#migration-understanding-data-copy-methods_migrating-3-4)

### Upstream migration tools

You can migrate PVs with [`pvc-migrate`](https://github.com/konveyor/pvc-migrate) or images with [`imagestream-migrate`](https://github.com/konveyor/imagestream-migrate).

These upstream tools offer advantages for large-scale migrations similar to the following example:

* ~50+ ImageStreams per namespace
* Multiple ~100GB+ persistent volumes

The tools are smaller and more focused. They are based on [Ansible Playbooks](https://docs.ansible.com/ansible/latest/user_guide/playbooks_intro.html), Python code snippets, [Rsync](https://rsync.samba.org/), and [Skopeo](https://github.com/containers/skopeo), which simplifies customization and debugging. Their performance is better than MTC.

#### Comparison of MTC and upstream tools

| | MTC  | Upstream tools |
| ------------- | ------------- | ------------- |
| **Processing** |Serial processing.<br><br> MTC processes one migration plan at a time, one backup/restore operation at a time. (MTC plans to support parallel execution in the future. See [velero-487](https://github.com/vmware-tanzu/velero/issues/487).)  |Parallel processing.<br><br> `imagestream-migrate` can perform parallel image migrations. |
| **Copying** |Two copy processes.<br><br> Backup and restore.  |Single copy process. |
| **Debugging** |Challenging.<br><br> A migration error can span different clusters, namespaces, and controller logs.   |Relatively easy. <br><br>Tools are based on Ansible and Python. |
| **Customization** |Difficult. <br><br>Requires updating the Golang code, recompiling, and then updating the MTC Operator to deliver the new version. |Relatively easy. <br><br>Tools are based on Ansible and Python.  |

#### Combining MTC and upstream tools

You can use a combination of upstream tools and MTC for migration.

Before migration, check your environment for the following requirements:

* There must be a direct network connection between the source and target clusters. A process running on each node of the source cluster must be able to connect to an exposed OpenShift Route on the target cluster.
* The host running `pvc-migrate` requires root access to each node of the source cluster.
* PVs must be OpenShift Container Storage (OCS). `pvc-migrate` does not support other storage providers.

The migration workflow is similar to the following procedure:

1. Configure MTC to omit PVs and/or images from the migration plan by setting the following parameters in the Migration Controller manifest:
    ```yaml
    disable_image_migration: true
    disable_pv_migration: true
    ```
2. Migrate the application workload with MTC.

3. Run `pvc-migrate` to migrate PVs and/or `imagestream-migrate` to migrate images.

## Migration environment considerations

OpenShift 4 introduces architectural changes and enhancements. The procedures that you used to manage your OpenShift 3 cluster might not apply to OpenShift 4.

You must consider the following:

* [Important differences between OpenShift 3 and 4 and their impact on migration](https://docs.openshift.com/container-platform/4.6/migration/migrating_3_4/planning-migration-3-to-4.html)
* How stored data will be migrated for stateful applications
* How much downtime your application can tolerate during migration
* How traffic will be redirected during migration

## Migration workflows

## Migration DNS considerations

In a typical migration scenario, the OpenShift 4 target cluster will in most cases be deployed in a different DNS domain from that of the OpenShift 3 source cluster, this implies that the applications migrated from the source to the target cluster will be accessible on a new URL different from the original one.  

Let’s assume for the rest of this section that the source cluster uses the base domain __ocp3.example.com__, and its applications default domain is __*.apps.ocp3.example.com__.  Unless special provisions are taken, the target cluster needs to be deployed on a different domain to avoid conflicts, for example __ocp4.example.com__, with the default applications defined as __*.apps.ocp4.example.com__.  

An application which uses the URL __http://app1.apps.ocp3.example.com__ in the source cluster, once migrate, will end up using the URL __http://app1.apps.ocp4.example.com__.  In most cases this is not a desirable result and would require the application clients to find out the new URL and adapt to it, which may not be a trivial or convenient thing to do.

If keeping the migrated applications in its original DNS domain is a requisite of the migration plan some options are available, three of them are described here.

First thing to consider is that once the OpenShift 4 cluster has been installed, its default DNS domain cannot be changed, this has some implications like:

* Creating a route without specifying the hostname parameter will use the default DNS domain for the target cluster (__*.apps.ocp4.example.com__). This can cause confusion when deploying a new application, especially after applying any of the options described next to have the migrated applications in the original source DNS domain.

* Internal services provided by the cluster like the web console, open authentication, alermanager and metrics UI, etc. will use the default DNS domain in their URLs, and with the exception of the web console these URLs cannot be easily changed. 

### Option 1: Isolate the target cluster DNS domain from the clients

To hide the OpenShift 4 DNS domain from the clients, a network element like an application load balancer or a reverse proxy is placed between the clients and the OpenShift 4 cluster. 

The Applications will be migrated from source to  target cluster and will get a FQDN in the target cluster’s default DNS domain (app1.apps.ocp4.example.com) 
The original application FQDN (app1.apps.ocp3.example.com) will be updated in the DNS server to return the IP of the network device.
The network device will contain rules to send the requests received for the application in the source domain to the load balancer in the target cluster using the target’s domain.

```
app1.apps.ocp3.example.com ---|reverse proxy|--->  app1.apps.ocp4.example.com
```

The Applications can be migrated using any of the network strategies described later, in particular it is possible to migrate one application at a time, having a wildcard DNS record for the *.apps.ocp3.example.com. domain pointing to the IP of the source cluster’s load balancer while having specific DNS records for each of the migrated applications pointing to the IP of the network device in front of the target cluster.  The specific DNS records have a higher priority over the wildcard one in a DNS server therefore there is no conflict when resolving the applications FQDNs.

Some additional considerations for this option are:

* The network device must terminate all secure TLS connections, otherwise if the connections are passed through to the Openshift load balancer, the target application’s FQDN will be exposed to the client and certificate errors will happen.

* The applications must support this configuration and not return any links referring the target cluster domain to the clients.  If such links are leaked back to the client inside the returned content, parts of the application may not load or work properly.

### Option 2: Set up the target cluster to accept the source DNS domain

It is possible to set up the target cluster to accept requests for the migrated applications in the same DNS domain that was defined in the source cluster. 

Two cases will be considered:

* Non secure (HTTP) access

  In the case of applications not requiring a secure TLS connection, two steps need to be taken:



### MTC workflow

MTC migrates applications from OCP 3 to OCP 4 in production and non-production environments.

The following diagram describes the MTC workflow:

![MTC-based](./images/mtc-promotion-flow.png)

### CI/CD workflow

A CI/CD pipeline deploys applications on OCP 4 production and non-production environments.

The following diagram describes a CI/CD workflow:

![CI-CD-based](./images/ci-cd-promotion-flow.png)

## Network traffic migration strategies

This section describes strategies for migrating network traffic for stateless applications.

Each strategy is based on this scenario:

* Applications are deployed on the 4.x cluster.
* If necessary, the 4.x router default certificate includes the 3.x wildcard SAN.
* Each application adds an additional route with the 3.x host name.
* Optional: The route with the 3.x host name contains an appropriate certificate.

### "Big Bang" migration

At migration, the 3.x wildcard DNS record is changed to point to the 4.x router virtual IP address (VIP).

![BigBang](./images/migration-strategy-bigbang.png)

### Individual applications

At migration, a new record is created for each application with the 3.x FQDN/host name pointing to the 4.x router VIP. This record takes precedence over the 3.x wildcard DNS record.

![Individual](./images/migration-strategy-individual.png)

### Canary-style migration of individual applications

A VIP/proxy with two backends, the 3.x router VIP and the 4.x router VIP, is created for each application.

At migration, a new record is created for each application with the 3.x FQDN/host name pointing to the VIP/proxy. This record takes precedence over the 3.x wildcard DNS record.

The proxy entry for the application is configured to route `X`% of the traffic to the 3.x router VIP and (100-`X`)% of the traffic to the 4.x VIP.

`X` is gradually moved from `100` to `0`.

![Canary](./images/migration-strategy-canary.png)

### Audience-based migration of individual applications

A VIP/proxy with two backends, the 3.x router VIP and the 4.x router VIP, is created for each application.

At migration, a new record is created for each application with the 3.x FQDN/host name pointing to the VIP/proxy. This record takes precedence over the 3.x wildcard DNS record.

The proxy entry for the application is configured to route traffic matching a given header pattern, for example, test customers, to the 4.x router VIP and the rest of the traffic to the 3.x VIP.

Traffic is moved to the 4.x VIP in waves until all the traffic is on the 4.x VIP.

![Audience](./images/migration-strategy-audience.png)

