# Switching

A switch straight out of the box does one job: it forwards frames within a single broadcast domain. This section is about everything you do beyond that to make a switch ready for a real network. You will divide one switch into many with VLANs, carry those VLANs between switches over trunks, restore communication between them with inter-VLAN routing, bundle links for bandwidth and resilience with EtherChannel, keep redundant paths loop-free with Spanning Tree, and finally assemble all of it into a single working campus network.

I have sequenced these seven labs deliberately. Each one solves a problem the previous lab created, which is how switching actually works in practice. VLANs introduce isolation, trunks let that isolation span multiple switches, inter-VLAN routing reconnects what VLANs separated, EtherChannel and Spanning Tree make redundancy both possible and safe, and the capstone proves you can run all of it at once. Work through them in order and the logic builds on itself.

---

## Who This Section Is For

- **Students** who have finished the Routing section and are ready to understand the Layer 2 world that sits underneath it
- **Instructors** looking for switching labs that teach troubleshooting, not just configuration
- Anyone who can configure a router but has always treated the switch as a black box

**Before you start, you should be comfortable with:**
- Building and navigating a topology in CML (see CML Foundations)
- Basic IOS device configuration and verification
- The idea of a subnet and a default gateway (the Routing section covers this thoroughly)

If routing still feels shaky, spend more time there first. Inter-VLAN routing in particular leans directly on the forwarding logic you built in the routing labs.

---

## The Learning Philosophy

Every lab in this section follows the same cycle:

> ### 🧱 Build It · 👀 Observe It · 💥 Break It · 🔧 Fix It

You build a working Layer 2 network, observe how the switch makes its decisions, break it in the ways switches actually break in the field, and troubleshoot your way back. Switching rewards this approach especially well, because so many switching failures are silent. A port in the wrong VLAN, a trunk that will not form, an EtherChannel that will not bundle, a spanning tree topology that blocks the wrong link: none of these throw an obvious error. You learn to read the evidence, and the only way to get good at that is to create the failure yourself and then chase it down.

---

## The Labs in This Section

### 🗂️ Lab 01 — VLANs
**`lab-01-vlans`**

The foundation of everything else here. A VLAN divides one physical switch into several logical ones, each its own broadcast domain.

You will build two VLANs on a single switch, assign access ports, and prove the separation works: same-VLAN hosts communicate, different-VLAN hosts do not. The break scenarios cover the three most common VLAN mistakes, including the deleted VLAN that orphans its ports rather than returning them to VLAN 1.

**You'll learn:**
- What a VLAN is and why it is a separate broadcast domain
- Creating and naming VLANs
- Configuring access ports and assigning VLAN membership
- Verifying with `show vlan brief` and `show interfaces status`

**CCNA aligned:** ✅ Yes (2.1)
**Nodes:** 5 (1 switch, 4 hosts)

---

### 🔗 Lab 02 — Trunking
**`lab-02-trunking`**

VLANs are only useful on one switch until you can carry them between switches. A trunk does exactly that, tagging frames so a single link can carry traffic for many VLANs at once.

You will connect two switches with an 802.1Q trunk, extend both VLANs across it, and confirm that a host on one switch can reach its VLAN partner on the other. The break scenarios focus on the classics: a trunk that will not form because of a mode mismatch, and the native VLAN mismatch that quietly drops traffic.

**You'll learn:**
- How 802.1Q tagging carries multiple VLANs over one link
- Configuring and verifying trunk links
- The role and risks of the native VLAN
- Troubleshooting trunks with `show interfaces trunk`

**CCNA aligned:** ✅ Yes (2.1, 2.2)
**Nodes:** Within the 5-node limit (2 switches, 3 hosts)

---

### 🔀 Lab 03 — Inter-VLAN Routing: Router-on-a-Stick
**`lab-03-inter-vlan-routing-router-on-a-stick`**

VLANs separate traffic on purpose, but eventually Sales needs to reach a server in the Engineering VLAN. Moving traffic between VLANs requires a Layer 3 device, and the first method you will learn uses a single router interface divided into subinterfaces.

You will configure a router with one subinterface per VLAN, each acting as the default gateway for its VLAN, and finally bring to life those gateway addresses the VLANs lab left unreachable. The break scenarios target subinterface encapsulation, the switch-to-router trunk, and the physical interface dependency.

**You'll learn:**
- Why inter-VLAN communication requires Layer 3
- Configuring router subinterfaces with 802.1Q encapsulation
- Assigning a default gateway per VLAN
- Verifying routing between VLANs

**CCNA aligned:** ✅ Yes (2.2, applied routing)
**Nodes:** Within the 5-node limit (1 router, 1 switch, 2 hosts)

---

### 🧭 Lab 04 — Inter-VLAN Routing: Layer 3 Switch
**`lab-04-inter-vlan-routing-layer3-switch`**

Router-on-a-stick works, but it forces all inter-VLAN traffic through a single router link. The modern approach moves the routing into the switch itself using switch virtual interfaces.

You will enable routing on a Layer 3 switch, create an SVI for each VLAN, and route between VLANs entirely within the switch. This lab pairs directly with Lab 03 so you can compare the two methods and understand when each one fits.

**You'll learn:**
- Enabling IP routing on a Layer 3 switch
- Configuring switch virtual interfaces (SVIs) per VLAN
- How SVI-based routing compares to router-on-a-stick
- Verifying inter-VLAN routing on a switch

**CCNA aligned:** ✅ Yes (2.2, applied routing)
**Nodes:** 3 (1 Layer 3 switch, 2 hosts)

---

### ⛓️ Lab 05 — EtherChannel
**`lab-05-etherchannel`**

Redundant links between switches give you more bandwidth and protection from a single cable failure, but only if you can use them all at once. EtherChannel bundles multiple physical links into one logical link that forwards on every member.

You will bond two links between a pair of switches into an LACP EtherChannel, confirm traffic uses the bundle, and watch it survive the loss of a member. The break scenarios center on the mode mismatches that keep a bundle from forming, including the static "on" versus LACP trap.

**You'll learn:**
- Why and how multiple links are bundled into one logical link
- Configuring Layer 2 EtherChannel with LACP
- The negotiation modes and which combinations actually bundle
- Verifying with `show etherchannel summary`

**CCNA aligned:** ✅ Yes (2.2.d)
**Nodes:** Within the 5-node limit (2 switches, 2 hosts, bundled links)

---

### 🌲 Lab 06 — Spanning Tree Protocol
**`lab-06-stp`**

Redundant links protect you from a single cable failure, and they also create loops that will bring a network down in seconds. Spanning Tree Protocol keeps a redundant Layer 2 topology loop-free by selectively blocking ports.

You will build a switched topology with a deliberate loop, watch STP elect a root bridge and block a redundant path, then break and observe the protocol as it reconverges. This is the lab where the silent, behind-the-scenes work of a switch finally becomes visible.

**You'll learn:**
- Why Layer 2 loops are catastrophic and how STP prevents them
- Root bridge election and how to influence it
- Reading port roles and states
- Observing reconvergence when a link changes

**CCNA aligned:** ✅ Yes (2.3, 2.4)
**Nodes:** Designed to stay within the 5-node limit (3 switches)

---

### 🏛️ Lab 07 — Switching Capstone: Collapsed Core Campus LAN
**`lab-07-capstone`**

The finale. Everything above, assembled into the shape of a real campus network.

You will build a collapsed core design with one Layer 3 core switch routing between VLANs and two Layer 2 access switches connecting the users, with the uplinks bundled into trunked EtherChannels for resilience. The break scenarios are different on purpose: each one produces a differently shaped failure, and your job is to read the shape and locate the fault before touching a command. This lab proves you can run VLANs, trunking, Layer 3 switching, and EtherChannel as one working system.

**You'll learn:**
- The collapsed core design and how it maps to the Cisco hierarchy
- Integrating VLANs, trunking, SVIs, and EtherChannel into one network
- How EtherChannel and Spanning Tree work together
- Diagnosing a layered network by isolating which tier holds a fault

**CCNA aligned:** ✅ Yes (1.1, 1.2, 2.1, 2.2)
**Nodes:** 5 (1 Layer 3 core, 2 access switches, 2 hosts)

---

## Recommended Path

```
Lab 01     Lab 02      Lab 03         Lab 04        Lab 05        Lab 06     Lab 07
VLANs  →  Trunking →  Inter-VLAN  →  Inter-VLAN →  Ether-    →   STP    →  Capstone
                      (Router on     (Layer 3      Channel
                       a Stick)       Switch)
  │          │            │              │            │            │          │
Divide    Carry       Reconnect      A better     Bundle      Keep        Run it
the       VLANs       the VLANs      way to       links for   redundant   all as
switch    between     with a         reconnect    bandwidth   links       one
          switches    router                      and         loop-free   network
                                                  resilience
```

Do Labs 03 and 04 back to back, since they solve the same problem two different ways and the comparison is where the understanding lives. Save Lab 07 for last no matter what, because it assumes everything before it.

---

## Where to Go Next

Once you finish switching, you will have the full Layer 2 and Layer 3 access picture, and you will have built it into a working network. From here the remaining sections build outward toward the services and protections that run on top of that foundation:

- **Services**, including DHCP, NAT, and DNS
- **Security**, including SSH and ACLs

DHCP is the natural next step, because it automates the very IP addressing you have been configuring by hand on every host throughout this section.

---

## A Note Before You Begin

Switching is where a lot of people decide the network is "just working" and stop looking closely. I want you to do the opposite. The switch is making dozens of decisions every second, and almost all of them are invisible until something goes wrong. These labs pull those decisions into the open and ask you to break them on purpose, because a switch you have broken and repaired is a switch you actually understand.

**Build it. Observe it. Break it. Fix it.**

---

*Part of the CCNA – Build It · Observe It · Break It · Fix It lab series.*
*This repository is an independent educational resource and is not officially affiliated with Cisco Systems. Cisco, CCNA, Cisco Modeling Labs, and Cisco Networking Academy are trademarks of Cisco Systems, Inc.*
