---
title: Lab 1 - Kali + Metasploitable Setup and First Scan
nav_order: 2
---

# Lab 1: Building an Isolated Pentest Lab and Running a First Scan
**Dates:** Nov 21-22, 2026

**Tools:** VMware Workstation Pro, Kali Linux, Metasploitable 2, nmap, searchsploit

## Goal
Set up a safe, isolated environment where I can practice attacking and defending on my own machines. For this first lab I wanted to:
 
1. Get Kali Linux running in VMware.
2. Set up a deliberately vulnerable target (Metasploitable 2) that can't touch the internet or my home network.
3. Scan the target, then cross-check what I found against a public exploit database.
Everything here was done against VMs I own, on a private virtual network.

## Lab overview

My host is a Windows PC (Ryzen 7 2700, 32 GB RAM, SSD). Both VMs are stored on my D: drive.
 
| VM | Role | Network adapters |
|---|---|---|
| Kali Linux | Attacker | NAT (internet) + Host-only (lab network) |
| Metasploitable 2 | Target | Host-only only |
 
```
Internet <--NAT--> Kali <--Host-only--> Metasploitable 2
                                (private 192.168.88.x network)
```

**A note on the IP addresses:** I swapped the real addresses in this post for made-up ones. Everything else, including the scan output, is unchanged. Here's what each address in the post stands for:

| Address in this post | What it is |
|---|---|
| `192.168.88.0` / `192.168.88.x` | The Host-only lab network |
| `192.168.88.20` | Metasploitable 2 (the target) |
| `192.168.88.x` on `eth1` | Kali's Host-only adapter |
| `192.168.99.x` on `eth0` | Kali's NAT adapter (internet side) |

**Why it's set up this way:** Host-only is a private virtual network, so the two VMs can talk to each other without anything outside reaching them. Metasploitable only has that adapter, so it has no path to the internet or my home network. Kali has both, which means it's the only machine that can get updates.

## Part 1: Kali setup
 
I downloaded the pre-built Kali VMware image, extracted it, and opened the `.vmx` file in VMware. When VMware asked whether the VM was moved or copied, I picked "I copied it".
 
**Password and snapshots**
 
I changed the default password with `passwd` and took snapshots at clean points (fresh boot, after changing the password, after updates) so I could roll back if I broke something.
 
**Troubleshooting: invisible mouse cursor**
 
The cursor wasn't showing inside Kali. I tried reinstalling the VMware guest tools:
 
```
sudo apt update && sudo apt install --reinstall open-vm-tools-desktop
sudo reboot
```
 
That didn't fix it. I then upgraded the VM's hardware version in VMware, and the cursor showed up afterward. I can't say for sure that the upgrade was the cause, only that the problem went away after it.
 
**Making my own account**
 
Staying on the default `kali` user isn't great practice, so I made my own:
 
```
sudo adduser myuser
groups            # check which groups the account is in
sudo usermod -aG sudo myuser   # add to the sudo group if it's missing
sudo whoami       # should print: root
```
 
`sudo whoami` printing `root` confirms the account can run commands with admin rights by putting `sudo` in front of them. I stay a normal user and only borrow root power when a command needs it, which is safer than working as root all the time.

**Locking the default account**
 
Once my own account worked, I locked the public default one:
 
```
sudo passwd -l kali    # lock
sudo passwd -u kali    # unlock if I ever need it back
```

## Part 2: Metasploitable 2 and the network
 
I downloaded Metasploitable 2, extracted it, and opened it in VMware.

**Before powering it on**, I set one of its network adapters to **Host-only** and removed the other. That's the key safety step since Metasploitable is intentionally vulnerable and should never be reachable from outside the lab.
 
On Kali, I added a second adapter and set it to Host-only and kept the original NAT adapter.

After booting both, I ran `ifconfig` on Metasploitable to get its address: `192.168.88.20`. In Kali, `ip a` showed two interfaces:
 
- `eth0`: `192.168.99.x` (NAT)
- `eth1`: `192.168.88.x` (Host-only, same network as the target)
The third number is what separates the networks, so Kali's `eth1` and the target share a network while the NAT side is separate.

## Part 3: Verifying isolation
 
I didn't want to assume the target was isolated, so I tested it.
 
| Check | Where | Result |
|---|---|---|
| `route -n` | Metasploitable | Only one route, to `192.168.88.0`. No default route, so no path out. |
| `ping -c 3 8.8.8.8` | Metasploitable | "Network is unreachable" |
| Ping my PC's home-network address | Metasploitable | "Network is unreachable" |
| `cat /proc/sys/net/ipv4/ip_forward` | Kali | `0`, so Kali doesn't relay traffic between its two networks |
| Adapter list in VMware | Both | Metasploitable: one Host-only adapter. Kali: NAT + Host-only. |
 
A small mistake along the way: my first ping returned "unknown host" because I'd typed the IP with a trailing period. Retyping it correctly gave the real result.

**What this does and doesn't prove.** The routing table shows where the target is *able* to send traffic, not what it's actually talking to. One caveat is that VMware also creates a virtual adapter on the Windows host for the Host-only network, so my Windows PC can reach the target too (a ping from Windows got replies). That's expected, and it means I should avoid running anything on the host that I wouldn't want a vulnerable box poking at. To see live connections, I can use `netstat -an` on the target or `tcpdump` / Wireshark on Kali.

## Part 4: Scanning the target
 
From Kali:
 
```
nmap -sV -oN scan1.txt 192.168.88.20
```
 
- `-sV` tries to identify the service and version on each open port.
- `-oN scan1.txt` saves the results to a file.
By default nmap checks the 1000 most common ports. The result was 977 closed and 23 open, which adds up to 1000. Scanning all 65,535 ports would need `-p-`, which takes much longer, so this only covers the common ones.

