---
title: Cloud egress IP component
authors:
  - "@alexanderConstantinescu"
reviewers:
  - "@danwinship"
  - "@abhat"
  - "@squeed"
  - "@tssurya"
  - "@trozet"
approvers:
  - "@danwinship"
  - "@abhat"
  - "@squeed"
  - "@tssurya"
creation-date: 2020-12-16
last-updated: 2021-01-29
status: implementable
---

# Cloud egress IP component

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift-docs](https://github.com/openshift/openshift-docs/)

## Summary

OVN-Kubernetes and openshift-sdn today support the feature egress IP on
bare-metal platforms. Going forward both network plugins will need to support
the feature on conventional cloud provider platforms (such as: GCP, AWS, Azure).
Cloud providers require additional management of the egress IP address via the
cloud provider's cloud API. That is required since the cloud provider will not
let the network plugin claim an additional IP on the node, which it does not
recognize.

To be able to do this we will need to add logic to a dedicated component as to
be able to handle this. This enhancement proposal goes into detail concerning:

- The cloud provider data model required as to manage egress IP assignments in
  the cloud
- The model and interaction between this future entity, the network plugin and
  any additional component required for this interaction to happen.

OpenShift 4.8 aims to finalize this component for cloud egress IP assignments,
functional with both openshift-sdn and OVN-Kubernetes. Azure is targeted for
OpenShift 4.9. GCP and AWS is targeted for OpenShift 4.10. This document tries
however to make an abstraction from the requirements and deadlines set for each
cloud w.r.t. each OpenShift release. Hence it does not solely focus on AWS or
Azure, but instead aims at defining an architecture which is cloud agnostic and
which would be applied with minimal implementation efforts for additional
clouds.

## Motivation

Customers would like to be able to run workloads on conventional cloud platforms
and assign them egress IPs, to be used as traffic "identifiers". The reasons for
this is usually within the scope of firewall-ing traffic within their corporate
network which is coming from the cloud, based on the source IP. They would like
the ability to filter pod egress traffic based upon an allow-list they define,
and then potentially tie all those allowed-to-egress services to an external IP
on a dedicated node that they have created in the cloud.

### Goals

- Designing a solution to the problem of managing additional private IP address
  assignments to the primary NIC associated with cloud VMs, in a cloud provider
  agnostic way.

### Non-Goals

- Assignment of external IP addresses to cloud VMs

- Assignment of additional NICs to cloud VMs

- Assignment of additional private IP address to any VM NIC besides the one the
  network plugin manages (for OpenShift today: this is anything != primary NIC).

- Providing a cloud provider solution for the egress router feature

## Proposal

### Implementation Details/Notes/Constraints

#### Cloud provider data model for additional private IP assignment

The following section describes initial researched information w.r.t. the API
for private IP address assignment to VMs, on the three major clouds (AWS, GCP
and Azure). The goal of this document is not to delve into the details of this,
as the implementation will have to consider all the specifics. Instead it
presents the general concepts as to give the reader an understanding of what
will be required by this future component. The API for doing additional IP
address assignment to VMs differ slightly between the clouds, but remain quite
similar in the grand scheme of things.

Note: although some cloud's go SDK API might not support the proper operations
required for assigning additional private IP addresses to VMs, it is expected
that their REST API will. We should thus be able to work-around any such SDK
limitations.

##### Azure

Azure requirement for assigning additional private IPs to an already existing
NIC is detailed in [the Azure API documentation]. Although the go SDK for
communicating with the Azure API has not been studied yet, it is expected to
resemble the Azure CLI API:

```bash
az network nic ip-config create \
--resource-group $RG_NAME \
--nic-name $NIC_NAME \
--private-ip-address 10.0.0.6 \
--name $NAME
```

Where `$RG_NAME` and `$NIC_NAME` needs to be discovered beforehand when
assigning an additional IP. `$NAME` needs to be a unique identifier. It is thus
expected that this component will be able to retrieve such information prior to
performing the IP assignment request to the Azure API.

Note also: the NIC configuration will need to be added to the list of ["load
balancer back-end address pools"]. This is because OpenShift IPI installs use
"loadbalancer" as the default mechanism for cluster ingress/egress, and in case
the NIC associated with the egress IP is not added to the list of back-end
address pools: traffic will not flow north for pods matching the egress IP.

[the Azure API documentation]:
https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-multiple-ip-addresses-cli#add

##### AWS

[The AWS API documentation] discusses additional private IP assignment to
existing NICs. AWS seems to differentiate their API between [IPv4]:

```bash
aws ec2 assign-private-ip-addresses --network-interface-id $NIC_ID --private-ip-addresses 10.0.0.82
```

and [IPv6] assignments:

```bash
aws ec2 assign-ipv6-addresses --network-interface-id $NIC_ID --ipv6-addresses 2001:db8:1234:1a00:3304:8879:34cf:4071

```

[IPv4] states the following which will be important to take into account during
the implementation of this component.

> Remapping an IP address is an asynchronous operation. When you move an IP
> address from one network interface to another, check
> network/interfaces/macs/mac/local-ipv4s in the instance metadata to confirm
> that the remapping is complete.

Hence, we would need to read the EC2 instance's metadata to ensure that the
re-assignment of egress IPs has been executed properly before notifying the
network plugin.

[the AWS API documentation]:
https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/MultipleIP.html#ManageMultipleIP
[IPv4]:
https://docs.aws.amazon.com/cli/latest/reference/ec2/assign-private-ip-addresses.html
[IPv6]:
https://docs.aws.amazon.com/cli/latest/reference/ec2/assign-ipv6-addresses.html
["load balancer back-end address pools"]:
https://docs.microsoft.com/en-us/azure/load-balancer/backend-pool-management

##### GCP

GCP uses a different networking model, detailed in [the GCP API documentation].
Instead of assigning additional IPs to NICs, they use something called "IP
aliasing". Important to mention is that OpenShift cluster nodes on GCP usually
get assigned a `/32` subnet on their primary interface. However, the IP aliasing
subnet is `/19` (master nodes have: `10.0.0.0/19` and worker nodes:
`10.32.0.0/19`). This facilitates this feature on GCP, as it means that we won't
need to change the physical networking model on GCP as to make egress IP work.
This however also means that we need to incorporate this fact into our design.
The networking plugin - currently - only looks at the subnet mask on the primary
interface defined on the host, when determining if an egress IP can be assigned
to a given OpenShift node.

The following shows the API on GCP for updating an instance and assigning it an
additional IP address

```bash
gcloud compute instances network-interfaces update $INSTANCE_NAME \
    --zone $ZONE \
    [--network-interface $NIC; default="nic0"]
    --aliases "$RANGE_NAME:$RANGE_CIDR,RANGE_NAME:RANGE_CIDR,..."
```

where

- `$ZONE` the zone that contains the instance.
- `$NIC` is the name of the network interface to which you're adding an alias IP
  address range.
- `$RANGE_NAME` the name of the subnet secondary range from which to draw the
  alias IP range.
- `$RANGE_CIDR` is the IP range to assign to the interface. The range can be a
  specific range (192.168.100.0/24), a single IP address (192.168.100.1), or a
  network mask in CIDR format (/24). If the IP range is specified by network
  mask only, the IP allocator chooses an available range with the specified
  network mask and allocates it to the network interface.

Note: an egress IP setup on GCP has already been accomplished using
OVN-Kubernetes in a proof of concept / experimental mode. A couple of things
were noticed during that experiment:

- the alias IP assigned does not have any impact on the host, meaning: once the
  alias IP is assigned using the GCP API, the IP address does not show up on the
  host (on any interface) when doing `ip addr`
- no additional host assignment of this alias IP is required for SNAT-ing to
  occur to the alias IP, i.e: `ip addr add $ALIAS_IP dev eth0`
- OVN's dedicated SNAT does not work, iptables rule were needed, such as:
  `iptables -t nat -D POSTROUTING -p tcp -m tcp -d $NODE_IP/32 -j
  SNAT--to-source $ALIAS_IP` to make things work. It is believed to be a bug in
  OVN, but further coordination with them will be required to figure out why it
  can't SNAT, whereas that iptables rule works. This is not a concern for
  openshift-sdn, as it programs that exact iptables rule by default for egress
  IP.

[the GCP API documentation]:
https://cloud.google.com/sdk/gcloud/reference/compute/instances/network-interfaces/update

#### Model

This cloud component is not intended on being "user facing", its purpose is a
means to an end for implementing a OpenShift networking feature in the cloud (in
this case: egress IP) by the networking stack. As such the primary "user" of
this cloud component will be the network plugins themselves. This impacts the
design decision with regards to the cloud component's failure notification, see
[CRD section](#crd).

##### network plugin <-> cloud API controller

The network plugin needs to remain "source of truth" of the egress IP setup. The
reason for this is that the network plugin is the one and only component which
health checks routes to egress nodes and re-balances the egress IP assignment
depending on the result. This component will need to execute what the network
plugin has determined is best. This is done since there is no traffic load
balancing done today for egress IP intended traffic. The network plugin
constantly initiates a TCP connection to each egress node and depending on the
success/failure of that: re-balances the egress assignment to other nodes (thus
avoiding packets being sent to a egress node which is currently un-reachable).
Going forward, both network plugins will come to support traffic load-balancing
and a more sophisticated route health detection. That means that the underlying
networking solution (OVS in the case of both OVN-Kubernetes and openshift-sdn)
will manage the route health check. It is worth noting that the "amount of
re-balancing" done by the network plugin is expected to be reduced, since the
network fabric will be able to load balance traffic between all egress nodes
provided and always use a healthy node (thus not requiring the network plugin to
perform an egress IP re-assignment every time a route goes down).

Given the above, it is however important that the network plugin is conveyed the
cloud's information concerning the nodes' subnet range. This is especially
evident on GCP, as without it: the network plugin won't be able to perform any
assignments at all, given the nodes' default `/32` subnet.

#### Inter-component communication

Multiple pieces of information needs to be communicated between the network
plugins and this cloud component:

1. The cloud's node subnet information.
2. The egress IP assignment to be performed by the cloud component
3. The status of the egress IP assignment performed by the cloud component

Two models have been decided for this: a CRD based approach and a node
annotated approach.

##### annotated approach

As to satisfy condition 1. the cloud component will annotate all cluster nodes
with the nodes' cloud subnet as retrieved from the cloud provider's API when it
is initialized. This annotation will have the following specification:

`cloud.openshift.io/node-ifaddr: {"ipv4": "$IPv4_ADDRESS/$IPv4_SUBNET_MASK", "ipv6": "$IPv6_ADDRESS/$IPv6_SUBNET_MASK"}`

This will in turn be read by the network plugin and used during the egress IP
assignment, superseding any host level subnet information already parsed by the
network plugin.

##### CRD

The cloud component will define a dedicated CRD that the network plugin will use
to issue cloud egress IP requests. The network plugin will in-turn finalize its
setup and set its final egress IP status pending the CR's status indicating a
successful/unsuccessful cloud assignment. For example: if the cloud assignment
should be unsuccessful; it will be up to the network plugin to notify the user
of this, by setting its own status field for the egress IP assignment
accordingly.

The CRD will have `scope: Cluster` (since egress IP is cluster-scoped and node
private IP assignment have no relation to namespaces), always have a single and
unique CR, and define the following format:

```yaml
apiVersion: cloud.openshift.io/v1
kind: CloudPrivateIPConfig
metadata:
  name: cluster
spec:
  items:
  - node: nodeX
    ip: 192.168.126.11
  - node: nodeY
    ip: 192.168.126.102
  - node: nodeZ
    ip: 192.168.126.103
status:
  items:
  - node: nodeX
    ip: 192.168.126.11
    status: SUCCESS 
    reason: ""
  - node: nodeY
    ip: 192.168.126.102
    status: PENDING
    reason: ""
  - node: nodeZ
    ip: 192.168.126.103
    status: ERROR
    reason: "Some reason why this would have failed"
```

where `nodeX`/`nodeY` will be cluster node names as retrieved by
`nodes[*].metadata.name`. The referential order between `.spec.items` and
`.status.items` will not be strict, meaning `.spec.items[0]` will not
necessarily correspond to `.spec.items[0]`.

There will always only exist one CR which has the name: `cluster`, following the
convention for OpenShift clusters with a singleton CRD. The reason for such a
pattern is because the egress IP assignments can change, as mentioned in
[network plugin <-> cloud API
controller](#network-plugin-<->-cloud-API-controller). In case an egress IP is
moved from node X to node Y, the cloud component would need to make sure that
the egress IP is not assigned to both nodes at no moment in time during this
change. Having one singleton CR to manage this will enforce that.

The CRD will thus be used to cover point 2. and 3. above.

The cloud components control loop (or dedicated admission controller) will need
to validate the following:

1. Any egress IP can only be assigned to one node, and one node only, at any
   given moment in time. The reason for this is "IP address uniqueness". This is
   validated on the network plugins' side already, but an additional check
   should be enforced here where the result should be flagging the duplicated
   egress IP as error.

2. No egress request IP can reference a node's IP. This is validated on the
   network plugins' side already, but an additional validation in this component
   is required.

3. No egress IP should be assigned a lower index than the default node IP in the
   cloud provider's referential model for the instance's IP addresses, see
   [section on node CSR renewal](#node-CSR-renewal)

##### Cloud credentials operator - CCO

[The cloud credentials operator (CCO)] is an OpenShift component that manages
the `CredentialRequest` CR. This resource allows other components access to the
cloud API by generating a secret in the requested namespace which the resource
issuing the `CredentialRequest` can use. The secret contains the specific
permissions required to communicate with the cloud API as specified by the
requesting component.

The deployment model for this component will thus require that such a
`CredentialRequest` is issued to the CCO as to retrieve the cloud credentials.
This will be done by the cluster-network-operator when deploying the networking
stack, which will include this component.

[The cloud credentials operator (CCO)]:
https://github.com/openshift/cloud-credential-operator


##### OpenShift vs. upstream project

This component will be an OpenShift dedicated component, however since
ovn-org/ovn-kubernetes does implement the egress IP feature supported on
bare-metal clusters: it has been discussed (during the upstream community
meeting on 27/01/2021) whether supporting the cloud egress IP feature would make
sense for that organization as well. The conclusion from the discussion was that
the cloud feature is seen as valuable pending the ability to enable/disable the
feature using a feature flag. This should not be a problem for OVN-Kubernetes,
since such config parameters already exists.

**upstream**

OVN-Kubernetes upstream will need to import and create a dependency to this
OpenShift specific component as to be able to watch for its CRD and issue cloud
egress IP requests. Given this, it is expected that OVN-Kubernetes upstream will
need to:

- Optionally allow the enablement of the cloud egress IP feature through a
  config parameter.
- In case the feature is enabled: it is up to the cluster admin deploying
  OVN-Kubernetes to also deploy this cloud egress IP component.
- In case the feature is enabled: create a secret in the namespace chosen for
  this component following the output format of the CCO. This secret will be
  mounted by the cloud egress IP component when deployed.

##### RBAC

Given the model defined above, a separate set of RBAC permissions will be
required for the cloud component / network plugins. These are listed below.

**Cloud component**:

WATCH/LIST/GET:

- `nodes` - verify node names / other info when communicating with the cloud API
- `cloudprivateipconfig.cloud.openshift.io` - to watch for incoming cloud egress IP
  requests

UPDATE/PATCH/CREATE:

- `nodes` - annotating the node with `cloud.openshift.io/node-ifaddr`
- `cloudprivateipconfig.cloud.openshift.io` - updating the cloud egress IP request
  with its status

**Network plugin**

WATCH/LIST/GET:

- `nodes` - as to read the `cloud.openshift.io/node-ifaddr` annotation
- `cloudprivateipconfig.cloud.openshift.io` - to watch for cloud egress IP
  status assignments

UPDATE/PATCH/CREATE:

- `cloudprivateipconfig.cloud.openshift.io` - issue a cloud egress IP request
  
##### Flow chart model

The following section describes the flow chart / sequence of events for both
network plugins and cloud component, following the model detailed above:

###### Cloud component

**Initialization**

1. Lists all nodes in the cluster and queries the cloud API for each nodes'
   subnet information
2. Annotates all nodes with `cloud.openshift.io/node-ifaddr`, to be read
   by the network plugin

**Egress IP assignment**

1. Retrieves a new `cloudprivateipconfig.cloud.openshift.io` CR and reads the
   `.status.items`
2. Communicates with the cloud API requesting the additional private IP
   assignment to the nodes' NIC.
3. Retrieves the results of 2. and updates the
   `cloudprivateipconfig.cloud.openshift.io` CR with `.status.items`

###### OVN-Kubernetes

**Initialization**

1. When OVN-Kubernetes starts: ovnkube-node will read the subnet off of the
   primary network interface (on AWS and Azure this should correspond to the
   subnet retrieved from the cloud API) and annotates the node object with
   `k8s.ovn.org/node-primary-ifaddr` for ovnkube-master to read once egress IP
   assignments need to be performed.
2. It will then continue watching nodes as to retrieve all nodes'
   `cloud.openshift.io/node-ifaddr` annotation (which will be updated by
   the cloud component during its initialization) and finalize the
   initialization of its required assignment data.

**Egress IP assignment**

1. User labels the nodes wished to be used for egress assignment with
   `k8s.ovn.org/egress-assignable`
2. User creates a `egressips.k8s.ovn.org` CR with its requested egress IP
   assignments.
3. ovnkube-master, having initialized all node level assignment data: performs
   an assignment and issues a cloud egress IP assignment request by creating a
   `cloudprivateipconfig.cloud.openshift.io` CR with `.spec.items` filled out.
4. Watches for the update of the CR and inspects `.status.items` to
   retrieve the assignment status.
5. Performs cluster level networking setup for the egress IPs.  
6. Updates the `egressips.k8s.ovn.org` CR created by the user in 2. with its
   `.status.items`, notifying the user of the final assignment status.

###### openshift-sdn

Note: openshift-sdn does not flag successful or unsuccessful assignment of
egress IPs in any explicit way, it only logs unsuccessful assignment in its own
logs. The user takes care of modifying the `hostsubnet` and `netnamespace` CRs,
but if an incorrect setup is performed (for example by not aligning the egress
IP with the nodes' subnet range), the user is never notified of this but needs
to take an active action and look in the openshift-sdn logs.

**Initialization**

1. When openshift-sdn starts: it initializes the `hostsubnet` CR and fills in
   the fields of each node's subnet information.  
2. It will then continue watching nodes as to retrieve all nodes'
   `cloud.openshift.io/node-ifaddr` (which will be updated by the cloud
   component during its initialization) and finalize the initialization of its
   required assignment data.

**Egress IP assignment**

1. User patches `hostsubnet.egressIP` / `hostsubnet.egressCIDRs`  corresponding
   to "flagging" the nodes wished to be used for egress assignment
2. User patches `netnamespace.egressIPs` on the namespace wished to be matched
   for egress IP assignment
3. openshift-sdn, having initialized all node level assignment data: performs
   an assignment and issues a cloud egress IP assignment request by creating a
   `cloudprivateipconfig.cloud.openshift.io` CR with `.spec.items` filled out.
4. Watches for the update of the CR and inspects `.status.items` to
   retrieve the assignment status.
5. Performs cluster level networking setup for the egress IPs.  
6. Triggers an `event` to the user notifying successful/unsuccessful assignment.

### Risks and Mitigations

#### Risks

Finalizing the work for 4.8 has a risk arising mainly CI/CD setup for this new
component.

Given future OpenShift features requiring cloud level modifications of the
network fabric (once such existing future enhancement is: node port range
modifications): the proposal here is to call the repository;
openshift/cloud-network-config-controller. The name would explicitly indicate
that this component performs cloud level network modifications, without digging
itself into a "egress IP dedicated hole". New features would be added and
supported by defining separate CRDs.

##### Node CSR renewal

Clusters deployed on clouds such as the ones described in this document,
retrieve the node addresses from the cloud provider API. The following links
show the process employed for each:

[AWS]

AWS seems to use all IPs listed in the instance's metadata:
`network/interfaces/macs/$mac/local-ipv4` as the node's `InternalIP`. The
metadata will also be populated with egress IP once the assignment has been
performed, see: [the AWS section](#aws). The node is thus expected to have the
egress IP listed as the node's `InternalIP` once the IP has been attached. This
might have impacts on the CSR renewal for the node.

[Azure]

Azure, seems to only grab the first private IP addresses listed for the primary
NIC associated with the instance. This means that the egress IP assignment,
should not show up in the node's `InternalIP` addresses.

[GCE]

GCE iterates over all NICs associated with the instance and retrieves the IP
address associated with each to set the node's `InternalIP` addresses. This
means that the egress IP assignment will not show up among the node's
`InternalIP` addresses, because the egress IP will be assigned as an "alias IP",
as described in [the GCP section](#gcp)

[AWS]: https://github.com/kubernetes/legacy-cloud-providers/blob/ce3a9094e0ca26c7c4676be32b292e9836affade/aws/aws.go#L1498-L1502
[Azure]: https://github.com/kubernetes/legacy-cloud-providers/blob/ce3a9094e0ca26c7c4676be32b292e9836affade/azure/azure_instances.go#L122-L127
[GCE]: https://github.com/kubernetes/legacy-cloud-providers/blob/ce3a9094e0ca26c7c4676be32b292e9836affade/gce/gce_instances.go#L107-L117

The Kubelet will issue a CSR renewal for the IPs known to it. On node reboot on
AWS it would come to pick up the new egress IP, subsequently causing issues if
the machine approver does not sync its expected IPs which are pending CSR
approval with the cloud API.

## Design Details

### Test Plan

The cloud component and egress IP assignment will undergo general feature
testing, meaning unit, CI and QE testing.  

Moreover, as to validate the functionality of this component we will need to
implement a specific cloud, at least as a proof of concept. Given that Azure is
planned to be the first cloud implemented in OpenShift 4.9, the focus will be on
it. The expectation will be to use Azure as a validator of this component's
functionality, but not GA Azure egress IP support in 4.8.

This component's impact on the node CSR renewal will need to undergo QE testing
as well, as there has already been [examples] of egress IP impacting the
Kubelet's CSR renewal mechanism in the past.

[examples]: https://bugzilla.redhat.com/show_bug.cgi?id=1860774

## Implementation History

- *v3.11*: egress IP for openshift-sdn was developed, supporting bare-metal and
  vSphere deployments.
- *v4.6*: egress IP for OVN-Kubernetes was developed, supporting bare-metal and
  vSphere deployments
- *v4.8*: this egress IP cloud component needs to be supported for both
  openshift-sdn and OVN-Kubernetes. Azure is used a "test cloud"
- *v4.9*: egress IP on Azure needs to be supported for both openshift-sdn and
  OVN-Kubernetes
- *v4.10*: egress IP on GCP and AWS needs to be supported for both openshift-sdn
  and OVN-Kubernetes
