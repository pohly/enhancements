---
title: Storage Capacity Constraints for Pod Scheduling
authors:
  - "@pohly"
  - "@cofyc"
owning-sig: sig-storage
participating-sigs:
  - sig-scheduling
reviewers:
  - TBD
approvers:
  - TBD
editor: "@pohly"
creation-date: 2019-09-19
last-updated: 2019-10-31
status: provisional
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
  - [Caching remaining capacity via the API server](#caching-remaining-capacity-via-the-api-server)
  - [API](#api)
    - [Storage capacity](#storage-capacity)
    - [Size of ephemeral inline volumes](#size-of-ephemeral-inline-volumes)
  - [Updating capacity information with external-provisioner](#updating-capacity-information-with-external-provisioner)
    - [Only inline ephemeral volumes](#only-inline-ephemeral-volumes)
    - [Central provisioning](#central-provisioning)
  - [Using capacity information](#using-capacity-information)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
  - [Separate objects](#separate-objects)
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

If these volume operations fail, pod creation gets stuck. The
operations will get retried and might eventually succeed, for example
because storage capacity gets freed up or extended. What does not
happen is that the pod is re-scheduled to some other node which has
enough storage capacity.

## Motivation

### Goals

The goal of this KEP is to increase the chance of choosing a node for
which volume creation will succeed by tracking the currently available
capacity available through a CSI driver and using that information
during pod scheduling.

Although the issue is more common with storage that is local to a node
(LVM, PMEM), it may also occur for storage providers that have
capacity limits for other topology segments (rack, data center,
region, etc.). Capacity tracking is meant to be generic enough to
support all of these cases.

Inline volumes currently do not have a standardized way of specify the
size; this KEP will introduce such a field and ensure that capacity
tracking also works for drivers which only support ephemeral inline
volumes.

For persistent volumes, capacity will be tracked per storage class, so
additional parameters in a storage class are taken into account. This
is important because those parameters might have a significant impact on
how much space a new volume actually needs in the underlying storage
system (for example, an LVM volume with mirroring needs more space
than a LVM volume that is striped). For ephemeral volumes there is no
storage class, so tracking is only done per node and provisioner.

### Non-Goals

Only CSI drivers will be supported.

The Kubernetes scheduler could try to anticipate the effect of
creating multiple volumes concurrently. But this depends on knowledge
about internal driver details that Kubernetes doesn’t have, so pending
volume operations are simply ignored when making scheduling decisions.

Because of that and also for other reasons (capacity changed via
operations outside of Kubernetes, like creating or deleting volumes,
or expanding the storage), it is expected that pod scheduling may
still end up with a node from time to time where volume creation then
fails. Rolling back in this case is complicated and outside of the
scope of this KEP. For example, a pod might use two persistent
volumes, of which one was created and the other not, and then it
wouldn’t be obvious whether the existing volume can or should be
deleted.

For persistent volumes that get created independently of a pod nothing
changes: it’s still the responsibility of the CSI driver to decide how
to create the volume and then communicate back through topology
information where pods using that volume need to run.

Inline volumes could be extended to reference a storage class, which
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


### Caching remaining capacity via the API server

CSI defines the GetCapacity RPC call for the controller service to
report information about the current capacity but it's not accessible
in the Kubernetes scheduler. The existing `CSIDriver` type gets
extended with a `Status` field that (initially) only holds the
capacity information in a format that is sufficiently flexible to
handle the use cases outlined above. In the future, that `Status`
field could also be extended to represent other dynamic information
about the driver.

The CSI driver deployment is already responsible for creating a
`CSIDriver` object for itself. Now it will also have to enable and
configure the adding, updating and removing of capacity
information. To simplify that, the `external-provisioner` will be
extended to handle this based on the already existing APIs (i.e. no
changes needed in a CSI driver itself).

The Kubernetes scheduler monitors `CSIDriver` objects and uses that
information when it comes to making scheduling decisions. If the
driver does not indicate that it supports capacity reporting, then the
scheduler proceeds just as it does now, so nothing changes for
existing CSI driver deployments.

### API

All Kubernetes API changes are under the `CSIStorageCapacity` feature
gate.

#### Storage capacity

The prefix “CSI” is used to avoid confusion with other types that have
a similar name and because some aspects are specific to CSI.

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
	// Each driver can provide different kinds of storage.
	// +patchMergeKey=storageClassName
	// +patchStrategy=merge
	// +listType=map
	// +listMapKey=storageClassName
	// +optional
	Storage []CSIStorage `patchStrategy:"merge" patchMergeKey:"storageClassName" json:"storage,omitempty" protobuf:"bytes,1,opt,name=storage"`
}

// CSIStorage contains information for one particular kind of storage
// provided by a CSI driver.
type CSIStorage struct {
	// The storage class name matches the name of some actual
	// `StorageClass`, in which case the information applies when
	// using that storage class for a volume. There are also two
	// special names:
	// - <ephemeral> for storage used by ephemeral inline volumes (which
	//   don't use a storage class)
	// - <fallback> for storage that is the same regardless of the storage class;
	//   it is applicable if there is no other, more specific entry
	StorageClassName string `json:"storageClassName" protobuf:"bytes,1,name=storageClassName"`

	// A CSI driver may allocate storage from one or more pools
	// with different attributes. The entries must have names that
	// are unique inside this list.
	//
	// +patchMergeKey=name
	// +patchStrategy=merge
	// +listType=map
	// +listMapKey=name
	// +optional
	Pools []CSIStoragePool `patchStrategy:"merge" patchMergeKey:"name" json:"pools,omitempty" protobuf:"bytes,2,opt,name=pools"`
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

// CSIStoragePoolInfo identifies one particular storage pool and
// stores the corresponding attributes.
//
// A pool might only be accessible from a subset of the nodes in the
// cluster. That subset can be identified either via NodeTopology or
// NodeList, but not both. If neither is set, the pool is assumed
// to be available in the entire cluster.
type CSIStoragePool struct {
	// The name is some user-friendly identifier for this entry.
	Name string `json:"name" protobuf:"bytes,1,name=name"`

	// NodeTopology can be used to describe a storage pool that is available
	// only for nodes matching certain criteria.
	// +optional
	NodeTopology *v1.NodeSelector `json:"nodeTopology,omitempty" protobuf:"bytes,2,opt,name=nodeTopology"`

	// NodeList can be used to describe a storage pool that is available
	// only for certain nodes in the cluster.
    //
    // +listType=set
	// +optional
	NodeList []string `json:"nodeList,omitempty" protobuf:"bytes,3,opt,name=nodeList"`

	// Capacity is the size of the largest volume that currently can
	// be created. This is a best-effort guess and even volumes
	// of that size might not get created successfully.
	// +optional
	Capacity *resource.Quantity `json:"capacity,omitempty" protobuf:"bytes,4,opt,name=capacity"`

	// ExpiryTime is the absolute time at which this entry becomes obsolete.
	// When not set, the entry is valid forever.
	ExpiryTime *metav1.Time `json:"expiryTime,omitempty" protobuf:"bytes,5,opt,name=expiryTime"`
}
```

This API is designed so that updating a `CSIStoragePool` can be done
with a `PATCH` operation. This way, individual writers can send
updates without having to get the latest full `CSIDriver` object and
there is no risk of conflicts due to concurrent writes.

Removing obsolete information when uninstalling a driver is simple
because it is part of the usual `CSIDriver` removal.

How the availability of the `CSIStoragePool` within the cluster is
defined is up to the CSI driver. This may get extended in the future
to also cover other use cases, like reporting the capacity of
individual disks inside a node.

#### Size of ephemeral inline volumes

A new field `fsSize` of type `*Quantity` in
[CSIVolumeSource](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.16/#csivolumesource-v1-core)
needs to be added, alongside the existing `fsType`. It must be a
pointer to distinguish between no size and zero size selected.

While `fsType` can (and does get) mapped to
`NodePublishVolumeRequest.volume_capability`, for `fsSize` we need a
different approach for passing it to the CSI driver because there is
no pre-defined field for it in `NodePublishVolumeRequest`. We can
extend the [pod info on
mount](https://kubernetes-csi.github.io/docs/pod-info.html) feature:
if (and only if) the driver enables that, then a new
`csi.storage.k8s.io/size` entry in
`NodePublishVolumeRequest.publish_context` is set to the string
representation of the size quantity. An unset size is passed as empty
string.

This has to be optional because CSI drivers written for 1.16 might do
strict validation of the `publish_context` content and reject volumes
with unknown fields. If the driver enables pod info, then new fields
in the `csi.storage.k8s.io` namespace are explicitly allowed.

Using that new `fsSize` field must be optional. If a CSI driver
already accepts a size specification via some driver-specific
parameter, then specifying the size that way must continue to
work. But if the `fsSize` field is set, a CSI driver should use that
and treat it as an error when both `fsSize` and the driver-specific
parameter are set.

Setting `fsSize` for a CSI driver which ignores the field is not an
error. This is similar to setting `fsType` which also may get ignored
silently.

### Updating capacity information with external-provisioner

Most (if not all) CSI drivers already get deployed on Kubernetes
together with the external-provisioner which then handles volume
provisioning via PVC. However, drivers which only handle inline
ephemeral volumes currently don’t need that and thus get deployed
without it. For the sake of avoiding yet another sidecar, the proposal
is to extend the external-provisioner such that it can be deployed
alongside a CSI node service on each node and then just handles
`CSIStorageInfo` creation and updating. This leads to different mode
of operations:

#### Only inline ephemeral volumes

In this mode, multiple different external-provisioner instances run in
the cluster and collaboratively need to update
`CSIDriver.Status`. There is a single `CSIStorage` with one pool per
node.  Each external-provisioner is responsible for one
`CSIStoragePool` entry with a `NodeList` that picks the node on which
the driver and the external-provisioner run.

A CSI driver using this mode has to implement the CSI controller
service and its `GetCapacity` call, which will be called with empty
`GetCapacityRequest` (no capabilities, no parameters, no topology).

This mode of operation is expected to be used by drivers that really
need to track capacity per node; in that case, `CSIStorage` has
to grow linearly with the number of nodes where the driver runs.

#### Central provisioning

A driver that supports PVCs has to deploy external-provisioner
together with a CSI controller service. That includes drivers that
support both persistent volumes and ephemeral inline volumes.

In this mode, external-provisioners relies on a feature of kubelet and
one additional requirement for CSI driver deployments:
- every topology segment that a CSI node service reports in response
 to GetNodeInfo (for example, `example.com/zone`: `Z1` +
 `example.com/rack`: `R3`) is copied into node labels by kubelet
- all keys must have the same prefix (`example.com` in the example
  above) and external-provisioner is given that prefix as a parameter

Without that parameter, external-provisioner does not create capacity
information.

With that parameter, it can reconstruct topology segments as follows:
- For each node, find the labels with that prefix and combine them
  into a `csi.Topology` instance.
- Remove duplicates.

Then for each storage class that uses the driver and for each of the
topology segments it calls `GetCapacity`, with parameters from the
storage class and the topology segment as
`GetCapacityRequest.accessible_topology`.

Optionally, enabled by another command line parameter, it also does
the same without parameters and stores those results in a `CSIStorage`
with `StorageClassName` = `<ephemeral>`. In other words, capacity
information for ephemeral inline volumes is gathered through the CSI
controller service and is assumed to be specific to the same topology
segments as normal volumes.

`CSIDriver.Status` then must be updated:
- when nodes change
- when volumes were created or deleted
- periodically, to detect changes in the underlying backing store; a
  CSI spec extension would be necessary to avoid this polling

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
- find a `CSIStorage` object based on the volume attributes
  (storage class vs. ephemeral)
- find all pools that are accessible and have sufficient capacity left

### Risks and Mitigations

TBD

## Design Details

### Test Plan

TBD

### Upgrade / Downgrade Strategy

TBD

### Version Skew Strategy

TBD

## Implementation History

## Drawbacks

Why should this KEP _not_ be implemented - TBD.

## Alternatives

### Separate objects

The proposed `CSIDriver.Status` combines all information in one object
in a way that is both human-readable and matches the lookup pattern of
the controller.

Alternatively, `CSIStoragePool` could also be its own type, with
multiple independent objects. But then each object would have to have
a copy of the driver and storage class name and both the number of
object as well as the total amount of data in etcd increases.

Naming those objects will become harder and most likely wouldn't be
useful to a human (UUID or hash of its attributes). Removing those
objects becomes harder.

### Prior work

The [Topology-aware storage dynamic
provisioning](https://docs.google.com/document/d/1WtX2lRJjZ03RBdzQIZY3IOvmoYiF5JxDX35-SsCIAfg)
design document used a different data structure and had not fully
explored how that data structure would be populated and used.
