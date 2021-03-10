---
title: Opt-in Autoscaling from Zero
authors:
  - "@elmiko"
reviewers:
  - "@"
creation-date: 2021-03-10
last-updated: 2021-03-10
status: implementable
---

# Opt-in Autoscaling from Zero

## Table of Contents

- [Title](#title)
  - [Table of Contents](#table-of-contents)
  - [Glossary](#glossary)
  - [Summary](#summary)
  - [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals/Future Work](#non-goalsfuture-work)
  - [Proposal](#proposal)
    - [User Stories](#user-stories)
      - [Story 1](#story-1)
      - [Story 2](#story-2)
    - [Security Model](#security-model)
    - [Risks and Mitigations](#risks-and-mitigations)
  - [Alternatives](#alternatives)
  - [Upgrade Strategy](#upgrade-strategy)
  - [Additional Details](#additional-details)
    - [Test Plan](#test-plan-optional)
  - [Implementation History](#implementation-history)

## Glossary

* **Node Group** This term has special meaning within the cluster autoscaler, it refers to collections
  of nodes, and related physical hardware, that are organized within the autoscaler for scaling operations.
  These node groups do not have a direct relation to specific CRDs within Kubernetes, and may be handled
  differently by each autoscaler cloud implementation. In the case of Cluster API, node groups correspond
  directly to MachineSets and MachineDeployments that are marked for autoscaling.

Refer to the [Cluster API Book Glossary](https://cluster-api.sigs.k8s.io/reference/glossary.html).

## Summary

The [Kubernetes cluster autoscaler](https://github.com/kubernetes/autoscaler) currently supports
scaling on Cluster API deployed clusters. One feature that is missing from this integration is
the ability to scale down to, and up from, a MachineSet or MachineDeployment with zero replicas.

This proposal defines an opt-in mechanism whereby Cluster API users and cloud providers can define
the specific resource requirements for each Infrastructure Machine Temalate they create. These
requirements are then utilized by the cluster autoscaler to make decisions about the number
of nodes to add during scale up operations.

## Motivation

Allowing the cluster autoscaler to scale down its node groups to zero replicas is a common feature
implemented for many of the integrated cloud providers. It is a popular feature that has been
requested for Cluster API on multiple occasions. This feature empowers users to reduce their
operational resource needs, and likewise reduce their operating costs.

Given that Cluster API is an abstraction point that provides access to multiple concrete cloud
implementations, this feature might not make sense in all scenarios. To accomodate the wide
range of deployment options in Cluster API, the scale to zero feature will be optional for
users and cloud providers.

### Goals

- Provide capability for Cluster API MachineSets and MachineDeployments to scale from and to zero replicas.
- Provide a mechanism for users to override the defaults for any given MachineSet or MachineDeployment.
- Create an optional API contract that defines the minimum needed information for scaling from and to zero replicas.

### Non-Goals/Future Work

- Create an API contract that cloud providers must follow.
- Create an API that replicates Taint and Label information from Machines to MachineSets and MachineDeployments.
- Create additional CRDs or controllers

## Proposal

To facilitate scaling from zero replicas, the minimal information needed by the cluster autoscaler
is the CPU and memory resources for the target node group that will be scaled. The autoscaler uses
this information to create a prediction about how many nodes should be created when scaling. In
most sitatuations this information can be directly read from the nodes that are running within a
node group. But, during a scale from zero sitatuation (ie when a node group has zero replicas) the
autoscaler needs to acquire this information from the cloud provider.

An optional field is proposed for users to add to their Infrastructure Machine Templates which will
carry information about the CPU, memory, and GPU requirements for the machines created from the template.
This field will then be copied to the status of any MachineSet or MachineDeployment with an infrastructure
reference to the template during reconciliation. The cluster autoscaler will then read this information
from the status field to perform its calculations.

A user may override the field in the associated infrastructure template by applying annotations to the
MachineSet or MachineDeployment in question. In these cases the autoscaler will evaluate the annotation
information in favor of reading the information from the status.

By keeping this information with the MachineSet and MachineDeployment resource objects, we reduce the
need for the cluster autoscaler to have a deep knowledge of Cluster API internals.

### User Stories

- Detail the things that people will be able to do if this proposal is implemented.
- Include as much detail as possible so that people can understand the "how" of the system.
- The goal here is to make this feel real for users without getting bogged down.

#### Story 1

As a Cluster API user, I would like to reduce my operating costs by scaling down my workload
cluster when they are not in use. Using the cluster autoscaler with a minimum size of zero for
a MachineSet or MachineDeployment will allow me to automate the scale down actions for my clusters.

#### Story 2

As a Cluster API user, I would like to have special resource nodes (eg GPU enabled) provided when needed by workloads
without the need for human intervention. As these nodes might be more expensive, I would also like to return them when
not in use. By using the cluster autoscaler with a zero-sized MachineSet or MachineDeployment, I can automate the
creation of nodes that will not consume resources until they are required by applications on my cluster.

### Implementation Details/Notes/Constraints

A new common API type will be created named `ResourceHints`, its definition will look like this:

```
type ResourceHints struct {
    vCPU   *string `json:"vCPU,omitempty"`
    Memory *string `json:"memory,omitempty"`
    GPU    *string `json:"gpu,omitempty"`
}
```

This new API type can be used with a machine template to inform about the resources used
by individual machines in the group. For example, here is a `DockerMachineTemplate` with
the integrated `ResourceHints`

```
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha4
kind: DockerMachineTemplate
metadata:
  name: workload-md-0
  namespace: default
spec:
  resourceHints:
    memory: 500mb
    vCPU: "1"
    GPU: "1"
  template:
    spec: {}
```

During reconciliation of MachineSets and MachineDeployments, if the `resourceHints` field is found
on a related `infrastructureRef`, then the `resourceHints` will be added to the `status` field
for that resource. For example, a MachineDeployment status might look like this:

```
status:
  observedGeneration: 1
  phase: Running
  selector: cluster.x-k8s.io/cluster-name=workload,cluster.x-k8s.io/deployment-name=workload-md-0
  resourceHints:
    memory: 500mb
    vCPU: "1"
    GPU: "1"
```

If a user wishes to override the resources hints as provided from the machine template, they may
do so by adding the following annotations to their MachineSets or MachineDeployments:

```
kind: MachineSet
metadata:
  annotations:
      machine.x-k8s.io/GPU: "0"
      machine.x-k8s.io/memoryMb: "8192"
      machine.x-k8s.io/vCPU: "2"
```

### Security Model

This feature should not impact the security model.

No new permissions are needed by the user for the information they will provide, nor by the
associated processes that will read it.

The addition of metadata about machine sizes will not short circuit any quotas associated with
the user for the cloud platform they are operating.

### Risks and Mitigations

One risk for this process is that the user experience is slightly convoluted. A user (or cloud provider)
will need to create these fields for any machine template which will be used to scale from zero.
This association might not seem obvious at first as the autoscaler will only see the information
throught the MachineSets and MachineDeployments.

Creating clear documentation about the flow information, and the action of the cluster autoscaler
will be the first line of mitigating the confusion around this process. Additionally, adding an
example in the test Docker provider will help to clarify usage for cloud providers.

## Alternatives

An alternative approach to reconciling the resource hints from the machine template to MachineSets
and MachineDeployments would be for the cluster autoscaler to directly read the machine template
through the `infrastructureRef` that exists within those objects. This would require the autoscaler
to query for the machine templates, as well as the required permissions for those objects.

Another alternative would be for cloud providers to dynamically provide this information at run time.
This may be an approach to consider in the future, but it would still require a method to expose
the data at the MachineSet and MachineDeployment level. In effect this might look like an
automated method of achieving the same action as this proposal.

A much larger alternative would be to create a new custom resource that would act as an autoscaling
abstraction. This new resource would be accessed by both the cluster autoscaler and the Cluster API
controllers, as well as potentially another operator to own its lifecycle. This approach would
provide the cleanest separation between the components, and allow for future features in a contained
environment. The downside is that this approach requires the most engineering and design work to
accomplish.

## Upgrade Strategy

As this field is optional, it should not negatively affect upgrades. That said, care should be taken
to ensure that this field is copied during any object upgrade as its absence will create unexpected
behavior for end users.

## Additional Details

### Test Plan

The cluster autoscaler tests for Cluster API integration do not currently exist outside of the downstream
testing done by Red Hat on the OpenShift platform. There have talks over the last year to improve this
situation, but it is slow moving currently.

The end goal for testing is to contribute the scale from zero tests that currently exist for OpenShift
to the wider Kubernetes community. This will not be possible until the testing infrastructure around
the cluster autoscaler and Cluster API have resolved more.

## Implementation History

- [X] 06/10/2021: Proposed idea in an issue or [community meeting]
- [ ] MM/DD/YYYY: Compile a Google Doc following the CAEP template (link here)
- [X] 10/07/2020: First round of feedback from community [initial proposal]
- [X] 03/10/2021: Present proposal at a [community meeting]
- [X] 03/10/2021: Open proposal PR

<!-- Links -->
[community meeting]: https://docs.google.com/document/d/1LW5SDnJGYNRB_TH9ZXjAn2jFin6fERqpC9a0Em0gwPE/edit#heading=h.bd545rc3d497
[initial proposal]: https://github.com/kubernetes-sigs/cluster-api/pull/2530
