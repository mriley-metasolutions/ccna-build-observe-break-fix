# NTP

**Build It · Observe It · Break It · Fix It**

---

## Overview

Every device on your network keeps its own clock, and left alone, those clocks drift. One router runs a few seconds fast, another a few seconds slow, a third was never set at all and thinks it is still the year it shipped. None of that matters until something goes wrong and you need to piece together what happened. Then it matters enormously, because the log entry on the switch and the log entry on the router that explain the same outage carry timestamps that do not line up, and you cannot tell which event came first. Synchronized time is the quiet foundation that makes logs, certificates, scheduled jobs, and security all trustworthy.

The Network Time Protocol solves this by letting devices agree on the time. One device serves as an authoritative source, the others sync to it, and they keep syncing continuously so they never drift apart. NTP organizes the world of clocks into a hierarchy measured in **stratum** levels. Stratum 0 is a true reference clock, an atomic or GPS source. A server directly attached to one of those is stratum 1, a device that syncs to a stratum 1 server is stratum 2, and so on down the line, with each step adding one. The higher the stratum number, the further you are from the original source. A device that has not synchronized to anything sits at stratum 16, which is NTP's way of saying its clock cannot be trusted.

In this lab you will make one router an authoritative time source, sync two more routers to it as clients, and watch their clocks fall into agreement. Then you will break time sync in the three ways it most often fails. I will warn you now about the one thing that catches everyone, including me the first few times: NTP is deliberately slow. Synchronization can take several minutes, so patience is part of the skill here.

**CCNA Exam Alignment:**
- 4.2 – Configure and verify NTP operating in a client and server mode

**Estimated Time:** 75–105 minutes, including waiting for synchronization

---

## Prerequisites

Before completing this lab, you should be comfortable with:
- Basic IOS navigation, including moving between command modes and reading `show` command output
- IPv4 addressing and verifying connectivity with `ping`
- The idea that services run over specific ports, since NTP uses UDP port 123
- CML fundamentals, including building or importing a topology and accessing a device console

---

## Key Concept: Stratum, and the Difference Between a Source and a Relay

NTP's whole model rests on the stratum hierarchy, and understanding it explains almost everything you will see in this lab.

| Stratum | What it is |
|---------|------------|
| 0 | A true reference clock, such as atomic or GPS. Not a network device. |
| 1 | A server directly attached to a stratum 0 source. The most authoritative network time you can get. |
| 2, 3, 4... | Each device that syncs to the level above it, adding one to its stratum. |
| 16 | Unsynchronized. The device's clock is not trusted by NTP. |

The key insight is that a device only has authoritative time if it is itself synchronized to a valid source. A router configured as an NTP server is not handing out trustworthy time unless its own clock is anchored to something. In a lab with no internet, you solve this with the `ntp master` command, which tells a router to treat its own internal clock as the reference and serve it at a stratum you choose. That is a small fiction, since the router's clock is not really atomic, but it gives the lab a stable source to sync to.

> **Get ahead of the confusion:** A client points at a server with `ntp server`, but if that server is not itself synchronized, the client cannot sync either. You will break the network exactly this way, by taking the authority away from the source while leaving the clients perfectly configured, and the lesson is that a server you cannot trust is no server at all.

---

## Topology

```
                    [R1]              NTP MASTER (stratum 2)
                 192.168.1.1          ntp master 2
                      |
                    [SW1]
                   /      \
               [R2]        [R3]       NTP CLIENTS (become stratum 3)
            192.168.1.2  192.168.1.3
            ntp server 192.168.1.1
```

**Nodes (4 total, within CML Free Edition limit):**
- R1 (IOSv router, the authoritative NTP master)
- R2, R3 (IOSv routers, NTP clients)
- SW1 (IOSvL2 switch connecting the three routers)

> **A note on interface names:** Port names vary by image, so confirm with `show ip interface brief` and adjust. All three routers sit on the 192.168.1.0/24 LAN through SW1.

---

## Addressing Table

| Device | Interface | IP Address    | Subnet Mask     | NTP Role |
|--------|-----------|---------------|-----------------|----------|
| R1     | G0/0      | 192.168.1.1   | 255.255.255.0   | Master (server) |
| R2     | G0/0      | 192.168.1.2   | 255.255.255.0   | Client |
| R3     | G0/0      | 192.168.1.3   | 255.255.255.0   | Client |

---

## Lab Objectives

By the end of this lab you will be able to:

1. Explain why synchronized time matters across a network
2. Describe the NTP stratum hierarchy and what stratum 16 means
3. Configure a router as an authoritative NTP master
4. Configure routers as NTP clients in client and server mode
5. Verify synchronization with `show ntp status` and `show ntp associations`
6. Identify and resolve three common NTP failures
7. Apply a structured troubleshooting process to time synchronization

---

## 🧱 Part 1: Build It

### Step 1 — Configure Interfaces

Bring up all three routers on the shared LAN.

**R1:**
```
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip address 192.168.1.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit
```

**R2:**
```
R2(config)# interface GigabitEthernet0/0
R2(config-if)# ip address 192.168.1.2 255.255.255.0
R2(config-if)# no shutdown
R2(config-if)# exit
```

**R3:**
```
R3(config)# interface GigabitEthernet0/0
R3(config-if)# ip address 192.168.1.3 255.255.255.0
R3(config-if)# no shutdown
R3(config-if)# exit
```

Confirm all three can ping each other before configuring NTP. Time cannot synchronize across a network that does not work.

### Step 2 — Set a Sane Clock and Make R1 the Master

Before R1 can serve time, give it a believable time to serve. Set the clock manually in privileged exec mode, then make R1 authoritative.

```
! Set the clock from privileged exec (not config mode)
R1# clock set 10:00:00 1 June 2026
```

Then, in configuration mode, optionally set the time zone and make R1 the master:

```
R1(config)# clock timezone EST -5
R1(config)# ntp master 2
```

> **What `ntp master 2` actually does:** It tells R1 to treat its own internal clock as the reference and serve it at stratum 2. Clients that sync to R1 will become stratum 3, one step further from the source. The stratum number you pick is arbitrary in a lab, but choosing 2 leaves room to show clients landing at 3. Setting the clock first matters, because a master serving a wildly wrong time is worse than no master at all.

### Step 3 — Configure R2 and R3 as Clients

Each client points at R1 as its time source. This single command per device is the client side of client and server mode.

**R2:**
```
R2(config)# ntp server 192.168.1.1
```

**R3:**
```
R3(config)# ntp server 192.168.1.1
```

> **Now comes the hard part, which is waiting.** NTP does not synchronize instantly. It exchanges several timing samples and adjusts gradually to avoid jerking the clock around, which can take several minutes. Configure both clients, then go work on something else and come back. If you check `show ntp status` ten seconds later and see "unsynchronized," that is normal, not a failure.

### Step 4 — Save the Configurations

```
R1# copy running-config startup-config
R2# copy running-config startup-config
R3# copy running-config startup-config
```

---

## 👀 Part 2: Observe It

Give the clients several minutes to synchronize before expecting clean results.

### 2.1 — Check Synchronization Status

Start on R1, then check a client.

```
R1# show ntp status
R2# show ntp status
```

**What to look for:**
- On R1, the clock is synchronized, the stratum is 2, and the reference is its own internal source.
- On R2, once it has synced, the clock is synchronized, the stratum is 3, and the reference clock is 192.168.1.1.
- If a client still shows "clock is unsynchronized," give it more time before assuming anything is wrong.

**✏️ Prediction checkpoint:** Before you look, predict what stratum each device will report and why R2 lands one higher than R1.

### 2.2 — Examine the Associations

```
R2# show ntp associations
```

**What to look for:** The configured server, 192.168.1.1, and its status. The detail that matters most is the symbol at the start of the line. An asterisk marks the peer the device has selected and synchronized to. No asterisk means the device has not locked onto that source yet.

> **The asterisk is the whole story in this output.** When you see the `*` next to 192.168.1.1, R2 has chosen R1 as its master and is keeping time with it. Until that asterisk appears, synchronization is still in progress.

### 2.3 — Compare the Clocks

This is the payoff. Check the time on all three devices and watch them agree.

```
R1# show clock
R2# show clock
R3# show clock
```

Once synchronization completes, the three clocks should read the same time, down to the second. Seeing three independent devices report identical time is NTP working, and it is far more satisfying than it has any right to be.

**✏️ Think it through:** Before NTP, each router kept whatever time it happened to have. Explain what would go wrong when you tried to read logs from all three during an outage, and how synchronized time fixes it.

---

## 💥 Part 3: Break It

> Work through each scenario one at a time. Document the symptoms fully before you move to Part 4. Remember that NTP is slow, so allow time after each change before judging the result.

---

### Break Scenario 1 — The Wrong Server Address

Point R2 at a time source that does not exist:

```
R2(config)# no ntp server 192.168.1.1
R2(config)# ntp server 192.168.1.99
R2(config)# exit
```

> 192.168.1.99 is not R1, and nothing on the network answers at that address.

Give it a few minutes, then check:
```
R2# show ntp status
R2# show ntp associations
R3# show ntp status
```

**✏️ Document the symptoms:**
- Does R2 ever synchronize? What stratum does it settle at?
- Is R3, which still points at the correct server, affected at all?
- The command was accepted without error, so why does R2 never sync?

> **Why this matters:** A wrong server address is the most common NTP misconfiguration, and it fails quietly. IOS accepts the command, R2 dutifully sends its timing requests into the void, and no answer ever comes back, so R2 stays at stratum 16, unsynchronized. R3 is untouched, which is your clue that the problem is specific to R2's configuration rather than the master. When one device will not sync while its peers will, check the server address it is actually pointed at.

---

### Break Scenario 2 — The Source With No Authority

Restore R2's correct server. Then strip the authority from the master itself by removing `ntp master` from R1:

```
R2(config)# no ntp server 192.168.1.99
R2(config)# ntp server 192.168.1.1
R2(config)# exit

R1(config)# no ntp master 2
R1(config)# exit
```

Give it a few minutes, then check both the master and the clients:
```
R1# show ntp status
R2# show ntp status
R3# show ntp status
```

**✏️ Document the symptoms:**
- What does R1 report about its own synchronization now?
- Do R2 and R3 stay synchronized, or do they eventually fall out?
- The clients are still perfectly configured and can still reach R1, so why do they lose their time source?

> **Why this matters:** This is the lesson the Key Concept section promised. The clients are flawless. Their configuration is correct, the network works, and they can reach R1 all day long. But R1 is no longer an authoritative source, so it has nothing trustworthy to hand out, and a client cannot sync to a server that is not itself synced. The failure is at the source, not the clients, even though the clients are where the symptom shows up. When every client loses time at once, look at the server they all depend on.

---

### Break Scenario 3 — The Patience Test

Restore R1 as the master. Then break the path between a client and the server, and pay attention to how NTP behaves over time rather than instantly.

```
R1(config)# ntp master 2
R1(config)# exit
```

Now, on R2, shut the interface that reaches the LAN. Use R2's console for this, since its network connection is about to go down:

```
R2(config)# interface GigabitEthernet0/0
R2(config-if)# shutdown
R2(config-if)# exit
```

Watch R2 over the next several minutes:
```
R2# show ntp status
```

**✏️ Document the symptoms:**
- Immediately after the link goes down, is R2 still synchronized, or does it lose sync instantly?
- After several minutes with no contact, what happens to R2's synchronization?
- When you restore the link, does R2 re-sync immediately, or does it take time again?

Restore the interface when you are done:
```
R2(config)# interface GigabitEthernet0/0
R2(config-if)# no shutdown
R2(config-if)# exit
```

> **Why this matters:** NTP is deliberate by design. When R2 loses contact, it does not panic and abandon its time immediately. It holds the time it has, because a recently good clock is still useful for a while, and only declares itself unsynchronized after it has gone too long without confirmation. When the link returns, R2 does not snap back instantly either. It re-exchanges timing samples and eases back into sync. This slow, stable behavior is a feature. NTP values not jerking clocks around over reacting quickly, and understanding that rhythm is what keeps you from "fixing" a configuration that simply had not finished converging.

---

## 🔧 Part 4: Fix It

---

### Fix Scenario 1 — The Wrong Server Address

**Your troubleshooting toolkit:**
```
show ntp associations         ! shows the server the client is actually using
show running-config | include ntp
```

**Structured process:**
1. Note that one device will not sync while its peers do.
2. Check the server address that device is configured to use.
3. Compare it against the real master's address.
4. Correct the address and wait for synchronization.

**✅ Solution:**
```
R2(config)# no ntp server 192.168.1.99
R2(config)# ntp server 192.168.1.1
```

**Verification (after a few minutes):**
```
R2# show ntp status
R2# show ntp associations
```

---

### Fix Scenario 2 — The Source With No Authority

**Your troubleshooting toolkit:**
```
show ntp status               ! run on the server itself
```

**Structured process:**
1. Note that every client lost time at once, which points at the shared source.
2. Check the master's own synchronization status.
3. If the master reports unsynchronized, it has no authority to hand out.
4. Restore the master's reference and wait for the clients to re-sync.

**✅ Solution:**
```
R1(config)# ntp master 2
```

**Verification (after a few minutes):**
```
R1# show ntp status
R2# show ntp status
```

---

### Fix Scenario 3 — The Patience Test

**Your troubleshooting toolkit:**
```
show ip interface brief
show ntp status
```

**Structured process:**
1. Confirm the client lost its path to the server.
2. Restore the connection.
3. Resist the urge to keep reconfiguring. Give NTP time to re-exchange samples.
4. Verify synchronization returns on its own.

**✅ Solution:**
```
R2(config)# interface GigabitEthernet0/0
R2(config-if)# no shutdown
R2(config-if)# exit
```

**Verification (after a few minutes):**
```
R2# show ntp status
R2# show clock
```

---

## 💬 Reflection Questions

1. Explain why synchronized time matters across a network. Give a concrete example of a task that becomes very hard when device clocks disagree.

2. Describe the NTP stratum hierarchy in your own words. What does a stratum of 16 indicate, and why can a client never reach a lower stratum number than its server?

3. In Break Scenario 2, the clients were perfectly configured and could reach the server, yet they lost synchronization. Explain why, using the idea that a server must itself be synchronized to a valid source.

4. NTP synchronization takes minutes rather than seconds, and a client that loses its source holds its time for a while before declaring itself unsynchronized. Explain why this slow, stable behavior is intentional and what would go wrong if NTP reacted instantly to every change.

5. When you ran `show ntp associations`, the asterisk marked the selected source. Why is that single symbol the most important thing in the output, and what does its absence tell you?

6. Looking ahead to logging, explain why configuring NTP before configuring Syslog is the right order. What value would your logs lose if the clocks were never synchronized?

---

## Challenge Extension (Optional)

1. **Build a stratum chain.** Reconfigure R3 to sync to R2 instead of R1, then check R3's stratum. Confirm it lands at stratum 4, one higher than R2, and explain how the hierarchy grew by one step.

2. **Secure NTP with authentication.** Research and configure NTP authentication using `ntp authenticate`, `ntp authentication-key`, and `ntp trusted-key` on the master and a client. Then deliberately mismatch the keys and observe the client refuse to sync even though the server is reachable.

3. **Make a client a server too.** Configure R2 to sync to R1 and also serve time to R3, so R2 acts as both client and server. Trace the stratum values through this two-tier design.

4. **Watch the drift.** Before syncing a fresh client, compare its clock to the master and note how far apart they are. After it syncs, compare again. Reason about how much a typical device clock drifts over a day without NTP.

---

## Instructor Notes

**Common Student Mistakes**
- Impatience above all. Students configure NTP correctly, check `show ntp status` within seconds, see "unsynchronized," and start changing a configuration that was never broken. Set the expectation early that synchronization takes minutes, and that the first check should come after a deliberate wait.
- Forgetting `ntp master` on the source, so clients point at a server that has no authoritative time of its own. The clients look misconfigured when the real problem is at the top.
- Not setting the master's clock to a believable time first, which leads to confusion about whether sync actually happened when every device agrees on a clearly wrong time.
- Pointing `ntp server` at the wrong address, or at the client's own interface, and not noticing because the command is accepted without complaint.
- Reading `show ntp associations` without knowing the asterisk marks the synchronized peer, so they cannot tell a locked source from a configured one.

**Teaching Tips**
- Put `show clock` from all three devices on screen side by side, before sync and after. The moment three independent clocks converge to the same second is the payoff of the whole lab, and seeing it lands the concept better than any explanation.
- Lean into the slowness rather than apologizing for it. Frame NTP's deliberate pace as a design choice that values stability over speed, and tie it to the real world, where you never want a server suddenly jerking every clock in the building. Patience here is a professional habit, not a lab inconvenience.

---

## Where This Leads

You have given your network a shared sense of time, and although it is invisible when everything works, it is the foundation that several other services quietly stand on. Every timestamp in a log, every certificate expiration, every scheduled task now means the same thing on every device, because every device agrees on what time it is.

That foundation is exactly what the next lab needs. Syslog collects the messages your devices generate, the record of what happened and when, and SNMP lets a monitoring station ask devices how they are doing. Both are far more useful on a network whose clocks agree, because a log is only as trustworthy as its timestamp. Having built NTP first, you are ready to build the logging and monitoring that depend on it, and that is where the Services section goes next.

---

*CCNA – Build It · Observe It · Break It · Fix It series*

*Next: Syslog and SNMP*

*This repository is an independent educational resource and is not officially affiliated with Cisco Systems. Cisco, CCNA, Cisco Modeling Labs, and Cisco Networking Academy are trademarks of Cisco Systems, Inc.*
