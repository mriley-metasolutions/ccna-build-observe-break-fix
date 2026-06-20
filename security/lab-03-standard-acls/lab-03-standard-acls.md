# Standard ACLs

**Build It · Observe It · Break It · Fix It**

---

## Overview

So far in the Security section you have controlled who can log into a device and which devices may use a switch port. Access Control Lists answer the next question, the one that runs through the rest of networking. Once traffic is on the network, what is it allowed to reach? An ACL is a list of rules that a router checks traffic against, permitting some packets and denying others. It is the most common traffic-filtering tool in the entire field, used for security, for policy, and as the matching engine behind NAT, route filtering, and more.

This lab covers **standard ACLs**, and the single most important thing to know about them is also their biggest limitation. A standard ACL can match on one thing only, the **source IP address** of the packet. It cannot look at the destination, the protocol, or the port. It only knows where a packet came from. That simplicity makes standard ACLs easy to write and easy to reason about, and it drives the one placement rule you must remember, which we will come back to.

Three ideas govern how every ACL behaves, and they are the ideas this lab is built to drill, because they are exactly what people forget. Every ACL ends with an invisible **implicit deny** that drops anything not explicitly permitted. Rules are checked **top to bottom, and the first match wins**, so order is not cosmetic, it is logic. And an ACL does nothing at all until you **apply it to an interface in a direction**. Master those three and ACLs will stop being mysterious. You will break each of them on purpose before you are done.

**CCNA Exam Alignment:**
- 5.6 – Configure and verify access control lists

**Estimated Time:** 90–120 minutes

---

## Prerequisites

Before completing this lab, you should be comfortable with:
- Basic IOS navigation, including moving between command modes and reading `show` command output
- IPv4 addressing and subnetting, since ACLs match on networks and hosts
- How a router forwards traffic between subnets (the Routing section)
- The least privilege and access control ideas from the Security Fundamentals concept guide
- CML fundamentals, including building or importing a topology and accessing a device console

---

## Key Concept: Wildcard Masks, the Implicit Deny, and Placement

Three pieces of knowledge make standard ACLs click. Take them one at a time.

**Wildcard masks** are how an ACL says which part of an address to match. A wildcard mask is the inverse of a subnet mask. A 0 means the corresponding bit must match exactly, and a 1, shown as 255 in an octet, means ignore it. That inversion catches everyone at first, so keep this table handy:

| Wildcard mask | Matches | Shortcut |
|---------------|---------|----------|
| 0.0.0.0 | One exact host | `host` keyword |
| 0.0.0.255 | An entire /24 subnet | last octet is a wildcard |
| 0.0.255.255 | An entire /16 | |
| 255.255.255.255 | Every address | `any` keyword |

So `192.168.1.0 0.0.0.255` matches every host in the 192.168.1.0/24 network, and `host 192.168.1.10` matches that one machine.

**The implicit deny** is the rule you never see and must never forget. At the bottom of every ACL sits an invisible `deny any`. If a packet reaches the end of your list without matching a permit, it is dropped. This means an ACL that contains only deny statements will block everything, because nothing was ever permitted. It is the most common ACL mistake in existence, and you will create it in Break Scenario 1.

**Placement** is where standard ACLs reveal their limitation. Because a standard ACL matches only the source, placing it near the source would block that source from reaching every destination, not just the one you meant to protect. So the rule is firm: **place a standard ACL as close to the destination as possible.** That way it filters traffic only where it is headed somewhere you care about, and leaves the source free to reach everything else.

> **Get ahead of the confusion:** Read every ACL out loud, line by line, and finish with the words "and deny everything else." That last phrase is the implicit deny, and saying it every time is the habit that prevents the most common ACL failure there is. An ACL is not what you wrote. It is what you wrote plus a silent deny at the end.

---

## Topology

```
        [PC1]                              [SRV1]
     192.168.1.10                        192.168.3.10
     (allowed to reach SRV1)             (protected server)
          |                                   |
        G0/0                                G0/2
        192.168.1.1                        192.168.3.1
                 \                         /
                  \                       /
                       [R1]  router with the ACL
                        |
                      G0/1
                      192.168.2.1
                        |
                     [PC2]
                  192.168.2.10
                  (to be denied to SRV1)
```

**Nodes (4 total, within CML Free Edition limit):**
- R1 (IOSv router, where the ACL is configured and applied)
- PC1 (an authorized source, on 192.168.1.0/24)
- PC2 (the source to be blocked from the server, on 192.168.2.0/24)
- SRV1 (the protected server, on 192.168.3.0/24)

> **R1 routes between all three subnets,** which are directly connected, so no extra routing is needed. The policy goal is simple. PC1's network may reach SRV1, and PC2's network may not, while PC2 keeps its access to everything else. Confirm interface names with `show ip interface brief` and adjust.

---

## Addressing Table

| Device | Interface | IP Address    | Subnet Mask     | Role |
|--------|-----------|---------------|-----------------|------|
| R1     | G0/0      | 192.168.1.1   | 255.255.255.0   | Gateway for PC1 |
| R1     | G0/1      | 192.168.2.1   | 255.255.255.0   | Gateway for PC2 |
| R1     | G0/2      | 192.168.3.1   | 255.255.255.0   | Gateway for SRV1 |
| PC1    | NIC       | 192.168.1.10  | 255.255.255.0   | Authorized source |
| PC2    | NIC       | 192.168.2.10  | 255.255.255.0   | Blocked source |
| SRV1   | NIC       | 192.168.3.10  | 255.255.255.0   | Protected server |

---

## Lab Objectives

By the end of this lab you will be able to:

1. Explain what an ACL does and why a standard ACL matches only the source address
2. Write wildcard masks to match a single host and an entire subnet
3. Configure a numbered standard ACL and apply it to an interface with a direction
4. Account for the implicit deny and for top-down, first-match processing
5. Place a standard ACL correctly, close to the destination
6. Verify ACLs with `show access-lists` and `show ip interface`
7. Troubleshoot the most common standard ACL failures

---

## 🧱 Part 1: Build It

### Step 1 — Configure Interfaces and Confirm Reachability

Bring up R1's three interfaces and configure the hosts with their addresses and gateways. Before adding any ACL, confirm everything can reach everything, so you have a clean baseline.

```
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip address 192.168.1.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit
R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip address 192.168.2.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit
R1(config)# interface GigabitEthernet0/2
R1(config-if)# ip address 192.168.3.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit
```

Configure each host with its address and a default gateway matching its R1 interface. Then confirm that PC1 and PC2 can both reach SRV1, which proves the network works before you restrict it.

```
PC1 ping 192.168.3.10      (should succeed)
PC2 ping 192.168.3.10      (should succeed)
```

### Step 2 — Write the Standard ACL

Create a numbered standard ACL that permits PC1's network. Standard ACLs use numbers 1 through 99. You permit what you want to allow, and you rely on the implicit deny to block the rest.

```
R1(config)# access-list 10 permit 192.168.1.0 0.0.0.255
```

> **Read it out loud:** "Permit the 192.168.1.0/24 network, and deny everything else." That second clause is the implicit deny, and it is doing the real work here. PC1's network is permitted, and PC2's network, never mentioned, falls through to the silent deny. One line of configuration enforces the whole policy.

### Step 3 — Apply the ACL Close to the Destination

This is where the placement rule matters. The destination you are protecting is SRV1, so apply the ACL outbound on the interface facing SRV1, which is G0/2. Outbound means the router checks the ACL as traffic leaves that interface toward the server.

```
R1(config)# interface GigabitEthernet0/2
R1(config-if)# ip access-group 10 out
R1(config-if)# exit
```

> **Why outbound on G0/2 and not near the sources:** Placed here, the ACL only filters traffic actually headed for SRV1. PC2 is blocked from the server but keeps its access to PC1 and everywhere else, because its traffic to those places never passes through this filter. Had you placed this near PC2 instead, you would have cut PC2 off from the entire network. Standard ACLs go near the destination for exactly this reason.

### Step 4 — Save the Configuration

```
R1# copy running-config startup-config
```

---

## 👀 Part 2: Observe It

### 2.1 — Test the Policy

The whole point, in two pings:

```
PC1 ping 192.168.3.10      (should succeed)
PC2 ping 192.168.3.10      (should now fail)
```

PC1 reaches SRV1 because it is permitted. PC2 is dropped by the implicit deny. Now confirm PC2 still has its other access:

```
PC2 ping 192.168.1.10      (should still succeed)
```

PC2 can reach PC1 freely, because that traffic never crosses the filtered interface. This is the placement rule paying off.

**✏️ Prediction checkpoint:** Before you run them, predict the result of each of the three pings above, and name which rule, explicit or implicit, decides each one.

### 2.2 — Read the ACL and Its Hit Counters

```
R1# show access-lists
```

**What to look for:** Your single permit line, and next to it a count of how many packets have matched it. Those match counters are one of the most useful troubleshooting tools you have, because they prove which lines are actually catching traffic. After PC1's pings, the permit line's counter climbs.

### 2.3 — Confirm Where the ACL Is Applied

```
R1# show ip interface GigabitEthernet0/2
```

Look for the line reporting that outgoing access list 10 is set on the interface. An ACL that is written but not applied does nothing, so confirming the application is a real verification step, not a formality.

**✏️ Think it through:** PC2's traffic to SRV1 is dropped, but `show access-lists` shows only a permit line with no deny. Explain where, exactly, PC2's packets are being denied, since you never wrote a deny statement.

---

## 💥 Part 3: Break It

> Work through each scenario one at a time. Document the symptoms fully before you move to Part 4.

---

### Break Scenario 1 — The Implicit Deny Bites

A colleague decides to rewrite the rule. Instead of permitting the allowed network, they reason that it is cleaner to just deny the blocked network and let everyone else through. Replace the ACL with their version:

```
R1(config)# no access-list 10
R1(config)# access-list 10 deny 192.168.2.0 0.0.0.255
R1(config)# interface GigabitEthernet0/2
R1(config-if)# ip access-group 10 out
R1(config-if)# exit
```

Now test both sources:
```
PC2 ping 192.168.3.10      (should fail, as intended)
PC1 ping 192.168.3.10      (what happens?)
```

**✏️ Document the symptoms:**
- PC2 is correctly blocked, so the deny worked. But can PC1 still reach SRV1?
- The ACL has no deny for PC1's network, so why is PC1 affected?
- What single line would have made this rule behave as the colleague intended?

> **Why this matters:** This is the implicit deny in action, and it is the most common ACL mistake there is. The list contains one deny and nothing else, so after PC1's traffic fails to match that deny, it falls straight into the silent `deny any` at the bottom and is dropped. An ACL that only denies, with no permit, blocks everyone. If you intend to deny one thing and allow the rest, you must explicitly permit the rest. Reading the list aloud and ending with "and deny everything else" would have caught this instantly.

---

### Break Scenario 2 — Order Changes Everything

Fix the previous scenario by returning to the working permit, then introduce an ordering mistake. Imagine you want to permit PC1's network but you lead with a broad permit, intending to tighten it afterward:

```
R1(config)# no access-list 10
R1(config)# access-list 10 permit any
R1(config)# access-list 10 deny 192.168.2.0 0.0.0.255
R1(config)# interface GigabitEthernet0/2
R1(config-if)# ip access-group 10 out
R1(config-if)# exit
```

Now test:
```
PC2 ping 192.168.3.10      (is it blocked?)
```

**✏️ Document the symptoms:**
- Is PC2 blocked from SRV1, or does it get through?
- The list clearly contains a deny for PC2's network, so why does it not take effect?
- What does this tell you about the order in which the router reads ACL lines?

> **Why this matters:** ACLs are processed top to bottom, and the first match wins. PC2's traffic hits `permit any` on the very first line, is permitted, and the router never even looks at the deny below it. A deny that sits below a permit covering the same traffic is dead code. Order is logic, not formatting. And note the practical wrinkle, that numbered ACLs only append new lines to the bottom, so fixing the order means deleting and rewriting the whole list. That limitation is one of the best reasons to prefer named ACLs, which let you insert and remove individual lines by sequence number.

---

### Break Scenario 3 — The Wrong Place

Restore the correct, working ACL, applied outbound on G0/2. Now move it to the wrong location to see why placement is a rule and not a preference. Remove it from G0/2 and instead apply it inbound on PC2's interface, as if trying to block PC2 at its source:

```
R1(config)# interface GigabitEthernet0/2
R1(config-if)# no ip access-group 10 out
R1(config-if)# exit
R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip access-group 10 in
R1(config-if)# exit
```

> ACL 10 permits only 192.168.1.0/24. Applied inbound on G0/1, it now judges all traffic entering from PC2's network.

Test what PC2 can reach:
```
PC2 ping 192.168.3.10      (blocked from the server, as wanted)
PC2 ping 192.168.1.10      (can PC2 still reach PC1?)
```

**✏️ Document the symptoms:**
- Is PC2 blocked from SRV1? Is it now also blocked from PC1?
- Why does placing the ACL at the source affect every destination PC2 tries to reach?
- How does moving the ACL back to the destination interface fix the over-blocking?

> **Why this matters:** Applied at the source, the ACL judges everything PC2 sends, no matter where it is going. PC2's traffic does not match the lone permit for PC1's network, so the implicit deny drops all of it, cutting PC2 off from PC1 and SRV1 alike. Because a standard ACL cannot see the destination, it has no way to filter "to SRV1 only" from this position. That is the entire reason the placement rule exists, and it is the limitation that extended ACLs were created to overcome.

---

## 🔧 Part 4: Fix It

---

### Fix Scenario 1 — The Implicit Deny Bites

**Your troubleshooting toolkit:**
```
show access-lists                  ! a list of only denies is the red flag
```

**Structured process:**
1. Confirm that traffic you intended to allow is being dropped.
2. Read the ACL and notice it contains no permit for that traffic.
3. Add an explicit permit for everything that should pass after the deny.
4. Verify the allowed traffic flows again.

**✅ Solution:**
```
R1(config)# access-list 10 permit any
```

> Added after the deny, this catches everything the deny did not, so legitimate traffic passes while PC2's network stays blocked. The order is correct because the specific deny comes before the broad permit.

**Verification:**
```
PC1 ping 192.168.3.10      (should succeed)
PC2 ping 192.168.3.10      (should fail)
```

---

### Fix Scenario 2 — Order Changes Everything

**Your troubleshooting toolkit:**
```
show access-lists                  ! check whether a broad permit sits above a deny
```

**Structured process:**
1. Confirm the deny is being bypassed.
2. Identify the broad permit sitting above it.
3. Rewrite the list with the specific rule first and the broad rule last.
4. Verify the deny now takes effect.

**✅ Solution:**
```
R1(config)# no access-list 10
R1(config)# access-list 10 deny 192.168.2.0 0.0.0.255
R1(config)# access-list 10 permit any
```

**Verification:**
```
PC2 ping 192.168.3.10      (should fail)
PC1 ping 192.168.3.10      (should succeed)
```

---

### Fix Scenario 3 — The Wrong Place

**Your troubleshooting toolkit:**
```
show ip interface GigabitEthernet0/1
show ip interface GigabitEthernet0/2
```

**Structured process:**
1. Recognize the over-blocking, where the source is cut off from everything.
2. Remove the ACL from the source-side interface.
3. Reapply it outbound on the interface facing the protected destination.
4. Verify the source reaches everything except the protected server.

**✅ Solution:**
```
R1(config)# interface GigabitEthernet0/1
R1(config-if)# no ip access-group 10 in
R1(config-if)# exit
R1(config)# interface GigabitEthernet0/2
R1(config-if)# ip access-group 10 out
R1(config-if)# exit
```

**Verification:**
```
PC2 ping 192.168.3.10      (should fail)
PC2 ping 192.168.1.10      (should succeed)
```

---

## 💬 Reflection Questions

1. A standard ACL can match on only one thing. What is it, and how does that single limitation lead directly to the rule about where you place a standard ACL?

2. Write the wildcard mask that matches a single host, and the one that matches an entire /24 subnet. Explain why a wildcard mask is described as the inverse of a subnet mask.

3. Explain the implicit deny in your own words. Why does an ACL containing only deny statements block all traffic, and what must you add to deny some traffic while allowing the rest?

4. ACLs are processed top to bottom with first match winning. Using Break Scenario 2 as your example, explain how a correctly written deny can be rendered completely ineffective by the line above it.

5. In Break Scenario 3, applying the ACL near the source cut PC2 off from everything. Explain why, and why a standard ACL is incapable of filtering "to one specific destination" when placed at the source.

6. Numbered ACLs only append lines to the bottom, which made fixing the order in this lab require a full rewrite. Explain how named ACLs improve on this, and why that makes them the better choice for anything you will maintain over time.

---

## Challenge Extension (Optional)

1. **Rewrite it as a named ACL.** Convert your numbered ACL 10 into a named standard ACL using `ip access-list standard`, and explore how sequence numbers let you insert and delete individual lines without rewriting the whole list.

2. **Filter a single host.** Modify the ACL to block one specific host on PC2's network rather than the whole subnet, using the `host` keyword, while letting the rest of that subnet reach SRV1.

3. **Secure the VTY lines.** Apply a standard ACL to the router's VTY lines with `access-class`, so that only PC1's network may SSH into R1. This connects ACLs back to the device access hardening from the SSH lab.

4. **Read the counters to prove a path.** Add a `permit` line for PC2 temporarily, watch its hit counter climb as PC2 sends traffic, then remove it. Use the counters to demonstrate exactly which line each ping matches.

---

## Instructor Notes

**Common Student Mistakes**
- Forgetting the implicit deny. Students write a list of denies, or a single permit, and are baffled when legitimate traffic is dropped. Make them say "and deny everything else" after every ACL, every time.
- Confusing wildcard masks with subnet masks. Writing `255.255.255.0` where `0.0.0.255` belongs is constant. Reinforce that the wildcard is the inverse.
- Applying a standard ACL near the source and over-blocking. This is the placement rule made painful, and it is worth letting them feel it before fixing it.
- Writing the ACL but never applying it to an interface, then wondering why nothing is filtered. An unapplied ACL is inert.
- Getting rule order wrong, especially leading with a broad permit that shadows everything below it.

**Teaching Tips**
- Lean on `show access-lists` and its hit counters constantly. The counters turn an abstract list into something students can watch react, and proving which line catches a ping makes first-match processing concrete in a way no diagram does.
- Have students read every ACL aloud as a sentence, ending with the implicit deny. This one habit prevents the majority of ACL errors, and it costs nothing to build.

---

## Where This Leads

You now own the foundational logic of access control, the wildcard masks, the implicit deny, the first-match order, and the placement rule. Those ideas do not change for the rest of your career. What changes is how much a single rule can see, and that is exactly the wall you hit in this lab. A standard ACL can only match the source, which forced it near the destination and made it incapable of saying "this source may reach that server but not this other one."

Extended ACLs remove that wall. They can match on source and destination, on protocol, and on port number, which means they can express far more precise policy, like "permit web traffic to the server but deny everything else from the same host." That precision also flips the placement rule, since an extended ACL can be placed near the source without over-blocking. Everything you drilled here carries forward, and the next lab adds the power that standard ACLs were missing. Extended ACLs are where the Security section goes next.

---

*CCNA – Build It · Observe It · Break It · Fix It series*

*Next: Extended ACLs*

*This repository is an independent educational resource and is not officially affiliated with Cisco Systems. Cisco, CCNA, Cisco Modeling Labs, and Cisco Networking Academy are trademarks of Cisco Systems, Inc.*
