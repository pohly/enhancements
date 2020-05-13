# Storage Capacity Constraints for Pod Scheduling

## Table of Contents

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
  - [User Stories](#user-stories)
    - [Ephemeral PMEM volume for Redis or memcached](#ephemeral-pmem-volume-for-redis-or-memcached)
    - [Different LVM configurations](#different-lvm-configurations)
    - [Network attached storage](#network-attached-storage)
    - [Custom schedulers](#custom-schedulers)
- [Proposal](#proposal)
  - [Caching remaining capacity via the API server](#caching-remaining-capacity-via-the-api-server)
  - [Gathering capacity information](#gathering-capacity-information)
  - [Pod scheduling](#pod-scheduling)
- [Design Details](#design-details)
  - [API](#api)
    - [CSIStorageCapacity](#csistoragecapacity)
      - [Example: local storage](#example-local-storage)
      - [Example: affect of storage classes](#example-affect-of-storage-classes)
      - [Example: network attached storage](#example-network-attached-storage)
    - [CSIDriver.spec.storageCapacity](#csidriverspecstoragecapacity)
  - [Updating capacity information with external-provisioner](#updating-capacity-information-with-external-provisioner)
    - [Available capacity vs. maximum volume size](#available-capacity-vs-maximum-volume-size)
    - [Without central controller](#without-central-controller)
    - [With central controller](#with-central-controller)
    - [With central controller, generic solution](#with-central-controller-generic-solution)
    - [Determining parameters](#determining-parameters)
    - [CSIStorageCapacity lifecycle](#csistoragecapacity-lifecycle)
  - [Using capacity information](#using-capacity-information)
  - [Test Plan](#test-plan)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha -&gt; Beta Graduation](#alpha---beta-graduation)
    - [Beta -&gt; GA Graduation](#beta---ga-graduation)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature enablement and rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
  - [CSI drivers without topology support](#csi-drivers-without-topology-support)
  - [Storage class parameters that never affect capacity](#storage-class-parameters-that-never-affect-capacity)
  - [Multiple capacity values](#multiple-capacity-values)
  - [Node list](#node-list)
  - [CSIDriver.Status](#csidriverstatus)
    - [Example: local storage](#example-local-storage-1)
    - [Example: affect of storage classes](#example-affect-of-storage-classes-1)
    - [Example: network attached storage](#example-network-attached-storage-1)
  - [CSIStoragePool](#csistoragepool)
      - [Example: local storage](#example-local-storage-2)
      - [Example: affect of storage classes](#example-affect-of-storage-classes-2)
      - [Example: network attached storage](#example-network-attached-storage-2)
  - [Prior work](#prior-work)
<!-- /toc -->

## Release Signoff Checklist

<!--
**ACTION REQUIRED:** In order to merge code into a release, there must be an
issue in [kubernetes/enhancements] referencing this KEP and targeting a release
milestone **before the [Enhancement Freeze](https://git.k8s.io/sig-release/releases)
of the targeted release**.

For enhancements that make changes to code or processes/procedures in core
Kubernetes i.e., [kubernetes/kubernetes], we require the following Release
Signoff checklist to be completed.

Check these off as they are completed for the Release Team to track. These
checklist items _must_ be updated for the enhancement to be released.
-->

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
- [ ] (R) Graduation criteria is in place
- [ ] (R) Production readiness review completed
- [ ] Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

<!--
**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.
-->

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

There are two types of volumes that are getting created after making a scheduling
decision for a pod:
- [ephemeral inline
  volumes](https://kubernetes-csi.github.io/docs/ephemeral-local-volumes.html) -
  a pod has been permanently scheduled onto a node
- persistent volumes with [delayed
  binding](https://kubernetes.io/docs/concepts/storage/storage-classes/#volume-binding-mode)
  (`WaitForFirstConsumer`) - a node has been selected tentatively

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

* Define an API for exposing storage capacity information.

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

* Nodes are not yet prioritized based on how much storage they have available.
  This and a way to specify the policy for the prioritization might be
  added later on.

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

* Nothing changes for the current CSI ephemeral inline volumes. A [new
  approach for ephemeral inline
  volumes](https://github.com/kubernetes/enhancements/issues/1698)
  will support capacity tracking, based on this proposal here.

### User Stories

#### Ephemeral PMEM volume for Redis or memcached

A [modified Redis server](https://github.com/pmem/redis) and the upstream
version of [memcached](https://memcached.org/blog/persistent-memory/)
can use [PMEM](https://pmem.io/) as DRAM replacement with
higher capacity and lower cost at almost the same performance. When they
start, all old data is discarded, so an inline ephemeral volume is a
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
regulatory reasons (data must only be stored and processed in a single
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

Alternatively, the [scheduling
framework](https://kubernetes.io/docs/concepts/configuration/scheduling-framework/)
can be used to build and run a custom scheduler where the desired
policy is compiled into the scheduler binary.

## Proposal

### Caching remaining capacity via the API server

The Kubernetes scheduler cannot talk directly to the CSI drivers to
retrieve capacity information because CSI drivers typically only
expose a local Unix domain socket and are not necessarily running on
the same host as the scheduler(s).

The key approach in this proposal for solving this is to gather
capacity information, store it in the API server, and then use that
information in the scheduler. That information then flows
through different components:
1. storage backend
2. CSI driver
3. Kubernetes-CSI sidecar
4. API server
5. Kubernetes scheduler

The first two are driver specific. The sidecar will be provided by
Kubernetes-CSI, but how it is used is determined when deploying the
CSI driver. Steps 3 to 5 are explained below.

### Gathering capacity information

A sidecar, external-provisioner in this proposal, will be extended to
handle the management of the new objects. This follows the normal
approach that integration into Kubernetes is managed as part of the
CSI driver deployment, ideally without having to modify the CSI driver
itself. This will work without changes for some storage systems while
others may have to support [a new CSI API
call](#with-central-controller-generic-solution).

### Pod scheduling

The Kubernetes scheduler watches the capacity information and excludes
nodes with insufficient remaining capacity when it comes to making
scheduling decisions for a pod which uses persistent volumes with
delayed binding. If the driver does not indicate that it supports
capacity reporting, then the scheduler proceeds just as it does now,
so nothing changes for existing CSI driver deployments.

## Design Details

### API

#### CSIStorageCapacity

```
// CSIStorageCapacity stores the result of one CSI GetCapacity call for one
// driver, one topology segment, and the parameters of one storage class.
type CSIStorageCapacity struct {
	metav1.TypeMeta
	// Standard object's metadata. The name has no particular meaning and just has to
	// meet the usual requirements (length, characters, unique). To ensure that
	// there are no conflicts with other CSI drivers on the cluster, the recommendation
	// is to use csisc-<uuid>.
	//
	// Objects are not namespaced.
	//
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
	// +optional
	metav1.ObjectMeta

    // Spec contains the properties that describe one capacity value.
    Spec CSIStorageCapacitySpec
}

// CSIStorageCapacitySpec contains the properties that describe one capacity value.
type CSIStorageCapacitySpec struct {
	// The CSI driver that provides the storage.
	// This must be the string returned by the CSI GetPluginName() call.
	DriverName string

    // NodeTopology defines which nodes have access to the storage for which
    // capacity was reported. If not set, the storage is accessible from all
    // nodes in the cluster.
	// +optional
	NodeTopology *v1.NodeSelector

	// The storage class name of the StorageClass which provided the
    // additional parameters for the GetCapacity call.
	StorageClassName string

	// Capacity is the value reported by the CSI driver in its GetCapacityResponse.
    // Depending on how the driver is implemented, this might be the total
    // size of the available storage which is only available when allocating
    // multiple smaller volumes ("total available capacity") or the
    // actual size that a volume may have ("maximum volume size").
    //
    // It is optional because in the future, other ways of describing capacity
    // might get added. If not set, the scheduler will simply ignore the
    // CSIStorageCapacity object.
    //
	// +optional
	Capacity *resource.Quantity
}
```

Compared to the alternatives with a single object per driver (see
[`CSIDriver.Status`](#csidriverstatus) below) and one object per
topology (see [`CSIStoragePool`](#csistoragepool)), this approach has
the advantage that the size of the `CSIStorageCapacity` objects does
not increase with the potentially unbounded number of some other
objects (like storage classes).

The downsides are:
- Some attributes (driver name, topology) must be stored multiple times
  compared to a more complex object, so overall data size in etcd is higher.
- Higher number of objects which all need to be retrieved by a client
  which does not already know which `CSIStorageCapacity` object it is
  interested in.

##### Example: local storage

```
apiVersion: storage.k8s.io/v1alpha1
kind: CSIStorageCapacity
metadata:
  name: sp-ab96d356-0d31-11ea-ade1-8b7e883d1af1
spec:
  driverName: hostpath.csi.k8s.io
  storageClassName: some-storage-class
  capacity: 256G
  nodeTopology:
    nodeSelectorTerms:
    - matchExpressions:
      - key: kubernetes.io/hostname
        operator: In
        values:
        - node-1

apiVersion: storage.k8s.io/v1alpha1
kind: CSIStorageCapacity
metadata:
  name: sp-c3723f32-0d32-11ea-a14f-fbaf155dff50
spec:
  driverName: hostpath.csi.k8s.io
  storageClassName: some-storage-class
  capacity: 512G
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
kind: CSIStorageCapacity
metadata:
  name: sp-9c17f6fc-6ada-488f-9d44-c5d63ecdf7a9
spec:
  driverName: lvm
  storageClassName: striped
  capacity: 256G
  nodeTopology:
    nodeSelectorTerms:
    - matchExpressions:
      - key: kubernetes.io/hostname
        operator: In
        values:
        - node-1

apiVersion: storage.k8s.io/v1alpha1
kind: CSIStorageCapacity
metadata:
  name: sp-f0e03868-954d-11ea-9d78-9f197c0aea6f
spec:
  driverName: lvm
  storageClassName: mirrored
  capacity: 128G
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
kind: CSIStorageCapacity
metadata:
  name: sp-b0963bb5-37cf-415d-9fb1-667499172320
spec:
  driverName: pd.csi.storage.gke.io
  storageClassName: some-storage-class
  capacity: 128G
  nodeTopology:
    nodeSelectorTerms:
    - matchExpressions:
      - key: topology.kubernetes.io/region
        operator: In
        values:
        - us-east-1

apiVersion: storage.k8s.io/v1alpha1
kind: CSIStorageCapacity
metadata:
  name: sp-64103396-0d32-11ea-945c-e3ede5f0f3ae
spec:
  driverName: pd.csi.storage.gke.io
  storageClassName: some-storage-class
  capacity: 256G
  nodeTopology:
    nodeSelectorTerms:
    - matchExpressions:
      - key: topology.kubernetes.io/region
        operator: In
        values:
        - us-west-1
```

#### CSIDriver.spec.storageCapacity

A new field `storageCapacity` of type `boolean` with default `false`
in
[CSIDriver.spec](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.16/#csidriverspec-v1beta1-storage-k8s-io)
indicates whether a driver deployment will create `CSIStorageCapacity`
objects with capacity information and wants the Kubernetes scheduler
to rely on that information when making scheduling decisions that
involve volumes that need to be created by the driver.

If not set, the scheduler makes such decisions without considering
whether the driver really can create the volumes (the current situation).

### Updating capacity information with external-provisioner

Most (if not all) CSI drivers already get deployed on Kubernetes
together with the external-provisioner which then handles volume
provisioning via PVC.

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
that a volume of that size can be created. This KEP uses the reported
capacity to rule out pools which clearly have insufficient storage
because the reported capacity is smaller than the size of a
volume. This will work better when CSI drivers implement `GetCapacity`
such that they consider constraints like fragmentation and report the
size that the largest volume can have at the moment.

#### Without central controller

This mode of operation is expected to be used by CSI drivers that need
to track capacity per node and only support persistent volumes with
delayed binding that get provisioned by an external-provisioner
instance that runs on the node where the volume gets provisioned (see
the proposal for csi-driver-host-path in
https://github.com/kubernetes-csi/external-provisioner/pull/367).

The CSI driver has to implement the CSI controller service and its
`GetCapacity` call. Its deployment has to add the external-provisioner
to the daemon set and enable the per-node capacity tracking with
`--enable-capacity=local`.

The resulting `CSIStorageCapacity` objects then use a node selector for
one node.

#### With central controller

For central provisioning, external-provisioner gets deployed together
with a CSI controller service and capacity reporting gets enabled with
`--enable-capacity=central`. In this mode, CSI drivers must report
topology information in `NodeGetInfoResponse.accessible_topology` that
matches the storage pool(s) that it has access to, with granularity
that matches the most restrictive pool.

For example, if the driver runs in a node with zone/region/rack
topology and has access to one per-zone pool and one per-zone/region
pool, then the driver should report the zone/region of the second pool
as its topology.

This is a slight deviation from the CSI spec, which defines that
"`accessible_topology` specifies where the *node* is accessible
from". For node-local storage there is no difference, but for
network-attached storage there might be one.

Assuming that a CSI driver meets the above requirement and enables
this mode, external-provisioner then can identify pools as follows:
- iterate over all `CSINode` objects and search for
  `CSINodeDriver` information for the CSI driver,
- compute the union of the topology segments from these
  `CSINodeDriver` entries.

For each entry in that union, one `CSIStorageCapacity` object is
created for each storage class where the driver reports a non-zero
capacity. The node selector uses the topology key/value pairs as node
labels. That works because kubelet automatically labels nodes based on
the CSI drivers that run on that node.

#### With central controller, generic solution

For all other cases, a new CSI call to enumerate storage topology
segments will be needed. That call then might also include other
information for each segment.

#### Determining parameters

After determining the topology as described above,
external-provisioner needs to figure out with which volume parameters
it needs to call `GetCapacity`. It does that by iterating over all
storage classes that reference the driver.

If the current combination of topology segment and storage class
parameters do not make sense, then a driver must return "zero
capacity" or an error, in which case external-provisioner will skip
this combination. This covers the case where some storage class
parameter limits the topology segment of volumes using that class,
because information will then only be recorded for the segments that
are supported by the class.

#### CSIStorageCapacity lifecycle

external-provisioner needs permission to create, update and delete
`CSIStorageCapacity` objects. Before creating a new object, it must
check whether one already exists with the relevant attributes (driver
name + nodes + storage class) and then update that one
instead. Obsolete objects need to be removed.

To ensure that `CSIStorageCapacity` objects get removed when the driver
deployment gets removed before it has a chance to clean up, each
`CSIStorageCapacity` object needs an [owner
reference](https://godoc.org/k8s.io/apimachinery/pkg/apis/meta/v1#OwnerReference).

For central provisioning, that has to be the deployment or stateful
set that defines the provisioner pods. That way, provisioning can
continue seamlessly when there are multiple instances with leadership
election.

For deployment with a daemon set, making the individual pod the owner
is better than the daemon set as owner because then other instances do
not need to figure out when to remove `CSIStorageCapacity` objects
created on another node. It is also better than the node as owner
because a driver might no longer be needed on a node although the node
continues to exist (for example, when deploying a driver to nodes is
controlled via node labels).

While external-provisioner runs, it needs to update and potentially
delete `CSIStorageCapacity` objects:
- when nodes change (for central provisioning)
- when storage classes change (for persistent volumes)
- when volumes were created or deleted (for central provisioning)
- when volumes are resized or snapshots are created or deleted (for persistent volumes)
- periodically, to detect changes in the underlying backing store (all cases)

Because sidecars are currently separated, external-provisioner is
unaware of resizing and snapshotting. The periodic polling will catch up
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
only necessary for PVCs that have not been bound yet.

The lookup sequence will be:
- find the `CSIDriver` object for the driver
- check whether it has `CSIDriver.spec.storageCapacity` enabled
- find all `CSIStorageCapacity` objects that have the right spec
  (driver, accessible by node, storage class) and sufficient capacity.

The specified volume size is compared against `Capacity` if
available. A topology segment which has no reported capacity or a
capacity that is too small is considered unusable at the moment and
ignored.

Each volume gets checked separately, independently of other volumes
that are needed by the current pod or by volumes that are about to be
created for other pods. Those scenarios remain problematic.

Trying to model how different volumes affect capacity would be
difficult. If the capacity represents "maximum volume size 10GiB", it may be possible
to create exactly one such volume or several, so rejecting the
pool after one volume could be a false negative. With "available
capacity 10GiB" it may or may not be possible to create two volumes of
5GiB each, so accepting the node for two such volumes could be a false
positive.

More promising might be to add prioritization of nodes based on how
much capacity they have left, thus spreading out storage usage evenly.
This is a likely future extension of this KEP.

Either way, the problem of recovering more gracefully from running out
of storage after a bad scheduling decision will have to be addressed
eventually. Details for that are in https://github.com/kubernetes/enhancements/pull/1703.

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
will only be valid when tested with Kubernetes clusters where
alpha features are enabled.

The CSI hostpath driver needs to be changed such that it reports the
remaining capacity of the filesystem where it creates volumes. The
existing raw block volume tests then can be used to ensure that pod
scheduling works:
- Those volumes have a size set.
- Late binding is enabled for the CSI hostpath driver.

A new test can be written which checks for `CSIStorageCapacity` objects,
asks for pod scheduling with a volume that is too large, and then
checks for events that describe the problem.

### Graduation Criteria

#### Alpha -> Beta Graduation

- Gather feedback from developers and users
- Integration with [Cluster Autoscaler](https://github.com/kubernetes/autoscaler)
- Generic solution for identifying storage pools
- Re-evaluate API choices, considering:
  - performance
  - extensions of the API that may or may not be needed (like
    [ignoring storage class
    parameters](#storage-class-parameters-that-never-affect-capacity))
- Tests are in Testgrid and linked in KEP

#### Beta -> GA Graduation

- 5 CSI drivers enabling the creation of `CSIStorageCapacity` data
- 5 installs
- More rigorous forms of testing e.g., downgrade tests and scalability tests
- Allowing time for feedback

### Upgrade / Downgrade Strategy

<!--
If applicable, how will the component be upgraded and downgraded? Make sure
this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this
enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to keep previous behavior?
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to make use of the enhancement?
-->

### Version Skew Strategy

<!--
If applicable, how will the component handle version skew with other
components? What are the guarantees? Make sure this is in the test plan.

Consider the following in developing a version skew strategy for this
enhancement:
- Does this enhancement involve coordinating behavior in the control plane and
  in the kubelet? How does an n-2 kubelet without this feature available behave
  when this feature is used?
- Will any other components on the node change? For example, changes to CSI,
  CRI or CNI may require updating that component before the kubelet.
-->

## Production Readiness Review Questionnaire

### Feature enablement and rollback

* **How can this feature be enabled / disabled in a live cluster?**
  - [X] Feature gate
    - Feature gate name: CSIStorageCapacity
    - Components depending on the feature gate:
      - apiserver
      - kube-scheduler

* **Does enabling the feature change any default behavior?**
  No changes for existing storage driver deployments.

* **Can the feature be disabled once it has been enabled (i.e. can we rollback
  the enablement)?**
  Yes.

  When disabling support in the API server, the new object type and
  thus all objects will disappear together with the new flag in
  `CSIDriver`, which will then cause kube-scheduler to revert to
  provisioning without capacity information. However, the new objects
  will still remain in the etcd database.

  When disabling it in the scheduler, provisioning directly reverts
  to the previous behavior.

  External-provisioner will be unable to update objects: this needs to
  be treated with exponential backoff just like other communication
  issues with the API server.

* **What happens if we reenable the feature if it was previously rolled back?**

  Stale objects will either get garbage collected via their ownership relationship
  or get updated by external-provisioner. Scheduling with capacity information
  resumes.

* **Are there any tests for feature enablement/disablement?**
  The e2e framework does not currently support enabling and disabling feature
  gates. However, unit tests in each component dealing with managing data created
  with and without the feature are necessary. At the very least, think about
  conversion tests if API types are being modified.

### Rollout, Upgrade and Rollback Planning

_This section must be completed when targeting beta graduation to a release._

* **How can a rollout fail? Can it impact already running workloads?**
  Try to be as paranoid as possible - e.g. what if some components will restart
  in the middle of rollout?

* **What specific metrics should inform a rollback?**

* **Were upgrade and rollback tested? Was upgrade->downgrade->upgrade path tested?**
  Describe manual testing that was done and the outcomes.
  Longer term, we may want to require automated upgrade/rollback tests, but we
  are missing a bunch of machinery and tooling and do that now.

* **Is the rollout accompanied by any deprecations and/or removals of features,
  APIs, fields of API types, flags, etc.?**
  Even if applying deprecation policies, they may still surprise some users.

### Monitoring requirements

_This section must be completed when targeting beta graduation to a release._

* **How can an operator determine if the feature is in use by workloads?**
  Ideally, this should be a metrics. Operations against Kubernetes API (e.g.
  checking if there are objects with field X set) may be last resort. Avoid
  logs or events for this purpose.

* **What are the SLIs (Service Level Indicators) an operator can use to
  determine the health of the service?**
  - [ ] Metrics
    - Metric name:
    - [Optional] Aggregation method:
    - Components exposing the metric:
  - [ ] Other (treat as last resort)
    - Details:

* **What are the reasonable SLOs (Service Level Objectives) for the above SLIs?**
  At the high-level this usually will be in the form of "high percentile of SLI
  per day <= X". It's impossible to provide a comprehensive guidance, but at the very
  high level (they needs more precise definitions) those may be things like:
  - per-day percentage of API calls finishing with 5XX errors <= 1%
  - 99% percentile over day of absolute value from (job creation time minus expected
    job creation time) for cron job <= 10%
  - 99,9% of /health requests per day finish with 200 code

* **Are there any missing metrics that would be useful to have to improve
  observability if this feature?**
  Describe the metrics themselves and the reason they weren't added (e.g. cost,
  implementation difficulties, etc.).

### Dependencies

_This section must be completed when targeting beta graduation to a release._

* **Does this feature depend on any specific services running in the cluster?**
  Think about both cluster-level services (e.g. metrics-server) as well
  as node-level agents (e.g. specific version of CRI). Focus on external or
  optional services that are needed. For example, if this feature depends on
  a cloud provider API, or upon an external software-defined storage or network
  control plane.

  For each of the fill in the following, thinking both about running user workloads
  and creating new ones, as well as about cluster-level services (e.g. DNS):
  - [Dependency name]
    - Usage description:
      - Impact of its outage on the feature:
      - Impact of its degraded performance or high error rates on the feature:


### Scalability

_For alpha, this section is encouraged: reviewers should consider these questions
and attempt to answer them._

_For beta, this section is required: reviewers must answer these questions._

_For GA, this section is required: approvers should be able to confirms the
previous answers based on experience in the field._

* **Will enabling / using this feature result in any new API calls?**
  Describe them, providing:
  - API call type (e.g. PATCH pods)
  - estimated throughput
  - originating component(s) (e.g. Kubelet, Feature-X-controller)
  focusing mostly on:
  - components listing and/or watching resources they didn't before
  - API calls that may be triggered by changes of some Kubernetes resources
    (e.g. update of object X triggers new updates of object Y)
  - periodic API calls to reconcile state (e.g. periodic fetching state,
    heartbeats, leader election, etc.)

* **Will enabling / using this feature result in introducing new API types?**
  Describe them providing:
  - API type
  - Supported number of objects per cluster
  - Supported number of objects per namespace (for namespace-scoped objects)

* **Will enabling / using this feature result in any new calls to cloud
  provider?**

* **Will enabling / using this feature result in increasing size or count
  of the existing API objects?**
  Describe them providing:
  - API type(s):
  - Estimated increase in size: (e.g. new annotation of size 32B)
  - Estimated amount of new objects: (e.g. new Object X for every existing Pod)

* **Will enabling / using this feature result in increasing time taken by any
  operations covered by [existing SLIs/SLOs][]?**
  Think about adding additional work or introducing new steps in between
  (e.g. need to do X to start a container), etc. Please describe the details.

* **Will enabling / using this feature result in non-negligible increase of
  resource usage (CPU, RAM, disk, IO, ...) in any components?**
  Things to keep in mind include: additional in-memory state, additional
  non-trivial computations, excessive access to disks (including increased log
  volume), significant amount of data send and/or received over network, etc.
  This through this both in small and large cases, again with respect to the
  [supported limits][].

### Troubleshooting

Troubleshooting section serves the `Playbook` role as of now. We may consider
splitting it into a dedicated `Playbook` document (potentially with some monitoring
details). For now we leave it here though.

_This section must be completed when targeting beta graduation to a release._

* **How does this feature react if the API server and/or etcd is unavailable?**

* **What are other known failure modes?**
  For each of them fill in the following information by copying the below template:
  - [Failure mode brief description]
    - Detection: How can it be detected via metrics? Stated another way:
      how can an operator troubleshoot without loogging into a master or worker node?
    - Mitigations: What can be done to stop the bleeding, especially for already
      running user workloads?
    - Diagnostics: What are the useful log messages and their required logging
      levels that could help debugging the issue?
      Not required until feature graduated to Beta.
    - Testing: Are there any tests for failure mode? If not describe why.

* **What steps should be taken if SLOs are not being met to determine the problem?**

[supported limits]: https://git.k8s.io/community//sig-scalability/configs-and-limits/thresholds.md
[existing SLIs/SLOs]: https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos


## Implementation History

- Kubernetes 1.19: alpha (tentative)


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

### CSI drivers without topology support

To simplify the implementation of external-provisioner, [topology
support](https://kubernetes-csi.github.io/docs/topology.html) is
expected from a CSI driver. A driver which does not really need
topology support can add it simply by always returning the same static
`NodeGetInfoResponse.AccessibleTopology`.

### Storage class parameters that never affect capacity

In the current proposal, `GetCapacity` will be called for every every
storage class. This is extra work and will lead to redundant
`CSIStorageCapacity` entries for CSI drivers where the storage
class parameters have no effect.

To handles this special case, a special `<fallback>` storage class
name and a corresponding flag in external-provisioner could be
introduced: if enabled by the CSI driver deployment, storage classes
then would be ignored and the scheduler would use the special
`<fallback>` entry to determine capacity.

This was removed from an earlier draft of the KEP to simplify it.

### Multiple capacity values

Some earlier draft specified `AvailableCapacity` and
`MaximumVolumeSize` in the API to avoid the ambiguity in the CSI
API. This was deemed unnecessary because it would have made no
difference in practice for the use case in this KEP.

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
	// using that storage class for a volume. There is also one
	// special name:
	// - <ephemeral> for storage used by ephemeral inline volumes (which
	//   don't use a storage class)
	StorageClassName string `json:"storageClassName" protobuf:"bytes,1,name=storageClassName"`

	// Capacity is the size of the largest volume that currently can
	// be created. This is a best-effort guess and even volumes
	// of that size might not get created successfully.
	// +optional
	Capacity *resource.Quantity `json:"capacity,omitempty" protobuf:"bytes,2,opt,name=capacity"`
}

const (
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
      storageClassName: some-storage-class
    name: node-1
    nodes:
    - node-1
  - classes:
    - capacity: 512G
      storageClassName: some-storage-class
    name: node-2
    nodes:
    - node-2
```

#### Example: affect of storage classes

This fictional LVM CSI driver can either use 256GB of local disk space
for striped or mirror volumes. Mirrored volumes need twice the amount
of local disk space, so the capacity is halved:

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
      storageClassName: some-storage-class
    name: region-east
    nodeTopology:
      nodeSelectorTerms:
      - matchExpressions:
        - key: topology.kubernetes.io/region
          operator: In
          values:
          - us-east-1
  - classes:
    - capacity: 256G
      storageClassName: some-storage-class
    name: region-west
    nodeTopology:
      nodeSelectorTerms:
      - matchExpressions:
        - key: topology.kubernetes.io/region
          operator: In
          values:
          - us-west-1
```

### CSIStoragePool

Instead of one `CSIStorageCapacity` object per `GetCapacity` call, in
this alternative API the result of multiple calls would be combined in
one object. This is potentially more efficient (less objects, less
redundant data) while still fitting the producer/consumer model in
this proposal. It also might be better aligned with other, future
enhancements.

It was rejected during review because the API is more complex.

```
// CSIStoragePool identifies one particular storage pool and
// stores its attributes. The spec is read-only.
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
// cluster as identified by NodeTopology. If not set, the pool is assumed
// to be available in the entire cluster.
//
// It is expected to be extended with other
// attributes which do not depend on the storage class, like health of
// the pool. Capacity may depend on the parameters in the storage class and
// therefore is stored in a list of `CSIStorageByClass` instances.
type CSIStoragePoolStatus struct {
	// NodeTopology can be used to describe a storage pool that is available
	// only for nodes matching certain criteria.
	// +optional
	NodeTopology *v1.NodeSelector

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
	// using that storage class for a volume.
	StorageClassName string `json:"storageClassName" protobuf:"bytes,1,name=storageClassName"`

	// Capacity is the value reported by the CSI driver in its GetCapacityResponse.
    // Depending on how the driver is implemented, this might be the total
    // size of the available storage which is only available when allocating
    // multiple smaller volumes ("total available capacity") or the
    // actual size that a volume may have ("maximum volume size").
	// +optional
	Capacity *resource.Quantity `json:"capacity,omitempty" protobuf:"bytes,2,opt,name=capacity"`
}
```

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
  - capacity: 256G
    storageClassName: some-storage-class
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
  - capacity: 512G
    storageClassName: some-storage-class
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
  - capacity: 256G
    storageClassName: striped
  - capacity: 128G
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
  - capacity: 128G
    storageClassName: some-storage-class
  nodeTopology:
    nodeSelectorTerms:
    - matchExpressions:
      - key: topology.kubernetes.io/region
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
  - capacity: 256G
    storageClassName: some-storage-class
  nodeTopology:
    nodeSelectorTerms:
    - matchExpressions:
      - key: topology.kubernetes.io/region
        operator: In
        values:
        - us-west-1
```

### Prior work

The [Topology-aware storage dynamic
provisioning](https://docs.google.com/document/d/1WtX2lRJjZ03RBdzQIZY3IOvmoYiF5JxDX35-SsCIAfg)
design document used a different data structure and had not fully
explored how that data structure would be populated and used.
