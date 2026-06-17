# Trunking
 
**Build It · Observe It · Break It · Fix It**
 
---
 
## Overview
 
In the VLANs lab you divided a single switch into separate broadcast domains, and everything stayed neatly inside that one switch. Real networks are not that tidy. The Sales team is spread across two floors, each floor has its own switch, and VLAN 10 needs to exist on both. The moment a VLAN has to span more than one switch, you need a way to carry several VLANs across the single cable that connects those switches. That cable is a trunk, and trunking is the bridge between the small, single-switch world and an actual building network.
 
An **access port** carries traffic for exactly one VLAN and connects to an end device. A **trunk port** carries traffic for many VLANs at once and connects switches to each other. The trick that makes a trunk work is **802.1Q tagging**. As a frame crosses the trunk, the switch inserts a small tag that records which VLAN the frame belongs to, so the switch on the far end knows precisely where to deliver it. The tag is added when the frame enters the trunk and stripped off when it leaves, so the end devices never see it and never need to know it exists.
 
In this lab you will connect two switches with an 802.1Q trunk, extend two VLANs across it, and prove that a host on one switch can reach its VLAN partner on the other while traffic between different VLANs stays separated. Then you will break the trunk in the three ways trunks most often fail in the field and troubleshoot each one back to health.
 
**CCNA Exam Alignment:**
- 2.2 – Configure and verify interswitch connectivity
- 2.2.a – Trunk ports
- 2.2.b – 802.1Q
- 2.2.c – Native VLAN

**Estimated Time:** 60–90 minutes
 
---
 
## Key Concept: The Native VLAN
 
Most of trunking is straightforward once you accept that a trunk tags frames with their VLAN ID. There is one exception worth understanding before you build, because it is both an exam favorite and a real-world trap: the native VLAN.
 
On an 802.1Q trunk, exactly one VLAN travels **untagged**. That is the native VLAN, and by default it is VLAN 1. Any frame belonging to the native VLAN crosses the trunk with no tag at all, and the receiving switch assumes that any untagged frame it sees must belong to its own native VLAN. This is why both ends of a trunk must agree on which VLAN is native. If SW1 calls VLAN 1 native and SW2 calls VLAN 99 native, each switch will misfile the other's untagged traffic, and you get a quiet, hard-to-spot leak between VLANs.
 
> **Get ahead of the confusion:** A native VLAN mismatch does not break the whole trunk. Tagged VLANs keep working fine, which is exactly what makes it sneaky. You will see this firsthand in Break Scenario 2, and the lesson is to check the native VLAN even when most traffic appears healthy.
 
---
 
## Topology
 
```
   [PC1]            [PC2]                    [PC3]
  VLAN 10          VLAN 20                  VLAN 10
  Sales         Engineering                 Sales
    |                |                         |
  G0/1             G0/2                      G0/1
        [SW1] ===== trunk ===== [SW2]
              G0/0          G0/0
 
  Trunk link: SW1 G0/0 <-> SW2 G0/0 (carries VLAN 10 and VLAN 20)
  VLAN 10 (Sales):        PC1 on SW1, PC3 on SW2
  VLAN 20 (Engineering):  PC2 on SW1
```

**Nodes (5 total, within CML Free Edition limit):**
- SW1, SW2 (IOSvL2 switches)
- PC1, PC2 (hosts on SW1)
- PC3 (host on SW2)
> **A note on interface names and encapsulation:** Port names vary by IOSvL2 image, so check `show ip interface brief` and adjust to match what you see. Also, some Cisco switch platforms require `switchport trunk encapsulation dot1q` before you can set trunk mode, while IOSvL2 typically defaults to dot1q and does not need it. If your switch rejects the trunk mode command, add the encapsulation command first.
 
---
 
## Addressing Table
 
| Device | Interface | IP Address     | Subnet Mask     | VLAN | Switch |
|--------|-----------|----------------|-----------------|------|--------|
| PC1    | NIC       | 192.168.10.11  | 255.255.255.0   | 10   | SW1    |
| PC2    | NIC       | 192.168.20.21  | 255.255.255.0   | 20   | SW1    |
| PC3    | NIC       | 192.168.10.12  | 255.255.255.0   | 10   | SW2    |
 
> **The design in one sentence:** PC1 and PC3 share VLAN 10 but live on different switches, so the only way they can reach each other is across a working trunk. That makes them your proof that trunking succeeded.
 
---
 
## Lab Objectives
 
By the end of this lab you will be able to:
 
1. Explain the difference between an access port and a trunk port
2. Describe how 802.1Q tagging carries multiple VLANs over one link
3. Configure and verify an 802.1Q trunk between two switches
4. Explain the role of the native VLAN and why both ends must agree on it
5. Confirm that a VLAN can span two switches across a trunk
6. Identify and resolve three common trunking failures
7. Apply a structured troubleshooting process to restore interswitch connectivity
---
 
## 🧱 Part 1: Build It
 
### Step 1 — Create the VLANs on Both Switches
 
A trunk can only carry VLANs that exist on each switch. This is the most common reason a perfectly configured trunk still fails to pass a VLAN, so build the VLANs on both ends first.
 
**On SW1:**
```
SW1> enable
SW1# configure terminal
SW1(config)# vlan 10
SW1(config-vlan)# name Sales
SW1(config-vlan)# exit
SW1(config)# vlan 20
SW1(config-vlan)# name Engineering
SW1(config-vlan)# exit
```
 
**On SW2:**
```
SW2> enable
SW2# configure terminal
SW2(config)# vlan 10
SW2(config-vlan)# name Sales
SW2(config-vlan)# exit
SW2(config)# vlan 20
SW2(config-vlan)# name Engineering
SW2(config-vlan)# exit
```
 
> **Why VLAN 20 on SW2 even though no host uses it there:** Creating both VLANs on both switches is a good habit and keeps the trunk consistent. It also sets you up cleanly for the day you add an Engineering host to SW2 without having to remember a missing VLAN.
 
### Step 2 — Configure the Access Ports
 
These are the ports facing the PCs, configured exactly as you did in the VLANs lab.
 
**On SW1:**
```
SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# description PC1 - Sales
SW1(config-if)# exit
 
SW1(config)# interface GigabitEthernet0/2
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 20
SW1(config-if)# description PC2 - Engineering
SW1(config-if)# exit
```
 
**On SW2:**
```
SW2(config)# interface GigabitEthernet0/1
SW2(config-if)# switchport mode access
SW2(config-if)# switchport access vlan 10
SW2(config-if)# description PC3 - Sales
SW2(config-if)# exit
```
 
### Step 3 — Configure the Trunk
 
Now configure the link between the two switches as a trunk. Both ends must be set to trunk mode.
 
**On SW1:**
```
SW1(config)# interface GigabitEthernet0/0
SW1(config-if)# switchport mode trunk
SW1(config-if)# description Trunk to SW2
SW1(config-if)# exit
```
 
**On SW2:**
```
SW2(config)# interface GigabitEthernet0/0
SW2(config-if)# switchport mode trunk
SW2(config-if)# description Trunk to SW1
SW2(config-if)# exit
```
 
> **A word on trunk negotiation:** Older configurations relied on Dynamic Trunking Protocol to negotiate trunk status automatically. Setting `switchport mode trunk` on both ends statically is cleaner and more predictable, and statically configured trunks are what I want you to reach for by default. You will see in Break Scenario 1 what happens when the two ends do not agree.
 
### Step 4 — Configure the Hosts
 
Set each PC according to the addressing table.
 
```
PC1:  ip 192.168.10.11 255.255.255.0 192.168.10.1
PC2:  ip 192.168.20.21 255.255.255.0 192.168.20.1
PC3:  ip 192.168.10.12 255.255.255.0 192.168.10.1
```
 
### Step 5 — Save Both Configurations
 
```
SW1# copy running-config startup-config
SW2# copy running-config startup-config
```
 
---
 
## 👀 Part 2: Observe It
 
### 2.1 — Verify the Trunk
 
This is the command you will live in for the rest of this lab.
 
```
SW1# show interfaces trunk
```
 
**What to look for:**
- The trunk port G0/0 listed with its mode and status
- The encapsulation in use, which should be 802.1Q
- The native VLAN, which defaults to 1
- The VLANs allowed and active on the trunk, which should include 10 and 20
Run the same command on SW2 and confirm both ends agree.
 
**✏️ Prediction checkpoint:** Before you run the command, predict which VLANs you expect to see allowed on the trunk and what the native VLAN will be. Then check.
 
### 2.2 — Confirm a VLAN Spans Both Switches
 
This is the headline test of the lab. From PC1 on SW1, ping PC3 on SW2:
```
PC1> ping 192.168.10.12
```
 
This should succeed. PC1 and PC3 are in VLAN 10 on different switches, and the trunk is carrying their tagged traffic between the two. You have just extended a single broadcast domain across two physical switches.
 
### 2.3 — Confirm Isolation Still Holds Across the Trunk
 
From PC1 in VLAN 10, try to reach PC2 in VLAN 20:
```
PC1> ping 192.168.20.21
```
 
This should fail. The trunk carries both VLANs, but it keeps them tagged and separate. A trunk does not merge VLANs, it multiplexes them, so the isolation you built in the VLANs lab survives the trip across the trunk.
 
**✏️ Think it through:** The trunk carries VLAN 10 and VLAN 20 over the same physical cable, yet VLAN 10 and VLAN 20 still cannot talk. Explain how 802.1Q tagging makes both of those statements true at once.
 
### 2.4 — Examine VLAN Membership
 
```
SW1# show vlan brief
```
 
Notice that the trunk port does not appear as a member of any single VLAN the way the access ports do. A trunk belongs to all of its allowed VLANs at once rather than to one, which is the whole reason it exists.
 
---
 
## 💥 Part 3: Break It
 
> Work through each scenario one at a time. Document the symptoms fully before you move to Part 4.
 
---
 
### Break Scenario 1 — The Trunk That Will Not Form
 
A trunk needs both ends to agree. Change SW2's end of the link back to an access port while SW1 still thinks it is a trunk:
 
```
SW2(config)# interface GigabitEthernet0/0
SW2(config-if)# switchport mode access
SW2(config-if)# exit
```
 
Now observe:
```
SW1# show interfaces trunk
SW2# show interfaces trunk
PC1> ping 192.168.10.12
```
 
**✏️ Document the symptoms:**
- Does G0/0 still appear as a trunk on SW1? On SW2?
- Can PC1 still reach PC3 across the link?
- What does the mismatch do to traffic that was working a moment ago?
> **Why this matters:** A trunk is an agreement between two switches, and when one side stops trunking, VLAN traffic stops crossing the link. The `show interfaces trunk` output on each switch tells you immediately whether both ends consider the link a trunk, which makes this one of the fastest faults to confirm once you know to compare both sides.
 
---
 
### Break Scenario 2 — The Native VLAN Mismatch
 
Restore SW2's trunk first. Then change the native VLAN on only one end:
 
```
SW2(config)# interface GigabitEthernet0/0
SW2(config-if)# switchport mode trunk
SW2(config-if)# exit
 
SW1(config)# interface GigabitEthernet0/0
SW1(config-if)# switchport trunk native vlan 99
SW1(config-if)# exit
```
 
> SW1 now calls VLAN 99 native while SW2 still calls VLAN 1 native.
 
Now observe:
```
SW1# show interfaces trunk
SW1# show logging
PC1> ping 192.168.10.12
```
 
**✏️ Document the symptoms:**
- Does the trunk still pass tagged VLAN 10 traffic? Can PC1 still reach PC3?
- What does the native VLAN column show on each switch?
- Check `show logging`. Does the switch report a native VLAN mismatch, and what protocol detected it?
> **Why this matters:** This is the sneaky one I warned you about. Tagged traffic keeps flowing, so PC1 to PC3 may still work, and a casual check makes the trunk look healthy. Meanwhile untagged traffic is being misfiled between two different VLANs, which is both a reliability problem and a security risk. Cisco Discovery Protocol usually catches the mismatch and logs it, and that log message is the clue most people miss because they were not looking for a problem in the first place.
 
---
 
### Break Scenario 3 — A Pruned VLAN
 
Restore SW1's native VLAN to the default. Then restrict the trunk so it no longer allows VLAN 10:
 
```
SW1(config)# interface GigabitEthernet0/0
SW1(config-if)# switchport trunk native vlan 1
SW1(config-if)# switchport trunk allowed vlan 20
SW1(config-if)# exit
```
 
> The `allowed vlan 20` command replaces the allowed list with VLAN 20 only, removing VLAN 10 from the trunk.
 
Now observe:
```
SW1# show interfaces trunk
PC1> ping 192.168.10.12
PC2> ping 192.168.20.21
```
 
**✏️ Document the symptoms:**
- Which VLANs now appear in the allowed list on the trunk?
- Can PC1 reach PC3 in VLAN 10? Can VLAN 20 traffic still cross?
- Why does one VLAN work across the trunk while the other does not?
> **Why this matters:** The allowed VLAN list controls exactly which VLANs a trunk will carry. Pruning is a legitimate and useful tool for controlling traffic, but an accidental or overly aggressive allowed list silently drops a VLAN while leaving the trunk otherwise healthy. This scenario teaches you to read the allowed list carefully rather than assuming a trunk carries everything.
 
---
 
## 🔧 Part 4: Fix It
 
---
 
### Fix Scenario 1 — The Trunk That Will Not Form
 
**Your troubleshooting toolkit:**
```
show interfaces trunk       ! run on BOTH switches and compare
show interfaces status
```
 
**Structured process:**
1. Note that interswitch VLAN traffic has stopped.
2. Run `show interfaces trunk` on both switches and compare. If the link appears as a trunk on one side only, you have found the mismatch.
3. Set both ends to the same trunk mode.
4. Verify the trunk forms and traffic returns.
**✅ Solution:**
```
SW2(config)# interface GigabitEthernet0/0
SW2(config-if)# switchport mode trunk
SW2(config-if)# exit
```
 
**Verification:**
```
SW1# show interfaces trunk
PC1> ping 192.168.10.12
```
 
---
 
### Fix Scenario 2 — The Native VLAN Mismatch
 
**Your troubleshooting toolkit:**
```
show interfaces trunk       ! compare the native VLAN column on both ends
show logging                ! look for the mismatch message
```
 
**Structured process:**
1. Even if tagged traffic works, check the native VLAN on both ends.
2. Compare the native VLAN column from `show interfaces trunk` on each switch.
3. Set both ends to the same native VLAN.
4. Confirm the mismatch log messages stop.
**✅ Solution:**
```
SW1(config)# interface GigabitEthernet0/0
SW1(config-if)# switchport trunk native vlan 1
SW1(config-if)# exit
```
 
**Verification:**
```
SW1# show interfaces trunk
SW2# show interfaces trunk
```
 
---
 
### Fix Scenario 3 — A Pruned VLAN
 
**Your troubleshooting toolkit:**
```
show interfaces trunk       ! read the allowed VLAN list carefully
```
 
**Structured process:**
1. Note that one VLAN crosses the trunk while another does not.
2. Run `show interfaces trunk` and read the allowed VLAN list.
3. If the missing VLAN is absent from the list, add it back.
4. Verify the VLAN crosses the trunk again.
**✅ Solution:**
```
SW1(config)# interface GigabitEthernet0/0
SW1(config-if)# switchport trunk allowed vlan add 10
SW1(config-if)# exit
```
 
> **Note the `add` keyword.** Without it, `switchport trunk allowed vlan 10` would replace the entire list with VLAN 10 only, dropping VLAN 20 in the process. Using `add` appends VLAN 10 to the existing list. This is a small detail that causes real outages, so it is worth committing to memory.
 
**Verification:**
```
SW1# show interfaces trunk
PC1> ping 192.168.10.12
```
 
---
 
## 💬 Reflection Questions
 
1. Explain the difference between an access port and a trunk port in your own words. What kind of device sits at the end of each?
2. 802.1Q tagging lets a single physical link carry many VLANs while keeping them separate. Describe what happens to a frame as it enters and then leaves a trunk, and explain why the end devices never see the tag.
3. The native VLAN travels untagged across a trunk. Why must both ends of a trunk agree on the native VLAN, and what specifically goes wrong when they do not?
4. In Break Scenario 2, tagged VLAN traffic kept working even though the native VLAN was mismatched. Why does a native VLAN mismatch fail to break the trunk completely, and why does that make it more dangerous rather than less?
5. The allowed VLAN list controls which VLANs a trunk carries. Describe a legitimate reason an engineer might deliberately restrict the allowed list, and a way that same feature could cause an accidental outage.
6. In the VLANs lab you proved VLAN 10 and VLAN 20 cannot communicate. After building a trunk that carries both, they still cannot communicate. Explain why adding a trunk did not change that, and name the kind of device you would need to let the two VLANs talk.
---
 
## Challenge Extension (Optional)
 
1. **Prune deliberately.** Configure the trunk to allow only VLAN 10, then verify with `show interfaces trunk` that VLAN 20 no longer crosses. This is the constructive version of Break Scenario 3, and it shows pruning used on purpose for traffic control.
2. **Change the native VLAN safely.** Create a dedicated unused VLAN, then set it as the native VLAN on both ends of the trunk at once. Confirm with `show interfaces trunk` that both agree. Moving the native VLAN away from VLAN 1 is a common security hardening step, and doing it on both ends together is the safe way.
3. **Add an Engineering host to SW2.** If you can free up a node, add a fourth PC in VLAN 20 on SW2 and confirm it reaches PC2 across the trunk. This proves both VLANs span both switches, not just VLAN 10.
4. **Read the tags.** Research how a frame on VLAN 10 differs on the wire from a frame on the native VLAN as each crosses the trunk. Describe what is physically different about the two frames.
---
 
## Where This Leads
 
You can now stretch a VLAN across as many switches as a building requires, and you understand the tagging that makes it possible. You have also confirmed something important by repetition: VLANs stay separate even when they share a trunk. That separation is deliberate and correct, and it is also a problem waiting to be solved, because at some point Sales really does need to reach a resource in the Engineering VLAN.
 
That is the job of inter-VLAN routing, and it is where this section turns next. You will start with router-on-a-stick, using a single router interface divided into subinterfaces to route between your VLANs, and finally bring to life the gateway addresses that have been sitting unreachable in your addressing tables since the VLANs lab.
 
---
 
*CCNA – Build It · Observe It · Break It · Fix It series*

*Next: Inter-VLAN Routing (Router-on-a-Stick)*
