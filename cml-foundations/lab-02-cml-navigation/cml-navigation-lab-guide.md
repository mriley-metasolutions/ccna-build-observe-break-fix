# Lab 02 – Navigating the Cisco Modeling Labs (CML) Interface

## Overview

Before building larger networks, it is important to become comfortable navigating the Cisco Modeling Labs (CML) interface.

Think of this lab as learning where the tools are in your workshop before beginning a project.

In this lab, you will explore the CML workspace, create a simple lab, add devices, connect them together, start and stop nodes, and access device consoles.

By the end of this lab, you should feel confident moving around the CML environment and preparing topologies for future exercises.

\---

## Learning Objectives

Upon completion of this lab, you will be able to:

* Navigate the CML dashboard
* Create a new lab
* Add devices to a topology
* Connect devices using links
* Start and stop nodes
* Access device consoles
* Save and reopen labs
* Identify common interface components

\---

# Build It

## Part 1: Explore the Dashboard

After logging into CML, spend a few minutes exploring the interface.

Locate the following:

* Lab Manager
* Node Palette
* Link Tool
* Start/Stop Controls
* Console Access Options

### Reflection Question

Before clicking anything:

> Which area of the interface do you think you will use most often?

\---

## Part 2: Create Your First Lab

1. Select **New Lab**
2. Name the lab:

```text
Lab-02-CML-Navigation
```

3. Save the lab

### Observe

Notice:

* Where the lab name appears
* How the workspace changes after creation
* Where lab management options are located

\---

## Part 3: Add Devices

Add the following nodes:

* 1 IOSv Router
* 1 Unmanaged Switch
* 1 Desktop Host

Arrange them neatly on the workspace.

### Reflection Question

Before connecting anything:

> What do you think will happen if you start these devices without any links configured?

\---

## Part 4: Connect Devices

Create the following connections:

```text
Desktop ---- Switch ---- Router
```

Observe:

* Interface labels
* Link indicators
* Connection behavior

\---

## Part 5: Start the Topology

Start all nodes.

Observe:

* Device status icons
* Boot progress indicators
* Changes in node appearance

### Prediction Question

Which device do you expect to boot fastest?

* Router
* Switch
* Desktop

Why?

\---

# Observe It

## Part 6: Open Device Consoles

Open a console session to:

* Router
* Desktop

Observe:

* CLI differences
* Prompt formats
* Boot messages

Run the following command on the router:

```bash
show version
```

### Questions

1. What IOS version is running?
2. How much memory is available?
3. How does this differ from Packet Tracer?

\---

## Part 7: Examine Node States

Stop and restart individual devices.

Observe:

* State changes
* Console behavior
* Reconnection process

\---

# Break It

Now we're going to intentionally make mistakes.

\---

## Scenario 1: Attempt Console Access Before Boot Completes

Open the router console immediately after starting the node.

### Questions

* What do you see?
* Is the router ready for commands?

\---

## Scenario 2: Delete a Link

Remove the switch-to-router connection.

Observe:

* Visual changes
* Interface indicators

### Question

> How could this impact troubleshooting later?

\---

## Scenario 3: Stop a Device Unexpectedly

Power off the router.

Attempt to:

* Access the console
* Communicate through the topology

Observe the results.

\---

## Scenario 4: Close Your Browser Tab

Close the console session.

Attempt to reconnect.

Observe:

* What persists
* What changes

\---

# Fix It

## Fix Scenario 1

Wait for the router to complete booting.

Verify:

```bash
show ip interface brief
```

\---

## Fix Scenario 2

Reconnect the missing link.

Verify that the topology visually matches:

```text
Desktop ---- Switch ---- Router
```

\---

## Fix Scenario 3

Restart the router.

Confirm:

* Console access works
* IOS loads successfully

\---

## Fix Scenario 4

Reopen the console.

Verify that the device state was maintained.

\---

# Common Navigation Tips

## Save Frequently

Although CML automatically tracks many changes, develop the habit of saving your work.

## Name Labs Clearly

Good names help later:

```text
01-CML-Navigation
02-Static-Routing
03-VLAN-Fundamentals
```

## Keep Topologies Organized

Neat diagrams are easier to troubleshoot.

Future-you will appreciate this.

\---

# Reflection Questions

1. Which area of the interface felt most intuitive?
2. Which area was least intuitive?
3. What surprised you about using CML?
4. How does the experience compare to Packet Tracer?
5. What troubleshooting skills did you practice during this lab?

\---

# Challenge Extension

Without instructions:

1. Create a new lab.
2. Add:

   * Two routers
   * One switch
   * One desktop
3. Connect them logically.
4. Start all devices.
5. Open a console on each router.

If you can complete these tasks without referring to the lab guide, you're ready for the next module:

> \*\*Lab 03 – Building Your First Multi-Device Network\*\*

\---

## Key Takeaway

This lab wasn't about learning networking concepts.

It was about becoming comfortable with the CML environment itself.

The more comfortable you become with the platform, the more energy you can devote to understanding networking concepts and troubleshooting skills in future labs.

Remember:

> \*\*Build it. Observe it. Break it. Fix it.\*\*
>
> The philosophy applies to the tools just as much as it applies to the networks you build.

---

*CCNA – Build It · Observe It · Break It · Fix It This repository is an independent educational resource and is not officially affiliated with Cisco Systems. Cisco, CCNA, Cisco Modeling Labs, and Cisco Networking Academy are trademarks of Cisco Systems, Inc.*

