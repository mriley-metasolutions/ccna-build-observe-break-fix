# CML Foundations
 
**Start here.** This section gets you from zero to a working Cisco Modeling Labs environment with your first network built, verified, and understood.
 
If you're new to CML, or new to hands-on networking entirely, these three labs lay the groundwork for everything else in this repository. Work through them in order. By the end, you'll have CML installed, you'll know your way around the interface, and you'll have built and troubleshot a real network running real Cisco IOS.
 
---
 
## Who This Section Is For
 
- **Students** beginning their CCNA journey who want a real lab environment, not just a simulator
- **Instructors** evaluating CML for their Cisco Networking Academy program
- **Career changers** entering IT who want hands-on practice from day one
- Anyone who has heard "you learn networking by doing" and is ready to actually do it
No prior experience with CML is assumed. The only thing you need is a computer that meets the system requirements (covered in Lab 01) and a free Cisco account.
 
---
 
## The Learning Philosophy
 
Every lab in this repository — starting with the foundations — follows a simple, deliberate cycle:
 
> ### 🧱 Build It · 👀 Observe It · 💥 Break It · 🔧 Fix It
 
You build a working network. You observe how it behaves. You intentionally break it. Then you troubleshoot and fix it.
 
Why break a network you just got working? Because that's where the real learning happens. Anyone can copy a configuration — but understanding *why* a network fails, recognizing the symptoms, and fixing it under pressure is what separates someone who memorized commands from someone who actually understands networking. It's also exactly how the modern CCNA exam tests you.
 
The foundations labs introduce this methodology gently. The installation lab skips it entirely (you can't break an install you're still doing), the navigation lab keeps it light, and the first-network lab gives you a single, beginner-friendly break/fix to get your feet wet.
 
---
 
## The Labs in This Section
 
### 🔧 Lab 01 — Getting Started with Cisco Modeling Labs
**`lab-01-cml-installation`**
 
Your "start here" resource. Before you can build anything, you need CML installed and running.
 
This lab explains what CML actually is, how network *emulation* differs from *simulation* (and why that matters for your CCNA prep), and walks you step by step through downloading, installing, and configuring CML Free — the no-cost edition Cisco makes available to anyone with a free account.
 
**You'll learn:**
- What CML is and why it belongs in a modern CCNA study plan
- How CML compares to Cisco Packet Tracer, and when to use each
- What CML Free includes (and its 5-node limit)
- System requirements and the platform gotchas to watch for (Apple Silicon, Hyper-V conflicts)
- A complete walkthrough: Cisco account → registration → download → VMware import → first login
**Prerequisites:** None. This is the true starting point.
**No topology file** — this is a setup guide.
 
---
 
### 🧭 Lab 02 — Navigating the CML Interface
**`lab-02-cml-navigation`**
 
Now that CML is installed, get comfortable driving it.
 
This lab is a guided tour of the CML web interface — the dashboard, the lab canvas, and the node console. You'll learn how to create a lab, drag nodes onto the canvas, wire them together, start and stop devices, and open a console session to type commands. Crucially, you'll also learn how to **import the `topology.yaml` files** included with every lab in this repository, so you can load a pre-built topology in seconds instead of wiring it by hand.
 
**You'll learn:**
- The CML dashboard and how to manage labs
- Building a topology on the canvas: adding nodes, drawing links
- Starting, stopping, and wiping nodes (and managing the 5-node limit)
- Opening and using the node console
- Importing and exporting labs as YAML files
**Prerequisites:** Lab 01 (CML installed and running).
**Light methodology** — mostly Build/Observe to get you oriented.
 
---
 
### 🌐 Lab 03 — Building Your First Network
**`lab-03-first-cml-network`**
 
Time to build something real. One router, one switch, two hosts — the classic first network.
 
This beginner-friendly lab takes you from an empty topology to a working LAN where two PCs communicate through a switch and reach a router serving as their gateway. Along the way you'll practice the IOS fundamentals every CCNA candidate needs: navigating command modes, configuring devices, assigning IP addressing, and verifying connectivity. It closes with a single, gentle break/fix scenario built around the most common beginner mistake in all of networking — the wrong default gateway.
 
**You'll learn:**
- Navigating IOS command modes (user, privileged, global config)
- Basic device configuration: hostnames, passwords, banners
- Configuring a router interface as a LAN gateway
- Configuring a switch management interface (SVI)
- Assigning IP addresses to end hosts
- Verifying Layer 2 and Layer 3 connectivity
- Recognizing and fixing a misconfigured default gateway
**Prerequisites:** Labs 01 and 02.
**Full methodology** — Build · Observe · Break · Fix (one beginner-friendly break/fix).
**Nodes:** 4 (R1, SW1, PC1, PC2) — within the CML Free 5-node limit.
 
---
 
## Recommended Path
 
```
Lab 01 (Install)  →  Lab 02 (Navigate)  →  Lab 03 (First Network)
       │                    │                       │
   Get CML up         Learn the tool          Build & break
   and running        confidently             a real network
```
 
Once you complete this section, you're ready to move into the topic-based lab series:
 
- **Routing** — static routing, RIPv2, default routing, single-area OSPF
- **Switching** — VLANs, trunking, inter-VLAN routing, STP
- **Services** — DHCP, NAT, DNS
- **Security** — SSH, ACLs
---
 
## A Note Before You Begin
 
Mistakes aren't failures here — they're the point. If a node won't boot, a link won't come up, or a ping won't go through, that's not a setback. That's the lab working exactly as intended. Every problem you troubleshoot in here is one you won't be stumped by on the exam or on the job.
 
Take your time. Be curious. Break things on purpose.
 
**Build it. Observe it. Break it. Fix it.**
 
---
 
*Part of the CCNA – Build It · Observe It · Break It · Fix It lab series.*
*This repository is an independent educational resource and is not officially affiliated with Cisco Systems. Cisco, CCNA, Cisco Modeling Labs, and Cisco Networking Academy are trademarks of Cisco Systems, Inc.*
