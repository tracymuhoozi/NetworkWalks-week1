# NetworkWalks-week1
This is for the Cyber Security Internship Project Submission for Week 1
# Week 1 – Cybersecurity Internship (Networkwalks)

**Intern:** <Tracy Muhoozi>
**Programme:** Networkwalks Cybersecurity Internship – Week 1
**Date completed:** <date>

This repository contains my Week 1 work:

1. **Part 1** – Cybersecurity testing lab setup (VirtualBox + Kali Linux)
2. **Part 2** – SOC Wireshark project module (6 labs)

---

## Repository structure

```
.
├── README.md
├── part1-lab-setup/
│   └── screenshots/
└── part2-wireshark-soc/
    └── screenshots/
```

---

## Part 1 – Cybersecurity Lab Setup

### Objective
Build an isolated lab on my own PC with Kali Linux as the attacking machine, using VirtualBox and a NAT Network on `10.0.0.0/24`.

### Requirements and how I met them

| Requirement | What I did | Evidence |
|---|---|---|
| Use VirtualBox (latest recommended version) | Installed VirtualBox <version> | `screenshots/01-virtualbox-version.png` |
| Kali Linux as attacking machine | Created the `kali-lab` VM from the Kali <version> installer ISO | `screenshots/03-kali-vm.png` |
| NAT Network `10.0.0.0/24` | Created `NatNetwork` with IPv4 prefix `10.0.0.0/24`, DHCP enabled | `screenshots/02-natnetwork.png` |
| Clipboard and drag/drop enabled | Set Shared Clipboard and Drag'n'Drop to Bidirectional | `screenshots/04-clipboard-dragdrop.png` |
| Shared folder (Downloads) from host | Added the host Downloads folder with Auto-mount and Make Permanent | `screenshots/05-shared-folder.png` |
| Kali IP `10.0.0.2/24` | Set a manual IPv4 address (gateway `10.0.0.1`, DNS `8.8.8.8`) | `screenshots/06-ip-config.png` |
| Full internet access | Verified with `ping` | `screenshots/07-ping-test.png` |
| Snapshot of the VM | Took snapshot "Fresh Kali – configured" | `screenshots/08-snapshot.png` |

### Steps followed
1. Installed/updated VirtualBox and checked the version (Help > About).
2. Created the NAT Network: File > Tools > Network > NAT Networks, name `NatNetwork`, prefix `10.0.0.0/24`, DHCP enabled.
3. Created the VM from the Kali installer ISO (Linux / Debian 64-bit, 4 GB RAM, 2 CPUs, 50 GB disk) and ran the graphical install.
4. Attached the VM's Adapter 1 to **NAT Network** (`NatNetwork`).
5. Enabled bidirectional clipboard and drag-and-drop; added the host Downloads folder as a shared folder.
6. Installed Guest Additions and added my user to the `vboxsf` group:
   ```bash
   sudo apt update
   sudo apt install -y virtualbox-guest-x11 virtualbox-guest-utils
   sudo usermod -aG vboxsf $USER
   ```
7. Set the static IP in Network Connections: `10.0.0.2`, netmask `24`, gateway `10.0.0.1`, DNS `8.8.8.8`.
8. Verified connectivity:
   ```bash
   ip a
   ping -c 4 10.0.0.1
   ping -c 4 google.com
   ```
9. Took a snapshot of the configured VM.

### Problems I met and how I fixed them
- **NAT Network had the wrong prefix (`10.0.2.0/24`).** It was an existing network from earlier use. I edited it to `10.0.0.0/24` and applied.
- **OS version "Debian (64-bit)" was missing in the New VM wizard.** The wizard had auto-detected Ubuntu from the ISO name; I changed *OS Distribution* to Debian first, then picked the 64-bit version.
- **ISO showed a red sync icon.** It was on OneDrive, so I moved it to a local folder (`C:\VMs\`) before using it.
- <add any internet / shared-folder issue you hit and the fix, e.g. the `nmcli ... ipv4.dad-timeout 0` commands>

### Notes
I already had a Kali VM in VMware Workstation for day-to-day work. For this task I built a separate VirtualBox VM because the assignment specifies VirtualBox.

---

## Part 2 – SOC Wireshark Project Module

### Objective
Investigate six browser-based SOC labs by analysing packet captures in Wireshark, then record findings, filters used and lessons learned.

### Summary

| # | Lab | Main finding | Key technique |
|---|-----|--------------|---------------|
| 1 | Identify the Attacker's IP | TCP SYN **port scan** from one host against one server | SYN filter, Conversations/Endpoints |
| 2 | Follow TCP Stream 1 | A plain-text office chat on **TCP port 5000** exposed a temporary password and Wi-Fi password | Follow TCP Stream |
| 3 | Follow TCP Stream, Part 2 | A hidden **HTML comment** on `/staff.html` leaked a backup server IP and an access code | Follow TCP Stream, reading HTTP responses |
| 4 | HTTP Plain Text Credentials | A sign-in sent over **HTTP POST** exposed the username and password | HTML Form URL Encoded details |
| 5 | Catch the Brute Force Attacker | **FTP brute-force** attack: many `530 Login incorrect` replies, then one `230` success | FTP response/request filters |
| 6 | Catch the Call Scammer | VoIP call: **SIP** set up the call, **RTP** carried the voice | Telephony > VoIP Calls |

### Lab 1 – Identify the Attacker's IP
- **Scenario:** Unusual traffic reported on a company web server.
- **Attacker:** `<attacker IP>`  **Target:** `<target IP>`
- **Filters used:**
  ```
  tcp.flags.syn == 1 && tcp.flags.ack == 0
  ip.src == <attacker IP> && ip.dst == <target IP> && tcp.flags.syn == 1 && tcp.flags.ack == 0
  tcp.flags.syn == 1 && tcp.flags.ack == 1      (open ports)
  tcp.flags.reset == 1                          (closed ports)
  ```
- **Findings:** Total packets `<n>`; SYN packets `<n>`; ports tried `<n>`; open ports `<list>`; closed ports `<n>`. Open ports replied with **SYN, ACK** and closed ports with **RST, ACK**.
- **Conclusion:** TCP SYN port scan (reconnaissance stage).

### Lab 2 – Follow TCP Stream 1
- **Filter used:** `tcp && !(tcp.port == 80) && !(tcp.port == 443)`
- **Findings:** Chat port `5000`; chat started by `<client IP>`; stream number `<n>`; the staff member asking for help was `<first name>`.
- **Lesson:** The chat tool sends everything in clear text, including passwords. Anyone capturing the traffic can read it.

### Lab 3 – Follow TCP Stream, Part 2
- **Filters used:** `http.request`, `http.request.uri == "/staff.html"`, `http.request.uri contains "/download/"`
- **Findings:** Hidden note was an **HTML comment** on `/staff.html` (stream `<n>`); it travelled **server to client**; status code `200`; downloaded file `<file name>`; report prepared by `<name>`.
- **Lesson:** Browsers hide HTML comments, but they are still sent over the network. Never put internal notes or credentials in page source.

### Lab 4 – HTTP Plain Text Credentials
- **Filter used:** `http.request.method == "POST"`
- **Findings:** Sign-in used **HTTP** and the **POST** method to path `<path>`; sent by `<IP>`; `<n>` sign-in attempt(s); username `<username>` found under *HTML Form URL Encoded*.
- **Lesson:** HTTP sends form values unencrypted. Use HTTPS for any login.

### Lab 5 – Catch the Brute Force Attacker
- **Filters used:** `ftp.request.command == "USER"`, `ftp.request.command == "PASS"`, `ftp.response.code == 530`, `ftp.response.code == 230`
- **Findings:** Attacker `<IP>` against FTP server `<IP>`; repeated username `<username>`; failed logins `<n>`; failure code `530`; success code `230`.
- **Lesson:** FTP is unencrypted and has no built-in lockout. Use SFTP/FTPS, strong passwords and account lockout.

### Lab 6 – Catch the Call Scammer
- **Filters used:** `sip`, `sip.Method == "INVITE"`, `rtp`; Telephony > VoIP Calls > Play Streams
- **Findings:** **SIP** (INVITE, Ringing, OK) set up the call; **RTP** carried the voice; calling extension `<ext>`; codec `<codec>`.
- **Lesson:** Unencrypted VoIP can be recorded and replayed. Use SRTP and SIP over TLS. (I have deliberately not recorded any card details in this repository.)

### Wireshark skills I practised
Display filters, Statistics > Conversations/Endpoints/Protocol Hierarchy, Follow TCP Stream, reading packet details, and Telephony > VoIP Calls.

---

## Key lessons
1. Clear-text protocols (HTTP, FTP, plain chat, unencrypted VoIP) expose credentials and sensitive data to anyone who can capture the traffic.
2. Port scans and brute-force attempts have clear patterns in packet captures (many SYNs from one source; many failed logins followed by one success).
3. Filters and statistics tools make large captures manageable.
4. Only test or capture traffic on networks you own or have written permission to test.

## Tools used
VirtualBox, Kali Linux, Wireshark, Firefox, GitHub

## Reflection
<2–4 sentences in your own words: what was hardest, what you learned, what you would like to practise next.>
