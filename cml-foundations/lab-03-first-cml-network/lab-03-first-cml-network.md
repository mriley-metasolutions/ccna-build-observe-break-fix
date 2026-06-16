# Building Your First Network in CML
 
**CCNA – Build It · Observe It · Break It · Fix It**
 
---
 
> ## 🎯 Foundations Lab
>
> This is a **beginner-friendly** lab designed for learners who have completed CCNA 1 (Introduction to Networks) or equivalent. It assumes you can navigate the CML interface (see the CML Navigation lab) but does **not** assume prior hands-on configuration experience.
>
> Take your time. The goal here isn't speed. The goal is building comfort with the IOS command line and understanding what each piece of a small network does.
 
---
 
## Overview
 
Every network engineer's journey starts with a single, simple network: one router, one switch, and a couple of hosts that can talk to each other. It sounds basic, but this small topology contains nearly every foundational concept you'll build on for the rest of your CCNA journey — device configuration, IP addressing, default gateways, switching, and verification.
 
In this lab you'll build that first network from scratch using Cisco Modeling Labs, observe how the pieces work together, then make one common beginner mistake on purpose and fix it — so that when you make it for real later (and you will), you'll recognize it instantly.
 
**Concepts Covered:**
- Navigating IOS command modes (user, privileged, global config)
- Basic device configuration (hostnames, passwords, banners)
- Configuring a router interface as a default gateway
- Basic switch configuration and the role of a Layer 2 switch
- Configuring end hosts with IP addresses
- Verifying connectivity and reading `show` command output
- Saving configurations
**Estimated Time:** 45–75 minutes
 
---
 
## What You'll Build
 
A single LAN where two PCs can communicate with each other through a switch, and reach a router that serves as their gateway to the rest of the network. To give the router a meaningful destination, we'll add a loopback interface representing "the rest of the network" — so you can see routing actually happen.
 
```
                  [R1]
            G0/0: 192.168.1.1
         Lo0: 8.8.8.8 (simulated "internet")
                   |
                   |  (router-to-switch link)
                   |
                 [SW1]
              Layer 2 switch
                /        \
               /          \
            [PC1]        [PC2]
        192.168.1.10   192.168.1.20
```
 
**Nodes (4 total — within CML Free Edition limit):**
- R1 (IOSv router)
- SW1 (IOSvL2 switch)
- PC1, PC2 (desktop hosts)
---
 
## Addressing Table
 
| Device | Interface  | IP Address    | Subnet Mask     | Default Gateway |
|--------|------------|---------------|-----------------|-----------------|
| R1     | G0/0       | 192.168.1.1   | 255.255.255.0   | —               |
| R1     | Loopback0  | 8.8.8.8       | 255.255.255.255 | —               |
| SW1    | VLAN 1     | 192.168.1.2   | 255.255.255.0   | 192.168.1.1     |
| PC1    | NIC        | 192.168.1.10  | 255.255.255.0   | 192.168.1.1     |
| PC2    | NIC        | 192.168.1.20  | 255.255.255.0   | 192.168.1.1     |
 
> **Why does the switch have an IP address?** A Layer 2 switch doesn't *need* an IP to do its job of switching frames — but assigning one to its management VLAN (VLAN 1 here) lets you remotely manage it (ping it, SSH to it). This is called a Switch Virtual Interface (SVI). It's optional for basic switching but standard practice in real networks.
 
> **Why is the loopback 8.8.8.8/32?** That's Google's public DNS address — we're using it purely as a recognizable stand-in for "somewhere out on the internet." It gives the PCs a remote destination to reach *through* the router, demonstrating the router's role. The /32 mask means it's a single host address.
 
---
 
## Lab Objectives
 
By the end of this lab you will be able to:
 
1. Move confidently between IOS command modes
2. Apply basic configuration to a router and a switch (hostname, passwords, banner)
3. Configure a router interface to serve as a LAN default gateway
4. Configure a switch management interface (SVI)
5. Assign IP addressing to end hosts
6. Verify Layer 2 and Layer 3 connectivity using `ping` and `show` commands
7. Recognize and fix a misconfigured default gateway
8. Save a running configuration so it survives a reboot
---
 
## Primer: IOS Command Modes
 
Before you type anything, understand where you're typing it. IOS has several modes, and the prompt tells you which one you're in:
 
| Prompt | Mode | What You Can Do |
|--------|------|-----------------|
| `Router>` | User EXEC | Basic monitoring only — limited commands |
| `Router#` | Privileged EXEC | Full monitoring, `show` commands, save config |
| `Router(config)#` | Global Config | Change device-wide settings |
| `Router(config-if)#` | Interface Config | Configure a specific interface |
| `Router(config-router)#` | Router Config | Configure a routing protocol |
 
**How to move between them:**
```
Router> enable                  ! user EXEC → privileged EXEC
Router# configure terminal      ! privileged EXEC → global config
Router(config)# interface g0/0  ! global config → interface config
Router(config-if)# exit         ! back up one level
Router(config)# end             ! jump all the way back to privileged EXEC
```
 
> **Tip:** If you ever get lost, press `Ctrl+Z` or type `end` to jump straight back to privileged EXEC mode. And `?` typed anywhere shows you the available commands — your single most useful tool while learning.
 
---
 
## 🧱 Part 1: Build It
 
### Step 1 — Basic Router Configuration
 
Open R1's console in CML and work through these foundational settings.
 
```
Router> enable
Router# configure terminal
 
! Give the device a meaningful name
Router(config)# hostname R1
 
! Set a password for privileged EXEC mode (encrypted)
R1(config)# enable secret cisco123
 
! Encrypt all plaintext passwords in the config
R1(config)# service password-encryption
 
! Configure a login banner — good security practice
R1(config)# banner motd #Authorized Access Only#
 
! Secure the console line
R1(config)# line console 0
R1(config-line)# password consolepw
R1(config-line)# login
R1(config-line)# exit
```
 
> **`enable secret` vs `enable password`:** Always use `enable secret` — it stores the password as a strong hash. The older `enable password` stores it weakly and should not be used. This distinction shows up on the exam.
 
### Step 2 — Configure the Router's LAN Interface
 
This is the interface that will serve as the default gateway for PC1 and PC2.
 
```
R1(config)# interface GigabitEthernet0/0
R1(config-if)# description LAN Gateway - PC Segment
R1(config-if)# ip address 192.168.1.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit
```
 
> **Don't forget `no shutdown`.** Router interfaces are administratively down by default (switches are up by default). Forgetting `no shutdown` is one of the most common beginner mistakes — the config looks perfect, but the interface is off. We'll revisit this idea in the Break It section.
 
### Step 3 — Configure the Loopback (Simulated Internet)
 
```
R1(config)# interface Loopback0
R1(config-if)# description Simulated Internet Destination
R1(config-if)# ip address 8.8.8.8 255.255.255.255
R1(config-if)# exit
```
 
> Loopbacks come up automatically. No `no shutdown` command is needed.
 
### Step 4 — Save the Router Configuration
 
```
R1(config)# end
R1# copy running-config startup-config
```
 
Press Enter to confirm. Your running config is now saved to startup config, meaning it will survive a reboot.
 
---
 
### Step 5 — Basic Switch Configuration
 
Open SW1's console. The switch configuration starts the same way as the router.
 
```
Switch> enable
Switch# configure terminal
Switch(config)# hostname SW1
SW1(config)# enable secret cisco123
SW1(config)# service password-encryption
SW1(config)# banner motd #Authorized Access Only#
```
 
### Step 6 — Configure the Switch Management Interface (SVI)
 
This gives the switch an IP address for remote management. On a switch, you assign the IP to a VLAN interface, not a physical port.
 
```
! Configure the management SVI on VLAN 1
SW1(config)# interface vlan 1
SW1(config-if)# ip address 192.168.1.2 255.255.255.0
SW1(config-if)# no shutdown
SW1(config-if)# exit
 
! Tell the switch how to reach networks beyond its own subnet
SW1(config)# ip default-gateway 192.168.1.1
SW1(config)# exit
```
 
> **Why does a switch need a default gateway?** The switch itself only needs this to send *its own* management traffic (like replying to a ping from a remote network) off the local subnet. It does NOT affect how the switch forwards user frames — that's pure Layer 2 switching and needs no IP at all. This is a subtle but important distinction.
 
### Step 7 — Save the Switch Configuration
 
```
SW1(config)# end
SW1# copy running-config startup-config
```
 
---
 
### Step 8 — Configure the End Hosts
 
In CML, open the console for each PC. The exact method depends on the host image (Desktop, Alpine, etc.). For a typical Linux-based host using the CML Desktop or Alpine node, configure the IP like this:
 
**PC1:**
```
ip 192.168.1.10 255.255.255.0 192.168.1.1
```
 
**PC2:**
```
ip 192.168.1.20 255.255.255.0 192.168.1.1
```
 
> **The default gateway is the key field.** Notice the third value — `192.168.1.1` — is R1's LAN interface. This tells each PC: "for any destination outside my own subnet, send the traffic here." Remember this field; it's the star of our Break It section.
 
---
 
## 👀 Part 2: Observe It
 
Now let's verify everything works and, more importantly, understand *why* it works.
 
### 2.1 — Verify Interface Status on the Router
 
```
R1# show ip interface brief
```
 
**What to look for:**
- G0/0 should show an IP of 192.168.1.1, status `up`, protocol `up`
- Loopback0 should show 8.8.8.8, `up`/`up`
- "up/up" means the interface is administratively enabled (no shutdown) AND physically/logically operational
**✏️ Prediction checkpoint:** Before running this, what status do you expect for any interface you didn't configure? Write it down, then check.
 
### 2.2 — Verify the Switch's MAC Address Table
 
This is the heart of how a switch works. After the PCs have sent some traffic (try a ping first), run:
 
```
SW1# show mac address-table
```
 
**What to look for:**
- Entries mapping each PC's MAC address to the switch port it's connected to
- This table is how the switch knows which port to send frames to — it *learns* MACs by watching incoming traffic
> **This is switching in action.** The switch built this table automatically by observing the source MAC of frames arriving on each port. When PC1 sends a frame to PC2, the switch looks up PC2's MAC in this table and forwards the frame out only that one port — not to everyone.
 
### 2.3 — Test Local (Same-Subnet) Connectivity
 
From PC1, ping PC2:
```
PC1> ping 192.168.1.20
```
 
**✏️ Think it through:** This traffic goes PC1 → SW1 → PC2. Does it ever touch the router? Why or why not? (Hint: are PC1 and PC2 on the same subnet?)
 
### 2.4 — Test Gateway Connectivity
 
From PC1, ping the router's LAN interface:
```
PC1> ping 192.168.1.1
```
 
This confirms PC1 can reach its default gateway — essential for reaching anything beyond the local subnet.
 
### 2.5 — Test Remote (Routed) Connectivity
 
From PC1, ping the simulated internet destination:
```
PC1> ping 8.8.8.8
```
 
**✏️ Think it through:** This is the big one. The traffic goes PC1 → SW1 → R1 → Loopback0. This is the first time a packet leaves the local subnet and gets *routed*. PC1 sends it to its default gateway (R1) because 8.8.8.8 is not on PC1's local subnet. Trace that logic in your head — it's the foundation of all routing.
 
### 2.6 — Trace the Path
 
```
PC1> traceroute 8.8.8.8
```
 
You should see R1 (192.168.1.1) as the first and only hop before reaching 8.8.8.8.
 
---
 
## 💥 Part 3: Break It
 
> One scenario for this foundational lab — the single most common beginner networking mistake.
 
---
 
### Break Scenario — The Wrong Default Gateway
 
On **PC2**, reconfigure it with an incorrect default gateway (a typo that points to a non-existent address):
 
```
ip 192.168.1.20 255.255.255.0 192.168.1.99
```
 
> `192.168.1.99` is not the router. It doesn't exist on this network.
 
**Now run these tests from PC2 and document what happens:**
 
```
! Test 1 — can PC2 still reach PC1 (same subnet)?
PC2> ping 192.168.1.10
 
! Test 2 — can PC2 reach the router (same subnet)?
PC2> ping 192.168.1.1
 
! Test 3 — can PC2 reach the simulated internet (different subnet)?
PC2> ping 8.8.8.8
```
 
**✏️ Document the symptoms:**
- Which of the three tests succeed, and which fail?
- Why does same-subnet communication still work while remote communication fails?
- What does this tell you about *when* the default gateway actually gets used?
> **Why this matters:** This is THE classic beginner symptom — "I can reach local devices but I can't get to the internet." It almost always points to a default gateway problem. The reason local traffic still works is that the default gateway is only consulted for destinations *outside* the local subnet. Same-subnet traffic is delivered directly via the switch and never needs the gateway. Recognizing this pattern instantly is a hallmark of someone who understands networking fundamentals.
 
---
 
## 🔧 Part 4: Fix It
 
**Your troubleshooting toolkit:**
- The three ping tests above
- Knowledge of which IP *should* be the gateway (check the addressing table)
**Structured process:**
1. Note the symptom: local works, remote fails
2. Recognize this signature points to a default gateway issue
3. Check PC2's configured gateway against the correct value (192.168.1.1)
4. Correct it and re-test all three scenarios
**✅ Solution:**
 
Reconfigure PC2 with the correct default gateway:
```
ip 192.168.1.20 255.255.255.0 192.168.1.1
```
 
**Verification — all three should now succeed:**
```
PC2> ping 192.168.1.10
PC2> ping 192.168.1.1
PC2> ping 8.8.8.8
```
 
> **Lock in the lesson:** The default gateway is only used for traffic leaving the local subnet. When "local works but remote doesn't," suspect the gateway first. You just experienced the most common help-desk ticket in all of networking.
 
---
 
## 💬 Reflection Questions
 
1. When PC1 pings PC2, the traffic never touches the router. When PC1 pings 8.8.8.8, it must go through the router. Explain in your own words what determines whether a packet stays local or gets sent to the default gateway.
2. The switch had no IP address assigned to any of the ports the PCs connect to, yet the PCs could still communicate through it. How is this possible? What does this tell you about the difference between Layer 2 switching and Layer 3 routing?
3. You configured the switch with an IP address on VLAN 1 and a default gateway. If you removed both, would the PCs still be able to ping each other? Would you still be able to manage the switch remotely? Explain.
4. In the Break It scenario, PC2 could still reach 192.168.1.1 (the router) even with a wrong default gateway configured. Why doesn't the wrong gateway setting prevent PC2 from reaching the router?
5. Router interfaces are shut down by default and require `no shutdown`, but loopback interfaces come up automatically. Why do you think IOS treats these two interface types differently?
6. You saved your configuration with `copy running-config startup-config`. What is the difference between the running config and the startup config? What happens to unsaved changes if the device reboots?
---
 
## Challenge Extension (Optional)
 
1. **Add SSH access.** Configure SSH on R1 so you can remotely manage it (you'll do this fully in the SSH lab later). For now, research the commands needed: a hostname, a domain name, an RSA key (`crypto key generate rsa`), a local user, and `transport input ssh` on the VTY lines.
2. **Break the interface, not the gateway.** Shut down R1's G0/0 interface (`shutdown`). Now try all three pings from PC1. How do the symptoms differ from the wrong-gateway scenario? What does `show ip interface brief` reveal?
3. **Add a third host.** If you want to push the node count, temporarily add a PC3 on the same subnet (192.168.1.30). Verify it can reach everyone. Then check the switch's MAC address table again — how many entries are there now?
4. **Examine ARP.** On PC1, after pinging around, run `arp -a` (or the host's equivalent). You'll see IP-to-MAC mappings the PC learned. Cross-reference these with the switch's MAC table. How do ARP (Layer 3-to-2 resolution) and switching (Layer 2 forwarding) work together?
---
 
*CCNA – Build It · Observe It · Break It · Fix It series*
*Foundations Lab — beginner friendly*
*Next: Static Routing (Routing series)*
 
