# gea-docs-
# User Guide

<!--ts-->
* [User Guide](#user-guide)
   * [Installing 3scale](#installing-3scale)
      * [Prerequisites](#prerequisites)
      * [Basic installation](#basic-installation)
      * [Deployment Configuration Options](#deployment-configuration-options)
         * [Evaluation Installation](#evaluation-installation)
         * [External Databases Installation](#external-databases-installation)
         * [Enabling Pod Disruption Budgets](#enabling-pod-disruption-budgets)
         * [Setting custom affinity and tolerations](#setting-custom-affinity-and-tolerations)
         * [Setting custom compute resource requirements at component level](#setting-custom-compute-resource-requirements-at-component-level)
         * [Setting custom storage resource requirements](#setting-custom-storage-resource-requirements)
         * [Setting porta client to skip certificate verification](#setting-porta-client-to-skip-certificate-verification)

<!--te-->

## Installing 3scale

This section will take you through installing and deploying the 3scale solution via the 3scale operator,
using the [*APIManager*](apimanager-reference.md) custom resource.

Deploying the APIManager custom resource will make the operator begin processing and will deploy a
3scale solution from it.

### Prerequisites

* OpenShift Container Platform 4.1
* Deploying 3scale using the operator first requires that you follow the steps
in [quickstart guide](quickstart-guide.md) about *Install the 3scale operator*
* Some [Deployment Configuration Options](#deployment-configuration-options) require OpenShift infrastructure to provide availability for the following persistent volumes (PV):
  * 3 RWO (ReadWriteOnce) persistent volumes
  * 1 RWX (ReadWriteMany) persistent volume
    * 3scale's System component needs a RWX(ReadWriteMany) PersistentVolume for
      its FileStorage when System's FileStorage is configured to be
      a PVC (default behavior). System's FileStorage characteristics:
      * Contains configuration files read by the System component at run-time
      * Stores Static files (HTML, CSS, JS, etc) uploaded to System by its
        CMS feature, for the purpose of creating a Developer Portal
      * System can be scaled horizontally with multiple pods uploading and
        reading said static files, hence the need for a RWX PersistentVolume
        when APIManager is configured to use PVC as System's FileStorage


The RWX persistent volume must be configured to be group writable.
For a list of persistent volume types that support the required access modes,
see the [OpenShift documentation](https://access.redhat.com/documentation/en-us/openshift_container_platform/4.1/html-single/storage/index#persistent-volumes_understanding-persistent-storage)

### Basic installation

To deploy the minimal APIManager object with all default values, follow the following procedure:
1. Click *Catalog > Installed Operators*. From the list of *Installed Operator*s, click _3scale Operator_.
1. Click *API Manager > Create APIManager*
1. Create *APIManager* object with the following content.

```yaml
apiVersion: apps.3scale.net/v1alpha1
kind: APIManager
metadata:
  name: example-apimanager
spec:
  wildcardDomain: <wildcardDomain>
```

The `wildcardDomain` parameter can be any desired name you wish to give that resolves to the IP addresses
of OpenShift router nodes. Be sure to remove the placeholder marks for your parameters: `< >`.

When 3scale has been installed, a default *tenant* is created for you ready to be used,
with a fixed URL: `3scale-admin.${wildcardDomain}`.
For instance, when the `<wildCardDomain>` is `example.com`, then the Admin Portal URL would be:

```
https://3scale-admin.example.com
```

Optionally, you can create new tenants on the _MASTER portal URL_, with a fixed URL:

```
https://master.example.com
```

All required access credentials are stored in `system-seed` secret.

### Deployment Configuration Options

By default, the following deployment configuration options will be applied:
* Containers will have [k8s resources limits and requests](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/) specified. The advantage is twofold: ensure a minimum performance level and limit resources to allow external services and solutions to be allocated.
* Internal databases will be deployed.
* Filestorage will be based on *Persistent Volumes*, one of them requiring
  *RWX* access mode. Openshift must be configured to provide them when
  requested. For the  RWX persistent volume, a preexisting custom storage
  class to be used can be specified by the user if desired
  (see [Setting a custom Storage Class for System FileStorage RWX PVC-based installations](#setting-a-custom-storage-class-for-system-filestorage-rwx-pvc-based-installations))
* Mysql will be the internal relational database deployed.

Default configuration option is suitable for PoC or evaluation by a customer.

One, many or all of the default configuration options can be overridden with specific field values in
the [*APIManager*](apimanager-reference.md) custom resource.

#### Evaluation Installation
Containers will not have [k8s resources limits and requests](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/) specified.

* Small memory footprint
* Fast startup
* Runnable on laptop
* Suitable for presale/sales demos

```yaml
apiVersion: apps.3scale.net/v1alpha1
kind: APIManager
metadata:
  name: example-apimanager
spec:
  wildcardDomain: lvh.me
  resourceRequirementsEnabled: false
```

Check [*APIManager*](apimanager-reference.md) custom resource for reference.

#### External Databases Installation
Suitable for production use where customers want self-managed databases.

3scale API Management has been tested and it’s supported with the following database versions:

| Database | Version |
| :--- | :--- |
| MySQL | 8.0 |


3scale API Management requires the following database instances:
* System RDBMS


* **System database secret**
MySQL database instance must be deployed by the customer. Then fill connection settings.

Example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: system-database
stringData:
  URL: "mysql2://root:password0@system-mysql/system"
  DB_USER: "mysql"
  DB_PASSWORD: "password1"
type: Opaque
```

Secret name must be `system-database`.

See [System database secret](apimanager-reference.md#system-database) for reference.

## NFS Filestorage Installation ##########
3scale’s FileStorage being in a S3 service instead of in a PVC.

- Create an object definition for the PV
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0001 
spec:
  capacity:
    storage: 5Gi 
  accessModes:
  - ReadWriteOnce 
  nfs: 
    path: /tmp 
    server: 172.17.0.2 
  persistentVolumeReclaimPolicy: Retain 
```
- The name of the volume. This is the PV identity in various oc <command> pod commands.
- The amount of storage allocated to this volume.
- Though this appears to be related to controlling access to the volume, it is actually used similarly to labels and used to match a PVC to a PV. Currently, no access rules are enforced based on the accessModes.
- The volume type being used, in this case the nfs plugin.
- The path that is exported by the NFS server.
- The hostname or IP address of the NFS server.
- The reclaim policy for the PV. This defines what happens to a volume when released.
- Verify that the PV is correctly created
```bash
oc get pv
NAME     LABELS    CAPACITY     ACCESSMODES   STATUS      CLAIM  REASON    AGE
pv0001   <none>    5Gi          RWO           Available                    31s
```
- Create a persistent volume claim that binds to the new PV:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-claim1
spec:
  accessModes:
    - ReadWriteOnce 
  resources:
    requests:
      storage: 5Gi 
  volumeName: pv0001
  storageClassName: ""
```
- Verify with the following command
```bash
oc get pvc
NAME         STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
nfs-claim1   Bound    pv0001   5Gi        RWO                           2m
```

Check [*APIManager DatabaseSpec*](apimanager-reference.md#DatabaseSpec) for reference.

#### Enabling Pod Disruption Budgets
The 3scale API Management solution DeploymentConfigs deployed and managed by the
APIManager will be configured with Kubernetes Pod Disruption Budgets
enabled.

A Pod Disruption Budget limits the number of pods related to an application
(in this case, pods of a DeploymentConfig) that are down simultaneously
from **voluntary disruptions**.

When enabling the Pod Disruption Budgets for non-database DeploymentConfigs will
be set with a setting of maximum of 1 unavailable pod at any given time.
Database-related DeploymentConfigs are excluded from this configuration.
Additionally, `system-sphinx` DeploymentConfig is also excluded.

For details about the behavior of Pod Disruption Budgets, what they perform and
what constitutes a 'voluntary disruption' see the following
[Kubernetes Documentation](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)

Pods which are deleted or unavailable due to a rolling upgrade to an application
do count against the disruption budget, but the DeploymentConfigs are not
limited by Pod Disruption Budgets when doing rolling upgrades or they are
scaled up/down.

In order for the Pod Disruption Budget setting to be effective the number of
replicas of each non-database component has to be set to a value greater than 1.

Example:
```yaml
apiVersion: apps.3scale.net/v1alpha1
kind: APIManager
metadata:
  name: example-apimanager
spec:
  wildcardDomain: lvh.me
  apicast:
    stagingSpec:
      replicas: 2
    productionSpec:
      replicas: 2
  backend:
    listenerSpec:
      replicas: 2
    workerSpec:
      replicas: 2
    cronSpec:
      replicas: 2
  system:
    appSpec:
      replicas: 2
    sidekiqSpec:
      replicas: 2
  zync:
    appSpec:
      replicas: 2
    queSpec:
      replicas: 2
  podDisruptionBudget:
    enabled: true
```

## Setting custom affinity and tolerations

Kubernetes [Affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity
) and [Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
can be customized in a 3scale API Management solution through APIManager
CR attributes in order to customize where/how the different 3scale components of
an installation are scheduled onto Kubernetes Nodes.

For example, setting a custom node affinity for backend listener
and custom tolerations for system's memcached would be done in the
following way:

```yaml
apiVersion: apps.3scale.net/v1alpha1
kind: APIManager
metadata:
  name: example-apimanager
spec:
  backend:
    listenerSpec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: "kubernetes.io/hostname"
                operator: In
                values:
                - ip-10-96-1-105
              - key: "beta.kubernetes.io/arch"
                operator: In
                values:
                - amd64
  system:
    memcachedTolerations:
    - key: key1
      value: value1
      operator: Equal
      effect: NoSchedule
    - key: key2
      value: value2
      operator: Equal
      effect: NoSchedule
```

See [APIManager reference](apimanager-reference.md) for a full list of
attributes related to affinity and tolerations.

## Setting custom storage resource requirements

Openshift [storage resource requirements](https://docs.openshift.com/container-platform/4.5/storage/persistent_storage/persistent-storage-local.html#create-local-pvc_persistent-storage-local)
can be customized through APIManager CR attributes.

This is the list of 3scale PVC's resources that can be customized with examples.

* *System Shared (RWX) Storage PVC*

```
apiVersion: apps.3scale.net/v1alpha1
kind: APIManager
metadata:
  name: apimanager1
spec:
  wildcardDomain: example.com
  system:
    fileStorage:
      persistentVolumeClaim:
        resources:
          requests: 2Gi
```

* *MySQL (RWO) PVC*
```
apiVersion: apps.3scale.net/v1alpha1
kind: APIManager
metadata:
  name: apimanager1
spec:
  wildcardDomain: example.com
  system:
    database:
      mysql:
        persistentVolumeClaim:
          resources:
            requests: 2Gi
```


*IMPORTANT NOTE*: Storage resource requirements are **usually** install only attributes.
Only when the underlying PersistentVolume's storageclass allows resizing, storage resource requirements can be modified after installation.
Check [Expanding persistent volumes](https://docs.openshift.com/container-platform/4.5/storage/expanding-persistent-volumes.html) official doc for more information.

## Setting porta client to skip certificate verification
Whenever a controller reconciles an object it creates a new porta client to make API calls. That client is configured to verify the server's certificate chain by default. For development/testing purposes, you may want the client to skip certificate verification when reconciling an object. This can be done using the annotation `insecure_skip_verify: true`, which can be added to the following objects:
* ActiveDoc
* Application
* Backend
* CustomPolicyDefinition
* DeveloperAccount
* DeveloperUser
* OpenAPI - backend and product
* Product
* ProxyConfigPromote
* Tenant
