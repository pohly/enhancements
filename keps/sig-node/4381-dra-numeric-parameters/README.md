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

# KEP-4381: Numeric Parameters for Dynamic Resource Allocation

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Publishing node resources](#publishing-node-resources)
  - [Using semantic parameters as claim parameters](#using-semantic-parameters-as-claim-parameters)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
- [Design Details](#design-details)
  - [ResourceClass extension](#resourceclass-extension)
  - [NodeResources](#noderesources)
  - [Builtin controllers](#builtin-controllers)
      - [Initialization](#initialization)
      - [Allocation](#allocation)
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
  - [Pre-defined parameter types](#pre-defined-parameter-types)
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

Dynamic Resource Allocation uses parameters for resources that are completely
opaque to core Kubernetes. They get interpreted by the DRA driver's controller
(for allocating claims) and the DRA drivers' kubelet plugin (for configuring
the resource on a node). During scheduling of a pod, kube-scheduler and the DRA
driver controller(s) that handle claims used by the pod communicate through the
apiserver by updating the PodSchedulingContext object, ultimately leading to
allocation of all pending claims and the pod being scheduled onto a node.

This approach poses a problem for [Cluster
Autoscaler](https://github.com/kubernetes/autoscaler) (CA) because it cannot
simulate the effect of allocating or deallocating claims. "Semantic parameters"
is an extension of DRA that addresses this by making claim parameters less
opaque. "Semantic models" define how resource capacity per node can be published
and the claim parameters that allocate some of that capacity for a claim.

This KEP describes the core changes needed for this. Specific semantic models
get defined in separate KEPs. The types defined by these models become
part of the resource.k8s.io API. Therefore developers who want to extend
a scheduler with some additional model have to modify the apiserver.

When opting into using semantic parameters for their DRA driver, driver
developers no longer need to provide a control plane controller which
handles claim allocation. Instead, they need to provide a CRD controller
which translates from their custom claim parameters CRD into the parameter
object understood by Kubernetes.

Scheduling performance is expected to become better compared to using the
PodSchedulingContext. However, supporting CA is the main reason for this KEP.

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

- Keep the core DRA functionality from [KEP
  #3063](https://github.com/kubernetes/enhancements/issues/3063) working as
  defined there for Kubernetes 1.29. Cluster autoscaling is not always needed
  and in-tree numeric model(s) are not expected to cover all use case, so the
  flexibility offered by opaque parameters remains relevant.

### Non-Goals

- Support semantic class parameters. They remain opaque and can be used to
  define configuration of resources, but must not affect where a claim can be
  allocated.

- Define exactly one semantic model that covers all current and future resources.
  There cannot be a "one size fits all" solution.

## Proposal

Each node publishes information about the nearly-opaque resources it currently
has. Each driver manages one or more resource instance per node. An instance
could be a single discreet piece of hardware like a PCI card or a pool of such
cards. Each instance has certain attributes. These can be qualitative (labels
or fields for comparison) and quantitative (capacity that gets reduced when
allocated for a claim). These attributes are described by the "attributes type"
of the semantic models. Each instance can combine attributes from different
semantic models. This way, each model can focus on one aspect, like counting
items or selection via labels.

Workloads make claims to request resources with claim parameters that use CRDs
provided together with a DRA driver. As soon as such a custom CR is created, a
CRD controller of the DRA driver must convert it into a in-tree claim parameter
object that uses the "parameter types" defined by the semantic models.

The scheduler receives both the node resource information and the claim
parameters. It uses the in-tree claim parameters to identify and allocate
suitable resources. It captures all information that the DRA driver kubelet
plugin will need in the "allocation types" of the semantic models and stores
that in the claim status. Then it schedules the pod.

The DRA driver kubelet plugin receives that information when the pod is about to
start. It checks that the hardware is still available and healthy, then
configures it for the pod as specified in the allocation types.

### Publishing node resources

A semantic model defines an attributes type. This is a Go type that gets
embedded inside the resource.k8s.io API, not a separate API kind. The kubelet
queries the registered DRA driver kubelet plugins for resource information
through a new gRPC call. A streaming gRPC API allows drivers to update the
capacity information.

For example, a simple counter could be described in a `counter` Go package as:
```
type Attributes struct {
    // Some meta information just for logging and debugging. Not used for counting.
	metav1.ObjectMeta

	Count int64
}
```

The kubelet then publishes this information in a new NodeResources object on
behalf of the driver. This implies that kubelet, DRA driver, and apiserver must
all support the same semantic models and the same versions of them. It might be
possible to support version skew (= keeping kubelet at an older version than
the control plane and the DRA drivers) in the future, but currently this is out
of scope.

### Using semantic parameters as claim parameters

The semantic model also defines a type for claim parameters. For the example
above, that could be:
```
type Parameters struct {
	Required int64
}
```

The `Parameters.Required` value gets compared against `Attributes.Count`.

Suppose a vendor has a CRD which allows requesting a certain number of `frobnicators`:

```
kind: ClaimParameters
apiVersion: dra.frobincatorInc.example.com/v1alpha1
metadata:
  name: my-claim-parameters
  uid: abcd
  resourceVersion: 123
spec:
  requiredFrobincators: 100
```

The vendor CRD controller needs to sync this CRs into one in-tree object:
```
kind: ResourceClaimParameters
apiVersion: resource.k8s.io/v1alpha2
metadata:
  name: frobnicator-claim-parameters
  namespace: default
generatedFrom:
  kind: ClaimParameters
  apiGroup: dra.frobincatorInc.example.com
  name: my-claim-parameters
  resourceVersion: 123
  uid: abcd
parameters:
  - counter:
      required: 100
```

### Notes/Constraints/Caveats

For this proposal to work, kube-scheduler must have permission to read, list
and watch the custom claim parameter resources. This is necessary because it is
not known in advance which CRDs will be used by DRA drivers. When deploying a
driver, suitable role bindings for
[`system:kube-scheduler`](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#core-component-roles)
have to be added which grant read, watch and list permissions for the vendor
CRD.

The scheduler does not need to interpret the custom data in those CRDs. It
still needs to read them for two purposes:
- Ensure that the in-tree claim parameters match the vendor claim parameters
  reference by the claim and are not out-dated.
- Include a snapshot of claim and class parameters in the allocation result,
  to ensure that configuration parameters are available when the DRA driver
  needs them.

The types that the scheduler needs to watch may change at runtime as classes
get added or removed. To avoid repeated GET operations for those CRDs, dynamic
informers will have to be used that get started and stopped as needed.

## Design Details

### ResourceClass extension

A new, optional field in ResourceClass enables semantic parameters for claims
using this class:

```
type ResourceClass struct {
	...

	// If (and only if) allocation of claims using this class is handled
	// via numeric parameters, then ParameterTypes describes those
	// parameters. A driver might support multiple different CRDs for
	// its claims.
	ParameterTypes []SemanticParameterType
}

// SemanticParameterType describes one resource that may be used for
// claim parameters of a class which uses semantic parameters.
type SemanticParameterType struct {
	// APIGroup is the group for the resource being referenced. It is
	// empty for the core API. This matches the group in the APIVersion
	// that is used when creating the resources.
	// +optional
	APIGroup string
	// Kind is the type of resource being referenced. This is the same
	// value as in the parameter object's metadata.
	Kind string

	// Shareable is copied into the AllocationResult when a claim
	// gets allocated.
	Shareable bool
}
```

### NodeResources

For each node, one or more NodeResources objects gets created:

```
// NodeResources provides information about available
// resources on individual nodes.
type NodeResources struct {
	metav1.TypeMeta
	// Standard object metadata
	// +optional
	metav1.ObjectMeta

	// NodeName identifies the node where the capacity is available.
	// A field selector can be used to list only NodeResources
	// objects with a certain node name.
	NodeName string

	// DriverName identifies the DRA driver providing the capacity information.
	// A field selector can be used to list only NodeResources
	// objects with a certain driver name.
	DriverName string

	// Instances describes discrete resources managed by the driver on
	// the node. It is possible to create more than one NodeResources
	// object for the same node and driver if this array becomes to large
	// for a single object.
	Instances []NodeResourceInstance
}

// NodeResourceInstance describes one discrete resource instance.
type NodeResourceInstance struct {
	// ID is chosen by the driver to distinguish between different resource
	// instances. It only needs to be unique for the driver and node.  In
	// other words, the tuple of NodeName, DriverName, InstanceID needs to
	// be unique in the cluster.
	ID string

    // Attributes describe the instance.
	Attributes []InstanceAttributes
}

// InstanceAttributes must have one and only one field set.
type InstanceAttributes struct {
	<one pointer per semantic model>
}
```

If a driver needs to reduce resource capacity, then there is a risk that a claim gets
allocated using that capacity while the kubelet is updating the
NodeResources object. The implementations of semantic models must handle
scenarios where more resources are allocated than available. The DRA driver
kubelet plugin must double-check that the allocated resources are still
available when NodePrepareResource is called. If not, the pod cannot
start until the resource comes back.

### Builtin controllers

`k8s.io/dynamic-resource-allocation/builtincontroller` defines Go interfaces
that need to be implemented by a semantic model to handle tracking of resource
usage and allocation/deallocation. The package also has a global registry of
builtin controllers.

##### Initialization

All registered builtin controllers get activated when kube-scheduler or CA
start. At this point they start building an internal model of which resources
are available (based on those entries in NodeResources objects that they
are responsible for) and how those resources are already used (based on the
status of allocated claims).

##### Allocation

When the dynamic resource plugin in kube-scheduler encounters a claim which,
according to the class, uses semantic parameters, it expands the CEL template
and decodes into the capacity type, using `Kind` and `APIVersion` to determine
that type. Then it looks up which builtin controller handles that claim
parameter type.

That controller then is asked to provide suitable nodes and, once a node has
been identified, to allocate the claim.

The controller itself must not modify the claim. Instead, it just returns the
AllocationResult and marks resources as being in use. It is the responsibility
of kube-scheduler to update the actual claim status during the bind operation
for a pod. If that fails for whatever reason, the controller is informed and
needs to revert the resource usage. As with opaque parameters, the persisted
status of the claim is the source of truth for which resources are used by
which claim.  This allows restarting kube-scheduler at any time without
resource leaks.

In CA, the dynamic resource plugin doesn't actually modify the claim object.
It just updates its snapshot of the claim to consider that state change for
future pod scheduling cycles.

As with opaque parameters, once a claim is allocated, its class and parameter
objects may be deleted and the claim has to remain usable. To achieve this,
during allocation the entire class and claim parameter objects must be stored
in the allocation result together with enough information for the DRA kubelet
plugin to determine which resources were reserved for the claim. This is again
specific to the semantic model, so each semantic model has to define an
allocation result type that gets populated by the builtin controller and
consumed by the DRA kubelet plugin.

As usual, allocated claims must have a finalizer set that prevents users from
accidentally deleting a claim that is or might be in use. The special
"semantic.dra.k8s.io/delete-protection" finalizer is used. This is relevant for
deallocation.

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

```
type ClusterAutoScalerPlugin interface {
	Plugin
	// StartSimulation is called when the cluster autoscaler begins
	// a simulation.
	StartSimulation(ctx context.Context, state *CycleState) *Status
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

Adding or removing nodes is not part of the interface. Instead, new unknown
nodes need to be detected in `Filter`:
- When creating a fictional node during simulation from an existing node, CA
  needs to add a special annotation `autoscaling.k8s.io/node-resource-capacity`
  with the name of the original node. Then the NodeResources object
  under that name, i.e. from the original node, gets used also for the
  fictional node. The assumption is that the new node will have the same
  hardware and configuration and thus the same capacity as the old one.
- When scaling from zero, the cluster administrator or cloud provider must
  create one such object per node group with a name that won't conflict with
  real nodes in the cluster and then ensure that fictional nodes created for
  that node group have the `autoscaling.k8s.io/node-resource-capacity` with
  that name.
- To the scheduler plugin, both cases look the same and thus no special cases
  are needed.

Removal of nodes doesn't need to be detected by the plugin. They simply won't
be used anymore and eventually will get discarded when simulation ends.

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
support a wide range of numeric type versions. Then problems due to version
skew are less likely to occur.

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

###### How can this feature be enabled / disabled in a live cluster?

- [X] Feature gate
  - Feature gate name: DRANumericParameters
  - Components depending on the feature gate:
    - kube-apiserver
    - kubelet

###### Does enabling the feature change any default behavior?

No.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Disabling the feature will remove the NodeResources type and prevent
adding the `NumericParameters` field to new classes. Existing classes with this
field set will continue to have it. The effect will be that no claims using
numeric parameters get allocated because there is no controller to handle
them. Pods depending on such claims cannot be scheduled.

If there are allocated claims, then kubelet and kube-controller-manager will
still try to handle them to ease the migration towards a state where the
feature is not used anymore in the cluster.

###### What happens if we reenable the feature if it was previously rolled back?

Claim allocation will resume.

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
- number of classes using numeric parameters
- number of parameter types:
  - total
  - which have been resolved to a resource
- number of claims which currently are allocated with numeric parameters

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
  - Other field: ".status.allocation" will be set for a claim using numeric parameters
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
parameters. They have to learn and understand how the different numeric models
work to pick one which fits their needs.

## Alternatives

### Pre-defined parameter types

Instead of relying on vendor CRDs for the parameter type, Kubernetes also could
define and serve one such type per numeric model. The downside is that this
type then cannot have vendor-specific validation because it will be shared by
different vendors. Such a type could have a field for configuration parameters,
for example with a string->string map as type, but that is less user-friendly
than a CRD.

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
