---
reviewers:
- jsafrane
- saad-ali
- thockin
- msau42
title: Storage Classes
content_type: concept
weight: 40
---

<!-- overview -->

This document describes the concept of a StorageClass in Kubernetes. Familiarity
with [volumes](/docs/concepts/storage/volumes/) and
[persistent volumes](/docs/concepts/storage/persistent-volumes) is suggested.

<!-- body -->

## Introduction

A StorageClass provides a way for administrators to describe the "classes" of
storage they offer. Different classes might map to quality-of-service levels,
or to backup policies, or to arbitrary policies determined by the cluster
administrators. Kubernetes itself is unopinionated about what classes
represent. This concept is sometimes called "profiles" in other storage
systems.

## The StorageClass Resource

Each StorageClass contains the fields `provisioner`, `parameters`, and
`reclaimPolicy`, which are used when a PersistentVolume belonging to the
class needs to be dynamically provisioned.

The name of a StorageClass object is significant, and is how users can
request a particular class. Administrators set the name and other parameters
of a class when first creating StorageClass objects, and the objects cannot
be updated once they are created.

Administrators can specify a default StorageClass only for PVCs that don't
request any particular class to bind to: see the
[PersistentVolumeClaim section](/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)
for details.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
reclaimPolicy: Retain
allowVolumeExpansion: true
mountOptions:
  - debug
volumeBindingMode: Immediate
```

### Default StorageClass

When a PVC does not specify a `storageClassName`, the default StorageClass is
used. The cluster can only have one default StorageClass. If more than one
default StorageClass is accidentally set, the newest default is used when the
PVC is dynamically provisioned.

For instructions on setting the default StorageClass, see
[Change the default StorageClass](/docs/tasks/administer-cluster/change-default-storage-class/).
Note that certain cloud providers may already define a default StorageClass.

### Provisioner

Each StorageClass has a provisioner that determines what volume plugin is used
for provisioning PVs. This field must be specified.

| Volume Plugin        | Internal Provisioner |            Config Example             |
| :------------------- | :------------------: | :-----------------------------------: |
| AWSElasticBlockStore |       &#x2713;       |          [AWS EBS](#aws-ebs)          |
| AzureFile            |       &#x2713;       |       [Azure File](#azure-file)       |
| AzureDisk            |       &#x2713;       |       [Azure Disk](#azure-disk)       |
| CephFS               |          -           |                   -                   |
| Cinder               |       &#x2713;       | [OpenStack Cinder](#openstack-cinder) |
| FC                   |          -           |                   -                   |
| FlexVolume           |          -           |                   -                   |
| GCEPersistentDisk    |       &#x2713;       |           [GCE PD](#gce-pd)           |
| iSCSI                |          -           |                   -                   |
| NFS                  |          -           |              [NFS](#nfs)              |
| RBD                  |       &#x2713;       |         [Ceph RBD](#ceph-rbd)         |
| VsphereVolume        |       &#x2713;       |          [vSphere](#vsphere)          |
| PortworxVolume       |       &#x2713;       |  [Portworx Volume](#portworx-volume)  |
| Local                |          -           |            [Local](#local)            |

You are not restricted to specifying the "internal" provisioners
listed here (whose names are prefixed with "kubernetes.io" and shipped
alongside Kubernetes). You can also run and specify external provisioners,
which are independent programs that follow a [specification](https://git.k8s.io/design-proposals-archive/storage/volume-provisioning.md)
defined by Kubernetes. Authors of external provisioners have full discretion
over where their code lives, how the provisioner is shipped, how it needs to be
run, what volume plugin it uses (including Flex), etc. The repository
[kubernetes-sigs/sig-storage-lib-external-provisioner](https://github.com/kubernetes-sigs/sig-storage-lib-external-provisioner)
houses a library for writing external provisioners that implements the bulk of
the specification. Some external provisioners are listed under the repository
[kubernetes-sigs/sig-storage-lib-external-provisioner](https://github.com/kubernetes-sigs/sig-storage-lib-external-provisioner).

For example, NFS doesn't provide an internal provisioner, but an external
provisioner can be used. There are also cases when 3rd party storage
vendors provide their own external provisioner.

### Reclaim Policy

PersistentVolumes that are dynamically created by a StorageClass will have the
[reclaim policy](/docs/concepts/storage/persistent-volumes/#reclaiming)
specified in the `reclaimPolicy` field of the class, which can be
either `Delete` or `Retain`. If no `reclaimPolicy` is specified when a
StorageClass object is created, it will default to `Delete`.

PersistentVolumes that are created manually and managed via a StorageClass will have
whatever reclaim policy they were assigned at creation.

### Allow Volume Expansion

PersistentVolumes can be configured to be expandable. This feature when set to `true`,
allows the users to resize the volume by editing the corresponding PVC object.

The following types of volumes support volume expansion, when the underlying
StorageClass has the field `allowVolumeExpansion` set to true.

{{< table caption = "Table of Volume types and the version of Kubernetes they require"  >}}

| Volume type          | Required Kubernetes version |
| :------------------- | :-------------------------- |
| gcePersistentDisk    | 1.11                        |
| awsElasticBlockStore | 1.11                        |
| Cinder               | 1.11                        |
| rbd                  | 1.11                        |
| Azure File           | 1.11                        |
| Azure Disk           | 1.11                        |
| Portworx             | 1.11                        |
| FlexVolume           | 1.13                        |
| CSI                  | 1.14 (alpha), 1.16 (beta)   |

{{< /table >}}

{{< note >}}
You can only use the volume expansion feature to grow a Volume, not to shrink it.
{{< /note >}}

### Mount Options

PersistentVolumes that are dynamically created by a StorageClass will have the
mount options specified in the `mountOptions` field of the class.

If the volume plugin does not support mount options but mount options are
specified, provisioning will fail. Mount options are not validated on either
the class or PV. If a mount option is invalid, the PV mount fails.

### Volume Binding Mode

The `volumeBindingMode` field controls when
[volume binding and dynamic provisioning](/docs/concepts/storage/persistent-volumes/#provisioning)
should occur. When unset, "Immediate" mode is used by default.

The `Immediate` mode indicates that volume binding and dynamic
provisioning occurs once the PersistentVolumeClaim is created. For storage
backends that are topology-constrained and not globally accessible from all Nodes
in the cluster, PersistentVolumes will be bound or provisioned without knowledge of the Pod's scheduling
requirements. This may result in unschedulable Pods.

A cluster administrator can address this issue by specifying the `WaitForFirstConsumer` mode which
will delay the binding and provisioning of a PersistentVolume until a Pod using the PersistentVolumeClaim is created.
PersistentVolumes will be selected or provisioned conforming to the topology that is
specified by the Pod's scheduling constraints. These include, but are not limited to, [resource
requirements](/docs/concepts/configuration/manage-resources-containers/),
[node selectors](/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector),
[pod affinity and
anti-affinity](/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity),
and [taints and tolerations](/docs/concepts/scheduling-eviction/taint-and-toleration).

The following plugins support `WaitForFirstConsumer` with dynamic provisioning:

- [AWSElasticBlockStore](#aws-ebs)
- [GCEPersistentDisk](#gce-pd)
- [AzureDisk](#azure-disk)

The following plugins support `WaitForFirstConsumer` with pre-created PersistentVolume binding:

- All of the above
- [Local](#local)

[CSI volumes](/docs/concepts/storage/volumes/#csi) are also supported with dynamic provisioning
and pre-created PVs, but you'll need to look at the documentation for a specific CSI driver
to see its supported topology keys and examples.

{{< note >}}
If you choose to use `WaitForFirstConsumer`, do not use `nodeName` in the Pod spec
to specify node affinity.
If `nodeName` is used in this case, the scheduler will be bypassed and PVC will remain in `pending` state.

Instead, you can use node selector for hostname in this case as shown below.
{{< /note >}}

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: task-pv-pod
spec:
  nodeSelector:
    kubernetes.io/hostname: kube-01
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
        claimName: task-pv-claim
  containers:
    - name: task-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: task-pv-storage
```

### Allowed Topologies

When a cluster operator specifies the `WaitForFirstConsumer` volume binding mode, it is no longer necessary
to restrict provisioning to specific topologies in most situations. However,
if still required, `allowedTopologies` can be specified.

This example demonstrates how to restrict the topology of provisioned volumes to specific
zones and should be used as a replacement for the `zone` and `zones` parameters for the
supported plugins.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
volumeBindingMode: WaitForFirstConsumer
allowedTopologies:
- matchLabelExpressions:
  - key: failure-domain.beta.kubernetes.io/zone
    values:
    - us-central-1a
    - us-central-1b
```

## Parameters

Storage Classes have parameters that describe volumes belonging to the storage
class. Different parameters may be accepted depending on the `provisioner`. For
example, the value `io1`, for the parameter `type`, and the parameter
`iopsPerGB` are specific to EBS. When a parameter is omitted, some default is
used.

There can be at most 512 parameters defined for a StorageClass.
The total length of the parameters object including its keys and values cannot
exceed 256 KiB.

### AWS EBS

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
provisioner: kubernetes.io/aws-ebs
parameters:
  type: io1
  iopsPerGB: "10"
  fsType: ext4
```

- `type`: `io1`, `gp2`, `sc1`, `st1`. See
  [AWS docs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSVolumeTypes.html)
  for details. Default: `gp2`.
- `zone` (Deprecated): AWS zone. If neither `zone` nor `zones` is specified, volumes are
  generally round-robin-ed across all active zones where Kubernetes cluster
  has a node. `zone` and `zones` parameters must not be used at the same time.
- `zones` (Deprecated): A comma separated list of AWS zone(s). If neither `zone` nor `zones`
  is specified, volumes are generally round-robin-ed across all active zones
  where Kubernetes cluster has a node. `zone` and `zones` parameters must not
  be used at the same time.
- `iopsPerGB`: only for `io1` volumes. I/O operations per second per GiB. AWS
  volume plugin multiplies this with size of requested volume to compute IOPS
  of the volume and caps it at 20 000 IOPS (maximum supported by AWS, see
  [AWS docs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSVolumeTypes.html)).
  A string is expected here, i.e. `"10"`, not `10`.
- `fsType`: fsType that is supported by kubernetes. Default: `"ext4"`.
- `encrypted`: denotes whether the EBS volume should be encrypted or not.
  Valid values are `"true"` or `"false"`. A string is expected here,
  i.e. `"true"`, not `true`.
- `kmsKeyId`: optional. The full Amazon Resource Name of the key to use when
  encrypting the volume. If none is supplied but `encrypted` is true, a key is
  generated by AWS. See AWS docs for valid ARN value.

{{< note >}}
`zone` and `zones` parameters are deprecated and replaced with
[allowedTopologies](#allowed-topologies)
{{< /note >}}

### GCE PD

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
  fstype: ext4
  replication-type: none
```

- `type`: `pd-standard` or `pd-ssd`. Default: `pd-standard`
- `zone` (Deprecated): GCE zone. If neither `zone` nor `zones` is specified, volumes are
  generally round-robin-ed across all active zones where Kubernetes cluster has
  a node. `zone` and `zones` parameters must not be used at the same time.
- `zones` (Deprecated): A comma separated list of GCE zone(s). If neither `zone` nor `zones`
  is specified, volumes are generally round-robin-ed across all active zones
  where Kubernetes cluster has a node. `zone` and `zones` parameters must not
  be used at the same time.
- `fstype`: `ext4` or `xfs`. Default: `ext4`. The defined filesystem type must be supported by the host operating system.

- `replication-type`: `none` or `regional-pd`. Default: `none`.

If `replication-type` is set to `none`, a regular (zonal) PD will be provisioned.

If `replication-type` is set to `regional-pd`, a
[Regional Persistent Disk](https://cloud.google.com/compute/docs/disks/#repds)
will be provisioned. It's highly recommended to have
`volumeBindingMode: WaitForFirstConsumer` set, in which case when you create
a Pod that consumes a PersistentVolumeClaim which uses this StorageClass, a
Regional Persistent Disk is provisioned with two zones. One zone is the same
as the zone that the Pod is scheduled in. The other zone is randomly picked
from the zones available to the cluster. Disk zones can be further constrained
using `allowedTopologies`.

{{< note >}}
`zone` and `zones` parameters are deprecated and replaced with
[allowedTopologies](#allowed-topologies). When
[GCE CSI Migration](/docs/concepts/storage/volumes/#gce-csi-migration) is
enabled, a GCE PD volume can be provisioned in a topology that does not match
any nodes, but any pod trying to use that volume will fail to schedule. With
legacy pre-migration GCE PD, in this case an error will be produced
instead at provisioning time. GCE CSI Migration is enabled by default beginning
from the Kubernetes 1.23 release.
{{< /note >}}

### NFS

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: example-nfs
provisioner: example.com/external-nfs
parameters:
  server: nfs-server.example.com
  path: /share
  readOnly: "false"
```

- `server`: Server is the hostname or IP address of the NFS server.
- `path`: Path that is exported by the NFS server.
- `readOnly`: A flag indicating whether the storage will be mounted as read only (default false).

Kubernetes doesn't include an internal NFS provisioner.
You need to use an external provisioner to create a StorageClass for NFS.
Here are some examples:

- [NFS Ganesha server and external provisioner](https://github.com/kubernetes-sigs/nfs-ganesha-server-and-external-provisioner)
- [NFS subdir external provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)

### OpenStack Cinder

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gold
provisioner: kubernetes.io/cinder
parameters:
  availability: nova
```

- `availability`: Availability Zone. If not specified, volumes are generally
  round-robin-ed across all active zones where Kubernetes cluster has a node.

{{< note >}}
{{< feature-state state="deprecated" for_k8s_version="v1.11" >}}
This internal provisioner of OpenStack is deprecated. Please use
[the external cloud provider for OpenStack](https://github.com/kubernetes/cloud-provider-openstack).
{{< /note >}}

### vSphere

There are two types of provisioners for vSphere storage classes:

- [CSI provisioner](#vsphere-provisioner-csi): `csi.vsphere.vmware.com`
- [vCP provisioner](#vcp-provisioner): `kubernetes.io/vsphere-volume`

In-tree provisioners are [deprecated](/blog/2019/12/09/kubernetes-1-17-feature-csi-migration-beta/#why-are-we-migrating-in-tree-plugins-to-csi).
For more information on the CSI provisioner, see
[Kubernetes vSphere CSI Driver](https://vsphere-csi-driver.sigs.k8s.io/) and
[vSphereVolume CSI migration](/docs/concepts/storage/volumes/#vsphere-csi-migration).

#### CSI Provisioner {#vsphere-provisioner-csi}

The vSphere CSI StorageClass provisioner works with Tanzu Kubernetes clusters.
For an example, refer to the [vSphere CSI repository](https://github.com/kubernetes-sigs/vsphere-csi-driver/blob/master/example/vanilla-k8s-RWM-filesystem-volumes/example-sc.yaml).

#### vCP Provisioner

The following examples use the VMware Cloud Provider (vCP) StorageClass provisioner.

1. Create a StorageClass with a user specified disk format.

   ```yaml
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
     name: fast
   provisioner: kubernetes.io/vsphere-volume
   parameters:
     diskformat: zeroedthick
   ```

   `diskformat`: `thin`, `zeroedthick` and `eagerzeroedthick`. Default: `"thin"`.

2. Create a StorageClass with a disk format on a user specified datastore.

   ```yaml
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
     name: fast
   provisioner: kubernetes.io/vsphere-volume
   parameters:
     diskformat: zeroedthick
     datastore: VSANDatastore
   ```

   `datastore`: The user can also specify the datastore in the StorageClass.
   The volume will be created on the datastore specified in the StorageClass,
   which in this case is `VSANDatastore`. This field is optional. If the
   datastore is not specified, then the volume will be created on the datastore
   specified in the vSphere config file used to initialize the vSphere Cloud
   Provider.

3. Storage Policy Management inside kubernetes

   - Using existing vCenter SPBM policy

     One of the most important features of vSphere for Storage Management is
     policy based Management. Storage Policy Based Management (SPBM) is a
     storage policy framework that provides a single unified control plane
     across a broad range of data services and storage solutions. SPBM enables
     vSphere administrators to overcome upfront storage provisioning challenges,
     such as capacity planning, differentiated service levels and managing
     capacity headroom.

     The SPBM policies can be specified in the StorageClass using the
     `storagePolicyName` parameter.

   - Virtual SAN policy support inside Kubernetes

     Vsphere Infrastructure (VI) Admins will have the ability to specify custom
     Virtual SAN Storage Capabilities during dynamic volume provisioning. You
     can now define storage requirements, such as performance and availability,
     in the form of storage capabilities during dynamic volume provisioning.
     The storage capability requirements are converted into a Virtual SAN
     policy which are then pushed down to the Virtual SAN layer when a
     persistent volume (virtual disk) is being created. The virtual disk is
     distributed across the Virtual SAN datastore to meet the requirements.

     You can see [Storage Policy Based Management for dynamic provisioning of volumes](https://github.com/vmware-archive/vsphere-storage-for-kubernetes/blob/fa4c8b8ad46a85b6555d715dd9d27ff69839df53/documentation/policy-based-mgmt.md)
     for more details on how to use storage policies for persistent volumes
     management.

There are few
[vSphere examples](https://github.com/kubernetes/examples/tree/master/staging/volumes/vsphere)
which you try out for persistent volume management inside Kubernetes for vSphere.

### Ceph RBD

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/rbd
parameters:
  monitors: 10.16.153.105:6789
  adminId: kube
  adminSecretName: ceph-secret
  adminSecretNamespace: kube-system
  pool: kube
  userId: kube
  userSecretName: ceph-secret-user
  userSecretNamespace: default
  fsType: ext4
  imageFormat: "2"
  imageFeatures: "layering"
```

- `monitors`: Ceph monitors, comma delimited. This parameter is required.
- `adminId`: Ceph client ID that is capable of creating images in the pool.
  Default is "admin".
- `adminSecretName`: Secret Name for `adminId`. This parameter is required.
  The provided secret must have type "kubernetes.io/rbd".
- `adminSecretNamespace`: The namespace for `adminSecretName`. Default is "default".
- `pool`: Ceph RBD pool. Default is "rbd".
- `userId`: Ceph client ID that is used to map the RBD image. Default is the
  same as `adminId`.
- `userSecretName`: The name of Ceph Secret for `userId` to map RBD image. It
  must exist in the same namespace as PVCs. This parameter is required.
  The provided secret must have type "kubernetes.io/rbd", for example created in this
  way:

  ```shell
  kubectl create secret generic ceph-secret --type="kubernetes.io/rbd" \
    --from-literal=key='QVFEQ1pMdFhPUnQrSmhBQUFYaERWNHJsZ3BsMmNjcDR6RFZST0E9PQ==' \
    --namespace=kube-system
  ```

- `userSecretNamespace`: The namespace for `userSecretName`.
- `fsType`: fsType that is supported by kubernetes. Default: `"ext4"`.
- `imageFormat`: Ceph RBD image format, "1" or "2". Default is "2".
- `imageFeatures`: This parameter is optional and should only be used if you
  set `imageFormat` to "2". Currently supported features are `layering` only.
  Default is "", and no features are turned on.

### Azure Disk

#### Azure Unmanaged Disk storage class {#azure-unmanaged-disk-storage-class}

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
provisioner: kubernetes.io/azure-disk
parameters:
  skuName: Standard_LRS
  location: eastus
  storageAccount: azure_storage_account_name
```

- `skuName`: Azure storage account Sku tier. Default is empty.
- `location`: Azure storage account location. Default is empty.
- `storageAccount`: Azure storage account name. If a storage account is provided,
  it must reside in the same resource group as the cluster, and `location` is
  ignored. If a storage account is not provided, a new storage account will be
  created in the same resource group as the cluster.

#### Azure Disk storage class (starting from v1.7.2) {#azure-disk-storage-class}

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
provisioner: kubernetes.io/azure-disk
parameters:
  storageaccounttype: Standard_LRS
  kind: managed
```

- `storageaccounttype`: Azure storage account Sku tier. Default is empty.
- `kind`: Possible values are `shared`, `dedicated`, and `managed` (default).
  When `kind` is `shared`, all unmanaged disks are created in a few shared
  storage accounts in the same resource group as the cluster. When `kind` is
  `dedicated`, a new dedicated storage account will be created for the new
  unmanaged disk in the same resource group as the cluster. When `kind` is
  `managed`, all managed disks are created in the same resource group as
  the cluster.
- `resourceGroup`: Specify the resource group in which the Azure disk will be created.
  It must be an existing resource group name. If it is unspecified, the disk will be
  placed in the same resource group as the current Kubernetes cluster.

* Premium VM can attach both Standard_LRS and Premium_LRS disks, while Standard
  VM can only attach Standard_LRS disks.
* Managed VM can only attach managed disks and unmanaged VM can only attach
  unmanaged disks.

### Azure File

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azurefile
provisioner: kubernetes.io/azure-file
parameters:
  skuName: Standard_LRS
  location: eastus
  storageAccount: azure_storage_account_name
```

- `skuName`: Azure storage account Sku tier. Default is empty.
- `location`: Azure storage account location. Default is empty.
- `storageAccount`: Azure storage account name. Default is empty. If a storage
  account is not provided, all storage accounts associated with the resource
  group are searched to find one that matches `skuName` and `location`. If a
  storage account is provided, it must reside in the same resource group as the
  cluster, and `skuName` and `location` are ignored.
- `secretNamespace`: the namespace of the secret that contains the Azure Storage
  Account Name and Key. Default is the same as the Pod.
- `secretName`: the name of the secret that contains the Azure Storage Account Name and
  Key. Default is `azure-storage-account-<accountName>-secret`
- `readOnly`: a flag indicating whether the storage will be mounted as read only.
  Defaults to false which means a read/write mount. This setting will impact the
  `ReadOnly` setting in VolumeMounts as well.

During storage provisioning, a secret named by `secretName` is created for the
mounting credentials. If the cluster has enabled both
[RBAC](/docs/reference/access-authn-authz/rbac/) and
[Controller Roles](/docs/reference/access-authn-authz/rbac/#controller-roles),
add the `create` permission of resource `secret` for clusterrole
`system:controller:persistent-volume-binder`.

In a multi-tenancy context, it is strongly recommended to set the value for
`secretNamespace` explicitly, otherwise the storage account credentials may
be read by other users.

### Portworx Volume

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: portworx-io-priority-high
provisioner: kubernetes.io/portworx-volume
parameters:
  repl: "1"
  snap_interval: "70"
  priority_io: "high"
```

- `fs`: filesystem to be laid out: `none/xfs/ext4` (default: `ext4`).
- `block_size`: block size in Kbytes (default: `32`).
- `repl`: number of synchronous replicas to be provided in the form of
  replication factor `1..3` (default: `1`) A string is expected here i.e.
  `"1"` and not `1`.
- `priority_io`: determines whether the volume will be created from higher
  performance or a lower priority storage `high/medium/low` (default: `low`).
- `snap_interval`: clock/time interval in minutes for when to trigger snapshots.
  Snapshots are incremental based on difference with the prior snapshot, 0
  disables snaps (default: `0`). A string is expected here i.e.
  `"70"` and not `70`.
- `aggregation_level`: specifies the number of chunks the volume would be
  distributed into, 0 indicates a non-aggregated volume (default: `0`). A string
  is expected here i.e. `"0"` and not `0`
- `ephemeral`: specifies whether the volume should be cleaned-up after unmount
  or should be persistent. `emptyDir` use case can set this value to true and
  `persistent volumes` use case such as for databases like Cassandra should set
  to false, `true/false` (default `false`). A string is expected here i.e.
  `"true"` and not `true`.

### Local

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

Local volumes do not currently support dynamic provisioning, however a StorageClass
should still be created to delay volume binding until Pod scheduling. This is
specified by the `WaitForFirstConsumer` volume binding mode.

Delaying volume binding allows the scheduler to consider all of a Pod's
scheduling constraints when choosing an appropriate PersistentVolume for a
PersistentVolumeClaim.
