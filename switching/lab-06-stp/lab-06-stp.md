# Spanning Tree Protocol
 
**Build It · Observe It · Break It · Fix It**
 
---
 
## Overview
 
Redundant links are a good thing. Two paths between switches mean a single cable failure does not take down the network. But at Layer 2 there is a catch that does not exist at Layer 3: an Ethernet frame has no TTL. A routed packet that gets stuck in a loop counts down its TTL and eventually dies, but a switched frame caught in a loop circulates forever, multiplying every time it passes a switch. Add a broadcast to that loop and you get a broadcast storm that saturates every link and pins every switch CPU within seconds. A single misplaced cable really can take an entire Layer 2 network down that fast.
 
Spanning Tree Protocol is the safeguard that lets you have redundant links without the catastrophe. STP looks at the physical topology, elects one switch as the **root bridge**, calculates the best path from every other switch back to that root, and then **blocks** just enough ports to leave exactly one active path between any two points. The redundant links are still physically there, sitting in a blocking state, ready to take over the instant an active link fails. You get redundancy and a loop-free topology at the same time.
 
In this lab you will build three switches in a triangle, which is a physical loop on purpose, and watch STP break that loop by blocking a single port. You will set the root bridge deliberately, identify every port role and state, and then break the network in three ways that get to the heart of how STP behaves: an accidental root bridge, a link failure that forces reconvergence, and the slow access port that PortFast exists to fix. By the end, reading `show spanning-tree` output should feel like reading a map.
 
**CCNA Exam Alignment:**
- 2.5 – Describe the need for and basic operations of Rapid PVST+ Spanning Tree Protocol
- 2.5.a – Root port, root bridge (primary and secondary), and other port names
- 2.5.b – Port states (forwarding and blocking)
- 2.5.c – PortFast
**Estimated Time:** 90–120 minutes
 
---
 
## Key Concept: Root, Roles, and States
 
STP makes three decisions, in order, and almost everything you will observe and troubleshoot comes back to them.
 
**First, it elects a root bridge.** Every switch has a **Bridge ID** made of a priority value plus its MAC address. In the per-VLAN spanning tree that Cisco runs, the VLAN number is folded into the priority. The switch with the lowest Bridge ID wins and becomes the root. The default priority is 32768, so if you change nothing, the election comes down to the lowest MAC address, which is effectively random. That is why setting the root deliberately is a best practice rather than a nicety. You do not want the oldest switch in the closet winning the election by accident.
 
**Second, every non-root switch chooses one root port.** The root port is the port with the lowest total path cost back to the root bridge. Cost is based on link speed, and lower is better:
 
| Link speed | STP path cost |
|------------|---------------|
| 10 Gbps | 2 |
| 1 Gbps | 4 |
| 100 Mbps | 19 |
| 10 Mbps | 100 |
 
**Third, it assigns a role to every port and blocks where needed.** These are the roles worth knowing cold:
 
| Port role | What it does |
|-----------|--------------|
| **Root port** | The one port on a non-root switch with the best path to the root. Forwards. |
| **Designated port** | The forwarding port for a given segment. Every segment has exactly one. The root bridge's ports are all designated. |
| **Alternate port** | A port with a path to the root that lost to a better one. This is the port STP blocks to break the loop. |
 
A blocked port is not dead. In Rapid PVST+, which is the modern Cisco default and the focus of the CCNA, a port is either **discarding** (the blocking state, it drops data but still listens to BPDUs), **learning**, or **forwarding**. The older 802.1D states were blocking, listening, learning, and forwarding, and you will still hear "blocking" used loosely to mean discarding.
 
> **Get ahead of the confusion:** When you see one port in a blocking or discarding state in this lab, that is not a fault. That is STP doing its job, holding a redundant link in reserve so the topology stays loop-free. The whole lab hinges on understanding that the blocked port is a feature, not a problem.
 
---
 
## Topology
 
```
                       [SW1]   (Root Bridge)
                      /      \
                 trunk        trunk
                    /            \
                [SW2] === trunk === [SW3]
                  |                   |
                G0/2                G0/2
                [PC1]               [PC2]
              VLAN 10             VLAN 10
            192.168.10.11       192.168.10.12
 
  Three switches wired in a triangle form one physical loop.
  STP blocks a single port to leave one loop-free active path.
```
 
**Nodes (5 total, within CML Free Edition limit):**
- SW1, SW2, SW3 (IOSvL2 switches, wired in a triangle)
- PC1 (on SW2), PC2 (on SW3)
> **The triangle is the whole point.** Three switches each connected to the other two is a deliberate loop. Without STP, that loop would storm the instant the switches powered on. When you build the topology, draw all three inter-switch links.
 
> **A note on interface names:** Port names vary by IOSvL2 image, so confirm with `show ip interface brief` and adjust. This guide uses G0/0 and G0/1 for the inter-switch trunks and G0/2 for the host ports.
 
---
 
## Addressing Table
 
| Device | Interface | IP Address     | Subnet Mask     | VLAN |
|--------|-----------|----------------|-----------------|------|
| PC1    | NIC       | 192.168.10.11  | 255.255.255.0   | 10   |
| PC2    | NIC       | 192.168.10.12  | 255.255.255.0   | 10   |
 
> **No gateway needed.** Both hosts share VLAN 10 and one subnet, so their traffic stays at Layer 2 and never has to be routed. That keeps the focus on the spanning tree itself.
 
---
 
## Lab Objectives
 
By the end of this lab you will be able to:
 
1. Explain why a Layer 2 loop is catastrophic and how STP prevents it
2. Describe root bridge election and set the root bridge deliberately
3. Identify root ports, designated ports, and the blocked alternate port
4. Read port states and explain what a discarding port is doing
5. Observe STP reconverge after a link failure
6. Configure PortFast on an access port and explain why edge ports need it
7. Diagnose and resolve common spanning tree problems
---
 
## 🧱 Part 1: Build It
 
### Step 1 — Set the STP Mode and Create the VLAN on All Three Switches
 
Set the mode to Rapid PVST+, which is the CCNA focus and the modern default, then create VLAN 10. Do this on SW1, SW2, and SW3.
 
```
SW1> enable
SW1# configure terminal
SW1(config)# spanning-tree mode rapid-pvst
SW1(config)# vlan 10
SW1(config-vlan)# name Sales
SW1(config-vlan)# exit
```
 
Repeat the same three commands on SW2 and SW3.
 
### Step 2 — Trunk the Three Inter-Switch Links
 
Each switch connects to the other two over trunks. Configure both uplink ports on each switch as trunks. For example, on SW1:
 
```
SW1(config)# interface range GigabitEthernet0/0 - 1
SW1(config-if-range)# switchport mode trunk
SW1(config-if-range)# exit
```
 
Do the same on SW2 and SW3 for their two inter-switch ports. Once all three links are trunked, you have created the loop, and STP will immediately begin blocking a port to break it.
 
### Step 3 — Configure the Host Access Ports
 
```
SW2(config)# interface GigabitEthernet0/2
SW2(config-if)# switchport mode access
SW2(config-if)# switchport access vlan 10
SW2(config-if)# description PC1 - Sales
SW2(config-if)# exit
 
SW3(config)# interface GigabitEthernet0/2
SW3(config-if)# switchport mode access
SW3(config-if)# switchport access vlan 10
SW3(config-if)# description PC2 - Sales
SW3(config-if)# exit
```
 
### Step 4 — Set the Root Bridge Deliberately
 
This is the single most important habit in spanning tree. Do not let the election fall to the lowest MAC address. Make SW1 the root and SW2 the backup.
 
```
! On SW1: become the primary root for VLAN 10
SW1(config)# spanning-tree vlan 10 root primary
 
! On SW2: become the secondary root, ready to take over if SW1 fails
SW2(config)# spanning-tree vlan 10 root secondary
```
 
> **Why this matters so much:** The `root primary` command lowers SW1's priority enough to win the election, and `root secondary` sets SW2 just above it. Now the root is where you decided it should be, near the center of your network, instead of wherever the oldest MAC address happens to live. Setting this deliberately also makes the rest of this lab predictable, since the blocked port lands in a known place.
 
### Step 5 — Configure the Hosts
 
```
PC1:  ip 192.168.10.11 255.255.255.0
PC2:  ip 192.168.10.12 255.255.255.0
```
 
### Step 6 — Save All Three Configurations
 
```
SW1# copy running-config startup-config
SW2# copy running-config startup-config
SW3# copy running-config startup-config
```
 
---
 
## 👀 Part 2: Observe It
 
### 2.1 — Confirm the Root Bridge
 
Start on SW1.
 
```
SW1# show spanning-tree vlan 10
```
 
**What to look for:** Near the top, a line stating that this bridge is the root, confirming SW1 won the election. Note its Bridge ID and priority.
 
Now run the same command on SW2 and SW3:
```
SW2# show spanning-tree vlan 10
SW3# show spanning-tree vlan 10
```
 
Neither should claim to be the root. Instead, each shows the root's Bridge ID and points to its own root port.
 
**✏️ Prediction checkpoint:** Before you look, predict which switch is the root and why. Then confirm SW1 won because you set its priority lowest.
 
### 2.2 — Identify Every Port Role
 
On each switch, read the port list at the bottom of the `show spanning-tree vlan 10` output. For each port you will see a role and a state.
 
**What to look for:**
- On **SW1**, the root, both inter-switch ports are **designated** and **forwarding**. The root's ports are always designated.
- On **SW2** and **SW3**, the port facing SW1 is the **root port** and forwarding, because the direct link to the root is the lowest-cost path.
- On exactly one of SW2 or SW3, the port on the SW2-to-SW3 link is an **alternate** port in the **discarding** (blocking) state. This is the port STP chose to block to break the loop.
**✏️ Find the blocked port.** Walk all three switches and locate the single discarding port. Because you set SW2 as secondary root, SW2 wins the designated role on the SW2-to-SW3 segment, so the blocked port should sit on SW3's link toward SW2. Write down exactly which port it is. This one port is what keeps the triangle loop-free.
 
### 2.3 — See the Root Detail
 
```
SW2# show spanning-tree root
```
 
This compact view shows the root Bridge ID, the cost to reach it, and which local port is the root port, all on one line per VLAN. It is the fastest way to confirm who the root is and how a switch reaches it.
 
### 2.4 — Test Connectivity Across the Loop-Free Path
 
```
PC1> ping 192.168.10.12
```
 
This should succeed. Trace the path in your head: because the SW2-to-SW3 link is blocked on one end, PC1's traffic travels from SW2 up to the root SW1 and back down to SW3 to reach PC2. The longer path is the price of a loop-free topology, and it is a price worth paying.
 
---
 
## 💥 Part 3: Break It
 
> Work through each scenario one at a time. Document the symptoms fully before you move to Part 4.
 
---
 
### Break Scenario 1 — The Accidental Root Bridge
 
This is the most common real-world spanning tree problem. Someone adds a switch, or changes a priority, and the root moves somewhere it should not be. Force SW3 to hijack the root role by giving it the lowest possible priority:
 
```
SW3(config)# spanning-tree vlan 10 priority 0
```
 
Wait a few seconds, then observe:
```
SW3# show spanning-tree vlan 10
SW1# show spanning-tree vlan 10
PC1> ping 192.168.10.12
```
 
**✏️ Document the symptoms:**
- Which switch is the root now?
- Did the blocked port move to a different location in the topology?
- Connectivity may still work, so why is an unplanned root bridge still a problem?
> **Why this matters:** STP will keep the network loop-free no matter which switch is root, so connectivity often survives a root change. The damage is subtler. The root should sit at the center of your network so traffic takes efficient paths, but an accidental root in a back corner forces traffic to detour through that corner, quietly degrading performance. A root that lives where you did not choose is a problem even when pings still succeed, and the fix is always to set priorities deliberately. This is exactly the case for `root primary` and `root secondary`.
 
---
 
### Break Scenario 2 — Reconvergence After a Link Failure
 
Restore SW1 as the deliberate root first. Then break the active path and watch STP heal itself. Start a continuous ping from PC1 to PC2, then shut SW1's link toward SW2:
 
```
SW3(config)# no spanning-tree vlan 10 priority 0
SW1(config)# spanning-tree vlan 10 root primary
SW2(config)# spanning-tree vlan 10 root secondary
 
! Now break the active path: shut SW1's link to SW2
SW1(config)# interface GigabitEthernet0/0
SW1(config-if)# shutdown
SW1(config-if)# exit
```
 
> Confirm which SW1 interface faces SW2 in your build before shutting it. Adjust the interface if yours differs.
 
Wait a few seconds, then observe:
```
SW2# show spanning-tree vlan 10
SW3# show spanning-tree vlan 10
```
 
**✏️ Document the symptoms:**
- SW2 lost its direct path to the root. What is its new root port now?
- The port that used to be blocking on the SW2-to-SW3 link, what state is it in now?
- Did the continuous ping recover on its own, and roughly how quickly?
> **Why this matters:** This is STP delivering on its entire promise. When the active path failed, the previously blocked link transitioned into forwarding and restored connectivity with no human intervention. SW2 now reaches the root through SW3 instead of directly. Rapid PVST+ does this in a few seconds, far faster than the thirty to fifty seconds the original 802.1D took, which is one of the main reasons the rapid version became the default. The redundant link you were "wasting" just earned its place.
 
Restore the link when you are done observing:
```
SW1(config)# interface GigabitEthernet0/0
SW1(config-if)# no shutdown
SW1(config-if)# exit
```
 
---
 
### Break Scenario 3 — The Slow Access Port
 
This break is about the edge of the network, where hosts connect. By default, an access port still walks through the spanning tree states before it forwards, which delays a host every time it connects. Watch it happen by bouncing PC1's port without PortFast:
 
```
SW2(config)# interface GigabitEthernet0/2
SW2(config-if)# shutdown
SW2(config-if)# no shutdown
SW2(config-if)# exit
```
 
Immediately try to use the link from PC1:
```
PC1> ping 192.168.10.12
```
 
**✏️ Document the symptoms:**
- Is there a delay before PC1 can pass traffic after the port comes back up?
- Run `show spanning-tree interface` for that port quickly and note the state it passes through before forwarding.
- Why would this delay cause real problems for a host trying to get a DHCP address at boot?
> **Why this matters:** An access port connected to a single host has no possibility of forming a loop, yet by default it still spends time transitioning before it forwards. On a real network that delay causes failed DHCP requests and hosts that appear dead for the first several seconds after boot. PortFast tells STP that a port is an edge port and may forward immediately. It is standard practice on every access port, and the absence of it is a classic source of "my PC could not get on the network when it started up" tickets.
 
---
 
## 🔧 Part 4: Fix It
 
---
 
### Fix Scenario 1 — The Accidental Root Bridge
 
**Your troubleshooting toolkit:**
```
show spanning-tree vlan 10        ! confirm which switch is root
show spanning-tree root           ! quick view of the root Bridge ID
```
 
**Structured process:**
1. Confirm the root is not where you intended it to be.
2. Identify the switch that won the election with too low a priority.
3. Remove the bad priority and reassert your deliberate root assignment.
4. Verify the root returns to the correct switch.
**✅ Solution:**
```
SW3(config)# no spanning-tree vlan 10 priority 0
SW1(config)# spanning-tree vlan 10 root primary
SW2(config)# spanning-tree vlan 10 root secondary
```
 
**Verification:**
```
SW1# show spanning-tree vlan 10
```
 
---
 
### Fix Scenario 2 — Reconvergence After a Link Failure
 
The fix here is to restore the failed link, and the lesson is that STP already healed on its own while the link was down.
 
**Your troubleshooting toolkit:**
```
show spanning-tree vlan 10
show ip interface brief
```
 
**Structured process:**
1. Note that connectivity recovered automatically when the link failed, because STP unblocked the redundant path.
2. Locate and restore the downed link.
3. Watch STP settle back to the original topology, with the redundant port returning to blocking.
4. Verify the root port assignments return to normal.
**✅ Solution:**
```
SW1(config)# interface GigabitEthernet0/0
SW1(config-if)# no shutdown
SW1(config-if)# exit
```
 
**Verification:**
```
SW2# show spanning-tree vlan 10
PC1> ping 192.168.10.12
```
 
---
 
### Fix Scenario 3 — The Slow Access Port
 
**Your troubleshooting toolkit:**
```
show spanning-tree interface GigabitEthernet0/2
```
 
**Structured process:**
1. Confirm the access port walks through the transitional states before forwarding.
2. Recognize that a port facing a single host is an edge port and should forward immediately.
3. Enable PortFast on the access port.
4. Bounce the port again and confirm it now forwards right away.
**✅ Solution:**
```
SW2(config)# interface GigabitEthernet0/2
SW2(config-if)# spanning-tree portfast
SW2(config-if)# exit
```
 
> **Pair PortFast with BPDU Guard in production.** PortFast assumes nothing but a host lives on that port. If someone plugs a switch into a PortFast port, you want BPDU Guard to shut it down rather than risk a loop. The command is `spanning-tree bpduguard enable` on the same interface. It is the safety net that makes PortFast safe to use.
 
**Verification:**
```
SW2# show spanning-tree interface GigabitEthernet0/2
```
 
---
 
## 💬 Reflection Questions
 
1. A routed packet caught in a loop eventually dies, but a switched frame caught in a loop does not. Explain the difference, and why that difference makes Layer 2 loops so much more dangerous than Layer 3 routing loops.
2. The root bridge is chosen by lowest Bridge ID, which combines priority and MAC address. If every switch keeps its default priority, what decides the election, and why is relying on that outcome a bad idea?
3. Define the three port roles you observed: root port, designated port, and alternate port. Which one does STP place in a blocking or discarding state, and why?
4. In Break Scenario 1, an accidental root bridge did not break connectivity, yet it was still a real problem. Explain what degrades when the root bridge sits in the wrong place even though pings still succeed.
5. In Break Scenario 2, a previously blocked link began forwarding the moment the active path failed. Explain how this delivers redundancy, and why Rapid PVST+ recovering in a few seconds is a meaningful improvement over the original STP timing.
6. PortFast lets an access port forward immediately instead of transitioning through the usual states. Why is this safe on a port connected to a single host but dangerous on a port connected to another switch, and how does BPDU Guard protect against the dangerous case?
---
 
## Challenge Extension (Optional)
 
1. **Influence the blocked port with cost.** Rather than letting STP choose, change the path cost on a specific port with `spanning-tree vlan 10 cost` and watch the blocked port move. Predict the new topology before you apply it.
2. **Witness a storm, carefully.** This is the dramatic version of why STP matters, and it can briefly overwhelm the lab, so do it deliberately and reverse it quickly. Disable spanning tree for VLAN 10 on all three switches with `no spanning-tree vlan 10`, watch for MAC flapping messages and rising CPU for just a few seconds, then immediately re-enable it with `spanning-tree vlan 10`. You will have seen firsthand the failure STP exists to prevent.
3. **Add BPDU Guard and trip it.** Enable PortFast and BPDU Guard on an access port, then move one of the inter-switch links onto that port to simulate a switch being plugged into an edge port. Watch BPDU Guard place the port in err-disabled state, and research how to recover it.
4. **Compare the timers.** Research the difference between the legacy 802.1D states and timers and the Rapid PVST+ states. Why can the rapid version converge in seconds when the original needed up to fifty?
---
 
## Where This Leads
 
You have reached the end of the core switching topics, and you did it by building the one thing every switched network depends on and most people never see working. You set a root bridge on purpose, found the single port STP blocks to keep a triangle loop-free, watched a redundant link spring to life the instant the primary failed, and learned why edge ports need PortFast. That is the full arc of how a redundant Layer 2 network stays both resilient and safe.
 
There is one lab left, and it is the one that ties the entire section together. The capstone assembles VLANs, trunking, Layer 3 switching, EtherChannel, and the spanning tree behavior you just studied into a single collapsed core network shaped like a real campus. Everything you have practiced in isolation will operate as one system, and the troubleshooting will ask you to tell, at a glance, which layer a fault lives in. With spanning tree now fresh in your mind, you are ready for it.
 
---
 
*CCNA – Build It · Observe It · Break It · Fix It series*

*Next: Switching Capstone (Collapsed Core Campus LAN)*
 
*This repository is an independent educational resource and is not officially affiliated with Cisco Systems. Cisco, CCNA, Cisco Modeling Labs, and Cisco Networking Academy are trademarks of Cisco Systems, Inc.*
 
