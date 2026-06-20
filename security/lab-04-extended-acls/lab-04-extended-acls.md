# Extended ACLs

**Build It · Observe It · Break It · Fix It**

---

## Overview

In the standard ACL lab you hit a wall, and naming it is the best way to start this one. A standard ACL can match only the source address, which meant it could say "this network may not reach the server," but it could never say "this network may reach the server's web service but not log into it." It was a blunt instrument, and that bluntness forced it near the destination and made it block all-or-nothing.

**Extended ACLs** remove that wall. An extended ACL can match on the **protocol**, the **source** address, the **destination** address, and the **port number**. That is a enormous jump in precision. Instead of permitting or denying an entire conversation, you can permit web traffic to a server while denying Telnet to that same server, allow DNS but block everything else, or let one host reach a service that every other host is denied. This is how real security policy is written, because real policy is rarely "block this network from everything." It is "allow exactly these services and nothing more," which is the principle of least privilege made into configuration.

That added precision changes one rule you learned last lab. Because an extended ACL can see the destination and the service, you no longer have to place it near the destination to avoid over-blocking. In fact, you place it as close to the **source** as possible, so unwanted traffic is dropped at the very edge of the network before it wastes bandwidth crossing the whole path only to be denied at the far end. Everything else you drilled, the implicit deny, the top-down first-match order, and the need to apply the ACL to an interface in a direction, carries over unchanged. This lab adds power, not new fundamentals.

**CCNA Exam Alignment:**
- 5.6 – Configure and verify access control lists

**Estimated Time:** 90–120 minutes

---

## Prerequisites

Before completing this lab, you should be comfortable with:
- The Standard ACLs lab, since this builds directly on wildcard masks, the implicit deny, first-match order, and applying ACLs to interfaces
- IPv4 addressing and subnetting
- The difference between TCP and UDP and the idea that services listen on port numbers (the Services section)
- Basic IOS navigation and reading `show` command output
- CML fundamentals, including building or importing a topology and accessing a device console

---

## Key Concept: More to Match, and a Flipped Placement Rule

An extended ACL statement reads left to right as a series of match conditions, and once you see the structure, every extended ACL becomes readable:

```
access-list 100 permit  tcp   host 192.168.1.10   host 192.168.3.10   eq 80
                 │      │     │                   │                   │
              number  proto  source              destination       dest port
```

That single line says "permit TCP traffic from host 192.168.1.10 to host 192.168.3.10 when the destination port is 80." Extended ACLs use numbers 100 through 199. The protocol can be `ip` to match everything, or `tcp`, `udp`, or `icmp` to narrow it. The port is compared with an operator, most often `eq` for equals, though `gt`, `lt`, `neq`, and `range` exist too.

To match on ports you have to know them, and you already met most of these in the Services section, so this is a refresher, not new memorization:

| Port | Protocol | Service |
|------|----------|---------|
| 22 | TCP | SSH |
| 23 | TCP | Telnet |
| 53 | TCP/UDP | DNS |
| 67, 68 | UDP | DHCP |
| 80 | TCP | HTTP |
| 443 | TCP | HTTPS |
| 123 | UDP | NTP |
| 161, 162 | UDP | SNMP |

The other shift is **placement**. The rule is the mirror image of standard ACLs: **place an extended ACL as close to the source as possible.** Because it can match the full conversation, including the exact destination and service, it can drop unwanted traffic right where it enters the network without harming anything else the source is allowed to do. Filtering at the source also saves bandwidth, since denied traffic never travels across the network only to be dropped at the end.

> **Get ahead of the confusion:** Precision cuts both ways. A standard ACL is hard to get subtly wrong, because it matches one thing. An extended ACL gives you protocol, source, destination, and port to specify, and a mistake in any one of them silently changes what the rule does. Match `eq 443` when you meant `eq 80`, or `udp` when you meant `tcp`, and the service you intended to allow is quietly blocked. The power is real, and so is the responsibility to get every field right.

---

## Topology

```
   [PC1]                                        [SRV1]
 192.168.1.10                                 192.168.3.10
 admin workstation                            web server (HTTP)
     |                                            |
    G0/0  <-- ACL applied inbound (near source)  G0/2
    192.168.1.1                                 192.168.3.1
            \                                   /
             \                                 /
                       [R1]  extended ACL router
                        |
                      G0/1
                      192.168.4.1
                        |
                     [SRV2]
                   192.168.4.10
                   (other server, fully reachable)
```

**Nodes (4 total, within CML Free Edition limit):**
- R1 (IOSv router, where the extended ACL is configured)
- PC1 (the admin workstation and traffic source, on 192.168.1.0/24)
- SRV1 (the web server to be selectively filtered, on 192.168.3.0/24)
- SRV2 (another server, reachable to prove the ACL does not over-block, on 192.168.4.0/24)

> **The policy goal:** PC1 may reach SRV1's web service on port 80, but nothing else on SRV1, no Telnet and no ping. PC1 keeps full access to SRV2 and the rest of the network. Only an extended ACL can express that, and it goes near the source. Confirm interface names with `show ip interface brief` and adjust.

---

## Addressing Table

| Device | Interface | IP Address    | Subnet Mask     | Role |
|--------|-----------|---------------|-----------------|------|
| R1     | G0/0      | 192.168.1.1   | 255.255.255.0   | Gateway for PC1, ACL applied here |
| R1     | G0/2      | 192.168.3.1   | 255.255.255.0   | Gateway for SRV1 |
| R1     | G0/1      | 192.168.4.1   | 255.255.255.0   | Gateway for SRV2 |
| PC1    | NIC       | 192.168.1.10  | 255.255.255.0   | Admin workstation, traffic source |
| SRV1   | NIC / G0/0 | 192.168.3.10 | 255.255.255.0   | Web server |
| SRV2   | NIC / G0/0 | 192.168.4.10 | 255.255.255.0   | Other server |

---

## Lab Objectives

By the end of this lab you will be able to:

1. Explain how extended ACLs differ from standard ACLs in what they can match
2. Write extended ACL statements that match protocol, source, destination, and port
3. Apply an extended ACL near the source and explain why placement flips
4. Build a precise policy that permits one service to a host while denying all others
5. Verify extended ACLs with `show access-lists` and per-line hit counters
6. Troubleshoot port, order, and direction mistakes specific to extended ACLs
7. Recognize the common service port numbers

---

## 🧱 Part 1: Build It

### Step 1 — Configure Interfaces and a Web Service

Bring up R1's three interfaces and configure the hosts. On SRV1, enable the built-in web server so there is a real service listening on port 80 to test against.

```
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip address 192.168.1.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit
R1(config)# interface GigabitEthernet0/2
R1(config-if)# ip address 192.168.3.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit
R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip address 192.168.4.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit
```

On SRV1, turn on the web server so port 80 responds:
```
SRV1(config)# ip http server
```

Configure each host with its address and gateway, then confirm a clean baseline before any filtering. From PC1, everything should be reachable.

```
PC1 ping 192.168.3.10      (SRV1, should succeed)
PC1 ping 192.168.4.10      (SRV2, should succeed)
```

### Step 2 — Write the Extended ACL

Build a numbered extended ACL that permits only web traffic from PC1 to SRV1, denies everything else from PC1 to SRV1, and permits all other traffic. Order is deliberate, and reading it aloud tells the whole story.

```
! Allow PC1 to reach SRV1's web service
R1(config)# access-list 100 permit tcp host 192.168.1.10 host 192.168.3.10 eq 80

! Deny PC1 anything else to SRV1 (Telnet, ping, all of it)
R1(config)# access-list 100 deny ip host 192.168.1.10 host 192.168.3.10

! Allow PC1 to reach everything else on the network
R1(config)# access-list 100 permit ip any any
```

> **Read it out loud:** "Permit PC1's web traffic to SRV1, deny PC1 everything else to SRV1, permit everything else, and deny anything left." The order matters enormously. The specific permit for web comes first, the specific deny for the rest of SRV1 comes second, and the broad permit comes last. Reverse any two of those and the policy breaks, which is exactly what you will prove in Break Scenario 2. Note also that you needed the explicit `permit ip any any`, because without it the implicit deny would block PC1 from SRV2 and everywhere else.

### Step 3 — Apply It Close to the Source

Here is the flipped rule in action. Apply the ACL inbound on G0/0, the interface where PC1's traffic enters the router. This filters PC1's packets the instant they arrive, before they travel anywhere.

```
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip access-group 100 in
R1(config-if)# exit
```

> **Why inbound on the source interface:** Because the ACL matches the full conversation, it can drop the unwanted traffic to SRV1 right at the entry point while letting PC1's traffic to SRV2 sail through. There is no over-blocking, because the rule sees the destination. Placing it here also means PC1's denied packets never cross the router at all, which is the efficiency benefit of source placement.

### Step 4 — Save the Configuration

```
R1# copy running-config startup-config
```

---

## 👀 Part 2: Observe It

### 2.1 — Test the Precise Policy

This is where extended ACLs earn their name. Test three different things to the same server SRV1, plus access to SRV2:

```
PC1 curl http://192.168.3.10      (web to SRV1: should succeed)
PC1 telnet 192.168.3.10           (Telnet to SRV1: should fail)
PC1 ping 192.168.3.10             (ping to SRV1: should fail)
PC1 ping 192.168.4.10             (SRV2: should succeed)
```

> **Testing note:** From a Linux host, `curl` is the cleanest way to test the web service. If your host lacks it, opening a TCP connection to port 80 from a router with `telnet 192.168.3.10 80` proves the same thing, a successful connection means the port is permitted. The result is what matters: one service to SRV1 works while others are blocked, and SRV2 is untouched.

The headline result is that PC1 can reach SRV1 for web and nothing else, while keeping full access to SRV2. A standard ACL could never have drawn that line.

**✏️ Prediction checkpoint:** Before running them, predict the outcome of each test, and name the exact ACL line that decides each one.

### 2.2 — Read the Per-Line Hit Counters

```
R1# show access-lists
```

**What to look for:** Each line with its own match counter. After your tests, the web permit shows hits from the curl, the deny shows hits from the blocked Telnet and ping, and the final permit shows hits from the SRV2 traffic. With multiple lines, these counters are even more revealing than in the standard lab, because they show you exactly which rule caught each kind of traffic.

### 2.3 — Confirm the Application and Direction

```
R1# show ip interface GigabitEthernet0/0
```

Confirm that inbound access list 100 is applied. Direction is half the meaning of an extended ACL, so verifying it is inbound on the source interface is a genuine check, not a formality.

**✏️ Think it through:** PC1's ping to SRV1 failed but its ping to SRV2 succeeded, and both are ICMP. Explain how the same protocol got two different outcomes, something a standard ACL could never produce.

---

## 💥 Part 3: Break It

> Work through each scenario one at a time. Document the symptoms fully before you move to Part 4.

---

### Break Scenario 1 — The Wrong Port

Extended ACLs live or die by the exact match. Change the permit to reference the HTTPS port, 443, instead of the HTTP port, 80, as if you misremembered which the server uses:

```
R1(config)# no access-list 100
R1(config)# access-list 100 permit tcp host 192.168.1.10 host 192.168.3.10 eq 443
R1(config)# access-list 100 deny ip host 192.168.1.10 host 192.168.3.10
R1(config)# access-list 100 permit ip any any
```

Now test the web service:
```
PC1 curl http://192.168.3.10      (should now fail)
```

**✏️ Document the symptoms:**
- Can PC1 still reach SRV1's web service?
- The ACL clearly permits TCP to SRV1, so why is the web request blocked?
- What does this reveal about how precisely an extended ACL must match?

> **Why this matters:** The web server listens on port 80, but your rule now permits only port 443, so the actual web traffic matches nothing in the permit, falls to the explicit deny, and is dropped. The ACL looks almost right, which is the danger. One wrong number in one field silently changed the entire policy. Extended ACLs demand precision in every field, and a single mismatched port is one of the most common reasons a permitted service mysteriously fails.

---

### Break Scenario 2 — Order Undoes the Policy

Restore port 80, then reorder the lines so the broad permit comes first, a tempting mistake when you add a general allow before tightening it:

```
R1(config)# no access-list 100
R1(config)# access-list 100 permit ip any any
R1(config)# access-list 100 permit tcp host 192.168.1.10 host 192.168.3.10 eq 80
R1(config)# access-list 100 deny ip host 192.168.1.10 host 192.168.3.10
```

Now test what should be blocked:
```
PC1 ping 192.168.3.10             (should this be blocked? test it)
PC1 telnet 192.168.3.10           (should this be blocked? test it)
```

**✏️ Document the symptoms:**
- Are Telnet and ping to SRV1 blocked anymore, or do they get through?
- The deny for SRV1 is still in the list, so why has it stopped working?
- What does this confirm about first-match processing that you also saw with standard ACLs?

> **Why this matters:** First-match processing does not care that you are using extended ACLs. PC1's ping to SRV1 hits `permit ip any any` on the first line, is permitted, and the router never reaches the deny below. The entire restrictive policy is bypassed by one misplaced broad permit. The specific rules must come before the general ones, every time. This is the same lesson as the standard lab, which is the point. The fundamentals do not change, no matter how much matching power you add.

---

### Break Scenario 3 — The Wrong Direction

Restore the correct ACL order. Now apply it in the wrong direction to see why direction is half of an extended ACL's meaning. Remove it from inbound and apply it outbound on the same interface:

```
R1(config)# interface GigabitEthernet0/0
R1(config-if)# no ip access-group 100 in
R1(config-if)# ip access-group 100 out
R1(config-if)# exit
```

Now test PC1 reaching SRV1's web service:
```
PC1 curl http://192.168.3.10      (does it still work?)
```

**✏️ Document the symptoms:**
- Does the policy still behave correctly with the ACL applied outbound on G0/0?
- The ACL matches source 192.168.1.10, so think about which packets actually travel outbound on G0/0. Are they from PC1, or to PC1?
- Why does the same ACL produce the wrong result simply by changing direction?

> **Why this matters:** Direction determines which packets the ACL ever sees. Inbound on G0/0, the ACL inspects traffic from PC1 as it enters the router, which is exactly what its source-based rules expect. Applied outbound on G0/0, the ACL inspects traffic leaving toward PC1, the replies, whose source is the server, not PC1. The rules no longer match the way you intended, and the policy misbehaves. An extended ACL's source and destination fields are written from the perspective of a direction, and applying it the wrong way is like reading a map upside down. Source placement, inbound on the interface closest to the traffic origin, is what keeps the perspective correct.

---

## 🔧 Part 4: Fix It

---

### Fix Scenario 1 — The Wrong Port

**Your troubleshooting toolkit:**
```
show access-lists                  ! check the port in the permit line against the real service
```

**Structured process:**
1. Confirm the intended service is being blocked.
2. Inspect the permit line and the port it references.
3. Compare it against the port the service actually uses.
4. Correct the port and retest.

**✅ Solution:**
```
R1(config)# no access-list 100
R1(config)# access-list 100 permit tcp host 192.168.1.10 host 192.168.3.10 eq 80
R1(config)# access-list 100 deny ip host 192.168.1.10 host 192.168.3.10
R1(config)# access-list 100 permit ip any any
```

**Verification:**
```
PC1 curl http://192.168.3.10      (should succeed)
```

---

### Fix Scenario 2 — Order Undoes the Policy

**Your troubleshooting toolkit:**
```
show access-lists                  ! look for a broad permit sitting above specific rules
```

**Structured process:**
1. Confirm the restrictive rules are being bypassed.
2. Find the broad `permit ip any any` placed too high.
3. Rewrite the list with specific rules first and the broad permit last.
4. Verify the policy is enforced again.

**✅ Solution:**
```
R1(config)# no access-list 100
R1(config)# access-list 100 permit tcp host 192.168.1.10 host 192.168.3.10 eq 80
R1(config)# access-list 100 deny ip host 192.168.1.10 host 192.168.3.10
R1(config)# access-list 100 permit ip any any
```

**Verification:**
```
PC1 ping 192.168.3.10             (should fail)
PC1 curl http://192.168.3.10      (should succeed)
```

---

### Fix Scenario 3 — The Wrong Direction

**Your troubleshooting toolkit:**
```
show ip interface GigabitEthernet0/0
```

**Structured process:**
1. Recognize the policy misbehaves even though the ACL itself is correct.
2. Check the direction the ACL is applied.
3. Reapply it inbound on the source interface so it inspects PC1's outbound traffic.
4. Verify the policy works as intended.

**✅ Solution:**
```
R1(config)# interface GigabitEthernet0/0
R1(config-if)# no ip access-group 100 out
R1(config-if)# ip access-group 100 in
R1(config-if)# exit
```

**Verification:**
```
PC1 curl http://192.168.3.10      (should succeed)
PC1 ping 192.168.3.10             (should fail)
```

---

## 💬 Reflection Questions

1. List the four things an extended ACL can match on. Using one of your tests from this lab, explain how matching the destination and port let you do something a standard ACL never could.

2. The placement rule for extended ACLs is the opposite of the rule for standard ACLs. State both rules and explain why the difference comes down to how much each ACL type can see.

3. In Break Scenario 1, a single wrong port number silently blocked the web service. Explain why extended ACLs are both more powerful and more error-prone than standard ACLs because of this.

4. In the Observe section, PC1's ping to SRV1 failed while its ping to SRV2 succeeded, even though both are ICMP. Explain exactly how the ACL produced two different outcomes for the same protocol.

5. Break Scenario 2 repeated the order lesson from the standard ACL lab. Explain why first-match processing behaves identically regardless of ACL type, and what that tells you about how much of ACL skill is fundamentals versus syntax.

6. In Break Scenario 3, the correct ACL applied in the wrong direction produced the wrong result. Explain how an extended ACL's source and destination fields depend on the direction it is applied, and why source placement keeps that perspective correct.

---

## Challenge Extension (Optional)

1. **Add a second permitted service.** Extend the policy so PC1 may also reach SRV1 over HTTPS on port 443, then test both web ports. Explain where in the list the new line must go.

2. **Write it as a named extended ACL.** Rebuild the policy using `ip access-list extended`, and use sequence numbers to insert a new rule in the middle without rewriting the whole list. Compare the maintainability with the numbered version.

3. **Filter a whole subnet by service.** Change the host matches to subnet matches with wildcard masks, so the entire 192.168.1.0/24 network is permitted web access to SRV1 but denied everything else to it.

4. **Explore the established keyword.** Research the `established` keyword for TCP, which matches only return traffic of an existing session, and explain how it lets you allow replies to connections your hosts initiated while blocking new connections from outside.

---

## Instructor Notes

**Common Student Mistakes**
- Matching the wrong protocol or port. Using `ip` when they need `tcp`, or `eq 443` when they mean `eq 80`, silently breaks the intended service. Drive home that every field must be exact.
- Placing the extended ACL near the destination out of habit from the standard lab, or applying it in the wrong direction. Reinforce source placement and inbound on the origin interface.
- Getting line order wrong, especially leading with a broad permit that shadows the specific rules. This is the same trap as standard ACLs, and it is worth showing that the fundamentals did not change.
- Forgetting the explicit `permit ip any any` and being surprised that the implicit deny cut the source off from everything except the one permitted flow.
- Forgetting that blocking a protocol can also block its return traffic, which becomes important the moment they try to filter real two-way sessions.

**Teaching Tips**
- Test the same destination on two different ports back to back, one allowed and one denied. Watching one service reach a server while another is refused, to the very same host, is the moment extended ACLs click, and it is impossible to demonstrate with standard ACLs.
- Keep `show access-lists` on screen and watch the per-line hit counters move as each test runs. With multiple lines, seeing exactly which rule catches each packet makes first-match processing and precise matching concrete at the same time.

---

## Where This Leads

You now have the full ACL toolkit, the blunt force of standard ACLs and the precision of extended ones, along with the judgment of where each belongs. You can write least privilege into a router, allowing exactly the services a host needs and denying the rest, which is one of the most valuable and most used skills in all of networking. Just as important, you saw that the fundamentals never changed. The implicit deny, first-match order, and the need to apply and direct an ACL governed both labs identically, and only the matching power grew.

ACLs filter traffic based on what is written in the packet headers, trusting that those headers are honest. The next labs question that trust. An attacker can forge a MAC address, stand up a rogue DHCP server, or poison the ARP cache to redirect traffic, and no ACL will stop them, because the deception happens below the layer ACLs inspect. Defending against that requires Layer 2 protections built for exactly these spoofing attacks. DHCP Snooping and Dynamic ARP Inspection are where the Security section goes next, and they close the gap that ACLs cannot reach.

---

*CCNA – Build It · Observe It · Break It · Fix It series*
*Next: DHCP Snooping and Dynamic ARP Inspection*

*This repository is an independent educational resource and is not officially affiliated with Cisco Systems. Cisco, CCNA, Cisco Modeling Labs, and Cisco Networking Academy are trademarks of Cisco Systems, Inc.*
