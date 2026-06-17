# VLANs
 
**Build It · Observe It · Break It · Fix It**
 
---
 
## Overview
 
A switch, straight out of the box, puts every port in the same broadcast domain. Every device can reach every other device, and every broadcast reaches everyone. That is fine for a handful of machines, but it does not hold up in a real organization where you want the sales team separated from engineering, or guest traffic walled off from the servers. VLANs are how you carve one physical switch into several logical ones, and they are where the switching half of the CCNA really begins.
 
A **VLAN (Virtual Local Area Network)** is a logical grouping of switch ports into a single broadcast domain. Ports in the same VLAN behave as though they share their own private switch. Ports in different VLANs cannot reach each other through Layer 2 at all, even though they sit in the same physical chassis. That isolation is the entire point, and it is worth saying clearly up front, because the first time students see two PCs on the same switch fail to ping each other, they assume something is broken. Nothing is broken. That is a VLAN doing exactly its job.
 
In this lab you will build two VLANs on a single switch, assign access ports to them, and prove the separation works: hosts in the same VLAN reach each other, and hosts in different VLANs do not. Then you will break the configuration in three of the most common ways and troubleshoot your way back. Restoring communication between VLANs comes later, in the inter-VLAN routing labs, and by the time you get there you will understand exactly why a router is required.
 
**CCNA Exam Alignment:**
- 2.1 – Configure and verify VLANs (normal range) spanning multiple switches
- 2.1.a – Access ports
- 2.1.b – Default VLAN
- 2.1.c – Connectivity
**Estimated Time:** 60–90 minutes
 
---
 
## Key Concept: A VLAN Is a Broadcast Domain
 
Hold onto one idea and most of this lab follows from it: a VLAN is a broadcast domain. When you place a port in VLAN 10, any broadcast that arrives on that port reaches only the other ports in VLAN 10. The switch treats VLAN 20 as if it were a separate device on the far side of the room.
 
This leads to a rule that catches people off guard. Two hosts plugged into the same switch cannot communicate if they sit in different VLANs, no matter how the cables are run. Layer 2 simply will not carry a frame from one VLAN to another. Moving traffic between VLANs requires a Layer 3 device, a router or a Layer 3 switch, and that is a separate topic for a later lab.
 
> **Get ahead of the confusion:** When you reach the Observe section and watch a ping between VLAN 10 and VLAN 20 fail, resist the urge to troubleshoot it. That failure is the proof your VLANs work. I have seen students spend twenty minutes "fixing" a network that was already configured correctly, simply because they expected every host on a switch to be reachable.
 
---
 
## Topology
 
```
                 [SW1]
        IOSvL2 Layer 2 switch
     /        |         |        \
   G0/1     G0/2      G0/3      G0/0
  [PC1]    [PC2]     [PC3]     [PC4]
 VLAN 10  VLAN 10   VLAN 20   VLAN 20
 Sales     Sales    Engineering Engineering
 
  VLAN 10 (Sales):        PC1, PC2
  VLAN 20 (Engineering):  PC3, PC4
```
 
> **Full topology file:** `topology.yaml`
 
**Nodes (5 total, within CML Free Edition limit):**
- SW1 (IOSvL2 switch)
- PC1, PC2, PC3, PC4 (desktop hosts)
> **A note on port numbering:** The exact interface names depend on your IOSvL2 image. This guide uses `GigabitEthernet0/0` through `0/3` for the four host connections. Check `show ip interface brief` on your switch and adjust the port numbers to match what you actually see.
 
---
 
## Addressing Table
 
| Device | Interface | IP Address     | Subnet Mask     | VLAN | Default Gateway |
|--------|-----------|----------------|-----------------|------|-----------------|
| PC1    | NIC       | 192.168.10.11  | 255.255.255.0   | 10   | 192.168.10.1    |
| PC2    | NIC       | 192.168.10.12  | 255.255.255.0   | 10   | 192.168.10.1    |
| PC3    | NIC       | 192.168.20.21  | 255.255.255.0   | 20   | 192.168.20.1    |
| PC4    | NIC       | 192.168.20.22  | 255.255.255.0   | 20   | 192.168.20.1    |
 
> **About the default gateways:** Each VLAN maps to its own subnet, and each subnet lists a gateway address that does not exist yet. That is intentional. There is no router in this lab, so those gateways are unreachable for now, which is fine because everything you test here stays inside a single subnet. When you build inter-VLAN routing later, those gateway addresses are exactly what you will bring to life.
 
---
 
## Lab Objectives
 
By the end of this lab you will be able to:
 
1. Explain what a VLAN is and why a VLAN is a separate broadcast domain
2. Create VLANs and assign meaningful names
3. Configure switch ports as access ports and place them in a VLAN
4. Verify VLAN membership using `show vlan brief` and `show interfaces status`
5. Demonstrate that same-VLAN hosts communicate while different-VLAN hosts do not
6. Identify and resolve three common VLAN misconfigurations
7. Apply a structured troubleshooting process to restore correct VLAN membership
---
 
## 🧱 Part 1: Build It
 
### Step 1 — Create the VLANs
 
Start by creating both VLANs and giving each a name. Naming is not cosmetic. On a switch with dozens of VLANs, clear names are the difference between a five-minute change and a long, nervous afternoon.
 
```
SW1> enable
SW1# configure terminal
 
! Create VLAN 10 for the Sales department
SW1(config)# vlan 10
SW1(config-vlan)# name Sales
SW1(config-vlan)# exit
 
! Create VLAN 20 for the Engineering department
SW1(config)# vlan 20
SW1(config-vlan)# name Engineering
SW1(config-vlan)# exit
```
 
> **A VLAN must exist before a port can join it.** Creating the VLAN first is a habit worth forming now. You will see in Break Scenario 2 what happens when a port is told to join a VLAN that was never created.
 
### Step 2 — Assign Access Ports to VLAN 10
 
An **access port** carries traffic for exactly one VLAN and connects to an end device like a PC. Configure the two Sales ports:
 
```
SW1(config)# interface GigabitEthernet0/0
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# description PC1 - Sales
SW1(config-if)# exit
 
SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# description PC2 - Sales
SW1(config-if)# exit
```
 
> **Why `switchport mode access` matters:** This command forces the port into access mode and stops it from trying to negotiate a trunk. Explicitly setting the mode is a security and predictability practice. You are telling the switch precisely what this port is for rather than leaving it to dynamic negotiation.
 
### Step 3 — Assign Access Ports to VLAN 20
 
Configure the two Engineering ports the same way:
 
```
SW1(config)# interface GigabitEthernet0/2
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 20
SW1(config-if)# description PC3 - Engineering
SW1(config-if)# exit
 
SW1(config)# interface GigabitEthernet0/3
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 20
SW1(config-if)# description PC4 - Engineering
SW1(config-if)# exit
```
 
### Step 4 — Configure the Hosts
 
Set each PC according to the addressing table. The key detail is that PC1 and PC2 share the 192.168.10.0/24 subnet, while PC3 and PC4 share 192.168.20.0/24.
 
```
PC1:  ip 192.168.10.11 255.255.255.0 192.168.10.1
PC2:  ip 192.168.10.12 255.255.255.0 192.168.10.1
PC3:  ip 192.168.20.21 255.255.255.0 192.168.20.1
PC4:  ip 192.168.20.22 255.255.255.0 192.168.20.1
```
 
### Step 5 — Save the Configuration
 
```
SW1# copy running-config startup-config
```
 
---
 
## 👀 Part 2: Observe It
 
### 2.1 — View the VLAN Database
 
This is the single most important verification command in the entire lab.
 
```
SW1# show vlan brief
```
 
**What to look for:**
- VLAN 10 named Sales, with G0/0 and G0/1 listed as members
- VLAN 20 named Engineering, with G0/2 and G0/3 listed as members
- The default VLAN 1, which should now hold only the ports you did not assign
**✏️ Prediction checkpoint:** Before you run the command, write down which ports you expect to see under each VLAN. Then confirm the switch agrees with you.
 
### 2.2 — Check Port Status
 
```
SW1# show interfaces status
```
 
This shows each port's status, the VLAN it belongs to, and its speed and duplex. It is a faster way to scan port assignments than reading the full VLAN database when you only care about a few interfaces.
 
### 2.3 — Prove Same-VLAN Communication Works
 
From PC1, ping its VLAN 10 partner:
```
PC1> ping 192.168.10.12
```
 
This should succeed. PC1 and PC2 are in the same VLAN and the same subnet, so the switch forwards the frames between them normally.
 
### 2.4 — Prove Different-VLAN Communication Fails
 
From PC1, try to reach a host in VLAN 20:
```
PC1> ping 192.168.20.21
```
 
This should fail, and that failure is the headline result of the entire lab. PC1 and PC3 are on the same physical switch, but they live in different VLANs and different subnets, so Layer 2 will not carry the frame and there is no router to bridge the gap.
 
**✏️ Think it through:** Two PCs, one switch, and yet they cannot communicate. Explain to yourself exactly why, using the idea that a VLAN is a broadcast domain. If you can articulate this clearly, you understand the most important concept in the lab.
 
### 2.5 — Examine the MAC Address Table
 
After your pings, look at what the switch learned:
```
SW1# show mac address-table
```
 
**What to look for:** The MAC addresses the switch learned, and notice that each one is associated with a specific VLAN. The switch keeps its forwarding decisions VLAN-aware, which is the mechanism behind the isolation you just observed.
 
---
 
## 💥 Part 3: Break It
 
> Work through each scenario one at a time. Document the symptoms fully before you move to Part 4.
 
---
 
### Break Scenario 1 — A Port in the Wrong VLAN
 
This is the most common VLAN mistake in the real world, usually from a typo or a copied configuration. Move PC2 into the wrong VLAN:
 
```
SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# switchport access vlan 20
SW1(config-if)# exit
```
 
Now test from PC2:
```
PC2> ping 192.168.10.11
PC2> ping 192.168.20.21
```
 
**✏️ Document the symptoms:**
- Can PC2 still reach PC1, its intended Sales partner?
- Can PC2 now reach PC3 over in Engineering?
- PC2's IP address never changed, so why did its reachability change completely?
> **Why this matters:** A port in the wrong VLAN is a quietly broken thing. The cable is fine, the PC is fine, the IP is fine, and yet the user cannot reach the resources they are supposed to and may be able to reach resources they should not. `show vlan brief` exposes it in seconds once you know to look, and this scenario trains your eye to catch a member port sitting in the wrong group.
 
---
 
### Break Scenario 2 — Assigning a Port to a VLAN That Does Not Exist
 
Restore PC2 to VLAN 10 first. Then assign PC4's port to a VLAN you never created:
 
```
SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# switchport access vlan 10
SW1(config-if)# exit
 
SW1(config)# interface GigabitEthernet0/3
SW1(config-if)# switchport access vlan 99
SW1(config-if)# exit
```
 
Now observe:
```
SW1# show vlan brief
SW1# show interfaces status
PC4> ping 192.168.20.22
```
 
**✏️ Document the symptoms:**
- Does VLAN 99 appear in `show vlan brief`? Did the switch create it for you, or not?
- What status does G0/3 show now?
- Can PC4 reach anything?
> **Why this matters:** Depending on the IOS version and VTP mode, assigning a port to a missing VLAN either auto-creates the VLAN or strands the port in an inactive state. Either way, the result surprises people who assumed the switch would simply do what they meant. This scenario reinforces the habit of creating VLANs deliberately before you assign ports to them.
 
---
 
### Break Scenario 3 — Deleting a VLAN That Still Has Members
 
Restore PC4 to VLAN 20. Then delete VLAN 20 entirely while its ports are still assigned to it:
 
```
SW1(config)# interface GigabitEthernet0/3
SW1(config-if)# switchport access vlan 20
SW1(config-if)# exit
 
SW1(config)# no vlan 20
SW1(config)# exit
```
 
Now observe:
```
SW1# show vlan brief
SW1# show interfaces status
PC3> ping 192.168.20.22
```
 
**✏️ Document the symptoms:**
- What happened to G0/2 and G0/3, the two ports that were in VLAN 20?
- Can PC3 and PC4 still reach each other, even though they share a subnet?
- Why does removing the VLAN definition break hosts whose configuration you never touched?
> **Why this matters:** Deleting a VLAN does not return its ports to VLAN 1. It orphans them. The ports remember a VLAN membership that no longer exists, and they go inactive until you either recreate the VLAN or reassign the ports. This is a genuinely confusing failure the first time you meet it, and meeting it here in a lab is far better than meeting it on a production switch during a change window.
 
---
 
## 🔧 Part 4: Fix It
 
---
 
### Fix Scenario 1 — A Port in the Wrong VLAN
 
**Your troubleshooting toolkit:**
```
show vlan brief
show interfaces status
```
 
**Structured process:**
1. Note the symptom. A host cannot reach its expected peers and may reach unexpected ones.
2. Run `show vlan brief` and find the port in question.
3. Confirm which VLAN the port should belong to using your documentation or addressing table.
4. Reassign the port to the correct VLAN and verify.
**✅ Solution:**
```
SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# switchport access vlan 10
SW1(config-if)# exit
```
 
**Verification:**
```
SW1# show vlan brief
PC2> ping 192.168.10.11
```
 
---
 
### Fix Scenario 2 — A Port Assigned to a Missing VLAN
 
**Your troubleshooting toolkit:**
```
show vlan brief
show interfaces status
```
 
**Structured process:**
1. Notice the port is inactive or sitting in an unexpected VLAN.
2. Check `show vlan brief` to see whether the target VLAN actually exists.
3. Decide the correct fix. Either create the intended VLAN or reassign the port to the right one.
4. Apply and verify.
**✅ Solution (reassign the port to the correct existing VLAN):**
```
SW1(config)# interface GigabitEthernet0/3
SW1(config-if)# switchport access vlan 20
SW1(config-if)# exit
```
 
**Verification:**
```
SW1# show vlan brief
PC4> ping 192.168.20.21
```
 
---
 
### Fix Scenario 3 — A Deleted VLAN
 
**Your troubleshooting toolkit:**
```
show vlan brief
show interfaces status
```
 
**Structured process:**
1. Note that multiple ports went inactive at once, which points to a VLAN-level problem rather than a single port.
2. Run `show vlan brief` and confirm the VLAN those ports expect is missing.
3. Recreate the VLAN, which immediately reactivates the orphaned ports.
4. Verify the ports return to service.
**✅ Solution:**
```
SW1(config)# vlan 20
SW1(config-vlan)# name Engineering
SW1(config-vlan)# exit
```
 
**Verification:**
```
SW1# show vlan brief
PC3> ping 192.168.20.22
```
 
---
 
## 💬 Reflection Questions
 
1. Explain in your own words why two PCs connected to the same switch cannot communicate when they are in different VLANs. Use the idea that a VLAN is a broadcast domain.
2. In Break Scenario 1, PC2's IP address never changed, yet moving its port to a different VLAN completely changed what it could reach. What does this tell you about the relationship between a port's VLAN membership and a host's IP configuration?
3. Each VLAN in this lab maps to its own subnet, VLAN 10 to 192.168.10.0/24 and VLAN 20 to 192.168.20.0/24. Why is it standard practice to give each VLAN its own subnet rather than sharing one subnet across VLANs?
4. In Break Scenario 3, deleting VLAN 20 did not move its ports to VLAN 1. Instead the ports went inactive. Why do you think IOS handles it this way rather than quietly reassigning the ports to the default VLAN?
5. The default VLAN is VLAN 1, and every unassigned port belongs to it. From a security standpoint, why is it considered poor practice to leave production devices in VLAN 1?
6. You named your VLANs Sales and Engineering rather than leaving them unnamed. On a switch carrying forty VLANs, how does consistent naming change the experience of troubleshooting a problem at 2 a.m.?
---
 
## Challenge Extension (Optional)
 
1. **Add a management SVI.** Create a switch virtual interface for VLAN 10 with `interface vlan 10` and give it an IP address. From PC1, ping that SVI. Why can PC1 reach it while a host in VLAN 20 cannot?
2. **Explore the default VLAN.** Move one host's port back to VLAN 1 and observe what it can and cannot reach. Then research why network engineers deliberately avoid using VLAN 1 for user traffic.
3. **Predict before you assign.** Without running any show commands, reassign two ports to swap their VLANs, write down exactly what each host should now be able to reach, and only then verify with pings. Getting your prediction right is a strong sign the concept has clicked.
4. **Look ahead to the next problem.** You proved VLAN 10 and VLAN 20 cannot communicate. Sketch out what you think it would take to let them talk to each other. You are describing inter-VLAN routing, and your sketch is a preview of the next labs in this section.
---
 
## Where This Leads
 
You have built the foundation that every other switching topic rests on. VLANs are the reason trunking exists, the reason inter-VLAN routing exists, and a major part of why spanning tree has to work as hard as it does. Each of those topics is, in one way or another, a response to a problem that VLANs introduce.
 
The immediate next question almost asks itself. Right now your two VLANs live on a single switch. What happens when VLAN 10 needs to span two switches across the building? Carrying multiple VLANs over a single link between switches is the job of a trunk, and that is exactly where the next lab picks up.
 
---
 
*CCNA – Build It · Observe It · Break It · Fix It series*
*Next: Trunking*
