# RIPv2 Routing
 
**Build It · Observe It · Break It · Fix It**
 
---
 
> ## 📌 Supplementary / Foundational Lab
>
> **RIP is NOT on the current CCNA 200-301 exam.** The current exam tests OSPFv2 as its dynamic routing protocol. RIP was removed in the 2020 revision.
>
> **So why include this lab?** RIP is the simplest dynamic routing protocol to understand, which makes it an excellent *first* exposure to dynamic routing concepts — automatic route sharing, convergence, metrics, and administrative distance — before tackling the heavier OSPF material. It also still appears in many introductory networking courses and some NetAcad modules.
>
> **Use this lab if:** you want to build intuition for how dynamic routing works before OSPF, or you're teaching an intro networking course.
>
> **Skip this lab if:** you are purely focused on CCNA exam prep and short on time — go straight to the OSPF labs instead.
 
---
 
## Overview
 
In the previous Static Routing lab, you manually configured every route. That works fine, but it does not scale well. Every time the network changes, you have to update every router by hand. Dynamic routing protocols solve this by having routers share what they know automatically.
 
RIP (Routing Information Protocol) is the oldest and simplest dynamic routing protocol still in use. It's a **distance-vector** protocol where each router shares its entire routing table with its directly connected neighbors at regular intervals, and routes are chosen based on a single metric: **hop count**.
 
In this lab you will configure RIPv2 across a three-router topology, observe how routes propagate automatically, then break the network with three realistic RIP misconfigurations and troubleshoot your way back.
 
**Concepts Covered (foundational, not exam-aligned):**
- Distance-vector routing fundamentals
- RIPv2 configuration and verification
- Automatic route advertisement and convergence
- Administrative distance and metrics
- Common RIP troubleshooting scenarios
**Estimated Time:** 60–90 minutes
 
---
 
## How RIP Works — The Essentials
 
Before building, understand the key characteristics of RIP. These directly explain the behavior you'll observe and the failures you'll troubleshoot.
 
| Characteristic | RIPv2 Behavior |
|----------------|----------------|
| **Type** | Distance-vector |
| **Metric** | Hop count (number of routers to cross) |
| **Maximum hops** | 15 — anything 16+ is "unreachable" (infinity) |
| **Update method** | Full routing table, every 30 seconds |
| **Update destination** | Multicast 224.0.0.9 (RIPv2) |
| **Subnet mask support** | Yes — RIPv2 is classless (sends mask in updates) |
| **Administrative distance** | 120 |
| **Authentication** | Supported (RIPv2 only) |
 
> **The 15-hop limit matters.** RIP considers 16 hops to be infinite/unreachable. This is a deliberate loop-prevention mechanism, but it also means RIP cannot be used in large networks. It's one of the main reasons RIP fell out of favor — and a great teaching point about why protocols like OSPF were developed.
 
> **RIPv2 vs RIPv1:** RIPv1 was *classful* — it didn't send subnet masks in its updates, which broke modern subnetting. RIPv2 is *classless* and sends the mask, supporting VLSM and CIDR. We use RIPv2 exclusively in this lab because RIPv1 is genuinely obsolete.
 
---
 
## Topology
 
```
[Lo0: 192.168.10.0/24]              [Lo0: 192.168.30.0/24]
        |                                      |
       R1 -------- R2 -------- R3
   10.0.12.1    10.0.12.2  10.0.23.1  10.0.23.2
 
        PC1 connected to R1 G0/0 (192.168.1.0/24)
        PC2 connected to R2 G0/0 (192.168.2.0/24)
```
 
> **Full topology file:** `topology.yaml`
 
**Nodes (5 total — within CML Free Edition limit):**
- R1, R2, R3 (IOSv routers)
- PC1 connected to R1
- PC2 connected to R2
> This is the **same topology as the Static Routing lab.** If you already built that one, you can reuse it — just clear the static routes before starting (`show run | include ip route` to find them, then `no` each one).
 
---
 
## Addressing Table
 
| Device | Interface  | IP Address      | Subnet Mask       | Default Gateway |
|--------|------------|-----------------|-------------------|-----------------|
| R1     | G0/0       | 192.168.1.1     | 255.255.255.0     | —               |
| R1     | G0/1       | 10.0.12.1       | 255.255.255.252   | —               |
| R1     | Loopback0  | 192.168.10.1    | 255.255.255.0     | —               |
| R2     | G0/0       | 192.168.2.1     | 255.255.255.0     | —               |
| R2     | G0/1       | 10.0.12.2       | 255.255.255.252   | —               |
| R2     | G0/2       | 10.0.23.1       | 255.255.255.252   | —               |
| R3     | G0/0       | 10.0.23.2       | 255.255.255.252   | —               |
| R3     | Loopback0  | 192.168.30.1    | 255.255.255.0     | —               |
| PC1    | NIC        | 192.168.1.10    | 255.255.255.0     | 192.168.1.1     |
| PC2    | NIC        | 192.168.2.10    | 255.255.255.0     | 192.168.2.1     |
 
> **A note on the /30 links and RIP:** Because RIPv2 is classless, it correctly advertises the /30 WAN links and /24 LANs with their actual masks. This is a good opportunity to show students *why* classful RIPv1 would have struggled here — it would have summarized everything to classful boundaries and broken the /30 subnets.
 
---
 
## Lab Objectives
 
By the end of this lab you will be able to:
 
1. Explain how distance-vector routing differs from static routing
2. Configure RIPv2 on multiple routers
3. Disable automatic summarization and explain why it matters
4. Verify RIP operation using `show ip route`, `show ip protocols`, and `debug ip rip`
5. Interpret RIP metrics (hop count) and administrative distance
6. Identify and resolve three common RIP misconfigurations
7. Apply a structured troubleshooting process to restore connectivity
---
 
## 🧱 Part 1: Build It
 
> This lab assumes interfaces and loopbacks are already configured with the IP addresses in the addressing table. If you're starting fresh, configure those first (refer to the Static Routing lab's Build section for the interface commands). This section focuses on the RIP configuration itself.
 
### Step 1 — Configure RIPv2 on R1
 
```
R1> enable
R1# configure terminal
 
! Enter RIP router configuration mode
R1(config)# router rip
 
! Specify RIP version 2 — always do this explicitly
R1(config-router)# version 2
 
! Disable automatic summarization so RIP advertises exact subnets
! Without this, RIP would summarize to classful boundaries and break /30s
R1(config-router)# no auto-summary
 
! Advertise the networks R1 is directly connected to
! RIP uses CLASSFUL network statements — you enter the major network
R1(config-router)# network 192.168.1.0
R1(config-router)# network 192.168.10.0
R1(config-router)# network 10.0.0.0
 
! Suppress RIP updates on interfaces with no RIP neighbors
R1(config-router)# passive-interface GigabitEthernet0/0
R1(config-router)# passive-interface Loopback0
R1(config-router)# exit
```
 
> **Key concept — the `network` command in RIP:** Unlike the 'network' command in OSPF, RIP's `network` command takes a **classful** network address with no wildcard mask. When you enter `network 10.0.0.0`, RIP enables itself on *every* interface whose IP falls within the 10.0.0.0/8 range. This is less precise than OSPF and occasionally surprises people — it's worth understanding the difference.
 
### Step 2 — Configure RIPv2 on R2
 
```
R2(config)# router rip
R2(config-router)# version 2
R2(config-router)# no auto-summary
R2(config-router)# network 192.168.2.0
R2(config-router)# network 10.0.0.0
R2(config-router)# passive-interface GigabitEthernet0/0
R2(config-router)# exit
```
 
> Note R2 only needs `network 10.0.0.0` once — it covers both the 10.0.12.0/30 and 10.0.23.0/30 links since both fall within the 10.0.0.0/8 classful range.
 
### Step 3 — Configure RIPv2 on R3
 
```
R3(config)# router rip
R3(config-router)# version 2
R3(config-router)# no auto-summary
R3(config-router)# network 192.168.30.0
R3(config-router)# network 10.0.0.0
R3(config-router)# passive-interface Loopback0
R3(config-router)# exit
```
 
### Step 4 — Save Configurations
 
```
# copy running-config startup-config
```
 
---
 
## 👀 Part 2: Observe It
 
RIP converges slowly — allow up to 60 seconds after configuring all three routers before expecting full connectivity.
 
### 2.1 — Watch RIP Updates in Real Time
 
This is the single best way to *see* distance-vector routing in action:
 
```
R2# debug ip rip
```
 
**What to look for:**
- Updates being **sent** and **received** on each interface every 30 seconds
- The networks contained in each update, along with their metric (hop count)
- The multicast address 224.0.0.9 (confirming RIPv2)
Watch a few update cycles, then turn off debugging:
```
R2# undebug all
```
 
> **Instructor note:** `debug ip rip` is one of the most pedagogically valuable commands in the entire RIP toolset. Students can literally watch routers gossip their tables to each other. Let them watch at least two full 30-second cycles.
 
**✏️ Prediction checkpoint:** Before running the debug, predict which networks R2 will advertise out its link to R3, and what hop count R3's loopback will have when it arrives back at R1.
 
### 2.2 — Examine the Routing Table
 
```
R1# show ip route rip
```
 
**What to look for:**
- `R` — routes learned via RIP
- `[120/X]` — 120 is RIP's administrative distance; X is the hop count metric
- A route to 192.168.30.0/24 (R3's loopback) — what hop count does it show from R1?
**✏️ Prediction checkpoint:** What metric (hop count) will R1 show for 192.168.2.0/24? For 192.168.30.0/24?
 
### 2.3 — Inspect the RIP Protocol Configuration
 
```
R1# show ip protocols
```
 
This is the master verification command for any routing protocol. For RIP it shows:
- The RIP version in use
- Whether auto-summary is on or off
- Which networks are being advertised
- Which interfaces are passive
- The update timers
- The administrative distance
**✏️ Prediction checkpoint:** What does the output tell you about the next scheduled update? About which interfaces RIP is active on?
 
### 2.4 — Test End-to-End Connectivity
 
```
R1# ping 192.168.30.1 source Loopback0
R1# traceroute 192.168.30.1 source Loopback0
PC1> ping 192.168.2.10
```
 
Verify all networks are reachable before moving on.
 
---
 
## 💥 Part 3: Break It
 
> Work through each scenario one at a time. Document symptoms fully before moving to Part 4.
 
---
 
### Break Scenario 1 — Version Mismatch
 
RIPv1 and RIPv2 don't fully interoperate. Force R3 back to RIPv1 behavior on its updates:
 
```
R3(config)# router rip
R3(config-router)# version 1
```
 
Wait 60 seconds, then observe:
 
```
R1# show ip route rip
R2# show ip route rip
R2# debug ip rip
```
 
**✏️ Document the symptoms:**
- Does R1 still have a route to 192.168.30.0/24?
- What does `debug ip rip` on R2 show about updates from R3? Are they being received? Ignored?
- Why does the version mismatch cause R2 to discard R3's updates?
> **Why this matters:** RIPv2 routers send updates via multicast (224.0.0.9), while RIPv1 uses broadcast and ignores subnet masks. When versions are mismatched, updates may be silently dropped. The interfaces look fine, the config looks "done," but routes mysteriously don't appear.
 
---
 
### Break Scenario 2 — The Auto-Summary Trap
 
Re-enable automatic summarization on R2 — a classic mistake:
 
```
R3(config)# router rip
R3(config-router)# version 2
R3(config-router)# exit
 
R2(config)# router rip
R2(config-router)# auto-summary
R2(config-router)# exit
R2# clear ip route *
```
 
Wait 60 seconds, then observe:
 
```
R1# show ip route rip
R3# show ip route rip
```
 
**✏️ Document the symptoms:**
- Do the /30 WAN subnets still appear correctly, or have they been summarized?
- Is connectivity affected? Test with sourced pings between loopbacks.
- What does `show ip protocols` now report about summarization?
> **Why this matters:** With auto-summary on, RIP summarizes routes to their classful boundary when advertising across a major network boundary. In a discontiguous network design, this causes routing black holes that are confusing to diagnose because the routes *look* present — just summarized incorrectly.
 
---
 
### Break Scenario 3 — The Missing Network Statement
 
Remove one of R3's network statements — simulating a forgotten advertisement:
 
```
R3(config)# router rip
R3(config-router)# no network 192.168.30.0
R3(config-router)# exit
R3# clear ip route *
```
 
Wait 60 seconds, then observe:
 
```
R1# show ip route rip
R3# show ip protocols
```
 
**✏️ Document the symptoms:**
- Can R1 still reach 192.168.30.0/24?
- Does R3 still know about its own loopback? (Check `show ip route` — is it still a connected route?)
- Why can R3 reach the loopback locally but R1 cannot reach it remotely?
> **Why this matters:** A missing `network` statement is the single most common dynamic-routing mistake. The interface is up, the router knows about the network locally as a connected route — but without the `network` statement, RIP never advertises it to neighbors. The symptom (local works, remote doesn't) is a strong diagnostic signal once students learn to recognize it.
 
---
 
## 🔧 Part 4: Fix It
 
---
 
### Fix Scenario 1 — Version Mismatch
 
**Your troubleshooting toolkit:**
```
show ip protocols           ! check version on each router
debug ip rip                ! watch what's being sent/received
```
 
**Structured process:**
1. Run `show ip protocols` on each router — compare the RIP version
2. Identify the router running a different version
3. Watch `debug ip rip` to confirm updates are being discarded
4. Align the version and verify
**✅ Solution:**
```
R3(config)# router rip
R3(config-router)# version 2
```
 
**Verification:**
```
R1# show ip route rip
R1# ping 192.168.30.1 source Loopback0
```
 
---
 
### Fix Scenario 2 — The Auto-Summary Trap
 
**Your troubleshooting toolkit:**
```
show ip protocols           ! check auto-summary status
show ip route rip           ! look for incorrectly summarized routes
```
 
**Structured process:**
1. Run `show ip protocols` — is auto-summarization enabled?
2. Examine the routing table for routes summarized to classful boundaries
3. Disable auto-summary on the offending router
4. Clear the route table and verify exact subnets return
**✅ Solution:**
```
R2(config)# router rip
R2(config-router)# no auto-summary
R2(config-router)# exit
R2# clear ip route *
```
 
**Verification:**
```
R1# show ip route rip
R3# show ip route rip
```
 
---
 
### Fix Scenario 3 — The Missing Network Statement
 
**Your troubleshooting toolkit:**
```
show ip protocols           ! see which networks RIP is advertising
show ip route               ! confirm the network exists locally as connected
```
 
**Structured process:**
1. On the router that owns the unreachable network, run `show ip route` — confirm it's a connected route
2. Run `show ip protocols` — is that network in the advertised list?
3. If the connected route exists but isn't advertised, the `network` statement is missing
4. Add the statement and verify
**✅ Solution:**
```
R3(config)# router rip
R3(config-router)# network 192.168.30.0
```
 
**Verification:**
```
R3# show ip protocols
R1# show ip route rip
R1# ping 192.168.30.1 source Loopback0
```
 
---
 
## 💬 Reflection Questions
 
1. RIP uses hop count as its only metric. Consider a path that crosses 2 routers over fast gigabit links versus a path that crosses 1 router over a slow, congested link. Which path does RIP prefer? Why is this a weakness of distance-vector protocols?
2. RIP's administrative distance is 120, while OSPF's is 110 and a static route is 1. If a router learned the same destination network from RIP, OSPF, and a static route simultaneously, which one would it install in the routing table? Why?
3. In Break Scenario 3, R3 could still reach its own loopback even though R1 couldn't. Explain the difference between a route existing locally as a *connected* route versus being *advertised* to neighbors.
4. RIP has a maximum hop count of 15, with 16 meaning "unreachable." Why does RIP impose this limit? What problem is it designed to prevent, and how does this limit affect where RIP can realistically be used?
5. You disabled auto-summary in this lab. Explain what auto-summarization does and describe a specific network design where leaving it enabled would cause routing failures.
6. RIP sends its full routing table every 30 seconds, even when nothing has changed. Compare this to how OSPF handles updates. What are the efficiency and scalability implications?
---
 
## Challenge Extension (Optional)
 
1. **Observe convergence time.** Shut down R2's G0/2 interface (the link to R3). Time how long it takes for R1's route to 192.168.30.0/24 to disappear. RIP relies on timers (invalid, holddown, flush) rather than fast detection — how does this compare to OSPF's convergence in the OSPF lab?
2. **Compare RIP and OSPF side by side.** If you've completed the OSPF lab, configure both RIP and OSPF on the same topology simultaneously. Use `show ip route` to see which protocol's routes win. Confirm it matches what administrative distance predicts.
3. **Explore RIP timers.** Run `show ip protocols` and note the update, invalid, holddown, and flush timers. Research what each one does. Why does RIP's reliance on these timers make it slow to converge compared to OSPF?
4. **Authentication.** RIPv2 supports MD5 authentication between neighbors. Configure a key chain and enable RIP authentication on the R1–R2 link. Verify with `debug ip rip` that authenticated updates are exchanged. What happens if the keys don't match?
---
 
*CCNA – Build It · Observe It · Break It · Fix It series*
