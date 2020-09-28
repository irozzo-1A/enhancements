---
title: Out-of-Tree Credential Providers
authors:
  - "@irozzo-1A"
  - "@youssefazrak"
owning-sig: sig-cloud-provider
reviewers:
  - "@cheftako"
approvers: TBD
editor: TBD
creation-date: 2020-09-28
last-updated: 2020-09-28
status: draft
---

# Extending Apiserver Network Proxy to handle traffic originated from Node network

## Table of Contents

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
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
  - [Graduation Criteria](#graduation-criteria)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Implementation History](#implementation-history)
- [Infrastructure Needed](#infrastructure-needed)
<!-- /toc -->

## Release Signoff Checklist

- [ ] kubernetes/enhancements issue in release milestone, which links to KEP (this should be a link to the KEP location in kubernetes/enhancements, not the initial KEP PR)
- [ ] KEP approvers have set the KEP status to `implementable`
- [ ] Design details are appropriately documented
- [ ] Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
- [ ] Graduation criteria is in place
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

## Summary

The goal of this proposal is to allow traffic to flow from the cluster network (agent side) to the control plane (Apiserver network proxy).

## Motivation

API server network proxy has been originally introduced to allow running the cluster nodes on distinct isolated networks with respect to the one hosting the control plane components. This provides a way to handle traffic originating from the Kube API Server and going to the cluster networks. When using this setup, there are no other options than to directly expose the KAS to the Internet or setting up a VPN to handle traffic originated from the cluster nodes (i.e. Kubelet, pods). This could lead to security risks or complicated setups. 
Extending the API server network proxy to allow handling traffic in both directions seems to be the most consistent approach to address this issue. 
Moreover, this will provide the possibility of routing agents’ traffic to their dedicated KAS based on SNI, enabling the option of load balancing traffic directed to different clusters with a single load balancer.

### Goals

* Handle requests from the Cluster to the Control Plane. Enable communication from the Node Network to the Master Network without having to expose the KAS to the internet.

### Non-Goals

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

Currently the Konnectivity Server is accepting requests from the KAS either with the gRPC or the HTTP Connect interfaces and is taking care of forwarding the traffic to the Konnectivity Agent using the previously established connections (initiated by the Agents). 

In order to enable traffic from Kubelets and Pods running on Master Network, the Konnectivity Agents have to expose an endpoint that will be listening on a specific port for each of the destinations on the Control Network. As opposed to the traffic flowing from the Control Network to the Cluster Network, the Konnectivity Agent should act transparently: From a Kubelets or Pods standpoint, the Konnectivity Agent should be the final destination instead of acting as a proxy. The reason why we do not intend to use the same strategy used in the other direction is that we do not have control over the clients using the Kubernetes default service to communicate with the KAS.

### Traffic Flow

```
client =TCP=> (:6443) agent GRPC=> server =TCP=> KAS(:6443)
                         |            ^
                         |   Tunnel   |
                         +------------+
```

The agent listens for TCP connections at a specific port for each configured destination. When a connection request is received by the Konnectivity Agent the following happens:
1. A GRPC DIAL_REQ message is sent to the Konnectivity server containing the destination address associated with the current port.
2. Upon reception of the DIAL_REQ the Konnectivity Server opens a TCP connection with the destination host/port and replies to the Konnectivity Agent with a GRPC DIAL_RES message.
3. At this point the tunnel is established and data is piped through it, carried over GRPC DATA packets.

An executable capable of providing container registry credentials will be pre-installed on each node so that it exists when kubelet starts running. This binary will be executed by the kubelet to obtain container registry credentials in a format compatible with container runtimes. Credential responses may be cached within the kubelet.

### Agent additional flags

* `--target=dst_host_ip:dst_port:local_port`: We can have multiple of those in order to support multiple destinations on the Master Network.
Dst_host_ip: end target IP (apiserver or something else)
e.g. --target=apiserver.svc.cluster.local:6443:6443

* `--bind-address=ip`: Local IP address where the Konnectivity Agent will listen for incoming requests. It will be bound to a dummy IP interface with IP x.y.z.v defined by the user. Must be used with the previous one to enable incoming requests. If not, and for backward compatibility, only the traffic initiated from the Control Plane will be allowed.

### Handling the Traffic from the Pods to the Agent

As mentioned above, pods make use of  the Kubernetes default service to reach the KAS. To keep things transparent from a Pod perspective, they will hit the Konnectivity Agent using the Kubernetes default service. The endpoint will be the Konnectivity Agent instead of the KAS. 
The configuration part of the Kubernetes default service will be done using the Apiserver flag `--advertise-address ip` on the Control Plane side.
`--advertise-address ip` should match the `--bind-address ip` of the Konnectivity Agent described above. 

### Handling the Traffic from the Kubelet to the Agent

Kubelet does not use the Kubernetes default service to reach the KAS. Instead it relies on a bootstrap kubeconfig file that is used to connect to the KAS. It then generates a proper kubeconfig file that will be using the same URL.
Instead of specifying the KAS FQDN/address in the bootstrap kubeconfig file, we will be using the local IP address of the Konnectivity agent (`--bind-address ip`).

### Deployment Model

The agent can be run as static pod or systemd units. In any case the agent should be started to give access to the KAS to the kubelet first and to the hosted pods later. This means that using DaemonSets or Deployments is not an option in this setup.

### Listening Interface for Konnectivity agent

As mentioned before, we will be using the Kubernetes default service to route traffic to the Agent. The service in itself has a couple of limitations: it can’t be used as a type externalName, thus preventing the use of dynamic IPs (say, Pods’ networking range). But also, some general services limitations apply: endpoints can’t use the link-local range and the localhost range. This means that we are left with the 3 private IPs ranges (10.0.0.0/8, 172.16.0.0/12 and 192.168.0.0/16). 

When using a StaticPod, the Agent Pod will create a dummy interface (using an init container?), bind to the IP given in the argument `--bind-address` and will start to listen on this IP:local_port (local_port is defined with the `--target` argument).
With systemd the principle is the same, but the creation and configuration of the interface could be handled with `ExecStartPre` commands.

### Authentication

Konnectivity agent currently support mTLS or Token based authentication. Note that API objects such as Secrets cannot be accessed either when a StaticPod or Systemd service deployment strategy is used. The authentication secret should be made available to the agent through a different channel (e.g. provisioned in the worker node file-system).

### Risks and Mitigations

TODO

## Design Details

TODO

### Test Plan

TODO

### Graduation Criteria

TODO

### Upgrade / Downgrade Strategy

TODO

### Version Skew Strategy

TODO

## Implementation History

TODO

## Infrastructure Needed
