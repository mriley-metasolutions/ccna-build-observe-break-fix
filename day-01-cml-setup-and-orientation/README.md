# Day 01 – CML Setup & Orientation

Welcome to Day 01 of the CCNA Build · Observe · Break · Fix study plan.

This first module is not about VLANs, routing, or IP addressing.

It’s about getting your lab environment working, understanding the Cisco Modeling Labs (CML) Free interface, and building confidence with the tools you’ll use throughout the rest of this repository.

If you can get CML installed, launch a lab, start a node, and open a console—you’ve already accomplished something meaningful.

---

## Why Start With CML Setup?

For many learners, the hardest part of hands-on networking is not the commands—it’s getting the lab tools running.

CML is a powerful and realistic platform, but it can be:

* Finicky about system requirements
* Sensitive to virtualization settings
* Confusing for first-time users

Even as someone who has worked with networking labs and simulations for nearly a decade as an instructor in the Cisco Networking Academy, I ran into issues getting CML installed and running cleanly.

That’s normal.

Getting CML up and running is a **feat in itself**, and this module is designed to walk you through that process in a calm, structured way.

---

## A Note on Tools: CML vs Packet Tracer

This repository is built primarily around **Cisco Modeling Labs (CML) Free**.

However, you are more than welcome to use **Packet Tracer** for the hands-on lab activities in this module if that works better for you.

The learning goals remain the same regardless of platform:

* Build simple topologies
* Observe network behavior
* Intentionally break things
* Troubleshoot and fix them

I’ve chosen CML because it provides more realistic IOS behavior and troubleshooting output, but the concepts you’ll learn here are platform-independent.

If Packet Tracer helps you get started more easily, use it. You can always transition to CML later.

---

## Learning Objectives

By the end of this first module, you should be able to:

* Install Cisco Modeling Labs (CML) Free
* Launch the CML interface
* Create a new lab
* Add a network device (node)
* Start a node and open its console
* Recognize when IOS is ready for input
* Understand basic CML terminology (labs, nodes, images, consoles)

---

## How This Module Is Structured

This module follows the same Build · Observe · Break · Fix learning cycle used throughout the repository—applied to the lab tool itself.

### 🧱 Build 

You will:

* Install CML Free
* Launch the CML interface
* Create your first lab
* Add a router node
* Start the node
* Open the console

---

### 👀 Observe

You will:

* Watch the IOS boot process
* Identify when the device is ready for input
* Run basic commands like:

  * `show version`
  * `show ip interface brief`
* Observe interface states and default behavior

---

### 💥 Break

You will intentionally create small setup problems, such as:

* Forgetting to start a node
* Selecting the wrong device image
* Misconnecting links
* Closing the console mid-boot
* Launching a lab without required resources

This helps you learn what “broken” looks like in CML.

---

### 🔧 Fix

You will troubleshoot and recover from those issues by:

* Restarting nodes
* Reopening consoles
* Selecting correct images
* Fixing topology wiring
* Verifying that IOS loads correctly

This mirrors real-world troubleshooting and builds early confidence.

---

## Files in This Folder

This folder will contain:

* `concepts.md` – CML basics, terminology, and environment concepts
* `build.md` – Step-by-step CML installation and first lab creation
* `observe.md` – Guided observation of the IOS boot process
* `break.md` – Common CML setup failures and mistakes
* `fix.md` – Troubleshooting and recovery steps
* `screenshots/` – Visual references for key steps
* `troubleshooting.md` – Extra help for common installation issues

---

## A Note on Learning

If you run into problems during setup, don’t get discouraged.

Troubleshooting the lab environment is part of learning networking.

And if you need to fall back to Packet Tracer temporarily, that’s completely okay.

What matters is that you:

* Keep moving forward
* Keep experimenting
* Keep observing cause and effect

---

## Instructor Insight

This first module is intentionally focused on tooling, not networking theory.

Once you can reliably build and launch labs, every future module becomes dramatically easier.

Think of this first learning module as laying the foundation for everything that follows.

---

**Build it. Observe it. Break it. Fix it.**
That’s how you’ll learn networking here.
