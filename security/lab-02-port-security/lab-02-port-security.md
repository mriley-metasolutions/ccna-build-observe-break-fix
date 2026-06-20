# Port Security

**Build It · Observe It · Break It · Fix It**

---

## Overview

The SSH lab secured how you log into a device across the network. But there is an attacker the management plane never sees, the one who simply walks up to a wall jack, plugs in a laptop, and is now on your network. Or worse, they plug in a small switch and connect a dozen unauthorized devices through a single network drop. Or worst of all, the attacker runs a tool that floods the switch with thousands of fake MAC addresses until its address table overflows, at which point the switch fails open and begins flooding every frame out every port, turning the whole segment into something an attacker can quietly listen to. The access layer, where users actually plug in, is its own security frontier, and **Port Security** is how you defend it.

Port security is a switch feature that controls which devices, and how many, are allowed to use a given access port. You decide the maximum number of MAC addresses the port will accept, you decide how those addresses are learned, and you decide what happens when an unauthorized device shows up. That last decision, the **violation mode**, is the heart of the feature, because it is the difference between silently dropping the intruder, logging the event, or shutting the port down entirely.

In this lab you will secure a switch port so that only an authorized device may use it, watch the port learn and lock onto that device, and then trigger a violation by introducing an unauthorized one. You will see the port react, recover it, and compare the three violation modes so you understand exactly what each one does and does not tell you. This is defense in depth made concrete, the next layer down from securing the login.

**CCNA Exam Alignment:**
- 5.7 – Configure Layer 2 security features, including port security

**Estimated Time:** 75–105 minutes

---

## Prerequisites

Before completing this lab, you should be comfortable with:
- Basic IOS navigation, including moving between command modes and reading `show` command output
- VLANs and access ports, since port security applies to access ports (see the VLANs lab)
- How a switch learns MAC addresses and builds its address table (Switching fundamentals)
- The threat and mitigation vocabulary from the Security Fundamentals concept guide
- CML fundamentals, including building or importing a topology and accessing a device console

---

## Key Concept: How MACs Are Learned, and What Happens When the Rule Breaks

Port security rests on two decisions, and the commands all flow from them.

The first decision is **how the allowed MAC addresses get onto the port**. There are three ways:

| Method | How the address is set | Survives a reboot? |
|--------|------------------------|--------------------|
| **Static** | You type the exact MAC address into the configuration by hand | Yes, it is in the config |
| **Dynamic** | The port learns addresses as devices connect, storing them only in memory | No, learned fresh each boot |
| **Sticky** | The port learns addresses dynamically, then writes them into the running configuration | Yes, if you save the config |

Sticky is the practical sweet spot, and the one you will use here. You do not have to look up and type a MAC address, since the port learns it automatically, but once learned it sticks in the configuration rather than evaporating on reboot. It is automatic convenience with permanent results.

The second decision is **what the port does when a violation occurs**, meaning when an unauthorized MAC appears or the maximum is exceeded. There are three violation modes, and knowing the differences cold is exam material and operational necessity:

| Violation mode | Action on violation | Notification | Port state afterward |
|----------------|---------------------|--------------|----------------------|
| **protect** | Drops the offending traffic | None at all | Stays up |
| **restrict** | Drops the offending traffic | Logs it, increments the counter, can send SNMP | Stays up |
| **shutdown** (default) | Err-disables the port | Logs it, increments the counter, can send SNMP | Down, err-disabled |

The default is shutdown, and it is the most decisive. The port goes into an err-disabled state and stays down until someone deliberately brings it back, which is exactly the recovery skill you will practice.

> **Get ahead of the confusion:** A violation in `protect` mode is completely silent. The unauthorized device is blocked, which sounds fine, but no log, no counter, and no alert means you have no idea it ever happened. An attacker probing your ports would be stopped and invisible at the same time. When you choose a violation mode, you are really choosing how much you want to know, and silence is rarely what you want.

---

## Topology

```
        [PC1]                 [PC2]
   authorized device     unauthorized device
   192.168.10.11           192.168.10.12
         \                   /
          \                 /
           [Hub1]   (unmanaged, the shared "wall jack")
              |
            G0/1   (secured access port, VLAN 10)
           [SW1]
        SVI VLAN 10: 192.168.10.1
```

**Nodes (3 counting toward the limit):**
- SW1 (IOSvL2 switch, the device enforcing port security)
- PC1 (the authorized device)
- PC2 (the unauthorized device)
- Hub1 (an unmanaged switch acting as a shared segment, which does not count toward the CML Free node limit)

> **Why a hub is in the picture:** A single secured port needs to see more than one MAC address for a violation to occur, and the unmanaged switch lets PC1 and PC2 share the one secured port the way an attacker's rogue switch or a flooding tool would present many addresses on a single drop. PC1 connects first as the legitimate device, and PC2 is the intruder you introduce later. Confirm interface names with `show ip interface brief` and adjust.

---

## Addressing Table

| Device | Interface | IP Address     | Subnet Mask     | Role |
|--------|-----------|----------------|-----------------|------|
| SW1    | SVI VLAN 10 | 192.168.10.1 | 255.255.255.0   | Target for test pings |
| PC1    | NIC       | 192.168.10.11  | 255.255.255.0   | Authorized device |
| PC2    | NIC       | 192.168.10.12  | 255.255.255.0   | Unauthorized device |

> **The switch SVI exists only so the PCs have something to ping,** which generates the traffic that makes their MAC addresses appear on the secured port. Port security itself operates at Layer 2 and does not need the SVI.

---

## Lab Objectives

By the end of this lab you will be able to:

1. Explain what port security protects against, including unauthorized devices and MAC flooding
2. Enable port security on an access port and set a maximum MAC count
3. Configure sticky MAC learning and verify the learned address
4. Trigger a violation and observe the default shutdown response
5. Recover an err-disabled port and automate recovery
6. Compare the protect, restrict, and shutdown violation modes
7. Troubleshoot common port security problems

---

## 🧱 Part 1: Build It

### Step 1 — Prepare the Switch and the Access Port

Create the VLAN, give the switch an SVI for testing, and set the port to access mode. Port security only works on a port that is explicitly an access port, so this is a real prerequisite, not a formality.

```
SW1(config)# vlan 10
SW1(config-vlan)# name Users
SW1(config-vlan)# exit

SW1(config)# interface vlan 10
SW1(config-if)# ip address 192.168.10.1 255.255.255.0
SW1(config-if)# no shutdown
SW1(config-if)# exit

SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# exit
```

### Step 2 — Enable Port Security and Set the Rules

Now secure the port. Enable port security, allow exactly one MAC address, learn it stickily, and leave the violation mode at its default of shutdown.

```
SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# switchport port-security
SW1(config-if)# switchport port-security maximum 1
SW1(config-if)# switchport port-security mac-address sticky
SW1(config-if)# switchport port-security violation shutdown
SW1(config-if)# exit
```

> **You just told the port four things.** Be secure, accept at most one device, remember whatever device you see by writing its address into the config, and if a second device ever appears, shut yourself down. The violation line is optional here since shutdown is the default, but configuring it explicitly makes your intent clear to the next person who reads the config.

### Step 3 — Connect Only the Authorized Device First

Make sure PC1 is the only device generating traffic at this point. PC2 should be powered off or otherwise not sending, so that the port learns the correct, authorized address. From PC1, ping the switch to make its MAC address appear on the port:

```
PC1 ping 192.168.10.1
```

The port now sees PC1's MAC, accepts it as the one allowed address, and writes it into the configuration as a sticky entry.

### Step 4 — Save the Configuration

```
SW1# copy running-config startup-config
```

> **Saving matters more than usual here.** The sticky MAC address lives in the running configuration, so if you do not save, a reload wipes the learned address and the port relearns whatever connects first next time. Saving locks in the authorized device.

---

## 👀 Part 2: Observe It

### 2.1 — Review the Port Security Status

```
SW1# show port-security interface GigabitEthernet0/1
```

**What to look for:** Port security enabled, the port status secure-up, the maximum set to 1, the current address count at 1, the violation mode shutdown, and the violation counter at 0. This is the healthy baseline, one authorized device and no violations.

**✏️ Prediction checkpoint:** Before you look, predict what the address count and violation count should be when only the authorized PC1 is connected.

### 2.2 — See the Sticky Address

```
SW1# show port-security address
```

This lists the secured MAC addresses and the port each belongs to. PC1's address should appear, marked as a sticky secure address on G0/1. Then confirm it really did write into the config:

```
SW1# show running-config interface GigabitEthernet0/1
```

You will see a line that pins PC1's exact MAC to this port. That line is the sticky learning made permanent.

### 2.3 — Trigger the Violation

This is the moment the lab is built around. Bring PC2 online, the unauthorized device, and have it send traffic on the same secured port:

```
PC2 ping 192.168.10.1
```

PC2's MAC address is a second address on a port that allows only one, which is a violation. With the mode set to shutdown, the port reacts immediately.

```
SW1# show port-security interface GigabitEthernet0/1
SW1# show interfaces GigabitEthernet0/1
```

**✏️ Document what you see:**
- What is the port status now? Look for secure-shutdown and err-disabled.
- What does the violation counter read?
- Did PC1, the legitimate device, also lose its connection when the port shut down?

> **Why this matters:** This is port security doing exactly its job. An unauthorized device appeared, and the port enforced the rule by shutting down completely. Notice the collateral effect, that the legitimate PC1 also lost connectivity, because the violation takes the whole port down, not just the offender. That tradeoff is the entire reason the three violation modes exist, and you are about to weigh them.

---

## 💥 Part 3: Break It

> Work through each scenario one at a time. Document the symptoms fully before you move to Part 4.

---

### Break Scenario 1 — The Port That Will Not Come Back

You triggered the violation, the port is err-disabled, and now you face the situation every operator eventually does. The port is down and will not recover on its own. First remove the cause by stopping PC2 from sending, then confront the recovery.

Check the port and watch it stay down:
```
SW1# show interfaces status err-disabled
```

**✏️ Document the symptoms:**
- Is the port still down even after the unauthorized device stops sending?
- Does removing the offending device automatically bring the port back?
- What would you have to do to restore service, and how would you avoid doing it by hand every time?

> **Why this matters:** An err-disabled port does not heal itself. Removing the violation is necessary but not sufficient, because the port stays administratively shut down until someone acts. On a busy network, a security feature that requires a manual visit for every tripped port becomes its own operational problem, which is why automated recovery exists. The fix below shows both the manual recovery and the way to automate it.

---

### Break Scenario 2 — The Stale Sticky Address

Restore the port first. Now imagine a common real-world change. The authorized user's computer is replaced, or its network card is swapped, so the legitimate device on this port now has a different MAC address. The port, however, still has the old sticky address locked in.

Simulate this by making PC2 the new authorized device while PC1 is disconnected, so the only device present is one whose MAC does not match the stored sticky entry. Have that device send traffic:

```
PC2 ping 192.168.10.1
```

**✏️ Document the symptoms:**
- The new device is legitimate, so why is it being treated as a violation?
- What does `show port-security address` still show as the allowed MAC?
- Why does a feature designed to help end up locking out an authorized device here?

> **Why this matters:** Sticky addresses persist, which is the point, but persistence cuts both ways. When the hardware behind a port legitimately changes, the old sticky entry is now wrong, and the port faithfully rejects the new, authorized device because it does not match. This is one of the most common port security support calls, a user who cannot get online after a computer swap. The fix is to clear the stale entry so the port can learn the new one.

---

### Break Scenario 3 — The Silent Mode

Restore normal operation with PC1 as the authorized device. Now change the violation mode to protect and trigger a violation again, paying close attention to what you are told:

```
SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# switchport port-security violation protect
SW1(config-if)# exit
```

Bring PC2 back and have it send traffic:
```
PC2 ping 192.168.10.1
```

Then check:
```
SW1# show port-security interface GigabitEthernet0/1
```

**✏️ Document the symptoms:**
- Is PC2's traffic blocked? Is the port still up?
- What does the violation counter read in protect mode?
- If you were the security team, would you ever know this violation happened?

> **Why this matters:** Protect mode blocks the unauthorized device but tells you nothing. The port stays up, the counter does not move, and no log is generated, so the intruder is stopped and completely invisible. That might sound efficient, but on a real network, silence about a security event is dangerous. You usually want at least restrict mode, which blocks the traffic and records it, so you keep service running on the port while still knowing something happened. Choosing the violation mode is choosing your balance between disruption and awareness.

---

## 🔧 Part 4: Fix It

---

### Fix Scenario 1 — The Port That Will Not Come Back

**Your troubleshooting toolkit:**
```
show interfaces status err-disabled
show port-security interface GigabitEthernet0/1
```

**Structured process:**
1. Confirm the violation is over by ensuring the unauthorized device is no longer present.
2. Manually recover the port by bouncing it.
3. Optionally, configure automatic recovery so future violations clear themselves after a delay.
4. Verify the port returns to service.

**✅ Solution, manual recovery:**
```
SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# shutdown
SW1(config-if)# no shutdown
SW1(config-if)# exit
```

**✅ Solution, automatic recovery:**
```
SW1(config)# errdisable recovery cause psecure-violation
SW1(config)# errdisable recovery interval 300
```

> This tells the switch to automatically re-enable a port that was err-disabled by a port security violation, after a 300-second wait. Useful on large networks, though some teams prefer manual recovery so a human always reviews the cause.

**Verification:**
```
SW1# show interfaces GigabitEthernet0/1
```

---

### Fix Scenario 2 — The Stale Sticky Address

**Your troubleshooting toolkit:**
```
show port-security address
show running-config interface GigabitEthernet0/1
```

**Structured process:**
1. Confirm the stored sticky address no longer matches the legitimate device.
2. Remove the stale sticky entry from the port.
3. Let the port learn the new authorized device, then save.
4. Verify the new address is secured.

**✅ Solution:**
```
SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# no switchport port-security mac-address sticky
SW1(config-if)# switchport port-security mac-address sticky
SW1(config-if)# exit
SW1# clear port-security sticky interface GigabitEthernet0/1
```

> Removing and re-adding sticky learning, then clearing the old learned address, lets the port relearn the current device. Have the new authorized device send traffic, then save the configuration to lock in the new sticky address.

**Verification:**
```
SW1# show port-security address
```

---

### Fix Scenario 3 — The Silent Mode

**Your troubleshooting toolkit:**
```
show port-security interface GigabitEthernet0/1
```

**Structured process:**
1. Recognize that protect mode blocks violations without any record.
2. Decide how much awareness you need versus how much disruption you can tolerate.
3. Set restrict mode to keep the port up while logging violations, or shutdown to stop hard.
4. Verify the mode and confirm violations now register.

**✅ Solution:**
```
SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# switchport port-security violation restrict
SW1(config-if)# exit
```

**Verification:**
```
SW1# show port-security interface GigabitEthernet0/1
```
Trigger a violation again and confirm the counter now increments and a log message appears, unlike in protect mode.

---

## 💬 Reflection Questions

1. Explain two distinct attacks that port security defends against, one involving a single unauthorized device and one involving a flood of MAC addresses. What does the switch do badly when its address table overflows?

2. Compare static, dynamic, and sticky MAC learning. Why is sticky often the practical choice, and what is the one step you must not forget when using it?

3. Lay out the three violation modes and what each one does to the traffic, the notification, and the port state. For a port serving a single office user, which mode would you choose and why?

4. In the Observe section, triggering a violation in shutdown mode also knocked the legitimate device offline. Explain why, and how a different violation mode would have avoided that collateral damage.

5. An err-disabled port does not recover on its own. Describe the manual recovery, and explain the tradeoff of configuring automatic err-disable recovery instead.

6. A sticky MAC address is meant to help, but in Break Scenario 2 it locked out an authorized device. Explain how persistence became a problem, and what routine maintenance step prevents it.

---

## Challenge Extension (Optional)

1. **Set a higher maximum for a phone and PC.** Many office ports carry an IP phone with a PC behind it, needing two MAC addresses. Reconfigure the port for a maximum of two and confirm both devices are allowed while a third triggers a violation.

2. **Use a static secure address.** Instead of sticky learning, configure a specific MAC address statically with `switchport port-security mac-address`, and explain when typing the address by hand is preferable to letting the port learn it.

3. **Watch the address table.** Use `show mac address-table secure` to see the secured addresses in the switch's table, and compare it with `show port-security address`. Explain what each view tells you.

4. **Test automatic recovery end to end.** Configure err-disable recovery with a short interval, trigger a violation, and time how long the port takes to come back on its own. Then reason about when automatic recovery is appropriate and when it is not.

---

## Instructor Notes

**Common Student Mistakes**
- Trying to enable port security on a port that is not explicitly an access port. Port security needs `switchport mode access` first, and students often skip it and then cannot understand why the security commands will not take.
- Forgetting to save after sticky learning, so a reload wipes the authorized address and the port relearns whatever connects first. Tie this directly to the running-config versus startup-config distinction.
- Expecting protect mode to alert someone. Make sure students leave understanding that protect is silent, and that restrict is usually the better default when you want the port to stay up.
- Not knowing how to recover an err-disabled port. The manual `shutdown` then `no shutdown` is the single most useful recovery step, and many students have never seen it.
- Being surprised that a shutdown violation also drops the legitimate device on the port. Use this to motivate the whole discussion of violation modes.

**Teaching Tips**
- Trigger the violation live and let the class watch the port slam into err-disabled. The drama of a port shutting itself down is what makes the feature memorable, and it motivates everything that follows.
- Run the same violation under all three modes back to back and show what the operator sees each time, nothing in protect, a log and a counter in restrict, a dead port in shutdown. Seeing the three side by side cements the tradeoffs better than the table alone.

---

## Where This Leads

You have now secured the access layer itself, controlling not just how someone logs into a device but which devices are even allowed to use a port. Combined with the SSH lab before it, you are building security in layers, the management plane and now the physical edge, which is exactly the defense in depth the fundamentals guide described. You also learned to weigh a real security tradeoff, between stopping a threat hard and staying aware of it, which is a judgment you will make constantly in this field.

Port security decides who may connect at Layer 2. The next question is what they are allowed to do once they are on the network, which is a Layer 3 concern. Access Control Lists are how you filter traffic, permitting some flows and denying others based on addresses and, later, on ports and protocols. They are one of the most heavily used tools in all of networking, and they are where the Security section goes next, starting with standard ACLs and the foundational idea of matching traffic and deciding its fate.

---

*CCNA – Build It · Observe It · Break It · Fix It series*

*Next: Standard ACLs*

*This repository is an independent educational resource and is not officially affiliated with Cisco Systems. Cisco, CCNA, Cisco Modeling Labs, and Cisco Networking Academy are trademarks of Cisco Systems, Inc.*
