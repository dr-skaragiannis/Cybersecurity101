# Raven 1 CTF

**Capture the Flag (CTF)** is a cybersecurity challenge in which the player attempts to locate hidden markers, known as "flags," within a target system (commonly referred to as a "box"). Such challenges offer an excellent opportunity for enjoyment, self-directed learning, and hands-on practice of new skills. In this lab, we will solve a CTF from the VulnHub platform named **Raven**.

**Raven 1** serves as an introductory CTF example with a strongly practical orientation. The virtual machine can be downloaded from its official page: [VulnHub - Raven: 1](https://www.vulnhub.com/entry/raven-1,256/)

> - **Write-up in English:** [Raven 1 CTF Write-up — Hariharan](https://corruptedprotocol.medium.com/raven-1-ctf-write-up-1ddaf2cff9e2)
> - **YouTube Video:** [Raven1 VulnHub CTF Walkthrough - Boot-To-Root](https://www.youtube.com/watch?v=Aha7JWSIfSI)

#### **Caution! All IP addresses shown in these instructions are illustrative only! It is essential that you interpret each situation according to your own environment; the aim is to avoid executing commands directly as they appear in the instructions. In every environment, DHCP assigns different IP addresses and physical (MAC) addresses.**

### Our Role (Identity & Persona Definition)
In this lab we assume the role of an **Ethical Hacker (White Hat)** and a **Penetration Tester**.

* **We do not act as malicious attackers (Black Hats):** Our objective is not sabotage, data theft, or service disruption (Denial of Service).
* **Our objective:** We emulate the methodologies and tactics of a real-world attacker (adversary emulation) in order to:
  1. Identify design flaws, misconfigurations, and software vulnerabilities.
  2. Demonstrate their practical exploitability (Proof of Concept - PoC).
  3. Provide well-documented findings and remediation guidance so that the system can be hardened.

### The Philosophy of CTFs (Capture The Flag)
**Boot-to-Root** machines (from initial boot to full administrative/root compromise) function as simulations of real-world scenarios.
* **Flags** are unique text strings placed at strategic locations (e.g., source code, the database, administrator system files).
* They serve as cryptographic proof that each stage of the attack has been completed successfully:
  * **Flag 1:** Initial reconnaissance and analysis of the external attack surface (Footprinting).
  * **Flags 2 & 3:** Compromise of the application and access to internal structures (Initial Access & Lateral Movement).
  * **Flag 4:** Full control of the system (Privilege Escalation to `root`).

---

## Penetration Testing Methodology (PTES Alignment)

The analysis of Raven 1 follows the international guidelines of the **Penetration Testing Execution Standard (PTES)**:

```
[ Preparation & Network Verification ]
                 │
                 ▼
[ Step 1: Reconnaissance (Reconnaissance / Discovery) ]
                 │
                 ▼
[ Step 2: Enumeration & Port Scanning (Enumeration) ]
                 │
                 ▼
[ Step 3: Attack Surface Mapping & Flag 1 ]
                 │
                 ▼
[ Step 4: Path & Directory Discovery (Fuzzing) ]
                 │
                 ▼
[ Step 5: Targeted Web Application Analysis (CMS Analysis) ]
```

---

![Overall approach to solving Raven](../../03.%20Raven-CTF/images/Οδηγός_Επίλυσης_Raven_1_CTF.jpg)

## Target & Lab Information

* **Target Name:** Raven: 1
* **Source Platform:** VulnHub ([VM Download Page](https://www.vulnhub.com/entry/raven-1,269/))
* **Lab Objective:** Capture all 4 flags and escalate privileges to the `root` user.
* **Attacking Machine:** Kali Linux (updated with the necessary network and web testing tools).

---

## Prerequisites & Network Verification (Pre-requisites & Sanity Checks)

Before any attack command is executed, it must be confirmed that the communication channel between the attacker and the target is functioning correctly.

### Virtual Machine Configuration (Hypervisor Setup)
* Attach both the Kali Linux VM and the Raven 1 VM to the same type of internal network (a **NAT Network** in VirtualBox / VMware Workstation is recommended).
* **Warning:** Avoid *Bridged* mode, as it exposes the vulnerable VM directly to your home or corporate local area network (LAN).

### Determining Your Local IP Address (`ip addr` / `ifconfig`)
To know which subnet to scan, inspect your network interface:

```bash
# Modern command (recommended)
ip addr show

# Legacy command
ifconfig

#Administrator privileges
sudo su

#Discovery of other devices
sudo arp-scan -l
## Ping connectivity check
ping <IP-Address-of-the-victim>
```

* **What we analyze:**
  * Identify the Ethernet interface (typically `eth0`, `ens33`, or `enp0s3`).
  * Record the IP address and CIDR prefix (e.g., `192.168.56.101/24`).
  * The subnet to be examined is the corresponding network (e.g., `192.168.56.0/24`).

### Checking the Routing Table (Default Gateway)
Verify that packet routing is correct and that there are no conflicts between network adapters:

```bash
ip route show
# or
route -n
```

### Connectivity & Responsiveness Check (`ping`)
Once the target's IP has been identified (in Step 1), confirm two-way communication:

```bash
sudo arp-scan -l
ping -c 4 <TARGET_IP>
```

> **Note:** If `ping` fails (100% packet loss) even though the device is active, the target's operating system may be dropping ICMP Echo requests. In that case, we use the `-Pn` option in Nmap to skip host discovery.

---

## Step 1: Network Reconnaissance (Reconnaissance)
When a machine needs to send data to an IP address on the same subnet (e.g., from `192.168.1.50` to `192.168.1.80`):

1. It checks its local ARP cache. If no entry exists, it sends an **ARP Request** as a **Broadcast** (`FF:FF:FF:FF:FF:FF`):
   > *"Who has IP 192.168.1.80? Tell [My MAC]."*
2. Every node in the same broadcast domain receives the query.
3. Only the machine that owns that IP responds with an **ARP Reply** (Unicast):
   > *"That is me, and my MAC is 08:00:27:xx:xx:xx."*

Discovery tools automate this process across the entire address range of the subnet.

The first step is to identify the IP address of the machine running as a virtual machine on our local network. To accomplish this, you may use the **`**netdiscover**`** tool (e.g., by running `netdiscover -r <network range>`), **arp-scan -l**, or any other network scanning tool of your choice (such as `nmap -sn`).

When scanning in CTF or lab environments, the first 3 bytes (OUI prefix) of the MAC address immediately reveal the hypervisor:

* **VirtualBox:** Often begins with `08:00:27`
* **VMware:** Often begins with `00:05:69`, `00:0c:29`, or `00:50:56`
* **QEMU / KVM:** Often begins with `52:54:00`

Alternatively, you can view the MAC address of the target VM in its VirtualBox settings, under the network settings.

## Step 2: Enumeration and Port Scanning (Enumeration & Port Scanning)
It is always critically important to perform systematic enumeration of the machine rather than relying on blind trial and error. Understanding each step is essential. A basic network scan (usually with **Nmap**) is indispensable in every CTF in order to reveal open ports and identify potential vulnerabilities using the tool's various scripts (e.g., with `nmap -sC -sV <IP>`).

![Nmap scan result showing the open ports](../../03.%20Raven-CTF/images/image.png)

*Figure 1 — Nmap scan: 22/tcp (ssh), 80/tcp (http), and 111/tcp (rpcbind) open.*

## Command Analysis

```bash
nmap 192.168.29.167 -Pn
```

* **`nmap`**: The network reconnaissance and security auditing tool (Network Mapper).
* **`192.168.29.167`**: The IPv4 address of the target machine (local private network / LAN).
* **`-Pn` (Disable Host Discovery / No Ping)**:
  * Instructs Nmap not to first attempt to determine whether the target is up (e.g., via ICMP Echo ping).
  * Treats the target directly as active (`Host is up`).
  * Useful primarily when an intermediate firewall blocks ICMP pings, so that the scan is not aborted.

---

## Identified Open Ports

| Port / Protocol | State | Service | Description |
| :--- | :--- | :--- | :--- |
| **`22/tcp`** | `open` | **ssh** | Secure Shell: Used for secure remote command-line administration. |
| **`80/tcp`** | `open` | **http** | HyperText Transfer Protocol: An open web server (unencrypted website or web application). |
| **`111/tcp`** | `open` | **rpcbind** | RPC Portmapper: Maps RPC program numbers to ports (frequently associated with services such as NFS). |

---

The scan results show that there are **3 open ports**. Of these three, the **SSH** (typically on port 22) and **HTTP** (port 80) services are the most common.
We begin by entering the machine's IP address into our web browser to see exactly what the site is serving.

![Home page of the web server over HTTP](../../03.%20Raven-CTF/images/image-1.png)

*Figure 2 — The "Raven Security" home page on port 80.*

## Step 3: Website Analysis and Discovery of the 1st Flag
The website contains numerous hyperlinks. The very next thing any security professional should examine is the browser's **Developer Tools** (by pressing F12), using "Inspect" to review the page's source code (HTML).
CTFs frequently hide useful clues in the form of HTML comments (`<!-- comment -->`).

![HTML comment containing the first flag in the source code](../../03.%20Raven-CTF/images/image-2.png)

*Figure 3 — The `<!--flag{...}-->` hidden in the HTML comments.*

Indeed, the **First Flag (Flag 1)** is well concealed within the code visible in the Developer Tools.

## Step 4: Directory Brute-Forcing (Directory Fuzzing)
The next logical step after examining the home page is to discover hidden directories on the server. For this purpose we use **DirBuster** (or similar tools such as Gobuster/Dirb). This is a widely used technique that makes it possible to locate paths not reachable through links, without resorting to blind guessing.

![Results from running DirBuster](../../03.%20Raven-CTF/images/image-3.png)

*Figure 4 — DirBuster: the `/wordpress/` directory was discovered (HTTP 301).*

The tool's results reveal the existence of a directory named **WordPress**.

## Step 5: Vulnerability Assessment with WPScan
The WordPress content management system has a rather poor track record with respect to vulnerabilities. By launching a scan with the specialized **WPScan** tool, we can determine the exact WordPress version, identify any vulnerable plugins, and enumerate users.

The WPScan command syntax may appear somewhat complex to a beginner (e.g., `wpscan --url http://<IP>/wordpress/ -e u`). Whenever difficulties arise, the tool's built-in documentation, accessible via `man wpscan` or `wpscan --help`, provides a complete explanation of the available options.

![Running WPScan in the terminal](../../03.%20Raven-CTF/images/image-4.png)

*Figure 5 — Running WPScan with user enumeration (`-eu`) and plugin/theme enumeration (`-ep -et`).*

The scan may take a few minutes to complete.

Once the scan has finished, the tool's database reveals the **usernames**. Could we perhaps use these usernames to log in to the machine via SSH? Let us find out.

![Results showing the usernames found by WPScan](../../03.%20Raven-CTF/images/image-5.png)

*Figure 6 — WPScan: the users `steven` and `michael` were identified.*

## Step 6: System Access and Discovery of the 2nd Flag
At this point we face a problem: the password for the username we discovered (user `michael`) is unknown to us. In such cases, it is very good practice to always try the most "common" or obvious passwords before launching time-consuming brute-force attacks.

Specifically, the user runs the command `hydra -l michael -P /usr/share/wordlists/rockyou.txt ssh://10.0.2.6`, which targets the account named `michael` at IP address `10.0.2.6`, leveraging the well-known `rockyou.txt` password wordlist for the large-scale, automated testing of thousands of candidate passwords. As shown in the execution results at the bottom of the terminal, the attack completes successfully: the tool identifies a valid password, revealing that the password for user `michael` is also the word "michael".

![SSH brute-force with hydra](../../03.%20Raven-CTF/images/image-13.png)

*Figure 7 — `hydra -l michael -P rockyou.txt ssh://…`: successful brute-force, `login: michael  password: michael`.*

Upon testing, we confirm that the username and password were identical: **"michael"**.
We therefore successfully log in via SSH (`ssh michael@<IP>`).

![SSH login as user michael](../../03.%20Raven-CTF/images/image-6.png)

*Figure 8 — Successful login as `michael` and access to MySQL as `root`.*

The next major step in any CTF, once access has been obtained as an unprivileged user, is **Privilege Escalation** in order to become an administrator (root). Before proceeding with this, however, let us examine the system files to which we already have access.

![Exploring files in the terminal](../../03.%20Raven-CTF/images/image-7.png)

*Figure 9 — Reading `flag2.txt` inside `/var/www/`.*

While exploring the directories, we navigate to `/var/www/html/`. Within this directory, we locate the **Second Flag (Flag 2)**.

## Step 7: Database Exploration (Discovery of Flags 3 & 4)
Continuing a deeper investigation of the directories, we move to the WordPress folder (the path is `/var/www/html/wordpress/`). It contains the file `wp-config.php`, which is extremely important because it holds the database connection details (database credentials).

We use these credentials to connect to **MySQL** from the terminal (e.g., `mysql -u <user> -p`).

![Successful login to the MySQL environment](../../03.%20Raven-CTF/images/image-8.png)

*Figure 10 — MySQL: the `mysql`, `performance_schema`, and `wordpress` databases.*

We switch the active database to the WordPress database (by typing `USE wordpress;`). The database tables are now visible to us (`SHOW TABLES;`). We notice several highly interesting tables, such as `wp_users` and `wp_posts`.

![List of tables in the wordpress database](../../03.%20Raven-CTF/images/image-9.png)

*Figure 11 — `SHOW TABLES`: the tables `wp_users`, `wp_posts`, `wp_options`, etc.*

By searching the `wp_posts` table (the site's posts), we find the **Third Flag (Flag 3)** and the **Fourth Flag (Flag 4)**. In effect, this marks the official completion of the CTF, since all 4 flags have been found.

![Contents of the wp_posts table with flags 3 and 4](../../03.%20Raven-CTF/images/image-10.png)

*Figure 12 — `flag3` and `flag4` appear in `wp_posts`.*

What about **Root Access** (obtaining administrative access)?

![Privilege Escalation on Raven](../../03.%20Raven-CTF/images/Odigos2.jpg)

## Step 8: Privilege Escalation (to Root)

The `sudo -l` command checks and lists the `sudo` privileges granted to the currently logged-in user (`steven`) in the `/etc/sudoers` file.

![alt text](../../03.%20Raven-CTF/images/imagea.png)

### Terminal Output
```text
Matching Defaults entries for steven on raven:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\...

User steven may run the following commands on raven:
    (ALL) NOPASSWD: /usr/bin/python
```

We observe that the user is permitted to execute Python as an administrator without a password (NOPASSWD). We exploit this configuration to perform privilege escalation, spawning an interactive bash shell with root privileges via the `os` library.

![alt text](../../03.%20Raven-CTF/images/image-1a.png)

`sudo python -c 'import os; os.system("/bin/bash")'`


This command abuses the potentially dangerous permission to run Python as `root` in order to break out of the interpreter environment (binary escape) and open a full root shell (a technique documented in **GTFOBins**).

### Parameter Analysis
* **`sudo python`**: Launches the Python interpreter under the superuser (`root`) account.
* **`-c`** (*command*): Allows Python code to be passed directly as a command-line argument, without the need to create or save a separate `.py` file.
* **`'import os; ...'`**: Imports the built-in `os` module, which provides interfaces for interacting with the operating system.
* **`os.system("/bin/bash")`**: Calls the `system()` function, executing the `/bin/bash` shell.
* **Privilege Escalation Mechanism**: Because the parent Python process runs as `root`, every child process spawned through `os.system` automatically inherits the same privileges. As a result, the user immediately obtains an interactive `root@Raven` shell.


## Step 8b: John The Ripper - Alternative Access (Root)
The `wp_users` table we saw earlier in the database contains the password "hashes" (one-way cryptographic representations) of the WordPress users' passwords.

![wp_users table with the password hashes](../../03.%20Raven-CTF/images/image-14.png)

*Figure 13 — `SELECT * FROM wp_users`: the hashes (phpass, `$P$…`) for `michael` and `steven`.*

We copy these hashes and use a password-cracking tool called **John The Ripper** (`john`) to recover the plaintext passwords (e.g., by supplying it with a wordlist such as `rockyou.txt`).
This process successfully reveals the password for user **`steven`**. Most notably, `steven`'s password happens to be **the same password required for root access** (either via `su root` or via `sudo` privileges).

```bash
# Save the hashes to a file and run a dictionary attack
john hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

![Running John The Ripper on the hashes](../../03.%20Raven-CTF/images/image-11.png)

*Figure 14 — John The Ripper: 2 hashes (phpass), 1 remaining to be cracked.*

![Result: the password of user steven](../../03.%20Raven-CTF/images/image-12.png)

*Figure 15 — Successful credential recovery: `steven:pink84`.*

The password is now available, and we can take full (root) control of the system.

### Summary
You may occasionally encounter difficulties while solving a CTF; however, with systematic practice, it is entirely possible to refine your techniques and your proficiency with the tools.
