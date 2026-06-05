# Getting Started with Cisco Modeling Labs (CML)

**CCNA – Build It · Observe It · Break It · Fix It**

---

## What Is Cisco Modeling Labs?

Cisco Modeling Labs — CML for short — is a **network emulation platform** developed and maintained by Cisco Systems. It allows you to build virtual network topologies using real Cisco IOS images, running directly on your laptop or desktop computer, without needing any physical hardware.

That word *emulation* is important. CML is not a simulator.

### Emulation vs. Simulation — Why It Matters

| | Simulator (Packet Tracer) | Emulator (CML) |
|---|---|---|
| **How it works** | Models the *behavior* of IOS commands | Runs the *actual* IOS software |
| **CLI accuracy** | Most CCNA commands work | Full IOS feature set available |
| **show command output** | Approximated | Identical to real hardware |
| **Troubleshooting realism** | Limited | True-to-life |
| **Best for** | Learning concepts, early CCNA study | Lab validation, advanced topics, real troubleshooting |

Cisco Packet Tracer is an excellent starting point — and it remains a core tool in the Cisco Networking Academy curriculum. But as you advance toward your CCNA exam and into real-world networking, the gap between simulated behavior and actual IOS behavior becomes noticeable. CML closes that gap entirely.

When you run `show ip ospf neighbor` in CML, you are reading the output of the same OSPF process that runs on a physical Cisco router. There is no approximation.

---

## Why CML Matters for CCNA Study

The new CCNA exam revision shifted significantly toward **understanding and troubleshooting** rather than rote command memorization. Questions now expect you to:

- Interpret real `show` command output
- Identify root causes from symptoms
- Troubleshoot errors
- Understand *why* a network behaves the way it does — not just how to configure it

That shift is exactly why hands-on lab work in a real emulation environment matters more than ever. The labs in this repository are built around the **Build It · Observe It · Break It · Fix It** methodology for exactly this reason — reading about OSPF neighbor states is useful, but watching a neighbor get stuck in **Init** because of a timer mismatch, and then troubleshooting your way back, is what makes the knowledge stick.

CML makes that kind of learning possible without a rack of physical hardware.

---

## CML Editions — What You Need to Know

Cisco offers several CML tiers. For self-study and the labs in this repository, **CML Free** is the right starting point.

| Edition | Cost | Node Limit | Best For |
|---------|------|------------|----------|
| **CML Free** | No cost | 5 nodes | CCNA self-study, this repo |
| **CML Personal** | ~$199/year | 20 nodes | Advanced labs, homelab |
| **CML Enterprise / Education** | Licensed | Unlimited | Organizations, academies |

### CML Free — Key Details

- **No license required.** You register with a free Cisco CCO account and download directly.
- **5-node limit.** Up to 5 virtual network devices running simultaneously. Unmanaged switches and external connectors do not count against this limit.
- **Included images.** CML Free includes IOL (IOS on Linux), IOL-L2, ASAv, Ubuntu Linux, Alpine Linux, and a Desktop image. These cover everything you need for CCNA-level labs.
- **Full feature parity.** CML Free includes all the same features as CML Personal — the only difference is the node count ceiling.
- **Telemetry.** CML Free sends anonymized usage data to Cisco. No configurations, passwords, or personally identifiable information are collected. This cannot be disabled on the free tier.

> **All labs in this repository are designed specifically for the 5-node limit.** Topologies use loopback interfaces to simulate LAN segments where possible, keeping active node counts at or below 5.

---

## Before You Install — System Requirements

CML runs as a virtual machine inside a hypervisor on your host computer. Because CML itself runs virtual machines *inside* it (your routers and switches), it requires **nested virtualization** support from your CPU. This is the most important hardware requirement to verify before you begin.

### Minimum Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| **CPU** | Intel, 4 physical cores, VT-x/EPT support | Intel, 8+ cores |
| **RAM** | 8 GB allocated to the CML VM | 16 GB or more |
| **Disk** | 32 GB for the CML VM | 64 GB+ SSD |
| **OS** | Windows 10/11, macOS (Intel), Linux | Any of the above |
| **Hypervisor** | VMware Workstation, Fusion, or ESXi | VMware Workstation Pro |

### Important Platform Notes

**Intel vs. AMD:** CML is officially supported on Intel processors only. AMD CPUs are known to work in most cases, but some bundled IOS images are compiled for Intel and may behave unexpectedly on AMD. If you have an Intel CPU, you are on solid ground.

**Apple Silicon (M1/M2/M3/M4):** CML does not currently support Apple M-series chips. The nested virtualization architecture CML depends on is not compatible with ARM-based Macs. If you have an Apple Silicon Mac, consider running CML on a cloud instance (AWS or Azure) or using a separate Intel-based machine.

**Windows + Hyper-V conflict:** If Hyper-V is enabled on your Windows machine (common on Windows 10/11 Pro and Enterprise), it can conflict with VMware's ability to use nested virtualization. If you encounter this, you will need to disable Hyper-V before CML will work correctly. Instructions are in the troubleshooting section below.

**RAM matters most:** Each IOL router node uses roughly 512 MB–1 GB of RAM when running. With 5 nodes active, you need at least 4–6 GB available *just for the nodes*, on top of what CML itself uses (~1 GB idle) and what your host OS uses. 16 GB of system RAM is a comfortable floor for CCNA-level labs.

---

## Step-by-Step Installation Guide

### Step 1 — Create a Cisco CCO Account

If you do not already have a Cisco account, you need one before you can download CML.

1. Navigate to **https://id.cisco.com**
2. Click **Sign Up** and complete your profile
3. Fill in all fields including Company Details — even if you are studying independently, this field is required due to U.S. export control regulations
4. Verify your email address

> **Already have a CCO ID?** If you have used Cisco Packet Tracer, the Cisco Learning Network, or NetAcad, you already have a CCO account. Use the same credentials.

---

### Step 2 — Register for CML Free

1. Navigate to **https://mkto.cisco.com/cml-free.html**
2. Sign in with your CCO ID
3. Complete the registration form
4. You will receive a confirmation email with a link to the download page on Cisco Software Central (SWC)

---

### Step 3 — Download the CML Free OVA File

1. Navigate to **https://software.cisco.com/download/home** and search for **CML-Free**, or use the direct link from your registration confirmation email
2. Select the latest CML-Free release
3. Download the **.ova file** — this is the virtual machine image you will import into your hypervisor
4. Also download the **refplat ISO** — this contains the IOS device images (IOL, IOL-L2, ASAv, etc.) that CML needs to run network nodes

> **File sizes:** The OVA is typically 1–2 GB. The refplat ISO is larger (several GB). Plan for a patient download on a slower connection.

---

### Step 4 — Install Your Hypervisor

CML Free runs inside **VMware Workstation Pro** on Windows/Linux, or **VMware Fusion** on Intel-based macOS.

**VMware Workstation Pro (Windows / Linux):**
- As of late 2024, VMware Workstation Pro is **free for personal use** following Broadcom's acquisition of VMware
- Download from: **https://www.vmware.com/products/workstation-pro.html**
- Install with default settings
- Restart your computer after installation

**VMware Fusion (Intel macOS):**
- Also free for personal use
- Download from: **https://www.vmware.com/products/fusion.html**

> **Do not use VirtualBox.** CML requires nested virtualization features that VirtualBox does not support reliably. VMware is the supported and recommended hypervisor for CML Free.

---

### Step 5 — Import the CML OVA into VMware

1. Open VMware Workstation Pro
2. Select **File > Open** (or **Import** on some versions)
3. Browse to the `.ova` file you downloaded and select it
4. Give the VM a name (e.g., `CML-Free`) and choose a storage location with adequate disk space
5. Click **Import** and wait for the process to complete (this may take several minutes)

**Configure the VM before powering on:**
- **RAM:** Allocate at least 8 GB; 12–16 GB is better if your system allows
- **CPU:** Allocate at least 4 vCPUs
- **Network Adapter:** Set to **NAT** — this gives CML internet access through your host machine's connection, which is needed for the initial setup wizard
- **Virtualize Intel VT-x/EPT:** Make sure this checkbox is enabled in the VM's processor settings. This enables the nested virtualization CML requires.

---

### Step 6 — Power On and Run the Setup Wizard

1. Power on the CML VM in VMware
2. Wait for the boot process to complete (first boot takes longer — allow 3–5 minutes)
3. The console will display the CML server's IP address once it's ready — note this down
4. Open a web browser on your **host machine** and navigate to that IP address
5. The CML setup wizard will walk you through:
   - Setting the admin username and password
   - Uploading the refplat ISO (the device image file you downloaded in Step 3)
   - Basic network configuration

> **Tip:** Use `http://` not `https://` for the initial setup if your browser reports a connection error. CML uses a self-signed certificate that some browsers flag on first access.

---

### Step 7 — Log Into the CML Web Interface

Once setup is complete:

1. Open your browser and navigate to the CML server IP address
2. Log in with the admin credentials you created during setup
3. You will land on the **CML Dashboard**

You are now ready to build labs.

---

## Navigating the CML Interface

Here is a brief orientation to the key areas you will use:

**Dashboard:** Your home screen. Shows running labs, resource usage, and quick-start options.

**Lab Canvas:** The main workspace where you drag and drop nodes, draw links, and configure your topology. Double-clicking a node opens its console.

**Node Console:** A terminal window directly into the device. This is where you type IOS commands — exactly as you would on a physical router.

**Tools Menu:** Access to system administration, user management, and the node image library.

**YAML Import/Export:** Every lab in this repository includes a `topology.yaml` file. To import it:
- From the Dashboard, click **Import Lab**
- Select the `.yaml` file
- The topology loads onto the canvas, pre-wired and ready to configure

---

## Troubleshooting Common Installation Issues

### CML VM boots but nodes won't start — "nested virtualization not supported"

This is the most common issue on Windows. The cause is almost always **Hyper-V interfering with VMware**.

To disable Hyper-V on Windows:
1. Open **PowerShell as Administrator**
2. Run the following command:
   ```
   bcdedit /set hypervisorlaunchtype off
   ```
3. Restart your computer
4. Try starting a CML node again

To re-enable Hyper-V later (if needed for other software):
```
bcdedit /set hypervisorlaunchtype auto
```

> **Note:** Windows Subsystem for Linux 2 (WSL2) and Windows Sandbox also use Hyper-V. Disabling Hyper-V will disable these features until re-enabled.

### Browser can't reach the CML web interface

- Confirm the VM is fully booted — check the VMware console for the CML IP address
- Make sure your VM network adapter is set to **NAT**, not Bridged or Host-Only
- Try `http://` instead of `https://`
- Check that your host machine's firewall isn't blocking the connection
- Try navigating to the IP address in an incognito/private browser window

### "5 node limit reached" error when starting a lab

This is expected behavior on CML Free. You can only have 5 nodes **running simultaneously**. Nodes that are defined in a topology but not started do not count. To work within the limit:
- Stop nodes you are not actively using
- Design topologies with 5 or fewer active nodes — all labs in this repository do this by default

### Refplat images not showing up in the node library

If you don't see IOL or other device types after setup, the refplat ISO may not have been uploaded correctly.
1. Go to **Tools > System Administration > Node and Image Definitions**
2. Check whether the image files are listed
3. If not, re-upload the refplat ISO through the administration interface

---

## CML Free vs. Packet Tracer — When to Use Which

You do not have to choose one over the other. Both tools have a place in a complete CCNA study plan.

**Use Packet Tracer when:**
- You are learning a concept for the first time and want a forgiving environment
- You are completing NetAcad course activities
- You need quick, lightweight practice on a low-spec machine
- You are working on IoT or home networking scenarios

**Use CML when:**
- You are working through the labs in this repository
- You want to verify that your understanding holds up against real IOS behavior
- You are practicing troubleshooting with realistic `show` command output
- You are building lab activities to share with others

Think of Packet Tracer as a flight simulator and CML as a full-motion flight trainer. Both are valuable — they serve different moments in the learning journey.

---

## What's Next

Once CML is installed and you can log into the web interface, you are ready to start the lab series.

The recommended starting point is **Static Routing** — it introduces the topology style used throughout this repository (point-to-point links, loopback interfaces, layered verification) and sets the foundation for everything that follows.

Each lab folder contains:
- `lab-guide.md` — the full Build · Observe · Break · Fix walkthrough
- `topology.yaml` — import directly into CML to load the pre-wired topology

**Recommended first labs:**
1. Static Routing
2. OSPF Single Area
3. ACLs
4. NAT
5. IPv6

Good luck — and remember: a broken network is not a failure. It's the beginning of a lesson.

---

## Helpful Resources

| Resource | URL |
|----------|-----|
| CML Free Sign-Up | https://mkto.cisco.com/cml-free.html |
| Cisco Software Central (downloads) | https://software.cisco.com/download/home |
| CML Official Documentation | https://developer.cisco.com/docs/modeling-labs |
| CML System Requirements | https://developer.cisco.com/docs/modeling-labs/system-requirements |
| CML Free Feature Details | https://developer.cisco.com/docs/modeling-labs/cml-free |
| Cisco Learning Network (CML community) | https://learningnetwork.cisco.com |
| VMware Workstation Pro (free personal use) | https://www.vmware.com/products/workstation-pro.html |
| Create a Cisco CCO Account | https://id.cisco.com |

---

*CCNA – Build It · Observe It · Break It · Fix It*
*This repository is an independent educational resource and is not officially affiliated with Cisco Systems.*
*Cisco, CCNA, Cisco Modeling Labs, and Cisco Networking Academy are trademarks of Cisco Systems, Inc.*
