<!--
**Note:** When your KEP is complete, all of these comment blocks should be removed.

To get started with this template:

- [ ] **Pick a hosting SIG.**
  Make sure that the problem space is something the SIG is interested in taking
  up. KEPs should not be checked in without a sponsoring SIG.
- [ ] **Create an issue in kubernetes/enhancements**
  When filing an enhancement tracking issue, please make sure to complete all
  fields in that template. One of the fields asks for a link to the KEP. You
  can leave that blank until this KEP is filed, and then go back to the
  enhancement and add the link.
- [ ] **Make a copy of this template directory.**
  Copy this template into the owning SIG's directory and name it
  `NNNN-short-descriptive-title`, where `NNNN` is the issue number (with no
  leading-zero padding) assigned to your enhancement above.
- [ ] **Fill out as much of the kep.yaml file as you can.**
  At minimum, you should fill in the "Title", "Authors", "Owning-sig",
  "Status", and date-related fields.
- [ ] **Fill out this file as best you can.**
  At minimum, you should fill in the "Summary" and "Motivation" sections.
  These should be easy if you've preflighted the idea of the KEP with the
  appropriate SIG(s).
- [ ] **Create a PR for this KEP.**
  Assign it to people in the SIG who are sponsoring this process.
- [ ] **Merge early and iterate.**
  Avoid getting hung up on specific details and instead aim to get the goals of
  the KEP clarified and merged quickly. The best way to do this is to just
  start with the high-level sections and fill out details incrementally in
  subsequent PRs.

Just because a KEP is merged does not mean it is complete or approved. Any KEP
marked as `provisional` is a working document and subject to change. You can
denote sections that are under active debate as follows:

```
<<[UNRESOLVED optional short context or usernames ]>>
Stuff that is being argued.
<<[/UNRESOLVED]>>
```

When editing KEPS, aim for tightly-scoped, single-topic PRs to keep discussions
focused. If you disagree with what is already in a document, open a new PR
with suggested changes.

One KEP corresponds to one "feature" or "enhancement" for its whole lifecycle.
You do not need a new KEP to move from beta to GA, for example. If
new details emerge that belong in the KEP, edit the KEP. Once a feature has become
"implemented", major changes should get new KEPs.

The canonical place for the latest set of instructions (and the likely source
of this file) is [here](/keps/NNNN-kep-template/README.md).

**Note:** Any PRs to move a KEP to `implementable`, or significant changes once
it is marked `implementable`, must be approved by each of the KEP approvers.
If none of those approvers are still appropriate, then changes to that list
should be approved by the remaining approvers and/or the owning SIG (or
SIG Architecture for cross-cutting KEPs).
-->

# KEP-4381: Semantic Parameters for Dynamic Resource Allocation

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Publishing node resources](#publishing-node-resources)
  - [Using semantic parameters as claim parameters](#using-semantic-parameters-as-claim-parameters)
  - [Communicating allocation to the DRA driver](#communicating-allocation-to-the-dra-driver)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
- [Design Details](#design-details)
  - [ResourceClass extension](#resourceclass-extension)
  - [NodeResourceSlice](#noderesourceslice)
  - [ResourceClaimParameters](#resourceclaimparameters)
  - [ResourceHandle extension](#resourcehandle-extension)
  - [Semantic models](#semantic-models)
  - [Scheduling + Allocation](#scheduling--allocation)
  - [Deallocation](#deallocation)
  - [Immediate allocation](#immediate-allocation)
  - [Simulation with CA](#simulation-with-ca)
  - [Test Plan](#test-plan)
      - [Prerequisite testing updates](#prerequisite-testing-updates)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Beta](#beta)
    - [GA](#ga)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
  - [Injecting vendor logic into CA](#injecting-vendor-logic-into-ca)
<!-- /toc -->

## Release Signoff Checklist

<!--
**ACTION REQUIRED:** In order to merge code into a release, there must be an
issue in [kubernetes/enhancements] referencing this KEP and targeting a release
milestone **before the [Enhancement Freeze](https://git.k8s.io/sig-release/releases)
of the targeted release**.

For enhancements that make changes to code or processes/procedures in core
Kubernetes—i.e., [kubernetes/kubernetes], we require the following Release
Signoff checklist to be completed.

Check these off as they are completed for the Release Team to track. These
checklist items _must_ be updated for the enhancement to be released.
-->

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input (including test refactors)
  - [ ] e2e Tests for all Beta API Operations (endpoints)
  - [ ] (R) Ensure GA e2e tests meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) 
  - [ ] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [ ] (R) Graduation criteria is in place
  - [ ] (R) [all GA Endpoints](https://github.com/kubernetes/community/pull/1806) must be hit by [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) 
- [ ] (R) Production readiness review completed
- [ ] (R) Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

<!--
**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.
-->

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

Dynamic Resource Allocation (DRA) was added to Kubernetes as an alpha feature in v1.26. It defines an alternative to the traditional device-plugin API for requesting access to third-party resources.


By design, DRA uses parameters for resources that are completely
opaque to core Kubernetes. They get interpreted by a DRA driver's controller
(for allocating claims) and a DRA driver's kubelet plugin (for configuring
resources a node). During scheduling of a pod, the kube-scheduler and any DRA
driver controller(s) handling claims for the pod communicate back-and-forth through the
apiserver by updating a `PodSchedulingContext` object, ultimately leading to the
allocation of all pending claims and the pod being scheduled onto a node.

This approach poses a problem for the [Cluster
Autoscaler](https://github.com/kubernetes/autoscaler) (CA) or for any higher
level controller that needs to make decisions for a group of pods (e.g. a job
scheduler). It cannot simulate the effect of allocating or deallocating
claims over time. Only the third-party DRA
drivers have the information available to do this.

"Semantic parameters" is an extension to DRA that addresses this problem by
making claim parameters less opaque. Instead of managing the semantics of all
claim parameters themselves, drivers now manage resources and describe them
using a specific "semantic model" pre-defined by Kubernetes. This allows
components aware of this "semantic model" to make decisions about these
resources without outsourcing them to some third-party controller. For example,
the scheduler is now able to allocate claims rapidly, without back-and-forth
communication with DRA drivers.

At a high-level, this extension takes the following form:

* DRA drivers publish their available resources in the form of a
  `NodeResourceSlice` object on a node-by-node basis according to one of the
  builtin "semantic models" known to Kubernetes. This object is stored in the
  API server and available to the scheduler (or Cluster Autoscaler) to query
  when a resource request comes in later on.

* When a user wants to consume a resource, they create a `ResourceClaim`,
  which, in turn, references a claim parameters object. This object defines how
  many resources are needed and which capabilities they must have. Typically, it
  is defined using a vendor-specific type which might also support configuration
  parameters (i.e. parameters that are *not* needed for allocation but *are*
  needed for configuration).

* With such a claim in place, DRA drivers "resolve" the contents of any
  vendor-specific claim parameters into a canonical form (i.e. a generic
  `ResourceClaimParameters` object in the `resource.k8s.io` API group) which
  the scheduler (or Cluster Autoscaler) can evaluate against the
  `NodeResourceSlice` of any candidate nodes without knowing exactly what is
  being requested. They then use this information to help decide which node to
  schedule a pod on (as well as allocate resources from its `NodeResourceSlice`
  in the process).

* Once a node is chosen and a workload is sent to it, DRA drivers are
  responsible for injecting any allocated resources into the Pod, according to
  the resource choices made by the scheduler. This includes applying any
  configuration information attached to the vendor-specific claim parameters
  object used in the request.

In this KEP, we define a single "semantic model" called
`PartitionableResources`, and use it throughout the discussion to help describe
the framework as a whole. This model is a natural extension to the API provided
by the traditional device-plugin, which supports advertising resources as a
finite set of opaque strings. The major difference being that under the new
model, one can attach a list of attributes to those opaque strings which the
scheduler can use to select a particular resource for allocation. Additionally,
each resource can be divided into a set of pre-defined partitions, each with
their own set of unique attributes attached to them. In this way, requests for
resources can be satisfied either by a complete resource instance or by one of
its partitions.  This model may get enhanced in the future and/or other models
might get added as needed.

## Motivation

<!--
This section is for explicitly listing the motivation, goals, and non-goals of
this KEP.  Describe why the change is important and the benefits to users. The
motivation section can optionally provide links to [experience reports] to
demonstrate the interest in a KEP within the wider Kubernetes community.

[experience reports]: https://github.com/golang/go/wiki/ExperienceReports
-->

### Goals

- Enable cluster autoscaling when pods use resource claims, with correct
  decisions and changing the cluster size by more than one node at a time.

- Support node-local resources. Adding or removing nodes has no effect
  on network-attached resources and therefore CA does not need to (and cannot)
  simulate them.

- Allow DRA driver developers to provide a user experience that is similar to
  the one possible without semantic parameters. Ideally, users should not notice
  at all that a driver is using semantic parameters under the hood.

### Non-Goals

- Scheduling performance is expected to become better compared to using the
  PodSchedulingContext. However, this is not the reason for this KEP.


## Proposal

### Publishing node resources

The kubelet publishes NodeResourceSlices to the API server with content
provided by DRA drivers running on each node. Access control through the node
authorizer ensures that the kubelet running on one node is not allowed to
create or modify NodeResourceSlices belonging to other nodes.

NodeResourceSlices are published separately for each driver, using a version of
the API that matches what that driver is using. This implies that the kubelet, DRA
driver, and apiserver must all support the *same* API version. It might be
possible to support version skew (= keeping kubelet at an older version than
the control plane and the DRA drivers) in the future, but currently this is out
of scope.

Embedded inside a NodeResourceSlice is the representation of the resources
managed by a driver. With the "semantic model" of `PartitionableResources` this
takes the form:

```yaml
kind: NodeResourceSlice
apiVersion: resource.k8s.io/v1alpha2
...
spec:
  nodeName: worker-1
  driverName: cards.dra.example.com
  partitionable:
    ...
```

Here all resources associated with the `cards.dra.example.com` driver are to
be placed under the `partitionable` field in the `NodeResourceSlice` object. As
we add more "semantic models" in the future, *alternate* fields will be added
at the same level as `partitionable`, and driver implementors will be able to
choose which one they want to use to represent their resources.

Below is an example of a driver that provides two discrete GPU cards using the
`partitionable` model. The first card can only be used as a whole. The second
card can be used as a whole, split up into four equal pieces, in halves, or one
half and two quarters:

```yaml
kind: NodeResourceSlice
apiVersion: resource.k8s.io/v1alpha2
...
spec:
  nodeName: worker-1
  driverName: cards.dra.example.com
  partitionable:
    commonAttributes:
    - name: drivers
      attributes:
      - name: driverVersion
        String: 1.2.3
      - name: runtimeVersion
        String: 11.1.42
    - name: t1000-gpu
      attributes:
      - name: type
        string: GPU
      - name: memory
        quantity: 16Gi
      - name: productName:
        string: ACME T1000 32GB
    - name: a4-gpu
      attributes:
      - name: type
        String: GPU
      - name: memory
        quantity: 32Gi
      - name: productName:
        String: ACME A4-PCIE-40GB
    - name: a4-half
      attributes:
      - name: memory
        quantity: 16Gi
    - name: a4-quarter
      attributes:
      - name: memory
        quantity: 8Gi

    resourceInstances:
    - name: gpu-0
      commonAttributesRefs:
      - drivers
      - t1000-gpu
      attributes:
      - name: UUID
        string: GPU-ceea231c-4257-7af7-6726-efcb8fc2ace9
    - name: partitionable-gpu-1
      commonAttributesRefs:
      - drivers
      - a4-gpu
      attributes:
      - name: UUID
        string: GPU-6aa0af9e-a2be-88c8-d2b3-2240d25318d7
      partitions:
      - name: full-gpu
        resourceInstances:
        - name: gpu-1
          # Same attributes as referenced under partitionable-gpu-2.
      - name: quarters
        commonAttributesRefs:
        - a4-quarter # less memory for each sub-instance.
        resourceInstances:
        - name: quarter-0
        - name: quarter-1
        - name: quarter-2
        - name: quarter-3
      - name: half-and-quarters
        resourceInstances:
        - name: first-half
          comonAttributesRefs:
          - a4-half
        - name: partitionable-second-half
          partitions:
          - name: half
            comonAttributesRefs:
            - a4-half
            resourceInstances:
            - name: second-half
          - name: quarters
            comonAttributesRefs:
            - a4-quarters
            resourceInstances:
            - name: quarter-0
            - name: quarter-1
```

As mentioned previously, exactly one semantic model must be used by a given
driver. If a new model is added to the schema but clients are not updated,
they'll encounter an object with no information from any known semantic model
when they serialize into their known version of a NodeResourceSlice. This tells
them that they cannot handle the object because the API was extended.

Compared to labels, attributes in this model have values of exactly one
type. As described later on, these attributes get used in CEL expressions that
the scheduler can use to select a specific resource for allocation on a node.

To avoid repetition, common sets of
attributes are defined once and referenced where needed. The example uses
relatively simple sets, however, real-world use cases will include more information.

The partitioning forms a tree. Nodes further down in that tree inherit the
attributes of their parent and can overwrite those attributes where the values
are different.

In this example, "partitionable-gpu-1/half-and-quarters/second-half/quarter-0"
indentifies one quarter of the second GPU in one particular configuration. It
has "8Gi" of memory. Note that "memory" for this hardware is not something
that can be sliced up and consumed arbitrarily. Instead, the partitioning
defines how much if it is available, which makes the memory size an attribute
of each individual sub-partition.

The "partitionable-gpu-1" GPU can be used either with the "full-gpu", "quarters",
or "half-and-quarters" partitioning scheme. When using "half-and-quarters", there's
another choice between a "half" and "quarters" partitioning scheme for the second half. Once a
partitioning scheme has been chosen to satisfy a claim, it cannot be changed
until all claims bound to resources from that partition have been deallocated.

At the moment, resource attribute names and their type are defined by
vendors. Resource names with ".k8s.io" as a suffix are reserved for future use
and standardization by Kubernetes. This could be used to describe topology
across resources from different vendors, but this is out-of-scope for now.

If a driver needs to reduce resource capacity, then there is a risk that a
claim gets allocated using that capacity while the kubelet is updating a
NodeResourceSlice. The implementations of semantic models must handle scenarios
where more resources are allocated than available. The kubelet plugin of a DRA
driver ***must*** double-check that the allocated resources are still available
when NodePrepareResource is called. If not, the pod cannot start until the
resource comes back. Treating this as a fatal error during pod admission would
allow us to delete the pod and trying again with a new one.

### Using semantic parameters

The following is an example CRD which the developer of the
`cards.dra.example.com` DRA driver might define as a valid claim parameters
object for requesting access to its GPUs:

```yaml
kind: CardParameters
apiVersion: dra.example.com/v1alpha1
metadata:
  name: my-parameters
  namespace: user-namespace
  uid: foobar-uid
...
spec:
  count: 2
  minimumRuntimeVersion: v12.0.0
  minimumMemory: 8Gi
  # "sharing" is a configuration parameter that does not
  # get translated into the selector below.
  sharing:
    strategy: TimeSliced
```

Note that all fields in this CRD can be fully validated since it is owned by
the DRA driver itself. This includes value ranges that are specific to the
underlying hardware. There's no risk of using invalid attribute names because
only the fields shown here are valid.

With this CRD in place, a DRA driver controller is able to convert instances of
it into a generic, in-tree `ResourceClaimParameters` object that the scheduler
is able to understand.

For the example above, the converted object would look as follows:

```yaml
kind: ResourceClaimParameters
apiVersion: resource.k8s.io/v1alpha2

metadata:
  # This cannot be the same as my-parameters because parameter objects with a different
  # type might also use it. Instead, the original object gets linked to below.
  name: someArbitraryName
  namespace: user-namespace

generatedFrom:
  name: my-parameters
  kind: CardParameters
  apiGroup: dra.example.com
  uid: foobar-uid

vendorParameters:
  # A vendor can put any kind of object here to pass the configuration
  # parameters down to the kubelet plugin. In this case, the vendor
  # driver simply copied the entire CR. It could also be some
  # separate, smaller configuration type.
  kind: CardParameters
  apiVersion: dra.example.com/v1alpha1
  metadata:
    name: my-parameters
    namespace: user-namespace
    uid: foobar-uid
  ...
  spec:
    count: 2
    minimumRuntimeVersion: v12.0.0
    minimumMemory: 8Gi
    sharing:
      strategy: TimeSliced

requests:
- driverName: cards.dra.example.com
  partitionable:
    required:
    - name: selected-gpu-0
      # A CEL expression with access to the attributes of the sub-partition
      # that is being checked for a match.
      selector: |-
        versions["runtimeVersion"] >= "v12.0.0" && quantities["memory"] >= "8Gi"
    - name: selected-gpu-1
      # Same as above in this example, but could also be different.
      selector: |-
        versions["runtimeVersion"] >= "v12.0.0" && quantities["memory"] >= "8Gi"
```

The semantic is that the selector expression must evaluate to true for a
particular sub-partition to be usable. The initial implementation will use a
very simplistic approach for satisfying requests: it iterates over the requests
in order and uses the first sub-partition in a depth-first search which
matches. This locks that instance into a certain partitioning, which then
influences whether the next request still can be satisfied. With that in mind,
a better way to list sub-partitions would be to list the small ones first in
the NodeResourceSlice, to avoid using a whole GPU for a claim that could have
used a smaller sub-partition instead.

Eventually, this implementation will have to become smarter. It needs to
support backtracking and scoring of possible solutions to find the best one
which satisfies the requests. This could be extended to support finding the
best solution for a set of pods and their claims.

Another future extension is to express constraints that must be satisfied for
the selected instances. For example, selecting two cards which are on the same
PCI root complex may be needed to get the required performance.

Instead of defining a vendor CRD, DRA driver authors or administrators may also
decide to allow users to create and reference ResourceClaimParameters directly
in their ResourceClaim. Then the translation step from vendor CRD is not
needed. The downside is a lack of validation of these user-created
ResourceClaimParameters.

Resource class parameters are supported the same way. To ensure that
permissions can be limited to administrators, there's a separate cluster-scoped
ResourceClassParameters type. Instead of individual requests, one additional
selector can be specified there which then also must be true for all individual
requests made with that class:

```yaml
kind: ResourceClassParameters
apiVersion: resource.k8s.io/v1alpha2

metadata:
  name: someArbitraryName

generatedFrom:
  name: gpu-parameters
  kind: CardClassParameters
  apiGroup: dra.example.com
  uid: foobar-uid

vendorParameters:
  ...

filters:
- driverName: cards.dra.example.com
  partitionable:
    selector: |-
      quantities["memory"] <= "16Gi"
```

In this example, the additional selector expression limits users of this class
to smaller sub-partitions. Together with limiting the number of claims that
users are allowed to create for this class (see resource quotas in the core
KEP) this can ensure that users do not consume too much resources. Allowing
resource quotas that are based on resource attributes may be a useful future
enhancement.

### Communicating allocation to the DRA driver

The scheduler decides which resources to use for a claim and how much of
them. It also needs to take a snapshot of the full class and claim parameter
objects at the time of allocation. For the claim parameters, this has to be of
the vendor CR because only that contains the configuration parameters. Class
parameters are optional but might also be used to provide additional
configuration.

All of this information gets stored in the allocation result inside the
ResourceClaim status. For the example above, the result is simply the list of
IDs of the selected sub-partitions and a snapshot of the original vendor CR
with the "sharing" configuration parameter:

```yaml
# Matches with the SemanticResourceHandle Go type defined below.
vendorClassParameters:
  ...
vendorClaimParameters:
  kind: CardParameters
  apiVersion: dra.example.com/v1alpha1
  metadata:
    name: my-parameters
    namespace: user-namespace
    uid: foobar-uid
  ...
  spec:
    count: 2
    minimumRuntimeVersion: v12.0.0
    minimumMemory: 8Gi
    sharing:
      strategy: TimeSliced

nodeName: worker-1
partitionable:
  instances:
  - partitionable-gpu-2/half-and-quarters/second-half/quarter-0
  - partitionable-gpu-2/half-and-quarters/second-half/quarter-1
```

Taking these snapshots ensures that the claim remains usable even when
everything else gets deleted. This is particularly important when it is already
in use when that happens because cleanup may depend on the information. It also
avoids the need to grant the kubelet and/or DRA driver access to the vendor
CRDs, something that couldn't get locked down by the node authorizer.

### Notes/Constraints/Caveats

When deploying a driver, suitable role bindings for
[`system:kube-scheduler`](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#core-component-roles)
have to be added which grant read, watch and list permissions for the vendor
CRD.

The types that the scheduler needs to watch may change at runtime as classes
get added or removed. To avoid repeated GET operations for those CRDs, dynamic
informers will have to be used that get started and stopped as needed.

## Design Details

### ResourceClass extension

A new, optional field in ResourceClass enables semantic parameters for claims
using this class:

```go
type ResourceClass struct {
    ...

    // If (and only if) allocation of claims using this class is handled
    // via semantic parameters, then SemanticParameters must be set to true.
    SemanticParameters bool
}
```

### NodeResourceSlice

For each node, one or more NodeResourceSlice objects get created. The kubelet
publishes them with the node as the owner, so they get deleted when a node goes
down and then gets removed.

All list types are atomic because that makes tracking the owner for
server-side-apply (SSA) simpler. Patching individual list elements is not
needed and there is a single owner (kubelet).

```go
// NodeResourceSlice provides information about available
// resources on individual nodes.
type NodeResourceSlice struct {
    metav1.TypeMeta
    // Standard object metadata
    // +optional
    metav1.ObjectMeta

    Spec NodeResourceSliceSpec
}
```

There is only a spec for a NodeResourceSlice at the moment. It holds the
information about available resources. A status is not strictly needed because
the information in the allocated claim statuses is sufficient to determine
which of those resources are reserved for claims.

However, despite the finalizer on the claims it could happen that a
sufficiently determined user deletes a claim while it is in use. Therefore
adding a status is a useful future extension. That status will include
information about reserved resources (set by schedulers before allocating a
claim) and in-use resources (set by kubelet). This then enables conflict
resolution when multiple schedulers schedule pods to the same node because they
would be required to set a reservation before proceeding with the
allocation. It also enables detecting inconsistencies and taking actions to fix
those, like deleting pods which use a deleted claim.

```go
type NodeResourceSliceSpec {
    // NodeName identifies the node where the capacity is available.
    // A field selector can be used to list only NodeResources
    // objects with a certain node name.
    NodeName string

    // DriverName identifies the DRA driver providing the capacity information.
    // A field selector can be used to list only NodeResources
    // objects with a certain driver name.
    DriverName string

    NodeResourceModel // inline, field names must not conflict with the ones above
}

// NodeResourceModel must have one and only one field set.
type NodeResourceModel struct {
    Partitionable *PartitionableResources
}

type PartitionableResources struct {
    CommonAttributes  []AttributeGroup
    ResourceInstances []ResourceInstance
}

type AttributeGroup struct {
    Name       string
    Attributes []Attribute
}

type Attribute struct {
    Name string
    AttributeValue // inline, field names must not conflict
}

// AttributeValue must have one and only one field set.
type AttributeValue {
    Quantity    *resource.Quantity
    Bool        *bool
    Int         *int64
    IntSlice    *[]int64
    String      *string
    StringSlice *[]string
    Version     *SemVersion
}

type ResourceInstance struct {
    Name                 string
    CommonAttributesRefs []string
    Attributes           []Attribute
    Partitions           []ResourceInstanceGroup // If empty, it is not partitionable
}

type ResourceInstanceGroup struct {
    Name               string
    ResourceInstances  []ResourceInstance
}

// A wrapper around https://pkg.go.dev/github.com/blang/semver/v4#Version which
// is encoded as a string. During decoding, it validates that the string
// can be parsed using tolerant parsing (currently trims spaces, removes a "v" prefix,
// adds a 0 patch number to versions with only major and minor components specified,
// and removes leading 0s).
type SemVersion {
   semverv4.Version
}
```

All names must be DNS sub-domains. This excludes the "/" character, therefore
combining different names with that separator to form an ID is valid.


### ResourceClaimParameters

```go
type ResourceClaimParameters struct {
    metav1.TypeMeta `json:",inline"`
    // Standard object metadata
    metav1.ObjectMeta

    // If this object was created from some other resource, then this links
    // back to that resource. This field is used to find the in-tree representation
    // of the claim parameters when the parameter reference of the claim refers
    // to some unknown type.
    GeneratedFrom *ResourceClaimParametersReference

    // Shareable indicates whether the allocated claim is meant to be shareable
    // by multiple consumers at the same time.
    Shareable bool

    // Requests describes all resources that are needed for the allocated claim.
    // A single claim may use resources coming from different drivers.
    Requests []ResourceRequest
}

// ResourceRequest is a request for resources from one particular driver.
type ResourceRequest struct {
    DriverName string
    ResourceRequestModel // inline, fields must not conflict
}

// ResourceRequestModel must have one and only one field set.
type ResourceRequestModel struct {
    Partitionable *PartitionableResourceRequest
}

type PartitionableResourceRequests struct {
    Required []PartitionableResourceRequest
}

type PartitionableResourceRequest struct {
    // The name is not used right now. It might get used in the future to define
    // cross-request constraints.
    Name string

    // Selector is a CEL expression which must evaluate to true if a
    // resource instances is suitable. The language is as defined in
    // https://kubernetes.io/docs/reference/using-api/cel/
    //
    // In addition, for each supported attribute value type there
    // is a map that resolves to the corresponding value of the
    // instance under evaluation.
    Selector string
}
```

### ResourceClassParameters

```go
type ResourceClassParameters struct {
    metav1.TypeMeta `json:",inline"`
    // Standard object metadata
    metav1.ObjectMeta

    // If this object was created from some other resource, then this links
    // back to that resource. This field is used to find the in-tree representation
    // of the claim parameters when the parameter reference of the claim refers
    // to some unknown type.
    GeneratedFrom *ResourceClassParametersReference

    // Filters describes additional contraints that must be met when using the class.
    Filters []ResourceFilter
}

// ResourceFilter is a filter for resources from one particular driver.
type ResourceFilter struct {
    DriverName string
    ResourceFilterModel // inline, fields must not conflict
}

// ResourceFilterModel must have one and only one field set.
type ResourceFilterModel struct {
    Partitionable *PartitionableResourceFilter
}

type PartitionableResourceFilter struct {
    // Selector is a selector like the one in PartitionableResourceRequest. It must be
    // true for a resource instance to be suitable for a claim using the class.
    Selector string
}
```

### ResourceHandle extension

The ResourceHandle is embedded inside the claim status. When using semantic parameters,
a new field must get populated instead of the opaque driver data.

```go
type ResourceHandle struct {
    // DriverName specifies the name of the resource driver whose kubelet
    // plugin should be invoked to process this ResourceHandle's data once it
    // lands on a node. This may differ from the DriverName set in
    // ResourceClaimStatus this ResourceHandle is embedded in.
    DriverName string

    // Data contains the opaque data associated with this ResourceHandle. It is
    // set by the controller component of the resource driver whose name
    // matches the DriverName set in the ResourceClaimStatus this
    // ResourceHandle is embedded in. It is set at allocation time and is
    // intended for processing by the kubelet plugin whose name matches
    // the DriverName set in this ResourceHandle.
    //
    // The maximum size of this field is 16KiB. This may get increased in the
    // future, but not reduced.
    // +optional
    Data string

    // If SemanticData is set, then it needs to be used instead of Data.
    SemanticData *SemanticResourceHandle
}

type SemanticResourceHandle struct {
    // VendorClassParameters are the parameters from the time that the claim
    // was allocated. They can be arbitrary setup parameters that are
    // ignore by the counter controller.
    VendorClassParameters *runtime.RawExtension

    // VendorClaimParameters are the parameters from the time that the claim was
    // allocated. Some of the fields were used by the counter controller to
    // allocated resources, others can be arbitrary setup parameters.
    VendorClaimParameters *runtime.RawExtension

   	// NodeName is the name of the node providing the necessary resources.
    // This mirrors the AllocationResult.AvailableOnNodes with a simpler
    // type.
    //
    // The driver name is the one stored in ResourceHandle.
    NodeName string

    AllocationResultModel // inline, fields must not conflict
}

// AllocationResultModel must have one and only one field set.
type AllocationResultModel struct {
    Partitionable *PartitionableAllocationResult
}

type PartitionableAllocationResult struct {
    Instances []AllocatedResourceInstance
}

type AllocatedResourceInstance struct {
   ID string // A concatenation with / of the individual names.
}
```

### Implementation of semantic models

In the Go types above, all structs starting with `Partitionable` are part of that
semantic model. In practice, organizing those inside their own Go package and
then importing that package in the definition of the resource.k8s.io API will
result in a cleaner separation at the source code level. It has no impact on
the resulting Kubernetes API.

### Scheduling + Allocation

The dynamic resource scheduler plugin handles the common fields of
NodeResourceSlice, ResourceClaimParameters and SemanticResourceHandle. For the
semantic model fields it calls out to code that is associated with the
corresponding model.

During filtering it is decided which nodes have the necessary resources. If a
node is found, the scheduler plugin updates the resource claim status as part
of goroutine which handles pod binding.

Like a normal DRA driver controller, the scheduler also sets a finalizer to
ensure that users cannnot accidentally delete the allocated claim while a pod
is about to start which depends on it. That finalizer is
"semantic.dra.k8s.io/delete-protection".

### Deallocation

Deallocation is handled by kube-controller-manager when its claim controller
observes that a claim is no longer in use *and* the claim has the special
"semantic.dra.k8s.io/delete-protection" finalizer. This finalizer tells the
controller that it may clear the allocation result directly instead of setting
the `DeletionRequested` field, which is what it normally would do.

Updating the claim during deallocation will be observed by kube-scheduler and
tells it that it can use the capacity set aside for the claim
again. kube-controller-manager itself doesn't need to support specific semantic
models.

### Immediate allocation

Because there is no separate controller anymore, claims with immediate
allocation will only get allocated once there is a pod which needs them. The
remaining semantic difference compared to delayed allocation is that claims
with immediate allocation remain allocated when no longer in use.

### Simulation with CA

**Note**: this section outlines one potential solution for Cluster Autoscaler.
During code review it will be decided whether the actual implement is done like
this. Other autoscalers will have to implement a similar logic if they don't
use the in-tree dynamic resourceclaim scheduler plugin.

The usual call sequence of a scheduler plugin when used in the scheduler is
at program startup:
- instantiate plugin
- EventsToRegister

For each new pod:
- PreEnqueue

For each pod that is ready to be scheduled, one pod at a time:
- PreFilter, Filter, etc.


CA works a bit differently. It identifies all pending pods,
takes a snapshot of the current cluster state, and then simulates the effect
of scheduling those pods with additional nodes added to the cluster. To
determine whether a pod fits into one of these simulated nodes, it
uses the same PreFilter and Filter plugins as the scheduler. Other extension
points (Reserve, Bind) are not used. Plugins which modify the cluster state
therefore need a different way of recording the result of scheduling
a pod onto a node.

This is done through a new `ClusterAutoScalerPlugin` interface defined in
`k/k/pkg/scheduler/framework` that is implemented by the dynamic resource
plugin:

```go
type ClusterAutoScalerPlugin interface {
    Plugin
    // StartSimulation is called when the cluster autoscaler begins
    // a simulation.
    StartSimulation(ctx context.Context, state *CycleState) *Status

    // SimulateAddNode is called when the cluster autoscaler adds a new fictional node.
    // It's the job of the autoscaler to provide the NodeResourceSlices for
    // that node. The plugin adapts its internal model of available resources
    // accordingly.
    SimulateAddNode(ctx context.Context, state *CycleState, nodeInfo *NodeInfo, resources []resource.NodeResourceSlice) *Status

    // SimulateRemoveNode is called when the cluster autoscaler removes a real or fictional node.
    // The plugin adapts its internal model of available resources accordingly.
    // Pods using resources on the node must be evicted via SimulateEvictPod first,
    // otherwise this call fails.
    SimulateRemoveNode(ctx context.Context, state *CycleState, nodeInfo *NodeInfo) *Status

    // SimulateBindPod is called when the cluster autoscaler decided to schedule
    // a pod onto a certain node.
    SimulateBindPod(ctx context.Context, state *CycleState, pod *v1.Pod, nodeInfo *NodeInfo) *Status

    // SimulateEvictPod is called when the cluster autoscaler simulates removal
    // of a node. All claims used only by this pod should be considered deallocated,
    // to enable starting the same pod elsewhere.
    SimulateEvictPod(ctx context.Context, state *CycleState, pod *v1.Pod, nodeName string) *Status
}
```

Cluster autoscaler will at program startup:
- instantiate plugin, with real informer factory and no Kubernetes client
- start informers

At the start of a simulation:
- call `StartSimulation` with a clean cycle state

For each pending pod:
- call PreFilter and Filter with the same cycle state that
  was passed to `StartSimulation`
- call `SimulateBindPod` and/or `SimulateEvictPod` with the same cycle state that
  was passed to StartSimulation (i.e. *not* the one which was modified
  by PreFilter or Filter) to indicate that a pod is being scheduled onto a node
  respectively evicted as part of the simulation

The dynamic resource plugin will:
- Take a snapshot of all relevant cluster state as part of `StartSimulation`
  and store it in the cycle state. This signals to the other extension
  points that the plugin is being used as part of autoscaling.
- In `PreFilter` and `Filter` use the cluster snapshot to make decisions
  instead of the normal "live" cluster state.
- In `SimulateBindPod` and `SimulateEvictPod` update the snapshot in the cycle state:
  claims get allocated when first used resp. deallocated when they become unused.

Node addition and removal has the corresponding effect on the current state.


### Test Plan

[X] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

##### Prerequisite testing updates

None.

##### Unit tests

<!--
In principle every added code should have complete unit test coverage, so providing
the exact set of tests will not bring additional value.
However, if complete unit test coverage is not possible, explain the reason of it
together with explanation why this is acceptable.
-->

<!--
Additionally, for Alpha try to enumerate the core package you will be touching
to implement this enhancement and provide the current unit coverage for those
in the form of:
- <package>: <date> - <current test coverage>
The data can be easily read from:
https://testgrid.k8s.io/sig-testing-canaries#ci-kubernetes-coverage-unit

This can inform certain test coverage improvements that we want to do before
extending the production code to implement this enhancement.
-->

- `<package>`: `<date>` - `<test coverage>`

##### Integration tests

<!--
Integration tests are contained in k8s.io/kubernetes/test/integration.
Integration tests allow control of the configuration parameters used to start the binaries under test.
This is different from e2e tests which do not allow configuration of parameters.
Doing this allows testing non-default options and multiple different and potentially conflicting command line options.
-->

<!--
This question should be filled when targeting a release.
For Alpha, describe what tests will be added to ensure proper quality of the enhancement.

For Beta and GA, add links to added tests together with links to k8s-triage for those tests:
https://storage.googleapis.com/k8s-triage/index.html
-->

- <test>: <link to test coverage>

##### e2e tests

<!--
This question should be filled when targeting a release.
For Alpha, describe what tests will be added to ensure proper quality of the enhancement.

For Beta and GA, add links to added tests together with links to k8s-triage for those tests:
https://storage.googleapis.com/k8s-triage/index.html

We expect no non-infra related flakes in the last month as a GA graduation criteria.
-->

- <test>: <link to test coverage>

### Graduation Criteria

#### Alpha

- Feature implemented behind a feature flag
- Initial e2e tests completed and enabled

#### Beta

- Gather feedback from developers and surveys
- Fully implemented
- Additional tests are in Testgrid and linked in KEP

#### GA

- 3 examples of real-world usage
- Allowing time for feedback

[conformance tests]: https://git.k8s.io/community/contributors/devel/sig-architecture/conformance-tests.md

### Upgrade / Downgrade Strategy

Because of the strongly-typed versioning of resource attributes and allocation
results, the gRPC interface between kubelet and the DRA driver is tied to the
version of the supported semantic models. A DRA driver has to implement all
gRPC interfaces that might be used by older releases of kubelet. The same
applies when upgrading kubelet while the DRA driver remains at an older
version.

### Version Skew Strategy

Ideally, the latest release of a DRA driver should be used and it should
support a wide range of semantic type versions. Then problems due to version
skew are less likely to occur.

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

###### How can this feature be enabled / disabled in a live cluster?

- [X] Feature gate
  - Feature gate name: DynamicResourceAllocation
  - Components depending on the feature gate:
    - kube-apiserver
    - kubelet
    - kube-scheduler
    - kube-controller-manager

###### Does enabling the feature change any default behavior?

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

###### What happens if we reenable the feature if it was previously rolled back?

###### Are there any tests for feature enablement/disablement?

<!--
The e2e framework does not currently support enabling or disabling feature
gates. However, unit tests in each component dealing with managing data, created
with and without the feature, are necessary. At the very least, think about
conversion tests if API types are being modified.

Additionally, for features that are introducing a new API field, unit tests that
are exercising the `switch` of feature gate itself (what happens if I disable a
feature gate after having objects written with the new field) are also critical.
You can take a look at one potential example of such test in:
https://github.com/kubernetes/kubernetes/pull/97058/files#diff-7826f7adbc1996a05ab52e3f5f02429e94b68ce6bce0dc534d1be636154fded3R246-R282
-->

### Rollout, Upgrade and Rollback Planning

<!--
This section must be completed when targeting beta to a release.
-->

###### How can a rollout or rollback fail? Can it impact already running workloads?

<!--
Try to be as paranoid as possible - e.g., what if some components will restart
mid-rollout?

Be sure to consider highly-available clusters, where, for example,
feature flags will be enabled on some API servers and not others during the
rollout. Similarly, consider large clusters and how enablement/disablement
will rollout across nodes.
-->

###### What specific metrics should inform a rollback?

<!--
What signals should users be paying attention to when the feature is young
that might indicate a serious problem?
-->

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

<!--
Describe manual testing that was done and the outcomes.
Longer term, we may want to require automated upgrade/rollback tests, but we
are missing a bunch of machinery and tooling and can't do that now.
-->

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

<!--
Even if applying deprecation policies, they may still surprise some users.
-->

### Monitoring Requirements

<!--
This section must be completed when targeting beta to a release.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.
-->

###### How can an operator determine if the feature is in use by workloads?

<!--
Ideally, this should be a metric. Operations against the Kubernetes API (e.g.,
checking if there are objects with field X set) may be a last resort. Avoid
logs or events for this purpose.
-->

Metrics in kube-scheduler (names to be decided):
- number of classes using semantic parameters
- number of claim parameter types:
  - total
  - which have been resolved to a resource in the apiserver
- number of claims which currently are allocated with semantic parameters

If not all parameters types can be resolved to a resource, then the
installation of one or more DRA driver is incomplete (class references unknown
resources).

###### How can someone using this feature know that it is working for their instance?

<!--
For instance, if this is a pod-related feature, it should be possible to determine if the feature is functioning properly
for each individual pod.
Pick one more of these and delete the rest.
Please describe all items visible to end users below with sufficient detail so that they can verify correct enablement
and operation of this feature.
Recall that end users cannot usually observe component logs or access metrics.
-->

- [X] API .status
  - Other field: ".status.allocation" will be set for a claim using semantic parameters
    when needed by a pod.

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

<!--
This is your opportunity to define what "normal" quality of service looks like
for a feature.

It's impossible to provide comprehensive guidance, but at the very
high level (needs more precise definitions) those may be things like:
  - per-day percentage of API calls finishing with 5XX errors <= 1%
  - 99% percentile over day of absolute value from (job creation time minus expected
    job creation time) for cron job <= 10%
  - 99.9% of /health requests per day finish with 200 code

These goals will help you determine what you need to measure (SLIs) in the next
question.
-->

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

<!--
Pick one more of these and delete the rest.
-->

- [ ] Metrics
  - Metric name:
  - [Optional] Aggregation method:
  - Components exposing the metric:
- [ ] Other (treat as last resort)
  - Details:

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

<!--
Describe the metrics themselves and the reasons why they weren't added (e.g., cost,
implementation difficulties, etc.).
-->

### Dependencies

<!--
This section must be completed when targeting beta to a release.
-->

###### Does this feature depend on any specific services running in the cluster?

<!--
Think about both cluster-level services (e.g. metrics-server) as well
as node-level agents (e.g. specific version of CRI). Focus on external or
optional services that are needed. For example, if this feature depends on
a cloud provider API, or upon an external software-defined storage or network
control plane.

For each of these, fill in the following—thinking about running existing user workloads
and creating new ones, as well as about cluster-level services (e.g. DNS):
  - [Dependency name]
    - Usage description:
      - Impact of its outage on the feature:
      - Impact of its degraded performance or high-error rates on the feature:
-->

### Scalability

<!--
For alpha, this section is encouraged: reviewers should consider these questions
and attempt to answer them.

For beta, this section is required: reviewers must answer these questions.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.
-->

###### Will enabling / using this feature result in any new API calls?

<!--
Describe them, providing:
  - API call type (e.g. PATCH pods)
  - estimated throughput
  - originating component(s) (e.g. Kubelet, Feature-X-controller)
Focusing mostly on:
  - components listing and/or watching resources they didn't before
  - API calls that may be triggered by changes of some Kubernetes resources
    (e.g. update of object X triggers new updates of object Y)
  - periodic API calls to reconcile state (e.g. periodic fetching state,
    heartbeats, leader election, etc.)
-->

###### Will enabling / using this feature result in introducing new API types?

<!--
Describe them, providing:
  - API type
  - Supported number of objects per cluster
  - Supported number of objects per namespace (for namespace-scoped objects)
-->

###### Will enabling / using this feature result in any new calls to the cloud provider?

<!--
Describe them, providing:
  - Which API(s):
  - Estimated increase:
-->

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

<!--
Describe them, providing:
  - API type(s):
  - Estimated increase in size: (e.g., new annotation of size 32B)
  - Estimated amount of new objects: (e.g., new Object X for every existing Pod)
-->

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

<!--
Look at the [existing SLIs/SLOs].

Think about adding additional work or introducing new steps in between
(e.g. need to do X to start a container), etc. Please describe the details.

[existing SLIs/SLOs]: https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos
-->

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

<!--
Things to keep in mind include: additional in-memory state, additional
non-trivial computations, excessive access to disks (including increased log
volume), significant amount of data sent and/or received over network, etc.
This through this both in small and large cases, again with respect to the
[supported limits].

[supported limits]: https://git.k8s.io/community//sig-scalability/configs-and-limits/thresholds.md
-->

###### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

<!--
Focus not just on happy cases, but primarily on more pathological cases
(e.g. probes taking a minute instead of milliseconds, failed pods consuming resources, etc.).
If any of the resources can be exhausted, how this is mitigated with the existing limits
(e.g. pods per node) or new limits added by this KEP?

Are there any tests that were run/should be run to understand performance characteristics better
and validate the declared limits?
-->

### Troubleshooting

<!--
This section must be completed when targeting beta to a release.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.

The Troubleshooting section currently serves the `Playbook` role. We may consider
splitting it into a dedicated `Playbook` document (potentially with some monitoring
details). For now, we leave it here.
-->

###### How does this feature react if the API server and/or etcd is unavailable?

###### What are other known failure modes?

<!--
For each of them, fill in the following information by copying the below template:
  - [Failure mode brief description]
    - Detection: How can it be detected via metrics? Stated another way:
      how can an operator troubleshoot without logging into a master or worker node?
    - Mitigations: What can be done to stop the bleeding, especially for already
      running user workloads?
    - Diagnostics: What are the useful log messages and their required logging
      levels that could help debug the issue?
      Not required until feature graduated to beta.
    - Testing: Are there any tests for failure mode? If not, describe why.
-->

###### What steps should be taken if SLOs are not being met to determine the problem?

## Implementation History

<!--
Major milestones in the lifecycle of a KEP should be tracked in this section.
Major milestones might include:
- the `Summary` and `Motivation` sections being merged, signaling SIG acceptance
- the `Proposal` section being merged, signaling agreement on a proposed design
- the date implementation started
- the first Kubernetes release where an initial version of the KEP was available
- the version of Kubernetes where the KEP graduated to general availability
- when the KEP was retired or superseded
-->

## Drawbacks

DRA driver developers have to give up some flexibility with regards to
parameters. They have to learn and understand how semantic models
work to pick something which fits their needs.

## Alternatives

### Publishing resource information in node status

This is not desirable for several reasons (most important one first):
- All data from all drivers must be in a single object which is already
  large. It might become too large, with no chance of mitigating that by
  splitting up the information.
- All watchers of node objects get the information even if they don't need it.
- It puts complex alpha quality fields into a GA API.

### Injecting vendor logic into CA

With this KEP, vendor's use resource tracking and simulation that gets
implemented in core Kubernetes. Alternatively, CA could support vendor logic in
several different ways:

- Call out to a vendor server via some RPC mechanism (similar to scheduler
  webhooks). The risk here is that simulation becomes to slow. Configuration
  and security would be more complex.

- Load code provided by a vendor as [Web Assembly
  (WASM)](https://webassembly.org/) at runtime and invoke it similar to the
  builtin controllers in this KEP.  WASM is currently too experimental and has
  several drawbacks (single-threaded, all data must be
  serialized). https://github.com/kubernetes-sigs/kube-scheduler-wasm-extension
  is currently exploring usage of WASM for writing scheduler plugins. If this
  becomes feasible, then implementing a builtin controller which delegates its
  logic to vendor WASM code will be possible.

- Require that vendors provide Go code with their custom logic and rebuild CA
  with that code included. The scheduler could continue to use
  PodSchedulingContext, as long as the custom logic exactly matches what the
  DRA driver controller does. This approach is not an option when a pre-built
  CA binary has to be used and leads to challenges around maintenance and
  support of such a rebuilt CA binary. However, technically it [becomes
  possible](https://github.com/kubernetes-sigs/kube-scheduler-wasm-extension)
  with this KEP.
