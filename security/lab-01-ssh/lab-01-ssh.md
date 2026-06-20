# Secure Device Access: SSH and Hardening

**Build It · Observe It · Break It · Fix It**

---

## Overview

Every network device you have configured in this entire project, you have reached by logging into it. That login is one of the most valuable things on the whole network, because whoever holds it can change anything. So it is worth asking a pointed question. How is that login protected, and who could be watching when you use it?

For years the default answer was Telnet, and Telnet has a fatal flaw. It sends everything, including your username and password, across the network in clear, plain text. Anyone capturing traffic on the path reads your administrator credentials as plainly as you are reading this sentence. That is the exact vulnerability the security fundamentals guide opened with, and this lab is where you close it. **SSH**, the Secure Shell, does the same job as Telnet, giving you a remote command-line session, but it encrypts the entire conversation. An attacker capturing an SSH session sees only scrambled text, known as ciphertext.

But securing the path is only half of device access. The other half is the credentials themselves and the habits around them, and that is where the hardening suite comes in. This lab folds the two together, because the exam and the real world treat them as one job. You will replace Telnet with SSH, protect privileged access with a strongly hashed secret, create a proper local account on the device, encrypt the remaining passwords, drop idle sessions, and post a warning banner. Along the way you will see the single most important password lesson on the CCNA, the difference between a password that merely looks protected and one that actually is.

**CCNA Exam Alignment:**
- 4.8 – Configure network devices for remote access using SSH
- 5.3 – Configure and verify device access control using local passwords

**Estimated Time:** 75–105 minutes

---

## Prerequisites

Before completing this lab, you should be comfortable with:
- Basic IOS navigation, including moving between command modes and reading `show` command output
- IPv4 addressing and verifying connectivity with `ping`
- Reading a device's `running-config` and locating specific lines within it
- The threat, vulnerability, and mitigation vocabulary from the Security Fundamentals concept guide, which this lab puts into practice
- CML fundamentals, including building or importing a topology and accessing a device console

---

## Key Concept: Encryption in Transit, and Strength at Rest

Two distinct security ideas live in this lab, and keeping them separate is what makes the whole thing click.

The first is **protecting access in transit**. This is what SSH does. When you log in across the network, the credentials travel over the wire, and SSH encrypts that journey so a packet capture reveals nothing useful. Telnet does not, which is why a clear-text protocol for management access is simply unacceptable on a real network. SSH needs a few things to work, and forgetting any of them is the most common reason it fails: a hostname, a domain name, an RSA key pair to do the encryption, a local user account to log in with, and the VTY lines configured to accept SSH and refuse Telnet.

The second is **protecting credentials at rest**, meaning how passwords are stored in the configuration itself. Here there is a crucial distinction the exam loves to test. Cisco devices can store a password three ways. Clear text is no protection at all. **Type 7**, applied by the `service password-encryption` command, is weak, reversible encryption that anyone can decode in seconds with a free tool. And strong **hashing**, used by `enable secret` and `username ... secret`, is a one-way function that cannot practically be reversed. The trap is that Type 7 looks encrypted in the configuration, so it feels secure, but it is not. Real credential protection comes from the secret commands, not from `service password-encryption`.

> **Get ahead of the confusion:** `service password-encryption` is worth enabling, but understand exactly what it does and does not do. It hides clear-text passwords from someone glancing over your shoulder at the config, and that is all. It does not protect anything against a person who copies the config and runs a Type 7 decoder. For anything that truly matters, the enable password and user accounts, you use `secret`, which hashes. You will prove this difference to yourself in Break Scenario 3.

---

## Topology

```
           [PC1]                       [R1]
       192.168.1.2                 192.168.1.1
     Admin workstation           Device being secured
     (SSH client, Linux)         (SSH server + hardening)
          |                           |
          +-----------[SW1]-----------+
```

**Nodes (3 total, within CML Free Edition limit):**
- R1 (IOSv router, the device you will secure)
- PC1 (a Linux host acting as the administrator's workstation and SSH client)
- SW1 (IOSvL2 switch)

> **A note on interface names and the host client:** Port names vary by image, so confirm with `show ip interface brief` and adjust. R1 is the device under configuration throughout. PC1 is the management workstation, the way a real administrator reaches network gear, and it exists to prove that SSH works and that Telnet no longer does. The SSH client command shown in this lab is the standard Linux syntax. If your host image differs, adjust accordingly, and if it has no SSH client at all, the fallback note in Part 2 shows how to test from another router instead.

---

## Addressing Table

| Device | Interface | IP Address    | Subnet Mask     | Role |
|--------|-----------|---------------|-----------------|------|
| R1     | G0/0      | 192.168.1.1   | 255.255.255.0   | Device being secured |
| PC1    | NIC       | 192.168.1.2   | 255.255.255.0   | Admin workstation / SSH client |

---

## Lab Objectives

By the end of this lab you will be able to:

1. Explain why SSH replaces Telnet for remote management
2. Configure SSH, including the RSA key pair and VTY line settings
3. Configure strong privileged access with `enable secret` and a local user account
4. Apply password and session hardening, including `service password-encryption`, `exec-timeout`, and a login banner
5. Distinguish weak Type 7 encryption from strong hashing in the configuration
6. Verify secure access and confirm that Telnet is refused
7. Troubleshoot the most common SSH and device access failures

---

## 🧱 Part 1: Build It

### Step 1 — Configure Interfaces

Bring up R1 on the LAN, give PC1 its address, and confirm they can ping each other, since you will be logging in from PC1 to R1 across this connection.

```
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip address 192.168.1.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit
```

Configure PC1 with the address 192.168.1.2 and a subnet mask of 255.255.255.0 using its host networking settings.

### Step 2 — Set the Identity SSH Requires

SSH needs the device to have a hostname and a domain name, because together they name the RSA key. These are prerequisites and not optional.

```
R1(config)# hostname R1
R1(config)# ip domain name lab.local
```

### Step 3 — Protect Privileged Access With a Strong Secret

Guard entry into privileged exec mode with a strongly hashed secret. Use `enable secret`, never `enable password`, and the reason will become vivid in Break Scenario 3.

```
R1(config)# enable secret StrongEnable!23
```

### Step 4 — Create a Local User Account

SSH logins are checked against the local user database, so create an account with a hashed secret. This account is what you will actually log in as.

```
R1(config)# username admin secret AdminPass!45
```

> **Note the `secret` keyword on the username.** Just like the enable secret, this stores the password as a strong one-way hash rather than weak Type 7. Always prefer `username ... secret` over the older `username ... password`.

### Step 5 — Generate the RSA Key Pair

This is the cryptographic heart of SSH. The key pair is what encrypts the session, and SSH cannot function without it.

```
R1(config)# crypto key generate rsa modulus 2048
```

> **Why 2048:** The modulus is the key length in bits, and longer is stronger. A 2048-bit key is the sensible modern minimum. Anything below 768 will not even support SSH version 2. Once this command runs, the device has the keys it needs to encrypt sessions.

### Step 6 — Force SSH Version 2

Version 2 fixed real security weaknesses found in version 1. Require it explicitly.

```
R1(config)# ip ssh version 2
```

### Step 7 — Lock Down the VTY Lines

The VTY lines are the virtual terminals used for remote access. Configure them to authenticate against the local account and, critically, to accept only SSH. This last setting is what actually closes the Telnet door.

```
R1(config)# line vty 0 4
R1(config-line)# login local
R1(config-line)# transport input ssh
R1(config-line)# exec-timeout 5 0
R1(config-line)# exit
```

> **`transport input ssh` is the most important security line in this lab.** It tells the VTY lines to accept SSH and nothing else. Without it, the lines may still accept Telnet, which means you built a secure door and left the insecure one wide open right beside it. You will see exactly that mistake in Break Scenario 2. The `exec-timeout 5 0` logs out an idle session after five minutes, so an unattended terminal does not stay open forever.

### Step 8 — Encrypt the Remaining Passwords

Apply `service password-encryption` so any clear-text passwords in the config are not shown in the open.

```
R1(config)# service password-encryption
```

> **Useful, but know its limits.** This applies only weak Type 7 encoding, good for hiding passwords from a casual glance and nothing more. Your important credentials are already protected by the stronger `secret` hashing from earlier steps. This command is a supplement, not your main defense.

### Step 9 — Harden the Console and Post a Banner

Secure the console line the same way, and post a legal warning banner that appears before login.

```
R1(config)# line console 0
R1(config-line)# login local
R1(config-line)# exec-timeout 5 0
R1(config-line)# exit

R1(config)# banner motd # Authorized access only. Violators will be prosecuted. #
```

> **A banner is a real control, not decoration.** A clear warning that access is restricted has genuine legal weight, and it removes any claim that a "confused" network intruder did not know the system was private. Keep banners stern and avoid the word "welcome'.

### Step 10 — Save the Configuration

```
R1# copy running-config startup-config
```

---

## 👀 Part 2: Observe It

### 2.1 — Confirm SSH Is Ready

```
R1# show ip ssh
```

**What to look for:** SSH enabled, and version 2.0 in force. This confirms the key generated and the version was set.

### 2.2 — Log In Securely From PC1

This is the headline test, and it is the moment students remember. Open a terminal on PC1, the admin workstation, and SSH into R1 using the standard Linux client:

```
ssh admin@192.168.1.1
```

The first time you connect, the client will ask you to accept R1's key fingerprint, which is the host confirming its identity. Type yes, then enter the account password when prompted. You are now logged into R1 over an encrypted channel, from a workstation, exactly the way an administrator reaches network gear in the real world.

> **Syntax and fallback note:** The command above is standard Linux. The form `ssh -l admin 192.168.1.1` is equivalent if your client prefers it. If your host image has no SSH client at all, you can prove the configuration from another router instead with `ssh -l admin 192.168.1.1` in IOS, which is a reliable fallback regardless of host image. The destination and the result are the same either way.

**✏️ Prediction checkpoint:** Before you connect, predict which account and password R1 will check your login against, and why the local user database is involved.

### 2.3 — Confirm Telnet Is Refused

Now prove the insecure door is closed. From PC1's terminal, try Telnet to R1:

```
telnet 192.168.1.1
```

This should be refused or simply fail to open a login, because the VTY lines accept only SSH. A refused Telnet attempt is the visible proof that `transport input ssh` did its job. If PC1's image has no telnet client, attempt the same from a router with `telnet 192.168.1.1` to see the refusal.

### 2.4 — See Strong Versus Weak Storage in the Config

This is the lesson worth slowing down for. Look at how the passwords are stored:

```
R1# show running-config | include secret|username|enable
```

**What to look for:** The `enable secret` and the `username admin secret` both appear as long, scrambled hashes. These are strong, one-way, and not practically reversible. Keep this picture in mind, because in Break Scenario 3 you will see what a weak Type 7 password looks like by comparison, and why the difference matters so much.

### 2.5 — View Active Sessions

While your SSH session from PC1 is open, view it from R1:

```
R1# show ssh
```

This lists the active encrypted sessions, including the version and the user. It is how you see who is currently connected over SSH.

---

## 💥 Part 3: Break It

> Work through each scenario one at a time. Document the symptoms fully before you move to Part 4.

---

### Break Scenario 1 — SSH With No Key

SSH cannot encrypt without its key pair. Remove it and watch SSH stop working:

```
R1(config)# crypto key zeroize rsa
```

Confirm, then try to connect from PC1:
```
R1# show ip ssh
```
Then, from PC1's terminal:
```
ssh admin@192.168.1.1
```

**✏️ Document the symptoms:**
- What does `show ip ssh` report now compared to before?
- Can PC1 establish an SSH session anymore?
- The username, the VTY lines, and the connectivity are all still perfect, so why does SSH fail completely?

> **Why this matters:** The RSA key is what performs the encryption, so without it there is nothing to secure the session and SSH simply will not run. This is one of the most common SSH failures in the field, and it usually traces back to a missing key or, just as often, a missing domain name that prevented the key from generating in the first place. When SSH refuses to work and everything else looks right, check for the key and the hostname and domain that the key depends on.

---

### Break Scenario 2 — The Door Left Open

Regenerate the key, then make the classic mistake of allowing Telnet back onto the VTY lines:

```
R1(config)# crypto key generate rsa modulus 2048
R1(config)# line vty 0 4
R1(config-line)# transport input all
R1(config-line)# exit
```

> `transport input all` lets the VTY lines accept Telnet again alongside SSH.

Now, from PC1's terminal, try the insecure protocol:
```
telnet 192.168.1.1
```

**✏️ Document the symptoms:**
- Does Telnet connect to R1 now?
- SSH still works perfectly, so is the device "secure" or not?
- What is the real risk of leaving Telnet enabled even when SSH is also available?

> **Why this matters:** This is the subtlest and most dangerous mistake in the lab, because nothing appears broken. SSH works, you can log in securely, and everything feels fine. But Telnet is wide open right next to it, and an attacker will always choose the clear-text door. Adding a secure option does not remove an insecure one. You have to explicitly close Telnet with `transport input ssh`, and forgetting that step is how networks end up with strong front doors and unlocked back ones.

---

### Break Scenario 3 — The Password That Only Looks Safe

Restore SSH-only access. Then make the password mistake the exam cares about most, using `enable password` instead of `enable secret`:

```
R1(config)# line vty 0 4
R1(config-line)# transport input ssh
R1(config-line)# exit

! The weak choice: enable password instead of enable secret
R1(config)# enable password WeakEnable99
```

Now look at how that password is stored, alongside the strong one:
```
R1# show running-config | include enable
```

**✏️ Document the symptoms:**
- The `enable password` appears encrypted thanks to `service password-encryption`, but as which type?
- How does its stored form compare to the `enable secret` hash you saw earlier?
- If someone obtained a copy of this configuration file, which of the two passwords could they recover, and which could they not?

> **Why this matters:** With `service password-encryption` on, the `enable password` is stored as Type 7, which looks encrypted but is trivially reversible with a free decoder. The `enable secret` next to it is a strong hash that cannot practically be reversed. This is the whole lesson of credential storage in one screen. Type 7 stops a shoulder surfer, hashing stops an attacker with your config file. Always use `enable secret`, and when both an enable password and an enable secret are configured, the secret is the one that actually takes effect, which is one more reason the weak command has no good use.

---

## 🔧 Part 4: Fix It

---

### Fix Scenario 1 — SSH With No Key

**Your troubleshooting toolkit:**
```
show ip ssh
show running-config | include domain
```

**Structured process:**
1. Confirm SSH is not running and no session can be established.
2. Verify the hostname and domain name are set, since the key depends on them.
3. Generate the RSA key pair.
4. Confirm SSH comes back.

**✅ Solution:**
```
R1(config)# ip domain name lab.local
R1(config)# crypto key generate rsa modulus 2048
```

**Verification:**
```
R1# show ip ssh
```
Then from PC1's terminal: `ssh admin@192.168.1.1`

---

### Fix Scenario 2 — The Door Left Open

**Your troubleshooting toolkit:**
```
show running-config | section vty
```

**Structured process:**
1. Recognize that Telnet is still reachable even though SSH works.
2. Inspect the VTY lines and their `transport input` setting.
3. Restrict the lines to SSH only.
4. Confirm Telnet is now refused.

**✅ Solution:**
```
R1(config)# line vty 0 4
R1(config-line)# transport input ssh
R1(config-line)# exit
```

**Verification:** from PC1's terminal:
```
telnet 192.168.1.1        (should be refused)
ssh admin@192.168.1.1     (should succeed)
```

---

### Fix Scenario 3 — The Password That Only Looks Safe

**Your troubleshooting toolkit:**
```
show running-config | include enable
```

**Structured process:**
1. Identify the weak Type 7 enable password in the configuration.
2. Remove it so it cannot be recovered from the config.
3. Rely on the strongly hashed `enable secret` instead.
4. Confirm only the hashed secret remains.

**✅ Solution:**
```
R1(config)# no enable password
R1(config)# enable secret StrongEnable!23
```

**Verification:**
```
R1# show running-config | include enable
```

---

## 💬 Reflection Questions

1. Explain why Telnet is unacceptable for managing a device on a real network, and what specifically SSH does to fix the problem. Tie your answer to the threat, vulnerability, and mitigation vocabulary from the fundamentals guide.

2. List the prerequisites SSH needs before it will function. If SSH fails to start, which of these would you check first, and why?

3. In Break Scenario 2, SSH worked perfectly and Telnet was still open. Explain why adding a secure access method does not make a device secure on its own, and what command actually closes the insecure path.

4. Describe the difference between Type 7 encryption and the hashing used by `enable secret`. Which protects against someone who has obtained your configuration file, and which does not?

5. Both `enable password` and `enable secret` set the privileged exec password. When both are present, which one takes effect, and why does the other have no good reason to exist?

6. A login banner has no technical effect on access at all. Explain why it is still considered a genuine security control, and what makes a banner effective versus counterproductive.

---

## Device Hardening Quick Reference

A scannable summary of the access-control and hardening commands in this lab. Use it before the exam, or when setting up a real device.

| Command | What it does | Why it matters |
|---------|--------------|----------------|
| `hostname` and `ip domain name` | Name the device and its domain | Required before an RSA key can be generated for SSH |
| `crypto key generate rsa modulus 2048` | Create the RSA key pair | The cryptographic basis of SSH. No key, no SSH |
| `ip ssh version 2` | Force SSH version 2 | Version 1 has known weaknesses. Always require version 2 |
| `enable secret` | Set a strongly hashed privileged password | Strong, one-way protection for enable mode. Always use this over enable password |
| `username ... secret` | Create a locally hashed user account | Strong credential storage and the account SSH logs in against |
| `line vty 0 4` then `login local` | Authenticate remote logins against the local accounts | Ties SSH access to your hashed user database |
| `transport input ssh` | Allow only SSH on the VTY lines | Closes the Telnet door. The key line that makes access actually secure |
| `service password-encryption` | Apply Type 7 encoding to clear-text passwords | Hides passwords from a casual glance. Weak and reversible, so a supplement only |
| `exec-timeout 5 0` | Log out idle sessions after five minutes | Prevents an unattended terminal from staying open |
| `banner motd` | Display a warning before login | A legal control that asserts the system is private |
| `login block-for 60 attempts 3 within 30` | Throttle repeated failed logins | Slows brute-force password attacks against the device |

---

## Instructor Notes

**Common Student Mistakes**
- Trying to generate the RSA key before setting the hostname and domain name, so the key generation is rejected. This is the number one reason SSH will not come up in student labs. Have them set identity first, every time.
- Configuring SSH correctly but leaving `transport input` as `all` or `telnet`, so the insecure path stays open. This is the most important security point in the lab, and students consistently miss it because nothing looks broken.
- Believing `service password-encryption` provides real security. Make sure they leave understanding it is reversible Type 7, not protection for anything that matters.
- Confusing `enable password` and `enable secret`, or setting both and not knowing the secret wins. Drive home that the password form has no good use.
- Forgetting `login local` on the VTY lines, so SSH has no user database to authenticate against and logins fail even though the rest is correct.

**Teaching Tips**
- Do the Type 7 reveal live. Show a Type 7 password in the config, then mention that a free decoder reverses it in seconds, and contrast it with the secret hash sitting right next to it. Seeing weak and strong storage side by side in one screen lands the lesson harder than any slide.
- Have every student attempt a Telnet to the hardened device and watch it get refused. The visible proof that the old, insecure door is now closed is far more convincing than telling them it is closed, and it cements why `transport input ssh` matters.

---

## Where This Leads

You have secured how you get into your devices. The login that controls everything now travels encrypted, the credentials that matter are hashed beyond practical recovery, idle sessions close themselves, and the clear-text door is bolted shut. Just as valuable, you can now look at a configuration and tell at a glance which passwords are genuinely protected and which only look that way, which is a skill that will serve you on the exam and in every config review you ever do.

Securing access to a device assumes the attacker is coming across the network to log in. But what about the attacker who simply walks up and plugs a laptop into a switch port, or floods that port to overwhelm the switch? Securing the management plane does nothing about the physical access layer where users and their devices actually connect. That is the next layer of defense in depth, and it is where the Security section goes next, with Port Security controlling exactly which devices are allowed to use a switch port at all.

---

*CCNA – Build It · Observe It · Break It · Fix It series*
*Next: Port Security*

*This repository is an independent educational resource and is not officially affiliated with Cisco Systems. Cisco, CCNA, Cisco Modeling Labs, and Cisco Networking Academy are trademarks of Cisco Systems, Inc.*
