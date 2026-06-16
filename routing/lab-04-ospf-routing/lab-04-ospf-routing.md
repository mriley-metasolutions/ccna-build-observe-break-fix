# Single-Area OSPF
 
**Build It · Observe It · Break It · Fix It**
 
---
 
## Overview
 
Up to this point you have configured every route by hand. That works, and the discipline of doing it manually is worth having, but it does not scale. The moment a network grows past a handful of routers, or a link fails at 2 a.m., manual routing becomes a liability. OSPF is the answer the CCNA exam expects you to know, and it is the protocol I want you to be genuinely comfortable with by the time you sit for the test.
 
OSPF (Open Shortest Path First) is a **link-state** routing protocol. Rather than passing around route tables the way the older distance-vector protocols do, OSPF routers share a complete map of the network, called the **Link State Database (LSDB)**, and each router runs the same shortest-path algorithm against that map to calculate its own best routes. Every router ends up with an identical picture of the network and reaches its own independent conclusions from it.
 
In this lab you will build a four-router single-area topology, watch neighbor adjacencies form and the database synchronize, then break the network with the three OSPF failures I have seen trip up students and working engineers alike. By the end, the output of `show ip ospf neighbor` should read like plain language to you, because reading it fluently is one of the most useful skills you can bring into the exam.
 
**CCNA Exam Alignment:**
- 3.4 – Configure and verify single area OSPFv2
- 3.4.a – Neighbor adjacencies
- 3.4.b – Point-to-point and broadcast network types
- 3.4.c – Router ID
- 3.1 – Interpret the components of a routing table
**Estimated Time:** 75–105 minutes
 
---
 
## A Note on Loopbacks in This Lab
 
This lab uses **loopback interfaces** on each router to simulate LAN segments instead of connecting separate PC nodes. That keeps the topology inside the 5-node limit of CML Free, and it adds realism rather than sacrificing it. In production networks, loopbacks are everywhere: management addresses, OSPF and BGP router IDs, and peering endpoints all tend to live on loopbacks precisely because a loopback never goes down.
 
That last point is worth holding onto. OSPF can pull its Router ID from a loopback, and because a loopback stays up even when physical interfaces flap, the Router ID stays stable. You will assign Router IDs manually in this lab, which I consider the best practice, but knowing why loopbacks are the preferred fallback source is exam-worthy on its own.
 
---
 
## Topology
 
```
  [Lo0: 192.168.1.0/24]                    [Lo0: 192.168.3.0/24]
          |                                          |
         R1 ----------- R2 ----------- R3
     10.0.12.1      10.0.12.2      10.0.23.1      10.0.23.2
                    10.0.24.1                   [Lo0: 192.168.4.0/24]
                        |                              |
                    10.0.24.2                         R4
                       R4 ---------------------------+
 
  All four routers in OSPF Area 0.
  R2 is the transit hub connecting R1, R3, and R4.
```
 
> **Full topology file:** `topology.yaml`
 
**Nodes (4 total, within CML Free Edition limit):**
- R1, R2, R3, R4 (IOSv routers)
- No separate PC nodes. Loopbacks simulate the LAN segments.
---
 
## Addressing Table
 
| Device | Interface  | IP Address      | Subnet Mask       |
|--------|------------|-----------------|-------------------|
| R1     | G0/0       | 10.0.12.1       | 255.255.255.252   |
| R1     | Loopback0  | 192.168.1.1     | 255.255.255.0     |
| R2     | G0/0       | 10.0.12.2       | 255.255.255.252   |
| R2     | G0/1       | 10.0.23.1       | 255.255.255.252   |
| R2     | G0/2       | 10.0.24.1       | 255.255.255.252   |
| R2     | Loopback0  | 192.168.2.1     | 255.255.255.0     |
| R3     | G0/0       | 10.0.23.2       | 255.255.255.252   |
| R3     | Loopback0  | 192.168.3.1     | 255.255.255.0     |
| R4     | G0/0       | 10.0.24.2       | 255.255.255.252   |
| R4     | Loopback0  | 192.168.4.1     | 255.255.255.0     |
 
> **OSPF Router IDs (configure manually):**
> R1 = 1.1.1.1, R2 = 2.2.2.2, R3 = 3.3.3.3, R4 = 4.4.4.4
 
---
 
## Lab Objectives
 
By the end of this lab you will be able to:
 
1. Configure single-area OSPFv2 across multiple routers using loopback interfaces
2. Explain why loopbacks are the preferred source for Router IDs in production
3. Verify OSPF neighbor adjacency formation using `show` commands
4. Interpret the Link State Database and understand its role
5. Explain how OSPF calculates cost and selects the best path
6. Identify and resolve three common OSPF neighbor adjacency failures
7. Apply a structured troubleshooting process to restore full OSPF convergence
---
 
## Background: The OSPF Neighbor State Machine
 
I want you to understand the handshake before you configure anything, because almost every OSPF troubleshooting problem comes down to knowing where in this sequence a relationship got stuck. Two OSPF routers move through a series of states before they fully exchange routing information:
 
| State | What Is Happening |
|-------|-------------------|
| **Down** | No hellos received yet |
| **Init** | A hello arrived, but my own Router ID is not in it yet |
| **2-Way** | My Router ID appears in their hello, so we see each other |
| **ExStart** | Negotiating who leads the database exchange |
| **Exchange** | Trading Database Description packets |
| **Loading** | Requesting the specific LSAs each router is missing |
| **Full** | The databases are synchronized, and routing can begin |
 
On point-to-point links, neighbors move straight to **Full**. On broadcast segments like Ethernet LANs, the routers elect a Designated Router and Backup Designated Router, and the non-designated routers settle at **2-Way** with each other while reaching **Full** with the DR and BDR.
 
Here is why this matters in practice. When you run `show ip ospf neighbor` and find a relationship parked at **Init** or **2-Way** when it should be **Full**, that state is a signpost. It tells you exactly how far the handshake got before it broke, and that narrows your search for the cause down to a short list. I will point this out again in the Break It section, because you will see it happen.
 
---
 
## 🧱 Part 1: Build It
 
### Step 1 — Configure Hostnames, Interfaces, and Loopbacks
 
**R1:**
```
Router> enable
Router# configure terminal
Router(config)# hostname R1
 
R1(config)# interface GigabitEthernet0/0
R1(config-if)# description WAN - Link to R2
R1(config-if)# ip address 10.0.12.1 255.255.255.252
R1(config-if)# no shutdown
R1(config-if)# exit
 
R1(config)# interface Loopback0
R1(config-if)# description Simulated LAN - R1 Site
R1(config-if)# ip address 192.168.1.1 255.255.255.0
R1(config-if)# exit
```
 
**R2:**
```
Router(config)# hostname R2
 
R2(config)# interface GigabitEthernet0/0
R2(config-if)# description WAN - Link to R1
R2(config-if)# ip address 10.0.12.2 255.255.255.252
R2(config-if)# no shutdown
R2(config-if)# exit
 
R2(config)# interface GigabitEthernet0/1
R2(config-if)# description WAN - Link to R3
R2(config-if)# ip address 10.0.23.1 255.255.255.252
R2(config-if)# no shutdown
R2(config-if)# exit
 
R2(config)# interface GigabitEthernet0/2
R2(config-if)# description WAN - Link to R4
R2(config-if)# ip address 10.0.24.1 255.255.255.252
R2(config-if)# no shutdown
R2(config-if)# exit
 
R2(config)# interface Loopback0
R2(config-if)# description Simulated LAN - R2 Site
R2(config-if)# ip address 192.168.2.1 255.255.255.0
R2(config-if)# exit
```
 
**R3:**
```
Router(config)# hostname R3
 
R3(config)# interface GigabitEthernet0/0
R3(config-if)# description WAN - Link to R2
R3(config-if)# ip address 10.0.23.2 255.255.255.252
R3(config-if)# no shutdown
R3(config-if)# exit
 
R3(config)# interface Loopback0
R3(config-if)# description Simulated LAN - R3 Site
R3(config-if)# ip address 192.168.3.1 255.255.255.0
R3(config-if)# exit
```
 
**R4:**
```
Router(config)# hostname R4
 
R4(config)# interface GigabitEthernet0/0
R4(config-if)# description WAN - Link to R2
R4(config-if)# ip address 10.0.24.2 255.255.255.252
R4(config-if)# no shutdown
R4(config-if)# exit
 
R4(config)# interface Loopback0
R4(config-if)# description Simulated LAN - R4 Site
R4(config-if)# ip address 192.168.4.1 255.255.255.0
R4(config-if)# exit
```
 
---
 
### Step 2 — Configure OSPF
 
Two concepts deserve your attention before you type these commands, because both surprise people the first time.
 
The first is the **`network` command**. It does not mean "advertise this network." It means "enable OSPF on any interface whose IP falls inside this range, and then advertise whatever subnet is attached to that interface." The wildcard mask is an inverted subnet mask, so `0.0.0.3` matches a /30 and `0.0.0.255` matches a /24.
 
The second concerns **loopbacks and OSPF network type**. By default, IOS treats a loopback as a host and advertises it as a /32, even when you configured it with a /24. If you want the full /24 to appear in everyone's routing table, you tell the loopback to behave like a point-to-point interface with `ip ospf network point-to-point`. I have watched this exact detail cost students an hour of confusion, so we handle it up front on every loopback.
 
**R1:**
```
R1(config)# interface Loopback0
R1(config-if)# ip ospf network point-to-point
R1(config-if)# exit
 
R1(config)# router ospf 1
R1(config-router)# router-id 1.1.1.1
R1(config-router)# network 192.168.1.0 0.0.0.255 area 0
R1(config-router)# network 10.0.12.0 0.0.0.3 area 0
R1(config-router)# passive-interface Loopback0
R1(config-router)# exit
```
 
**R2:**
```
R2(config)# interface Loopback0
R2(config-if)# ip ospf network point-to-point
R2(config-if)# exit
 
R2(config)# router ospf 1
R2(config-router)# router-id 2.2.2.2
R2(config-router)# network 192.168.2.0 0.0.0.255 area 0
R2(config-router)# network 10.0.12.0 0.0.0.3 area 0
R2(config-router)# network 10.0.23.0 0.0.0.3 area 0
R2(config-router)# network 10.0.24.0 0.0.0.3 area 0
R2(config-router)# passive-interface Loopback0
R2(config-router)# exit
```
 
**R3:**
```
R3(config)# interface Loopback0
R3(config-if)# ip ospf network point-to-point
R3(config-if)# exit
 
R3(config)# router ospf 1
R3(config-router)# router-id 3.3.3.3
R3(config-router)# network 192.168.3.0 0.0.0.255 area 0
R3(config-router)# network 10.0.23.0 0.0.0.3 area 0
R3(config-router)# passive-interface Loopback0
R3(config-router)# exit
```
 
**R4:**
```
R4(config)# interface Loopback0
R4(config-if)# ip ospf network point-to-point
R4(config-if)# exit
 
R4(config)# router ospf 1
R4(config-router)# router-id 4.4.4.4
R4(config-router)# network 192.168.4.0 0.0.0.255 area 0
R4(config-router)# network 10.0.24.0 0.0.0.3 area 0
R4(config-router)# passive-interface Loopback0
R4(config-router)# exit
```
 
> **Why `passive-interface` on loopbacks:** OSPF tries to send hello packets out every interface it is enabled on. A loopback has no neighbor to greet, so those hellos accomplish nothing. Passive mode tells OSPF to advertise the network attached to the interface while keeping quiet on it. Use this on loopbacks and on any LAN-facing interface in production.
 
---
 
### Step 3 — Save Configurations
 
On each router:
```
# copy running-config startup-config
```
 
---
 
## 👀 Part 2: Observe It
 
Give OSPF 30 to 60 seconds to converge after you finish configuring all four routers before you run these commands.
 
### 2.1 — Verify Neighbor Adjacencies
 
Start on R2, since it connects to all three of the other routers and should show three neighbors.
 
```
R2# show ip ospf neighbor
```
 
**What to look for:**
- **Neighbor ID**, which is the Router ID of the neighbor
- **State**, which should read `FULL/-` on these point-to-point links. The `-` means no DR election happened, which is correct for this link type.
- **Dead Time**, a countdown that resets every time a hello arrives. If it reaches zero, the neighbor is declared dead.
- **Interface**, the local interface the neighbor sits behind
**✏️ Prediction checkpoint:** Before you run the command, predict how many neighbors R1 should have, and how many R4 should have. Write your answers down, then check them on each router.
 
### 2.2 — Examine the OSPF Routing Table
 
```
R1# show ip route ospf
```
 
**What to look for:**
- `O`, which marks a route learned through OSPF
- `[110/X]`, where 110 is OSPF's administrative distance and X is the cost
- OSPF cost, calculated as 100,000,000 divided by the interface bandwidth in bits per second. On a 100 Mbps interface, that comes out to a cost of 1.
**✏️ Prediction checkpoint:** What total cost do you expect for R1's route to 192.168.4.0/24? Trace the path R1 takes to reach it, and add up the cost of each link along the way.
 
### 2.3 — Inspect the Link State Database
 
```
R1# show ip ospf database
```
 
Every router in Area 0 should hold an identical copy of this database. Run the same command on R4 and compare the two outputs. If OSPF is healthy, they match exactly.
 
**What to look for:**
- **Router Link States**, one LSA per router, so you should see Router IDs 1.1.1.1 through 4.4.4.4
- **Link ID**, the Router ID of the router that originated each LSA
- **Age**, the age of the LSA in seconds, refreshed every 1800 seconds
### 2.4 — Inspect OSPF Interface Details
 
```
R1# show ip ospf interface GigabitEthernet0/0
R1# show ip ospf interface Loopback0
```
 
Put the two outputs side by side. This is the command I reach for first when an adjacency will not form, because it shows the area, the cost, the hello and dead timers, the network type, and the neighbor count all in one place.
 
**✏️ Prediction checkpoint:** What network type will each interface report? Will either one show a DR or BDR? Will the hello interval match on both?
 
### 2.5 — Test End-to-End Connectivity
 
```
R1# ping 192.168.4.1 source Loopback0
R1# traceroute 192.168.4.1 source Loopback0
```
 
Always add `source Loopback0` when you test reachability between the simulated LANs. Without it, the router sources the ping from its outgoing physical interface, which can succeed even when the loopback network itself is not properly reachable. Sourcing from the loopback gives you an honest test of LAN-to-LAN connectivity. Confirm all four loopback networks reach each other before you move on.
 
---
 
## 💥 Part 3: Break It
 
> Work through each scenario one at a time. Document the symptoms fully before you move to Part 4.
 
---
 
### Break Scenario 1 — Mismatched Hello/Dead Timers
 
OSPF requires the Hello and Dead timers to match between neighbors. The defaults on Ethernet are a 10-second hello and a 40-second dead interval. Change R3's timers on its WAN link:
 
```
R3(config)# interface GigabitEthernet0/0
R3(config-if)# ip ospf hello-interval 5
R3(config-if)# ip ospf dead-interval 20
```
 
Wait 60 seconds, then observe:
 
```
R2# show ip ospf neighbor
R2# show ip route ospf
R2# show ip ospf interface GigabitEthernet0/1
R3# show ip ospf interface GigabitEthernet0/0
```
 
**✏️ Document the symptoms:**
- What happened to R2's neighbor relationship with R3?
- Are routes to 192.168.3.0/24 still present on R1?
- What do the timer values look like on each side of the R2 to R3 link?
> **Why this matters:** Timer mismatches cause neighbors to flap, forming an adjacency and then dropping it over and over. The network appears to work intermittently, which is one of the most frustrating failure modes to diagnose if you do not already know to compare the timers. Now you do.
 
---
 
### Break Scenario 2 — Mismatched Area ID
 
Restore R3's timers first, then move R3's WAN link into the wrong area:
 
```
R3(config)# router ospf 1
R3(config-router)# no network 10.0.23.0 0.0.0.3 area 0
R3(config-router)# network 10.0.23.0 0.0.0.3 area 1
```
 
Wait 60 seconds, then observe:
 
```
R2# show ip ospf neighbor
R3# show ip ospf neighbor
R2# show ip ospf interface GigabitEthernet0/1
R3# show ip ospf interface GigabitEthernet0/0
```
 
**✏️ Document the symptoms:**
- Does R2 show R3 as a neighbor at all?
- What area does each router report for its side of the link?
- Is there any log message? Check `show logging` on both routers.
> **Why this matters:** An area mismatch prevents the adjacency silently. The two routers simply ignore each other's hellos, and IOS gives you no error that says "wrong area." You have to know to check the per-interface area assignment, which is exactly why I am having you look at it now.
 
---
 
### Break Scenario 3 — Duplicate Router ID
 
Restore R3's area, then force R4 to claim R3's Router ID:
 
```
R4(config)# router ospf 1
R4(config-router)# router-id 3.3.3.3
R4(config-router)# exit
R4# clear ip ospf process
! Type 'yes' when prompted
```
 
Wait 60 seconds, then observe:
 
```
R2# show ip ospf neighbor
R2# show ip ospf database
R2# show logging
```
 
**✏️ Document the symptoms:**
- Does R2 show both R3 and R4 as neighbors, or does one disappear?
- Are there duplicate entries in the database?
- Can R1 still reach 192.168.4.0/24?
- What warnings appear in `show logging`?
> **Why this matters:** Duplicate Router IDs produce unpredictable behavior that is genuinely hard to chase down, because the routers disagree about which LSA belongs to whom. In production this usually happens when someone clones a router config and forgets to change the Router ID, which is more common than you might expect.
 
---
 
## 🔧 Part 4: Fix It
 
---
 
### Fix Scenario 1 — Mismatched Hello/Dead Timers
 
**Your troubleshooting toolkit:**
```
show ip ospf neighbor
show ip ospf interface GigabitEthernet0/0   ! on both sides of the link
```
 
**Structured process:**
1. Confirm the neighbor is missing or stuck in a state below Full.
2. Run `show ip ospf interface` on both sides and compare the Hello and Dead timers.
3. Identify the router carrying the non-default timers.
4. Remove the custom timers to restore the defaults.
**✅ Solution:**
```
R3(config)# interface GigabitEthernet0/0
R3(config-if)# no ip ospf hello-interval
R3(config-if)# no ip ospf dead-interval
```
 
**Verification:**
```
R2# show ip ospf neighbor
R1# show ip route ospf
R1# ping 192.168.3.1 source Loopback0
```
 
---
 
### Fix Scenario 2 — Mismatched Area ID
 
**Your troubleshooting toolkit:**
```
show ip ospf neighbor
show ip ospf interface GigabitEthernet0/0     ! on both sides
show running-config | section ospf
```
 
**Structured process:**
1. Confirm the neighbor is entirely absent from `show ip ospf neighbor`.
2. Run `show ip ospf interface` on both sides and note the area each reports.
3. Compare `show running-config | section ospf` on both routers.
4. Correct the area and verify.
**✅ Solution:**
```
R3(config)# router ospf 1
R3(config-router)# no network 10.0.23.0 0.0.0.3 area 1
R3(config-router)# network 10.0.23.0 0.0.0.3 area 0
```
 
**Verification:**
```
R2# show ip ospf neighbor
R1# show ip route ospf
R1# ping 192.168.3.1 source Loopback0
```
 
---
 
### Fix Scenario 3 — Duplicate Router ID
 
**Your troubleshooting toolkit:**
```
show ip ospf neighbor
show ip ospf database
show logging
```
 
**Structured process:**
1. Look for duplicate Link IDs in `show ip ospf database`.
2. Check `show logging`, since IOS logs a warning when it detects a duplicate Router ID.
3. Identify the router carrying the conflicting ID.
4. Correct the Router ID and reset the OSPF process.
**✅ Solution:**
```
R4(config)# router ospf 1
R4(config-router)# router-id 4.4.4.4
R4(config-router)# exit
R4# clear ip ospf process
! Type 'yes' when prompted
```
 
> **Production note:** `clear ip ospf process` drops every OSPF adjacency and forces a full reconvergence. On a live network that means a brief outage, so plan a maintenance window before you change a Router ID in production.
 
**Verification:**
```
R2# show ip ospf neighbor
R2# show ip ospf database
R1# ping 192.168.4.1 source Loopback0
```
 
---
 
## 💬 Reflection Questions
 
1. OSPF cost is calculated as 100,000,000 divided by bandwidth in bits per second. What is the cost of a 1 Gbps link, and what about a 10 Gbps link? What problem does the default reference bandwidth create on modern networks, and how do you correct it?
2. You added `ip ospf network point-to-point` to each loopback. What would have appeared in everyone else's routing table for 192.168.1.0/24 if you had left that command off? Explain why.
3. You set `passive-interface` on every loopback. If you forgot one, would OSPF still work? What is the practical downside of letting hellos leave an interface that has no neighbors?
4. In Break Scenario 1, the timer mismatch dropped the adjacency. Why does OSPF insist that Hello and Dead timers match between neighbors? What would go wrong if it did not enforce that?
5. In Break Scenario 2, the area mismatch produced no error message. The routers simply ignored each other. Why does IOS stay silent here, and what does that silence teach you about how to verify OSPF systematically rather than waiting for an error to point the way?
6. You used `clear ip ospf process` to make R4 adopt its corrected Router ID. Why does a Router ID change not take effect immediately on its own? What does OSPF do during the reset?
---
 
## Challenge Extension (Optional)
 
1. **Adjust the reference bandwidth.** Run `auto-cost reference-bandwidth 1000` on all four routers to set the reference to 1 Gbps. Watch how the cost values shift in the routing table, then explain why this command has to be applied identically on every router in the OSPF domain.
2. **Influence path selection.** Set the OSPF cost on R2's G0/1 interface to 100. Does R1's path to 192.168.3.0/24 change? Verify with `show ip route ospf` and `traceroute 192.168.3.1 source Loopback0`.
3. **Observe convergence in real time.** Shut down R2's G0/1 interface, the link to R3. Watch `show ip ospf neighbor` on R2 and `show ip route ospf` on R1. How long does reconvergence take, and which timer controls how quickly R2 decides R3 is gone?
4. **Explore the DR/BDR election.** Change R1's G0/0 interface to a broadcast network type with `ip ospf network broadcast`, and do the same on R2's matching interface. What changes in `show ip ospf neighbor`? Which router becomes the DR, how did OSPF decide, and what would you change to influence that election?
---
 
## Where This Leads
 
You have now built the protocol the CCNA exam cares about most, and more importantly, you have broken it in the three ways it most often breaks in the field. The timer mismatch that flaps, the area mismatch that fails in silence, and the duplicate Router ID that sows confusion are not just exam trivia. They are the calls you will get as a working engineer, and you have already solved each one.
 
From here, the natural next step is to carry these OSPF skills into the switching world, where inter-VLAN routing will ask a router to make forwarding decisions for traffic between VLANs. The routing logic you have built across this entire section is exactly what makes that topic click. When you are ready, the Switching section is waiting.
 
---
 
*CCNA – Build It · Observe It · Break It · Fix It series*
