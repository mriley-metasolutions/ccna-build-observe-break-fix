# DHCP
 
**Build It · Observe It · Break It · Fix It**
 
---
 
## Overview
 
Through the entire switching section you typed an IP address, a subnet mask, and a gateway onto every single host by hand. That works for a lab with two PCs. It does not work for a building with two hundred of them, and it certainly does not work for a coffee shop where a different set of phones and laptops walks in every hour. DHCP is the service that solves this. It hands out IP addresses, masks, gateways, and DNS servers automatically, on demand, and then takes them back when a device leaves.
 
The Dynamic Host Configuration Protocol works through a short four-step conversation that is worth knowing by name, because the exam asks about it and the steps map directly to what you will troubleshoot. A client that needs an address **discovers** by broadcasting a request, the server **offers** an address, the client formally **requests** that specific offer, and the server **acknowledges**. Discover, Offer, Request, Acknowledge, remembered as **DORA**.
 
There is one wrinkle that drives half of all real DHCP problems, and this lab is built around it. The initial Discover is a broadcast, and routers do not forward broadcasts. So a DHCP server happily serves any client on its own local subnet, but a client one router away never even reaches it, unless you configure a **relay agent** to carry that conversation across the router. In this lab you will set up one router as a DHCP server feeding its own local LAN directly, configure a second router as a relay so a remote subnet can be served by that same server, and then break both paths in the ways they most often fail.
 
**CCNA Exam Alignment:**
- 4.3 – Explain the role of DHCP and DNS within the network
- 4.6 – Configure and verify DHCP client and relay

**Estimated Time:** 75–105 minutes
 
---
 
## Key Concept: DORA, and Why Routers Get in the Way
 
The DORA exchange is simple, but the detail that matters for troubleshooting is which messages are broadcasts.
 
| Step | Message | Sent by | How |
|------|---------|---------|-----|
| **D** | Discover | Client | Broadcast. "Is there a DHCP server out there?" |
| **O** | Offer | Server | The server proposes an address. |
| **R** | Request | Client | Broadcast. "I accept that offer." |
| **A** | Acknowledge | Server | The lease is confirmed and handed over. |
 
The two broadcasts are the catch. A broadcast stays inside its own subnet, because a router will not forward it. That means a DHCP server on a different subnet from the client never hears the Discover at all, no matter how healthy the network is otherwise.
 
The fix is a **relay agent**, configured with the `ip helper-address` command on the router interface facing the clients. The relay catches the client's broadcast, converts it into a unicast aimed straight at the DHCP server, and stamps it with the client's subnet so the server knows which pool to draw from. The server's reply travels back through the relay to the waiting client. One server can feed many subnets this way, which is exactly how real networks centralize DHCP.
 
> **Get ahead of the confusion:** When you reach the Break It section and the remote subnet stops getting addresses while the local subnet is fine, do not go hunting through the server's pools. The split between "local works, remote fails" is the unmistakable signature of a relay problem, and `ip helper-address` is the first place to look. I have watched this exact symptom send people on long detours through perfectly healthy server configuration.
 
---
 
## Topology
 
```
   [PC1]                                            [PC2]
 DHCP client                                     DHCP client
 VLAN/subnet 192.168.10.0/24                  subnet 192.168.20.0/24
     |                                              |
    G0/0                                          G0/0
   [R1] ------------- 10.0.12.0/30 ------------- [R2]
  G0/0: 192.168.10.1   G0/1          G0/1   G0/0: 192.168.20.1
  DHCP SERVER          10.0.12.1   10.0.12.2  DHCP RELAY AGENT
  (serves both pools)                          (ip helper-address)
```

**Nodes (4 total, within CML Free Edition limit):**
- R1 (IOSv router, the DHCP server)
- R2 (IOSv router, the DHCP relay agent)
- PC1 (DHCP client on R1's local LAN)
- PC2 (DHCP client on R2's remote LAN)
> **The design in one sentence:** R1 serves PC1 directly because they share a subnet, and R1 serves PC2 through R2's relay because they do not. Same server, two delivery paths, and that is the whole lesson.
 
> **A note on interface names and host DHCP:** Port names vary by image, so confirm with `show ip interface brief` and adjust. The method for putting a host into DHCP mode also varies by node type in CML, so set each PC's interface to obtain its address automatically using whatever mechanism your host image provides.
 
---
 
## Addressing Table
 
| Device | Interface | IP Address     | Subnet Mask       | Role |
|--------|-----------|----------------|-------------------|------|
| R1     | G0/0      | 192.168.10.1   | 255.255.255.0     | Gateway + server for LAN 10 |
| R1     | G0/1      | 10.0.12.1      | 255.255.255.252   | Link to R2 |
| R2     | G0/1      | 10.0.12.2      | 255.255.255.252   | Link to R1 |
| R2     | G0/0      | 192.168.20.1   | 255.255.255.0     | Gateway + relay for LAN 20 |
| PC1    | NIC       | DHCP           | DHCP              | Client on LAN 10 |
| PC2    | NIC       | DHCP           | DHCP              | Client on LAN 20 |
 
> **The clients have no static addresses, on purpose.** That is the entire point of the lab. PC1 and PC2 will receive everything they need from R1, one directly and one through a relay.
 
---
 
## Lab Objectives
 
By the end of this lab you will be able to:
 
1. Explain the role of DHCP and the DORA exchange
2. Configure a Cisco router as a DHCP server with address pools
3. Reserve addresses with `ip dhcp excluded-address`
4. Configure a DHCP relay agent with `ip helper-address`
5. Verify leases with `show ip dhcp binding` and related commands
6. Identify and resolve three common DHCP failures
7. Apply a structured troubleshooting process to dynamic addressing
---
 
## 🧱 Part 1: Build It
 
### Step 1 — Configure Interfaces and Routing
 
Bring up the interfaces and add static routes so R1 can reach the remote LAN and the two subnets can communicate. The relay depends on R1 being able to route back to the remote subnet, so this step is not optional.
 
**R1:**
```
R1> enable
R1# configure terminal
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip address 192.168.10.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit
R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip address 10.0.12.1 255.255.255.252
R1(config-if)# no shutdown
R1(config-if)# exit
 
! R1 must be able to reach the remote LAN to serve it through the relay
R1(config)# ip route 192.168.20.0 255.255.255.0 10.0.12.2
```
 
**R2:**
```
R2(config)# interface GigabitEthernet0/1
R2(config-if)# ip address 10.0.12.2 255.255.255.252
R2(config-if)# no shutdown
R2(config-if)# exit
R2(config)# interface GigabitEthernet0/0
R2(config-if)# ip address 192.168.20.1 255.255.255.0
R2(config-if)# no shutdown
R2(config-if)# exit
 
! Route back to R1's LAN so the two subnets can communicate
R2(config)# ip route 192.168.10.0 255.255.255.0 10.0.12.1
```
 
### Step 2 — Reserve Addresses That DHCP Must Not Hand Out
 
Before defining the pools, tell DHCP which addresses are off limits. The gateway addresses and a small block for future static assignments should never be leased to a client.
 
```
! Protect the gateways and a range for static devices on each LAN
R1(config)# ip dhcp excluded-address 192.168.10.1 192.168.10.10
R1(config)# ip dhcp excluded-address 192.168.20.1 192.168.20.10
```
 
> **Do this first, and do not skip it.** A DHCP pool will otherwise hand out every address in the subnet, including the router's own gateway IP. That produces a duplicate address and a very confusing outage. You will create exactly that failure on purpose in Break Scenario 3, so the reason for this step will stick.
 
### Step 3 — Create the DHCP Pools on R1
 
R1 serves both subnets, so it needs two pools. Each pool defines the network, the gateway clients should use, a DNS server, and a lease time.
 
```
! Pool for R1's own local LAN
R1(config)# ip dhcp pool LAN10
R1(dhcp-config)# network 192.168.10.0 255.255.255.0
R1(dhcp-config)# default-router 192.168.10.1
R1(dhcp-config)# dns-server 8.8.8.8
R1(dhcp-config)# lease 0 12 0
R1(dhcp-config)# exit
 
! Pool for the remote LAN, served through R2's relay
R1(config)# ip dhcp pool LAN20
R1(dhcp-config)# network 192.168.20.0 255.255.255.0
R1(dhcp-config)# default-router 192.168.20.1
R1(dhcp-config)# dns-server 8.8.8.8
R1(dhcp-config)# lease 0 12 0
R1(dhcp-config)# exit
```
 
> **The `default-router` line is the one people underestimate.** A client that receives an address but the wrong gateway can talk to its own subnet and nothing else. The lease looks successful, the address looks fine, and the user still cannot reach anything. You will see precisely that in Break Scenario 2. The `lease 0 12 0` here sets a twelve-hour lease, written as days, hours, and minutes.
 
### Step 4 — Configure the Relay Agent on R2
 
R2 sits between PC2 and the DHCP server. The `ip helper-address` command on R2's client-facing interface turns it into a relay that carries DHCP across the router boundary.
 
```
R2(config)# interface GigabitEthernet0/0
R2(config-if)# ip helper-address 10.0.12.1
R2(config-if)# exit
```
 
> **The helper address points at the server, not the client.** It tells R2, "when you hear a DHCP broadcast on this interface, forward it as a unicast to the server at 10.0.12.1." R2 also stamps the request with its own interface address, 192.168.20.1, which is how R1 knows to draw from the LAN20 pool rather than LAN10.
 
### Step 5 — Set the Hosts to DHCP
 
Configure both PC1 and PC2 to obtain their addressing automatically rather than statically. The exact method depends on your host image in CML.
 
### Step 6 — Save Both Router Configurations
 
```
R1# copy running-config startup-config
R2# copy running-config startup-config
```
 
---
 
## 👀 Part 2: Observe It
 
### 2.1 — Confirm the Clients Received Addresses
 
On each PC, check the assigned address. PC1 should hold an address from the 192.168.10.0/24 range, and PC2 from 192.168.20.0/24, both above the excluded block, so .11 or higher.
 
**✏️ Prediction checkpoint:** Before you look, predict the first address each pool will hand out. Remember that .1 through .10 are excluded on both subnets.
 
### 2.2 — View the Lease Bindings on the Server
 
This is the server's record of every address it has handed out, and the first command to reach for.
 
```
R1# show ip dhcp binding
```
 
**What to look for:** One binding per client, showing the leased IP address, the client's hardware address, and the lease expiration. You should see both PC1 and PC2 here, which proves the local path and the relay path are both working.
 
### 2.3 — Check the Pool Statistics
 
```
R1# show ip dhcp pool
```
 
This shows each pool, how many addresses are in it, and how many are currently leased. It is the fastest way to spot a pool that is filling up or one that nobody is drawing from.
 
### 2.4 — Watch the Conversation, Optionally
 
If you want to see DORA happen live, enable debugging on R1, then release and renew an address on a client:
 
```
R1# debug ip dhcp server events
```
 
Watch the request arrive and the assignment go out, then turn debugging off with `undebug all`. This is the clearest way to see the relay in action, because the request from PC2 arrives stamped with R2's relay address.
 
### 2.5 — Test End-to-End Connectivity
 
From PC1, confirm it can reach its gateway, the remote gateway, and the remote host:
```
PC1> ping 192.168.10.1
PC1> ping 192.168.20.1
PC1> ping 192.168.20.<PC2 address>
```
 
If all three succeed, both clients have complete, correct addressing and the routing between them works.
 
---
 
## 💥 Part 3: Break It
 
> Work through each scenario one at a time. Document the symptoms fully before you move to Part 4.
 
---
 
### Break Scenario 1 — The Missing Relay
 
This is the defining DHCP-across-a-router failure. Remove the relay configuration from R2:
 
```
R2(config)# interface GigabitEthernet0/0
R2(config-if)# no ip helper-address 10.0.12.1
R2(config-if)# exit
```
 
Now release and renew PC2's address, and check PC1 as a control:
```
(on PC2: release and renew the DHCP lease)
(on PC1: confirm it still has an address)
R1# show ip dhcp binding
```
 
**✏️ Document the symptoms:**
- Can PC2 obtain an address anymore?
- Is PC1 affected at all?
- Why does the local client keep working perfectly while the remote client fails completely?
> **Why this matters:** Without the relay, PC2's Discover broadcast dies at R2 and never reaches the server, while PC1's broadcast is heard directly by R1 on the same subnet. The split is total and it is diagnostic. Local works, remote fails, points straight at the relay every time. This single pattern explains a huge share of real DHCP tickets, and now you can recognize it on sight.
 
---
 
### Break Scenario 2 — The Wrong Gateway
 
Restore the relay. Then misconfigure the gateway handed out by the LAN20 pool so it points to an address that is not the real gateway:
 
```
R2(config)# interface GigabitEthernet0/0
R2(config-if)# ip helper-address 10.0.12.1
R2(config-if)# exit
 
R1(config)# ip dhcp pool LAN20
R1(dhcp-config)# default-router 192.168.20.99
R1(dhcp-config)# exit
```
 
> 192.168.20.99 is not R2's interface, so it is not a usable gateway.
 
Release and renew PC2, then test:
```
(on PC2: release and renew the DHCP lease)
PC2> ping 192.168.20.1
PC2> ping 192.168.10.1
```
 
**✏️ Document the symptoms:**
- Did PC2 still receive an IP address successfully?
- Can PC2 reach hosts on its own subnet? Can it reach anything beyond it?
- The lease succeeded, so why is PC2 still cut off from the rest of the network?
> **Why this matters:** A DHCP lease is more than an address. The gateway is part of the package, and a client with a wrong gateway is the very picture of "I have an IP but no internet." Same-subnet traffic works because it never needs a gateway, while everything beyond the subnet fails because the client is sending it to an address that cannot route. The lease looking successful is exactly what makes this one slippery, and `show ip dhcp binding` will not reveal it. You have to check what the pool actually hands out.
 
---
 
### Break Scenario 3 — The Forgotten Exclusion
 
Restore the correct gateway. Then remove the exclusion on the local LAN so the pool is free to hand out the gateway's own address:
 
```
R1(config)# ip dhcp pool LAN20
R1(dhcp-config)# default-router 192.168.20.1
R1(dhcp-config)# exit
 
R1(config)# no ip dhcp excluded-address 192.168.10.1 192.168.10.10
```
 
Now clear the existing bindings and have PC1 request a fresh address. With the exclusion gone, the pool may now offer an address inside the .1 to .10 range, including the gateway:
 
```
R1# clear ip dhcp binding *
(on PC1: release and renew the DHCP lease)
R1# show ip dhcp binding
```
 
**✏️ Document the symptoms:**
- What address range can PC1 now be assigned from that it could not before?
- If a client receives 192.168.10.1, what conflict does that create with R1 itself?
- Why is excluding the gateway and static range a required step rather than an optional nicety?
> **Why this matters:** A DHCP pool will hand out every address in its network statement unless you tell it not to. Forget the exclusion and the pool can lease the gateway address, a server address, or anything else you assigned statically, producing a duplicate-address conflict that is maddening to track down because it appears intermittently as clients come and go. Excluding the gateway and a static block up front prevents the whole class of problem.
 
---
 
## 🔧 Part 4: Fix It
 
---
 
### Fix Scenario 1 — The Missing Relay
 
**Your troubleshooting toolkit:**
```
show ip dhcp binding                       ! local client present, remote absent
show running-config interface GigabitEthernet0/0   ! on the relay router
```
 
**Structured process:**
1. Note the signature. The local subnet gets addresses, the remote subnet does not.
2. That split points at the router between the remote clients and the server.
3. Check the client-facing interface on that router for `ip helper-address`.
4. Add the helper address pointing at the server, then renew the client.
**✅ Solution:**
```
R2(config)# interface GigabitEthernet0/0
R2(config-if)# ip helper-address 10.0.12.1
R2(config-if)# exit
```
 
**Verification:**
```
R1# show ip dhcp binding
(on PC2: release and renew, confirm an address arrives)
```
 
---
 
### Fix Scenario 2 — The Wrong Gateway
 
**Your troubleshooting toolkit:**
```
show running-config | section dhcp     ! inspect what each pool hands out
```
 
**Structured process:**
1. Note that the client has an address but cannot reach beyond its own subnet.
2. That points at the gateway, not at the address itself.
3. Compare the pool's `default-router` against the real gateway interface.
4. Correct the gateway, then renew the client so it picks up the fix.
**✅ Solution:**
```
R1(config)# ip dhcp pool LAN20
R1(dhcp-config)# default-router 192.168.20.1
R1(dhcp-config)# exit
```
 
**Verification:**
```
(on PC2: release and renew the DHCP lease)
PC2> ping 192.168.10.1
```
 
---
 
### Fix Scenario 3 — The Forgotten Exclusion
 
**Your troubleshooting toolkit:**
```
show ip dhcp binding                   ! look for a leased gateway or static address
show running-config | include excluded
```
 
**Structured process:**
1. Note the duplicate-address symptoms and check the bindings for an address that should never have been leased.
2. Confirm the exclusion for the gateway and static range is missing.
3. Restore the exclusion.
4. Clear the bad bindings so clients request again from the safe range.
**✅ Solution:**
```
R1(config)# ip dhcp excluded-address 192.168.10.1 192.168.10.10
R1(config)# exit
R1# clear ip dhcp binding *
```
 
**Verification:**
```
(on PC1: release and renew)
R1# show ip dhcp binding
```
 
---
 
## 💬 Reflection Questions
 
1. Lay out the DORA exchange in your own words. Which two of the four messages are broadcasts, and why does that detail matter so much for networks with more than one subnet?
2. A DHCP server serves clients on its own subnet without any special configuration, but needs help to serve clients a router away. Explain why, and describe what the `ip helper-address` relay actually does to bridge that gap.
3. In Break Scenario 2, the client received a valid IP address but still could not reach other networks. Explain why a successful lease is not the same as working connectivity, and what part of the lease was actually broken.
4. The relay agent stamps each forwarded request with its own interface address. Why does the DHCP server need that information, and what would go wrong if the relay did not provide it?
5. In Break Scenario 3, forgetting the exclusion let the pool hand out the gateway's own address. Why does a DHCP pool lease every address in its network statement by default, and why is reserving the gateway and static range a required habit rather than an optional one?
6. DHCP leases expire after a set time rather than lasting forever. What is the benefit of a finite lease, and what tradeoff are you making when you choose a very short lease versus a very long one?
---
 
## Challenge Extension (Optional)
 
1. **Reserve an address for a specific device.** Configure a manual DHCP binding so a particular client always receives the same address based on its hardware address. Where in a real network would you want a printer or a server to get a predictable DHCP address rather than a static one?
2. **Shorten the lease and watch renewal.** Set a very short lease on one pool, then observe a client renewing partway through its lease lifetime. Research what the T1 and T2 renewal timers do.
3. **Serve a third subnet.** If you can add a node, give R2 a second LAN and a second helper address, and create a third pool on R1. Confirm one central server can feed three subnets through relays.
4. **Make the router a client.** Configure one of R2's interfaces to obtain its address via DHCP with `ip address dhcp`, which is exactly the client side of objective 4.6. Where would a router acting as a DHCP client actually happen in the real world?
---
 
## Where This Leads
 
You have automated the one task you spent the entire switching section doing by hand. Every host on these two subnets now configures itself, the addressing stays consistent, and one central server feeds a subnet a router away through a relay you built yourself. That relay concept, a router carrying a service across a boundary it would otherwise block, is a pattern you will meet again.
 
The next service tackles a different boundary. Your clients now have private addresses in the 192.168 range, and those addresses cannot travel across the public internet. NAT is the service that translates private addresses into public ones and back, and it is what lets a whole network of privately addressed hosts share a handful of public addresses. With DHCP handing out those private addresses automatically, NAT is the natural next step, and it is where the Services section goes next.
 
---
 
*CCNA – Build It · Observe It · Break It · Fix It series*

*Next: NAT*
 
*This repository is an independent educational resource and is not officially affiliated with Cisco Systems. Cisco, CCNA, Cisco Modeling Labs, and Cisco Networking Academy are trademarks of Cisco Systems, Inc.*
