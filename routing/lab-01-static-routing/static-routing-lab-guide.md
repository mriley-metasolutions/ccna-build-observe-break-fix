# Static Routing

**Build It · Observe It · Break It · Fix It**

---

## Overview

Static routing is the foundation of all IP routing. Before a network engineer can appreciate dynamic routing protocols, they must understand what a router actually does when it forwards a packet — and why it sometimes doesn't.

In this lab you will manually configure static routes across a three-router topology, use loopback interfaces to simulate LAN segments, observe exactly how traffic flows, deliberately break the network in realistic ways, and troubleshoot your way back to full connectivity.

**CCNA Exam Alignment:**
- 3.1 – Interpret the components of a routing table
- 3.2 – Determine how a router makes a forwarding decision by default
- 3.3 – Configure and verify IPv4 static routing (including default routes)

**Estimated Time:** 60–90 minutes

---

## A Note on Loopback Interfaces

This lab uses **loopback interfaces** on R1 and R3 to simulate LAN segments instead of connecting physical PCs. This is intentional — and worth understanding.

A loopback interface is a **virtual, software-only interface** that is always up as long as the router is running. It never goes down due to a cable pull or a connected device powering off. In production networks, loopbacks are used for:

- **Router management** — a stable IP address you can always reach the router on
- **OSPF/BGP Router IDs** — a reliable, always-up source for the Router ID
- **Simulating networks in labs** — exactly what we are doing here

On the CCNA exam, you will see loopbacks used in both lab tasks and troubleshooting scenarios. Getting comfortable with them now pays off.

> **Real-world callout:** When you see a network engineer SSH into a router using an address like `10.0.0.1/32`, they are almost certainly connecting to a loopback. It works regardless of which physical path the traffic takes to get there.

---

## Topology

```
[Lo0: 192.168.1.0/24]        [Lo0: 192.168.3.0/24]
        |                              |
       R1 -------- R2 -------- R3
   10.0.12.1    10.0.12.2  10.0.23.1  10.0.23.2

        PC1 connected to R1 G0/0
        PC2 connected to R2 G0/0
```

> **Full topology file:** `topology.yaml` (import into CML)

**Nodes (5 total — within CML Free Edition limit):**
- R1, R2, R3 (IOSv routers)
- PC1 connected to R1
- PC2 connected to R2

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

> **Why /30 on WAN links?** The point-to-point links between routers use /30 subnets (255.255.255.252), giving exactly 2 usable host addresses. This is standard practice for router-to-router links and something the CCNA exam expects you to recognize.

> **Why /32 isn't used here:** Loopback interfaces are often configured with a /32 (host route) in production. In this lab we use /24 so the loopback simulates a realistic LAN segment that can be summarized and routed like any other subnet. Both are valid — and worth knowing the difference.

---

## Lab Objectives

By the end of this lab you will be able to:

1. Configure loopback interfaces and explain their purpose
2. Configure IPv4 static routes using next-hop IP address syntax
3. Configure a default static route (`0.0.0.0/0`)
4. Interpret routing table entries including connected, static, and local routes
5. Verify end-to-end connectivity across a multi-router topology
6. Identify the symptoms and root cause of three common static routing misconfigurations
7. Apply a structured troubleshooting process to restore connectivity

---

## 🧱 Part 1: Build It

### Step 1 — Configure Hostnames, Interfaces, and Loopbacks

**R1:**
```
Router> enable
Router# configure terminal
Router(config)# hostname R1

! Physical LAN interface — connects to PC1
R1(config)# interface GigabitEthernet0/0
R1(config-if)# description LAN - PC1 Segment
R1(config-if)# ip address 192.168.1.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit

! Point-to-point WAN link to R2
R1(config)# interface GigabitEthernet0/1
R1(config-if)# description WAN - Link to R2
R1(config-if)# ip address 10.0.12.1 255.255.255.252
R1(config-if)# no shutdown
R1(config-if)# exit

! Loopback — simulates a remote LAN on R1's side
! Note: no 'no shutdown' needed — loopbacks are always up
R1(config)# interface Loopback0
R1(config-if)# description Simulated LAN on R1
R1(config-if)# ip address 192.168.10.1 255.255.255.0
R1(config-if)# exit
```

**R2:**
```
Router> enable
Router# configure terminal
Router(config)# hostname R2

! Physical LAN interface — connects to PC2
R2(config)# interface GigabitEthernet0/0
R2(config-if)# description LAN - PC2 Segment
R2(config-if)# ip address 192.168.2.1 255.255.255.0
R2(config-if)# no shutdown
R2(config-if)# exit

! WAN link to R1
R2(config)# interface GigabitEthernet0/1
R2(config-if)# description WAN - Link to R1
R2(config-if)# ip address 10.0.12.2 255.255.255.252
R2(config-if)# no shutdown
R2(config-if)# exit

! WAN link to R3
R2(config)# interface GigabitEthernet0/2
R2(config-if)# description WAN - Link to R3
R2(config-if)# ip address 10.0.23.1 255.255.255.252
R2(config-if)# no shutdown
R2(config-if)# exit
```

**R3:**
```
Router> enable
Router# configure terminal
Router(config)# hostname R3

! WAN link back to R2 — R3 has no physical LAN in this topology
R3(config)# interface GigabitEthernet0/0
R3(config-if)# description WAN - Link to R2
R3(config-if)# ip address 10.0.23.2 255.255.255.252
R3(config-if)# no shutdown
R3(config-if)# exit

! Loopback — simulates a remote LAN on R3's side
R3(config)# interface Loopback0
R3(config-if)# description Simulated LAN on R3
R3(config-if)# ip address 192.168.30.1 255.255.255.0
R3(config-if)# exit
```

---

### Step 2 — Configure Static Routes

> **Key concept:** Routing is always two-way. A packet must be able to get there *and* the reply must be able to get back. The most common static routing mistake is configuring the forward route but forgetting the return route.

> **Best practice reminder:** Always use the **next-hop IP address** for static routes on Ethernet interfaces — not the exit interface name. See the note in the lab introduction for why.

**R1** needs routes to R2's LAN, R3's loopback, and the far WAN segment:
```
R1(config)# ip route 192.168.2.0 255.255.255.0 10.0.12.2
R1(config)# ip route 192.168.30.0 255.255.255.0 10.0.12.2
R1(config)# ip route 10.0.23.0 255.255.255.252 10.0.12.2
```

**R2** sits in the middle and needs routes in both directions:
```
R2(config)# ip route 192.168.1.0 255.255.255.0 10.0.12.1
R2(config)# ip route 192.168.10.0 255.255.255.0 10.0.12.1
R2(config)# ip route 192.168.30.0 255.255.255.0 10.0.23.2
```

**R3** is a stub router — it only connects one direction. A default route covers everything:
```
! A default route matches ANY destination not in the routing table.
! This is how stub routers and home routers are typically configured.
R3(config)# ip route 0.0.0.0 0.0.0.0 10.0.23.1
```

---

### Step 3 — Save Configurations

On each router:
```
# copy running-config startup-config
```

---

## 👀 Part 2: Observe It

### 2.1 — Examine the Routing Table

```
R1# show ip route
```

**What to look for:**

| Code | Meaning |
|------|---------|
| `C`  | Directly connected network |
| `L`  | Local route — the router's own interface IP as a /32 |
| `S`  | Static route you configured |
| `S*` | Candidate default route |

The `[1/0]` after a static route means `[Administrative Distance / Metric]`. AD of 1 means static routes are trusted more than any dynamic routing protocol except connected routes (AD 0).

**✏️ Prediction checkpoint:** Before running `show ip route` on R3, write down how many entries you expect to see and what codes they will have. Will the loopback show up as `C` or something else?

### 2.2 — Verify Loopback Interfaces

```
R1# show interfaces Loopback0
R3# show interfaces Loopback0
```

**What to look for:**
- The line protocol should show `up/up` — confirm this and note why a loopback can never be `down/down`
- The IP address assigned

### 2.3 — Test Connectivity Layer by Layer

Always test from closest to farthest. This isolates failures to a specific segment.

```
! From R1 — test the directly connected WAN link first
R1# ping 10.0.12.2

! Test R2's LAN gateway
R1# ping 192.168.2.1

! Test end-to-end to PC2
R1# ping 192.168.2.10

! Test R3's loopback (simulated far LAN)
R1# ping 192.168.30.1

! Test from PC1 to PC2
PC1> ping 192.168.2.10
```

### 2.4 — Trace the Path

```
R1# traceroute 192.168.30.1
```

**✏️ Prediction checkpoint:** Write down every IP address you expect to appear as a hop before running the command. How many hops total?

### 2.5 — Inspect a Specific Route

```
R1# show ip route 192.168.30.0
```

This shows exactly which routing table entry will be used for a given destination — useful during troubleshooting to confirm the router is using the route you think it is.

---

## 💥 Part 3: Break It

> Work through each scenario one at a time. Document symptoms fully before moving to Part 4 to fix it.

---

### Break Scenario 1 — The Missing Return Route

On **R3**, remove the default route:
```
R3(config)# no ip route 0.0.0.0 0.0.0.0 10.0.23.1
```

**Now test from R1:**
```
R1# ping 192.168.30.1
R1# ping 10.0.23.2
```

**✏️ Document the symptoms:**
- What does the ping output look like — timeout, or `U` (unreachable)?
- Can R3 still ping R2's G0/2 address (`10.0.23.1`)? Why?
- Is the behavior different when pinging R3's loopback vs. pinging R3's physical interface?

> **Why this matters:** The packet gets to R3, but the reply has no route back. The source sees a timeout — not an unreachable message — which makes this failure mode deceptive. Many students assume a timeout means the packet never arrived. It may have arrived just fine.

---

### Break Scenario 2 — The Wrong Next Hop

Restore R3's default route. Then on **R1**, introduce a typo in the next-hop:
```
R1(config)# no ip route 192.168.30.0 255.255.255.0 10.0.12.2
R1(config)# ip route 192.168.30.0 255.255.255.0 10.0.12.3
```

> `10.0.12.3` does not exist in this topology.

**Now test:**
```
R1# ping 192.168.30.1
R1# show ip route 192.168.30.0
```

**✏️ Document the symptoms:**
- Does the route still appear in the routing table? What next-hop does it show?
- What happens when you ping?
- How is the ping output different from Scenario 1?

> **Why IOS allows this:** IOS installs a static route with a non-existent next-hop because it assumes the next-hop may become reachable later (e.g. after another interface comes up). The route is in the table but is effectively a black hole.

---

### Break Scenario 3 — The Asymmetric Path

On **R2**, remove only the return route toward R1's loopback:
```
R2(config)# no ip route 192.168.10.0 255.255.255.0 10.0.12.1
```

**Now test:**
```
R1# ping 192.168.30.1 source Loopback0
R1# ping 192.168.30.1
```

**✏️ Document the symptoms:**
- What is the difference in results between the two pings above?
- Why does the `source Loopback0` keyword change the behavior?
- Run `traceroute 192.168.30.1 source Loopback0` — what do you observe?

> **The `source` keyword:** When a router pings without a source specified, it uses the IP of the outgoing interface — which may have a valid return path even when the loopback network doesn't. Using `source Loopback0` forces the ping to originate from `192.168.10.1`, exposing the missing return route. This is a critical troubleshooting technique.

---

## 🔧 Part 4: Fix It

---

### Fix Scenario 1 — The Missing Return Route

**Your troubleshooting toolkit:**
```
show ip route
show ip route 0.0.0.0
ping <destination>
traceroute <destination>
```

**Structured process:**
1. Ping the destination — does it time out or return unreachable?
2. Run `traceroute` — how far does the packet get before it stops?
3. Log into the router at the far end — does it have a route back to your source?
4. Add the missing route and verify

**✅ Solution:**
```
R3(config)# ip route 0.0.0.0 0.0.0.0 10.0.23.1
```

**Verification:**
```
R3# show ip route
R1# ping 192.168.30.1 repeat 5
```

---

### Fix Scenario 2 — The Wrong Next Hop

**Your troubleshooting toolkit:**
```
show ip route 192.168.30.0
ping 10.0.12.2
ping 10.0.12.3
```

**Structured process:**
1. Check the routing table — what next-hop does the route show?
2. Ping that next-hop directly — is it reachable?
3. If the next-hop is unreachable, the route is a black hole regardless of what the table says
4. Correct the next-hop and verify

**✅ Solution:**
```
R1(config)# no ip route 192.168.30.0 255.255.255.0 10.0.12.3
R1(config)# ip route 192.168.30.0 255.255.255.0 10.0.12.2
```

**Verification:**
```
R1# show ip route 192.168.30.0
R1# ping 192.168.30.1
```

---

### Fix Scenario 3 — The Asymmetric Path

**Your troubleshooting toolkit:**
```
show ip route                              ! on R1 and R2
ping 192.168.30.1 source Loopback0        ! sourced ping from R1
traceroute 192.168.30.1 source Loopback0  ! sourced traceroute from R1
```

**Structured process:**
1. Notice that an unsourced ping works but a sourced ping from Loopback0 fails
2. This tells you the problem is with the *return path* to 192.168.10.0/24
3. Check R2's routing table — is there a route back to 192.168.10.0/24?
4. Add the missing route and verify

**✅ Solution:**
```
R2(config)# ip route 192.168.10.0 255.255.255.0 10.0.12.1
```

**Verification:**
```
R2# show ip route
R1# ping 192.168.30.1 source Loopback0 repeat 5
```

---

## 💬 Reflection Questions

1. What is the administrative distance of a static route, and why does a lower AD value mean higher priority? What is the AD of a directly connected route?

2. R3 used a default route (`0.0.0.0/0`) while R1 and R2 used specific routes. In what real-world scenarios would you prefer a default route over specific static routes? What are the risks?

3. In Break Scenario 1, the ping timed out rather than returning a "Destination Unreachable" message. Explain why. What does each response type tell you about where in the path the problem exists?

4. You configured a static route pointing to `10.0.12.3`, which doesn't exist, and IOS installed it in the routing table anyway. Why does IOS allow this? How could this behavior be used intentionally with a **floating static route**?

5. In Break Scenario 3, pinging without a source worked but pinging with `source Loopback0` failed. Explain why the source address of a ping matters during troubleshooting. When would you always want to use a sourced ping?

6. Loopback interfaces are configured with `no shutdown` already in effect — you never need to type it. Why are loopbacks always up? What does this tell you about how IOS treats virtual vs. physical interfaces?

---

## Challenge Extension (Optional)

1. **Summarize routes on R2.** R2 currently has separate routes to `192.168.1.0/24` and `192.168.10.0/24`. Can you write a single summary route that covers both? What is the risk of over-summarizing?

2. **Floating static route.** Add a second loopback on R2 (`Loopback1: 172.16.0.1/32`) and write a static route on R1 to reach it with AD 5. Then write a second static route to the same destination with AD 200. Which one appears in the routing table? Remove the AD 5 route — what happens?

3. **Simulate an ISP connection.** On R3, add a second loopback (`Loopback1: 8.8.8.8/32`) representing an internet destination. Verify that R1 can reach it via the default route on R3 combined with R2's static routes. Trace the full path.

4. **Recursive routing.** On R1, configure a static route using R2's loopback as the next-hop instead of R2's physical interface IP. What happens? What is recursive lookup, and why can it cause problems?

---

*CCNA – Build It · Observe It · Break It · Fix It series*
*Next: OSPF Single Area*
