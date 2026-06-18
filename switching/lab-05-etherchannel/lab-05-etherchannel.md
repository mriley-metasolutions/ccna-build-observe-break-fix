# EtherChannel

**Build It · Observe It · Break It · Fix It**

---

## Overview

You run a single cable between two switches and it works, until that cable becomes the bottleneck, or it fails and takes the whole link with it. The obvious fix is to add a second cable. The problem is that two cables between the same two switches form a loop, and Spanning Tree (we will go over STP in the next lab) will respond by blocking one of them, so you are back to a single active link with an idle standby. You paid for two and you are using one.

EtherChannel is super cool and it solves this cleanly. It bundles multiple physical links into one **logical** link, so all of the members forward at the same time, the combined bandwidth is available to traffic, and the loss of any single member is absorbed without dropping the connection. Just as importantly, Spanning Tree sees the bundle as one link rather than several, so it never blocks a member. You get more bandwidth and real redundancy out of the same cables that would otherwise sit idle.

In this lab you will bundle two links between a pair of switches into an EtherChannel using LACP, confirm the bundle carries traffic as a single trunk, and watch it survive losing a member. Then you will break it in the three ways EtherChannels most often refuse to form, which is exactly the kind of thing worth drilling, because the failures are quiet and the fix depends on knowing a small set of rules cold.

**CCNA Exam Alignment:**
- 2.2.d – Configure and verify (Layer 2 and Layer 3) EtherChannel (LACP)

**Estimated Time:** 60–90 minutes

---

## Key Concept: How a Bundle Negotiates

You can build an EtherChannel three ways, and knowing which is which is half the battle on the exam and all of the battle when one will not form.

- **LACP** (Link Aggregation Control Protocol) is the open, IEEE standard, and the one to reach for by default. Its modes are **active** (actively asks the other side to bundle) and **passive** (only responds if asked).
- **PAgP** (Port Aggregation Protocol) is Cisco's older proprietary equivalent. Its modes are **desirable** (actively asks) and **auto** (only responds).
- **Static "on"** forms the bundle unconditionally with no negotiation protocol at all. Both ends must be set to "on," and neither will check that the other agrees.

The rule that ties it together is that at least one side has to actively initiate, and the two ends have to speak the same language. This table is worth memorizing:

| Left end | Right end | Bundle forms? |
|----------|-----------|---------------|
| LACP active | LACP active | Yes |
| LACP active | LACP passive | Yes |
| LACP passive | LACP passive | No, neither initiates |
| PAgP desirable | PAgP desirable | Yes |
| PAgP desirable | PAgP auto | Yes |
| PAgP auto | PAgP auto | No, neither initiates |
| on | on | Yes, static, no negotiation |
| on | LACP or PAgP anything | No, static does not negotiate |

> **Get ahead of the confusion:** Two of the three break scenarios in this lab come straight out of that table, passive paired with passive, and static "on" paired with LACP. If you understand why each of those fails, you will diagnose a dead bundle in seconds. We use LACP active on both ends in the build, because active-active is the cleanest, most predictable choice.

There is one more rule that has nothing to do with negotiation: **every member port in a bundle must share the same settings.** Same speed, same duplex, same access or trunk mode, same VLAN configuration, on both ends. A member that disagrees with the others gets left out of the bundle, which is the third failure you will create.

---

## Topology

```
        [PC1]                              [PC2]
       VLAN 10                            VLAN 10
     192.168.10.11                      192.168.10.12
         |                                  |
        G0/2                              G0/2
       [SW1] ========= Po1 (LACP) ========= [SW2]
        G0/0 ------------------------------ G0/0   (bundled member)
        G0/1 ------------------------------ G0/1   (bundled member)

  Po1 = a two-link LACP EtherChannel, configured as a trunk
  PC1 and PC2 share VLAN 10, so they communicate across the bundle
```

**Nodes (4 total, within CML Free Edition limit):**
- SW1, SW2 (IOSvL2 switches)
- PC1 (on SW1), PC2 (on SW2)

> **EtherChannel costs links, not nodes.** The bundle is two parallel cables between the same two switches, so it fits the node budget easily. When you build the topology, draw both links between SW1 and SW2.

> **A note on interface names:** Port names vary by IOSvL2 image, so confirm with `show ip interface brief` and adjust. This guide bundles G0/0 and G0/1 on each switch and uses G0/2 for the host.

---

## Addressing Table

| Device | Interface | IP Address     | Subnet Mask     | VLAN |
|--------|-----------|----------------|-----------------|------|
| PC1    | NIC       | 192.168.10.11  | 255.255.255.0   | 10   |
| PC2    | NIC       | 192.168.10.12  | 255.255.255.0   | 10   |

> **No gateway needed.** Both hosts sit in VLAN 10 on the same subnet, so their traffic never has to be routed. They communicate at Layer 2 straight across the bundle, which keeps the focus of this lab exactly where it belongs, on the EtherChannel itself.

---

## Lab Objectives

By the end of this lab you will be able to:

1. Explain what an EtherChannel is and why it provides bandwidth and redundancy
2. Describe how EtherChannel and Spanning Tree work together
3. Compare LACP, PAgP, and static "on," and predict which mode combinations form a bundle
4. Configure a Layer 2 EtherChannel using LACP
5. Verify a bundle with `show etherchannel summary` and confirm it survives a member loss
6. Identify and resolve three common reasons an EtherChannel fails to form
7. Apply a structured troubleshooting process to a bundled link

---

## 🧱 Part 1: Build It

### Step 1 — Create the VLAN on Both Switches

The bundle will run as a trunk carrying VLAN 10, so both switches need that VLAN.

**On SW1:**
```
SW1> enable
SW1# configure terminal
SW1(config)# vlan 10
SW1(config-vlan)# name Sales
SW1(config-vlan)# exit
```

**On SW2:**
```
SW2> enable
SW2# configure terminal
SW2(config)# vlan 10
SW2(config-vlan)# name Sales
SW2(config-vlan)# exit
```

### Step 2 — Bundle the Links on SW1

Add both uplink ports to the same channel group with LACP active mode. Setting `channel-group` on the physical ports automatically creates the logical Port-channel interface.

```
! Bundle G0/0 and G0/1 into an LACP EtherChannel
SW1(config)# interface range GigabitEthernet0/0 - 1
SW1(config-if-range)# channel-group 1 mode active
SW1(config-if-range)# exit

! Configure the logical bundle as a trunk. Settings here apply to all members.
SW1(config)# interface Port-channel 1
SW1(config-if)# switchport mode trunk
SW1(config-if)# description EtherChannel to SW2
SW1(config-if)# exit
```

> **Configure the bundle, not the members.** Once the physical ports are in the channel group, you apply switchport settings to the Port-channel interface and they propagate to every member. Configuring members individually is how you end up with the inconsistency that breaks scenario three. Treat the Port-channel as the real interface from here on.

### Step 3 — Bundle the Links on SW2

SW2 mirrors SW1, also using LACP active.

```
SW2(config)# interface range GigabitEthernet0/0 - 1
SW2(config-if-range)# channel-group 1 mode active
SW2(config-if-range)# exit

SW2(config)# interface Port-channel 1
SW2(config-if)# switchport mode trunk
SW2(config-if)# description EtherChannel to SW1
SW2(config-if)# exit
```

### Step 4 — Configure the Host Access Ports

```
SW1(config)# interface GigabitEthernet0/2
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# description PC1 - Sales
SW1(config-if)# exit

SW2(config)# interface GigabitEthernet0/2
SW2(config-if)# switchport mode access
SW2(config-if)# switchport access vlan 10
SW2(config-if)# description PC2 - Sales
SW2(config-if)# exit
```

### Step 5 — Configure the Hosts

```
PC1:  ip 192.168.10.11 255.255.255.0
PC2:  ip 192.168.10.12 255.255.255.0
```

### Step 6 — Save Both Configurations

```
SW1# copy running-config startup-config
SW2# copy running-config startup-config
```

---

## 👀 Part 2: Observe It

### 2.1 — Verify the Bundle

This is the command that matters most in this lab.

```
SW1# show etherchannel summary
```

**What to look for:**
- Port-channel Po1 listed with protocol LACP
- Both member ports, G0/0 and G0/1, shown inside the bundle
- Status flags, where you want to see `SU` next to the port-channel, meaning it is in use at Layer 2, and `P` next to each member, meaning it is bundled into the channel

> **Learn to read the flags, because they tell the whole story.** A member marked `P` is participating. A member marked `I` is independent and not bundling. A member marked `s` is suspended for a configuration mismatch. When you reach the Break It section, the flag on the failing member is the fastest clue you have.

**✏️ Prediction checkpoint:** Before you run the command, predict how many member ports will show as bundled and what protocol the summary will report.

### 2.2 — Confirm the Trunk Rides the Bundle

```
SW1# show interfaces trunk
```

Notice that the trunk appears on Po1, the logical bundle, not on the individual physical ports. The EtherChannel is the trunk now.

### 2.3 — See What Spanning Tree Thinks

```
SW1# show spanning-tree vlan 10
```

**What to look for:** Po1 appears as a single interface in the spanning tree, not two. This is the point worth pausing on. Because Spanning Tree sees one logical link, it has no loop to resolve and blocks nothing, which is exactly why both physical cables can forward at once. Without the bundle, STP would block one of these two links.

### 2.4 — The Headline Test: Traffic Across the Bundle

From PC1, ping PC2:
```
PC1> ping 192.168.10.12
```

This should succeed. The two hosts are in the same VLAN, and their traffic crosses the EtherChannel as if it were a single, fatter link.

### 2.5 — Prove the Bundle Survives a Failed Member

This is the payoff. Start a continuous ping from PC1 to PC2, then shut one member on SW1:

```
SW1(config)# interface GigabitEthernet0/0
SW1(config-if)# shutdown
SW1(config-if)# exit
```

Check the bundle and the ping:
```
SW1# show etherchannel summary
```

**✏️ Document what you see:**
- Does Po1 stay up with one remaining member?
- Did the continuous ping keep succeeding, or did it drop?
- What does this prove about why a bundle beats a single cable?

Bring the member back when you are done:
```
SW1(config)# interface GigabitEthernet0/0
SW1(config-if)# no shutdown
SW1(config-if)# exit
```

> **Why this matters:** A single trunk cable would have taken the entire link down when it failed. The bundle simply kept forwarding on its surviving member while you fixed the other. That resilience, with no reconvergence and no outage for the hosts, is the whole reason EtherChannel exists.

---

## 💥 Part 3: Break It

> Work through each scenario one at a time. Two of the three come straight from the mode table in the Key Concept section, so use that table as you diagnose. Document the symptoms fully before you move to Part 4.

---

### Break Scenario 1 — Passive on Both Ends

LACP needs at least one side to actively initiate. Set both ends to passive and watch the bundle fail to form. Change SW2 first, then SW1:

```
SW2(config)# interface range GigabitEthernet0/0 - 1
SW2(config-if-range)# channel-group 1 mode passive
SW2(config-if-range)# exit

SW1(config)# interface range GigabitEthernet0/0 - 1
SW1(config-if-range)# channel-group 1 mode passive
SW1(config-if-range)# exit
```

Now observe:
```
SW1# show etherchannel summary
PC1> ping 192.168.10.12
```

**✏️ Document the symptoms:**
- Does the bundle form, or do the members sit waiting?
- What flag do the member ports show now compared to the healthy `P`?
- Why does passive paired with passive never come together?

> **Why this matters:** Passive mode only answers when spoken to, so when both ends are passive, neither one ever speaks first and the bundle stalls. This is one of the most common LACP mistakes, and the fix is simply to make at least one side active. The mode table predicts this exact outcome, which is why it is worth knowing by heart.

---

### Break Scenario 2 — Mixing Static and Dynamic

Restore SW1 to active. Then set SW2 to static "on" while SW1 speaks LACP, the classic do-not-mix mistake:

```
SW1(config)# interface range GigabitEthernet0/0 - 1
SW1(config-if-range)# channel-group 1 mode active
SW1(config-if-range)# exit

SW2(config)# interface range GigabitEthernet0/0 - 1
SW2(config-if-range)# channel-group 1 mode on
SW2(config-if-range)# exit
```

> SW2 now forms the channel unconditionally and sends no LACP, while SW1 waits for LACP that never comes.

Now observe:
```
SW1# show etherchannel summary
SW2# show etherchannel summary
PC1> ping 192.168.10.12
```

**✏️ Document the symptoms:**
- What does each switch report about its side of the bundle?
- SW2 thinks the bundle is up, so why does traffic still fail?
- Why can a static side and an LACP side never agree?

> **Why this matters:** Static "on" does not negotiate and does not send LACP packets. SW1, set to active, is waiting for a negotiation that the "on" side will never start, so the two ends never truly agree on a bundle even though the static side believes it is up. Mixing static and dynamic is a genuine trap because one side looks healthy. The rule is simple once you hold it: both ends must use the same method.

---

### Break Scenario 3 — An Inconsistent Member

Restore SW2 to LACP active. Then create a configuration mismatch on a single member so it gets left out of the bundle. Change SW1's G0/1 to access mode while the Port-channel remains a trunk:

```
SW2(config)# interface range GigabitEthernet0/0 - 1
SW2(config-if-range)# channel-group 1 mode active
SW2(config-if-range)# exit

SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# switchport mode access
SW1(config-if)# exit
```

Now observe:
```
SW1# show etherchannel summary
SW1# show interfaces GigabitEthernet0/1 switchport
```

**✏️ Document the symptoms:**
- How many members are now bundled in Po1 versus before?
- What flag does the misconfigured member show?
- The bundle may still pass traffic on its remaining member, so why is this still a problem?

> **Why this matters:** Every member of a bundle has to share the same settings, and a member that disagrees gets suspended rather than bundled. The channel can limp along on its remaining good member, which means the symptom is reduced bandwidth and lost redundancy rather than a dead link, and that makes it easy to overlook. This is why you configure the Port-channel interface and let the settings propagate, instead of touching members individually.

---

## 🔧 Part 4: Fix It

---

### Fix Scenario 1 — Passive on Both Ends

**Your troubleshooting toolkit:**
```
show etherchannel summary       ! on both switches, check the member flags
show running-config | include channel-group
```

**Structured process:**
1. Note that the bundle never came up and the members are not participating.
2. Check the configured mode on both ends.
3. If both are passive, neither is initiating. Set at least one side to active.
4. Verify the bundle forms.

**✅ Solution:**
```
SW1(config)# interface range GigabitEthernet0/0 - 1
SW1(config-if-range)# channel-group 1 mode active
SW1(config-if-range)# exit
```

**Verification:**
```
SW1# show etherchannel summary
PC1> ping 192.168.10.12
```

---

### Fix Scenario 2 — Mixing Static and Dynamic

**Your troubleshooting toolkit:**
```
show etherchannel summary       ! the protocol column tells you each side's method
show running-config | include channel-group
```

**Structured process:**
1. Note that one side reports the bundle up while traffic still fails.
2. Compare the negotiation method on each end. One side static, the other LACP, is the mismatch.
3. Set both ends to the same method, LACP active being the clean choice.
4. Verify the bundle and traffic.

**✅ Solution:**
```
SW2(config)# interface range GigabitEthernet0/0 - 1
SW2(config-if-range)# channel-group 1 mode active
SW2(config-if-range)# exit
```

**Verification:**
```
SW2# show etherchannel summary
PC1> ping 192.168.10.12
```

---

### Fix Scenario 3 — An Inconsistent Member

**Your troubleshooting toolkit:**
```
show etherchannel summary                      ! count bundled members, read the flags
show interfaces GigabitEthernet0/1 switchport  ! compare the odd member's settings
```

**Structured process:**
1. Note that the bundle has fewer active members than you cabled.
2. Identify the suspended member from the summary flags.
3. Compare its settings against the Port-channel and bring them back into agreement.
4. Verify the member rejoins the bundle.

**✅ Solution:**
```
SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# switchport mode trunk
SW1(config-if)# exit
```

> **Note:** The cleanest long-term habit is to configure the Port-channel interface and leave the members alone, so their settings always inherit from the bundle and stay consistent.

**Verification:**
```
SW1# show etherchannel summary
```

---

## 💬 Reflection Questions

1. Two cables between the same pair of switches would normally cause Spanning Tree to block one of them. Explain why bundling those cables into an EtherChannel lets both forward at once.

2. Compare LACP, PAgP, and static "on." Which is the open standard, which is Cisco proprietary, and what is the risk of using static "on" rather than a negotiation protocol?

3. Using the mode table, explain why LACP passive paired with LACP passive fails to form a bundle, while LACP active paired with LACP passive succeeds.

4. In Break Scenario 2, SW2 was set to static "on" and believed its bundle was up, yet traffic still failed. Explain why a static side and an LACP side never truly agree, and why the static side's healthy appearance makes this failure harder to spot.

5. In Break Scenario 3, the bundle kept passing traffic even with a suspended member. Why is a degraded bundle that still works arguably more dangerous to overlook than a bundle that fails completely?

6. EtherChannel load balances traffic across its members, but a single conversation between two hosts tends to use only one member rather than splitting across both. Why might bundling two 1 Gbps links not give a single large file transfer 2 Gbps of throughput?

---

## Challenge Extension (Optional)

1. **Try a Layer 3 EtherChannel.** Research how to convert the bundle into a routed EtherChannel with `no switchport` on the Port-channel and an IP address assigned to it. Where in a network would a routed EtherChannel make more sense than a Layer 2 one?

2. **Examine load balancing.** Run `show etherchannel load-balance` to see the hashing method in use, then research what `src-mac`, `dst-mac`, and `src-dst-ip` mean. Why does the choice of hashing method affect how evenly traffic spreads across members?

3. **Prove the mode table.** Deliberately configure LACP active on one side and PAgP desirable on the other. Predict the result first, then verify. Explain why two different negotiation protocols cannot form a bundle together.

4. **Bundle three links.** If you can free the ports, add a third member to the channel group and confirm all three bundle. Then shut two of them and watch the channel survive on the last remaining member.

---

## Where This Leads

You now have a link between two switches that is faster and more resilient than a single cable, and you understand the small set of rules that decide whether a bundle forms. You also saw the most important relationship in this part of the section, that EtherChannel and Spanning Tree are partners rather than rivals. The bundle hands Spanning Tree a single logical link, so STP has nothing to block, and both cables stay in service.

That partnership is the perfect setup for the next lab. EtherChannel removed one source of redundant links by bundling them, but real networks still have redundant paths that are not bundled, and those paths absolutely will form loops. Spanning Tree Protocol is what keeps those loops from melting the network down, and it is the last core topic before the capstone pulls everything together. With the way a bundle hides its members from STP fresh in your mind, watching STP elect a root and block a path will make immediate sense.

---

*CCNA – Build It · Observe It · Break It · Fix It series*

*Next: Spanning Tree Protocol*

*This repository is an independent educational resource and is not officially affiliated with Cisco Systems. Cisco, CCNA, Cisco Modeling Labs, and Cisco Networking Academy are trademarks of Cisco Systems, Inc.*
