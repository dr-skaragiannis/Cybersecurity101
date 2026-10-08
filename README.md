# Cybersecurity 101 / Introduction to Cybersecurity: Εισαγωγή στην Κυβερνοασφάλεια

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)

> **Languages / Γλώσσες:** [English](#english) | [Ελληνικά](#ελληνικά)

---

<a name="english"></a>
## English

### Overview
**Cybersecurity 101** is a practical, open-source educational repository featuring step-by-step lab guides, penetration testing tutorials, and defensive security walkthroughs. It is designed to take learners from foundational Linux command-line mastery through service enumeration, vulnerability exploitation, network packet analysis, and defensive deception mechanisms.

---

### Repository Contents & Learning Path

| # | Guide / Lab Module | Focus Area | Language |
| :---: | :--- | :--- | :---: |
| **01** | `01-linux-for-beginners-part-01.md` | Linux CLI essentials, file hierarchy, navigation, and core utilities | English |
| **02** | `02-linux-for-beginners-part-02.md` | User management, Linux permissions (`chmod`, `chown`), and process control | English |
| **03** | `03-linux-for-beginners-part-03.md` | Package management, Bash scripting fundamentals, and networking tools | English |
| **04** | `04 - SSH Penetration Testing - Port 22.md` | Auditing Port 22, credential brute-forcing, key authentication, and misconfigurations | English |
| **05** | `05-anonymous-logins-file-shares.md` | Enumerating and exploiting unauthenticated access (FTP, SMB, NFS) | English |
| **06** | `06. Raven_1_Educational_Lab_el.md` | Complete Boot-to-Root educational CTF walkthrough (Raven 1 VM) | Greek |
| **08** | `08. Uncomplicated Firewall - Educational_guide_el.md` | Host-based defense, ingress/egress filtering, and UFW rule configuration | Greek |
| **09** | `09. Wordpress_penetration_testing_lab_guide_el.md` | CMS vulnerability assessment, plugin auditing, WPScan, and exploitation | Greek |
| **10** | `10. Honeypot_lab_guide_el.md` | Deploying deceptive traps, threat telemetry gathering, and attacker monitoring | Greek |
| **11** | `11. Tcpdump_educational_lab_guide_el.md` | Low-level packet capture, Berkeley Packet Filters (BPF), and traffic inspection | Greek |

---

### Prerequisites & Recommended Lab Setup

Most modules are intended to be executed in an isolated virtualization or container environment:

* **Attacking Machine:** Kali Linux, Parrot Security OS, or standard Linux terminal with security tools installed (`nmap`, `hydra`, `wpscan`).
* **Packet & Traffic Analysis:** `tcpdump`, `wireshark` / `tshark`.
* **Hypervisor / Sandboxing:** VirtualBox, VMware Workstation, or UTM (for hosting vulnerable targets like Raven 1).

---

### Quick Start

```bash
# Clone the repository
git clone [https://github.com/dr-skaragiannis/Cybersecurity101.git](https://github.com/dr-skaragiannis/Cybersecurity101.git)
cd Cybersecurity101

# View available guides
ls -l *.md
