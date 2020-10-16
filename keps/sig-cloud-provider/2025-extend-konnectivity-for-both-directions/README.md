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
# KEP-2025: Extending Apiserver Network Proxy to handle traffic originated from Node network

<!--
This is the title of your KEP. Keep it short, simple, and descriptive. A good
title can help communicate what the KEP is and should be considered as part of
any review.
-->

<!--
A table of contents is helpful for quickly jumping to sections of a KEP and for
highlighting any additional information provided beyond the standard KEP
template.

Ensure the TOC is wrapped with
  <code>&lt;!-- toc --&rt;&lt;!-- /toc --&rt;</code>
tags, and then generate with `hack/update-toc.sh`.
-->

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Definitions](#definitions)
- [Proposal](#proposal)
  - [Traffic flow](#traffic-flow)
  - [Agent additional flags](#agent-additional-flags)
  - [Handling the Traffic from the Pods to the Agent](#handling-the-traffic-from-the-pods-to-the-agent)
  - [Handling the Traffic from the Kubelet to the Agent](#handling-the-traffic-from-the-kubelet-to-the-agent)
  - [Deployment Model](#deployment-model)
  - [Listening Interface for Konnectivity agent](#listening-interface-for-konnectivity-agent)
  - [Authentication](#authentication)
  - [Risks and Mitigations](#risks-and-mitigations)
    - [Allow list](#allow-list)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
  - [Graduation Criteria](#graduation-criteria)
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
- [Infrastructure Needed](#infrastructure-needed)
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
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
- [ ] (R) Graduation criteria is in place
- [ ] (R) Production readiness review completed
- [ ] Production readiness review approved
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

<!--
This section is incredibly important for producing high-quality, user-focused
documentation such as release notes or a development roadmap. It should be
possible to collect this information before implementation begins, in order to
avoid requiring implementors to split their attention between writing release
notes and implementing the feature itself. KEP editors and SIG Docs
should help to ensure that the tone and content of the `Summary` section is
useful for a wide audience.

A good summary is probably at least a paragraph in length.

Both in this section and below, follow the guidelines of the [documentation
style guide]. In particular, wrap lines to a reasonable length, to make it
easier for reviewers to cite specific portions, and to minimize diff churn on
updates.

[documentation style guide]: https://github.com/kubernetes/community/blob/master/contributors/guide/style-guide.md
-->

The goal of this proposal is to allow traffic to flow from the Node Network to the Master Network.

## Motivation

<!--
This section is for explicitly listing the motivation, goals, and non-goals of
this KEP.  Describe why the change is important and the benefits to users. The
motivation section can optionally provide links to [experience reports] to
demonstrate the interest in a KEP within the wider Kubernetes community.

[experience reports]: https://github.com/golang/go/wiki/ExperienceReports
-->

API server network proxy has been originally introduced to allow running the cluster nodes on distinct isolated networks with respect to the one hosting the control plane components. This provides a way to handle traffic originating from the Kube API Server and going to the node networks. When using this setup, there are no other options than to directly expose the KAS to the Internet or setting up a VPN to handle traffic originated from the cluster nodes (i.e. Kubelet, pods).
Exposing Konnectivity Proxy Server only would add an additional layer of security, given that the tunnels with proxy agents are secured with mTLS or Token authentication. This would protect it, for instance, from KAS misconfigurations or vulnerabilities that could expose sensitive information and/or access to unauthenticated users.
Furthermore there is no standard solution for this kind of setup. It is possible to rely on VPNs for example to achieve a similar goal, but this requires specific implementations. What we propose here is to build on top of what we have, and having a consistent approach for master to node and node to master communications.
Extending the API server network proxy to allow handling traffic in both directions seems to be the most consistent approach to address this issue. 
Moreover, this will provide the possibility of routing agents’ traffic to their dedicated KAS based on SNI, enabling the option of load balancing traffic directed to different clusters with a single load balancer.

### Goals

<!--
List the specific goals of the KEP. What is it trying to achieve? How will we
know that this has succeeded?
-->
* Handle requests from the nodes to the control plane. Enable communication from the Node Network to the Master Network without having to expose the KAS to the Node Network.

### Non-Goals

<!--
What is out of scope for this KEP? Listing non-goals helps to focus discussion
and make progress.
-->
* Define a mechanism for exchanging authentication information used for establishing the secure channels between agents and server (e.g. certificates, tokens). 
* Define a solution involving less than one agent per node. 

## Definitions

* **Master Network** An IP reachable network space which contains the master components, such as Kubernetes API Server, Connectivity Proxy and ETCD server.
* **Node Network** An IP reachable network space which contains all the clusters Nodes, for alpha. Worth noting that the Node Network may be completely disjoint from the Master network. It may have overlapping IP addresses to the Master Network or other means of network isolation. Direct IP routability between cluster and master networks should not be assumed. Later versions may relax the all node requirement to some.
* **KAS** Kubernetes API Server, responsible for serving the Kubernetes API to clients.
* **Konnectivity Server** The proxy server which runs in the master network. It has a secure channel established to the cluster network. It could work on either a HTTP Connect mechanism or gRPC. If the former it would expose a gRPC interface to KAS to provide connectivity service. If the latter it would use standard HTTP Connect. Formerly known as the Network Proxy Server.
* **Konnectivity Agent** A proxy agent which runs in the node network for establishing the tunnel. Formerly known as the Network Proxy Agent.
* **Flat Network** A network where traffic can be successfully routed using IP. Implies no overlapping (i.e. shared) IPs on the network.

## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation. What is the desired outcome and how do we measure success?.
The "Design Details" section below is for the real
nitty-gritty.
-->
Currently the Konnectivity Server is accepting requests from the KAS either with the gRPC or the HTTP Connect interfaces and is taking care of forwarding the traffic to the Konnectivity Agent using the previously established connections (initiated by the Agents). 

In order to enable traffic from Kubelets and Pods running on Master Network, the Konnectivity Agents have to expose an endpoint that will be listening on a specific port for each of the destinations on the Master Network. As opposed to the traffic flowing from the Master Network to the Node Network, the Konnectivity Agent should act transparently: From a Kubelets or Pods standpoint, the Konnectivity Agent should be the final destination instead of acting as a proxy. 

The reason why we do not intend to use the same strategy used in the other direction is that we do not have control over the clients using the Kubernetes default service to communicate with the KAS.

### Traffic Flow

```
client =TCP=> (:6443) agent GRPC=> server =TCP=> KAS(:6443)
                         |            ^
                         |   Tunnel   |
                         +------------+
```

The agent listens for TCP connections at a specific port for each configured destination. When a TCP connection is established between the client and the Konnectivity Agent, the following happens:
1. A GRPC DIAL_REQ message is sent to the Konnectivity server containing the destination address associated with the current port.
2. Upon reception of the DIAL_REQ the Konnectivity Server verifies if the destination is allowed (see [allow list](#allow-list) section). If not allowed a DIAL_RSP containing an error is sent back immediately to the agent that terminates the connection.
3. In case the destination is allowed Konnectivity Server opens a TCP connection with the destination host/port and replies to the Konnectivity Agent with a GRPC DIAL_RES message.
4. At this point the tunnel is established and data is piped through it, carried over GRPC DATA packets.

### Agent additional flags

* `--target=local_port:dst_host_ip:dst_port`:
    - local_port: The local port where the agent binds to.
    - dst_host_ip: Ip or hostname used by the KAS. In case of IPv6
    the address should be wrapped in squared brackets, consistently with [RFC3986](https://www.ietf.org/rfc/rfc3986.txt) notation.
    - dst_port: The remote port used by the KAS. 
e.g. --target=6443:apiserver.svc.cluster.local:6443
e.g. --target=6443:[2001:db8:1f70::999:de8:7648:6e8]:6443

* `--bind-address=ip`: Local IP address where the Konnectivity Agent will listen for incoming requests. It will be bound to a dummy IP interface with IP x.y.z.v defined by the user. Must be used with the previous one to enable incoming requests. If not, and for backward compatibility, only the traffic initiated from the Control Plane will be allowed.

### Handling the Traffic from the Pods to the Agent

As mentioned above, pods make use of  the Kubernetes default service to reach the KAS. To keep things transparent from a Pod perspective, they will hit the Konnectivity Agent using the Kubernetes default service. The endpoint will be the Konnectivity Agent instead of the KAS. 
The configuration part of the Kubernetes default service will be done using the Apiserver flag `--advertise-address ip` and `--secure-port port` on the Control Plane side.
The `advertise-address` and the `secure-port` should match respectively the `bind-address` and the `local_port` of the Konnectivity Agent described above. 

### Handling the Traffic from the Kubelet to the Agent

Kubelet does not use the Kubernetes default service to reach the KAS. Instead it relies on a bootstrap kubeconfig file that is used to connect to the KAS. It then generates a proper kubeconfig file that will be using the same URL.
Instead of specifying the KAS FQDN/address in the bootstrap kubeconfig file, we will be using the local IP address of the Konnectivity agent (`--bind-address ip`).

### Deployment Model

The agent can be run as static pod or systemd units. In any case the agent should be started to give access to the KAS to the kubelet first and to the hosted pods later. This means that using DaemonSets or Deployments is not an option in this setup, because the kubelet would not be able to get the pod manifests from the KAS.

### Listening Interface for Konnectivity agent

As mentioned before, we will be using the Kubernetes default service to route traffic to the Agent. The service in itself has a couple of limitations: it can’t be used as a type externalName, thus preventing the use of dynamic IPs (say, Pods’ networking range). But also, some general services limitations apply: endpoints can’t use the link-local range and the localhost range. This means that we are left with the 3 private IPs ranges (10.0.0.0/8, 172.16.0.0/12 and 192.168.0.0/16). 

When using a StaticPod, the Agent Pod will create a dummy interface (using an init container?), bind to the IP given in the argument `--bind-address` and will start to listen on this IP:local_port (local_port is defined with the `--target` argument).
With systemd the principle is the same, but the creation and configuration of the interface could be handled with `ExecStartPre` commands.

### Authentication between Konnectivity agent and server

Konnectivity agent currently support mTLS or Token based authentication. Note that API objects such as Secrets cannot be accessed either when a StaticPod or Systemd service deployment strategy is used. The authentication secret should be made available to the agent through a different channel (e.g. provisioned in the worker node file-system).

### Risks and Mitigations

<!--
What are the risks of this proposal, and how do we mitigate? Think broadly.
For example, consider both security and how this will impact the larger
Kubernetes ecosystem.

How will security be reviewed, and by whom?

How will UX be reviewed, and by whom?

Consider including folks who also work outside the SIG or subproject.
-->
#### Allow List

> A generic TCP tunnel is fraught with security risks. First, such
> authorization should be limited to a small number of known ports.
> The Upgrade: mechanism defined here only requires onward tunneling at
> port 80. Second, since tunneled data is opaque to the proxy, there
> are additional risks to tunneling to other well-known or reserved
> ports. A putative HTTP client CONNECTing to port 25 could relay spam
> via SMTP, for example.
> -- <cite>[rfc2817][http_upgrade_tls_rfc]</cite>

Similarly to CONNECT tunnel this feature opens the door to generic TCP tunnels
to arbitrary destinations.

A mechanism for restricting access to the possible destinations on the Master Network should be implemented on the Konnectivity server.
The list of allowed destinations will be provided with the following flag:

* `--allowed-destination=dst_host:dst_port`: We can have multiple of those in
    order to allow access to multiple destinations.

TODO(irozzo/yazrak): What happens if DIAL_REQ is received by the server for a destination
that is not in the allow list? Should we put in place a health-check mechanism
to make sure agent configuration is consistent with the allow list?

## Design Details

<!--
This section should contain enough information that the specifics of your
change are understandable. This may include API specs (though not always
required) or even code snippets. If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.
-->

### Test Plan

<!--
**Note:** *Not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?

No need to outline all of the test cases, just the general strategy. Anything
that would count as tricky in the implementation, and anything particularly
challenging to test, should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations). Please adhere to the [Kubernetes testing guidelines][testing-guidelines]
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
definition](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-versioning)
or by redefining what graduation means.

In general we try to use the same stages (alpha, beta, GA), regardless of how the
functionality is accessed.

[maturity-levels]: https://git.k8s.io/community/contributors/devel/sig-architecture/api_changes.md#alpha-beta-and-stable-versions
[deprecation-policy]: https://kubernetes.io/docs/reference/using-api/deprecation-policy/

Below are some examples to consider, in addition to the aforementioned [maturity levels][maturity-levels].

#### Alpha -> Beta Graduation

- Gather feedback from developers and surveys
- Complete features A, B, C
- Tests are in Testgrid and linked in KEP

#### Beta -> GA Graduation

- N examples of real-world usage
- N installs
- More rigorous forms of testing—e.g., downgrade tests and scalability tests
- Allowing time for feedback

**Note:** Generally we also wait at least two releases between beta and
GA/stable, because there's no opportunity for user feedback, or even bug reports,
in back-to-back releases.

#### Removing a Deprecated Flag

- Announce deprecation and support policy of the existing flag
- Two versions passed since introducing the functionality that deprecates the flag (to address version skew)
- Address feedback on usage/changed behavior, provided on GitHub issues
- Deprecate the flag

**For non-optional features moving to GA, the graduation criteria must include 
[conformance tests].**

[conformance tests]: https://git.k8s.io/community/contributors/devel/sig-architecture/conformance-tests.md
-->

### Upgrade / Downgrade Strategy

<!--
If applicable, how will the component be upgraded and downgraded? Make sure
this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this
enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade, in order to maintain previous behavior?
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade, in order to make use of the enhancement?
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

<!--

Production readiness reviews are intended to ensure that features merging into
Kubernetes are observable, scalable and supportable; can be safely operated in
production environments, and can be disabled or rolled back in the event they
cause increased failures in production. See more in the PRR KEP at
https://git.k8s.io/enhancements/keps/sig-architecture/1194-prod-readiness/README.md.

The production readiness review questionnaire must be completed and approved
for the KEP to move to `implementable` status and be included in the release.

In some cases, the questions below should also have answers in `kep.yaml`. This
is to enable automation to verify the presence of the review, and to reduce review
burden and latency.

The KEP must have a approver from the
[`prod-readiness-approvers`](http://git.k8s.io/enhancements/OWNERS_ALIASES)
team. Please reach out on the
[#prod-readiness](https://kubernetes.slack.com/archives/CPNHUMN74) channel if
you need any help or guidance.

-->

### Feature Enablement and Rollback

_This section must be completed when targeting alpha to a release._

* **How can this feature be enabled / disabled in a live cluster?**
  - [ ] Feature gate (also fill in values in `kep.yaml`)
    - Feature gate name:
    - Components depending on the feature gate:
  - [ ] Other
    - Describe the mechanism:
    - Will enabling / disabling the feature require downtime of the control
      plane?
    - Will enabling / disabling the feature require downtime or reprovisioning
      of a node? (Do not assume `Dynamic Kubelet Config` feature is enabled).

* **Does enabling the feature change any default behavior?**
  Any change of default behavior may be surprising to users or break existing
  automations, so be extremely careful here.

* **Can the feature be disabled once it has been enabled (i.e. can we roll back
  the enablement)?**
  Also set `disable-supported` to `true` or `false` in `kep.yaml`.
  Describe the consequences on existing workloads (e.g., if this is a runtime
  feature, can it break the existing applications?).

* **What happens if we reenable the feature if it was previously rolled back?**

* **Are there any tests for feature enablement/disablement?**
  The e2e framework does not currently support enabling or disabling feature
  gates. However, unit tests in each component dealing with managing data, created
  with and without the feature, are necessary. At the very least, think about
  conversion tests if API types are being modified.

### Rollout, Upgrade and Rollback Planning

_This section must be completed when targeting beta graduation to a release._

* **How can a rollout fail? Can it impact already running workloads?**
  Try to be as paranoid as possible - e.g., what if some components will restart
   mid-rollout?

* **What specific metrics should inform a rollback?**

* **Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?**
  Describe manual testing that was done and the outcomes.
  Longer term, we may want to require automated upgrade/rollback tests, but we
  are missing a bunch of machinery and tooling and can't do that now.

* **Is the rollout accompanied by any deprecations and/or removals of features, APIs, 
fields of API types, flags, etc.?**
  Even if applying deprecation policies, they may still surprise some users.

### Monitoring Requirements

_This section must be completed when targeting beta graduation to a release._

* **How can an operator determine if the feature is in use by workloads?**
  Ideally, this should be a metric. Operations against the Kubernetes API (e.g.,
  checking if there are objects with field X set) may be a last resort. Avoid
  logs or events for this purpose.

* **What are the SLIs (Service Level Indicators) an operator can use to determine 
the health of the service?**
  - [ ] Metrics
    - Metric name:
    - [Optional] Aggregation method:
    - Components exposing the metric:
  - [ ] Other (treat as last resort)
    - Details:

* **What are the reasonable SLOs (Service Level Objectives) for the above SLIs?**
  At a high level, this usually will be in the form of "high percentile of SLI
  per day <= X". It's impossible to provide comprehensive guidance, but at the very
  high level (needs more precise definitions) those may be things like:
  - per-day percentage of API calls finishing with 5XX errors <= 1%
  - 99% percentile over day of absolute value from (job creation time minus expected
    job creation time) for cron job <= 10%
  - 99,9% of /health requests per day finish with 200 code

* **Are there any missing metrics that would be useful to have to improve observability 
of this feature?**
  Describe the metrics themselves and the reasons why they weren't added (e.g., cost,
  implementation difficulties, etc.).

### Dependencies

_This section must be completed when targeting beta graduation to a release._

* **Does this feature depend on any specific services running in the cluster?**
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


### Scalability

_For alpha, this section is encouraged: reviewers should consider these questions
and attempt to answer them._

_For beta, this section is required: reviewers must answer these questions._

_For GA, this section is required: approvers should be able to confirm the
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
  Describe them, providing:
  - API type
  - Supported number of objects per cluster
  - Supported number of objects per namespace (for namespace-scoped objects)

* **Will enabling / using this feature result in any new calls to the cloud 
provider?**

* **Will enabling / using this feature result in increasing size or count of 
the existing API objects?**
  Describe them, providing:
  - API type(s):
  - Estimated increase in size: (e.g., new annotation of size 32B)
  - Estimated amount of new objects: (e.g., new Object X for every existing Pod)

* **Will enabling / using this feature result in increasing time taken by any 
operations covered by [existing SLIs/SLOs]?**
  Think about adding additional work or introducing new steps in between
  (e.g. need to do X to start a container), etc. Please describe the details.

* **Will enabling / using this feature result in non-negligible increase of 
resource usage (CPU, RAM, disk, IO, ...) in any components?**
  Things to keep in mind include: additional in-memory state, additional
  non-trivial computations, excessive access to disks (including increased log
  volume), significant amount of data sent and/or received over network, etc.
  This through this both in small and large cases, again with respect to the
  [supported limits].

### Troubleshooting

The Troubleshooting section currently serves the `Playbook` role. We may consider
splitting it into a dedicated `Playbook` document (potentially with some monitoring
details). For now, we leave it here.

_This section must be completed when targeting beta graduation to a release._

* **How does this feature react if the API server and/or etcd is unavailable?**

* **What are other known failure modes?**
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

* **What steps should be taken if SLOs are not being met to determine the problem?**

[supported limits]: https://git.k8s.io/community//sig-scalability/configs-and-limits/thresholds.md
[existing SLIs/SLOs]: https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos

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

<!--
Why should this KEP _not_ be implemented?
-->

## Alternatives

<!--
What other approaches did you consider, and why did you rule them out? These do
not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable.
-->

## Infrastructure Needed (Optional)

<!--
Use this section if you need things from the project/SIG. Examples include a
new subproject, repos requested, or GitHub details. Listing these here allows a
SIG to get the process for these resources started right away.
-->
