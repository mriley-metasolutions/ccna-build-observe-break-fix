# NAT and PAT
 
**Build It · Observe It · Break It · Fix It**
 
---
 
## Overview
 
The hosts on your network carry private addresses, the 192.168, 10, and 172.16 ranges that DHCP has been handing out. Those addresses are free, plentiful, and completely unroutable on the public internet. No packet with a 192.168 source address will ever cross the open internet, by design, because everyone uses those same private ranges. So how does a laptop with a 192.168 address load a web page? Network Address Translation, or NAT, is the answer. NAT sits at the border of your network and rewrites private inside addresses into public ones on the way out, and reverses the translation on the way back, so a privately addressed host can hold a conversation with the wider world.
 
There are three forms of NAT, and they build on one another, which is exactly how this lab walks through them. **Static NAT** maps one inside address to one outside address permanently, a fixed one-to-one relationship. **Dynamic NAT** hands out public addresses from a shared pool, first come first served, which works until the pool runs dry. And **PAT**, also called NAT overload, lets many inside hosts share a single public address by tracking port numbers to keep the conversations apart. PAT is the form that actually runs the internet. It is how your home router puts every phone, laptop, and smart television in your house behind one public address from your Internet service provider.
 
You will configure all three in sequence. Along the way you will deliberately exhaust a dynamic pool, watch a host get left out in the cold, and then solve that exact problem with PAT, which is the most honest way to understand why overload exists.
 
**CCNA Exam Alignment:**
- 4.1 – Configure and verify inside source NAT using static and pools

**Estimated Time:** 90–120 minutes
 
---
 
## Key Concept: Inside, Outside, Local, and Global
 
NAT has a vocabulary that trips people up until they see it laid out, and the exam expects you to know it. Every translated address has two names depending on which side of the border you are standing on.
 
| Term | What it means | Example in this lab |
|------|---------------|---------------------|
| **Inside local** | An inside host's real private address, as seen inside | 192.168.1.10 |
| **Inside global** | The public address that represents that inside host to the outside world | 203.0.113.17 |
| **Outside global** | A real outside host's public address | 8.8.8.8 |
| **Outside local** | How that outside host appears to the inside, usually the same in basic NAT | 8.8.8.8 |
 
The pair that matters most for this lab is inside local and inside global, because those are the two columns you will read in the translation table. Inside local is who the host really is. Inside global is the disguise NAT gives it for its trip across the internet.
 
There is one more idea that makes the whole thing work, and it is the key to PAT specifically. When many hosts share a single inside global address, NAT keeps their conversations separate by also tracking the **port number** of each connection. Host A talking to a web server and host B talking to the same web server look identical on the outside except for their port numbers, and that is enough for NAT to deliver each reply back to the right host.
 
> **Get ahead of the confusion:** NAT only acts on traffic that crosses between an interface marked `ip nat inside` and one marked `ip nat outside`. If those designations are missing or backwards, NAT does absolutely nothing, silently, no matter how perfect the rest of your configuration is. That missing designation is the single most common NAT failure, which is why it is the first thing you will break.
 
---
 
## Topology
 
```
   [PC1]            [PC2]
 192.168.1.10     192.168.1.11      (inside local, private)
      \             /
       \           /
         [SW1]
           |
         G0/0  192.168.1.1   (ip nat inside)
         [R1]   NAT border router
         G0/1  203.0.113.1    (ip nat outside)
           |
     203.0.113.0/30
           |
         G0/0  203.0.113.2
         [R2]   ISP / Internet
         Lo0:  8.8.8.8        (a server out on the internet)
```
 
**Nodes (5 total, within CML Free Edition limit):**
- R1 (IOSv router, the NAT border router)
- R2 (IOSv router, the ISP and the internet)
- SW1 (IOSvL2 switch for the inside LAN)
- PC1, PC2 (inside hosts with private addresses)
> **A note on the public addresses:** This guide uses 203.0.113.0/30 for the ISP link and a 203.0.113.16/28 block for the public addresses your ISP "assigned" to you. These are documentation ranges, standing in for real public space. R2 routes that public block back to R1, the way a real ISP would. As always, confirm interface names with `show ip interface brief` and adjust.
 
---
 
## Addressing Table
 
| Device | Interface | IP Address     | Subnet Mask       | NAT Role |
|--------|-----------|----------------|-------------------|----------|
| R1     | G0/0      | 192.168.1.1    | 255.255.255.0     | inside |
| R1     | G0/1      | 203.0.113.1    | 255.255.255.252   | outside |
| R2     | G0/0      | 203.0.113.2    | 255.255.255.252   | ISP link |
| R2     | Loopback0 | 8.8.8.8        | 255.255.255.255   | Internet host |
| PC1    | NIC       | 192.168.1.10   | 255.255.255.0     | inside local |
| PC2    | NIC       | 192.168.1.11   | 255.255.255.0     | inside local |
 
> **Public addresses used for translation (from the 203.0.113.16/28 block):** Static NAT uses 203.0.113.17 for PC1. Dynamic NAT uses a tiny pool to demonstrate exhaustion. PAT overloads R1's outside interface, 203.0.113.1.
 
---
 
## Lab Objectives
 
By the end of this lab you will be able to:
 
1. Explain why NAT is necessary and define inside local, inside global, outside local, and outside global
2. Designate inside and outside NAT interfaces
3. Configure static NAT for a one-to-one mapping
4. Configure dynamic NAT with an address pool and observe pool exhaustion
5. Configure PAT (NAT overload) so many hosts share one public address
6. Read and interpret the NAT translation table, including port numbers
7. Identify and resolve three common NAT failures
---
 
## 🧱 Part 1: Build It
 
### Step 1 — Configure Interfaces and Routing
 
Bring up the interfaces and add the routing that lets inside traffic reach the internet and replies find their way back.
 
**R1:**
```
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip address 192.168.1.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit
R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip address 203.0.113.1 255.255.255.252
R1(config-if)# no shutdown
R1(config-if)# exit
 
! Send all internet-bound traffic to the ISP
R1(config)# ip route 0.0.0.0 0.0.0.0 203.0.113.2
```
 
**R2 (the ISP):**
```
R2(config)# interface GigabitEthernet0/0
R2(config-if)# ip address 203.0.113.2 255.255.255.252
R2(config-if)# no shutdown
R2(config-if)# exit
R2(config)# interface Loopback0
R2(config-if)# ip address 8.8.8.8 255.255.255.255
R2(config-if)# exit
 
! Route the public NAT block back to R1, the way an ISP routes your assigned space
R2(config)# ip route 203.0.113.16 255.255.255.240 203.0.113.1
```
 
> Configure PC1 and PC2 with their static inside addresses and a default gateway of 192.168.1.1, then confirm PC1 can reach R1's inside interface before going further.
 
### Step 2 — Designate the Inside and Outside Interfaces
 
This is the foundation every form of NAT depends on. Mark the interface facing the LAN as inside and the interface facing the ISP as outside.
 
```
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip nat inside
R1(config-if)# exit
R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip nat outside
R1(config-if)# exit
```
 
> **Nothing translates without these two lines.** NAT only operates on traffic moving between an inside-marked interface and an outside-marked one. This is the first thing you will break later, because it is the first thing real engineers forget.
 
---
 
### Stage A — Static NAT
 
Static NAT gives one inside host a permanent, dedicated public address. It is the right tool for a server that must always be reachable at the same public address.
 
```
! Permanently map PC1's private address to a public one
R1(config)# ip nat inside source static 192.168.1.10 203.0.113.17
```
 
**Verify it:**
```
R1# show ip nat translations
```
 
You should see a permanent mapping from 192.168.1.10 to 203.0.113.17. Generate traffic from PC1:
```
PC1> ping 8.8.8.8
```
 
The ping should succeed, and the translation now shows the active conversation. PC1 reaches the internet wearing the address 203.0.113.17.
 
**✏️ Observe:** Notice the mapping exists even before PC1 sends any traffic. Static NAT is configured once and stands permanently, which is exactly why it suits servers.
 
---
 
### Stage B — Dynamic NAT, and Watching the Pool Run Dry
 
Now remove the static mapping and switch to a dynamic pool, where inside hosts borrow a public address on demand. To make the lesson land, the pool will hold just one address, so the second host to need it gets nothing.
 
```
! Remove the static mapping so it does not conflict with the dynamic rule
R1(config)# no ip nat inside source static 192.168.1.10 203.0.113.17
 
! Define which inside hosts are allowed to be translated
R1(config)# access-list 1 permit 192.168.1.0 0.0.0.255
 
! A pool with exactly one usable address, to force exhaustion
R1(config)# ip nat pool ISP-POOL 203.0.113.18 203.0.113.18 netmask 255.255.255.240
 
! Translate the permitted hosts using the pool
R1(config)# ip nat inside source list 1 pool ISP-POOL
```
 
**Now create the traffic jam.** Ping the internet from both hosts:
```
PC1> ping 8.8.8.8
PC2> ping 8.8.8.8
```
 
Then look at what happened:
```
R1# show ip nat translations
R1# show ip nat statistics
```
 
**✏️ Document what you see:**
- One host gets a translation from the pool and succeeds. Which one, and what address did it borrow?
- The other host finds the pool empty and fails. What do the statistics show about missed translations?
- This is dynamic NAT working exactly as designed. There were more hosts than public addresses, and someone was left out.
> **This is the problem PAT exists to solve.** A pool of public addresses is finite, and the moment you have more simultaneous inside hosts than pool addresses, the extras are stranded. Buying more public addresses is expensive and often impossible. The next stage is the elegant answer.
 
---
 
### Stage C — PAT, the Address Everyone Shares
 
PAT lets every inside host share a single public address by tracking port numbers. Replace the dynamic pool rule with an overload rule pointed at the outside interface.
 
```
! Remove the dynamic pool rule
R1(config)# no ip nat inside source list 1 pool ISP-POOL
 
! Overload: translate every permitted host onto the outside interface address,
! using port numbers to keep their conversations separate
R1(config)# ip nat inside source list 1 interface GigabitEthernet0/1 overload
```
 
**Now both hosts ping at once:**
```
PC1> ping 8.8.8.8
PC2> ping 8.8.8.8
```
 
Both should succeed. The single magic word, **overload**, is what turns one shared address into capacity for thousands of hosts.
 
### Step — Save the Configuration
 
```
R1# copy running-config startup-config
```
 
---
 
## 👀 Part 2: Observe It
 
### 2.1 — Read the PAT Translation Table
 
This is the single most illuminating view in the entire lab.
 
```
R1# show ip nat translations
```
 
**What to look for:** Multiple inside hosts, 192.168.1.10 and 192.168.1.11, both translated to the same inside global address, 203.0.113.1, and distinguished only by port number. That shared address with differing ports is PAT in one picture. Two private hosts, one public address, kept apart by ports.
 
**✏️ Think it through:** Explain how a reply coming back to 203.0.113.1 finds its way to the correct inside host when both hosts share that one address. The answer is in the port column.
 
### 2.2 — Check the Statistics
 
```
R1# show ip nat statistics
```
 
This shows how many active translations exist, which interfaces are inside and outside, and how many translation attempts have hit or missed. It is your dashboard for the health of NAT on this router.
 
### 2.3 — Watch a Translation Build Live
 
```
R1# debug ip nat
```
 
With debugging on, ping 8.8.8.8 from a host and watch the translation get created and used in real time, then turn it off with `undebug all`. Seeing the inside local rewritten to the inside global as the packet leaves, and reversed as the reply returns, makes the whole mechanism concrete.
 
### 2.4 — Confirm From the Inside and the Outside
 
```
PC1> ping 8.8.8.8
PC2> ping 8.8.8.8
```
 
Both inside hosts reach the internet through one shared public address. That is the everyday reality of nearly every home and small office network in the world.
 
---
 
## 💥 Part 3: Break It
 
> Work through each scenario one at a time. Document the symptoms fully before you move to Part 4.
 
---
 
### Break Scenario 1 — The Missing Designation
 
This is the most common NAT failure there is. Remove the inside designation from R1's LAN interface:
 
```
R1(config)# interface GigabitEthernet0/0
R1(config-if)# no ip nat inside
R1(config-if)# exit
```
 
Now test:
```
PC1> ping 8.8.8.8
R1# show ip nat translations
```
 
**✏️ Document the symptoms:**
- Can either host reach the internet now?
- Does the translation table build any entries when the hosts send traffic?
- The NAT rule and the ACL are both still perfectly configured, so why does nothing translate?
> **Why this matters:** NAT only acts on traffic crossing from inside to outside, and with no inside interface, there is no such traffic, so NAT sits idle. The maddening part is that everything else looks correct. The pool, the ACL, the overload rule are all intact, and `show running-config` looks healthy. The fault is one missing line on an interface. Whenever NAT does nothing at all, check the inside and outside designations before anything else.
 
---
 
### Break Scenario 2 — The ACL That Does Not Match
 
Restore the inside designation. Then change the ACL so it no longer matches your actual inside hosts:
 
```
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip nat inside
R1(config-if)# exit
 
! Point the ACL at the wrong subnet
R1(config)# no access-list 1
R1(config)# access-list 1 permit 192.168.99.0 0.0.0.255
```
 
Now test:
```
PC1> ping 8.8.8.8
R1# show ip nat translations
```
 
**✏️ Document the symptoms:**
- Do the hosts get translated now?
- The interfaces are correctly marked inside and outside, so why does traffic still fail?
- What is the ACL actually controlling in a NAT configuration?
> **Why this matters:** The ACL in a NAT rule is the guest list. It decides which inside hosts are allowed to be translated, and a host that is not permitted simply is not translated, so its private address leaves unchanged and the reply never comes back. This is a quieter failure than the missing designation, because some hosts could work while others do not, depending on what the ACL permits. When specific hosts fail to get out, suspect the NAT ACL.
 
---
 
### Break Scenario 3 — The Forgotten Overload
 
Restore the correct ACL. Then reconfigure PAT but leave off the one word that makes it PAT:
 
```
R1(config)# no access-list 1
R1(config)# access-list 1 permit 192.168.1.0 0.0.0.255
 
! Remove the working overload rule
R1(config)# no ip nat inside source list 1 interface GigabitEthernet0/1 overload
 
! Re-add it WITHOUT the overload keyword
R1(config)# ip nat inside source list 1 interface GigabitEthernet0/1
```
 
Now ping the internet from both hosts at the same time:
```
PC1> ping 8.8.8.8
PC2> ping 8.8.8.8
R1# show ip nat translations
```
 
**✏️ Document the symptoms:**
- One host works and the other fails, much like the dynamic pool did earlier. Why?
- Without `overload`, how many hosts can share that single interface address at once?
- How is this failure related to the pool exhaustion you saw in Stage B?
> **Why this matters:** Without the `overload` keyword, the router treats the interface address as a one-address pool and does not use port numbers, so only a single host can use it at a time. The fix is the one word that turns plain NAT into port-based sharing. This drives home what PAT really is. It is not a different address strategy, it is the addition of port tracking to a single address, and that single word is what unlocks it.
 
---
 
## 🔧 Part 4: Fix It
 
---
 
### Fix Scenario 1 — The Missing Designation
 
**Your troubleshooting toolkit:**
```
show ip nat statistics                 ! shows which interfaces are inside and outside
show running-config interface GigabitEthernet0/0
```
 
**Structured process:**
1. Note that NAT is doing nothing at all and no translations are forming.
2. Check `show ip nat statistics` for the list of inside and outside interfaces.
3. If an interface is missing its designation, add it.
4. Verify translations begin to build.
**✅ Solution:**
```
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip nat inside
R1(config-if)# exit
```
 
**Verification:**
```
PC1> ping 8.8.8.8
R1# show ip nat translations
```
 
---
 
### Fix Scenario 2 — The ACL That Does Not Match
 
**Your troubleshooting toolkit:**
```
show access-lists
show running-config | include access-list
```
 
**Structured process:**
1. Note that the inside and outside interfaces are correct but hosts still are not translated.
2. Examine the ACL referenced by the NAT rule.
3. Confirm whether it actually matches your inside subnet.
4. Correct the ACL to permit the real inside network.
**✅ Solution:**
```
R1(config)# no access-list 1
R1(config)# access-list 1 permit 192.168.1.0 0.0.0.255
```
 
**Verification:**
```
PC1> ping 8.8.8.8
R1# show ip nat translations
```
 
---
 
### Fix Scenario 3 — The Forgotten Overload
 
**Your troubleshooting toolkit:**
```
show running-config | include ip nat inside source
show ip nat translations               ! a PAT table shows port numbers; plain NAT does not
```
 
**Structured process:**
1. Note that only one host can get out at a time.
2. Inspect the NAT source rule and check for the `overload` keyword.
3. If it is missing, replace the rule with one that includes it.
4. Verify both hosts can translate at once and the table now shows ports.
**✅ Solution:**
```
R1(config)# no ip nat inside source list 1 interface GigabitEthernet0/1
R1(config)# ip nat inside source list 1 interface GigabitEthernet0/1 overload
```
 
**Verification:**
```
PC1> ping 8.8.8.8
PC2> ping 8.8.8.8
R1# show ip nat translations
```
 
---
 
## 💬 Reflection Questions
 
1. Explain why a host with a 192.168 address cannot communicate directly across the public internet, and what NAT does to make that communication possible.
2. Define inside local and inside global in your own words. When PC1 pings 8.8.8.8 under PAT, which address is which, and what does the outside server believe it is talking to?
3. Compare static NAT, dynamic NAT, and PAT. For each, describe one situation where it is the right choice.
4. In Stage B, a one-address pool left the second host stranded. Explain why dynamic NAT has this limitation, and how PAT removes it without requiring any additional public addresses.
5. Under PAT, many inside hosts share a single public address. Explain the specific mechanism that keeps their separate conversations from getting mixed up, and where you saw evidence of it in the translation table.
6. In Break Scenario 1, NAT silently did nothing even though the pool, ACL, and overload rule were all correct. Why is the inside and outside interface designation the first thing to check when NAT appears completely inert?
---
 
## Challenge Extension (Optional)
 
1. **Static NAT for a server.** Reconfigure a static mapping so an inside server is always reachable at a fixed public address, then test reaching it from R2 toward that public address. This is how port forwarding and published servers work.
2. **A real pool, sized right.** Rebuild dynamic NAT with a pool of several addresses instead of one, and confirm multiple hosts translate simultaneously. Then reason about how many hosts a pool of five addresses can support before exhaustion versus how many PAT could support.
3. **Static PAT, also called port forwarding.** Research `ip nat inside source static tcp` and configure a rule that forwards a specific port on the public address to an inside host. Where would a home user use exactly this to host a game server?
4. **Read the ports closely.** Generate several simultaneous connections from one host and watch the translation table assign a different outside port to each. Explain how a single host with one private address ends up using many outside ports at once.
---
 
## Where This Leads
 
You have given a network of privately addressed hosts a way onto the public internet, and you did it three ways, ending with the one that actually runs the world. More than that, you watched dynamic NAT fail by design and then solved that failure with PAT, which is the clearest way to understand why overload is the default everywhere you look. The translation table, with its shared address and differing ports, is a picture worth keeping in mind, because it explains how billions of devices share a limited supply of public addresses.
 
The next service is the one that turns names into addresses. Right now you have been typing 8.8.8.8 to reach the internet, but no human remembers addresses for the sites they visit. DNS is the directory that translates a name like a website into the IP address underneath it, and it is the last of the core services in this section. With addressing and translation behind you, resolving names is the natural next step, and it is where the Services section goes next.
 
---
 
*CCNA – Build It · Observe It · Break It · Fix It series*

*Next: DNS*
 
*This repository is an independent educational resource and is not officially affiliated with Cisco Systems. Cisco, CCNA, Cisco Modeling Labs, and Cisco Networking Academy are trademarks of Cisco Systems, Inc.*
