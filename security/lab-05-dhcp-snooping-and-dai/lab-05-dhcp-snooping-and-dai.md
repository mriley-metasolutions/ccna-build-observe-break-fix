# DHCP Snooping and Dynamic ARP Inspection

**Build It · Observe It · Break It · Fix It**

---

## Overview

The ACL labs gave you precise control over traffic, but they share a blind spot. An ACL trusts the addresses written in a packet, and at the access layer those addresses can be forged. Two classic attacks exploit exactly this trust, and no ACL will stop either one, because the deception happens beneath the layer an ACL inspects.

The first is the **rogue DHCP server**. An attacker plugs in a device running its own DHCP server and starts answering client requests. If the attacker's offer reaches a client first, that client accepts a poisoned configuration, most dangerously a default gateway pointing at the attacker's machine. Now all of the victim's traffic to the outside world flows through the attacker, who can read or alter it. The user notices nothing, because they got an address and the network appears to work.

The second is **ARP spoofing**, also called ARP poisoning. ARP has no built-in verification, so an attacker can send forged ARP replies claiming that the gateway's IP address belongs to the attacker's MAC address. Every device that believes the lie starts sending its gateway-bound traffic to the attacker instead. Again it is a man-in-the-middle attack, and again it is silent.

Two Layer 2 features defend against these, and they work as a team. **DHCP Snooping** watches DHCP traffic and only allows server responses from ports you have explicitly trusted, which shuts down rogue servers. As it works, it builds a **binding table**, a trusted record of which IP address legitimately belongs to which MAC address on which port. **Dynamic ARP Inspection**, or DAI, then uses that binding table to check every ARP message and drop any that does not match a real binding, which shuts down ARP spoofing. DAI cannot work without the table that snooping builds, which is why you configure them together and in that order.

**CCNA Exam Alignment:**
- 5.7 – Configure Layer 2 security features, including DHCP snooping and dynamic ARP inspection

**Estimated Time:** 90–120 minutes

---

## Prerequisites

Before completing this lab, you should be comfortable with:
- The DHCP lab from the Services section, since both features build on how DHCP and the DORA exchange work
- The Port Security lab, especially the idea of treating some ports differently from others
- VLANs and access ports from the Switching section
- The spoofing and man-in-the-middle threats from the Security Fundamentals concept guide
- CML fundamentals, including building or importing a topology and accessing a device console

---

## Key Concept: The Trust Boundary and the Binding Table

Almost everything in this lab comes down to two ideas. Get these and the commands follow naturally.

The first idea is the **trust boundary**. Both features divide every port into trusted or untrusted, and you decide which is which based on what lives on the other end. A port is **trusted** if it faces legitimate infrastructure, the real DHCP server or an uplink toward it. A port is **untrusted** if it faces users and their devices, where an attacker might plug in. On untrusted ports, the features enforce their rules. On trusted ports, they step aside.

For DHCP snooping, the rule on an untrusted port is simple: client messages like Discover and Request are fine, but **server messages like Offer and Acknowledge are dropped**. A real DHCP server never lives on a user port, so a DHCP offer arriving on an untrusted port can only be a rogue, and it is silently discarded. The legitimate server, sitting on a trusted port, answers without interference.

The second idea is the **binding table**. As snooping watches legitimate clients receive their addresses, it records each one as a binding: this IP address, with this MAC address, on this port, in this VLAN, for this lease. That table is a trusted map of who is really who at the access layer, and it is the foundation that DAI stands on. When DAI inspects an ARP message on an untrusted port, it asks one question: does the IP-to-MAC claim in this ARP match a real binding in the table? If yes, the ARP passes. If no, it is dropped as a forgery.

> **Get ahead of the confusion:** DAI is only as good as the binding table beneath it, and the binding table comes from DHCP. This has a real consequence you will hit in this lab. A device with a static IP address never did a DHCP exchange, so it has no binding, and DAI will drop its ARP as if it were an attack. That is not a bug, it is the design, and the solution for legitimate static hosts is an ARP ACL that vouches for them. Keep this dependency in mind, because it explains the trickiest scenario ahead.

---

## Topology

```
                  [R1]  legitimate DHCP server + gateway
                192.168.10.1
                    |  TRUSTED uplink (G0/1)
                  [SW1]  DHCP Snooping + DAI on VLAN 10
                 /            \
            untrusted          untrusted
              G0/2              G0/3
             [PC1]              [R2]
        legitimate client    attacker
           (DHCP)            (rogue DHCP server,
                              spoofing device)
```

> **Full topology file:** `topology.yaml`

**Nodes (4 total, within CML Free Edition limit):**
- SW1 (IOSvL2 switch, where snooping and DAI are configured)
- R1 (IOSv router, the legitimate DHCP server and gateway, on a trusted port)
- PC1 (the legitimate client, on an untrusted port)
- R2 (the attacker, on an untrusted port, used to play the rogue DHCP server and the spoofing device)

> **A scope note worth reading:** Snooping's defense against a rogue DHCP server is fully demonstrable here, since R2 can run a real DHCP server. Fully simulating a live ARP-poisoning flood needs a dedicated attack tool beyond CML's basic nodes, so DAI is demonstrated through its binding-table validation and its very real effect on a host that has no binding. That validation is the exact mechanism that stops ARP spoofing, so you see the defense at work even without the attack traffic. Confirm interface names with `show ip interface brief` and adjust.

---

## Addressing Table

| Device | Interface | IP Address     | Subnet Mask     | Role |
|--------|-----------|----------------|-----------------|------|
| R1     | G0/0      | 192.168.10.1   | 255.255.255.0   | Legitimate DHCP server and gateway |
| PC1    | NIC       | DHCP           | DHCP            | Legitimate client |
| R2     | G0/0      | varies         | 255.255.255.0   | Attacker (rogue server, static host) |
| SW1    | VLAN 10   | (optional)     | 255.255.255.0   | Switch enforcing snooping and DAI |

---

## Lab Objectives

By the end of this lab you will be able to:

1. Explain the rogue DHCP and ARP spoofing attacks and why ACLs cannot stop them
2. Configure DHCP snooping and designate trusted and untrusted ports
3. Read and interpret the DHCP snooping binding table
4. Configure Dynamic ARP Inspection and explain its dependence on the binding table
5. Verify both features with the appropriate `show` commands
6. Account for statically addressed hosts using an ARP ACL
7. Troubleshoot trusted-port, enablement, and binding-dependency failures

---

## 🧱 Part 1: Build It

### Step 1 — Set Up the Legitimate DHCP Service and VLAN

Configure R1 as the legitimate DHCP server and gateway for VLAN 10, the way you did in the DHCP lab. On SW1, create VLAN 10 and place the access ports in it.

**On R1:**
```
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip address 192.168.10.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit
R1(config)# ip dhcp excluded-address 192.168.10.1 192.168.10.10
R1(config)# ip dhcp pool LAN10
R1(dhcp-config)# network 192.168.10.0 255.255.255.0
R1(dhcp-config)# default-router 192.168.10.1
R1(dhcp-config)# exit
```

**On SW1:**
```
SW1(config)# vlan 10
SW1(config-vlan)# name Users
SW1(config-vlan)# exit
SW1(config)# interface range GigabitEthernet0/2 - 3
SW1(config-if-range)# switchport mode access
SW1(config-if-range)# switchport access vlan 10
SW1(config-if-range)# exit
SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# exit
```

Confirm that PC1 can get an address from R1 before adding any protection, so you have a working baseline.

### Step 2 — Enable DHCP Snooping

DHCP snooping takes two commands to turn on, a global enable and a per-VLAN enable, and it needs both. Then you trust the one port that faces the real server.

```
! Global enable
SW1(config)# ip dhcp snooping

! Enable for the VLAN you want protected
SW1(config)# ip dhcp snooping vlan 10

! Trust the uplink toward the legitimate DHCP server
SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# ip dhcp snooping trust
SW1(config-if)# exit
```

> **Both enable commands are required, and the trust is the crucial piece.** The global `ip dhcp snooping` and the per-VLAN `ip dhcp snooping vlan 10` must both be present, or snooping silently does nothing, which you will prove in a break scenario. Every other port is untrusted by default, which is what you want for the user ports G0/2 and G0/3. Trusting G0/1 is what lets the real server's offers through while rogue offers on user ports are dropped.

### Step 3 — Let the Client Build a Binding

With snooping active, have PC1 obtain or renew its address from R1. As it does, snooping records the binding.

```
(on PC1: release and renew its DHCP lease)
```

PC1 now has a legitimate address, and the switch has a binding that records it. That binding is what DAI will rely on next.

### Step 4 — Enable Dynamic ARP Inspection

DAI mirrors snooping's structure. Enable it for the VLAN, and trust the same uplink so the server side is not inspected.

```
! Inspect ARP on this VLAN
SW1(config)# ip arp inspection vlan 10

! Trust the same uplink toward the infrastructure
SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# ip arp inspection trust
SW1(config-if)# exit
```

> **DAI leans entirely on the binding table snooping built.** On the untrusted user ports, every ARP message is now checked against that table, and only ARP that matches a real binding is allowed. This is why the order matters. Snooping had to be working and PC1 had to get its binding before DAI could validate anything.

### Step 5 — Save the Configuration

```
SW1# copy running-config startup-config
```

---

## 👀 Part 2: Observe It

### 2.1 — Verify DHCP Snooping and the Trust Boundary

```
SW1# show ip dhcp snooping
```

**What to look for:** Snooping enabled, VLAN 10 listed as protected, and the interface list showing G0/1 as trusted while the user ports are untrusted. This is your trust boundary made visible.

### 2.2 — Read the Binding Table

This is the heart of both features.

```
SW1# show ip dhcp snooping binding
```

**What to look for:** An entry for PC1 showing its MAC address, its leased IP address, the VLAN, the lease time, and the port it lives on. That single row is the trusted fact that DAI will use to judge every ARP claim about PC1.

**✏️ Prediction checkpoint:** Before you look, predict whether PC1's binding will be present and what information it will contain.

### 2.3 — Confirm DAI Is Inspecting

```
SW1# show ip arp inspection
SW1# show ip arp inspection statistics
```

**What to look for:** DAI enabled on VLAN 10, the trusted and untrusted ports listed, and statistics counters for forwarded and dropped ARP packets. As legitimate traffic flows, the forwarded counter climbs.

### 2.4 — Demonstrate the Rogue Server Being Blocked

Now play the attacker. Configure R2 as a rogue DHCP server handing out a poisoned gateway, and connect it to the untrusted port G0/3:

```
R2(config)# interface GigabitEthernet0/0
R2(config-if)# ip address 192.168.10.66 255.255.255.0
R2(config-if)# no shutdown
R2(config-if)# exit
R2(config)# ip dhcp pool ROGUE
R2(dhcp-config)# network 192.168.10.0 255.255.255.0
R2(dhcp-config)# default-router 192.168.10.66
R2(dhcp-config)# exit
```

Have PC1 release and renew several times, then check which gateway it received and look at the snooping drops:

```
(on PC1: release and renew, check the assigned gateway)
SW1# show ip dhcp snooping
```

**✏️ Document what you see:**
- Does PC1 ever receive the rogue gateway 192.168.10.66, or does it keep getting the legitimate 192.168.10.1?
- The rogue server is running and answering, so why do its offers never reach PC1?
- What would have happened to PC1 without snooping enabled?

> **Why this matters:** R2 is a fully functional DHCP server, eagerly offering a poisoned gateway that would route every one of PC1's packets through the attacker. But R2 sits on an untrusted port, so the switch drops its server messages before they ever reach a client. PC1 keeps getting the legitimate configuration from R1, oblivious to the attack that just failed. This is snooping doing precisely the job it exists for, and the entire defense rested on one decision, which ports you trusted.

---

## 💥 Part 3: Break It

> Work through each scenario one at a time. Document the symptoms fully before you move to Part 4.

---

### Break Scenario 1 — The Untrusted Server

The most common snooping mistake is forgetting to trust the port facing the real server. Remove the trust from the uplink to R1:

```
SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# no ip dhcp snooping trust
SW1(config-if)# exit
```

Now have PC1 try to get an address:
```
(on PC1: release and renew the DHCP lease)
```

**✏️ Document the symptoms:**
- Can PC1 obtain an address from R1 anymore?
- R1 is the legitimate server and is working fine, so why are its offers now being dropped?
- What does this tell you about how snooping treats an untrusted port, even one facing a real server?

> **Why this matters:** With the uplink untrusted, snooping treats R1 exactly as it would treat a rogue, dropping its server messages because they arrived on an untrusted port. The feature cannot tell a legitimate server from an attacker by intent, only by which port you trusted. Forgetting to trust the server uplink is the single most common way people break their own DHCP when they first deploy snooping, and the symptom, clients suddenly unable to get addresses right after enabling the feature, is the tell.

---

### Break Scenario 2 — Half-Enabled and Silent

Restore the trust on G0/1. Now create the sneaky misconfiguration where snooping looks configured but is not actually running. Remove only the global enable, leaving the per-VLAN command in place:

```
SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# ip dhcp snooping trust
SW1(config-if)# exit
SW1(config)# no ip dhcp snooping
```

Check the status, then let the rogue R2 try again:
```
SW1# show ip dhcp snooping
(on PC1: release and renew several times, watch which gateway it gets)
```

**✏️ Document the symptoms:**
- What does `show ip dhcp snooping` report about whether snooping is operational?
- With the per-VLAN command still in the config, is snooping actually protecting anything?
- Could the rogue server's poisoned gateway reach PC1 now?

> **Why this matters:** DHCP snooping requires both the global `ip dhcp snooping` and the per-VLAN command to function. With the global command gone, the per-VLAN line sits in the configuration doing absolutely nothing, and the feature is silently inactive. This is dangerous precisely because the config still looks protective at a glance. The rogue server can now poison clients freely. Always confirm with `show ip dhcp snooping` that the feature is operational, never assume it is working just because a line is present in the configuration.

---

### Break Scenario 3 — The Static Host DAI Will Not Trust

Restore the global enable so snooping and DAI are fully working again. Now meet the consequence of DAI's dependence on the binding table. Give R2 a static address with no DHCP binding, and connect it as if it were a legitimate static device like a printer or server on the untrusted port:

```
SW1(config)# ip dhcp snooping
```

On R2, use a static address and try to reach the gateway:
```
R2(config)# interface GigabitEthernet0/0
R2(config-if)# ip address 192.168.10.50 255.255.255.0
R2(config-if)# no shutdown
R2(config-if)# exit
R2# ping 192.168.10.1
```

**✏️ Document the symptoms:**
- Can the static device communicate, or is it cut off?
- It has a valid, non-conflicting IP address, so why does DAI block it?
- What is missing for this device that PC1 has?

> **Why this matters:** DAI validates ARP against the snooping binding table, and a statically addressed device never did a DHCP exchange, so it has no binding. With nothing to match, DAI treats its ARP as a forgery and drops it, isolating a perfectly legitimate device. This is the most common DAI surprise in production, where printers, servers, and other static hosts suddenly go dark after DAI is enabled. The fix is not to weaken DAI, but to vouch for the static host explicitly with an ARP ACL that maps its IP to its MAC.

---

## 🔧 Part 4: Fix It

---

### Fix Scenario 1 — The Untrusted Server

**Your troubleshooting toolkit:**
```
show ip dhcp snooping              ! check which ports are trusted
```

**Structured process:**
1. Note that clients cannot get addresses right after snooping was enabled.
2. Check which ports are trusted and find that the server uplink is not.
3. Trust the uplink toward the legitimate DHCP server.
4. Verify clients can lease again.

**✅ Solution:**
```
SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# ip dhcp snooping trust
SW1(config-if)# exit
```

**Verification:**
```
(on PC1: release and renew, confirm it gets 192.168.10.1 as its gateway)
SW1# show ip dhcp snooping binding
```

---

### Fix Scenario 2 — Half-Enabled and Silent

**Your troubleshooting toolkit:**
```
show ip dhcp snooping              ! confirm whether it is actually operational
```

**Structured process:**
1. Notice the feature is not protecting anything despite being partly configured.
2. Confirm whether the global enable is present.
3. Add the global command so both halves are in place.
4. Verify snooping reports as operational and the rogue is blocked again.

**✅ Solution:**
```
SW1(config)# ip dhcp snooping
```

**Verification:**
```
SW1# show ip dhcp snooping
(on PC1: release and renew, confirm it gets the legitimate gateway)
```

---

### Fix Scenario 3 — The Static Host DAI Will Not Trust

**Your troubleshooting toolkit:**
```
show ip arp inspection statistics      ! watch the dropped counter climb for the static host
show ip dhcp snooping binding          ! confirm the static host has no binding
```

**Structured process:**
1. Confirm the static host is being dropped by DAI for lack of a binding.
2. Create an ARP ACL that maps the host's IP to its MAC address.
3. Apply that ARP ACL to the VLAN so DAI accepts the static host.
4. Verify the static host can now communicate.

**✅ Solution:**
```
! Vouch for the static host by IP and MAC
SW1(config)# arp access-list STATIC-HOSTS
SW1(config-arp-nacl)# permit ip host 192.168.10.50 mac host <R2-mac-address>
SW1(config-arp-nacl)# exit

! Apply the ARP ACL to the inspected VLAN
SW1(config)# ip arp inspection filter STATIC-HOSTS vlan 10
```

> Replace the MAC placeholder with R2's actual address from `show interfaces`. The ARP ACL is how you tell DAI that this specific IP-to-MAC pairing is legitimate even though it has no DHCP binding.

**Verification:**
```
R2# ping 192.168.10.1
SW1# show ip arp inspection statistics
```

---

## 💬 Reflection Questions

1. Describe the rogue DHCP server attack and the ARP spoofing attack. What do they have in common, and why is an ACL powerless to stop either one?

2. Explain the trust boundary. How do you decide which ports to trust, and what does snooping do differently on a trusted port versus an untrusted one?

3. What information does a single DHCP snooping binding contain, and why is that binding table the foundation that DAI depends on?

4. In Break Scenario 2, snooping was half-enabled and silently inactive. Why does the feature require both a global and a per-VLAN command, and why is a partly configured security feature especially dangerous?

5. In Break Scenario 3, DAI blocked a legitimate static host. Explain exactly why, connecting it to how DAI validates ARP, and describe the correct fix.

6. DHCP snooping and DAI are configured together and in a specific order. Explain why snooping must come first and be working before DAI can do its job.

---

## Challenge Extension (Optional)

1. **Rate limit the untrusted ports.** Research `ip dhcp snooping limit rate` and apply it to a user port to defend against DHCP starvation, where an attacker drains the pool by requesting many addresses. Note what happens to a port that exceeds the limit.

2. **Inspect additional ARP fields.** Explore `ip arp inspection validate` with the src-mac, dst-mac, and ip options, which make DAI check the ARP body against the Ethernet header. Explain what additional forgeries these catch.

3. **Add a second static host.** Extend your ARP ACL to vouch for a second statically addressed device, and confirm both are accepted while an unlisted device is still dropped.

4. **Trace a binding end to end.** Clear PC1's lease, watch the binding disappear from the table, renew it, and watch the binding return. Then reason about what DAI would do to PC1's ARP during the window when no binding existed.

---

## Instructor Notes

**Common Student Mistakes**
- Forgetting to trust the uplink toward the real DHCP server, which drops legitimate offers and breaks DHCP the moment snooping is enabled. This is the number one snooping error, and the timing of the symptom is the clue.
- Configuring snooping per-VLAN but forgetting the global `ip dhcp snooping`, leaving the feature silently inactive. Teach students to verify with `show ip dhcp snooping` rather than trusting the config text.
- Not understanding that DAI depends on the snooping binding table, and being baffled when DAI drops legitimate ARP because snooping was never populating bindings.
- Forgetting that static hosts have no binding and need an ARP ACL. Printers and servers going dark after DAI is enabled is the classic production incident.
- Mixing up which ports are trusted. Drive home that trusted faces infrastructure and untrusted faces users, every time.

**Teaching Tips**
- Draw the trust boundary on the board before configuring anything, marking which ports face users and which face infrastructure. Nearly every error in this lab is a trust-boundary error, and making the boundary visual prevents most of them.
- Show the binding table, then immediately show DAI relying on it, so the dependency is concrete. When students later see DAI drop a static host, they will remember that DAI only knows what the binding table told it, and the fix will make sense rather than feeling arbitrary.

---

## Where This Leads

You have now built all three of the access-layer defenses the CCNA asks for, port security controlling which devices may connect, DHCP snooping shutting down rogue servers and building a table of who is really who, and Dynamic ARP Inspection using that table to stop ARP forgery. Together they close the spoofing gap that ACLs could never reach, defending the layer where users actually plug in. This is defense in depth at the access edge, and it completes the hands-on portion of the Security section.

What remains in Security shifts from configuration to understanding, because the remaining topics are tested at the describe level and are better served as concept guides than as five-node labs. Those cover how centralized systems decide who may log in and what they may do, how traffic is protected as it crosses untrusted networks, and how wireless networks are secured. The first of them looks at the framework you met by name in the fundamentals guide, authentication, authorization, and accounting, and how RADIUS and TACACS+ bring it to life. That AAA concept guide is where the Security section goes next.

---

*CCNA – Build It · Observe It · Break It · Fix It series*
*Next: AAA, RADIUS, and TACACS+ (concept guide)*

*This repository is an independent educational resource and is not officially affiliated with Cisco Systems. Cisco, CCNA, Cisco Modeling Labs, and Cisco Networking Academy are trademarks of Cisco Systems, Inc.*
