---
title: Runtime Class
authors:
  - "@tallclair"
owning-sig: sig-node
participating-sigs:
  - sig-architecture
reviewers:
  - dchen1107
  - derekwaynecarr
  - yujuhong
approvers:
  - dchen1107
  - derekwaynecarr
creation-date: 2018-06-19
status: implementable
---

# Runtime Class

## Table of Contents

<!-- toc -->
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
  - [User Stories](#user-stories)
- [Proposal](#proposal)
  - [API](#api)
    - [Examples](#examples)
    - [Runtime Handler](#runtime-handler)
  - [Versioning, Updates, and Rollouts](#versioning-updates-and-rollouts)
  - [Implementation Details](#implementation-details)
    - [Monitoring](#monitoring)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Graduation Criteria](#graduation-criteria)
- [Implementation History](#implementation-history)
- [Appendix](#appendix)
  - [Proposed Future Enhancements](#proposed-future-enhancements)
  - [Examples of runtime variation](#examples-of-runtime-variation)
<!-- /toc -->

## Summary

`RuntimeClass` is a new cluster-scoped resource that surfaces container runtime properties to the
control plane. RuntimeClasses are assigned to pods through a `runtimeClass` field on the
`PodSpec`. This provides a new mechanism for supporting multiple runtimes in a cluster and/or node.

## Motivation

There is growing interest in using different runtimes within a cluster. [Sandboxes][] are the
primary motivator for this right now, with both Kata containers and gVisor looking to integrate with
Kubernetes. Other runtime models such as Windows containers or even remote runtimes will also
require support in the future. RuntimeClass provides a way to select between different runtimes
configured in the cluster and surface their properties (both to the cluster & the user).

In addition to selecting the runtime to use, supporting multiple runtimes raises other problems to
the control plane level, including: accounting for runtime overhead, scheduling to nodes that
support the runtime, and surfacing which optional features are supported by different
runtimes. Although these problems are not tackled by this initial proposal, RuntimeClass provides a
cluster-scoped resource tied to the runtime that can help solve these problems in a future update.

[Sandboxes]: https://docs.google.com/document/d/1QQ5u1RBDLXWvC8K3pscTtTRThsOeBSts_imYEoRyw8A/edit

### Goals

- Provide a mechanism for surfacing container runtime properties to the control plane
- Support multiple runtimes per-cluster, and provide a mechanism for users to select the desired
  runtime

### Non-Goals

- RuntimeClass is NOT RuntimeComponentConfig.
- RuntimeClass is NOT a general policy mechanism.
- RuntimeClass is NOT "NodeClass". Although different nodes may run different runtimes, in general
  RuntimeClass should not be a cross product of runtime properties and node properties.

### User Stories

- As a cluster operator, I want to provide multiple runtime options to support a wide variety of
  workloads. Examples include native linux containers, "sandboxed" containers, and windows
  containers.
- As a cluster operator, I want to provide stable rolling upgrades of runtimes. For
  example, rolling out an update with backwards incompatible changes or previously unsupported
  features.
- As an application developer, I want to select the runtime that best fits my workload.
- As an application developer, I don't want to study the nitty-gritty details of different runtime
  implementations, but rather choose from pre-configured classes.
- As an application developer, I want my application to be portable across clusters that use similar
  but different variants of a "class" of runtimes.

## Proposal

The initial design includes:

- `RuntimeClass` API resource definition
- `RuntimeClass` pod field for specifying the RuntimeClass the pod should be run with
- Kubelet implementation for fetching & interpreting the RuntimeClass
- CRI API & implementation for passing along the [RuntimeHandler](#runtime-handler).

### API

`RuntimeClass` is a new cluster-scoped resource in the `node.k8s.io` API group.

> _The `node.k8s.io` API group would eventually hold the Node resource when `core` is retired.
> Alternatives considered: `runtime.k8s.io`, `cluster.k8s.io`_

_(This is a simplified declaration, syntactic details will be covered in the API PR review)_

```go
type RuntimeClass struct {
    metav1.TypeMeta
    // ObjectMeta minimally includes the RuntimeClass name, which is used to reference the class.
    // Namespace should be left blank.
    metav1.ObjectMeta

    Spec RuntimeClassSpec
}

type RuntimeClassSpec struct {
    // RuntimeHandler specifies the underlying runtime the CRI calls to handle pod and/or container
    // creation. The possible values are specific to a given configuration & CRI implementation.
    // The empty string is equivalent to the default behavior.
    // +optional
    RuntimeHandler string
}
```

The runtime is selected by the pod by specifying the RuntimeClass in the PodSpec. Once the pod is
scheduled, the RuntimeClass cannot be changed.

```go
type PodSpec struct {
    ...
    // RuntimeClassName refers to a RuntimeClass object with the same name,
    // which should be used to run this pod.
    // +optional
    RuntimeClassName string
    ...
}
```

An unspecified `nil` or empty `""` RuntimeClassName is equivalent to the backwards-compatible
default behavior as if the RuntimeClass feature is disabled.

#### Examples

Suppose we operate a cluster that lets users choose between native runc containers, and gvisor and
kata-container sandboxes. We might create the following runtime classes:

```yaml
kind: RuntimeClass
apiVersion: node.k8s.io/v1alpha1
metadata:
    name: native  # equivalent to 'legacy' for now
spec:
    runtimeHandler: runc
---
kind: RuntimeClass
apiVersion: node.k8s.io/v1alpha1
metadata:
    name: gvisor
spec:
    runtimeHandler: gvisor
----
kind: RuntimeClass
apiVersion: node.k8s.io/v1alpha1
metadata:
    name: kata-containers
spec:
    runtimeHandler: kata-containers
----
# provides the default sandbox runtime when users don't care about which they're getting.
kind: RuntimeClass
apiVersion: node.k8s.io/v1alpha1
metadata:
  name: sandboxed
spec:
  runtimeHandler: gvisor
```

Then when a user creates a workload, they can choose the desired runtime class to use (or not, if
they want the default).

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: sandboxed-nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sandboxed-nginx
  template:
    metadata:
      labels:
        app: sandboxed-nginx
    spec:
      runtimeClassName: sandboxed   #   <----  Reference the desired RuntimeClass
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
          protocol: TCP
```

#### Runtime Handler

The `RuntimeHandler` is passed to the CRI as part of the `RunPodSandboxRequest`:

```proto
message RunPodSandboxRequest {
    // Configuration for creating a PodSandbox.
    PodSandboxConfig config = 1;
    // Named runtime configuration to use for this PodSandbox.
    string RuntimeHandler = 2;
}
```

The RuntimeHandler is provided as a mechanism for CRI implementations to select between different
predetermined configurations. The initial use case is replacing the experimental pod annotations
currently used for selecting a sandboxed runtime by various CRI implementations:

| CRI Runtime | Pod Annotation                                              |
| ------------|-------------------------------------------------------------|
| CRIO        | io.kubernetes.cri-o.TrustedSandbox: "false"                 |
| containerd  | io.kubernetes.cri.untrusted-workload: "true"                |
| frakti      | runtime.frakti.alpha.kubernetes.io/OSContainer: "true"<br>runtime.frakti.alpha.kubernetes.io/Unikernel: "true" |
| windows     | experimental.windows.kubernetes.io/isolation-type: "hyperv" |

These implementations could stick with scheme ("trusted" and "untrusted"), but the preferred
approach is a non-binary one wherein arbitrary handlers can be configured with a name that can be
matched against the specified RuntimeHandler. For example, containerd might have a configuration
corresponding to a "kata-runtime" handler:

```
[plugins.cri.containerd.kata-runtime]
    runtime_type = "io.containerd.runtime.v1.linux"
    runtime_engine = "/opt/kata/bin/kata-runtime"
    runtime_root = ""
```

This non-binary approach is more flexible: it can still map to a binary RuntimeClass selection
(e.g. `sandboxed` or `untrusted` RuntimeClasses), but can also support multiple parallel sandbox
types (e.g. `kata-containers` or `gvisor` RuntimeClasses).

### Versioning, Updates, and Rollouts

Runtimes are expected to be managed by the cluster administrator (or provisioner). In most cases,
runtime upgrades (and downgrades) should be handled by the administrator, without requiring any
interaction from the user. In these cases, the runtimes can be treated the same way we treat other
node components such as the Kubelet, node OS, and CRI runtime. In other words, the upgrade process
means rolling out nodes with the updated runtime, and gradually draining and removing old nodes from
the pool. For more details, see [Maintenance on a
Node](https://kubernetes.io/docs/tasks/administer-cluster/cluster-management/#maintenance-on-a-node).

If the upgraded runtime includes new features that users wish to take advantage of immediately, then
node labels can be used to select nodes supporting the updated runtime. In the uncommon scenario
where substantial changes to the runtime are made and application changes may be required, we
recommend that the updated runtime be treated as a _new_ runtime, with a separate RuntimeClass
(e.g. `sandboxed-v2`). This approach has the advantage of native support for rolling updates through
the same mechanisms as any other application update, so the updated applications can be carefully
rolled out to the new runtime.

Runtime upgrades will benefit from better scheduling support, which is a feature we plan to add in a
future release.

### Implementation Details

The Kubelet uses an Informer to keep a local cache of all RuntimeClass objects. When a new pod is
added, the Kubelet resolves the Pod's RuntimeClass against the local RuntimeClass cache.  Once
resolved, the RuntimeHandler field is passed to the CRI as part of the
[`RunPodSandboxRequest`][runpodsandbox]. At that point, the interpretation of the RuntimeHandler is
left to the CRI implementation, but it should be cached if needed for subsequent calls.

If the RuntimeClass cannot be resolved (e.g. doesn't exist) at Pod creation, then the request will
be rejected in admission (controller to be detailed in a following update). If the RuntimeClass
cannot be resolved by the Kubelet when `RunPodSandbox` should be called, then the Kubelet will fail
the Pod. The admission check on a replica recreation will prevent the scheduler from thrashing. If
the `RuntimeHandler` is not recognized by the CRI implementation, then `RunPodSandbox` will return
an error.

[runpodsandbox]: https://github.com/kubernetes/kubernetes/blob/b05a61e299777c2030fbcf27a396aff21b35f01b/pkg/kubelet/apis/cri/runtime/v1alpha2/api.proto#L344

#### Monitoring

The first round of monitoring implementation for `RuntimeClass` covers the
following two areas and is finished (tracked in
[#73058](https://github.com/kubernetes/kubernetes/issues/73058)):

- `how robust is every runtime?` A new metric
  [RunPodSandboxErrors](https://github.com/kubernetes/kubernetes/blob/596a48dd64bcaa01c1d2515dc79a558a4466d463/pkg/kubelet/metrics/metrics.go#L351)
  is added to track the RunPodSandbox operation errors, broken down by
  RuntimeClass.
- `how expensive is every runtime in terms of latency?` A new metric
  [RunPodSandboxDuration](https://github.com/kubernetes/kubernetes/blob/596a48dd64bcaa01c1d2515dc79a558a4466d463/pkg/kubelet/metrics/metrics.go#L341)
  is added to track the duration of RunPodSandbox operations, broken down by
  RuntimeClass.

The following monitoring areas will be skipped for now, but may be considered
after the RuntimeClass scheduling is implemented:

- how many runtimes does a cluster support?
- how many scheduling failures were caused by unsupported runtimes or insufficient
  resources of a certain runtime?

Currently, we assume that all the nodes in a cluster are homogeneous. After
heterogeneous clusters are implemented, we may need to monitor how many runtimes
a node supports.

### Risks and Mitigations

**Scope creep.** RuntimeClass has a fairly broad charter, but it should not become a default
dumping ground for every new feature exposed by the node. For each feature, careful consideration
should be made about whether it belongs on the Pod, Node, RuntimeClass, or some other resource. The
[non-goals](#non-goals) should be kept in mind when considering RuntimeClass features.

**Becoming a general policy mechanism.** RuntimeClass should not be used a replacement for
PodSecurityPolicy. The use cases for defining multiple RuntimeClasses for the same underlying
runtime implementation should be extremely limited (generally only around updates & rollouts). To
enforce this, no authorization or restrictions are placed directly on RuntimeClass use; in order to
restrict a user to a specific RuntimeClass, you must use another policy mechanism such as
PodSecurityPolicy.

**Pushing complexity to the user.** RuntimeClass is a new resource in order to hide the complexity
of runtime configuration from most users (aside from the cluster admin or provisioner). However, we
are still side-stepping the issue of precisely defining specific types of runtimes like
"Sandboxed". However, it is still up for debate whether precisely defining such runtime categories
is even possible. RuntimeClass allows us to decouple this specification from the implementation, but
it is still something I hope we can address in a future iteration through the concept of pre-defined
or "conformant" RuntimeClasses.

**Non-portability.** We are already in a world of non-portability for many features (see [examples
of runtime variation](#examples-of-runtime-variation). Future improvements to RuntimeClass can help
address this issue by formally declaring supported features, or matching the runtime that supports a
given workload automitaclly. Another issue is that pods need to refer to a RuntimeClass by name,
which may not be defined in every cluster. This is something that can be addressed through
pre-defined runtime classes (see previous risk), and/or by "fitting" pod requirements to compatible
RuntimeClasses.

## Graduation Criteria

Alpha:

- [x] Everything described in the current proposal:
  - [x] Introduce the RuntimeClass API resource
  - [x] Add a RuntimeClassName field to the PodSpec
  - [x] Add a RuntimeHandler field to the CRI `RunPodSandboxRequest`
  - [x] Lookup the RuntimeClass for pods & plumb through the RuntimeHandler in the Kubelet (feature
    gated)
- [x] RuntimeClass support in at least one CRI runtime & dockershim
  - [x] Runtime Handlers can be statically configured by the runtime, and referenced via RuntimeClass
  - [x] An error is reported when the handler or is unknown or unsupported
- [x] Testing
  - [x] Kubernetes E2E tests (only validating single runtime handler cases)

Beta:

- [x] Several major runtimes support RuntimeClass, and the current [untrusted annotations](#runtime-handler) are
  deprecated.
  - [x] [containerd](https://github.com/containerd/cri/pull/891)
  - [x] [CRI-O](https://github.com/kubernetes-sigs/cri-o/pull/1847)
  - [x] [dockershim](https://github.com/kubernetes/kubernetes/pull/67909)
- [ ] Comprehensive test coverage
  - [ ] [CRI validation tests][cri-validation]
  - [ ] RuntimeClasses are configured in the E2E environment with test coverage of a non-default
    RuntimeClass
- [x] Comprehensive coverage of RuntimeClass metrics. [#73058](http://issue.k8s.io/73058)
- [x] The update & upgrade story is revisited, and a longer-term approach is implemented as necessary.

[cri-validation]: https://github.com/kubernetes-sigs/cri-tools/blob/master/docs/validation.md

## Implementation History

- 2018-09-27: RuntimeClass released as alpha with Kubernetes v1.12
- 2018-06-11: SIG-Node decision to move forward with proposal
- 2018-06-19: Initial KEP published.

## Appendix

### Proposed Future Enhancements

The following ideas may be explored in a future iteration:

- Surfacing support for optional features by runtimes, and surfacing errors caused by
  incompatible features & runtimes earlier.
- Automatic runtime or feature discovery - initially RuntimeClasses are manually defined (by the
  cluster admin or provider), and are asserted to be an accurate representation of the runtime.
- Scheduling in heterogeneous clusters - it is possible to operate a heterogeneous cluster
  (different runtime configurations on different nodes) through scheduling primitives like
  `NodeAffinity` and `Taints+Tolerations`, but the user is responsible for setting these up and
  automatic runtime-aware scheduling is out-of-scope.
- Define standardized or conformant runtime classes - although I would like to declare some
  predefined RuntimeClasses with specific properties, doing so is out-of-scope for this initial KEP.
- [Pod Overhead][] - Although RuntimeClass is likely to be the configuration mechanism of choice,
  the details of how pod resource overhead will be implemented is out of scope for this KEP.
- Provide a mechanism to dynamically register or provision additional runtimes.
- Requiring specific RuntimeClasses according to policy. This should be addressed by other
  cluster-level policy mechanisms, such as PodSecurityPolicy.
- "Fitting" a RuntimeClass to pod requirements - In other words, specifying runtime properties and
  letting the system match an appropriate RuntimeClass, rather than explicitly assigning a
  RuntimeClass by name. This approach can increase portability, but can be added seamlessly in a
  future iteration.
- The cluster admin can choose which RuntimeClass is the default in a cluster.

[Pod Overhead]: https://docs.google.com/document/d/1EJKT4gyl58-kzt2bnwkv08MIUZ6lkDpXcxkHqCvvAp4/edit

### Examples of runtime variation

- Linux Security Module (LSM) choice - Kubernetes supports both AppArmor & SELinux options on pods,
  but those are mutually exclusive, and support of either is not required by the runtime. The
  default configuration is also not well defined.
- Seccomp-bpf - Kubernetes has alpha support for specifying a seccomp profile, but the default is
  defined by the runtime, and support is not guaranteed.
- Windows containers - isolation features are very OS-specific, and most of the current features are
  limited to linux. As we build out Windows container support, we'll need to add windows-specific
  features as well.
- Host namespaces (Network,PID,IPC) may not be supported by virtualization-based runtimes
  (e.g. Kata-containers & gVisor).
- Per-pod and Per-container resource overhead varies by runtime.
- Device support (e.g. GPUs) varies wildly by runtime & nodes.
- Supported volume types varies by node - it remains TBD whether this information belongs in
  RuntimeClass.
- The list of default capabilities is defined in Docker, but not Kubernetes. Future runtimes may
  have differing defaults, or support a subset of capabilities.
- `Privileged` mode is not well defined, and thus may have differing implementations.
- Support for resource over-commit and dynamic resource sizing (e.g. Burstable vs Guaranteed
  workloads)
