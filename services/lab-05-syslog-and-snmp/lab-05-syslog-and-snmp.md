# Syslog and SNMP

**Build It · Observe It · Break It · Fix It**

---

## Overview

A network that you cannot see is a network you cannot run. When an interface flaps at three in the morning, or a device reboots, or a link quietly saturates, something has to record it and ideally tell someone. The two services in this lab are how that happens, and they approach the problem from opposite directions. **Syslog** is the network talking to you, every device generating a running stream of messages about what it is doing and what is going wrong. **SNMP** is you talking to the network, a monitoring station polling devices to ask how they are and receiving alerts when something crosses a threshold. Together they are the difference between operating a network and merely hoping it keeps working.

Syslog messages are not all equally urgent, and the system that sorts them is the **severity level**, a scale from 0 to 7 where lower means more serious. An emergency that makes a device unusable is level 0, while a routine informational note is level 6 and a debugging message is level 7. Knowing this scale is the heart of using Syslog well, because it lets you say "send me everything this serious or worse" and filter out the noise.

SNMP works through a manager and agents. The agents are your devices, each maintaining a structured database of information about itself, and the manager is the monitoring station that either polls the agents for data or receives unsolicited alerts, called traps, when something noteworthy happens. In this lab you will point two devices at a central server, configure them to ship their logs there, set up SNMP so they can be monitored, and then break the telemetry in the three ways it most commonly fails. And because you built NTP first, every log message will carry a timestamp that actually means something.

**CCNA Exam Alignment:**
- 4.4 – Explain the function of SNMP in network operations
- 4.5 – Describe the use of syslog features, including facilities and severity levels

**Estimated Time:** 75–105 minutes

---

## Prerequisites

Before completing this lab, you should be comfortable with:
- Basic IOS navigation, including moving between command modes and reading `show` command output
- IPv4 addressing and verifying connectivity with `ping`
- Synchronized time and log timestamps from the NTP lab, since a log is only as trustworthy as its timestamp
- CML fundamentals, including building or importing a topology and accessing a device console

---

## Key Concept: Severity Levels, and Polling Versus Traps

Two ideas unlock this entire lab. The first is the Syslog severity scale.

| Level | Name | Meaning |
|-------|------|---------|
| 0 | Emergency | System is unusable |
| 1 | Alert | Immediate action needed |
| 2 | Critical | Critical conditions |
| 3 | Error | Error conditions |
| 4 | Warning | Warning conditions |
| 5 | Notification | Normal but significant |
| 6 | Informational | Informational messages |
| 7 | Debugging | Debug-level messages |

A common mnemonic for the order is "Every Awesome Cisco Engineer Will Need Icecream Daily." The crucial and counterintuitive part is the direction. **Lower numbers are more severe.** When you tell a device `logging trap 4`, you are asking for level 4 and everything more serious, meaning levels 0 through 4, while levels 5, 6, and 7 are filtered out. Set that threshold too low and your server hears nothing but emergencies. Set it too high and you drown in debug noise. Choosing the right level is the skill.

The second idea is how SNMP moves information. There are two directions:

- **Polling** is the manager reaching out to ask. The monitoring station sends a request, the agent answers with the data, and this repeats on a schedule. It is steady and predictable, but the manager only learns about a problem the next time it happens to ask.
- **Traps** are the agent speaking up on its own. When something noteworthy happens, the agent immediately sends an alert to the manager without waiting to be asked. Traps deliver bad news fast, which is exactly when speed matters.

> **Get ahead of the confusion:** A device can be perfectly reachable by ping and still fail to deliver a single log message or answer a single SNMP poll. Reachability is not the same as telemetry. When the server is silent, do not assume the network is down. Check the severity threshold, the destination address, and the SNMP community before you suspect connectivity.

---

## Topology

```
                  [SRV1]        Syslog + SNMP server
               192.168.1.100    (the monitoring station)
                    |
                  [SW1]
                 /      \
             [R1]        [R2]    Monitored devices
          192.168.1.1  192.168.1.2
          (syslog source + SNMP agent)
```

> **Full topology file:** `topology.yaml`

**Nodes (4 total, within CML Free Edition limit):**
- R1, R2 (IOSv routers, the monitored devices, acting as syslog sources and SNMP agents)
- SW1 (IOSvL2 switch)
- SRV1 (a host acting as the syslog and SNMP server)

> **A note on the server:** Whether SRV1 actually records syslog messages or answers SNMP polls depends on the software in your host image, just as host DNS resolution did in an earlier lab. The rock-solid verification in this lab happens on the IOS devices themselves, through their local logging buffer and `show snmp` counters. If your server runs a real syslog daemon or SNMP manager, you will also see the data arrive there, which is the full experience. As always, confirm interface names with `show ip interface brief` and adjust.

---

## Addressing Table

| Device | Interface | IP Address     | Subnet Mask     | Role |
|--------|-----------|----------------|-----------------|------|
| R1     | G0/0      | 192.168.1.1    | 255.255.255.0   | Syslog source, SNMP agent |
| R2     | G0/0      | 192.168.1.2    | 255.255.255.0   | Syslog source, SNMP agent |
| SRV1   | NIC       | 192.168.1.100  | 255.255.255.0   | Syslog and SNMP server |

---

## Lab Objectives

By the end of this lab you will be able to:

1. Explain the function of Syslog and SNMP in network operations
2. Describe the Syslog severity levels and how the threshold filters messages
3. Configure devices to send timestamped logs to a central server
4. Configure SNMP agents with community strings and a trap destination
5. Verify logging and SNMP with the appropriate `show` commands
6. Identify and resolve three common monitoring failures
7. Apply a structured troubleshooting process to network telemetry

---

## 🧱 Part 1: Build It

### Step 1 — Configure Interfaces

Bring up R1 and R2 on the LAN, and give SRV1 its address. Confirm all three can ping each other before going further, since telemetry cannot travel a path that does not work.

```
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip address 192.168.1.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit
```

Repeat with the matching address on R2, and configure SRV1 with 192.168.1.100.

---

### Stage A — Syslog

### Step 2 — Timestamp Every Message

Before sending logs anywhere, make sure each one carries a meaningful time. This is where NTP pays off.

```
R1(config)# service timestamps log datetime msec
```

> **This single line is why NTP came first.** It stamps every log message with the date and time down to the millisecond. With synchronized clocks behind it, a message from R1 and a message from R2 about the same event will carry timestamps you can actually line up. Without NTP, this stamp would still appear, but the times would disagree and the correlation you need during an outage would be lost.

### Step 3 — Send Logs to the Server at the Right Severity

Tell the device where to send logs and how serious a message must be to qualify.

```
! Define the syslog server
R1(config)# logging host 192.168.1.100

! Send level 6 (informational) and everything more severe to the server
R1(config)# logging trap informational

! Keep a local copy in the buffer so you can always read recent history on the device
R1(config)# logging buffered 8192
```

> **`logging trap informational` means level 6 and below in number, which is 6 and above in severity.** You will receive informational messages, notices, warnings, errors, and everything up to emergencies. Only debugging, level 7, is filtered out. You will deliberately set this threshold wrong in Break Scenario 1 to see what filtering does.

Apply the same three commands on R2.

### Step 4 — Generate an Event to Log

Create a clean, harmless event by bouncing a loopback, which produces link state messages without disrupting anything real.

```
R1(config)# interface Loopback0
R1(config-if)# shutdown
R1(config-if)# no shutdown
R1(config-if)# exit
```

---

### Stage B — SNMP

### Step 5 — Configure the SNMP Agent

Set up R1 and R2 to be monitored. Start with identifying information, then a read-only community string, which acts as a password for read access.

```
! Helpful identifying details for whoever monitors this device
R1(config)# snmp-server location Athens-IDF-1
R1(config)# snmp-server contact netadmin@lab.local

! A read-only community string. Treat this like a password.
R1(config)# snmp-server community CCNA-RO ro
```

> **A community string is a password in plain clothes.** In SNMP version 2c, which this is, the community string travels across the network in clear text, so anyone watching can read it. That is a real security weakness, and the answer to it is SNMP version 3, which adds authentication and encryption. The CCNA expects you to know that distinction. We use 2c here because it is simpler to see working, but in production you would reach for version 3.

### Step 6 — Send Traps to the Server

Point the agent at the monitoring station and enable it to send traps.

```
! Where to send unsolicited alerts
R1(config)# snmp-server host 192.168.1.100 version 2c CCNA-RO

! Allow the device to generate traps
R1(config)# snmp-server enable traps
```

Apply the same SNMP configuration on R2, adjusting the location for that device.

### Step 7 — Save the Configurations

```
R1# copy running-config startup-config
R2# copy running-config startup-config
```

---

## 👀 Part 2: Observe It

### 2.1 — Read the Local Log Buffer

Every device keeps its own recent history, and this is the first place to look.

```
R1# show logging
```

**What to look for:** The configuration summary at the top, showing the logging host, the trap level, and the buffer, followed by the actual messages. You should see the loopback going down and back up, each line stamped with the date and time. Confirm the timestamps look correct, which tells you NTP and `service timestamps` are doing their jobs.

**✏️ Prediction checkpoint:** Before you look, predict whether the loopback messages will appear in the buffer, and at roughly what severity link state changes are logged.

### 2.2 — Confirm the Logging Destination

In that same `show logging` output, find the line describing the trap logging. It states the server address and the severity level being sent. This is your proof that the device knows where to ship logs and how much to send.

### 2.3 — Verify the SNMP Agent

```
R1# show snmp
```

This shows SNMP is enabled and reports counters for the packets it has handled. To confirm your community string and host specifically:

```
R1# show snmp community
R1# show snmp host
```

**What to look for:** The community string CCNA-RO marked read-only, and the trap host 192.168.1.100 configured for version 2c. These two views confirm both halves of SNMP, the read access and the trap destination.

### 2.4 — See the Telemetry Arrive, If Your Server Supports It

If SRV1 runs a syslog daemon, the loopback messages you generated will appear in its log. If it runs an SNMP manager, you can poll R1 and read its data, or watch a trap arrive. This is the full picture: two devices reporting to one central station.

**✏️ Think it through:** You generated the loopback event on R1 only. Explain the path that message took from the interface changing state to landing in the buffer and being sent toward the server.

---

## 💥 Part 3: Break It

> Work through each scenario one at a time. Document the symptoms fully before you move to Part 4.

---

### Break Scenario 1 — The Threshold Set Too Low

This is the most common Syslog mistake, and it teaches the severity scale better than any table. Set R1's trap level so restrictive that ordinary events no longer qualify:

```
R1(config)# logging trap errors
R1(config)# exit
```

> `errors` is level 3. Only levels 0 through 3 will now be sent to the server. The link state messages from a loopback bounce are less severe than that, so they will be filtered out.

Now generate an event and check both places:
```
R1(config)# interface Loopback0
R1(config-if)# shutdown
R1(config-if)# no shutdown
R1(config-if)# exit
R1# show logging
```

**✏️ Document the symptoms:**
- Do the loopback messages still appear in R1's local buffer?
- Would they have been sent to the server at this trap level? Why or why not?
- If the server operator complained they were "seeing nothing from R1," what would you check first?

> **Why this matters:** The device is healthy, the server is reachable, and the events are happening, yet nothing reaches the server because the threshold filters them out before they are ever sent. This is the trap that makes people chase phantom network problems. The local buffer still shows everything, which is the giveaway that the device is fine and the filtering is the issue. When a server hears less than it should, the trap level is the first suspect, and remember that a lower number means a stricter filter.

---

### Break Scenario 2 — The Community Mismatch

Restore R1's trap level. Then change the SNMP community string so it no longer matches what the monitoring station expects:

```
R1(config)# logging trap informational

! Remove the known community and set a different one
R1(config)# no snmp-server community CCNA-RO ro
R1(config)# snmp-server community WRONG-STRING ro
R1(config)# exit
```

Now consider what the monitoring station experiences. It is still configured to poll R1 using CCNA-RO, but R1 now only answers to WRONG-STRING.

```
R1# show snmp community
```

**✏️ Document the symptoms:**
- If the manager polls R1 with the community string CCNA-RO, will R1 respond?
- Is R1 reachable by ping during all of this?
- What role is the community string actually playing in this failure?

> **Why this matters:** The community string is SNMP's password, and a poll with the wrong string is simply ignored, exactly as a login with the wrong password would be. The device is reachable, SNMP is running, and monitoring still fails, because the manager and the agent no longer share a secret. This is the SNMP version of the authentication failures you have seen throughout this project. When polling fails but the device is up, compare the community string on the agent against the one the manager is using.

---

### Break Scenario 3 — The Telemetry That Goes Nowhere

Restore the correct community. Then point R1's logging at a server that is not there:

```
R1(config)# snmp-server community CCNA-RO ro

! Send logs to an address where nothing is listening
R1(config)# no logging host 192.168.1.100
R1(config)# logging host 192.168.1.200
R1(config)# exit
```

> 192.168.1.200 is not the server. Nothing there will ever receive a message.

Generate an event and look at both the device and, if available, the server:
```
R1(config)# interface Loopback0
R1(config-if)# shutdown
R1(config-if)# no shutdown
R1(config-if)# exit
R1# show logging
```

**✏️ Document the symptoms:**
- Does R1's local buffer still capture the event?
- Does anything arrive at the real server?
- The device is logging perfectly to itself, so why is the central record empty?

> **Why this matters:** Syslog over UDP is fire and forget. The device sends its messages toward the configured address and never checks whether anything received them, so a wrong destination fails completely silently. The local buffer looks perfect, which can fool you into thinking logging works, while the central server that you actually rely on records nothing. When logs are present on the device but absent on the server, the destination address is the thing to verify.

---

## 🔧 Part 4: Fix It

---

### Fix Scenario 1 — The Threshold Set Too Low

**Your troubleshooting toolkit:**
```
show logging                       ! the header shows the current trap level
```

**Structured process:**
1. Confirm the events exist in the local buffer but are not reaching the server.
2. Check the trap level in the `show logging` header.
3. Recognize that the level is too strict to pass the events you expect.
4. Set the threshold to an appropriate level and regenerate an event.

**✅ Solution:**
```
R1(config)# logging trap informational
```

**Verification:**
```
R1# show logging
```

---

### Fix Scenario 2 — The Community Mismatch

**Your troubleshooting toolkit:**
```
show snmp community                ! shows the configured community strings
```

**Structured process:**
1. Confirm the device is reachable but polling fails.
2. Compare the agent's community string against the one the manager uses.
3. Reconfigure the agent to use the agreed string.
4. Verify the community is correct.

**✅ Solution:**
```
R1(config)# no snmp-server community WRONG-STRING ro
R1(config)# snmp-server community CCNA-RO ro
```

**Verification:**
```
R1# show snmp community
```

---

### Fix Scenario 3 — The Telemetry That Goes Nowhere

**Your troubleshooting toolkit:**
```
show logging                       ! the header shows the configured host
```

**Structured process:**
1. Confirm logs are present locally but absent on the server.
2. Check the logging host address in the configuration.
3. Compare it against the real server address.
4. Correct the destination and regenerate an event.

**✅ Solution:**
```
R1(config)# no logging host 192.168.1.200
R1(config)# logging host 192.168.1.100
```

**Verification:**
```
R1# show logging
```

---

## 💬 Reflection Questions

1. Explain the different jobs Syslog and SNMP do. How is the network talking to you different from you talking to the network, and why do you want both?

2. Describe the Syslog severity scale. Why is it counterintuitive that level 0 is more serious than level 7, and what does setting `logging trap warning` actually send?

3. Compare SNMP polling and traps. Give one situation where polling is the right tool and one where a trap is, and explain the tradeoff in how quickly you learn about a problem.

4. In Break Scenario 1, the events were in the local buffer but never reached the server. Explain how a healthy device with a reachable server can still fail to deliver logs, and how the local buffer helps you diagnose it.

5. A community string in SNMP version 2c travels in clear text. Explain why that is a security weakness and what SNMP version 3 adds to address it.

6. This lab depended on the NTP lab that came before it. Explain specifically what `service timestamps log datetime` contributes, and why its value collapses if the device clocks are not synchronized.

---

## Challenge Extension (Optional)

1. **Tune the severity by destination.** Set the console to one severity level and the syslog server to another, so urgent messages reach the screen while a fuller record goes to the server. Research `logging console` and compare it to `logging trap`.

2. **Correlate across devices.** Generate the same kind of event on both R1 and R2 within a few seconds, then read both logs and confirm the timestamps line up. This is the payoff of NTP plus Syslog working together.

3. **Step up to SNMPv3.** Research configuring an SNMPv3 user with authentication and encryption, and explain what it protects that a version 2c community string does not.

4. **Read the buffer like an operator.** Fill the buffer with several different events, then use `show logging` to read the history and practice reconstructing the order of what happened purely from the timestamps.

---

## Instructor Notes

**Common Student Mistakes**
- Getting the severity direction backwards. Students reliably assume a higher number is more urgent, then set `logging trap` to a low level and wonder why the server hears almost nothing. Reinforce that level 0 is the emergency and level 7 is the chatter.
- Forgetting `service timestamps log datetime`, or configuring it without NTP, so the logs either lack a useful time or carry times that do not agree across devices.
- Treating a community string as harmless and reusing an obvious one, without registering that in version 2c it is a clear-text password. This is the moment to plant the seed for SNMPv3 and the Security section.
- Configuring `snmp-server host` for traps but forgetting `snmp-server enable traps`, so the destination is set yet nothing is ever sent.
- Concluding the network is broken when the local buffer is full and only the server is empty. The buffer being healthy is the clue that delivery, not the device, is the problem.

**Teaching Tips**
- Make the cause and effect visible. Bounce a loopback and immediately show the message appearing in `show logging`, then walk it outward to the server. Watching an event become a log entry in real time is what makes the abstraction concrete.
- Use the community string discussion as a bridge to security. Pause on the fact that 2c sends it in clear text, and let students sit with the implication before you mention version 3. It primes them for the access and hardening topics coming in the Security section.

---

## Where This Leads

You have given your network a voice and a way to be questioned. Your devices now report what they are doing to a central place, timestamped so the records actually line up, and they can be monitored and alerted on through SNMP. With this lab, the Services section is complete. You have built the services that configure clients automatically, connect them to the wider internet, resolve the names they use, synchronize their clocks, and now record and report what they all do. That is the full set of services that turn a collection of configured devices into a network someone can actually operate.

From here the project turns to protecting all of it. The Security section takes the network you have built and hardens it, starting with how you log into devices safely, moving through controlling who can reach what, and on to defending the access layer itself. You already met the first hint of it in this lab, when an SNMP community string turned out to be a clear-text password. Securing the management of your devices is where the project goes next, and it begins with replacing insecure access with SSH.

---

*CCNA – Build It · Observe It · Break It · Fix It series*

*Next: SSH (Security series)*

*This repository is an independent educational resource and is not officially affiliated with Cisco Systems. Cisco, CCNA, Cisco Modeling Labs, and Cisco Networking Academy are trademarks of Cisco Systems, Inc.*
