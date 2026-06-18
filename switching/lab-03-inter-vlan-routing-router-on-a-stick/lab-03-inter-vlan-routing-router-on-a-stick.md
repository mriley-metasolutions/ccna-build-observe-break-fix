# Inter-VLAN Routing: Router-on-a-Stick
 
**Build It · Observe It · Break It · Fix It**
 
---
 
## Overview
 
Two labs ago you configured some VLANs and proved they keep traffic separate. In the previous lab you built a trunk and proved that separation holds even when VLANs share a cable. Both times you ended at the same wall: VLAN 10 and VLAN 20 cannot talk to each other, by design, because a switch will not move a frame from one VLAN to another. That separation is exactly what you wanted, right up until the moment Sales needs to reach a server that lives over in the Engineering VLAN. Then you need a way back across the boundary, and crossing a boundary between subnets is job of a router.
 
This lab introduces the first and most economical way to route between VLANs, a design known affectionately as **router-on-a-stick**. The idea is simple and a little clever. You connect a single router interface to the switch with one cable, you make that cable a trunk so it carries every VLAN, and then you divide the router's one physical interface into several **subinterfaces**, one per VLAN. Each subinterface tags its traffic for a specific VLAN and holds the IP address that the hosts in that VLAN use as their default gateway. One physical link, one router interface, and suddenly every VLAN has a gateway and can reach every other VLAN.
 
This is also the moment I have been pointing you toward since the VLANs lab. Those default gateway addresses you configured on your hosts and then could not reach, 192.168.10.1 and 192.168.20.1, are about to become real. By the end of this lab, a ping from VLAN 10 to VLAN 20 will succeed, and you will be able to trace exactly how it crossed the boundary.
 
**CCNA Exam Alignment:**
- 2.2 – Configure and verify interswitch connectivity (trunk ports, 802.1Q)
- 1.1 – Compare network components and their roles in inter-VLAN routing
- Builds directly on the routing fundamentals from Section 3

**Estimated Time:** 60–90 minutes
 
---
 
## Key Concept: One Interface, Many Gateways
 
A router normally needs a separate physical interface for every network it routes between. Router-on-a-stick gets around that by splitting a single physical interface into logical subinterfaces, written with a dot and a number, such as `GigabitEthernet0/0.10`. Each subinterface behaves like an independent interface with its own IP address and its own VLAN tag.
 
Two pieces have to line up for this to work, and they are the two pieces students most often miss:
 
- The **subinterface number** is just a label, but the **`encapsulation dot1q` VLAN ID** is what actually matters. That encapsulation command tells the subinterface which VLAN's tagged traffic it should claim. The convention is to make the subinterface number match the VLAN ID, which keeps your configuration readable, but it is the encapsulation command that does the real work.
- The **switch port facing the router must be a trunk**. The router is expecting tagged frames for multiple VLANs, and only a trunk delivers those. An access port would hand the router traffic for a single VLAN with no tags, and the design falls apart.
> **Get ahead of the confusion:** When inter-VLAN routing fails in this lab, the cause is almost always one of those two things. Either a subinterface is tagged for the wrong VLAN, or the switch port to the router is not trunking. Keep both in mind and you will diagnose every failure in the Break It section quickly.
 
---
 
## Topology
 
```
        [PC1]                  [PC2]
       VLAN 10               VLAN 20
      192.168.10.11        192.168.20.21
         |                     |
        G0/1                 G0/2
              [SW1]
               G0/0
                |
            trunk link
                |
               G0/0  (physical, no IP)
              [R1]
        G0/0.10 -> 192.168.10.1 (VLAN 10 gateway)
        G0/0.20 -> 192.168.20.1 (VLAN 20 gateway)
```
 
**Nodes (4 total, within CML Free Edition limit):**
- R1 (IOSv router)
- SW1 (IOSvL2 switch)
- PC1 (VLAN 10), PC2 (VLAN 20)
> **A note on interface names:** As in the other switching labs, port names vary by image. Confirm with `show ip interface brief` and adjust. The router-to-switch link uses G0/0 on both ends in this guide.
 
---
 
## Addressing Table
 
| Device | Interface    | IP Address     | Subnet Mask     | VLAN | Role |
|--------|--------------|----------------|-----------------|------|------|
| R1     | G0/0         | (no IP)        | —               | —    | Trunk to SW1 |
| R1     | G0/0.10      | 192.168.10.1   | 255.255.255.0   | 10   | VLAN 10 gateway |
| R1     | G0/0.20      | 192.168.20.1   | 255.255.255.0   | 20   | VLAN 20 gateway |
| PC1    | NIC          | 192.168.10.11  | 255.255.255.0   | 10   | Sales host |
| PC2    | NIC          | 192.168.20.21  | 255.255.255.0   | 20   | Engineering host |
 
> **Notice the gateways.** R1's subinterfaces hold 192.168.10.1 and 192.168.20.1, the exact addresses your hosts have been pointing at as their default gateway since the VLANs lab. The physical interface G0/0 deliberately has no IP of its own. It exists only to carry the subinterfaces.
 
---
 
## Lab Objectives
 
By the end of this lab you will be able to:
 
1. Explain why moving traffic between VLANs requires a Layer 3 device
2. Describe how router-on-a-stick uses subinterfaces to route between VLANs over one link
3. Configure router subinterfaces with 802.1Q encapsulation
4. Configure the switch port facing the router as a trunk
5. Verify inter-VLAN routing with `show ip route` and end-to-end pings
6. Identify and resolve three common router-on-a-stick failures
7. Apply a structured troubleshooting process to restore inter-VLAN connectivity
---
 
## 🧱 Part 1: Build It
 
### Step 1 — Configure the Switch
 
Create the VLANs, assign the host access ports, and make the port facing the router a trunk. This should feel familiar from the trunking lab, with one difference: the trunk now points at a router rather than another switch.
 
```
SW1> enable
SW1# configure terminal
 
! Create both VLANs
SW1(config)# vlan 10
SW1(config-vlan)# name Sales
SW1(config-vlan)# exit
SW1(config)# vlan 20
SW1(config-vlan)# name Engineering
SW1(config-vlan)# exit
 
! Access port for PC1 in VLAN 10
SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# description PC1 - Sales
SW1(config-if)# exit
 
! Access port for PC2 in VLAN 20
SW1(config)# interface GigabitEthernet0/2
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 20
SW1(config-if)# description PC2 - Engineering
SW1(config-if)# exit
 
! Trunk port facing the router
SW1(config)# interface GigabitEthernet0/0
SW1(config-if)# switchport mode trunk
SW1(config-if)# description Trunk to R1 (router-on-a-stick)
SW1(config-if)# exit
```
 
### Step 2 — Configure the Router Physical Interface
 
The physical interface carries the subinterfaces but holds no IP address of its own. Its only job here is to come up.
 
```
R1> enable
R1# configure terminal
 
! Bring up the physical interface. No IP address goes here.
R1(config)# interface GigabitEthernet0/0
R1(config-if)# description Trunk to SW1 (router-on-a-stick)
R1(config-if)# no shutdown
R1(config-if)# exit
```
 
> **Do not skip the `no shutdown` on the physical interface.** The subinterfaces depend on it. If the physical interface is down, every subinterface riding on it is down too, no matter how perfectly you configure them. You will see exactly this in Break Scenario 3.
 
### Step 3 — Configure the Subinterfaces
 
Now create one subinterface per VLAN. Each one gets an 802.1Q tag matching its VLAN and the IP address that serves as that VLAN's gateway.
 
```
! Subinterface for VLAN 10
R1(config)# interface GigabitEthernet0/0.10
R1(config-subif)# encapsulation dot1q 10
R1(config-subif)# ip address 192.168.10.1 255.255.255.0
R1(config-subif)# exit
 
! Subinterface for VLAN 20
R1(config)# interface GigabitEthernet0/0.20
R1(config-subif)# encapsulation dot1q 20
R1(config-subif)# ip address 192.168.20.1 255.255.255.0
R1(config-subif)# exit
```
 
> **The `encapsulation dot1q` command is the heart of this lab.** It binds the subinterface to a VLAN. When a tagged frame for VLAN 10 arrives from the trunk, the router hands it to G0/0.10 because that subinterface claimed VLAN 10. Get this number wrong and the router will route for a VLAN that no host is actually using, which is the single most common router-on-a-stick mistake.
 
### Step 4 — Configure the Hosts
 
```
PC1:  ip 192.168.10.11 255.255.255.0 192.168.10.1
PC2:  ip 192.168.20.21 255.255.255.0 192.168.20.1
```
 
### Step 5 — Save Both Configurations
 
```
R1# copy running-config startup-config
SW1# copy running-config startup-config
```
 
---
 
## 👀 Part 2: Observe It
 
### 2.1 — Confirm the Router Learned Both Networks
 
```
R1# show ip route
```
 
**What to look for:**
- Two connected routes, `C`, one for 192.168.10.0/24 and one for 192.168.20.0/24
- Two local routes, `L`, for the subinterface addresses themselves
- Each connected route associated with its subinterface, G0/0.10 or G0/0.20
The router now has a route to both VLAN subnets, which is what lets it move traffic between them. Seeing both networks here is your first confirmation the design is working.
 
**✏️ Prediction checkpoint:** Before you run the command, predict how many connected routes you expect and which subinterface each will point to.
 
### 2.2 — Verify the Subinterfaces Are Up
 
```
R1# show ip interface brief
```
 
Confirm that G0/0, G0/0.10, and G0/0.20 all show up and up. Notice that the subinterfaces depend on the physical G0/0 being up, which is why the physical interface had to be enabled first.
 
### 2.3 — Verify the Trunk on the Switch
 
```
SW1# show interfaces trunk
```
 
Confirm G0/0 is trunking and that both VLAN 10 and VLAN 20 are allowed and active. This is the switch side of the router-on-a-stick handshake.
 
### 2.4 — The Headline Test: Route Between VLANs
 
From PC1 in VLAN 10, ping PC2 in VLAN 20:
```
PC1> ping 192.168.20.21
```
 
This should succeed, and it is the result the entire switching section has been building toward. Two hosts in different VLANs, on different subnets, are now communicating, and they are doing it through R1.
 
**✏️ Think it through:** In the VLANs lab this exact ping failed, and that failure was correct. Now it succeeds. Explain what changed. What is the router doing for this packet that the switch alone could not do?
 
### 2.5 — Trace the Path
 
```
PC1> traceroute 192.168.20.21
```
 
You should see the packet pass through the gateway, 192.168.10.1, on its way to PC2. That gateway hop is the router doing its job, accepting the frame on the VLAN 10 subinterface and forwarding it out the VLAN 20 subinterface. Seeing the router appear in the path makes the whole design concrete.
 
---
 
## 💥 Part 3: Break It
 
> Work through each scenario one at a time. Document the symptoms fully before you move to Part 4.
 
---
 
### Break Scenario 1 — The Encapsulation VLAN Mismatch
 
This is the defining router-on-a-stick mistake. Change the VLAN 20 subinterface so it tags for the wrong VLAN:
 
```
R1(config)# interface GigabitEthernet0/0.20
R1(config-subif)# encapsulation dot1q 99
R1(config-subif)# exit
```
 
> The subinterface still holds 192.168.20.1, but it now claims VLAN 99 instead of VLAN 20.
 
Now test from PC2:
```
PC2> ping 192.168.20.1
PC2> ping 192.168.10.11
```
 
**✏️ Document the symptoms:**
- Can PC2 reach its own gateway at 192.168.20.1 anymore?
- Can PC2 reach PC1 over in VLAN 10?
- The subinterface still shows up in `show ip interface brief`, so why has VLAN 20 lost its gateway?
> **Why this matters:** The subinterface looks healthy. It has an IP, it is up, and `show ip route` still lists 192.168.20.0/24. But because it is now tagged for VLAN 99, the tagged VLAN 20 frames arriving from the trunk have no subinterface to land on. The gateway is effectively listening on the wrong channel. This is why I keep pointing at the encapsulation command. It is invisible in a quick interface check and it is the first thing to verify when one VLAN loses its gateway.
 
---
 
### Break Scenario 2 — The Switch Port Is Not a Trunk
 
Restore the VLAN 20 subinterface to its correct VLAN. Then change the switch port facing the router from a trunk back to an access port:
 
```
R1(config)# interface GigabitEthernet0/0.20
R1(config-subif)# encapsulation dot1q 20
R1(config-subif)# exit
 
SW1(config)# interface GigabitEthernet0/0
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# exit
```
 
> The router still expects tagged traffic for both VLANs, but the switch now sends only untagged VLAN 10 traffic.
 
Now test:
```
PC1> ping 192.168.20.21
PC1> ping 192.168.10.1
PC2> ping 192.168.10.1
```
 
**✏️ Document the symptoms:**
- Can PC1 reach its own gateway? Can PC2 reach its gateway?
- Why might one VLAN appear to partly work while the other fails completely?
- What does `show interfaces trunk` on SW1 show now?
> **Why this matters:** Router-on-a-stick depends entirely on the switch delivering tagged frames for every VLAN. The instant that port stops trunking, the router stops receiving the tags it needs, and inter-VLAN routing collapses. This scenario reinforces that router-on-a-stick is a two-sided agreement. The router subinterfaces and the switch trunk have to be configured in concert, and checking only the router side will leave you puzzled.
 
---
 
### Break Scenario 3 — The Physical Interface Is Down
 
Restore the switch trunk. Then shut down the router's physical interface, the one the subinterfaces ride on:
 
```
SW1(config)# interface GigabitEthernet0/0
SW1(config-if)# switchport mode trunk
SW1(config-if)# exit
 
R1(config)# interface GigabitEthernet0/0
R1(config-if)# shutdown
R1(config-if)# exit
```
 
Now observe:
```
R1# show ip interface brief
PC1> ping 192.168.20.21
PC1> ping 192.168.10.1
```
 
**✏️ Document the symptoms:**
- What state do G0/0.10 and G0/0.20 show now, even though you never touched them directly?
- Can either VLAN reach its gateway?
- What does this tell you about the relationship between a physical interface and the subinterfaces built on it?
> **Why this matters:** Subinterfaces are not independent. They inherit the state of the physical interface they live on. Shut the physical interface and every subinterface goes down with it, which means both VLANs lose their gateway at once. When all inter-VLAN routing fails simultaneously, the physical interface is the first place to look, because a single down interface explains every symptom.
 
---
 
## 🔧 Part 4: Fix It
 
---
 
### Fix Scenario 1 — The Encapsulation VLAN Mismatch
 
**Your troubleshooting toolkit:**
```
show running-config | section GigabitEthernet0/0.20
show ip interface brief
```
 
**Structured process:**
1. Note that one VLAN lost its gateway while the other still works.
2. Look at the running config for the failing VLAN's subinterface.
3. Compare the `encapsulation dot1q` VLAN ID against the VLAN the hosts actually use.
4. Correct the encapsulation to match and verify.
**✅ Solution:**
```
R1(config)# interface GigabitEthernet0/0.20
R1(config-subif)# encapsulation dot1q 20
R1(config-subif)# exit
```
 
**Verification:**
```
PC2> ping 192.168.20.1
PC2> ping 192.168.10.11
```
 
---
 
### Fix Scenario 2 — The Switch Port Is Not a Trunk
 
**Your troubleshooting toolkit:**
```
show interfaces trunk       ! on the switch
show ip route               ! on the router
```
 
**Structured process:**
1. Note that inter-VLAN routing broke for at least one VLAN.
2. Check the switch port facing the router. Is it still a trunk?
3. If it reverted to access, set it back to trunk so it delivers tagged frames again.
4. Verify both VLANs reach their gateways.
**✅ Solution:**
```
SW1(config)# interface GigabitEthernet0/0
SW1(config-if)# switchport mode trunk
SW1(config-if)# exit
```
 
**Verification:**
```
SW1# show interfaces trunk
PC1> ping 192.168.20.21
```
 
---
 
### Fix Scenario 3 — The Physical Interface Is Down
 
**Your troubleshooting toolkit:**
```
show ip interface brief
```
 
**Structured process:**
1. Note that both VLANs lost their gateway at the same time, which points to a shared cause.
2. Run `show ip interface brief` and check the physical interface state.
3. If the physical interface is administratively down, bring it back up.
4. Confirm the subinterfaces follow it up and routing returns.
**✅ Solution:**
```
R1(config)# interface GigabitEthernet0/0
R1(config-if)# no shutdown
R1(config-if)# exit
```
 
**Verification:**
```
R1# show ip interface brief
PC1> ping 192.168.20.21
```
 
---
 
## 💬 Reflection Questions
 
1. Explain in your own words why a switch alone cannot move traffic between VLAN 10 and VLAN 20, and why a router can.
2. The router's physical interface G0/0 has no IP address, yet the design depends on it. What is the physical interface's role, and why do the subinterfaces carry the IP addresses instead?
3. The `encapsulation dot1q 10` command is what binds a subinterface to a VLAN. Walk through what happens to a tagged VLAN 10 frame as it arrives from the trunk and reaches the router. How does the router know which subinterface should handle it?
4. In Break Scenario 1, the subinterface still showed as up with a valid IP, yet VLAN 20 lost its gateway. Explain why a healthy-looking subinterface can still fail to serve its VLAN.
5. Router-on-a-stick sends all inter-VLAN traffic across a single physical link between the switch and the router. On a busy network with many VLANs, what limitation does that single link create? This limitation is a major reason the next lab exists.
6. In Break Scenario 3, shutting one physical interface took down both VLAN gateways at once. What does this teach you about diagnosing a failure where every VLAN loses connectivity simultaneously versus only one VLAN failing?
---
 
## Challenge Extension (Optional)
 
1. **Add a third VLAN.** Create VLAN 30 on the switch, add a matching subinterface on the router with its own encapsulation and gateway, and confirm a host in VLAN 30 can reach both other VLANs. This shows how router-on-a-stick scales by simply adding subinterfaces.
2. **Watch the tag drive the decision.** Deliberately swap the encapsulation IDs on the two subinterfaces so G0/0.10 tags for VLAN 20 and G0/0.20 tags for VLAN 10. Predict what each host will be able to reach before you test, then verify. This drives home that the encapsulation command, not the subinterface number, controls everything.
3. **Compare the gateway behavior.** From PC1, ping 192.168.10.1 and then 192.168.20.1. Both are on the same router. Explain why PC1 can reach both gateway addresses and what path each ping takes.
4. **Look ahead.** Router-on-a-stick funnels every VLAN through one router link. Sketch how you might instead push the routing function directly into the switch so traffic never has to leave for a separate router. You are describing a Layer 3 switch, which is the next lab.
---
 
## Where This Leads
 
You have closed the loop the switching section opened. VLANs separated your traffic, trunking carried that separation between switches, and now router-on-a-stick reconnects the VLANs on your terms, through a router that enforces a clean Layer 3 boundary between them. The gateway addresses that sat unreachable for two labs are live, and you can trace a packet across the VLAN boundary and name every step.
 
Router-on-a-stick has one real weakness, and you may have already felt it: every bit of inter-VLAN traffic crosses a single link between the switch and the router. On a small network that is fine. On a busy one it becomes a bottleneck. The next lab solves that by moving the routing into the switch itself with switch virtual interfaces, and doing the two labs back to back is the clearest way to understand why a Layer 3 switch became the standard answer in modern networks.
 
---
 
*CCNA – Build It · Observe It · Break It · Fix It series*

*Next: Inter-VLAN Routing (Layer 3 Switch)*
