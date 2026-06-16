# Default Routing
 
**Build It · Observe It · Break It · Fix It**
 
---
 
## Overview
 
When a router receives a packet for a destination it has no specific route to, it drops the packet. The default route is how you prevent that. It is the route a router falls back on when nothing else in its table matches, and it quietly carries the majority of internet-bound traffic on most networks you will ever touch.
 
A default route matches any destination not covered by a more specific entry. You write it as `0.0.0.0/0`, a notation that represents every possible IPv4 address at once. Because it matches everything, it is always the least specific route in the table, which means a router only reaches for it after every more specific option has failed.
 
In ten years of teaching this topic, I have found that students grasp default routes quickly but stumble on how a router *chooses* between a default route and a specific one. That choice is the real lesson here, so I have built the lab around it. You will configure a stub-and-edge topology modeled on a branch office connecting to an ISP, watch how each router decides where to send traffic, then break the network in three realistic ways and troubleshoot your way back.
 
**CCNA Exam Alignment:**
- 3.3 – Configure and verify IPv4 static routing, including default routing
**Estimated Time:** 60–90 minutes
 
---
 
## Key Concept: Longest Prefix Match
 
I want you to understand one rule before you configure anything, because nearly every confusing default routing problem traces back to it. When a router forwards a packet, it compares the destination against every route in its table and selects the **longest prefix match**, meaning the most specific route that still contains the destination.
 
Picture a router that holds all three of these routes and receives a packet for `8.8.8.8`:
 
| Route | Prefix Length | Matches 8.8.8.8? |
|-------|---------------|------------------|
| `8.8.8.0/24` | /24 | Yes (most specific) |
| `8.0.0.0/8` | /8 | Yes |
| `0.0.0.0/0` | /0 | Yes (least specific) |
 
All three match. The router picks the `/24` because it is the longest prefix. The default route, sitting at `/0`, only wins when nothing else matches at all. This is why people call a default route the **gateway of last resort**. It is the fallback, used only after every more specific option has been ruled out.
 
Hold onto this rule. When you reach the Break It section, you will watch a forgotten specific route quietly hijack traffic that should have followed the default, and longest prefix match is the key that unlocks the whole mystery.
 
---
 
## Topology
 
```
[PC1]                                    [Simulated Internet]
  |                                       Lo0: 8.8.8.8
[R1 G0/0]                                 Lo1: 1.1.1.1
192.168.1.1                                    |
[R1 G0/1]                                  [R3 G0/0]
10.0.12.1                                  10.0.23.2
  |                                            |
10.0.12.2                                  10.0.23.1
[R2 G0/1]                                  [R2 G0/2]
[R2 G0/0]
192.168.2.1
  |
[PC2]
 
  R1 = Branch (stub) router
  R2 = HQ / transit router
  R3 = ISP router (simulated internet)
```
 
> **Full topology file:** `topology.yaml`
 
**Nodes (5 total, within CML Free Edition limit):**
- R1, R2, R3 (IOSv routers)
- PC1 connected to R1
- PC2 connected to R2
---
 
## Addressing Table
 
| Device | Interface  | IP Address    | Subnet Mask       | Default Gateway |
|--------|------------|---------------|-------------------|-----------------|
| R1     | G0/0       | 192.168.1.1   | 255.255.255.0     | —               |
| R1     | G0/1       | 10.0.12.1     | 255.255.255.252   | —               |
| R2     | G0/0       | 192.168.2.1   | 255.255.255.0     | —               |
| R2     | G0/1       | 10.0.12.2     | 255.255.255.252   | —               |
| R2     | G0/2       | 10.0.23.1     | 255.255.255.252   | —               |
| R3     | G0/0       | 10.0.23.2     | 255.255.255.252   | —               |
| R3     | Loopback0  | 8.8.8.8       | 255.255.255.255   | —               |
| R3     | Loopback1  | 1.1.1.1       | 255.255.255.255   | —               |
| PC1    | NIC        | 192.168.1.10  | 255.255.255.0     | 192.168.1.1     |
| PC2    | NIC        | 192.168.2.10  | 255.255.255.0     | 192.168.2.1     |
 
> **About the simulated internet:** R3 stands in for your ISP and the wider internet. Its two loopback addresses, `8.8.8.8` and `1.1.1.1`, are real public DNS addresses I borrowed as recognizable stand-ins for "somewhere out on the internet." The /32 masks mean each one is a single host destination.
 
---
 
## Lab Objectives
 
By the end of this lab you will be able to:
 
1. Explain the purpose of a default route and the term "gateway of last resort"
2. Configure a default static route using `0.0.0.0 0.0.0.0`
3. Describe how longest prefix match selects between a default and a specific route
4. Identify the gateway of last resort in `show ip route` output
5. Build a realistic stub-and-edge design with chained default routes
6. Identify and resolve three common default routing failures
7. Apply a structured troubleshooting process to restore connectivity
---
 
## 🧱 Part 1: Build It
 
> This lab assumes interfaces are already configured with the IP addresses in the addressing table. If you are starting fresh, configure those first (the Static Routing lab's Build section has the interface commands). This section focuses on the routing configuration itself.
 
### Step 1 — Configure the Stub Router (R1)
 
R1 is a branch router with exactly one path out, the link to R2. That single exit is the whole reason a default route fits here so well. Everything that is not on R1's own LAN gets sent to R2, and you can express that with one line.
 
```
R1> enable
R1# configure terminal
 
! A single default route sends ALL non-local traffic to R2.
! This is the entire routing policy for a stub router.
R1(config)# ip route 0.0.0.0 0.0.0.0 10.0.12.2
R1(config)# exit
```
 
> **Why a default route here:** R1 has no reason to learn every individual network beyond it. There is only one way out, so a default route is the simplest and most efficient choice. Your home router does exactly this, and so does nearly every branch office router in production.
 
### Step 2 — Configure the Transit Router (R2)
 
R2 sits in the middle, and it is the router I want you to study most closely. It needs **specific routes** for the internal networks it knows about and a **default route** pointing toward the ISP for everything else. This blend of specific and default routes is the heart of the lab.
 
```
! Specific route back to R1's LAN
R2(config)# ip route 192.168.1.0 255.255.255.0 10.0.12.1
 
! Default route toward the ISP for all internet-bound traffic
R2(config)# ip route 0.0.0.0 0.0.0.0 10.0.23.2
R2(config)# exit
```
 
> **Notice the design:** R2 knows the internal networks specifically and sends everything else to the ISP by default. When PC2 sends traffic to `8.8.8.8`, R2 has no specific route for it, so the default route takes over and forwards it to R3. Watch for this behavior in the Observe section, because it is longest prefix match deciding in real time.
 
### Step 3 — Configure the ISP Router (R3)
 
R3 represents the ISP. In the real world an ISP would not hold a route to your private addresses unless you arranged it, but for this lab R3 needs return routes so traffic can come back. I use a single summary route to cover both customer LANs.
 
```
! A summary route covering both internal LANs (192.168.1.0/24 and 192.168.2.0/24).
! 192.168.0.0/16 covers everything in the 192.168 range.
R3(config)# ip route 192.168.0.0 255.255.0.0 10.0.23.1
R3(config)# exit
```
 
> **Why summarize here:** Rather than writing two separate routes for `192.168.1.0/24` and `192.168.2.0/24`, a single `192.168.0.0/16` summary covers both. This is a small taste of route summarization, a concept you will lean on heavily later, and it keeps the ISP's table clean the way a real ISP would want it.
 
### Step 4 — Save Configurations
 
On each router:
```
# copy running-config startup-config
```
 
---
 
## 👀 Part 2: Observe It
 
### 2.1 — Find the Gateway of Last Resort
 
```
R1# show ip route
```
 
**What to look for:**
- Near the top of the output, a line reading `Gateway of last resort is 10.0.12.2 to network 0.0.0.0`
- The default route itself, shown as `S*  0.0.0.0/0 [1/0] via 10.0.12.2`
- The `*` beside the `S`, which marks it as a candidate default route
> **The gateway of last resort line is the first thing I read on an unfamiliar router.** That single line tells you whether the router has a way to reach the wider internet and where it sends that traffic. If it says "not set," the router will drop anything it does not have a specific route for, and you have likely found your problem already.
 
**✏️ Prediction checkpoint:** Before you run the command on R2, predict what its gateway of last resort will be. Which next-hop will it point to, and why?
 
### 2.2 — Trace a Packet to the Internet
 
From PC1, trace the path to the simulated internet:
```
PC1> traceroute 8.8.8.8
```
 
**✏️ Think it through:** The packet leaves PC1, reaches R1, and R1 has no specific route for `8.8.8.8`. Walk through what happens at each hop. How many times does a default route get used along this path? Check both R1 and R2 before you answer.
 
### 2.3 — Compare Specific vs Default Decisions on R2
 
R2 is the interesting router because it carries both a specific route and a default route. Test both kinds of traffic:
 
```
! Traffic to an internal network (R2 has a SPECIFIC route)
R2# ping 192.168.1.10
 
! Traffic to the internet (R2 uses its DEFAULT route)
R2# ping 8.8.8.8
```
 
Then ask R2 to show you exactly which route it would use for each destination:
```
R2# show ip route 192.168.1.10
R2# show ip route 8.8.8.8
```
 
**✏️ Compare the output:** The first command shows R2 matching a specific `/24` route. The second shows R2 falling back to the `0.0.0.0/0` default. This is longest prefix match in action, and I find it lands much harder when you see the router name the exact route it picked rather than just reading about the rule.
 
### 2.4 — Verify Full Connectivity
 
```
PC1> ping 8.8.8.8
PC1> ping 1.1.1.1
PC2> ping 8.8.8.8
PC1> ping 192.168.2.10
```
 
Confirm that both PCs reach the simulated internet and each other before you move on.
 
---
 
## 💥 Part 3: Break It
 
> Work through each scenario one at a time. Document the symptoms fully before you move to Part 4.
 
---
 
### Break Scenario 1 — No Gateway of Last Resort
 
On **R1**, remove the default route entirely:
```
R1(config)# no ip route 0.0.0.0 0.0.0.0 10.0.12.2
```
 
Now test from PC1:
```
PC1> ping 192.168.1.1
PC1> ping 8.8.8.8
PC1> ping 192.168.2.10
```
 
Then check R1's routing table:
```
R1# show ip route
```
 
**✏️ Document the symptoms:**
- Which ping still works, and which fail?
- What does the gateway of last resort line say now?
- Can PC1 still reach its own gateway at `192.168.1.1`, and why does that still work?
> **Why this matters:** With no default route, R1 has no idea where to send traffic for anything beyond its directly connected networks. The "gateway of last resort is not set" message is one of the most valuable things you can learn to spot in `show ip route`. When a router can reach local devices but nothing beyond them, this is the first line I check.
 
---
 
### Break Scenario 2 — A Specific Route Overrides the Default
 
Restore R1's default route. Then, on **R1**, add a specific route for the `8.8.8.0/24` network that points to a wrong, non-existent next-hop:
```
R1(config)# ip route 0.0.0.0 0.0.0.0 10.0.12.2
R1(config)# ip route 8.8.8.0 255.255.255.0 10.0.12.99
```
 
> `10.0.12.99` does not exist on this network.
 
Now test from R1:
```
R1# ping 8.8.8.8
R1# ping 1.1.1.1
R1# show ip route 8.8.8.8
```
 
**✏️ Document the symptoms:**
- Can R1 still reach `1.1.1.1`? Can it reach `8.8.8.8`?
- Why does one work and the other fail, when both used to rely on the default route?
- What does `show ip route 8.8.8.8` reveal about which route the router selected?
> **Why this matters:** This is longest prefix match working exactly as designed, and it is one of the hardest problems to spot in production. The default route is still there and still correct, but the more specific `/24` route wins for any destination inside `8.8.8.0/24`. Traffic to `1.1.1.1` keeps using the default and works fine, which makes the failure look random until the rule clicks. I have watched a forgotten specific route silently steer traffic into a black hole more than once, and it is always this pattern.
 
---
 
### Break Scenario 3 — The Default Route Loop
 
Remove the bad specific route from Scenario 2. Then, on **R2**, misconfigure the default route so it points back toward R1 instead of toward the ISP:
```
R1(config)# no ip route 8.8.8.0 255.255.255.0 10.0.12.99
 
R2(config)# no ip route 0.0.0.0 0.0.0.0 10.0.23.2
R2(config)# ip route 0.0.0.0 0.0.0.0 10.0.12.1
```
 
> Now R1 sends unknown traffic to R2, and R2 sends unknown traffic back to R1. Both routers point their default at each other.
 
Now test from PC1:
```
PC1> ping 8.8.8.8
PC1> traceroute 8.8.8.8
```
 
**✏️ Document the symptoms:**
- What does the traceroute output look like, and how many hops appear before it stops?
- Do you see the same two addresses repeating back and forth?
- What eventually stops the packet from looping forever?
> **Why this matters:** When two routers point their default routes at each other, traffic for any unknown destination bounces between them until its TTL (Time To Live) expires. The traceroute shows this plainly as the same hops repeating. TTL is the safety mechanism that keeps a misrouted packet from circulating endlessly, and I think watching it actually halt a loop is one of the most instructive things you can see in a routing lab.
 
---
 
## 🔧 Part 4: Fix It
 
---
 
### Fix Scenario 1 — No Gateway of Last Resort
 
**Your troubleshooting toolkit:**
```
show ip route
ping <destination>
```
 
**Structured process:**
1. Note the symptom. Local works, everything remote fails.
2. Run `show ip route` and read the gateway of last resort line.
3. If it reads "not set," the router has no default route.
4. Add the default route and verify.
**✅ Solution:**
```
R1(config)# ip route 0.0.0.0 0.0.0.0 10.0.12.2
```
 
**Verification:**
```
R1# show ip route
PC1> ping 8.8.8.8
```
 
---
 
### Fix Scenario 2 — A Specific Route Overrides the Default
 
**Your troubleshooting toolkit:**
```
show ip route 8.8.8.8       ! see exactly which route is selected
show ip route               ! review all routes for unexpected entries
```
 
**Structured process:**
1. Notice that some internet destinations work and others fail.
2. Run `show ip route <destination>` on a failing address to see which route the router chose.
3. If the router selected a specific route you did not expect, that route is overriding your default.
4. Remove or correct the unintended specific route.
**✅ Solution:**
```
R1(config)# no ip route 8.8.8.0 255.255.255.0 10.0.12.99
```
 
**Verification:**
```
R1# show ip route 8.8.8.8
R1# ping 8.8.8.8
```
 
---
 
### Fix Scenario 3 — The Default Route Loop
 
**Your troubleshooting toolkit:**
```
traceroute <destination>    ! reveals the loop as repeating hops
show ip route               ! check where each router's default points
```
 
**Structured process:**
1. Run a traceroute and watch for the same addresses repeating.
2. That repetition is the signature of a routing loop.
3. Check the gateway of last resort on each router in the loop.
4. Correct the default route so it points toward the actual exit, the ISP.
**✅ Solution:**
```
R2(config)# no ip route 0.0.0.0 0.0.0.0 10.0.12.1
R2(config)# ip route 0.0.0.0 0.0.0.0 10.0.23.2
```
 
**Verification:**
```
R2# show ip route
PC1# traceroute 8.8.8.8
PC1# ping 8.8.8.8
```
 
---
 
## 💬 Reflection Questions
 
1. A default route is written as `0.0.0.0/0`. Explain why this notation represents "every possible destination," and why that makes it the least specific route a router can have.
2. Describe the longest prefix match rule in your own words. If a router holds both a `0.0.0.0/0` default route and a `10.0.0.0/8` route, which does it use for a packet destined to `10.5.5.5`? Which does it use for `200.1.1.1`?
3. R1 was configured with only a default route, while R2 needed both a specific route and a default route. Explain why a stub router like R1 can get away with a single default route while a transit router like R2 cannot.
4. In Break Scenario 2, traffic to `1.1.1.1` kept working while traffic to `8.8.8.8` failed, even though both had relied on the same default route moments earlier. Explain exactly why the two destinations behaved differently.
5. In Break Scenario 3, the looping packet eventually stopped. What mechanism stopped it, and what would happen on a real network if that mechanism did not exist?
6. The "gateway of last resort is not set" message appears when a router has no default route. Why is this one of the first things you should check when a router reaches local devices but cannot reach the internet?
---
 
## Challenge Extension (Optional)
 
1. **Floating default route.** Configure a second default route on R1 with a higher administrative distance, for example `ip route 0.0.0.0 0.0.0.0 10.0.12.2 200`. Research how this creates a backup path. If R1 had a second link to a different router, how would a floating default provide redundancy?
2. **Replace the summary with specific routes.** On R3, remove the `192.168.0.0/16` summary and replace it with two specific routes for `192.168.1.0/24` and `192.168.2.0/24`. Verify connectivity is identical, then weigh the tradeoffs between a summary route and specific routes.
3. **Watch a default route reach its limit.** Shut down R2's link to R3, the ISP. From PC1, ping `8.8.8.8`. The traffic still leaves R1 and reaches R2 by the default route, then stops. Trace where it dies and explain why a default route cannot help once traffic reaches the router whose exit link is down.
4. **Add a more specific route that helps instead of hurts.** On R2, add a specific `/32` route for `8.8.8.8` pointing to R3, leaving the default in place. Confirm with `show ip route 8.8.8.8` that the specific route is now preferred over the default. This is the constructive version of Break Scenario 2.
---
 
## Where This Leads
 
Default routing is the last piece you need before dynamic routing starts to feel inevitable. You have now configured every route by hand, and you have felt the friction of it: the forgotten return route, the specific route that quietly wins, the loop you have to trace by eye. Single-Area OSPF, the next lab, hands that work to the routers themselves. When you watch OSPF build its routing table automatically, I think you will appreciate exactly what it is saving you from, because you will have done all of it manually first.
 
---
 
*CCNA – Build It · Observe It · Break It · Fix It series*

*Next: Single-Area OSPF*
