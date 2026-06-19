# Services

A switched and routed network can move packets, but on its own it does very little that a user would notice. The services in this section are what turn that working plumbing into a network people can actually use and that you can actually operate. They hand out addresses automatically, connect privately addressed hosts to the public internet, translate the names people type into the addresses machines need, keep every device's clock in agreement, and record and report what the whole network is doing.

I have organized these five labs into two natural halves. The first three are **user-facing services**, the ones that make the network effortless for the people on it. DHCP addresses their devices, NAT gets those devices online, and DNS lets them use names instead of numbers. The last two are **operator-facing services**, the ones that make the network manageable for you. NTP synchronizes time so your records mean something, and Syslog with SNMP give you the visibility to know what happened and to be told when something breaks. Work through them in order, because the later labs lean on the earlier ones in ways that are part of the lesson.

---

## Who This Section Is For

- **Students** who have finished the Routing and Switching sections and are ready to build the services that run on top of a working network
- **Instructors** looking for services labs that teach troubleshooting and real operational habits, not just configuration syntax
- Anyone who can route and switch but has never configured the services that make a network usable day to day

**Before you start, you should be comfortable with:**
- Building and navigating a topology in CML (see CML Foundations)
- Configuring router and switch interfaces and verifying connectivity
- IPv4 addressing, subnetting, and default gateways (the Routing section covers this thoroughly)

If addressing still feels shaky, spend more time in Routing first. Nearly every lab here depends on hosts and routers reaching each other before any service can do its job.

---

## The Learning Philosophy

Every lab in this section follows the same cycle:

> ### 🧱 Build It · 👀 Observe It · 💥 Break It · 🔧 Fix It

You build a working service, observe how it behaves, break it in the ways it actually breaks in the field, and troubleshoot your way back. Services reward this approach especially well, because so many service failures look nothing like a dead network. A device can be perfectly reachable by ping and still hand out the wrong gateway, resolve a name to nowhere, refuse to synchronize its clock, or fail to deliver a single log. Learning to separate "the network is down" from "the service is misconfigured" is the single most valuable skill this section teaches, and you only build it by creating those failures yourself.

---

## The Labs in This Section

### 📩 Lab 01 — DHCP
**`lab-01-dhcp`**

The service that automates the addressing you spent the entire Switching section typing by hand. DHCP hands out addresses, masks, gateways, and DNS servers on demand.

You will configure a router as a DHCP server, then configure a second router as a relay agent so a subnet a router away can be served by that same server. The break scenarios center on the relay, the wrong gateway that leaves a client with an address but no reach, and the forgotten exclusion that hands out the gateway's own IP.

**You'll learn:**
- The DORA exchange and why two of its messages are broadcasts
- Configuring a router as a DHCP server with pools and exclusions
- Configuring a DHCP relay agent with `ip helper-address`
- Verifying leases with `show ip dhcp binding`

**CCNA aligned:** ✅ Yes (4.3, 4.6)
**Nodes:** Within the 5-node limit (2 routers, 2 hosts)

---

### 🌐 Lab 02 — NAT and PAT
**`lab-02-nat`**

The service that lets privately addressed hosts reach the public internet. This lab walks the three forms of NAT as a progression and arrives at the one that runs the world.

You will configure static NAT, then dynamic NAT, then deliberately exhaust a one-address pool so that PAT arrives as the answer to a problem you just watched happen. The break scenarios cover the missing inside and outside designation, the ACL that excludes your hosts, and the forgotten `overload` keyword.

**You'll learn:**
- Why NAT is necessary and the inside local, inside global, outside local, and outside global terminology
- Configuring static NAT, dynamic NAT, and PAT in sequence
- Reading the translation table, including the port numbers that make PAT work
- Diagnosing the most common NAT failures

**CCNA aligned:** ✅ Yes (4.1)
**Nodes:** 5 (2 routers, 1 switch, 2 hosts)

---

### 🧭 Lab 03 — DNS
**`lab-03-dns`**

The directory that turns names into addresses. This is a hybrid lab, hands-on with real IOS commands but honest that a router is not a production DNS server.

You will turn one router into a simple name server using `ip host` entries, configure another device to resolve through it, and hand out the DNS server address through DHCP so a host learns its resolver automatically. The break scenarios include the famous hostname trap, where a mistyped command hangs the router trying to resolve your typo.

**You'll learn:**
- The role of DNS and the basic name resolution process
- Configuring a router as a simple DNS server and as a DNS client
- Handing out a DNS server through DHCP, tying the services together
- The hostname trap and how to harden against it

**CCNA aligned:** ✅ Yes (4.3)
**Nodes:** Within the 5-node limit (2 routers, 1 switch, 1 host)

---

### ⏱️ Lab 04 — NTP
**`lab-04-ntp`**

The service that keeps every device's clock in agreement, which sounds minor until you try to line up logs from devices whose times disagree.

You will make one router an authoritative time source, sync two more to it as clients, and watch their clocks fall into agreement. The break scenarios cover the wrong server address, the source that has no authority of its own, and the patience test that teaches NTP's deliberately slow, stable behavior.

**You'll learn:**
- Why synchronized time matters and what the stratum hierarchy means
- Configuring a router as an NTP master
- Configuring NTP clients in client and server mode
- Verifying synchronization with `show ntp status` and `show ntp associations`

**CCNA aligned:** ✅ Yes (4.2)
**Nodes:** Within the 5-node limit (3 routers, 1 switch)

---

### 📊 Lab 05 — Syslog and SNMP
**`lab-05-syslog-and-snmp`**

The two halves of network visibility. Syslog is the network telling you what happened, and SNMP is you asking it how it is doing. This lab closes the section and depends directly on the NTP lab before it.

You will send timestamped logs from two devices to a central server at a chosen severity, configure SNMP so those devices can be monitored, and break the telemetry in three ways. The break scenarios cover the severity threshold set too low, the SNMP community mismatch, and the logging destination that goes nowhere.

**You'll learn:**
- The function of Syslog and SNMP and how they complement each other
- The Syslog severity levels and how the threshold filters messages
- Configuring centralized logging with meaningful timestamps
- Configuring SNMP agents, community strings, and traps

**CCNA aligned:** ✅ Yes (4.4, 4.5)
**Nodes:** Within the 5-node limit (2 routers, 1 switch, 1 server)

---

## Recommended Path

```
        USER-FACING SERVICES                 OPERATOR-FACING SERVICES
   ┌─────────────────────────────┐      ┌──────────────────────────────┐

   Lab 01      Lab 02     Lab 03         Lab 04          Lab 05
   DHCP    →   NAT    →   DNS       →    NTP        →    Syslog & SNMP
     │          │          │              │                  │
   Address    Reach the  Resolve        Synchronize       Record and
   hosts      internet   names          the clocks        report it all
   automatically                                          (needs NTP first)
```

Follow the order. DNS builds on the DHCP lab, since it hands out the DNS server through DHCP, and Syslog depends on the NTP lab, because a log is only as trustworthy as its timestamp. Doing them out of order still works, but you lose the connective tissue that makes the section feel like one system.

---

## Where to Go Next

Once you finish Services, you have a network that is not only working but usable and observable. The remaining sections turn to protecting and extending it:

- **Security**, which hardens everything you have built, starting with SSH for safe device access, then Port Security and Access Control Lists
- **Automation**, which looks at how modern networks are managed programmatically

Security is the natural next step, and you already met its first hint here, in the moment an SNMP community string turned out to be a clear-text password.

---

## A Note Before You Begin

Services are where a lot of people stop reading the output closely, because the network already "works." I want you to do the opposite. A service that is misconfigured will sit quietly on a perfectly healthy network and fail in ways that look like everything and nothing. These labs teach you to tell the difference, by handing you a working service and then asking you to break it on purpose. A service you have broken and repaired is a service you truly understand.

**Build it. Observe it. Break it. Fix it.**

---

*Part of the CCNA – Build It · Observe It · Break It · Fix It lab series.*
*This repository is an independent educational resource and is not officially affiliated with Cisco Systems. Cisco, CCNA, Cisco Modeling Labs, and Cisco Networking Academy are trademarks of Cisco Systems, Inc.*
