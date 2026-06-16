# Routing
 
How do packets find their way across a network? That's the question this entire section answers.
 
These labs take you from manually configuring every route by hand, through the automation of dynamic routing protocols, to the protocol the CCNA exam actually tests: single-area OSPFv2. Work through them in order — each one builds intuition the next one relies on.
 
---
 
## Who This Section Is For
 
- **Students** who have completed the CML Foundations section and are ready to dig into how routing works
- **Instructors** looking for classroom-ready routing labs aligned to the CCNA exam domains
- Anyone who wants to understand not just *how* to configure routing, but *why* routers make the decisions they do
**Before you start this section, you should be comfortable with:**
- Building and navigating a topology in CML (see CML Foundations)
- Basic IOS device configuration — hostnames, interfaces, IP addressing
- Verifying connectivity with `ping`, `traceroute`, and `show ip interface brief`
If any of that feels shaky, spend more time in the Foundations section first. Routing builds directly on those skills.
 
---
 
## The Learning Philosophy
 
Every lab in this section follows the same deliberate cycle:
 
> ### 🧱 Build It · 👀 Observe It · 💥 Break It · 🔧 Fix It
 
You build a working routed network. You observe how routes propagate and how the router chooses paths. You intentionally break it with wrong next-hops, missing routes, mismatched timers, duplicate router IDs. Then you troubleshoot your way back using a structured process.
 
Routing is where this methodology really earns its keep. The CCNA exam is full of "this network isn't working, why?" scenarios, and the only way to get good at those is to break real networks and fix them. Reading about a missing return route teaches you the concept; watching a ping time out and tracing it back to the missing route teaches you the *skill*.
 
---
 
## The Labs in This Section
 
### 🗺️ Lab 01 — Static Routing
**`lab-01-static-routing`**
 
The foundation of all IP routing. Before you can appreciate dynamic protocols, you need to understand what a router actually does when it forwards a packet — and why it sometimes doesn't.
 
You'll manually configure static routes across a three-router topology, use loopback interfaces to simulate LAN segments, and configure a default route. The break/fix scenarios cover the three most common static routing mistakes: the missing return route, the wrong next-hop, and the asymmetric path.
 
**You'll learn:**
- Configuring IPv4 static routes with next-hop addressing
- Configuring a default static route (`0.0.0.0/0`)
- Interpreting connected, local, and static routes in the routing table
- The role of administrative distance and the value of sourced pings in troubleshooting
**CCNA aligned:** ✅ Yes (3.1, 3.2, 3.3)
**Nodes:** 5 (3 routers + 2 PCs, with loopbacks)
 
---
 
### 📡 Lab 02 — RIPv2 Routing
**`lab-02-rip-routing`**
 
> **Supplementary / Foundational — not on the current CCNA exam.** RIP was removed in the 2020 revision. Included here as the gentlest possible introduction to *dynamic* routing concepts before tackling OSPF.
 
Your first dynamic routing protocol. RIP is the simplest one still in use, which makes it an excellent way to build intuition for automatic route sharing, convergence, metrics, and administrative distance — before the heavier OSPF material.
 
You'll configure RIPv2, watch routers exchange their tables in real time with `debug ip rip`, and troubleshoot version mismatches, the auto-summary trap, and the missing network statement.
 
**You'll learn:**
- Distance-vector routing fundamentals
- RIPv2 configuration, including `no auto-summary`
- Verifying dynamic routing with `show ip protocols` and `debug ip rip`
- How hop count and administrative distance shape routing decisions
**CCNA aligned:** ⚠️ No — supplementary/foundational only
**Nodes:** 5 (3 routers + 2 PCs, with loopbacks — same topology as Lab 01)
 
---
 
### 🎯 Lab 03 — Default Routing
**`lab-03-default-routing`**
 
What happens when a router needs to reach a destination it has no specific route for? It uses the default route — the "when in doubt, send it here" route that underpins how stub networks and internet edge routers operate.
 
This lab focuses specifically on the default route (`0.0.0.0/0`): when to use one, how it interacts with more specific routes, and how routers apply the longest-prefix-match rule. You'll build a stub topology, configure a gateway of last resort, and troubleshoot scenarios where default and specific routes conflict.
 
**You'll learn:**
- Configuring a gateway of last resort with a default static route
- How longest-prefix match decides between a default and a specific route
- The role of default routes in stub and edge network designs
- Recognizing and fixing default-route misconfigurations
**CCNA aligned:** ✅ Yes (part of 3.3)
**Nodes:** Within the CML Free 5-node limit
 
---
 
### 🌐 Lab 04 — Single-Area OSPF
**`lab-04-single-area-ospf`**
 
The main event for CCNA routing. OSPFv2 is *the* dynamic routing protocol the exam tests, and this lab covers it thoroughly.
 
You'll build a four-router single-area topology with loopbacks simulating LAN segments, watch neighbor adjacencies form and the link-state database synchronize, then break the network with the three most common real-world OSPF failures: mismatched hello/dead timers, mismatched area IDs, and duplicate router IDs.
 
**You'll learn:**
- Configuring single-area OSPFv2 across multiple routers
- The OSPF neighbor state machine and adjacency formation
- Interpreting the Link State Database (LSDB)
- How OSPF cost determines path selection
- Troubleshooting the three classic OSPF adjacency failures
**CCNA aligned:** ✅ Yes (3.4, 3.4.a, 3.4.b, 3.4.c)
**Nodes:** 4 routers (loopbacks simulate all LAN segments)
 
---
 
## Recommended Path
 
```
Lab 01            Lab 02            Lab 03            Lab 04
Static     →      RIPv2      →      Default    →      Single-Area
Routing           (optional)        Routing           OSPF
   │                 │                 │                 │
Manual           First taste       Gateway of        The exam's
route            of dynamic        last resort       dynamic
control          routing                             protocol
```
 
**Short on time and focused purely on the exam?** Do Labs 01, 03, and 04. Lab 02 (RIP) is valuable for building intuition but isn't tested — skip it if you need to prioritize.
 
**Building a complete conceptual foundation?** Do all four in order. The progression from manual → distance-vector → default → link-state mirrors how routing itself evolved, and each lab makes the next one click faster.
 
---
 
## Where to Go Next
 
Once you've worked through routing, you're ready to branch into the other topic areas:
 
- **Switching** — VLANs, trunking, inter-VLAN routing, STP
- **Services** — DHCP, NAT, DNS
- **Security** — SSH, ACLs
Inter-VLAN routing in the Switching section, in particular, will feel much easier now that you understand how routers make forwarding decisions.
 
---
 
## A Note Before You Begin
 
Routing is where a lot of students either fall in love with networking or get intimidated by it. The difference almost always comes down to one thing: whether they treated a broken network as a failure or as a puzzle.
 
In here, every broken network is a puzzle — and you're given the tools to solve it. Take your time, predict before you verify, and when something breaks, resist the urge to immediately look at the solution. Sit with the symptoms first. That's where the real skill is built.
 
**Build it. Observe it. Break it. Fix it.**
 
---
 
*Part of the CCNA – Build It · Observe It · Break It · Fix It lab series.*
*This repository is an independent educational resource and is not officially affiliated with Cisco Systems. Cisco, CCNA, Cisco Modeling Labs, and Cisco Networking Academy are trademarks of Cisco Systems, Inc.*
