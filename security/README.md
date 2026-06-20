# Security

A network that moves traffic flawlessly but lets anyone read it, redirect it, or shut it down is not finished. It is a liability. This section takes the working, usable network you built across Routing, Switching, and Services, and hardens it. The work happens in layers, the way real security always does, because no single control is enough on its own. You secure how administrators log in, you control which devices may use the network at all, you filter what traffic is allowed to go where, and you defend the access edge against the spoofing attacks that slip beneath the other controls.

That layering is not an accident of ordering, it is the whole idea. The first concept guide gives you the vocabulary and principles, threats and vulnerabilities, least privilege, and defense in depth, and every lab after it is one concrete layer of that defense. Read the fundamentals first, then build the layers in order, and the section adds up to a genuine security posture rather than a pile of disconnected features.

---

## Who This Section Is For

- **Students** who have finished the earlier sections and are ready to protect the network they built
- **Instructors** looking for security labs that teach the reasoning behind each control, not just the commands, and that include the troubleshooting students actually face
- Anyone who can build a working network but has never hardened one

**Before you start, you should be comfortable with:**
- Configuring routers and switches, including VLANs, access ports, and basic routing
- IPv4 addressing and subnetting
- The Services section, especially DHCP, since two of these labs build directly on it
- CML fundamentals, including building or importing a topology and accessing a device console

If DHCP is not fresh, revisit it before the DHCP Snooping lab, which assumes it.

---

## The Learning Philosophy

Most of this section follows the same cycle as the rest of the project:

> ### 🧱 Build It · 👀 Observe It · 💥 Break It · 🔧 Fix It

You build a control, observe it working, break it the way it actually breaks in the field, and troubleshoot your way back. Security rewards this approach as much as any topic, because so many security failures are quiet. A feature that looks configured but is silently inactive, a control that blocks the very traffic it was meant to protect, a rule that is bypassed by the line above it. You learn to find those by creating them yourself. The one exception to the format is the opening guide, which is conceptual by design, since its subject is the vocabulary and principles that the labs put into practice.

---

## The Material in This Section

### 🛡️ Security Fundamentals (Concept Guide)
**`concept-01-security-fundamentals`**

The front door to the unit, and the only piece here that is not a hands-on lab. It frames why security belongs in a networking course and gives you the vocabulary the exam tests and the labs depend on.

**Covers:**
- The four words everyone confuses: threats, vulnerabilities, exploits, and mitigations
- The major threat categories, from malware and social engineering to spoofing and reconnaissance
- The defense principles: the CIA triad, defense in depth, least privilege, and AAA
- The human side: user awareness, training, and physical access control

**CCNA aligned:** ✅ Yes (5.1, 5.2)
**Format:** Concept guide, read this first

---

### 🔐 Lab 01 — Secure Device Access: SSH and Hardening
**`lab-01-ssh`**

The first concrete mitigation. It closes the very first vulnerability the fundamentals guide names, clear-text management access, and folds in the password and hardening suite that protects the credentials behind it.

**You'll learn:**
- Why SSH replaces Telnet, and configuring it including the RSA key and VTY lines
- Strong privileged access with `enable secret` and a local user account
- Password and session hardening, and the difference between weak Type 7 and strong hashing
- A device hardening quick reference for the exam and for real devices

**CCNA aligned:** ✅ Yes (4.8, 5.3)
**Nodes:** 3 (a router being secured, a workstation, a switch)

---

### 🔌 Lab 02 — Port Security
**`lab-02-port-security`**

The access layer is its own frontier. Port security controls which devices, and how many, may use a switch port at all.

**You'll learn:**
- What port security defends against, including unauthorized devices and MAC flooding
- Maximum MAC counts and static, dynamic, and sticky learning
- The three violation modes, protect, restrict, and shutdown, and their tradeoffs
- Recovering an err-disabled port and automating recovery

**CCNA aligned:** ✅ Yes (5.7)
**Nodes:** 3 counting toward the limit (a switch, two hosts, plus an unmanaged hub)

---

### 📋 Lab 03 — Standard ACLs
**`lab-03-standard-acls`**

The foundational logic of traffic filtering. Standard ACLs match only the source address, which makes them the cleanest place to learn the rules that govern every ACL.

**You'll learn:**
- Wildcard masks for matching hosts and subnets
- The implicit deny, top-down first-match order, and applying an ACL with a direction
- Why standard ACLs go close to the destination
- Verifying with `show access-lists` and its hit counters

**CCNA aligned:** ✅ Yes (5.6)
**Nodes:** 4 (a router, two source hosts, a protected server)

---

### 🎯 Lab 04 — Extended ACLs
**`lab-04-extended-acls`**

The same logic with far more reach. Extended ACLs match on protocol, source, destination, and port, which lets you write precise least-privilege policy.

**You'll learn:**
- Matching on protocol and port, and recognizing the common service ports
- Building a policy that permits one service to a host while denying everything else
- Why extended ACLs flip the placement rule and go close to the source
- Troubleshooting port, order, and direction mistakes

**CCNA aligned:** ✅ Yes (5.6)
**Nodes:** 4 (a router, a workstation, two servers)

---

### 🕵️ Lab 05 — DHCP Snooping and Dynamic ARP Inspection
**`lab-05-dhcp-snooping-and-dai`**

The defense ACLs cannot provide. These paired Layer 2 features stop the spoofing attacks that forge the very addresses an ACL trusts.

**You'll learn:**
- The rogue DHCP and ARP spoofing attacks, and why ACLs cannot stop them
- The trust boundary, and how DHCP snooping builds a binding table
- How Dynamic ARP Inspection uses that table to drop forged ARP
- Accounting for static hosts with an ARP ACL

**CCNA aligned:** ✅ Yes (5.7)
**Nodes:** 4 (a switch, a DHCP server and gateway, a client, an attacker)

---

## Recommended Path

```
Concept 01      Lab 01       Lab 02        Lab 03       Lab 04        Lab 05
Fundamentals →  SSH      →   Port      →   Standard →   Extended  →   DHCP Snooping
(read first)    Hardening    Security     ACLs         ACLs          & DAI
   │              │            │            │             │              │
 Vocabulary    Secure the   Control      Filter        Filter        Defend the
 and           management   who can      traffic by    by service    edge against
 principles    plane        connect      source        and port      spoofing
```

Read the fundamentals guide before anything else, since every lab assumes its vocabulary. Do the two ACL labs back to back, because the extended lab builds directly on the standard one and the pairing is where the understanding lives. Save DHCP Snooping and DAI for last, since it draws on both the DHCP lab from Services and the trust-boundary mindset from Port Security.

---

## A Note on the Describe-Level Topics

The CCNA security domain also covers a few topics at the describe level that are not built as labs here: the AAA framework with RADIUS and TACACS+, VPN and IPsec types, and wireless security with WPA2 and WPA3. These do not lend themselves to a five-node hands-on topology, and the exam tests them as concepts rather than configuration. The Security Fundamentals concept guide names AAA and points to where these fit, and they are worth reviewing from the official exam topics as part of your study. The hands-on labs in this section deliberately focus on the security features you actually configure and verify.

---

## Where to Go Next

With Security complete, you have taken a working network and made it defensible across multiple layers. From here the project turns to how modern networks are managed and scaled:

- **Automation**, which covers controller-based networking, REST APIs, data formats, and the configuration management tools that are reshaping the field

Automation is largely a describe-level domain on the exam, with a hands-on thread for those who want to see it in action, and it is the final section of the project.

---

## A Note Before You Begin

Security is where it is tempting to believe that configuring a feature means you are protected. This section is built to teach the opposite. A control you have not tested, broken, and watched fail is a control you do not really understand, and a security feature that is silently misconfigured is worse than none, because it offers false confidence. Every lab here hands you a working defense and then asks you to break it, so that you learn not just how to turn a control on, but how to know it is genuinely doing its job.

**Build it. Observe it. Break it. Fix it.**

---

*Part of the CCNA – Build It · Observe It · Break It · Fix It lab series.*
*This repository is an independent educational resource and is not officially affiliated with Cisco Systems. Cisco, CCNA, Cisco Modeling Labs, and Cisco Networking Academy are trademarks of Cisco Systems, Inc.*
