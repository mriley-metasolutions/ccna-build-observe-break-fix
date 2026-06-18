# Inter-VLAN Routing: Layer 3 Switch
 
**Build It · Observe It · Break It · Fix It**
 
---
 
## Overview
 
In the last lab you routed between VLANs with router-on-a-stick, and it worked great. You also found its weakness on your own: every packet moving between VLANs had to leave the switch, travel to a router, and come back across a **single** shared link. On a small network nobody notices. On a busy one, that single link becomes the bottleneck that everything queues behind.
 
This lab removes the bottleneck by moving the routing function directly into the switch. A **Layer 3 switch**, sometimes called a multilayer switch, does everything a Layer 2 switch does and also routes between VLANs in hardware, at switching speed, without ever sending traffic to an external router. Instead of router subinterfaces, you create a **switch virtual interface (SVI)** for each VLAN, give it the gateway IP for that VLAN, and turn on routing. The VLANs you already know how to build suddenly gain the ability to talk to each other inside the same box that separates them.
 
This method is the modern standard. Walk into almost any campus network built in the last fifteen years and the inter-VLAN routing happens on Layer 3 switches in the distribution layer, not on routers hanging off on a stick. By the end of this lab you will have configured it both ways, and you will understand exactly why the industry settled on this one.
 
**CCNA Exam Alignment:**
- 2.2 – Configure and verify interswitch connectivity
- 1.1 – Compare network components, including Layer 2 and Layer 3 switches
- Builds on inter-VLAN routing and the routing fundamentals from Section 3

**Estimated Time:** 60–90 minutes
 
---
 
## Key Concept: The SVI and the `ip routing` Switch
 
Two ideas carry this whole lab, and both are small commands with large consequences.
 
The first is the **switch virtual interface (SVI)**. You have actually met an SVI before, back in the first-network foundations lab, where you gave a Layer 2 switch a management IP on `interface vlan 1`. That was an SVI used purely for management. Here you use the same kind of interface for a bigger purpose. An SVI for VLAN 10 with the address 192.168.10.1 becomes the default gateway for every host in VLAN 10, and the switch routes traffic between SVIs internally.
 
The second is the command **`ip routing`**. A Layer 3 Cisco switch ships with routing turned off, behaving like an ordinary Layer 2 switch until you tell it otherwise. The single global command `ip routing` flips it into a router. You can configure every SVI perfectly, and nothing will route between them until that command is in place.
 
> **Get ahead of the confusion:** The most common reason a freshly built Layer 3 switch refuses to route is a missing `ip routing` command. Everything looks correct, the SVIs are up, the hosts have gateways, and still nothing crosses between VLANs. You will break it exactly this way in Break Scenario 1, because I want the symptom burned into your memory before you ever meet it for real.
 
---
 
## Topology
 
```
        [PC1]                  [PC2]
       VLAN 10               VLAN 20
      192.168.10.11        192.168.20.21
         |                     |
        G0/1                 G0/2
              [MLS1]
        Layer 3 (multilayer) switch
        ip routing ENABLED
        SVI VLAN 10 -> 192.168.10.1
        SVI VLAN 20 -> 192.168.20.1
```
 
**Nodes (3 total, within CML Free Edition limit):**
- MLS1 (IOSvL2 switch acting as a Layer 3 switch)
- PC1 (VLAN 10), PC2 (VLAN 20)
> **A note on the switch image:** In CML, the IOSvL2 image supports Layer 3 switching features including SVIs and `ip routing`, so it stands in well for a multilayer switch. As always, interface names vary by image, so confirm with `show ip interface brief` and adjust. If a specific command is not supported on your image version, note it and we can adapt.
 
---
 
## Addressing Table
 
| Device | Interface  | IP Address     | Subnet Mask     | VLAN | Role |
|--------|------------|----------------|-----------------|------|------|
| MLS1   | SVI VLAN 10 | 192.168.10.1  | 255.255.255.0   | 10   | VLAN 10 gateway |
| MLS1   | SVI VLAN 20 | 192.168.20.1  | 255.255.255.0   | 20   | VLAN 20 gateway |
| PC1    | NIC        | 192.168.10.11  | 255.255.255.0   | 10   | Sales host |
| PC2    | NIC        | 192.168.20.21  | 255.255.255.0   | 20   | Engineering host |
 
> **Same gateways, different home.** The gateway addresses are identical to the router-on-a-stick lab, 192.168.10.1 and 192.168.20.1. The difference is where they live. Last lab they sat on router subinterfaces. In this lab they sit on SVIs inside the switch itself, and no external router exists at all in this activity.
 
---
 
## Lab Objectives
 
By the end of this lab you will be able to:
 
1. Explain what a Layer 3 switch is and how it differs from a Layer 2 switch and a router
2. Enable routing on a switch with the `ip routing` command
3. Create switch virtual interfaces (SVIs) that serve as VLAN gateways
4. Verify inter-VLAN routing performed entirely within the switch
5. Compare the Layer 3 switch approach against router-on-a-stick and explain when each fits
6. Identify and resolve three common Layer 3 switch failures
7. Apply a structured troubleshooting process to restore inter-VLAN connectivity
---
 
## 🧱 Part 1: Build It
 
### Step 1 — Create the VLANs
 
Same as always. The VLANs must exist before anything can use them.
 
```
MLS1> enable
MLS1# configure terminal
 
MLS1(config)# vlan 10
MLS1(config-vlan)# name Sales
MLS1(config-vlan)# exit
MLS1(config)# vlan 20
MLS1(config-vlan)# name Engineering
MLS1(config-vlan)# exit
```
 
### Step 2 — Configure the Access Ports
 
The host-facing ports are ordinary access ports, exactly as in the VLANs lab.
 
```
MLS1(config)# interface GigabitEthernet0/1
MLS1(config-if)# switchport mode access
MLS1(config-if)# switchport access vlan 10
MLS1(config-if)# description PC1 - Sales
MLS1(config-if)# exit
 
MLS1(config)# interface GigabitEthernet0/2
MLS1(config-if)# switchport mode access
MLS1(config-if)# switchport access vlan 20
MLS1(config-if)# description PC2 - Engineering
MLS1(config-if)# exit
```
 
### Step 3 — Enable IP Routing
 
This is the command that turns a switch into a router. Without it, the SVIs you build next will come up but will not route between each other.
 
```
! Turn the multilayer switch into a router
MLS1(config)# ip routing
```
 
> **One command, the whole lab.** I want you to configure this deliberately and remember it, because its absence is the most common Layer 3 switching failure there is. A switch with `ip routing` missing will happily host SVIs and still refuse to move a single packet between VLANs.
 
### Step 4 — Create the SVIs
 
Now create a switch virtual interface for each VLAN. Each SVI holds the gateway address for its VLAN.
 
```
! SVI for VLAN 10
MLS1(config)# interface vlan 10
MLS1(config-if)# description VLAN 10 Gateway - Sales
MLS1(config-if)# ip address 192.168.10.1 255.255.255.0
MLS1(config-if)# no shutdown
MLS1(config-if)# exit
 
! SVI for VLAN 20
MLS1(config)# interface vlan 20
MLS1(config-if)# description VLAN 20 Gateway - Engineering
MLS1(config-if)# ip address 192.168.20.1 255.255.255.0
MLS1(config-if)# no shutdown
MLS1(config-if)# exit
```
 
> **An SVI needs two things to come up.** It needs at least one active access port in its VLAN, and it needs the VLAN itself to exist and be active. If an SVI stays down, the usual cause is that no port is up in that VLAN yet. The SVI follows the health of its VLAN, which is a sensible design once you see the logic: there is no point routing for a VLAN that has nobody in it.
 
### Step 5 — Configure the Hosts
 
```
PC1:  ip 192.168.10.11 255.255.255.0 192.168.10.1
PC2:  ip 192.168.20.21 255.255.255.0 192.168.20.1
```
 
### Step 6 — Save the Configuration
 
```
MLS1# copy running-config startup-config
```
 
---
 
## 👀 Part 2: Observe It
 
### 2.1 — Confirm Routing Is Enabled and the Networks Are Known
 
```
MLS1# show ip route
```
 
**What to look for:**
- Two connected routes, `C`, for 192.168.10.0/24 and 192.168.20.0/24
- Two local routes, `L`, for the SVI addresses
- Each connected route pointing at its VLAN interface, Vlan10 or Vlan20
If you see both networks here, `ip routing` is active and the switch knows how to reach both VLANs. An empty or router-less table is your sign that `ip routing` never took effect.
 
**✏️ Prediction checkpoint:** Before you run this, predict whether the routes will point at physical interfaces or at VLAN interfaces, and why.
 
### 2.2 — Verify the SVIs Are Up
 
```
MLS1# show ip interface brief
```
 
Confirm that Vlan10 and Vlan20 both show up and up. If one is down, check that its VLAN exists and that at least one access port in that VLAN is active.
 
### 2.3 — The Headline Test: Route Between VLANs Inside the Switch
 
From PC1 in VLAN 10, ping PC2 in VLAN 20:
```
PC1> ping 192.168.20.21
```
 
This should succeed, exactly as it did with router-on-a-stick. The difference is invisible from the host's point of view and enormous in practice: no traffic ever left the switch. The routing happened internally, in hardware, with no external router involved.
 
**✏️ Think it through:** This is the same successful result you got in the router-on-a-stick lab, but the path the packet took is completely different. Describe where the routing decision happened in each case, and why the Layer 3 switch version avoids the single-link bottleneck.
 
### 2.4 — Trace the Path
 
```
PC1> traceroute 192.168.20.21
```
 
You should see a single gateway hop at 192.168.10.1 before reaching PC2, just as before. The gateway is now an SVI inside MLS1 rather than a subinterface on a separate router. Same logical hop, very different hardware reality.
 
### 2.5 — Confirm No Router Exists
 
Take a moment to appreciate what is not in this topology. There is no router node at all. Every gateway, every routing decision, and every VLAN lives inside one device. That consolidation is the entire value proposition of a Layer 3 switch.
 
---
 
## 💥 Part 3: Break It
 
> Work through each scenario one at a time. Document the symptoms fully before you move to Part 4.
 
---
 
### Break Scenario 1 — Routing Disabled
 
This is the signature Layer 3 switch failure, and the one I most want you to recognize on sight. Turn routing off:
 
```
MLS1(config)# no ip routing
```
 
Now test from PC1:
```
PC1> ping 192.168.10.1
PC1> ping 192.168.20.21
```
 
**✏️ Document the symptoms:**
- Can PC1 still reach its own gateway at 192.168.10.1?
- Can PC1 reach PC2 across VLANs?
- Run `show ip route`. What happened to the routing table?
> **Why this matters:** With routing disabled, the SVIs still exist and a host can usually still reach the SVI in its own VLAN, so the gateway appears reachable. But the switch will not move a packet from one VLAN to another, because you took away its router role. Everything looks configured, and nothing routes. When a Layer 3 switch hosts perfect SVIs and still will not route, `ip routing` is the very first thing to check. Now you have felt the symptom, so you will not forget the cause.
 
---
 
### Break Scenario 2 — A Missing or Shutdown SVI
 
Restore routing first. Then shut down the VLAN 20 SVI, simulating a gateway that was never brought up or was disabled by mistake:
 
```
MLS1(config)# ip routing
 
MLS1(config)# interface vlan 20
MLS1(config-if)# shutdown
MLS1(config-if)# exit
```
 
Now test:
```
PC2> ping 192.168.20.1
PC2> ping 192.168.10.11
PC1> ping 192.168.10.1
```
 
**✏️ Document the symptoms:**
- Can PC2 reach its gateway at 192.168.20.1 anymore?
- Can PC1 still reach its own VLAN 10 gateway?
- Why does VLAN 20 lose connectivity while VLAN 10 keeps working?
> **Why this matters:** Each SVI is the gateway for exactly one VLAN. Take one SVI down and you remove the gateway for that VLAN only, while the others carry on untouched. This is the Layer 3 switch version of a per-VLAN gateway failure, and the symptom, one VLAN dead while others are fine, points you straight at that VLAN's SVI.
 
---
 
### Break Scenario 3 — The Orphaned VLAN
 
Restore the VLAN 20 SVI. Then create the conditions where an SVI cannot come up because its VLAN has no active port. Remove PC2's port from VLAN 20 by moving it elsewhere, and watch what happens to the VLAN 20 SVI:
 
```
MLS1(config)# interface vlan 20
MLS1(config-if)# no shutdown
MLS1(config-if)# exit
 
! Move the only VLAN 20 access port into a different VLAN
MLS1(config)# interface GigabitEthernet0/2
MLS1(config-if)# switchport access vlan 10
MLS1(config-if)# exit
```
 
Now observe:
```
MLS1# show ip interface brief
MLS1# show ip route
PC2> ping 192.168.20.1
```
 
**✏️ Document the symptoms:**
- What state does the Vlan20 interface show now that no access port lives in VLAN 20?
- Did the 192.168.20.0/24 route disappear from the routing table?
- Why would the switch bring down a gateway just because its VLAN has no active members?
> **Why this matters:** An SVI stays up only while its VLAN has at least one active access port. Strand the VLAN with no members and its SVI goes down, taking the connected route with it. This catches people who configure SVIs first and wire the hosts later, then panic when the gateway will not come up. The SVI was waiting for a live port the whole time. Knowing this saves you from chasing a problem that is not really a problem, just a sequencing quirk.
 
---
 
## 🔧 Part 4: Fix It
 
---
 
### Fix Scenario 1 — Routing Disabled
 
**Your troubleshooting toolkit:**
```
show ip route               ! a near-empty table is the tell
show running-config | include ip routing
```
 
**Structured process:**
1. Note that hosts reach their own gateway but not other VLANs.
2. Run `show ip route`. If the connected VLAN routes are missing, routing is off.
3. Confirm with `show running-config | include ip routing`.
4. Enable routing and verify the routes return.
**✅ Solution:**
```
MLS1(config)# ip routing
```
 
**Verification:**
```
MLS1# show ip route
PC1> ping 192.168.20.21
```
 
---
 
### Fix Scenario 2 — A Missing or Shutdown SVI
 
**Your troubleshooting toolkit:**
```
show ip interface brief     ! find the down SVI
```
 
**Structured process:**
1. Note that one VLAN lost connectivity while others are fine.
2. Run `show ip interface brief` and look for that VLAN's SVI.
3. If the SVI is administratively down, bring it back up.
4. Verify the affected VLAN reaches its gateway again.
**✅ Solution:**
```
MLS1(config)# interface vlan 20
MLS1(config-if)# no shutdown
MLS1(config-if)# exit
```
 
**Verification:**
```
MLS1# show ip interface brief
PC2> ping 192.168.20.1
```
 
---
 
### Fix Scenario 3 — The Orphaned VLAN
 
**Your troubleshooting toolkit:**
```
show ip interface brief
show vlan brief             ! confirm which ports are in the VLAN
```
 
**Structured process:**
1. Note that the SVI is down and its connected route is gone.
2. Check `show vlan brief` to see whether any active port lives in that VLAN.
3. If the VLAN has no members, return a port to it.
4. Confirm the SVI comes up and the route returns.
**✅ Solution:**
```
MLS1(config)# interface GigabitEthernet0/2
MLS1(config-if)# switchport access vlan 20
MLS1(config-if)# exit
```
 
**Verification:**
```
MLS1# show ip interface brief
MLS1# show ip route
PC2> ping 192.168.10.11
```
 
---
 
## 💬 Reflection Questions
 
1. Explain the difference between a Layer 2 switch, a Layer 3 switch, and a router. What can a Layer 3 switch do that a Layer 2 switch cannot, and what does it share with a router?
2. The `ip routing` command is what makes inter-VLAN routing possible on a multilayer switch. Describe what the switch does differently before and after that command is entered.
3. An SVI for VLAN 10 serves as the default gateway for every host in VLAN 10. Compare this with the router-on-a-stick subinterface that did the same job in the previous lab. What is functionally the same, and what is physically different?
4. In Break Scenario 3, the VLAN 20 SVI went down once its VLAN had no active ports. Explain why a switch ties an SVI's status to whether its VLAN has live members, and why that behavior is reasonable rather than a flaw.
5. Router-on-a-stick funnels all inter-VLAN traffic across one link to an external router. A Layer 3 switch routes internally in hardware. Describe a specific scenario where this difference would noticeably affect network performance.
6. You have now built inter-VLAN routing two ways. For a small branch office with one router already on site and only two VLANs, which method would you choose and why? For a campus distribution layer with twenty VLANs and heavy east-west traffic, which would you choose and why?
---
 
## Challenge Extension (Optional)
 
1. **Add a third VLAN.** Create VLAN 30 with its own SVI and gateway, move a host into it, and confirm it routes to both other VLANs. Notice how much less work this is than adding a subinterface and a switch trunk change in the router-on-a-stick design.
2. **Add a default route.** Give MLS1 a static default route pointing at an imaginary upstream next hop, then check `show ip route` to see the gateway of last resort appear on a switch. This previews how a Layer 3 switch participates in routing beyond its own VLANs.
3. **Prove the consolidation.** Count the devices in this lab versus the router-on-a-stick lab that did the same job. Write one or two sentences on what the Layer 3 switch removed from the design and what that simplification is worth on a large network.
4. **Compare the two labs side by side.** Open your router-on-a-stick guide next to this one. List every configuration difference between the two approaches. This comparison is some of the best inter-VLAN exam review you can do, because the exam expects you to know both.
---
 
## Where This Leads
 
You have now solved inter-VLAN routing twice, and more importantly you understand the tradeoff between the two methods well enough to choose between them. Router-on-a-stick is simple and reuses a router you may already own. A Layer 3 switch consolidates everything into one device and routes at hardware speed, which is why it dominates modern campus designs. That kind of informed choice, not just knowing how but knowing when, is exactly what separates someone who passed the exam from someone who can design a network.
 
From here the switching section turns toward keeping a network with redundant links healthy. You will bundle parallel links together with EtherChannel for more bandwidth and resilience, then let Spanning Tree Protocol manage whatever redundancy remains so your loops never become outages. The switch you just turned into a router will feel a lot less mysterious as you work through those final two topics.
 
---
 
*CCNA – Build It · Observe It · Break It · Fix It series*

*Next: EtherChannel*
