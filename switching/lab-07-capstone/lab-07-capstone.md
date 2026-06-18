# Switching Capstone: Collapsed Core Campus LAN
 
**Build It · Observe It · Break It · Fix It**
 
---
 
## Overview
 
This is the finale of the switching section, and it is the lab where everything you have learned stops being separate techniques and becomes a network. You built VLANs to separate traffic, trunks to carry that separation between switches, Layer 3 switching to route between VLANs, EtherChannel to bundle links for bandwidth and resilience, and Spanning Tree to keep redundant paths from looping. Here you put all of it into one topology shaped like an actual campus network and prove it works end to end.
 
The design is a **collapsed core**. In the full Cisco hierarchy there are three tiers, the core, the distribution, and the access layer. On a network the size of one building, the core and distribution functions are usually combined into a single Layer 3 tier, and that combination is what "collapsed" means. A multilayer switch sits on top doing all the routing, two Layer 2 switches sit below connecting the users, and the uplinks between them are not single cables but **EtherChannel bundles**, so the loss of one physical link does not isolate anyone.
 
I am not going to reteach the individual pieces here. By this point you know them, and a capstone should ask you to assemble what you know rather than walk you through it again. What is genuinely new is the integration, and above all the troubleshooting, because when a fault appears in a layered network the most valuable skill you can have is the ability to tell which layer it lives in before you touch a single command.
 
**CCNA Exam Alignment:**
- 1.2 – Network topology architectures, including two-tier (collapsed core) and three-tier
- 1.1 – Compare network components, including Layer 2 and Layer 3 switches
- 2.1 – Configure and verify VLANs spanning multiple switches
- 2.2 – Configure and verify interswitch connectivity (trunk ports, 802.1Q)
- 2.2.d – Configure and verify (Layer 2/Layer 3) EtherChannel (LACP)
**Estimated Time:** 120–150 minutes
 
---
 
## Key Concept: Where Each Job Lives
 
A collapsed core network divides responsibility cleanly between its two switch tiers, and that division is the key to both building it and fixing it.
 
- The **access switches** are pure Layer 2. They connect end devices, place each device in the correct VLAN, and hand all of that VLAN traffic upward to the core over a trunked EtherChannel. They do no routing and own no gateway addresses. They are the front door.
- The **core switch** is Layer 3. It defines every VLAN, connects to each access switch over a trunked EtherChannel, and runs `ip routing` with one SVI per VLAN. Every default gateway in the network lives here. When a host in VLAN 10 needs VLAN 20, its traffic rides the bundle up to the core, the core routes between its SVIs, and the answer rides a bundle back down.
There is one more relationship worth naming up front, because it ties EtherChannel and Spanning Tree together. **To Spanning Tree, an EtherChannel looks like a single logical link.** That is why both physical cables in a bundle can forward at the same time without STP blocking one of them as a loop. Bundle first, and the redundancy becomes usable bandwidth instead of a blocked standby path.
 
> **Get ahead of the confusion:** The gateways live only on the core, never on the access switches. Students often assume the switch a host plugs into must hold that host's gateway. It does not. The access switch forwards the frames upward, and the core does the routing. Keep that picture of traffic flowing up to be routed and back down, and the whole design stays clear.
 
---
 
## Topology
 
```
                           [CORE1]
                  Layer 3 multilayer switch
                  ip routing ENABLED
                  SVI VLAN 10 -> 192.168.10.1
                  SVI VLAN 20 -> 192.168.20.1
              Po1 (G0/0 + G0/1)   Po2 (G0/2 + G0/3)
                     //                   \\
          EtherChannel trunk        EtherChannel trunk
              (2 bundled links)      (2 bundled links)
                   //                       \\
          Po1 (G0/0 + G0/1)          Po1 (G0/0 + G0/1)
              [ACC1]                      [ACC2]
           Layer 2 access             Layer 2 access
              G0/2                        G0/2
               |                           |
             [PC1]                       [PC2]
            VLAN 10                     VLAN 20
          192.168.10.11               192.168.20.21
```
 
> **Full topology file:** `topology.yaml`
 
**Nodes (5 total, within CML Free Edition limit):**
- CORE1 (IOSvL2 switch acting as the Layer 3 core)
- ACC1, ACC2 (IOSvL2 switches acting as Layer 2 access switches)
- PC1 (VLAN 10, on ACC1), PC2 (VLAN 20, on ACC2)
> **EtherChannel costs links, not nodes.** Each uplink is two physical cables bundled into one logical link, which is what lets this capstone include redundancy while staying within the five-node limit. When you build the topology, draw two parallel links between the core and each access switch.
 
> **A note on switch ports and images:** Interface names vary by IOSvL2 image, so confirm with `show ip interface brief` and adjust. On some multilayer platforms a port defaults to a routed Layer 3 port and must be converted with the `switchport` command before it can join a Layer 2 EtherChannel or become a trunk. IOSvL2 in CML generally defaults to Layer 2 ports, but if a command is rejected, enter `switchport` on the interface first.
 
---
 
## Addressing and Port Table
 
| Device | Interface        | Bundled Into | IP Address     | VLAN | Role |
|--------|------------------|--------------|----------------|------|------|
| CORE1  | G0/0, G0/1       | Po1 (trunk)  | —              | —    | Uplink to ACC1 |
| CORE1  | G0/2, G0/3       | Po2 (trunk)  | —              | —    | Uplink to ACC2 |
| CORE1  | SVI VLAN 10      | —            | 192.168.10.1   | 10   | VLAN 10 gateway |
| CORE1  | SVI VLAN 20      | —            | 192.168.20.1   | 20   | VLAN 20 gateway |
| ACC1   | G0/0, G0/1       | Po1 (trunk)  | —              | —    | Uplink to CORE1 |
| ACC1   | G0/2             | —            | —              | 10   | Access port to PC1 |
| ACC2   | G0/0, G0/1       | Po1 (trunk)  | —              | —    | Uplink to CORE1 |
| ACC2   | G0/2             | —            | —              | 20   | Access port to PC2 |
| PC1    | NIC              | —            | 192.168.10.11  | 10   | Sales host |
| PC2    | NIC              | —            | 192.168.20.21  | 20   | Engineering host |
 
> **Port-channel numbers are locally significant.** Each access switch uses Po1 for its own uplink, and the core uses Po1 for ACC1 and Po2 for ACC2. The numbers do not have to match across devices, only within a device. The members on each end of a given bundle do have to agree on settings.
 
---
 
## Lab Objectives
 
By the end of this lab you will be able to:
 
1. Describe the collapsed core design and how it maps to the Cisco hierarchy
2. Configure Layer 2 access switches that reach the core over a trunked EtherChannel
3. Configure a Layer 3 core switch with `ip routing` and per-VLAN SVIs
4. Configure and verify Layer 2 EtherChannel bundles using LACP
5. Verify a complete data path from a host, up the hierarchy, through routing, and back down
6. Demonstrate that an EtherChannel survives the loss of a member link
7. Diagnose a layered network by isolating which tier or technology holds a fault
---
 
## 🧱 Part 1: Build It
 
Configure the access switches first, then the core. Building the access layer first means the bundles and the VLANs they carry are already in place when you create the SVIs on the core.
 
### Step 1 — Configure Access Switch ACC1
 
ACC1 defines both VLANs, places PC1 in VLAN 10, and reaches the core over a two-link LACP EtherChannel running as a trunk.
 
```
SW> enable
SW# configure terminal
SW(config)# hostname ACC1
 
! Define both VLANs so the trunked bundle can carry them
ACC1(config)# vlan 10
ACC1(config-vlan)# name Sales
ACC1(config-vlan)# exit
ACC1(config)# vlan 20
ACC1(config-vlan)# name Engineering
ACC1(config-vlan)# exit
 
! Bundle the two uplink ports into an LACP EtherChannel
ACC1(config)# interface range GigabitEthernet0/0 - 1
ACC1(config-if-range)# channel-group 1 mode active
ACC1(config-if-range)# exit
 
! Configure the logical bundle as a trunk. Settings here propagate to both members.
ACC1(config)# interface Port-channel 1
ACC1(config-if)# switchport mode trunk
ACC1(config-if)# description Uplink bundle to CORE1
ACC1(config-if)# exit
 
! Access port for PC1 in VLAN 10
ACC1(config)# interface GigabitEthernet0/2
ACC1(config-if)# switchport mode access
ACC1(config-if)# switchport access vlan 10
ACC1(config-if)# description PC1 - Sales
ACC1(config-if)# exit
```
 
> **Why `mode active`:** That selects LACP, the open-standard negotiation protocol, and tells this side to actively ask the other end to form a bundle. LACP active on both ends is the cleanest, most predictable choice, and it is what I want you reaching for by default. You will see in Break Scenario 1 what happens when the two ends disagree on protocol.
 
### Step 2 — Configure Access Switch ACC2
 
ACC2 mirrors ACC1, with PC2 in VLAN 20.
 
```
SW(config)# hostname ACC2
 
ACC2(config)# vlan 10
ACC2(config-vlan)# name Sales
ACC2(config-vlan)# exit
ACC2(config)# vlan 20
ACC2(config-vlan)# name Engineering
ACC2(config-vlan)# exit
 
ACC2(config)# interface range GigabitEthernet0/0 - 1
ACC2(config-if-range)# channel-group 1 mode active
ACC2(config-if-range)# exit
 
ACC2(config)# interface Port-channel 1
ACC2(config-if)# switchport mode trunk
ACC2(config-if)# description Uplink bundle to CORE1
ACC2(config-if)# exit
 
ACC2(config)# interface GigabitEthernet0/2
ACC2(config-if)# switchport mode access
ACC2(config-if)# switchport access vlan 20
ACC2(config-if)# description PC2 - Engineering
ACC2(config-if)# exit
```
 
### Step 3 — Configure the Core Switch VLANs and Bundles
 
CORE1 needs both VLANs and a separate trunked EtherChannel down to each access switch.
 
```
SW(config)# hostname CORE1
 
CORE1(config)# vlan 10
CORE1(config-vlan)# name Sales
CORE1(config-vlan)# exit
CORE1(config)# vlan 20
CORE1(config-vlan)# name Engineering
CORE1(config-vlan)# exit
 
! Bundle to ACC1 -> Port-channel 1
CORE1(config)# interface range GigabitEthernet0/0 - 1
CORE1(config-if-range)# channel-group 1 mode active
CORE1(config-if-range)# exit
CORE1(config)# interface Port-channel 1
CORE1(config-if)# switchport mode trunk
CORE1(config-if)# description Bundle to ACC1
CORE1(config-if)# exit
 
! Bundle to ACC2 -> Port-channel 2
CORE1(config)# interface range GigabitEthernet0/2 - 3
CORE1(config-if-range)# channel-group 2 mode active
CORE1(config-if-range)# exit
CORE1(config)# interface Port-channel 2
CORE1(config-if)# switchport mode trunk
CORE1(config-if)# description Bundle to ACC2
CORE1(config-if)# exit
```
 
### Step 4 — Enable Routing and Create the SVIs
 
This is where the collapsed core earns its name. Turn on routing, then create one SVI per VLAN to act as the gateway for the entire network.
 
```
! Turn the core into a router
CORE1(config)# ip routing
 
! Gateway for VLAN 10
CORE1(config)# interface vlan 10
CORE1(config-if)# description VLAN 10 Gateway - Sales
CORE1(config-if)# ip address 192.168.10.1 255.255.255.0
CORE1(config-if)# no shutdown
CORE1(config-if)# exit
 
! Gateway for VLAN 20
CORE1(config)# interface vlan 20
CORE1(config-if)# description VLAN 20 Gateway - Engineering
CORE1(config-if)# ip address 192.168.20.1 255.255.255.0
CORE1(config-if)# no shutdown
CORE1(config-if)# exit
```
 
> **Why the SVIs come up:** An SVI needs its VLAN to exist and to be active on at least one live port. On the core, the trunked bundles down to the access switches carry both VLANs, so once those bundles are up the SVIs come up with them. The core can only be a gateway for a VLAN that actually reaches it.
 
### Step 5 — Configure the Hosts
 
```
PC1:  ip 192.168.10.11 255.255.255.0 192.168.10.1
PC2:  ip 192.168.20.21 255.255.255.0 192.168.20.1
```
 
### Step 6 — Save All Three Configurations
 
```
ACC1#  copy running-config startup-config
ACC2#  copy running-config startup-config
CORE1# copy running-config startup-config
```
 
---
 
## 👀 Part 2: Observe It
 
Verify down the hierarchy, because that is also how you will troubleshoot it.
 
### 2.1 — Verify the EtherChannel Bundles
 
This is the new command for this lab, and the first thing to check.
 
```
CORE1# show etherchannel summary
ACC1#  show etherchannel summary
ACC2#  show etherchannel summary
```
 
**What to look for:**
- Each port-channel listed with protocol LACP
- Both member ports shown in each bundle
- Status flags indicating the bundle is in use, typically `SU` for the port-channel (in use, Layer 2) and `P` for each bundled member port
**✏️ Prediction checkpoint:** Before you run this, predict how many port-channels the core will show versus each access switch, and how many member ports will be in each.
 
### 2.2 — Verify the Trunks Ride the Bundles
 
```
CORE1# show interfaces trunk
```
 
Notice that the trunks now appear on Po1 and Po2, the logical bundles, rather than on individual physical ports. The EtherChannel is the trunk now, and both VLANs should be allowed and active on each.
 
### 2.3 — Verify Routing Lives Only on the Core
 
```
CORE1# show ip route
ACC1#  show ip route
```
 
CORE1 holds connected routes for both VLAN subnets through its SVIs. ACC1, a pure Layer 2 switch, has nothing to route. That contrast is the collapsed core design made visible.
 
### 2.4 — The Capstone Test: A Packet Across the Whole Hierarchy
 
From PC1 in VLAN 10 on ACC1, ping PC2 in VLAN 20 on ACC2:
 
```
PC1> ping 192.168.20.21
```
 
This one successful ping is the whole lab. The frame entered ACC1 on an access port, rode the EtherChannel up to CORE1, was routed from the VLAN 10 SVI to the VLAN 20 SVI, rode the other EtherChannel down to ACC2, and reached PC2. Every technique in the switching section took part in that round trip.
 
**✏️ Think it through:** Name every device the packet touched and what each one did to it. If you can narrate the full path, up through routing and back down, you understand the collapsed core completely.
 
### 2.5 — Prove the EtherChannel Survives a Failed Link
 
This is the payoff of bundling, and it is worth seeing directly. Start a continuous ping from PC1 to PC2, then shut a single member of one bundle on the core:
 
```
CORE1(config)# interface GigabitEthernet0/0
CORE1(config-if)# shutdown
CORE1(config-if)# exit
```
 
Check the bundle and the ping:
```
CORE1# show etherchannel summary
```
 
**✏️ Document what you see:**
- Does Po1 stay up with just one remaining member?
- Did the ping from PC1 to PC2 keep succeeding, or did it drop entirely?
- What does this prove about why we bundled the uplinks instead of using a single cable?
Bring the member back when you are done:
```
CORE1(config)# interface GigabitEthernet0/0
CORE1(config-if)# no shutdown
CORE1(config-if)# exit
```
 
> **Why this matters:** A single trunk would have taken the whole uplink down with one failed cable, isolating everything behind that switch. The bundle absorbs the loss and keeps forwarding on its remaining member. That resilience is the entire reason EtherChannel exists, and you just watched it work.
 
---
 
## 💥 Part 3: Break It
 
> This is a capstone, so the troubleshooting goal is bigger than fixing each fault. For every scenario, your first task is to read the shape of the failure and decide which layer or technology holds the problem before you touch anything. Work through them one at a time and document the symptoms fully.
 
---
 
### Break Scenario 1 — An EtherChannel That Will Not Bundle
 
A bundle needs both ends to agree on protocol. Change ACC2's members to static "on" mode while the core still uses LACP:
 
```
ACC2(config)# interface range GigabitEthernet0/0 - 1
ACC2(config-if-range)# channel-group 1 mode on
ACC2(config-if-range)# exit
```
 
> Static "on" mode does not negotiate. The core is still speaking LACP, so the two ends cannot agree, and the bundle to ACC2 fails to form.
 
Now test:
```
CORE1# show etherchannel summary
PC2> ping 192.168.20.1
PC1> ping 192.168.20.21
PC1> ping 192.168.10.1
```
 
**✏️ Document the symptoms:**
- Which host lost connectivity, and which host is unaffected?
- What state do the ACC2 bundle and its members show in `show etherchannel summary`?
- How does the location of the symptom, everything behind ACC2, point you to the part of the network where the fault lives?
> **Why this matters:** In a layered network, where the symptom appears tells you where to look. Only the hosts behind ACC2 went dark, which clears the core and ACC1 and focuses you on the ACC2 uplink. A protocol mismatch is one of the most common EtherChannel faults, and `show etherchannel summary` confirms it the moment the symptom has told you where to point it. Static "on" and LACP will never form a bundle together, and that is a detail worth committing to memory.
 
---
 
### Break Scenario 2 — A VLAN Missing on the Core
 
Restore ACC2 to LACP. Then remove VLAN 20 from the core entirely, while the access switches still have it:
 
```
ACC2(config)# interface range GigabitEthernet0/0 - 1
ACC2(config-if-range)# channel-group 1 mode active
ACC2(config-if-range)# exit
 
CORE1(config)# no vlan 20
CORE1(config)# exit
```
 
Now observe:
```
CORE1# show ip interface brief
CORE1# show ip route
CORE1# show vlan brief
PC2> ping 192.168.20.1
```
 
**✏️ Document the symptoms:**
- What happened to the VLAN 20 SVI on the core once VLAN 20 was removed?
- Did the 192.168.20.0/24 route disappear?
- PC2 is still in VLAN 20 and the bundle is healthy, so why did VLAN 20 lose its gateway?
> **Why this matters:** A gateway can only work if its VLAN exists on the device hosting it. Remove VLAN 20 from the core and its SVI has no VLAN to attach to, so it goes down and the gateway goes with it. The bundle is fine, the trunk is fine, the access switch is fine, and still VLAN 20 cannot be routed. This is a different shape of failure from Scenario 1, and noticing that the bundle is healthy while one VLAN is dead is what steers you to the VLAN and SVI layer rather than the uplink.
 
---
 
### Break Scenario 3 — Routing Disabled on the Core
 
Recreate VLAN 20 on the core. Then disable routing, the single command the whole network's inter-VLAN traffic depends on:
 
```
CORE1(config)# vlan 20
CORE1(config-vlan)# name Engineering
CORE1(config-vlan)# exit
 
CORE1(config)# no ip routing
```
 
Now test:
```
PC1> ping 192.168.10.1
PC2> ping 192.168.20.1
PC1> ping 192.168.20.21
```
 
**✏️ Document the symptoms:**
- Can each host still reach its own gateway?
- Can any host reach a host in a different VLAN?
- How is the shape of this failure different from Scenario 1, where only one access switch went dark?
> **Why this matters:** This failure has its own signature. Every host can still reach its own gateway, but no host can reach any other VLAN anywhere in the network. A failure that is total across VLANs yet leaves same-VLAN reachability intact points straight at the core's routing function, not at any bundle or access switch. Reading that pattern correctly sends you to `ip routing` immediately instead of chasing uplinks all over the topology. Across these three scenarios you have now seen three distinct failure shapes, and learning to read them is the real capstone skill.
 
---
 
## 🔧 Part 4: Fix It
 
---
 
### Fix Scenario 1 — An EtherChannel That Will Not Bundle
 
**Your troubleshooting toolkit:**
```
show etherchannel summary       ! on both ends of the suspect bundle
```
 
**Structured process:**
1. Note that the symptom is confined to the hosts behind one access switch.
2. That localization points to the uplink between that switch and the core.
3. Run `show etherchannel summary` on both ends and check the protocol and member status.
4. Set both ends to the same negotiation protocol, LACP active, and verify.
**✅ Solution:**
```
ACC2(config)# interface range GigabitEthernet0/0 - 1
ACC2(config-if-range)# channel-group 1 mode active
ACC2(config-if-range)# exit
```
 
**Verification:**
```
CORE1# show etherchannel summary
PC1> ping 192.168.20.21
```
 
---
 
### Fix Scenario 2 — A VLAN Missing on the Core
 
**Your troubleshooting toolkit:**
```
show vlan brief             ! compare across all three switches
show ip interface brief     ! find the down SVI on the core
```
 
**Structured process:**
1. Note that one VLAN lost its gateway while its hosts can still reach each other locally.
2. Compare `show vlan brief` across the switches to find where the VLAN is missing.
3. Recreate the VLAN on the device that lacks it, which reactivates the SVI.
4. Verify the gateway and routing return.
**✅ Solution:**
```
CORE1(config)# vlan 20
CORE1(config-vlan)# name Engineering
CORE1(config-vlan)# exit
```
 
**Verification:**
```
CORE1# show ip interface brief
PC2> ping 192.168.20.1
```
 
---
 
### Fix Scenario 3 — Routing Disabled on the Core
 
**Your troubleshooting toolkit:**
```
show ip route                          ! a table missing its connected VLAN routes is the tell
show running-config | include ip routing
```
 
**Structured process:**
1. Note that every VLAN reaches its own gateway but none reach another VLAN.
2. That signature points at the core's routing function rather than any single link.
3. Confirm routing is disabled and re-enable it.
4. Verify inter-VLAN traffic returns across the whole hierarchy.
**✅ Solution:**
```
CORE1(config)# ip routing
```
 
**Verification:**
```
CORE1# show ip route
PC1> ping 192.168.20.21
```
 
---
 
## 💬 Reflection Questions
 
1. Describe the collapsed core design in your own words. How does it relate to the full three-tier Cisco hierarchy, and why is collapsing the core and distribution tiers reasonable for a single building?
2. Every default gateway in this network lives on the core switch, and none live on the access switches. Explain why this placement makes sense and what would be lost by putting gateways on the access switches instead.
3. Spanning Tree treats an EtherChannel as a single logical link. Explain why that behavior is what allows both physical cables in a bundle to forward at once, and what STP would do to those two links if they were not bundled.
4. In the resilience test you shut one member of a bundle and traffic kept flowing. Explain what EtherChannel did to keep the path alive, and contrast that with what a single trunk cable would have done in the same situation.
5. Break Scenario 1 produced a failure localized behind one access switch, while Break Scenario 3 produced a network-wide inter-VLAN failure that still allowed same-VLAN traffic. Explain how the shape of each failure helps you locate the fault faster than checking devices at random.
6. Static "on" mode and LACP active will not form a bundle together. Explain why two negotiation approaches that each work on their own fail when mixed, and what that teaches you about configuring both ends of any link consistently.
---
 
## Challenge Extension (Optional)
 
1. **Add a management VLAN.** Create VLAN 99 for management, give each access switch an SVI in VLAN 99 with a management IP, and add a matching SVI and gateway on the core. Confirm you can ping each access switch from the core. This is how real access switches are managed despite being pure Layer 2 for user traffic.
2. **Add a third VLAN end to end.** Introduce VLAN 30, define it on all three switches, add its SVI and gateway on the core, and move a host into it. Notice how little has to change because the trunked bundles already carry it.
3. **Load balance across the bundle.** Research how EtherChannel decides which physical member carries a given flow, then look at the load-balancing method in use with `show etherchannel load-balance`. Why does a single conversation between two hosts tend to use only one member rather than splitting across both?
4. **Predict the blast radius.** Before breaking anything, write down which hosts would lose connectivity if an entire bundle to ACC1 failed, both members at once, then prove your prediction. Accurately predicting the blast radius of a failure is a core network design skill.
---
 
## Where This Leads
 
You have built a network, not a demonstration. Every concept in the switching section now operates as one system: VLANs separate the traffic, trunks and EtherChannel carry it between tiers with bandwidth and resilience to spare, Spanning Tree keeps the redundancy safe, and the Layer 3 core ties it all together by routing between VLANs. Just as valuable, you have practiced the skill that matters most in a layered design, reading the shape of a failure to find the layer it lives in before you touch a command. That ability to localize a fault under pressure is exactly what the exam and the job will ask of you.
 
With switching complete, the project turns toward the services that run on top of a working network. Next comes the Services section, starting with DHCP, where you will hand out the very IP addresses you have been configuring by hand on every host so far. After that comes NAT and DNS, and then the Security section. The solid Layer 2 and Layer 3 foundation you just finished is what every one of those topics will build on.
 
---
 
*CCNA – Build It · Observe It · Break It · Fix It series*
*Next: DHCP (Services series)*
 
*This repository is an independent educational resource and is not officially affiliated with Cisco Systems. Cisco, CCNA, Cisco Modeling Labs, and Cisco Networking Academy are trademarks of Cisco Systems, Inc.*
