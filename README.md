# CCNA – Build It · Observe It · Break It · Fix It

Welcome! This repository contains hands-on CCNA study materials designed to help learners truly *understand* networking—not just memorize commands.

These materials are built around a simple teaching philosophy I’ve used for years in the Cisco Networking Academy classroom and beyond:

> **Build It · Observe It · Break It · Fix It**

Because the best way to learn networking is to *experience* it.

---

## Why This Repository Exists

Too many CCNA resources focus on:

* Memorizing syntax and commands
* Copying configurations
* Passing the exam without understanding network behavior

While those approaches may help short-term, they often fail students when it comes to:

* Troubleshooting and Problem Solving
* Explaining *why* something works, or doesn't work
* Adapting knowledge to real-world networks

In addition, there is currently a lack of resources using Cisco Modeling Labs Free edition.  

This project aims to bridge those gaps.

---

## The Learning Philosophy

Every concept in this repository follows the same intentional learning cycle:

### 🧱 Build It

You start by building a clean, working network with a clear purpose.

* Minimal configurations
* Clear intent behind every command
* Strong alignment with CCNA exam objectives

### 👀 Observe It

You then observe how the network behaves.

* Use `show` commands
* Test connectivity
* Watch protocol behavior and convergence
* Predict outcomes before verifying them

### 💥 Break It

Next, you intentionally break the network.

* Introduce common misconfigurations
* Unplug a cable
* Change one variable at a time
* Experience failure in a safe environment

This is where real learning happens.

### 🔧 Fix It

Finally, you troubleshoot and fix the issue.

* Identify the root cause using evidence
* Apply a structured troubleshooting process
* Validate that the network works again

This mirrors how networking works in the real world, and how CCNA questions are designed.

---

## Repository Structure

The repository is organized into topic sections that follow the natural learning order of the CCNA. Each section has its own README with a guided path through its labs.

### [01 · CML Foundations](./cml-foundations/)
Getting Cisco Modeling Labs Free running and building your first network.
* `lab-01-cml-installation`
* `lab-02-cml-navigation`
* `lab-03-first-network`

### [02 · Routing](./routing/)
Moving packets between networks, from static routes to OSPF.
* `lab-01-static-routing`
* `lab-02-rip-routing`
* `lab-03-default-routing`
* `lab-04-single-area-ospf`

### [03 · Switching](./switching/)
The Layer 2 world, ending in a full collapsed core campus build.
* `lab-01-vlans`
* `lab-02-trunking`
* `lab-03-inter-vlan-routing-router-on-a-stick`
* `lab-04-inter-vlan-routing-layer3-switch`
* `lab-05-etherchannel`
* `lab-06-stp`
* `lab-07-capstone`

### [04 · Services](./services/)
The services that make a network usable and operable.
* `lab-01-dhcp`
* `lab-02-nat`
* `lab-03-dns`
* `lab-04-ntp`
* `lab-05-syslog-and-snmp`

### [05 · Security](./security/)
Hardening the network in layers, from device access to the spoofing defenses at the edge.
* `concept-01-security-fundamentals` *(concept guide, read first)*
* `lab-01-ssh`
* `lab-02-port-security`
* `lab-03-standard-acls`
* `lab-04-extended-acls`
* `lab-05-dhcp-snooping-and-dai`

---

## What Every Lab Contains

Labs follow a consistent structure, so once you have done one, you know how to navigate them all. Each lab guide includes:

* **Overview** with CCNA exam objective alignment and an estimated time
* **Prerequisites** so you can confirm you are ready before you start
* **Key Concept** primer that explains the idea before you configure it
* **Topology diagram** and an **addressing table**
* **Lab Objectives**, the concrete skills you will walk away with
* The four phases: **🧱 Build It · 👀 Observe It · 💥 Break It · 🔧 Fix It**
* **Reflection Questions** to check your understanding
* **Challenge Extension** for going further
* **Instructor Notes** with common student mistakes and teaching tips

That last section is deliberately uncommon in public repositories. The Instructor Notes are written for the educator in the room, surfacing the mistakes students actually make and the tips that help concepts land. If you are teaching from these materials, that is where the classroom experience lives.

---

## Why Cisco Modeling Labs (CML)

This repository uses **Cisco Modeling Labs (CML) Free** as its primary lab platform.

While many CCNA resources rely on Packet Tracer, which is also a great tool, CML offers key advantages such as:

* Real Cisco IOS images and behavior
* Accurate routing protocol operation
* Realistic timing, convergence, and troubleshooting output
* A closer match to real networking equipment

CML allows learners to experience how networks *actually* behave, which aligns perfectly with the Build–Observe–Break–Fix philosophy.

> Some labs may be reproducible in Packet Tracer, but CML is the primary platform for accuracy and realism.

### A Note on the Five-Node Design Constraint

CML Free Edition allows up to five running nodes at once, and every lab in this repository is deliberately designed to fit within that limit. That constraint shaped many of the topologies in useful ways. Loopback interfaces stand in for additional networks, unmanaged switches extend a topology without counting against the limit, and extra links rather than extra devices are used where possible. The result is that every lab here runs on the free edition, with no licensing required to follow along.

---

## What You’ll Find in This Repository

* Topic-based folders aligned to the CCNA domains
* Step-by-step labs using the Build / Observe / Break / Fix model
* Full device configurations and topology diagrams for every lab
* Built-in troubleshooting practice through deliberate Break and Fix scenarios
* Instructor insights drawn from years of teaching experience

These materials are suitable for:

* CCNA students
* Cisco Networking Academy learners
* Career-changers entering IT
* Cisco Networking Academy Instructors looking for classroom-ready labs

---

## Tools Used

* Cisco Modeling Labs (CML) Free
* Cisco IOS-style CLI
* Real-world troubleshooting methodology

---

## Suggested Learning Path

If you are working through the whole CCNA, the sections are meant to be taken in order, because each one builds on the last:

1. **CML Foundations**, to get your lab environment running
2. **Routing**, the core logic of moving packets between networks
3. **Switching**, the Layer 2 world that sits beneath routing
4. **Services**, the features that make a network usable and manageable
5. **Security**, which hardens everything you have built

If you are studying a specific topic instead, jump straight to that section. Each one stands on its own, and any prerequisites are listed at the top of every lab.

---

## How to Use This Repository

1. Follow the study plan **or** jump to a topic you’re studying
2. Complete the labs in order:

   * Build
   * Observe
   * Break
   * Fix
3. Take notes on *why* the network behaves the way it does
4. Repeat labs and experiment, curiosity is encouraged

---

## Project Status

This repository has grown into a substantial, classroom-ready library. The following sections are complete, each with a full set of labs and a guided section README:

* ✅ **CML Foundations** — installation, navigation, and first network
* ✅ **Routing** — static, default, RIP, and single-area OSPF
* ✅ **Switching** — VLANs through a full collapsed core capstone
* ✅ **Services** — DHCP, NAT, DNS, NTP, Syslog, and SNMP
* ✅ **Security** — fundamentals, SSH and hardening, port security, ACLs, and Layer 2 spoofing defenses

The project continues to grow, and labs are refined over time as they are taught and tested.

---

## A Note on Learning

Mistakes are not failures. They’re an important part of the learning process.

If you’ve ever learned more from a broken network than a working one, you’re in the right place.

---

## Disclaimer

This project is an independent educational resource and is **not** officially affiliated with Cisco Systems. Cisco, CCNA, and Cisco Networking Academy are trademarks of Cisco Systems, Inc.

---

## Contributions & Feedback

Contributions, suggestions, and constructive feedback are welcome.
If something can be clearer, more realistic, or more helpful—let’s make it better.

---

**Build it. Observe it. Break it. Fix it.**
That’s how networks—and networking careers—are built.
