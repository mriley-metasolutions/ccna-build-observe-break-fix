# DNS

**Build It · Observe It · Break It · Fix It**

---

## Overview

You have spent this entire project typing IP addresses. Ping 192.168.1.1, ping 8.8.8.8, point a name server at 10.0.12.1. No human navigates the internet that way. People type names, and something quietly turns those names into the addresses the network actually uses. That something is DNS, the Domain Name System, and it is the directory that lets you ask for a name and get back an address.

The idea is simple. A client that wants to reach a name asks a DNS server, "what is the address for this name?" and the server answers. Underneath that simple exchange sits one of the largest distributed systems humanity has ever built, a hierarchy of root servers, top-level domains, and recursive resolvers spread across the planet. This lab does not build that. What it builds is the small, honest slice of DNS that a Cisco device actually touches, which is exactly the slice the CCNA expects you to understand. You will turn one router into a simple name server that answers queries from a local table, configure another device to resolve names through it, and prove that pinging a name works just like pinging an address.

Along the way you will meet the most famous DNS-related gotcha in all of Cisco networking, the one where a mistyped command makes the router hang for half a minute while it tries to look up your typo as if it were a website. Every Cisco admin has hit it, and after this lab you will know exactly why it happens and how to stop it.

**CCNA Exam Alignment:**
- 4.3 – Explain the role of DHCP and DNS within the network

**Estimated Time:** 60–90 minutes

---

## Prerequisites

Before completing this lab, you should be comfortable with:
- Basic IOS navigation, including moving between command modes and reading `show` command output
- IPv4 addressing and verifying connectivity with `ping`
- Configuring a router as a DHCP server, which this lab builds on directly (the DHCP lab in this section)
- CML fundamentals, including building or importing a topology and accessing a device console

---

## An Honest Word About Scope

Before you build, I want to be straight with you about what this lab is and is not. A Cisco router is not a real DNS server, and you would never run production DNS on one. Real DNS is a vast hierarchical system, with root servers at the top, top-level domain servers beneath them, and recursive resolvers that walk that hierarchy on your behalf, all of it cached and replicated worldwide.

What a router can do is keep a small local table of names and addresses, answer simple queries from that table, and act as a DNS client that forwards lookups to a real server. That is genuinely useful for managing a handful of devices by name, and it is precisely what the exam tests at the describe level. So treat this lab as a clear, hands-on window into how name resolution behaves, not as a blueprint for running DNS at scale. The concepts are real even though the scale is small.

---

## Key Concept: How a Name Becomes an Address

When you type a name, resolution follows a predictable order, and knowing it tells you exactly where to look when a name fails to resolve.

1. The client checks whatever it already knows, a local cache or a static host entry, and uses that if it has a match.
2. If it does not know the answer, it sends a query to its configured **DNS server**, asking for the address that goes with the name.
3. The server answers from its own records, or in the real world, queries up the hierarchy until it finds the answer, then returns it.
4. The client caches the answer for a while and uses the address to make its connection.

In this lab, R1 plays the server and answers from a local table you build with `ip host` entries, while the clients are configured to send their queries to R1 with `ip name-server`. That is the whole loop in miniature.

> **Get ahead of the confusion:** When a name fails but the matching IP address works, the network is fine and the problem is resolution. That single distinction, name fails but address works, is the heart of DNS troubleshooting, and you will use it in every Break scenario. If you can still ping the address, stop checking cables and routing and start checking DNS.

---

## Topology

```
                 [PC1]   DHCP + DNS client
              (receives DNS server via DHCP)
                   |
                 [SW1]
                /      \
            G0/0        G0/0
            [R1]        [R2]
        192.168.1.1   192.168.1.2
        DNS SERVER    DNS CLIENT
        DHCP server   (ip name-server 192.168.1.1)
        ip host table
        Lo1: 203.0.113.10  (web.lab.local, a "server out there")
```

**Nodes (4 total, within CML Free Edition limit):**
- R1 (IOSv router, the DNS server and DHCP server)
- R2 (IOSv router, a DNS client configured explicitly)
- SW1 (IOSvL2 switch for the LAN)
- PC1 (a host that receives its DNS server through DHCP)

> **Two kinds of client, on purpose.** R2 is configured by hand with `ip name-server`, which is the explicit way. PC1 receives the same DNS server address automatically through DHCP, which is how nearly every real client learns its resolver. Seeing both is the integration payoff of this lab.

> **A note on interface names and host resolution:** Port names vary by image, so confirm with `show ip interface brief` and adjust. Host DNS behavior also varies by node type in CML, so the rock-solid demonstration here is the router client R2. Use PC1 to observe the DHCP integration if your host image supports name resolution.

---

## Addressing Table

| Device | Interface | IP Address     | Subnet Mask     | Role |
|--------|-----------|----------------|-----------------|------|
| R1     | G0/0      | 192.168.1.1    | 255.255.255.0   | DNS server, DHCP server, gateway |
| R1     | Loopback1 | 203.0.113.10   | 255.255.255.255 | Simulated web server (web.lab.local) |
| R2     | G0/0      | 192.168.1.2    | 255.255.255.0   | DNS client |
| PC1    | NIC       | DHCP           | DHCP            | DNS client via DHCP |

> **The names you will resolve (held in R1's table):**
> r1.lab.local maps to 192.168.1.1, r2.lab.local maps to 192.168.1.2, and web.lab.local maps to 203.0.113.10.

---

## Lab Objectives

By the end of this lab you will be able to:

1. Explain the role of DNS and the basic name resolution process
2. Configure a Cisco router as a simple DNS server using `ip host` and `ip dns server`
3. Configure a Cisco device as a DNS client with `ip name-server` and `ip domain lookup`
4. Hand out a DNS server address automatically through DHCP
5. Verify name resolution and read the host cache
6. Diagnose three common DNS failures, including the famous hostname trap
7. Apply a structured troubleshooting process to name resolution

---

## 🧱 Part 1: Build It

### Step 1 — Configure Interfaces

**R1:**
```
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip address 192.168.1.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit
R1(config)# interface Loopback1
R1(config-if)# ip address 203.0.113.10 255.255.255.255
R1(config-if)# exit
```

**R2:**
```
R2(config)# interface GigabitEthernet0/0
R2(config-if)# ip address 192.168.1.2 255.255.255.0
R2(config-if)# no shutdown
R2(config-if)# exit
```

### Step 2 — Turn R1 Into a DNS Server

Build the name-to-address table with `ip host` entries, then enable the router to answer queries from that table.

```
! Build the local name table
R1(config)# ip host r1.lab.local 192.168.1.1
R1(config)# ip host r2.lab.local 192.168.1.2
R1(config)# ip host web.lab.local 203.0.113.10

! Allow R1 to answer DNS queries from other devices using that table
R1(config)# ip dns server
```

> **`ip dns server` is the line that makes R1 a server rather than just a router with a notepad.** Without it, R1 holds the name table for its own use but will not answer anyone else's questions. With it, R1 becomes the resolver that R2 and PC1 can query.

### Step 3 — Configure R2 as a DNS Client

Point R2 at R1 for name resolution, turn lookups on, and set a default domain so short names get completed automatically.

```
! Send all name queries to R1
R2(config)# ip name-server 192.168.1.1

! Enable DNS resolution
R2(config)# ip domain lookup

! Append this domain to bare names, so "web" becomes "web.lab.local"
R2(config)# ip domain name lab.local
```

> **The `ip domain name` line is a convenience that matters.** With it, typing `ping web` quietly becomes `ping web.lab.local`, which is what R1's table actually holds. Without it, you would have to type the full name every time. This is the same reason your work laptop lets you reach an internal server by its short name.

### Step 4 — Hand Out DNS Through DHCP

This is the integration step. Configure R1 to serve DHCP and include itself as the DNS server, so any host on the LAN learns where to resolve names without anyone touching the host.

```
R1(config)# ip dhcp excluded-address 192.168.1.1 192.168.1.10
R1(config)# ip dhcp pool LAN
R1(dhcp-config)# network 192.168.1.0 255.255.255.0
R1(dhcp-config)# default-router 192.168.1.1
R1(dhcp-config)# dns-server 192.168.1.1
R1(dhcp-config)# exit
```

> **This is where the services connect.** Back in the DHCP lab you handed out a DNS server address and may have wondered what the client did with it. Here is the answer. The `dns-server` line tells every DHCP client which resolver to use, so PC1 will learn to resolve names through R1 the moment it gets its lease, with no manual configuration at all. DHCP and DNS were always meant to work together.

### Step 5 — Set PC1 to DHCP

Configure PC1 to obtain its addressing automatically. It will receive an IP address, a gateway, and the DNS server address, all from R1.

### Step 6 — Save the Configurations

```
R1# copy running-config startup-config
R2# copy running-config startup-config
```

---

## 👀 Part 2: Observe It

### 2.1 — Resolve a Name From the Router Client

This is the headline test. From R2, ping a name instead of an address:

```
R2# ping web.lab.local
```

R2 sends the name to R1, R1 answers with 203.0.113.10 from its table, and the ping proceeds to that address. You just reached a host by name.

**✏️ Prediction checkpoint:** Before you run it, predict which address `web.lab.local` will resolve to, and confirm it matches R1's `ip host` entry.

### 2.2 — Watch the Name Get Cached

```
R2# show hosts
```

**What to look for:** The names R2 has resolved, the addresses they map to, and how each entry was learned. A name R2 looked up through R1 appears here in its cache, which is why a second lookup of the same name is instant.

### 2.3 — Confirm the Server Side

On R1, view its own host table:

```
R1# show hosts
```

This shows the static `ip host` entries you configured, the ones R1 hands out in answer to queries. This is R1's authoritative table for the lab.

### 2.4 — Prove the DHCP Integration

Check what PC1 received from DHCP. It should hold an address, a gateway of 192.168.1.1, and a DNS server of 192.168.1.1. If your host image resolves names, try reaching a name from PC1 and watch it work without you ever configuring DNS on the host. That automatic handoff is the whole point of the integration.

**✏️ Think it through:** PC1 was never told about DNS directly. Trace how it ended up knowing which server to ask. The answer runs straight back through the DHCP lease.

### 2.5 — Confirm Resolution, Not Just Connectivity

```
R2# ping r1.lab.local
R2# ping 192.168.1.1
```

Both reach the same place. The first went through name resolution, the second went straight to the address. When both work, name resolution and connectivity are both healthy, and you have a clean baseline before you start breaking things.

---

## 💥 Part 3: Break It

> Work through each scenario one at a time. Document the symptoms fully before you move to Part 4.

---

### Break Scenario 1 — The Client With No Resolver

Remove R2's name server so it has nowhere to send its questions:

```
R2(config)# no ip name-server 192.168.1.1
```

Now test both ways:
```
R2# ping web.lab.local
R2# ping 203.0.113.10
```

**✏️ Document the symptoms:**
- Does the name resolve anymore?
- Does pinging the address directly still work?
- The network is clearly fine, since the address is reachable, so what exactly is broken?

> **Why this matters:** This is the cleanest DNS failure there is, and the most common one in the real world. The name fails because R2 has no server to ask, but the address still works because the underlying network is perfectly healthy. That contrast, name fails while address succeeds, is the signature that tells you the problem is resolution and not connectivity. The moment you see it, you check the client's DNS configuration before anything else.

---

### Break Scenario 2 — The Wrong Answer

Restore R2's name server. Then corrupt the server's table so a name resolves to the wrong address:

```
R2(config)# ip name-server 192.168.1.1
R2(config)# exit

R1(config)# no ip host web.lab.local 203.0.113.10
R1(config)# ip host web.lab.local 203.0.113.99
R1(config)# exit
```

> 203.0.113.99 is not the web server. The name now points somewhere wrong.

Now test from R2:
```
R2# clear host *
R2# ping web.lab.local
```

**✏️ Document the symptoms:**
- Does the name resolve successfully this time?
- Does the ping reach anything?
- Resolution clearly worked, so why did the connection still fail?

> **Why this matters:** DNS can succeed at its job and still send you to the wrong place. The name resolved cleanly, R2 got an answer, and the answer was simply wrong, so the ping went off to a host that does not exist. This is a subtler failure than no resolver at all, because resolution itself looks healthy. When a name resolves but the destination is unreachable or behaves strangely, suspect the record itself. The directory had the wrong number in it. Note the `clear host *` command, which flushes the client cache so it asks again rather than reusing an old good answer.

---

### Break Scenario 3 — The Hostname Trap

This is the famous one, and it is worth experiencing on purpose so you recognize it forever. Restore the correct record first. Then, on R2, with name resolution enabled but pointed at a server that cannot help, mistype a command in privileged exec mode:

```
R1(config)# no ip host web.lab.local 203.0.113.99
R1(config)# ip host web.lab.local 203.0.113.10
R1(config)# exit

! On R2, point resolution at an unreachable server to set up the trap
R2(config)# ip name-server 10.255.255.255
R2(config)# exit

! Now fat-finger a command, the way everyone does eventually
R2# shoq running-config
```

**✏️ Document the symptoms:**
- What does R2 do after you enter the mistyped word? How long does it sit there?
- Why is the router trying to "resolve" your typo at all?
- How does this connect to having `ip domain lookup` enabled?

> **Why this matters:** When you type a word IOS does not recognize as a command, and name resolution is on, the router assumes you must be trying to reach a host by that name and tries to resolve and connect to it. With an unreachable name server, it hangs for many seconds waiting for an answer that never comes. Every Cisco admin has watched a router freeze after a typo and wondered if it crashed. It did not. It is loyally trying to look up your mistake. This single behavior is why experienced engineers harden their devices against it, which is exactly what you will do in the fix.

---

## 🔧 Part 4: Fix It

---

### Fix Scenario 1 — The Client With No Resolver

**Your troubleshooting toolkit:**
```
show running-config | include name-server
ping <name>      then     ping <address>
```

**Structured process:**
1. Note the signature. The name fails but the address succeeds.
2. That points at resolution, so check the client's DNS configuration.
3. If there is no name server configured, the client has nowhere to send queries.
4. Add the name server and confirm names resolve again.

**✅ Solution:**
```
R2(config)# ip name-server 192.168.1.1
```

**Verification:**
```
R2# ping web.lab.local
```

---

### Fix Scenario 2 — The Wrong Answer

**Your troubleshooting toolkit:**
```
show hosts          ! on the server, inspect the records it hands out
clear host *        ! on the client, flush a cached bad answer
```

**Structured process:**
1. Note that the name resolves but the destination is wrong or unreachable.
2. That points at the record itself, not the client.
3. On the server, compare the `ip host` entry against the real address.
4. Correct the record, flush the client cache, and verify.

**✅ Solution:**
```
R1(config)# no ip host web.lab.local 203.0.113.99
R1(config)# ip host web.lab.local 203.0.113.10
```

**Verification:**
```
R2# clear host *
R2# ping web.lab.local
```

---

### Fix Scenario 3 — The Hostname Trap

There are two ways to fix this, and the difference is worth understanding.

**Your troubleshooting toolkit:**
```
show running-config | include domain lookup
```

**Structured process:**
1. Recognize that mistyped commands are being treated as hostnames to resolve.
2. Decide whether this device needs DNS resolution at all.
3. Apply the appropriate hardening.
4. Confirm a typo now fails instantly instead of hanging.

**✅ Solution, the surgical fix.** This keeps DNS resolution working but stops the router from trying to telnet to your typos:
```
R2(config)# line console 0
R2(config-line)# transport preferred none
R2(config-line)# exit
```

**✅ Solution, the blunt fix.** On a device that has no need to resolve names at all, simply turn resolution off:
```
R2(config)# no ip domain lookup
```

> **Which to use:** On lab routers and switches that never need DNS, `no ip domain lookup` is the common and perfectly reasonable choice. On a device that genuinely uses DNS, `transport preferred none` is the elegant fix, because it keeps resolution working while refusing to treat your command-line typos as connection attempts. Many engineers apply one of these to every device they touch.

**Verification:**
```
R2# shoq running-config
```
The typo should now be rejected immediately, with no hang.

---

## 💬 Reflection Questions

1. Explain the role of DNS in your own words. Why do networks need it, and what would using the internet be like without it?

2. Walk through what happens when R2 runs `ping web.lab.local`, from the moment you press enter to the moment the ping reaches its destination. Name each step of the resolution.

3. In Break Scenario 1, the name failed but the address worked. Explain why that specific combination of symptoms points to a DNS problem rather than a connectivity problem, and why it is such a useful diagnostic.

4. In Break Scenario 2, resolution succeeded but the connection still failed. How can DNS do its job correctly and still leave you unable to reach the destination? What does this teach you about trusting that a name resolved?

5. PC1 learned its DNS server through DHCP, while R2 was configured by hand. Describe the path by which PC1 ended up knowing which server to query, and explain why handing out DNS through DHCP is the standard approach for real clients.

6. The hostname trap happens because the router treats an unknown command as a name to resolve. Explain why `transport preferred none` is often a better fix than `no ip domain lookup` on a device that actually uses DNS.

---

## Challenge Extension (Optional)

1. **Add local host entries on the client.** Configure `ip host` entries directly on R2 so it can resolve a few names without asking R1 at all. Then check `show hosts` and explain the difference between a locally defined entry and one learned from the server.

2. **Resolve a real domain.** Point R1 or R2 at a public DNS server such as 8.8.8.8 with `ip name-server`, give the device a route to the internet, and try resolving a real domain name. Watch a query leave your network for a real resolver.

3. **Break it at the DHCP layer.** Remove the `dns-server` line from R1's DHCP pool, renew PC1, and observe what the host can and cannot do. This shows how a DNS problem can originate one layer away, in the DHCP configuration.

4. **Trace the cache behavior.** Resolve a name, then use `show hosts` to watch it cached, and `clear host *` to flush it. Explain why caching matters for performance and what the tradeoff is when a record changes.

---

## Instructor Notes

**Common Student Mistakes**
- Building the `ip host` table on R1 but forgetting `ip dns server`, so the router holds the records yet never answers a single query from R2 or PC1. The table looks complete, and nothing resolves.
- Typing a short name like `ping web` without configuring `ip domain name lab.local`, so the bare name never matches the full `web.lab.local` entry. Students assume the record is wrong when the domain suffix is the real issue.
- Treating a resolution failure as a connectivity failure. Students see a name fail and start checking cables and routing, when a quick `ping` to the address would have told them the network is fine and the problem is DNS.
- Changing a record on the server but forgetting `clear host *` on the client, then chasing a stale cached answer that is no longer correct.
- Reaching for `no ip domain lookup` to fix the hostname trap when `transport preferred none` was the better tool, or not understanding why the router hung on a typo in the first place.

**Teaching Tips**
- The single most valuable habit to instill is testing both ways, the name and the address, on every failure. Have students say out loud which one is broken before they touch anything. Once "name fails, address works equals DNS problem" becomes automatic, they troubleshoot resolution quickly for the rest of their careers.
- Demonstrate the hostname trap live to the whole class. Mistype a command on a router with an unreachable name server and let everyone watch it hang. It is memorable, it gets a laugh, and it gives students a reason they will never forget for hardening their devices.

---

## Where This Leads

You have now built the last of the core network services, and with it the picture comes together. DHCP hands a client its address and tells it which resolver to use, NAT lets that privately addressed client reach the public internet, and DNS turns the names people actually type into the addresses the network needs. Those three services, working as a system, are what stand between a pile of configured devices and a network that feels effortless to the people using it.

Two services remain in this section, and they shift the focus from helping users to helping you, the operator. NTP keeps every device's clock synchronized, which sounds minor until you try to correlate logs across devices whose times disagree. Syslog and SNMP then give you the visibility to know what your network is doing and to be told the moment something goes wrong. With the user-facing services behind you, the operator-facing ones are next, and NTP is where the section goes from here.

---

*CCNA – Build It · Observe It · Break It · Fix It series*

*Next: NTP*

*This repository is an independent educational resource and is not officially affiliated with Cisco Systems. Cisco, CCNA, Cisco Modeling Labs, and Cisco Networking Academy are trademarks of Cisco Systems, Inc.*
