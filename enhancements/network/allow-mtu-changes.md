---
title: allow-mtu-changes
authors:
  - "@juanluisvaladas"
  - "@jcaamano"
reviewers:
  - "@danwinship"
  - "@dcbw"
  - "@knobunc"
  - "@msherif1234"
approvers:
  - TBD
creation-date: 2021-10-07
last-updated: 2021-10-14
status: provisional
---

# Allow MTU changes

This covers adding the capability to the cluster network operator of changing
the MTU post installation.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Operational readiness criteria is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift-docs](https://github.com/openshift/openshift-docs/)

## Summary

Customers may need to change the MTU post-installation. However these changes
aren't trivial and may cause downtime, hence the CNO currently forbids them.

This enhancement proposes an automated procedure launched on demand and
coordinated by the Cluster Network Operator. This procedure is based on
draining and rebooting the nodes sequentially to minimize service disruption
and to ensure that the nodes have a healthy configuration and preserve
connectivity with each other.

## Motivation

While cluster administrators usually set the MTU correctly during the
installation, sometimes they need to change it afterwards for reasons such as
changes in the underlay or because they were set incorrectly at install time.

### Goals

* Allow to change MTU post install on OVN Kubernetes.

### Non goals

* Other safe or unsafe configuration changes.

## Proposal

Cluster Network Operator (CNO from now on) will drive the MTU change when it 
detects a change on the configured value in the operator configuration. The
procedure will include an automatic double rolling reboot triggered by CNO
through Machine Config Operator (MCO from now on). Due to the reboots it is
adecuate to consider this an MTU migration rather than just simply an MTU
change. Given the impact that the reboots have on the cluster, a specific
annotation will be introduced in the operator configuration to safeguard it.

Up to now, CNO has not had any responsibility over the host MTU but it is
critical to consider it for this MTU migration scenario to avoid service
disruptions and unnecessary reboots.

Changes are required on OVN-Kubernetes, CNO and MCO.

### OVN-Kubernetes: new `routable-mtu` configuration parameter

A new `routable-mtu` setting is introduced in OVN-Kubernetes to faciliate a
procedure allowing to change the MTU on a running cluster with minimum service
disruption.

OVN-Kube Node, upon start, sets that `routable-mtu` on the appropriate host
and new pod routes. This will make all node-wide outgoing traffic effectively
adjust to that MTU value while the node and pod interfaces remain 
configured with the original `mtu` value and thus capable of handling incoming
traffic adjusted to that MTU value.

This new setting enables a double rolling reboot procedure to change the MTU
during which different MTU configurations accross nodes are tolerated with
no service disruption.

The general procedure, given current and target MTU values, is:
1. Set `mtu` to the higher MTU value and `routable-mtu` to the lower MTU value.
2. 
2. Do a rolling reboot. As a node restarts, routable-mtu is set on
   all appropriate routes while interfaces have mtu configured.
   The node will effectively use the lower routable-mtu for outgoing
   traffic, but be able to handle incoming traffic up to the higher
   mtu.
3. Change the MTU on all interfaces not handled by OVN-K to the target
   MTU value. Change the MTU on all routes not handled by OVN-K
   Since the MTU effectively used in the cluster is the lower
   one, this has no impact on traffic.
4. Set mtu to the target MTU value and unset routable-mtu.
5. Do a rolling reboot. As a node restarts, the target MTU value is set
   on the interfaces and the routes are reset to default MTU values.
   Since the MTU effectively used in other nodes of the cluster is the
   lower one but able to handle the higher one, this has no impact on
   traffic.

`routable-mtu` will be set as MTU in the following routes:
* default route in pods, covering traffic from a pod to any destination that is
  not a pod on the same host.
* non link scoped management port routes in nodes, covering traffic from nodes
  to services or pods on different nodes.

#### MTU Decrease example

Current MTU: 9000
Current OVN-K MTU: 8900
Target MTU: 1500
Target OVN-K MTU: 1400

* Set in ovn-config:
  `mtu`: `8900` (no change)
  `routable-mtu`: `1400` (previously unset)
* Do a rolling reboot of all nodes. As nodes restart, outgoing traffic will be
  adjusted to the `1400` MTU value which both restarted and non-restarted nodes
  will be able to handle. Interfaces MTU will remain unchanged. Wait until all
  nodes have restarted.
* Set in ovn-config:
  `mtu`: `1400` (changed from `8900`)
  `routable-mtu`: `` (unset)
* Set host default interface MTU to `1500`.
* Do a rolling reboot of all nodes. This has the effect of setting the MTU on 
  interfaces to `1400` and reseting the routes to a default unset MTU. At this
  stage, since the effectively used MTU was already `1400`, there is no impact.

#### MTU increase example

Current MTU: 1500
Current OVN-K MTU: 1400
Target MTU: 9100
Target OVN-K MTU: 9000

* Set in ovn-config:
  `mtu`: `9000` (changed from `1400`)
  `routable-mtu`: `1400` (previously unset)
* Do a rolling reboot of all nodes. As nodes restart, outgoing traffic will
  still be adjusted to the `1400` MTU value which both restarted and
  non-restarted nodes will be able to handle. Interfaces MTU will be reset to
  `9000`. Wait until all nodes have restarted.
* Set in ovn-config:
  `mtu`: `9000` (no change)
  `routable-mtu`: `` (unset)
* Set host default interface MTU to `9100`.
* Do a rolling reboot of all nodes. This has the effect of reseting the routes
  to a default unset MTU and change the effective MTU in use to `9000`. At this
  stage, since all cluster interfaces are already adjusted to that MTU, there
  is no impact.

### MCO: configurable MTU for configure-ovs service

configure-ovs is a systemd service deployed through MCO that prepares host 
networking for OVN-Kubernetes. The proposal is to introduce a systemd dropin
for this service with two configuration parameters: `MTU` and `ROUTABLE_MTU`:
* If `MTU` is set, configure-ovs will set that MTU value in br-ex and
  ovs-if-phys0 interfaces.
* If `ROUTABLE_MTU` is set, configure-ovs will set that MTU value to all
  br-ex link scoped routes.
* If `ROUTABLE_MTU` is unset, configure-ovs will unset the MTU value on all
  br-ex link scoped routes.

MCO will deploy by default this dropin with both parameters unset.

### CNO: coordinate MTU change

The CNO monitors changes on the operator configuration. When it detects a MTU
change, given current and target MTU values:
1. Check if an annotation `mtu-migration` is set in the operator. If not,     
   consider the MTU change unsafe as done currently and don't proceed.
2. Check if annotation `mtu-migration` has a value, and if so, that it is a 
   valid host MTU value with respect to the OVN-Kubernetes MTU value set in the
   operator configuration. If not, consider the MTU change unsafe and don't
   proceed. Otherwise, consider this to be the host MTU value or assume a
   default of the target MTU value + 100.
2. Render the OVN-Kubernetes configuration with `mtu` set to the high MTU
   value and `routable-mtu` to the low MTU value of the current and target MTU
   values.
3. Render the `cno-set-mtu-<pool>` MachineConfig object for both the `master`
   and `worker` machine config pools containing the configure-ovs dropin with
   `MTU` set to the host MTU value and `ROUTABLE_MTU` to the low MTU
   value + 100.
4. Wait until MCO has applied the Machine Config object and performed the
   rolling reboot of all nodes.
5. Render a OVN-Kubernetes configuration with `mtu` set to the target MTU
   value and `routable-mtu` unset.
6. Render the `cno-set-mtu-<pool>` MachineConfig object for both the `master`
   and `worker` machine config pools containing a configure-ovs dropin with 
   `MTU` and `ROUTABLE_MTU` unset.
7. Wait until MCO has applied the Machine Config object and performed the
   rolling reboot of all nodes.
8. Set the target MTU value in CNO `applied-cluster` config map. MTU value in 
   in CNO `applied-cluster` config map should stay at the current MTU value
   until this step.

CNO should be able to derive operator status from MachineConfig pool status.
The MachineConfig pool status should only be taken into account if it sources a
MachineConfig that has been rendered by CNO. If that is the case:
* CNO status should be progressing if MachineConfig pool is status is updating.
* CNO status should be degraded if MachineConfig pool is status is degraded.

The CNO status is expected to be derived from MachineConfig pool status
throughout the procedure since CNO renders MachineConfig objects. Outside of
this procedure there is no MachineConfig objects rendered by CNO so CNO status
should not be derived from MachineConfig pool status.

Note that CNO transitioning from rendering to not rendering MachineConfig
objects should not result in any machine updates since the contents of the
final MachineConfig object rendered are effectively the default configure-ovs
dropin resulting in no machine state change required.

### User Stories

#### As an administrator, I want to change the cluster MTU

An administrator should be able to change the cluster MTU

### Implementation Details/Notes/Constraints

## Design Details

### Open Questions

* The proposed procedure is completly automated. Should we consider making some
  of the steps manual as an option? Perhaps indicated through an alternate
  annotation `manual-mtu-migration` that would have the effect of rendering the
  MachineConfig objects unlabeled. The administrator could then add the label
  to manually trigger the reboots. We could also offer the option to set which
  MachineConfig pools to work with, to provide more control over the reboot
  process.


### Test Plan

Two new CI tests, one for IPv4 and another for IPv6, that will run upstream
network tests and verify that the MTU set is the one effectively used, both
after increasing and decreasing the MTU values.

### Risks and Mitigations

* A cluster wide MTU change procedure cannot be carried out in an instant and
  incurs the risk of having different MTUs in use accross the cluster at a given
  time. While the described procedure in this enhancement prevents this with the
  use of route MTUs, some consequences from using different MTUs in the cluster 
  are analized in the following section for context.

#### Running the cluster with different MTUs

On the process of a `live` change of the MTU, there is going to be traffic
endpoints temporarily using different MTU values. In general, if the path MTU
to an endpoint is known, fragmentation will occur or the application will be
informed that it is trying to send larger packets than possible so that it can
adjust. Additionally, connection oriented protocols, such as TCP, usually
negotiate their segment size based on the lower MTU of the endpoints on 
connection.

So generally, different MTUs on endpoints affect ongoing connection-oriented
traffic or connection-less traffic, when the known destination MTU is not the
actual destination MTU. In this case, the most likely scenario is that traffic
is dropped on the receiving end by OVS if larger than the destination MTU.

There are circumstances that prevent an endpoint from being aware of the actual
MTU to a destination, which depends on Path MTU discovery and specific ICMP
`FRAG_NEEDED` messages:
* A firewall is blocking these ICMP messages or the ICMP messages are not being
  relayed to the endpoint.
* There is no router between the endpoints generating these ICMP messages.

Let's look at different scenarios.

##### Node to node

On the receiving end, a nic driver might size the buffers in relation to the
configured MTU and drop larger packets before they are handed off to the system.

Past that, OVN-K sets up flows in OVS br-ex for packets that are larger than pod
MTU and sends them off to the network stack to generate ICMP `FRAG_NEEDED`
messages. If these packets exceed the MTU of br-ex, they will be dropped by OVS
and never reach the network stack. Otherwise they will reach the network stack
but not generate ICMP `FRAG_NEEDED` messages as network stack only does so for
traffic being forwarded and not for traffic with that node as final destination.

As there is generally no router in between two cluster nodes, more than likely a
node would not be aware of the path MTU to another node.

##### Node to pod

As explained before, network stack receives larger packets betwewen host MTU and
pod MTU and might cause ICMP `FRAG_NEEDED` messages to be sent to the originating
node such that a node might be aware of the proper path MTU when reaching out to
pod. Otherwise larger than pod MTU traffic will dropped by OVS.

##### Pod to Node

On this datapath, OVS at the destination node will drop the larger packets without
generating ICMP `FRAG_NEEDED` messages as the node is the final destination of the
traffic. The originating pod is never aware of the actual path MTU.

##### Pod to Pod

This traffic is encapsulated with geneve. The geneve driver might drop it and
generate ICMP `FRAG_NEEDED` messages back to the originating pod if it is trying
to send packets that would not fit in the originating node MTU once encapsulated.
But OVN is not prepared to relay back these ICMP messages to the originating pod
so it would not be aware of an appropiate MTU to use.

On the receiving end, OVS would drop the packet silently if larger than the
destination MTU of the veth interface. Even if this would not be the case, the
veth driver itself would drop the packet silently if over the MTU of the pod's
end veth interface.

## Alternatives

* An alternative was considered to perform the MTU change directly rolling out
  the ovnkube-node daemon set with a new MTU value and changing all the running
  pods MTU. This approach, while quicker and more convinient, was discarded
  as it was considered less safe due to possible edge cases where the odd
  chance of leaving pods behind with wrong MTU values was dificult to avoid, as
  well as offering lease guarantess of minimizing service disruption.

