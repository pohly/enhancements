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
in the Kubernetes scheduler. A new API type which represents capacity
is introduced to solve this (details below).

The CSI driver deployment is responsible for creating and updating
objects of that type. The external-provisioner gets extended to handle
this, so CSI drivers only need to implement the GetCapacity call. Each
capacity value has a lifetime chosen by the creator. Removal of
expired values is left to the Kubernetes scheduler. This avoids the
issue that two different processes both try the same operation, which
would be redundant work. Making the Kubernetes scheduler responsible
for this has the advantage that it also works when the driver crashes
or gets de-installed.

The Kubernetes scheduler monitors those objects and uses that
information when it comes to making scheduling decisions. If the
scheduler has no current capacity information, it proceeds without it,
so nothing changes for CSI driver deployments that do not support
capacity tracking.

This is a problem for deployments that do support it, but haven’t been
able to provide the necessary information yet. TODO: extend
CSIDriverInfo with a field that tells the scheduler to retry later
when information is missing? Avoid the situation by (Somehow? How?!)
always providing the information before registering the driver -
probably doesn’t work for PVCs with late binding because the pod gets
scheduled even if the driver isn’t installed (which also is a problem
for the CSIDriverInfo approach…).

### API

All Kubernetes API changes are under the `CSIStorageCapacity` feature
gate.

#### Storage capacity

The prefix “CSI” is used to avoid confusion with other types that have
a similar name and because some aspects are specific to CSI.

```
// CSIStorageInfo represents information about storage provided by a certain CSI driver,
// like for example current capacity. As in CSIDriverInfo, the name of the CSIStorageInfo
// objects is the same as the driver name. Unlike CSIDriverInfo, these objects are
// created dynamically and get updated regularly when there are changes.
type CSIStorageInfo struct {
    metav1.TypeMeta
    metav1.ObjectMeta

   // The actual information may depend on the storage class and therefore
   // is provided separately for each storage class that uses the driver.
   // The empty class name is valid and represents information relevant for
   // ephemeral inline volumes as those don't use storage classes.
   Info: map[string]CSIStorageClassInfo
}

// CSIStorageClassInfo contains information for one particular storage class
// of a CSI driver.
type CSIStorageClassInfo struct {
    // A CSI driver may allocate storage from one or more pools with different
    // attributes.
    Pools []CSIStoragePoolInfo
}

// CSIStoragePoolInfo identifies one particular storage pool
// and stores the corresponding attributes.
type CSIStoragePoolInfo struct {
    // NodeTopology can be used to describe a storage pool that is available
    // only for certain nodes in the cluster. If not set, the pool is consider
    // to be available from all nodes.
    // +optional
    NodeTopology *v1.NodeSelector

    // Capacity is the size of the largest volume that currently can
    // be created. This is a best-effort guess and even volumes
    // of that size might not get created successfully.
    // +optional
    Capacity *resource.Quantity

    // ExpiryTime is the absolute time at which this entry becomes obsolete.
    // When not set, the entry is valid forever.
    // +optional
    ExpiryTime: *metav1.Time
}
```

How the NodeSelector is composed is not defined by the API and may
change over time. How the external-provisioner handles this is
explained in the next section.

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
`CSIStorageInfo`. CSICapacity just has one entry with the empty
storage class name. Each external-provisioner is responsible for one
`CSIStorageClassInfo` entry with a node selector that matches the node
name.

A CSI driver using this mode has to implement the CSI controller
service and the GetCapacity call, which will be called with empty
GetCapacityRequest (no capabilities, no parameters, no topology).

This mode of operation is expected to be used by drivers that really
need to track capacity per node; in that case, `CSIStorageClassInfo` has
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
For each node, find the labels with that prefix and combine them into
a Topology instance.  Remove duplicates.

Then for each storage class that uses the driver and for each of the
topology segments it calls GetCapacity, with parameters from the
storage class and the topology segment as
`GetCapacityRequest.accessible_topology`.

Optionally, enabled by another command line parameter, it also does
the same without parameters and stores those results in CSICapacity
with the empty storage class name. In other words, capacity
information for ephemeral inline volumes is gathered through the CSI
controller service and is assumed to be specific to the same topology
segments as normal volumes.

`CSIStorageInfo` then must be updated:
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

It can be done initially with a brute-force evaluation of each
`NodeSelector` in the `CSIStorageClassInfo` for the driver and storage
class of the PVC or later by maintaining an in-memory map from node
name to `CSIStoragePoolInfo`. The same code that maintains that map can
also deal with removing expired entries.

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

The [Topology-aware storage dynamic
provisioning](https://docs.google.com/document/d/1WtX2lRJjZ03RBdzQIZY3IOvmoYiF5JxDX35-SsCIAfg)
design document used a different data structure and had not fully
explored how that data structure would be populated and used.

