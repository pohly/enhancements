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
  - [User Stories](#user-stories)
    - [Persistent Memory as DRAM replacement for memcached](#persistent-memory-as-dram-replacement-for-memcached)
    - [Local LVM storage as scratch space](#local-lvm-storage-as-scratch-space)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [PVC meta data](#pvc-meta-data)
  - [Preventing accidental collision with existing PVCs](#preventing-accidental-collision-with-existing-pvcs)
  - [Feature gate](#feature-gate)
  - [Modifying volumes](#modifying-volumes)
  - [Late binding](#late-binding)
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

This mimics a `PersistentVolumeClaimVolumeSource`, except that it
contains a `PersistentVolumeClaim` instead of referencing one by name.

A new controller in `kube-controller-manager` is responsible for
creating new PVCs for each such inline volume. It does that:
- with a deterministic name that is a concatenation of pod name and
  the `Volume.Name` of the volume,
- in the namespace of the pod,
- with the pod as owner.

Kubernetes already prevents adding, removing or updating volumes in a
pod and the ownership relationship ensures that volumes get deleted
when the pod gets deleted, so the new controller only needs to take
care of creating missing PVCs.

When the [volume scheduling
library](https://github.com/kubernetes/kubernetes/tree/v1.18.0/pkg/controller/volume/scheduling)
inside the kube-scheduler or kubelet encounter such a volume inside a
pod, they determine the name of the PVC and proceed as they currently
do for a `PersistentVolumeClaimVolumeSource`.

### User Stories

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

### PVC meta data

The namespace is the same as for the pod and the name is the
concatenation of pod name and the `Volume.Name`. This is guaranteed to
be unique for the pod and each volume in that pod. Care must be taken
by the user to not exceed the length limit for object names, otherwise
volumes cannot be created.

Other meta data is copied from the `VolumeClaimTemplate.ObjectMeta`.

### Preventing accidental collision with existing PVCs

The controller will only create missing PVCs. It does not need to
delete (handled by garbage collection) nor update (`PodSpec.Volumes`
is immutable) existing PVCs. Therefore there is no risk that the
controller will accidentally modify unrelated PVCs.

The volume scheduling library and kubelet must check that a PVC is
owned by the pod before using it. Otherwise they must ignored it. This
ensures that a volume is not used accidentally for a pod in case of a
name conflict.

To surface such a name conflict to the user, the controller can also
check for PVCs that aren't owned by the pod in its reconciliation loop
and emit a pod event that describes the problem.

### Feature gate

The `GenericInlineVolumes` feature gate controls whether:
- the new controller is active in the `kube-controller-manager`,
- new pods can be created with a `InlineVolumeSource`,
- kubelet and scheduler accept such volumes.

Existing pods with such a volumes will not be scheduled respectively
started when the feature gate is off.

### Modifying volumes

Once the PVC for an inline volume has been created, it can be updated
directly like other PVCs. The controller will not interfere with that
because it never updates PVCs. This can be used to control features
like volume resizing.

### Late binding

Ideally, provisioning should use late binding (aka
`WaitForFirstConsumer`). The initial implementation assumes that the
user is taking care of that by selecting that provisioning mode in the
storage class for the PVC.

Later, `kube-scheduler` and `external-provisioner` can be changed to
automatically enable late binding for PVCs which are owned by a pod.

### Test Plan

- Unit tests will be added for the API change.
- Unit tests will cover the functionality of the controller, similar
  to
  https://github.com/kubernetes/kubernetes/blob/v1.18.2/pkg/controller/volume/persistentvolume/pv_controller_test.go.
- Unit tests need to cover the positive case (feature enabled) as
  well as negative case (feature disabled or feature used incorrectly).
- A new [storage test
  suite](https://github.com/kubernetes/kubernetes/blob/2b2cf8df303affd916bbeda8c2184b023f6ee53c/test/e2e/storage/testsuites/base.go#L84-L94)
  will be added which tests inline volume creation, usage and deletion
  in combination with all drivers that support dynamic volume
  provisioning.

### Graduation Criteria

#### Alpha -> Beta Graduation

- Gather feedback from developers and surveys
- Volume name collisions emitted as pod events
- Tests are in Testgrid and linked in KEP

#### Beta -> GA Graduation

- 3 examples of real world usage
- Downgrade tests and scalability tests
- Allowing time for feedback

### Upgrade / Downgrade Strategy

When downgrading to a cluster which does not support generic inline
volumes (either by disabling the feature flag or an actual version
downgrade), pods using generic inline volumes will no longer be
scheduled or started.

### Version Skew Strategy

As with downgrades, having some of the relevant components at an older
version will prevent pods from starting.

## Production Readiness Review Questionnaire

### Feature enablement and rollback

* **How can this feature be enabled / disabled in a live cluster?**
  - [X] Feature gate
    - Feature gate name: GenericInlineVolumes
    - Components depending on the feature gate:
      - kube-apiserver
      - kube-controller-manager
      - kubelet

* **Does enabling the feature change any default behavior?**
  If users are allowed to create pods but not PVCs, then generic inline volumes
  grants them permission to create PVCs indirectly. Cluster admins must take
  that into account in their permission model.

* **Can the feature be disabled once it has been enabled (i.e. can we rollback
  the enablement)?**
  Yes, by disabling the feature gates. Existing pods will continue to run and
  volumes will be deleted once they stop. Existing pods with generic inline
  volumes that haven't started yet will not be able to start up anymore, either
  because they do not get scheduled because their volumes are missing or because
  kubelet does not know what to do with the volume.

* **What happens if we reenable the feature if it was previously rolled back?**
  Pods that got stuck will work again.

* **Are there any tests for feature enablement/disablement?**
  Yes, unit tests for the apiserver and kubelet.

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
  No.

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
