<!--
**Note:** When your KEP is complete, all of these comment blocks should be removed.

To get started with this template:

- [ ] **Pick a hosting SIG.**
  Make sure that the problem space is something the SIG is interested in taking
  up.  KEPs should not be checked in without a sponsoring SIG.
- [ ] **Create an issue in kubernetes/enhancements**
  When filing an enhancement tracking issue, please ensure to complete all
  fields in that template.  One of the fields asks for a link to the KEP.  You
  can leave that blank until this KEP is filed, and then go back to the
  enhancement and add the link.
- [ ] **Make a copy of this template directory.**
  Copy this template into the owning SIG's directory and name it
  `NNNN-short-descriptive-title`, where `NNNN` is the issue number (with no
  leading-zero padding) assigned to your enhancement above.
- [ ] **Fill out as much of the kep.yaml file as you can.**
  At minimum, you should fill in the "title", "authors", "owning-sig",
  "status", and date-related fields.
- [ ] **Fill out this file as best you can.**
  At minimum, you should fill in the "Summary", and "Motivation" sections.
  These should be easy if you've preflighted the idea of the KEP with the
  appropriate SIG(s).
- [ ] **Create a PR for this KEP.**
  Assign it to people in the SIG that are sponsoring this process.
- [ ] **Merge early and iterate.**
  Avoid getting hung up on specific details and instead aim to get the goals of
  the KEP clarified and merged quickly.  The best way to do this is to just
  start with the high-level sections and fill out details incrementally in
  subsequent PRs.

Just because a KEP is merged does not mean it is complete or approved.  Any KEP
marked as a `provisional` is a working document and subject to change.  You can
denote sections that are under active debate as follows:

```
<<[UNRESOLVED optional short context or usernames ]>>
Stuff that is being argued.
<<[/UNRESOLVED]>>
```

When editing KEPS, aim for tightly-scoped, single-topic PRs to keep discussions
focused.  If you disagree with what is already in a document, open a new PR
with suggested changes.

One KEP corresponds to one "feature" or "enhancement", for its whole lifecycle.
You do not need a new KEP to move from beta to GA, for example.  If there are
new details that belong in the KEP, edit the KEP.  Once a feature has become
"implemented", major changes should get new KEPs.

The canonical place for the latest set of instructions (and the likely source
of this file) is [here](/keps/NNNN-kep-template/README.md).

**Note:** Any PRs to move a KEP to `implementable` or significant changes once
it is marked `implementable` must be approved by each of the KEP approvers.
If any of those approvers is no longer appropriate than changes to that list
should be approved by the remaining approvers and/or the owning SIG (or
SIG Architecture for cross cutting KEPs).
-->
# KEP-1698: generic inline volumes

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories (optional)](#user-stories-optional)
    - [Persistent Memory as DRAM replacement for memcached](#persistent-memory-as-dram-replacement-for-memcached)
    - [Local LVM storage as scratch space](#local-lvm-storage-as-scratch-space)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
  - [Graduation Criteria](#graduation-criteria)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
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

- [ ] Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] KEP approvers have approved the KEP status as `implementable`
- [ ] Design details are appropriately documented
- [ ] Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
- [ ] Graduation criteria is in place
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

This KEP proposes an alternative mechanism for specifying volumes
inside a pod spec. Those inline volumes then get converted to normal
volume claim objects for provisioning, so storage drivers do not need
to be modified.

Because this is expected to be slower than [CSI ephemeral inline
volumes](https://github.com/kubernetes/enhancements/issues/596), both
approaches will need to be supported.


## Motivation

The current CSI ephemeral inline volume feature has demonstrated that
ephemeral inline volumes are a useful concept, for example to provide
additional scratch space for a pod. Several CSI drivers support it
now, most of them in addition to normal volumes (see
https://kubernetes-csi.github.io/docs/drivers.html).

However, the original design was intentionally focused on
light-weight, local volumes. It is not a good fit for volumes provided
by a more traditional storage system because:
- The normal API for selecting volume parameters (like size and
  storage class) is not supported.
- Integration into storage capacity aware pod scheduling is
  challenging and would depend on extending the CSI ephemeral inline
  volume API.
- CSI drivers need to be adapted and have to take over some of the
  work normally done by Kubernetes, like tracking of orphaned volumes.


### Goals

- Volumes can be specified inside the pod spec ("inline").
- Volumes are created for specific pods and deleted after the pod
  terminates ("ephemeral").
- A normal, unmodified storage driver can be selected via a storage class.
- The volume will be created using the normal storage provisioning
  mechanism, without having to modify the driver or its deployment.
- Storage capacity tracking can be enabled also for such inline
  volumes.

### Non-Goals

- This will not replace CSI ephemeral inline volumes.
- Inline volumes could also be kept alive after their pod terminates,
  but that is only useful if some higher-level logic then takes care
  of deletion. For now this is out of scope.

## Proposal

A new volume source will be introduced:

```
typedef InlineVolumeSource struct {
    VolumeClaimTemplate PersistentVolumeClaim
    ReadOnly            bool
}
```

When the [volume scheduling
library](https://github.com/kubernetes/kubernetes/tree/v1.18.0/pkg/controller/volume/scheduling)
inside the kube-scheduler encounters such a volume inside a pod, it
creates a PVC inside the same namespace as the pod. The name of that
PVC is a concatenation of pod name and the `Volume.Name` of the volume
and thus unique for the pod and each volume in that pod. Care must be
taken by the user to not exceed the length limit for object names,
otherwise volumes cannot be created.

The `VolumeClaimTemplate.ObjectMeta.Name` and
`VolumeClaimTemplate.ObjectMeta.Namespace` are ignored. Labels from
`VolumeClaimTemplate.ObjectMeta.Labels` are copied.


<<[UNRESOLVED @pohly ]>>

Using a full-blown `PersistentVolumeClaim` instead of just
`PersistentVolumeClaimSpec` might be overkill. The approach above
follows the example set by statefulset.

<<[/UNRESOLVED]>>

When creating that PVC, the pod is set as owner. That ensures that the
volume will be deleted automatically when the pod gets deleted.

When a PVC with that name already exists, it is only used if it has
the pod as owner. Otherwise it is left unmodified and the pod cannot
start because the volume cannot be provisioned. This covers the case
where there is some accidental conflict with some unrelated PVC.

When kubelet is asked to start a pod with such a volume, it uses the
same code as for `PersistentVolumeClaimVolumeSource`, with the only
exception that the claim name is computed dynamically.

<<[UNRESOLVED @pohly ]>>

Which part of a PersistentVolumeClaimSpec are immutable? Do we need to
support updating the `VolumeClaimTemplate` by copying the mutable
fields into the PVC?

<<[/UNRESOLVED]>>

<<[UNRESOLVED @pohly ]>>

Ideally, the storage class should use late binding. Do we want to
leave that to the user (no further changes needed) or change the late
binding check in kube-scheduler and external-provisioner so that PVCs
with a pod as owner are always treated as "late binding"?

<<[/UNRESOLVED]>>

### User Stories (optional)

#### Persistent Memory as DRAM replacement for memcached

Recent releases of memcached added [support for using Persistent
Memory](https://memcached.org/blog/persistent-memory/) (PMEM) instead
of normal DRAM. When deploying memcached through one of the app
controllers, `InlineVolumeSource` makes it possible to request a volume
of a certain size from a CSI driver like
[PMEM-CSI](https://github.com/intel/pmem-csi).

#### Local LVM storage as scratch space

Applications working with data sets that exceed the RAM size can
request local storage with performance characteristics or size that is
not met by the normal Kubernetes `EmptyDir` volumes. For example,
[TopoLVM](https://github.com/cybozu-go/topolvm) was written for that
purpose.

### Risks and Mitigations

Enabling this feature allows users to create PVCs indirectly if they can
create pods, even if they do not have permission to create them
directly. Cluster administrators must be made aware of this. If this
does not fit their security model, they can disable the feature
through the feature gate that will be added for the feature.

Alternatively, a label on a namespace could be used to disable the
feature just for that namespace.

## Design Details

<!--
This section should contain enough information that the specifics of your
change are understandable.  This may include API specs (though not always
required) or even code snippets.  If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.
-->

### Test Plan

<!--
**Note:** *Not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?

No need to outline all of the test cases, just the general strategy.  Anything
that would count as tricky in the implementation and anything particularly
challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations).  Please adhere to the [Kubernetes testing guidelines][testing-guidelines]
when drafting this test plan.

[testing-guidelines]: https://git.k8s.io/community/contributors/devel/sig-testing/testing.md
-->

### Graduation Criteria

<!--
**Note:** *Not required until targeted at a release.*

Define graduation milestones.

These may be defined in terms of API maturity, or as something else. The KEP
should keep this high-level with a focus on what signals will be looked at to
determine graduation.

Consider the following in developing the graduation criteria for this enhancement:
- [Maturity levels (`alpha`, `beta`, `stable`)][maturity-levels]
- [Deprecation policy][deprecation-policy]

Clearly define what graduation means by either linking to the [API doc
definition](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-versioning),
or by redefining what graduation means.

In general, we try to use the same stages (alpha, beta, GA), regardless how the
functionality is accessed.

[maturity-levels]: https://git.k8s.io/community/contributors/devel/sig-architecture/api_changes.md#alpha-beta-and-stable-versions
[deprecation-policy]: https://kubernetes.io/docs/reference/using-api/deprecation-policy/

Below are some examples to consider, in addition to the aforementioned [maturity levels][maturity-levels].

#### Alpha -> Beta Graduation

- Gather feedback from developers and surveys
- Complete features A, B, C
- Tests are in Testgrid and linked in KEP

#### Beta -> GA Graduation

- N examples of real world usage
- N installs
- More rigorous forms of testing e.g., downgrade tests and scalability tests
- Allowing time for feedback

**Note:** Generally we also wait at least 2 releases between beta and
GA/stable, since there's no opportunity for user feedback, or even bug reports,
in back-to-back releases.

#### Removing a deprecated flag

- Announce deprecation and support policy of the existing flag
- Two versions passed since introducing the functionality which deprecates the flag (to address version skew)
- Address feedback on usage/changed behavior, provided on GitHub issues
- Deprecate the flag

**For non-optional features moving to GA, the graduation criteria must include [conformance tests].**

[conformance tests]: https://git.k8s.io/community/contributors/devel/sig-architecture/conformance-tests.md
-->

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

## Implementation History

<!--
Major milestones in the life cycle of a KEP should be tracked in this section.
Major milestones might include
- the `Summary` and `Motivation` sections being merged signaling SIG acceptance
- the `Proposal` section being merged signaling agreement on a proposed design
- the date implementation started
- the first Kubernetes release where an initial version of the KEP was available
- the version of Kubernetes where the KEP graduated to general availability
- when the KEP was retired or superseded
-->

## Drawbacks

<!--
Why should this KEP _not_ be implemented?
-->

## Alternatives

The alternative to creating the PVC is to modify components that
currently interact with a PVC such that they can work with stand-alone
PVC objects (like they do now) and with the embedded PVCs inside
pods. The downside is that this then no longer works with unmodified
CSI deployments because extensions in the CSI external-provisioner
will be needed.

Some of the current usages of PVC will become a bit unusual (status
update inside pod spec) or tricky (references from PV to PVC).

The advantage is that no automatically created PVCs are
needed. However, other controllers also create user-visible objects
(statefulset -> pod and PVC, deployment -> replicaset -> pod), so this
concept is familiar to users.
