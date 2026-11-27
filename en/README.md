# CSEC 101 — Cybersecurity Laboratory Material

A collection of laboratory exercises that progressively develop core skills, from basic use of the command line through to complete penetration tests against intentionally vulnerable virtual machines.

> 🇬🇷 The original Greek edition of this material is available in the [repository root](../README.md).

---

## Contents

| # | Lab | Topic | Level | Documentation |
| :--- | :--- | :--- | :--- | :--- |
| 00 | **Virtualization** | Hypervisors, OVA import, snapshots, VM networking modes | Beginner | [README.md](00.%20Virtualization/README.md) |
| 01 | **Introduction to the Linux CLI** | File-system navigation, commands, networking, permissions | Beginner | [README.md](01.%20Introduction%20to%20the%20Linux%20CLI/README.md) |
| 02 | **Bandit** | OverTheWire wargame: files, pipes, networking, cron, SUID, git, shell escapes | Beginner | [README.md](02.%20Bandit/README.md), [PDF (Greek)](../02.%20Bandit/Lab%2002%20-%20Bandit.pdf) |
| 03 | **Raven-CTF** | Write-up: reconnaissance, WordPress, SSH, MySQL, privilege escalation | Beginner | [README.md](03.%20Raven-CTF/README.md), [Hardening](03.%20Raven-CTF/Hardening-Guide.md) |
| 05 | **DC-CTF** | Walkthrough: Drupalgeddon (CVE-2014-3704) → SUID `find` → root | Beginner | [README.md](05.%20DC-CTF/README.md), [Hardening](05.%20DC-CTF/Hardening-Guide.md) |
| 06 | **Raven2-CTF** | Write-up: PHPMailer RCE (CVE-2016-10033) → MySQL UDF → SUID `find` | Intermediate | [README.md](06.%20Raven2-CTF/README.md), [Hardening](06.%20Raven2-CTF/Hardening-Guide.md) |
| 07 | **Mr Robot** | Write-up: WordPress enumeration → reverse shell → SUID nmap | Intermediate | [README.md](07.%20Mr%20Robot/README.md), [Hardening](07.%20Mr%20Robot/Hardening-Guide.md) |
| 08 | **Kioptrix** | Walkthrough: service enumeration → patching and compiling a legacy mod_ssl exploit → root | Beginner | [README.md](08.%20Kioptrix/README.md), [Hardening](08.%20Kioptrix/Hardening-Guide.md) |

---

## Folder Structure

```
en/
├── README.md                        ← this file (index)
├── 00. Virtualization/
│   ├── README.md                    Hypervisors, OVA/OVF, snapshots, VM networking
│   └── Detailed-Notes.md            Detailed notes with theory
├── 01. Introduction to the Linux CLI/
│   ├── README.md                    Theory + four-part exercise with questions
│   └── Detailed-Notes.md            Detailed notes + answers
├── 02. Bandit/
│   └── README.md                    Walkthrough of 34 levels with explanations
├── 03. Raven-CTF/
│   ├── README.md                    Eight-step write-up
│   ├── Detailed-Notes.md            Detailed notes for each step
│   └── Hardening-Guide.md           Hardening: SSH, WordPress, MySQL, sudo
├── 05. DC-CTF/
│   ├── README.md                    Four-phase manual + remediation measures
│   ├── Detailed-Notes.md            Detailed notes for each phase
│   └── Hardening-Guide.md           Hardening: Drupal, PHP, egress, SUID, mount options
├── 06. Raven2-CTF/
│   ├── README.md                    Six-section write-up
│   ├── Detailed-Notes.md            Detailed notes for each section
│   └── Hardening-Guide.md           Hardening: PHPMailer, web root, MySQL (UDF, FILE, root)
├── 07. Mr Robot/
│   ├── README.md                    Write-up: six phases, 32 figures
│   └── Hardening-Guide.md           Hardening: login, Theme Editor, PHP, hashes, SUID nmap
└── 08. Kioptrix/
    ├── README.md                    Video-based walkthrough: enumeration, exploit patching, root
    └── Hardening-Guide.md           Hardening: OS/package upgrades, Apache/mod_ssl, Samba
```

Screenshots and PDFs are not duplicated: the English documents reference the images and PDFs stored in the corresponding folders of the original (Greek) edition. The PDFs are available in Greek only.

---

## 00. Virtualization

An introduction to virtualization and the practical configuration of **Oracle VM VirtualBox**: how the hypervisor operates beneath the virtual machines, how a pre-built image is imported, and how the laboratory network is set up correctly. This is a prerequisite for all subsequent labs, since both the attacking machine (Kali) and the targets (Raven 1, DC-1, Raven 2) run as VMs.

**Topics covered:**

- **Hypervisors:** the distinction between **Type 1 (bare-metal** — ESXi, KVM, Xen, Proxmox) and **Type 2 (hosted** — VirtualBox, VMware Workstation) hypervisors, with hardware support from **Intel VT-x/EPT** and **AMD-V/NPT**. A diagram of the host → hypervisor → guest VMs stack is included.
- **OVF/OVA packages:** the **Open Virtualization Format** (DMTF DSP0243), and why an **OVA** is simply the same package bundled into a single `tar` file — containing an integrity manifest (`.mf`), a hardware description (`.ovf`) and disk images (`.vmdk`/`.vhd`).
- **Importing an OVA into VirtualBox:** `File → Import Appliance…` (`Ctrl+I`), reviewing CPU/RAM allocation (up to approximately 50–60% of host memory), choosing the storage folder, and selecting **Generate new MAC addresses** when importing a clone.
- **Snapshots and disks:** copy-on-write **differencing images**, why a snapshot is not a backup, and why long snapshot chains degrade machine performance.
- **Networking modes:** a detailed comparison of **NAT**, **NAT Network**, **Host-Only** and **Bridged**, with step-by-step creation of a shared NAT Network (`Tools → Network → NAT Networks`).

**Network mode selection table (summary):**

| Mode | VM → Internet | Host → VM directly | VM ↔ VM | Typical use |
| :--- | :---: | :---: | :---: | :--- |
| **NAT** (default) | Yes | No (port forwarding) | No | A single machine with Internet access |
| **NAT Network** | Yes | No (port forwarding) | Yes | Multi-node labs that require updates |
| **Host-Only** | No | Yes | Yes | Isolated labs, malware analysis |
| **Bridged** | Yes | Yes | Yes | The VM as a peer device on the physical LAN |

> ⚠️ **NAT ≠ NAT Network.** The plain **NAT** option gives each machine its own private subnet, so two VMs cannot see each other. For a lab with multiple machines, select **Attached to: NAT Network** together with the name of the network you created.

**Quick start:** download the pre-built **Kali Linux** image for VirtualBox from [kali.org/get-kali](https://www.kali.org/get-kali/#kali-platforms) (on macOS, UTM is preferable), make sure that virtualization is enabled in the BIOS/UEFI (Intel VT-x / AMD-V), and import the OVA — no installation is required.

📘 **Detailed notes:** [Detailed-Notes.md](00.%20Virtualization/Detailed-Notes.md) — theory, analysis of every step, troubleshooting tables, a glossary and self-assessment questions.

---

## 01. Introduction to the Linux CLI

A laboratory exercise intended to familiarize students with the command-line environment and essential networking commands.

📘 **Detailed notes:** [Detailed-Notes.md](01.%20Introduction%20to%20the%20Linux%20CLI/Detailed-Notes.md) — an explanation of every command, networking theory, **answers to the questions** and a glossary.

**Topics covered:**

- **Part A — Navigation:** `pwd`, `mkdir -p`, `cd` with absolute and relative paths, `cd ..`, `cd ../..`, `cd -`, as well as `ls` with the options `-l`, `-a`, `-h`, `-t`, `-r`, `-R` and wildcards.
- **Part B — Filtering and pipes:** `grep`, output redirection (`>`), piping (`|`), `wc`.
- **Part C — Network diagnostics:** `ifconfig` / `ip addr show`, `ping`, `ss -tuln`, `nslookup`, `traceroute`.
- **Part D — File permissions:** `chmod +x`, the numeric form `755`, and the owner/group/others model.

**Deliverable:** answers to questions 1.1, 1.2, 2.1, 3.1, 3.2 and 4.1, which appear within the text.

**Requirements:** any Linux distribution (or VM), basic networking commands, and `sudo` privileges for certain operations.

---

## 02. Bandit

An enriched edition of the walkthrough for the **Bandit** wargame by the OverTheWire community: 34 levels on a real Linux server, in which you connect via SSH at each level and search for the password of the next user.

📘 **Full text:** [README.md](02.%20Bandit/README.md) · **PDF:** [Lab 02 - Bandit.pdf (Greek)](../02.%20Bandit/Lab%2002%20-%20Bandit.pdf)

**Topics covered:**

- **Files and filenames (0–6):** `ls`, `cat`, hidden files, special filenames, `file`, `find`.
- **Text processing (7–12):** `grep`, `sort | uniq`, `strings`, Base64, ROT13, compression.
- **Networking and keys (13–20):** SSH keys, `nc`, `openssl s_client`, port scanning, `diff`, SUID.
- **Automation (21–24):** `cron`, bash scripts, brute-force search.
- **Shell escapes and Git (25–33):** escaping via `more` / `vi`, history, branches, tags, `$0`.

**Source:** [OverTheWire — Bandit](https://overthewire.org/wargames/bandit/), [Write-up: A. Cheron](https://axcheron.github.io/writeups/otw/bandit/)

---

## 03. Raven-CTF

A translated and enriched write-up for the **Raven 1** machine from VulnHub.

📘 **Detailed notes:** [Detailed-Notes.md](03.%20Raven-CTF/Detailed-Notes.md) — methodology, analysis of every step and figure, common pitfalls and self-assessment questions.

🛡️ **Hardening guide:** [Hardening-Guide.md](03.%20Raven-CTF/Hardening-Guide.md) — firewall, Apache, WordPress, key-based SSH with PAM, least-privilege MySQL, correcting the `sudo` configuration.

**Solution path:**

1. **Network reconnaissance** — `netdiscover` / `nmap -sn` to identify the target's IP address.
2. **Port scanning** — `nmap -sC -sV` (SSH, HTTP and one additional service).
3. **Web page analysis** — Developer Tools (F12), HTML comments → **Flag 1**.
4. **Directory fuzzing** — DirBuster / Gobuster → discovery of the `wordpress` directory.
5. **WPScan** — WordPress version, plugins, user enumeration (`-e u`).
6. **SSH access** — brute force with `hydra` using the `rockyou.txt` wordlist → weak password (`michael:michael`) → **Flag 2** in `/var/www/html/`.
7. **Database** — `wp-config.php` → MySQL → `wp_posts` → **Flags 3 and 4**.
8. **Root** — hashes from `wp_users` → John the Ripper (`rockyou.txt`) → `steven` → `root`.

**Tools:** `netdiscover`, `nmap`, `DirBuster`/`gobuster`, `WPScan`, `hydra`, `ssh`, `mysql`, `john`.
**Source:** [VulnHub — Raven: 1](https://www.vulnhub.com/entry/raven-1,256/), [Write-up: Hariharan](https://corruptedprotocol.medium.com/raven-1-ctf-write-up-1ddaf2cff9e2)

---

## 05. DC-CTF

A complete penetration-testing manual for the **DC-1** machine, structured in four methodological phases.

📘 **Detailed notes:** [Detailed-Notes.md](05.%20DC-CTF/Detailed-Notes.md) — theory of Drupalgeddon, Metasploit, PTYs, SUID and CVSS, with a glossary and questions.

🛡️ **Hardening guide:** [Hardening-Guide.md](05.%20DC-CTF/Hardening-Guide.md) — Drupal upgrade/hotfix, `php.ini`, egress filtering, SUID and `dpkg-statoverride`, `nosuid/noexec`.

**Phases:**

1. **Reconnaissance and Service Discovery**
   - Isolated virtual network (Host-Only / NAT).
   - `netdiscover -r 192.168.35.0/24` — host discovery via ARP.
   - `nmap -sV -p- 192.168.35.131` — full scan of all 65,535 ports.
   - Identification of Apache / Debian Wheezy / **Drupal 7.x**.
2. **Vulnerability Analysis and Exploitation**
   - `msfconsole` → `search drupal` → `exploit/multi/http/drupal_drupageddon`.
   - **CVE-2014-3704** (SQL injection → RCE), CVSS **9.8 Critical**.
   - Opening a Meterpreter session.
3. **Post-Exploitation**
   - `shell` → upgrade to a PTY: `python -c 'import pty; pty.spawn("/bin/bash")'`.
   - Locating `flag1.txt` in `/var/www` and the hints pointing to `settings.php`.
4. **Privilege Escalation**
   - `find / -perm -u=s -type f 2>/dev/null` — identification of SUID binaries.
   - Abuse of the SUID bit on `/usr/bin/find`: `/usr/bin/find . -exec /bin/sh \; -quit`.
   - Verification with `whoami` and retrieval of `thefinalflag.txt` from `/root`.

**Technical summary and remediation:** a risk table (CVSS v3) and hardening measures — patch management, WAF/ModSecurity with the OWASP CRS, removal of the SUID bit (`chmod u-s /usr/bin/find`), integrity monitoring (AIDE/Tripwire), AppArmor/SELinux.

**Tools:** `netdiscover`, `nmap`, Metasploit Framework, Meterpreter, `find`, `cat`, `whoami`.
**Source:** [VulnHub — DC: 1](https://www.vulnhub.com/entry/dc-1,292/), [Write-up: Hacking Articles](https://www.hackingarticles.in/dc-1-vulnhub-walkthrough/)

---

## 06. Raven2-CTF

A penetration-testing write-up for the **Raven 2** machine (VulnHub), rated *Beginner/Intermediate*, with the objective of capturing four flags and obtaining root privileges.

📘 **Detailed notes:** [Detailed-Notes.md](06.%20Raven2-CTF/Detailed-Notes.md) — PHPMailer RCE, MySQL UDF and SUID, with a detailed explanation of every command.

🛡️ **Hardening guide:** [Hardening-Guide.md](06.%20Raven2-CTF/Hardening-Guide.md) — post-compromise clean-up, PHPMailer 6 via Composer, running MySQL as `mysql`, `secure_file_priv`, `plugin_dir`.

**Solution path:**

1. **Network reconnaissance** — `netdiscover` to locate the target (`192.168.1.101`), `nmap -A` for OS and version detection (22 SSH, 80 HTTP, 111 RPC).
2. **Web enumeration** — `dirb` → the `/vendor/` directory, where the `PATH` file contains **Flag 1** and the `VERSION` file reveals **PHPMailer 5.2.6**.
3. **Initial access** — Exploit-DB **40974** / **CVE-2016-10033** (PHPMailer < 5.2.18 RCE) in Python: `nc -lvp 4444` for the reverse shell, upgrade to a TTY with `pty.spawn`, retrieval of **Flag 2** from `/var/www`.
4. **System enumeration** — `wp-config.php` → MySQL credentials (`R@v3nSecurity`), `LinEnum.sh` to identify weaknesses.
5. **MySQL UDF (CVE-2016-6662)** — compiling `1518.c` (`gcc -shared`), loading it as a plugin via `dumpfile`, and calling `do_system('chmod u+s /usr/bin/find')`.
6. **Root** — `find raj -exec "/bin/sh" \;` → root shell, **Flag 4** in `/root` and **Flag 3** in `wp-content/uploads/2018/11`.

**Tools:** `netdiscover`, `nmap`, `dirb`, Exploit-DB / `searchsploit`, `nc`, MySQL client, `gcc`, `find`.
**Source:** [VulnHub — Raven: 2](https://www.vulnhub.com/entry/raven-2,269/), [Write-up: Hacking Articles](https://www.hackingarticles.in/raven-2-vulnhub-walkthrough/)

---

## 07. Mr Robot

A complete write-up for the **Mr. Robot** machine (TryHackMe): from WordPress enumeration through to `root`, with 32 screenshots, an analysis of every command and an explanation of every figure.

📘 **Full text:** [README.md](07.%20Mr%20Robot/README.md)

🛡️ **Hardening guide:** [Hardening-Guide.md](07.%20Mr%20Robot/Hardening-Guide.md) — uniform login error messages, brute-force limits, `DISALLOW_FILE_EDIT`, `disable_functions`, password hashes, SUID nmap.

**Solution path:**

1. **Reconnaissance** — `nmap -sV -T4` → only ports 80/443, Apache httpd.
2. **Enumeration** — `gobuster` → `/wp-login`, `/robots.txt`; `robots.txt` yields **Key 1** and the `fsocity.dic` wordlist.
3. **Wordlist clean-up** — `sort | uniq` reduces it from 858,160 to **11,451** words (−98.6%).
4. **Credentials** — distinct error messages (user enumeration) + Burp Proxy + `hydra` (two passes) → `elliot:ER28-0652`.
5. **Initial access** — Theme Editor → PHP reverse shell (`archive.php`) → `nc -lnvp 1234` → shell as `daemon`.
6. **Pivoting** — `password.raw-md5` → CrackStation/`john` → `su robot` (with `pty.spawn`) → **Key 2**.
7. **Privilege escalation** — SUID `/usr/local/bin/nmap` → GTFOBins `--interactive` + `!sh` → root → **Key 3**.
8. **Alternatives** — Burp Intruder, `wpscan`, John the Ripper, Hashcat.

**Tools:** `nmap`, `gobuster`, Burp Suite, `hydra`, `wpscan`, `nc`, `john`, `hashcat`, GTFOBins.
**Source:** [Mr Robot CTF Walkthrough (2023) — Charalampos Spanias](https://medium.com/@cspanias/thms-mr-robot-ctf-walkthrough-2023-55ca5c19fbaf), [TryHackMe room](https://tryhackme.com/room/mrrobot)

---

## 08. Kioptrix

A video-based walkthrough for the **Kioptrix: Level 1** machine (VulnHub). Rather than a simple flag hunt, it emphasizes the mindset of a real penetration test — searching for weaknesses in the target's **services** — and the skill that makes this lab particularly instructive: **patching and compiling an outdated exploit** so that it builds and runs on a modern Kali Linux system.

📘 **Full text:** [README.md](08.%20Kioptrix/README.md)

🛡️ **Hardening guide:** [Hardening-Guide.md](08.%20Kioptrix/Hardening-Guide.md) — system inventory, OS and package upgrades, migration to a supported platform, and remediation of the exposed services.

**Solution path:**

1. **Reconnaissance** — `netdiscover` to locate the target, `nmap` to map open ports and exact service versions.
2. **SMB enumeration** — `enum4linux` / `rpcclient`.
3. **Web server examination** — `dirb`; recognizing the web server as a rabbit hole.
4. **Vulnerability analysis** — matching banners to exploits with `searchsploit`.
5. **Patching and compilation** — installing the required libraries, modifying `764.c` and compiling it.
6. **Exploitation** — selecting the correct offset and executing the attack.
7. **Confirmation** — post-exploitation verification of `uid=0`; an alternative path via Samba (trans2open) is also presented.

**Tools:** `netdiscover`, `nmap`, `enum4linux`, `rpcclient`, `dirb`, `searchsploit`, `gcc`, Metasploit Framework.
**Source:** [VulnHub — Kioptrix: Level 1 (#1)](https://www.vulnhub.com/entry/kioptrix-level-1-1,22/)

---

## Environment Requirements

- **Software:** VirtualBox or VMware Workstation (see lab [00](00.%20Virtualization/README.md)).
- **Hardware virtualization:** enabled in the BIOS/UEFI (Intel VT-x / AMD-V).
- **Attacking machine:** Kali Linux (or any distribution with the tools used in the exercises).
- **Targets:** VM images from [VulnHub](https://www.vulnhub.com/) (Raven 1, DC-1, Raven 2, Kioptrix: Level 1) and the [Mr Robot](https://tryhackme.com/room/mrrobot) room on TryHackMe.
- **Network:** an isolated NAT Network or Host-Only network.
- **Background knowledge:** comfort with the terminal and basic TCP/IP concepts.

---

## Recommended Order

```
00. Virtualization  →  01. Linux CLI  →  02. Bandit      →  03. Raven-CTF  →  05. DC-CTF     →  06. Raven2-CTF  →  07. Mr Robot  →  08. Kioptrix
(VM setup,             (foundations:     (CLI practice      (first CTF         (pentest          (web RCE +          (WordPress →     (exploit
networking,            CLI,              over SSH)          experience)        methodology)      UDF privesc)        root)            patching)
snapshots)             networking)
```

Lab 00 is a prerequisite for all others: without a virtual machine and a correctly configured network, none of the subsequent labs can be carried out. Lab 01, in turn, provides the commands (`ls`, `grep`, `chmod`, `ss`, `ping`) that are used directly in the labs that follow, while Lab 02 (Bandit) puts them into practice on a real remote system over SSH before the CTF machines in Labs 03–07. Lab 06 is the natural continuation of Lab 03 (the same Raven series, with a more demanding route to root), Lab 07 combines web enumeration, brute force and SUID abuse in a single scenario, and Lab 08 focuses on adapting and compiling legacy exploit code.

---

## Legal Notice and Ethics

All techniques and tools described here are intended **exclusively for educational use** in controlled, isolated environments and on systems for which explicit authorization has been granted.

Unauthorized access to third-party systems is a criminal offence. Perform these exercises only on local virtual machines or on platforms designed for this purpose.

---

## Sources and References

### Lab 00 — Virtualization
- [Oracle VM VirtualBox — Documentation](https://www.virtualbox.org/manual/)
- [DMTF — Open Virtualization Format (DSP0243)](https://www.dmtf.org/standards/ovf)
- [Kali Linux — Downloads (VirtualBox images)](https://www.kali.org/get-kali/#kali-platforms)

### Lab 02 — Bandit
- **Platform:** [OverTheWire — Bandit](https://overthewire.org/wargames/bandit/)
- **Reference write-up:** [Bandit Write-up — A. Cheron](https://axcheron.github.io/writeups/otw/bandit/)

### Lab 03 — Raven: 1
- **Machine download:** [VulnHub — Raven: 1](https://www.vulnhub.com/entry/raven-1,256/)
- **Reference write-up:** [Raven 1 CTF Write-up — Hariharan (corruptedprotocol.medium.com)](https://corruptedprotocol.medium.com/raven-1-ctf-write-up-1ddaf2cff9e2)
- **Tools:** [WPScan](https://wpscan.com/), [OWASP DirBuster/Gobuster](https://www.kali.org/tools/gobuster/), [John the Ripper](https://www.openwall.com/john/)

### Lab 05 — DC: 1
- **Machine download:** [VulnHub — DC: 1](https://www.vulnhub.com/entry/dc-1,292/)
- **Reference write-up:** [DC-1: Vulnhub Walkthrough — Hacking Articles](https://www.hackingarticles.in/dc-1-vulnhub-walkthrough/)
- **Vulnerability:** [CVE-2014-3704 — Drupalgeddon (NVD)](https://nvd.nist.gov/vuln/detail/CVE-2014-3704)
- **Module:** [Metasploit — `exploit/multi/http/drupal_drupageddon`](https://www.rapid7.com/db/modules/exploit/multi/http/drupal_drupageddon/)
- **Remediation:** [OWASP Core Rule Set (ModSecurity)](https://owasp.org/www-project-modsecurity-core-rule-set/)

### Lab 06 — Raven: 2
- **Machine download:** [VulnHub — Raven: 2](https://www.vulnhub.com/entry/raven-2,269/)
- **Reference write-up:** [Raven 2: Vulnhub Walkthrough — Hacking Articles](https://www.hackingarticles.in/raven-2-vulnhub-walkthrough/)
- **Vulnerabilities:** [CVE-2016-10033 — PHPMailer RCE (NVD)](https://nvd.nist.gov/vuln/detail/CVE-2016-10033), [CVE-2016-6662 — MySQL/MariaDB privilege escalation (NVD)](https://nvd.nist.gov/vuln/detail/CVE-2016-6662)
- **Exploit:** [Exploit-DB 40974 — PHPMailer < 5.2.18 RCE (Python)](https://www.exploit-db.com/exploits/40974)

### Lab 07 — Mr Robot
- **Reference write-up:** [Mr Robot CTF Walkthrough (2023) — Charalampos Spanias](https://medium.com/@cspanias/thms-mr-robot-ctf-walkthrough-2023-55ca5c19fbaf)
- **Platform:** [TryHackMe — Mr Robot CTF](https://tryhackme.com/room/mrrobot)
- **GTFOBins:** [nmap](https://gtfobins.github.io/gtfobins/nmap/)
- **Reverse shell:** [pentestmonkey/php-reverse-shell](https://github.com/pentestmonkey/php-reverse-shell)

### Lab 08 — Kioptrix: Level 1
- **Machine download:** [VulnHub — Kioptrix: Level 1 (#1)](https://www.vulnhub.com/entry/kioptrix-level-1-1,22/)

### General
- [Metasploit Framework](https://www.metasploit.com/)
- [Nmap — Reference Guide](https://nmap.org/book/man.html)
- [VulnHub — Vulnerable Machines](https://www.vulnhub.com/)
