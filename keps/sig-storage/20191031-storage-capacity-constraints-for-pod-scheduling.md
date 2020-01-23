---
title: Storage Capacity Constraints for Pod Scheduling
authors:
  - "@pohly"
  - "@cofyc"
owning-sig: sig-storage
participating-sigs:
  - sig-scheduling
reviewers:
  - "@saad-ali"
approvers:
  - "@saad-ali"
editor: "@pohly"
creation-date: 2019-09-19
last-updated: 2020-01-17
status: implementable
see-also:
  - "https://docs.google.com/document/d/1WtX2lRJjZ03RBdzQIZY3IOvmoYiF5JxDX35-SsCIAfg"
---

# Storage Capacity Constraints for Pod Scheduling

## Table of Contents

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Ephemeral PMEM volume for Redis](#ephemeral-pmem-volume-for-redis)
    - [Different LVM configurations](#different-lvm-configurations)
    - [Network attached storage](#network-attached-storage)
    - [Custom schedulers](#custom-schedulers)
    - [Operators for applications](#operators-for-applications)
  - [Caching remaining capacity via the API server](#caching-remaining-capacity-via-the-api-server)
  - [Identifying storage pools](#identifying-storage-pools)
  - [Size of ephemeral inline volumes](#size-of-ephemeral-inline-volumes)
  - [Pod scheduling](#pod-scheduling)
  - [API](#api)
    - [CSIStoragePool](#csistoragepool)
      - [Example: local storage](#example-local-storage)
      - [Example: affect of storage classes](#example-affect-of-storage-classes)
      - [Example: network attached storage](#example-network-attached-storage)
    - [CSIDriver.spec.storageCapacity](#csidriverspecstoragecapacity)
  - [Updating capacity information with external-provisioner](#updating-capacity-information-with-external-provisioner)
    - [Only local volumes](#only-local-volumes)
    - [Central provisioning](#central-provisioning)
    - [CSIStoragePool lifecycle](#csistoragepool-lifecycle)
  - [Using capacity information](#using-capacity-information)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
  - [CSIDriver.Status](#csidriverstatus)
    - [Example: local storage](#example-local-storage-1)
    - [Example: affect of storage classes](#example-affect-of-storage-classes-1)
    - [Example: network attached storage](#example-network-attached-storage-1)
  - [Prior work](#prior-work)
<!-- /toc -->

## Release Signoff Checklist

- [ ] kubernetes/enhancements issue in release milestone, which links
      to KEP (this should be a link to the KEP location in
      kubernetes/enhancements, not the initial KEP PR)
- [ ] KEP approvers have set the KEP status to `implementable`
- [ ] Design details are appropriately documented
- [ ] Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
- [ ] Graduation criteria is in place
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in
      [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation e.g., additional design documents,
      links to mailing list discussions/SIG meetings, relevant
      PRs/issues, release notes

## Summary

There are two types of volumes that are getting created after scheduling a pod onto a node:
- [ephemeral inline
  volumes](https://kubernetes-csi.github.io/docs/ephemeral-local-volumes.html)
- persistent volumes with [delayed
  binding](https://kubernetes.io/docs/concepts/storage/storage-classes/#volume-binding-mode)
  (`WaitForFirstConsumer`)

In both cases the Kubernetes scheduler currently picks a node without
knowing whether the storage system has enough capacity left for
creating a volume of the requested size. In the first case, `kubelet`
will ask the CSI node service to stage the volume. Depending on the
CSI driver, this involves creating the volume. In the second case, the
`external-provisioner` will note that a `PVC` is now ready to be
provisioned and ask the CSI controller service to create the volume
such that is usable by the node (via
[`CreateVolumeRequest.accessibility_requirements`](https://kubernetes-csi.github.io/docs/topology.html)).

If these volume operations fail, pod creation may get stuck. The
operations will get retried and might eventually succeed, for example
because storage capacity gets freed up or extended. A pod with an
ephemeral volume will not get rescheduled to another node. A pod with
a volume that uses delayed binding should get scheduled multiple times,
but then might always land on the same node unless there are multiple
nodes with equal priority.

A new API for exposing storage capacity currently available via CSI
drivers and a scheduler enhancement that uses this information will
reduce the risk of that happening.

## Motivation

### Goals

* Define an API for exposing information about storage that is
  flexible enough for a variety of use cases and that can be extended
  later on.

* Expose capacity information at the semantic
  level that Kubernetes currently understands, i.e. in a way that
  Kubernetes can compare capacity against the requested size of
  volumes. This has to work for local storage, network-attached
  storage and for drivers where the capacity depends on parameters in
  the storage class.

* Increase the chance of choosing a node for which volume creation
  will succeed by tracking the currently available capacity available
  through a CSI driver and using that information during pod
  scheduling.

### Non-Goals

* Only CSI drivers will be supported.

* No attempts will be made to model how capacity will be affected by
  pending volume operations. This would depend on internal driver
  details that Kubernetes doesn’t have.

* Nodes are not prioritized based on how much storage they have available.
  This and a way to specify the policy for the prioritization might be
  added later on in a separate KEP.

* Because of that and also for other reasons (capacity changed via
  operations outside of Kubernetes, like creating or deleting volumes,
  or expanding the storage), it is expected that pod scheduling may
  still end up with a node from time to time where volume creation
  then fails. Rolling back in this case is complicated and outside of
  the scope of this KEP. For example, a pod might use two persistent
  volumes, of which one was created and the other not, and then it
  wouldn’t be obvious whether the existing volume can or should be
  deleted.

* For persistent volumes that get created independently of a pod
  nothing changes: it’s still the responsibility of the CSI driver to
  decide how to create the volume and then communicate back through
  topology information where pods using that volume need to run.
  However, a CSI driver may use the capacity information exposed
  through the proposed API to make its choice.

* Inline volumes could be extended to reference a storage class, which
  then could be used to handle more complex situations (like the LVM
  mirror vs. striped case) also for inline volumes. But this is outside
  the scope of this KEP.

## Proposal

### User Stories

#### Ephemeral PMEM volume for Redis

A [modified Redis server](https://github.com/pmem/redis) can use
[PMEM](https://pmem.io/) via
[memkind](http://memkind.github.io/memkind/) as DRAM replacement with
higher capacity and lower cost at almost the same performance. When it
starts, all old data is discarded, so an inline ephemeral volume is a
suitable abstraction for declaring the need for a volume that is
backed by PMEM and provided by
[PMEM-CSI](https://github.com/intel/pmem-csi). But PMEM is a resource
that is local to a node and thus the scheduler has to be aware whether
enough of it is available on a node before assigning a pod to it.

#### Different LVM configurations

A user may want to choose between higher performance of local disks
and higher fault tolerance by selecting striping respectively
mirroring or raid in the storage class parameters of a driver for LVM,
like for example [TopoLVM](https://github.com/cybozu-go/topolvm).

The maximum size of the resulting volume then depends on the storage
class and its parameters.

#### Network attached storage

In contrast to local storage, network attached storage can be made
available on more than just one node. However, for technical reasons
(high-speed network for data transfer inside a single data center) or
political reasons (data must only be stored and processed in a single
jurisdication) availability may still be limited to a subset of the
nodes in a cluster.

#### Custom schedulers

For situations not handled by the Kubernetes scheduler now and/or in
the future, a [scheduler
extender](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/scheduler_extender.md)
can influence pod scheduling based on the information exposed via the
new API. The
[topolvm-scheduler](https://github.com/cybozu-go/topolvm/blob/master/docs/design.md#how-the-scheduler-extension-works)
currently does that with a driver-specific way of storing capacity
information.

#### Operators for applications

Application operators for modern scale out storage services
(e.g. MongoDB, ElasticSearch, Kafka, MySQL, PostgreSQL, Minio, etc.)
may want to control creation of volumes carefully in order to optimize
availability, durability, performance and cost. For more information
about this, see the [StoragePool API for Advanced Storage Placement
KEP](https://github.com/kubernetes/enhancements/pull/1347).

Storage pools as introduced in this KEP enable the creation of such
operators by providing them the necessary information about storage in
the cluster.

### Caching remaining capacity via the API server

CSI defines the `GetCapacity` RPC call for the controller service to
report information about the current capacity but it's not accessible
in the Kubernetes scheduler. A new type `CSIStoragePool` type gets
added to store the result of the `GetCapacity` calls for one storage
pool in the API server and thus make it available to the scheduler. In
the future it might get extended to also store other information like
health of the pool.

### Identifying storage pools

A method for identifying storage pools accessible via a CSI driver
based on the existing topology information gets outlined below. The
`external-provisioner` will be extended to handle the management of
`CSIStoragePool` objects.

For local storage in the node, there will be one such `CSIStoragePool`
object per node and CSI driver on that node. For network attached
storage, there will be one `CSIStoragePool` per CSI
[Topology](https://github.com/container-storage-interface/spec/blob/4731db0e0bc53238b93850f43ab05d9355df0fd9/lib/go/csi/csi.pb.go#L1662-L1691)
and driver.

### Size of ephemeral inline volumes

Currently Kubernetes has no information about the size of an ephemeral
inline volume. A new `CSIVolumeSource.fsSize` field needs to be added
to expose that in a vendor-agnostic way. Details for that are in
https://github.com/kubernetes/enhancements/pull/1353.

### Pod scheduling

The Kubernetes scheduler monitors `CSIStoragePool` objects and
excludes nodes with insufficient remaining capacity when it comes to
making scheduling decisions for a pod which uses ephemeral inline
volumes or persistent volumes with delayed binding. If the driver does
not indicate that it supports capacity reporting, then the scheduler
proceeds just as it does now, so nothing changes for existing CSI
driver deployments.

### API

All Kubernetes API changes are under the `CSIStorageCapacity` feature
gate.

#### CSIStoragePool

The prefix “CSI” is used to avoid confusion with other types that have
a similar name and because some aspects are specific to CSI.

```
// CSIStoragePool identifies one particular storage pool and
// stores the corresponding attributes. The spec is read-only.
type CSIStoragePool struct {
	metav1.TypeMeta
	// Standard object's metadata. The name has no particular meaning and just has to
	// meet the usual requirements (length, characters, unique). To ensure that
	// there are no conflicts with other CSI drivers on the cluster, the recommendation
	// is to use sp-<uuid>.
	//
	// Objects are not namespaced.
	//
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
	// +optional
	metav1.ObjectMeta

	Spec   CSIStoragePoolSpec
	Status CSIStoragePoolStatus
}

// CSIStoragePoolSpec contains the constant attributes of a CSIStoragePool.
type CSIStoragePoolSpec struct {
	// The CSI driver that provides access to the storage pool.
	// This must be the string returned by the CSI GetPluginName() call.
	DriverName string
}

// CSIStoragePoolStatus contains runtime information about a CSIStoragePool.
//
// A pool might only be accessible from a subset of the nodes in the
// cluster. That subset can be identified either via NodeTopology or
// Nodes, but not both. If neither is set, the pool is assumed
// to be available in the entire cluster.
//
// It is expected to be extended with other
// attributes which do not depend on the storage class, like health of
// the pool. Therefore it has the list of
// `CSIStorageByClass` instances instead of just the capacity
// and the storage class being in the spec.
type CSIStoragePoolStatus struct {
	// NodeTopology can be used to describe a storage pool that is available
	// only for nodes matching certain criteria.
	// +optional
	NodeTopology *v1.NodeSelector

	// Nodes can be used to describe a storage pool that is available
	// only for certain nodes in the cluster.
	//
	// +listType=set
	// +optional
	Nodes []string

	// Some information, like the actual usable capacity, may
	// depend on the storage class used for volumes.
	//
	// +patchMergeKey=storageClassName
	// +patchStrategy=merge
	// +listType=map
	// +listMapKey=storageClassName
	// +optional
	Classes []CSIStorageByClass `patchStrategy:"merge" patchMergeKey:"storageClassName" json:"classes,omitempty" protobuf:"bytes,4,opt,name=classes"`
}

// CSIStorageByClass contains information that applies to one storage
// pool of a CSI driver when using a certain storage class.
//
// Both AvailableCapacity and MaximumVolumeSize are optional. If both
// are provided, then the Kubernetes scheduler compares the size of
// a volume against MaximumVolumeSize to determine whether the
// volume has a chance of being created. If only AvailableCapacity is
// set, then the scheduler will use that, which may be a good enough
// approximation. If neither is set, then the scheduler will ignore
// the pool.
type CSIStorageByClass struct {
	// The storage class name matches the name of some actual
	// `StorageClass`, in which case the information applies when
	// using that storage class for a volume. There are also two
	// special names:
	// - <ephemeral> for storage used by ephemeral inline volumes (which
	//   don't use a storage class)
	// - <fallback> for storage that is the same regardless of the storage class;
	//   it is applicable if there is no other, more specific entry
	StorageClassName string `json:"storageClassName" protobuf:"bytes,1,name=storageClassName"`

	// AvailableCapacity is the sum of all storage that may be provided by the
	// CSI driver through the given storage class. There is no guarantee that
	// a single volume really can use all of that space. For example, fragmentation
	// might prevent creating such a volume.
	// +optional
	AvailableCapacity *resource.Quantity `json:"availableCapacity,omitempty" protobuf:"bytes,2,opt,name=availableCapacity"`

	// MaximumVolumeSize is the size of the largest volume that currently can
	// be created. This is a best-effort guess and even volumes
	// of that size might not get created successfully, either because
	// conditions changed between providing this information and attempting
	// to create a volume or because the guess was not accurate enough.
	// +optional
	MaximumVolumeSize *resource.Quantity `json:"maximumVolumeSize,omitempty" protobuf:"bytes,3,opt,name=maximumVolumeSize"`
}

const (
	// FallbackStorageClassName is used for a CSIStorage element which
	// applies when there isn't a more specific element for the
	// current storage class or ephemeral volume.
	FallbackStorageClassName = "<fallback>"

	// EphemeralStorageClassName is used for storage from which
	// ephemeral volumes are allocated.
	EphemeralStorageClassName = "<ephemeral>"
)
```

This approach has the advantage that the size of the `CSIStoragePool`
objects do not depend on the number of nodes in the cluster for local
storage.

The downsides are:
- Some attributes (driver name, topology) must be stored multiple times
  compared to a single, more complex object that contains all information
  for one CSI driver, so overall size in etcd is higher.
- Higher number of objects which all need to be retrieved by a client
  which does not already know which `CSIStoragePool` is is interested in.

##### Example: local storage

```
apiVersion: storage.k8s.io/v1alpha1
kind: CSIStoragePool
metadata:
  name: sp-ab96d356-0d31-11ea-ade1-8b7e883d1af1
spec:
  driverName: hostpath.csi.k8s.io
status:
  classes:
  - maximumVolumeSize: 256G
    storageClassName: <fallback>
  nodeTopology:
    nodeSelectorTerms:
    - matchExpressions:
      - key: kubernetes.io/hostname
        operator: In
        values:
        - node-1

apiVersion: storage.k8s.io/v1alpha1
kind: CSIStoragePool
metadata:
  name: sp-c3723f32-0d32-11ea-a14f-fbaf155dff50
spec:
  driverName: hostpath.csi.k8s.io
status:
  classes:
  - maximumVolumeSize: 512G
    storageClassName: <fallback>
  nodeTopology:
    nodeSelectorTerms:
    - matchExpressions:
      - key: kubernetes.io/hostname
        operator: In
        values:
        - node-2
```

##### Example: affect of storage classes

```
apiVersion: storage.k8s.io/v1alpha1
kind: CSIStoragePool
metadata:
  name: sp-9c17f6fc-6ada-488f-9d44-c5d63ecdf7a9
spec:
  driverName: lvm
status:
  classes:
  - maximumVolumeSize: 256G
    storageClassName: striped
  - maximumVolumeSize: 128G
    storageClassName: mirrored
  nodeTopology:
    nodeSelectorTerms:
    - matchExpressions:
      - key: kubernetes.io/hostname
        operator: In
        values:
        - node-1
```

##### Example: network attached storage

```
apiVersion: storage.k8s.io/v1alpha1
kind: CSIStoragePool
metadata:
  name: sp-b0963bb5-37cf-415d-9fb1-667499172320
spec:
  driverName: pd.csi.storage.gke.io
status:
  classes:
  - maximumVolumeSize: 128G
    storageClassName: <fallback>
  nodeTopology:
    nodeSelectorTerms:
    - matchExpressions:
      - key: failure-domain.beta.kubernetes.io/region
        operator: In
        values:
        - us-east-1

apiVersion: storage.k8s.io/v1alpha1
kind: CSIStoragePool
metadata:
  name: sp-64103396-0d32-11ea-945c-e3ede5f0f3ae
spec:
  driverName: pd.csi.storage.gke.io
status:
  classes:
  - maximumVolumeSize: 256G
    storageClassName: <fallback>
  nodeTopology:
    nodeSelectorTerms:
    - matchExpressions:
      - key: failure-domain.beta.kubernetes.io/region
        operator: In
        values:
        - us-west-1
```

#### CSIDriver.spec.storageCapacity

A new field `storageCapacity` of type `boolean` with default `false`
in
[CSIDriver.spec](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.16/#csidriverspec-v1beta1-storage-k8s-io)
indicates whether a driver deployment will create `CSIStoragePool`
objects with capacity information and wants the Kubernetes scheduler
to rely on that information when making scheduling decisions that
involve volumes that need to be created by the driver.

If not set, the scheduler makes such decisions without considering
whether the driver really can create the volumes (the current situation).


### Updating capacity information with external-provisioner

Most (if not all) CSI drivers already get deployed on Kubernetes
together with the external-provisioner which then handles volume
provisioning via PVC. However, drivers which only handle inline
ephemeral volumes currently don’t need that and thus get deployed
without it.

For the sake of avoiding yet another sidecar, the proposal is to
extend the external-provisioner such that it can also be deployed
alongside a CSI driver on each node and then just handles
`CSIStoragePool` creation and updating. This leads to different modes
of operations.

Because the external-provisioner is part of the deployment of the CSI
driver, that deployment can configure the behavior of the
external-provisioner via command line parameters. There is no need to
introduce heuristics or other, more complex ways of changing the
behavior (like extending the `CSIDriver` API).

#### Available capacity vs. maximum volume size

The CSI spec up to and including the current version 1.2 just
specifies that ["the available
capacity"](https://github.com/container-storage-interface/spec/blob/314ac542302938640c59b6fb501c635f27015326/lib/go/csi/csi.pb.go#L2548-L2554)
is to be returned by the driver. It is left open whether that means
that a volume of that size can be created.

This KEP introduces the more precise definition of a "maximum volume
size". Unless told otherwise, the external-provisioner assumes that
the `GetCapacityResponse` is the sum of all currently available
storage and then sets `CSIStorageByClass.AvailableCapacity` instead of
`CSIStorageByClass.MaximumVolumeSize`. A CSI driver deployment can use
`--capacity-is-maximum-volume-size=true` to change that, in which
case only `CSIStorageByClass.MaximumVolumeSize` will be set.

An extension of the CSI spec would be needed to enable a CSI driver to
report both values.

#### Only local volumes

This mode of operation is expected to be used by CSI drivers that need
to track capacity per node and only support ephemeral inline volumes
and/or persistent volumes with delayed binding (as proposed for the
csi-driver-host-path in
https://github.com/kubernetes-csi/external-provisioner/pull/367).

The CSI driver has to implement the CSI controller service and its
`GetCapacity` call. Its deployment has to add the external-provisioner
to the daemon set and enable the per-node capacity tracking with
`--enable-capacity=local-ephemeral`, `--enable-capacity=local-persistent`,
and/or `--enable-capacity=local-fallback`.

With `local-ephemeral` enabled, external-provisioner will call
`GetCapacity` with an empty `GetCapacityRequest` (no capabilities, no
parameters, no topology). With `local-persistent` enabled, it will
call `GetCapacity` once for each storage class and a
`GetCapacityRequest` that has no capabilities, parameters from the
storage class, and no topology.

`local-fallback` is the same as `local-ephemeral`, but creates a
`CSIStorageByClass` with `StorageClassName` set to
`FallbackStorageClassName`. This should be used instead of the other
two options when the capacity is independent of any parameter in a
storage class.

The result is then stored in a `CSIStoragePool` for the node and
driver.

#### Central provisioning

Normally, external-provisioner gets deployed together with a CSI
controller service in a stateful set. That allows drivers to
support persistent volumes without late binding.

When deployed with `--enable-capacity=topology-ephemeral`,
`--enable-capacity=topology-persistent` and/or
`--enable-capacity=topology-fallback`, external-provisioner
determines storage pools based on the topology information provided by
the driver on each node and gets capacity for the resulting pools from
the driver's controller service that it is connected to.

This relies on a feature of kubelet and one additional requirement for CSI
driver deployments:
- Every topology segment that a CSI node service reports in response
 to GetNodeInfo (for example, `example.com/zone`: `Z1` +
 `example.com/rack`: `R3`) is copied into node labels by kubelet.
- All keys must have the same prefix (`example.com` in that example)
  and external-provisioner is given that prefix with the
  `--topology-prefix=example.com` parameter.

With that parameter, external-provisioner can reconstruct topology segments as follows:
- For each node, find the labels with that prefix and combine them
  into a `csi.Topology` instance.
- Remove duplicates.

When `topology-ephemeral` is active, for each storage class that uses
the driver and for each of the topology segments it calls
`GetCapacity`, with parameters from the storage class and the topology
segment as `GetCapacityRequest.accessible_topology`.

When `topology-ephemeral` is active, it also calls
`GetCapacity` for the same segment without parameters. In other words, capacity
information for ephemeral inline volumes is gathered through the CSI
controller service and is assumed to be specific to the same topology
segments as normal volumes.

As before, `topology-fallback` covers the case where storage class
parameters do not affect capacity.

The result is then stored in a different `CSIStoragePool` object for
each identified pool, with a `NodeTopology` that selects the right
nodes via their labels.

Drivers that don't support topology can use
`--enable-capacity=global-ephemeral`,
`--enable-capacity=global-persistent` and/or
`--enable-capacity=global-fallback`. Those are similar to their
`topology-*` counterparts but without identifying pools and thus
without `GetCapacityRequest.accessible_topology` The resulting
`CSIStoragePool` objects then have neither `Nodes` nor `NodeSelector` set and
thus apply to all nodes in the cluster.

#### CSIStoragePool lifecycle

external-provisioner needs permission to create, update and delete
`CSIStoragePool` objects. Before creating a new object, it must check
whether one already exists with the relevant attributes (driver name +
nodes) and then update that one instead.

To ensure that `CSIStoragePool` objects get removed when the driver
deployment gets removed before it gets a chance to clean up, each
`CSIStoragePool` object needs an [owner
reference](https://godoc.org/k8s.io/apimachinery/pkg/apis/meta/v1#OwnerReference)
that points to the pod which runs the external-provisioner that
created it. Then those objects will get garbage collected.

It is an intentional side effect that this happens also when the pod
merely gets restarted: the external-provisioner instance can reuse
objects that still exist and update their owner, but it doesn't need
take care of removing old objects that it no longer needs. That is
important when changing to a deployment that is configured differently
because the new deployment does not need to scan for old objects.

While external-provisioner runs, it needs to update and potentially
delete `CSIStoragePool` objects:
- when nodes change (for central provisioning)
- when storage classes change (for persistent volumes)
- when volumes were created or deleted (for central provisioning)
- when volumes are resized or snapshots are created or deleted (for persistent volumes)
- periodically, to detect changes in the underlying backing store (all cases)

Because sidecars are currently separated, external-snapshotter is
unaware of resizing and snapshotting. It also not involved with
ephemeral inline volumes. The periodic polling will catch up
with changes caused by those operations.

### Using capacity information

The Kubernetes scheduler already has a component, the [volume
scheduling
library](https://github.com/kubernetes/kubernetes/tree/master/pkg/controller/volume/scheduling),
which implements [topology-aware
scheduling](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/storage/volume-topology-scheduling.md).

The
[CheckVolumeBinding](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/storage/volume-topology-scheduling.md#integrating-volume-binding-with-pod-scheduling)
function gets extended to not only check for a compatible topology of
a node (as it does now), but also to verify whether the node falls
into a topology segment that has enough capacity left. This check is
only necessary for PVCs that have not been bound yet and for inline
volumes.

The lookup sequence will be:
- find the `CSIDriver` object for the driver
- check whether it has `CSIDriver.spec.storageCapacity` enabled
- find all `CSIStoragePool` objects that have the right spec
  (driver, accessible by node) and sufficient capacity for the
  volume attributes (storage class vs. ephemeral)

The specified volume size is compared against `MaximumVolumeSize` if
available, otherwise `AvailableCapacity`. A pool which has neither is
considered unusable at the moment and ignored.

## Design Details

### Test Plan

The Kubernetes scheduler extension will be tested with new unit tests
that simulate a variety of scenarios:
- different volume sizes and types
- driver with and without storage capacity tracking enabled
- capacity information for node local storage (node selector with one
  host name), network attached storage (more complex node selector),
  storage available in the entire cluster (no node restriction)
- no suitable node, one suitable node, several suitable nodes

Producing capacity information in external-provisioner also can be
tested with new unit tests. This has to cover:
- different modes
- different storage classes
- a driver response where storage classes matter and where they
  don't matter
- different topologies
- various older capacity information, including:
  - no entries
  - obsolete entries
  - entries that need to be updated
  - entries that can be left unchanged

This needs to run with mocked CSI driver and API server interfaces to
provide the input and capture the output.

Full end-to-end testing is needed to ensure that new RBAC rules are
identified and documented properly. For this, a new alpha deployment
in csi-driver-host-path is needed because we have to make changes to
the deployment like setting `CSIDriver.spec.storageCapacity` which
will only be valid when tested with Kubernetes 1.18 cluster where
alpha features are enabled.

The CSI hostpath driver needs to be changed such that it reports the
remaining capacity of the filesystem where it creates volumes. The
existing raw block volume tests then can be used to ensure that pod
scheduling works:
- Those volumes have a size set.
- Late binding is enabled for the CSI hostpath driver.

A new test can be written which checks for `CSIStoragePool` objects,
asks for pod scheduling with a volume that is too large, and then
checks for events that describe the problem.

## Implementation History

- Kubernetes 1.18: alpha (tentative)

## Drawbacks

The attempt to define and implement a generic solution may end up with
something that isn't capable enough in practice because it does not
know enough about specific limitations of individual CSI drivers.

At the moment, storage vendor can already achieve the same goals
entirely without changes in Kubernetes or Kubernetes-CSI, it's just a
lot of work (see the [TopoLVM
design](https://github.com/cybozu-go/topolvm/blob/master/docs/design.md#diagram))
- custom node and controller sidecars
- scheduler extender (probably slower than a builtin scheduler
  callback)


## Alternatives

### Single capacity value

Some earlier draft only had a single `Capacity` value, with a
definition that this is the "maximum volume size". This was replaced
by `AvailableCapacity` and `MaximumVolumeSize` to avoid potential
confusion and make the API more flexible.

### Node list

Instead of a full node selector expression, a simple list of node
names could make objects in some special cases, in particular
node-local storage, smaller and easier to read. This has been removed
from the KEP because a node selector can be used instead and therefore
the node list was considered redundant and unnecessary. The examples
in the next section use `nodes` in some cases to demonstrate the
difference.


### CSIDriver.Status

Alternatively, a `CSIDriver.Status` could combine all information in
one object in a way that is both human-readable (albeit potentially
large) and matches the lookup pattern of the scheduler. Updates could
be done efficiently via `PATCH` operations. Finding information about
all pools at once would be simpler.

However, continuously watching this single object and retrieving
information about just one pool would become more expensive. Because
the API needs to be flexible enough to also support this for future
use cases, this approach has been rejected.

```
type CSIDriver struct {
    ...

	// Specification of the CSI Driver.
	Spec CSIDriverSpec `json:"spec" protobuf:"bytes,2,opt,name=spec"`

	// Status of the CSI Driver.
	// +optional
	Status CSIDriverStatus `json:"status,omitempty" protobuf:"bytes,3,opt,name=status"`
}

type CSIDriverSpec struct {
    ...

	// CapacityTracking defines whether the driver deployment will provide
	// capacity information as part of the driver status.
	// +optional
	CapacityTracking *bool `json:"capacityTracking,omitempty" protobuf:"bytes,4,opt,name=capacityTracking"`
}

// CSIDriverStatus represents dynamic information about the driver and
// the storage provided by it, like for example current capacity.
type CSIDriverStatus struct {
	// Each driver can provide access to different storage pools
	// (= subsets of the overall storage with certain shared
	// attributes).
	//
	// +patchMergeKey=name
	// +patchStrategy=merge
	// +listType=map
	// +listMapKey=name
	// +optional
	Storage []CSIStoragePool `patchStrategy:"merge" patchMergeKey:"name" json:"storage,omitempty" protobuf:"bytes,1,opt,name=storage"`
}

// CSIStoragePool identifies one particular storage pool and
// stores the corresponding attributes.
//
// A pool might only be accessible from a subset of the nodes in the
// cluster. That subset can be identified either via NodeTopology or
// Nodes, but not both. If neither is set, the pool is assumed
// to be available in the entire cluster.
type CSIStoragePool struct {
	// The name is some user-friendly identifier for this entry.
	Name string `json:"name" protobuf:"bytes,1,name=name"`

	// NodeTopology can be used to describe a storage pool that is available
	// only for nodes matching certain criteria.
	// +optional
	NodeTopology *v1.NodeSelector `json:"nodeTopology,omitempty" protobuf:"bytes,2,opt,name=nodeTopology"`

	// Nodes can be used to describe a storage pool that is available
	// only for certain nodes in the cluster.
	//
	// +listType=set
	// +optional
	Nodes []string `json:"nodes,omitempty" protobuf:"bytes,3,opt,name=nodes"`

	// Some information, like the actual usable capacity, may
	// depend on the storage class used for volumes.
	//
	// +patchMergeKey=storageClassName
	// +patchStrategy=merge
	// +listType=map
	// +listMapKey=storageClassName
	// +optional
	Classes []CSIStorageByClass `patchStrategy:"merge" patchMergeKey:"storageClassName" json:"classes,omitempty" protobuf:"bytes,4,opt,name=classes"`
}

// CSIStorageByClass contains information that applies to one storage
// pool of a CSI driver when using a certain storage class.
type CSIStorageByClass struct {
	// The storage class name matches the name of some actual
	// `StorageClass`, in which case the information applies when
	// using that storage class for a volume. There are also two
	// special names:
	// - <ephemeral> for storage used by ephemeral inline volumes (which
	//   don't use a storage class)
	// - <fallback> for storage that is the same regardless of the storage class;
	//   it is applicable if there is no other, more specific entry
	StorageClassName string `json:"storageClassName" protobuf:"bytes,1,name=storageClassName"`

	// Capacity is the size of the largest volume that currently can
	// be created. This is a best-effort guess and even volumes
	// of that size might not get created successfully.
	// +optional
	Capacity *resource.Quantity `json:"capacity,omitempty" protobuf:"bytes,2,opt,name=capacity"`
}

const (
	// FallbackStorageClassName is used for a CSIStorage element which
	// applies when there isn't a more specific element for the
	// current storage class or ephemeral volume.
	FallbackStorageClassName = "<fallback>"

	// EphemeralStorageClassName is used for storage from which
	// ephemeral volumes are allocated.
	EphemeralStorageClassName = "<ephemeral>"
)
```

#### Example: local storage

In this example, one node has the hostpath example driver
installed. Storage class parameters do not affect the usable capacity,
so there is only one `CSIStorageByClass`:

```
apiVersion: storage.k8s.io/v1beta1
kind: CSIDriver
metadata:
  creationTimestamp: "2019-11-13T15:36:00Z"
  name: hostpath.csi.k8s.io
  resourceVersion: "583"
  selfLink: /apis/storage.k8s.io/v1beta1/csidrivers/hostpath.csi.k8s.io
  uid: 6040df83-a938-4b1a-aea6-92360b0a3edc
spec:
  attachRequired: true
  podInfoOnMount: true
  volumeLifecycleModes:
  - Persistent
  - Ephemeral
status:
  storage:
  - classes:
    - capacity: 256G
      storageClassName: <fallback>
    name: node-1
    nodes:
    - node-1
  - classes:
    - capacity: 512G
      storageClassName: <fallback>
    name: node-2
    nodes:
    - node-2
```

#### Example: affect of storage classes

This fictional LVM CSI driver can either use 256GB of local disk space
for striped or mirror volumes. Mirrored volumes need twice the amount
of local disk space, so the maximum volume size is halved:

```
apiVersion: storage.k8s.io/v1beta1
kind: CSIDriver
metadata:
  creationTimestamp: "2019-11-13T15:36:01Z"
  name: lvm
  resourceVersion: "585"
  selfLink: /apis/storage.k8s.io/v1beta1/csidrivers/lvm
  uid: 9c17f6fc-6ada-488f-9d44-c5d63ecdf7a9
spec:
  attachRequired: true
  podInfoOnMount: false
  volumeLifecycleModes:
  - Persistent
status:
  storage:
  - classes:
    - capacity: 256G
      storageClassName: striped
    - capacity: 128G
      storageClassName: mirrored
    name: node-1
    nodes:
    - node-1
```

#### Example: network attached storage

The algorithm outlined in [Central
provisioning](#central-provisioning) will result in `CSIStoragePool`
entries using `NodeTopology`, similar to this hand-crafted example:

```
apiVersion: storage.k8s.io/v1beta1
kind: CSIDriver
metadata:
  creationTimestamp: "2019-11-13T15:36:01Z"
  name: pd.csi.storage.gke.io
  resourceVersion: "584"
  selfLink: /apis/storage.k8s.io/v1beta1/csidrivers/pd.csi.storage.gke.io
  uid: b0963bb5-37cf-415d-9fb1-667499172320
spec:
  attachRequired: true
  podInfoOnMount: false
  volumeLifecycleModes:
  - Persistent
status:
  storage:
  - classes:
    - capacity: 128G
      storageClassName: <fallback>
    name: region-east
    nodeTopology:
      nodeSelectorTerms:
      - matchExpressions:
        - key: failure-domain.beta.kubernetes.io/region
          operator: In
          values:
          - us-east-1
  - classes:
    - capacity: 256G
      storageClassName: <fallback>
    name: region-west
    nodeTopology:
      nodeSelectorTerms:
      - matchExpressions:
        - key: failure-domain.beta.kubernetes.io/region
          operator: In
          values:
          - us-west-1
```

### Prior work

The [Topology-aware storage dynamic
provisioning](https://docs.google.com/document/d/1WtX2lRJjZ03RBdzQIZY3IOvmoYiF5JxDX35-SsCIAfg)
design document used a different data structure and had not fully
explored how that data structure would be populated and used.
