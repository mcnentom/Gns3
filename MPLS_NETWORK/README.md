# MultiProtocol Label Switching

This a networking technique that routes data by assigning labels to packets instead of relying on traditional IP address lookups.

This allows for faster packet forwarding, as routers make decisions based solely on the label, which specifies a predetermined path, rather than examining the packet's full IP address.

MPLS operates between the data link layer (Layer 2) and the network layer (Layer 3) of the OSI model, often referred to as Layer 2.5 and is designed to be protocol-independent, enabling it to carry various types of traffic, including IP, ATM, and Frame Relay, over a single network.

### MPLS in the VRF / MP-BGP context

When ISPs provides Layer 3 VPNs (MPLS L3VPNs), three big pieces come together:

1. VRFs (Virtual Routing and Forwarding)

* Keep customer routing tables separate.

* Allow overlapping IPs between customers.

2. MP-BGP (Multiprotocol BGP)

* Carries customer routes across the ISP backbone.

* Tags routes with RD (Route Distinguisher) + RT (Route Target).

* Makes customer prefixes unique and tells which VRF to import them into.

3. MPLS

* Provides the transport mechanism across the ISP core.

* Core routers (called P routers) don't need customer routes; they just forward packets based on MPLS labels.

* Ensures scalability and efficiency.

### How it works 

Customer edge router (CE) sends a packet to its ISP provider edge router (PE).

PE router looks up the destination in the VRF for that customer.

PE assigns a label and sends the packet into the MPLS backbone.

Core (P) routers switch packets only using the MPLS labels (no IP lookups).

On the remote end, the destination PE removes the label, consults the correct VRF and delivers the packet to the correct CE/customer.

MP-BGP ensures that the routes for each VRF are exchanged between PEs, so the remote PE knows which VRF the packet belongs to.

### Analogy

Think of MPLS as a postal service with labels:

The VRF is the mailbox for each customer.

MP-BGP is the central database of who owns which mailbox.

MPLS labels are like the stickers on envelopes that help post offices route mail quickly without opening them.

MPLS:

* Scales better than just static VRF-to-VRF routing.

* Keeps customer traffic isolated.

* Supports overlapping IPs.

* Provides a service provider backbone that doesn't need to know customer routes (only labels).

* Enables advanced services like QoS, TE (Traffic Engineering) and Fast Reroute.

## MP-BGP

MP-BGP stands for Multiprotocol Border Gateway Protocol.

It's an extension of BGP-4 that allows BGP to carry routing information for multiple address families, not just IPv4 unicast.

Normal (classical) BGP = carries only IPv4 unicast routes.

MP-BGP = can carry multiple protocols and services, such as:

* IPv4 unicast (default)

* IPv6 unicast

* IPv4/IPv6 multicast

* VPNv4 / VPNv6 (used in MPLS Layer 3 VPNs)

* EVPN (Ethernet VPN)

Why MP-BGP is important

It's the control plane for MPLS Layer 3 VPNs (using VRFs, RDs, and Route Targets).

It allows ISPs to support multi-tenant customer routing without mixing customer routes.

It carries customer prefixes tagged with RD + RT so that PE routers can import/export them into the correct VRFs.

Example use case: MPLS L3VPN

2 customers (CustA, CustB) use the same ISP:

Both have 192.168.10.0/24 networks.

ISP must keep their routes separate (using VRFs).

MP-BGP advertises each customer's prefixes between ISP edge routers with an RD and RT.

RD makes the prefix unique (CustA: 100:1:192.168.10.0/24, CustB: 200:1:192.168.10.0/24).

RT tells other PE routers which VRF should import the route.

Without MP-BGP, the ISP could not separate overlapping prefixes.

How it works 

PE router learns customer routes (via static, OSPF, EIGRP, etc.) in a VRF.

The route is tagged with an RD and RT.

MP-BGP carries the VPNv4/VPNv6 routes across the ISP backbone.

Remote PE imports the route into the correct VRF using the RT.

Customer sites can communicate, but remain isolated from other tenants.

To enable MP-BGP:
* _router bgp 65000_
 * _address-family ipv4_
 * _neighbor 10.1.1.2 activate_
 * _exit-address-family_

 * _address-family vpnv4_
  * _neighbor 10.1.1.2 activate_
  * _neighbor 10.1.1.2 send-community extended_
 * _exit-address-family_

### BGP (Border Gateway Protocol)

Purpose: Standard inter-domain routing protocol (used between ISPs or within large networks).

Default Capability:

Exchanges IPv4 unicast routes only.

Each route consists of a prefix + next hop + attributes.

Use Cases:

Internet routing (public AS to AS).

Enterprise WAN edge routing.

Large-scale data centers.

Flavors:

eBGP (External BGP) : between different ASes (ISP to ISP, ISP to enterprise).

iBGP (Internal BGP) : inside one AS, often alongside IGP.

Main BGP Attributes (in order of decision-making)

Here's the typical Cisco order:

* Highest Weight (Cisco-specific)

* Local to the router only.

* Not shared with neighbors.

* Higher weight = preferred path.

    Default: 0 (except for routes you originate, which get 32768).

* Highest Local Preference (Local_Pref)

  Shared within the AS (propagated in iBGP).

  Controls exit point for outbound traffic.

* Higher value = preferred.

  Default: 100.

* Locally originated routes (network or aggregate)

  Routes originated by the local router (via network command or redistribution) are preferred.

* Shortest AS Path

  Counts number of AS hops.

  Fewer AS hops = preferred.

  Used mainly in eBGP.

* Lowest Origin Type

  IGP < EGP < Incomplete (lowest wins).

  Incomplete = learned via redistribution.

* Lowest MED (Multi-Exit Discriminator)

   Suggests preferred inbound entry point into an AS.

   Lower MED = preferred.

   Compared only when paths come from the same neighboring AS (unless bgp always-compare-med is set).

* eBGP over iBGP

    Prefer eBGP-learned routes over iBGP-learned ones.

    Lowest IGP Metric to Next Hop

   Checks IGP cost to reach the next-hop router.

   Lower IGP cost = preferred.

* Oldest Route (to ensure stability)

* Lowest Router ID (BGP router-id of advertising router)

* Lowest Neighbor IP address (as a final tie-breaker).


### IGP (Interior Gateway Protocol)

A routing protocol used within a single Autonomous System (AS) (one organization’s network).

Examples:

RIP (Routing Information Protocol) : old, distance vector.

EIGRP (Enhanced Interior Gateway Routing Protocol) : Cisco proprietary (hybrid).

OSPF (Open Shortest Path First) : link-state, widely used.

IS-IS (Intermediate System to Intermediate System) : link-state, used by large ISPs.

Characteristics:

* Fast convergence.

* Optimized for speed and efficiency within one network.

* Does not scale well to the whole internet.

For instance for the network description below:

A Provider Core (ISP) network that connects two Customer Sites.

Components

1. Customer Edge (CE) Routers

   Belong to the customer (e.g., Customer A).

   Do NOT run MPLS.

   Run static routes, OSPF, or EBGP with the Provider Edge.

2. Provider Edge (PE) Routers

   Sit at the boundary of ISP and Customer.

   Run VRFs (one per customer).

   Run MP-BGP with other PEs to exchange customer VPN routes.

   Run IGP (OSPF/IS-IS) inside the ISP core.

   Run MPLS (LDP or RSVP) with P routers.

3. Provider Core (P) Routers

   Internal routers of the ISP.

   Only care about IGP + MPLS labels.

   Do NOT know customer routes.

   Act as transit between PE routers.

* IP Plan

#### Core loopbacks (used for BGP + LDP IDs):

PE1 = 10.0.0.1/32

P = 10.0.0.2/32

PE2 = 10.0.0.3/32

#### Core links:

PE1-P = 10.0.0.12/30

P-PE2 = 10.0.0.16/30

#### Customer links:

CE1-PE1 = 192.168.1.0/30

CE2-PE2 = 192.168.2.0/30

#### Customer LANs (behind CEs):

Site 1 LAN = 10.1.1.0/24

Site 2 LAN = 10.2.2.0/24

#### Roles of Protocols

1. OSPF (IGP in Core)

    Runs between PE1-P-PE2.

    Advertises loopbacks and core links.

    Provides reachability for MPLS and MP-BGP sessions.

2. MPLS (LDP in Core)

    Labels all OSPF-learned routes.

    Allows PE1 to send VPN traffic to PE2 across P without exposing customer routes.

3. MP-BGP (between PEs)

    Runs over loopbacks (10.0.0.1 <-> 10.0.0.3).

    Carries VPNv4 routes (customer VRFs).

    Uses RD/RT to separate/associate VRF routes.

4. VRFs on PEs

   PE1 and PE2 create VRF CUST-A.

   CE1's LAN (10.1.1.0/24) and CE2's LAN (10.2.2.0/24) are imported/exported into that VRF.

   Result: CE1 <-> CE2 communication across ISP cloud.

## Core setup.

Here we'll begin with OSPF(A type of IGP) in the ISP Core (PE1-P-PE2), since MPLS and MP-BGP depend on it.

PE1 Loopback = 10.0.0.1/32

P Loopback = 10.0.0.2/32

PE2 Loopback = 10.0.0.3/32

PE1-P link = 10.0.0.12/30

P-PE2 link = 10.0.0.16/30

On PE1

* _PE1(config)# interface Loopback0_
* _PE1(config-if)# ip address 10.0.0.1 255.255.255.255_

* _PE1(config)# interface FastEthernet0/0_
* _PE1(config-if)# ip address 10.0.0.13 255.255.255.252_   ! Link to P
* _PE1(config-if)# no shut_

On P

* _P(config)# interface Loopback0_
* _P(config-if)# ip address 10.0.0.2 255.255.255.255_

* _P(config)# interface FastEthernet0/0_
* _P(config-if)# ip address 10.0.0.14 255.255.255.252_   ! Link to PE1
* _P(config-if)# no shut_

* _P(config)# interface FastEthernet0/1_
* _P(config-if)# ip address 10.0.0.17 255.255.255.252_   ! Link to PE2
* _P(config-if)# no shut_

On PE2

* _PE2(config)# interface Loopback0_
* _PE2(config-if)# ip address 10.0.0.3 255.255.255.255_

* _PE2(config)# interface FastEthernet0/0_
* _PE2(config-if)# ip address 10.0.0.18 255.255.255.252_   ! Link to P
* _PE2(config-if)# no shut_

We'll run OSPF area 0 across the whole ISP backbone.

On PE1

* _PE1(config)# router ospf 1_
* _PE1(config-router)# network 10.0.0.0 0.0.0.255 area 0_   ! Covers Loopback0 + core link

On P

* _P(config)# router ospf 1_
* _P(config-router)# network 10.0.0.0 0.0.0.255 area 0_

On PE2

* _PE2(config)# router ospf 1_
* _PE2(config-router)# network 10.0.0.0 0.0.0.255 area 0_

To check OSPF neighbors

* _show ip ospf neighbor_

Ping the loopback to confirm reachability.

**Note:** If you add PE-CE links address to the provider OSPF, you would "leak" customer routes into the provider core breaking isolation between customers. The customer network (CE) is separate and must be isolated inside a VRF.

## MPLS (LDP) Setup

MPLS (Multiprotocol Label Switching) replaces pure IP forwarding with label switching.

LDP (Label Distribution Protocol) is used by routers to assign labels to prefixes (usually loopbacks).

These labels build LSPs (Label Switched Paths) across the core.

BGP will later use these LSPs as tunnels to carry VRF traffic between PE1 and PE2.

Topology Reminder

PE1 <-> P <-> PE2 (all running OSPF in area 0).

Loopbacks:

PE1 = 10.0.0.1/32

P = 10.0.0.2/32

PE2 = 10.0.0.3/32

MPLS Configuration on Each Router

On PE1

* _PE1(config)# mpls ip_                ! Enable global MPLS
* _PE1(config)# interface fa0/0_
* _PE1(config-if)# ip address 192.168.12.1 255.255.255.0_
* _PE1(config-if)# mpls ip_             ! Enable MPLS on core-facing link

* _PE1(config)# interface loopback0_
* _PE1(config-if)# ip address 10.0.0.1 255.255.255.255_   // If not set earlier

On P (Provider Core Router)

* _P(config)# mpls ip_

* _P(config)# interface fa0/0_
* _P(config-if)# ip address 192.168.12.2 255.255.255.0_
* _P(config-if)# mpls ip_

* _P(config)# interface fa0/1_
* _P(config-if)# ip address 192.168.23.2 255.255.255.0_
* _P(config-if)# mpls ip_

* _P(config)# interface loopback0_
* _P(config-if)# ip address 10.0.0.2 255.255.255.255_   //If not set earlier

On PE2

* _PE2(config)# mpls ip_
* _PE2(config)# interface fa0/1_
* _PE2(config-if)# ip address 10.0.1.1 255.255.255.252_
* _PE2(config-if)# mpls ip_

**Note:** Repeated on all PE-PE int and PE-P int. CE does not particpate in the mpls so not configured with.

* _PE2(config)# interface loopback0_
* _PE2(config-if)# ip address 10.0.0.3 255.255.255.255_  //If not set earlier

Verification Commands

Check LDP neighbors

* _PE1# show mpls ldp neighbor_

   Should list P (10.0.0.2) as neighbor.

   PE2 should also see P as neighbor.

Check MPLS forwarding table

* _PE1# show mpls forwarding-table_

    Shows labels assigned to prefixes (e.g., loopbacks).

## MP-BGP Setup

OSPF (Step 1) built basic reachability in the core.

MPLS LDP (Step 2) created label-switched paths (LSPs) for forwarding.

Now, MP-BGP exchanges VPNv4 routes (VRF routes + labels) between PE routers.

This lets customer networks (e.g., LAN1 behind PE1 and LAN2 behind PE2) communicate securely over the shared provider core.

We'll configure iBGP between PE1 and PE2 using their loopbacks (10.0.0.1 <-> 10.0.0.3).

We'll use address-family vpnv4 for MP-BGP.

* _router bgp 65000_

**Note:** Intiates bgp process on this router.

**Note:** 65000 = the Autonomous System Number (ASN) of the provider's backbone.This ASN will be used in the AS Path attribute of routes.

* _bgp log-neighbor-changes_

**Note:** Tells the router to log messages whenever a BGP neighbor goes up or down. Helpful for troubleshooting BGP session flaps.

* _neighbor 10.0.0.2 remote-as 6500_

**Note:** Defines a BGP neighbor with IP 10.0.0.3.

**Note:** remote-as 65000 means this neighbor is also in AS 65000 -> so this is an iBGP session (internal BGP). 10.0.0.3 is the loopback address of PE2

* _neighbor 10.0.0.3 remote-as 65000_
* _neighbor 10.0.0.4 remote-as 65000_
* _neighbor 10.0.0.5 remote-as 65000_
* _neighbor 10.0.0.6 remote-as 65000_
* _neighbor 10.0.0.2 update-source loopback0_

**Note:** By default, BGP uses the outgoing interface's IP as the source of TCP sessions. Since we're peering loopback-to-loopback (instead of physical interface addresses), we must force PE1 to use its own loopback0 as the source IP for the BGP session.

**Note:** This ensures reliability if one physical link between PE1 and PE2 fails, the loopback session stays up as long as there is another path through the MPLS core.

**Note:** loopback0 is the loopbck int for PE1 and 10.0.0.2 is loopback int for the destination for instance PE2

* _neighbor 10.0.0.3 update-source loopback0_
* _neighbor 10.0.0.4 update-source loopback0_
* _neighbor 10.0.0.5 update-source loopback0_
* _neighbor 10.0.0.6 update-source loopback0_


* _address-family vpnv4_

**Note:** Switches the BGP context to the VPNv4 address family.

**Note:** VPNv4 = IPv4 routes + Route Distinguisher (RD). This is how PE routers exchange customer routes across the MPLS backbone

* _neighbor 10.0.0.2 activate_

**Note:** Activates VPNv4 peering with neighbor 10.0.0.2 (PE2).

**Note:** Without this, PE1 and PE2 would not exchange VPNv4 routes (they would only have IPv4 unicast).

* _neighbor 10.0.0.2 send-community extended_

**Note:** Ensures that extended BGP communities are sent to PE2.

**Note:** Extended communities carry the Route Target (RT), which tells PE2 which VPN a route belongs to. Without this, VPN separation would break, and all customer routes would mix.

* _neighbor 10.0.0.3 activate_
* _neighbor 10.0.0.3 send-community extended_
* _neighbor 10.0.0.4 activate_
* _neighbor 10.0.0.4 send-community extended_
* _neighbor 10.0.0.5 activate_
* _neighbor 10.0.0.5 send-community extended_
* _neighbor 10.0.0.6 activate_
* _neighbor 10.0.0.6 send-community extended_


_Repeat the configurations for all PE routers in the setup with_
_mp-bgp is only confiured on PE routers loopback/int and not P loopback/int connecting to the PE_

The vrf network has also to be advised underthe mp-bgp:

* _router bgp 65000_

**Note:** Enters BGP process with AS number 65000 (yours may differ). On an MPLS PE, this is the process that participates in VPNv4 
exchange with other PEs.

* _address-family ipv4 vrf VRF-NAME_

**Note:** Switches into the VRF-specific BGP context for CUST-A.This is where you control how prefixes inside the VRF are injected into BGP.Without entering this sub-mode, PE would only handle global BGP, not customer VRF routes.

* _redistribute connected_

**Note:** Tells BGP to advertise all connected subnets in the CUST-A VRF into MP-BGP. That means any subnet that is directly configured on a PE interface and belongs to the VRF (e.g., CE-facing link, LANs behind CE if directly attached) will now be injected into BGP and won't be advertised without this.

**Note:** redistribute connected can sometimes leak unwanted connected interfaces (like loopbacks or management addresses) if they are bound to the VRF. A safer approach is often: network <Subnet> mask <mask> i.e network 192.168.15.0  mask 255.255.255.0,this only advertises the network you want to be advertised in the bgp.

* _exit-address-family_

**Note:** Exits VRF-specific BGP mode and returns to the general BGP config context



On PE1 : Configure VRF (example for CustomerA)

* _PE1(config)# ip vrf CUST-A_
* _PE1(config-vrf)# rd 100:1_
* _PE1(config-vrf)# route-target export 100:1_
* _PE1(config-vrf)# route-target import 100:1_

* _PE1(config)# interface fa1/0_
* _PE1(config-if)# ip vrf forwarding CUST-A_
* _PE1(config-if)# ip address 192.168.10.1 255.255.255.0_

On PE2 : Same VRF (CustomerA)

* _PE2(config)# ip vrf CUST-A_
* _PE2(config-vrf)# rd 100:1_
* _PE2(config-vrf)# route-target export 100:1_
* _PE2(config-vrf)# route-target import 100:1_

* _PE2(config)# interface fa1/0_
* _PE2(config-if)# ip vrf forwarding CUST-A_
* _PE2(config-if)# ip address 192.168.20.1 255.255.255.0_

On CE configure default routes to the gateway:

* _ip route 0.0.0.0 0.0.0.0 CE-gateway-ip_

Verification

Check BGP neighbors

* _PE1# show bgp vpnv4 unicast all summary_

Check received VRF routes

* _PE1# show bgp vpnv4 unicast all_

Should list 192.168.20.0/24 (from PE2 VRF).

Check VRF routing table

* _PE1# show ip route vrf CUST-A_

Should have route to 192.168.20.0/24 via BGP.

Ping across VRFs

* _PE1# ping vrf CUST-A 192.168.20.4_

**Note:** CUST-A is the name of the vrf you want to reach
**Note:** 192.168.20.4 is the address of the end device or intermediary device connected to the PE router and is in vrf CUST-A




