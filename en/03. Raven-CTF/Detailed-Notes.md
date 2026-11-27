
# Detailed Notes — Lab 03: Raven 1 (VulnHub)

This file is the **extended edition** of the lab's `README.md`. The original text is preserved **in full**, and after each step an *"Explanation"* box is added, covering the theory behind each action, a breakdown of the commands, and the points at which students most frequently encounter difficulty.

**The methodology at a glance**

A *boot-to-root* machine is not solved through isolated, unrelated techniques. Its solution follows a structured methodological cycle, which is common to every professional penetration test and is reflected in international standards such as the **PTES** (Penetration Testing Execution Standard):

```
 1. Reconnaissance   →  who is the target? (IP, ports, services)
 2. Enumeration      →  what is running and what does it expose? (web, users, versions)
 3. Exploitation     →  which vulnerability is exploited? (RCE, weak creds)
 4. Post-exploit     →  what can I see from the inside? (files, configs, DB)
 5. Privilege esc.   →  how do I go from user to root? (SUID, sudo, creds)
 6. Reporting        →  what did I learn and how is it fixed? (flags, remediation)
```

Each step of this guide corresponds to one of these phases. Whenever progress appears to stall, the analyst should return to the question: *which phase am I in, and what specific piece of information am I missing in order to move on to the next one?* This systematic approach replaces random trial and error with a well-founded methodology.

**Learning outcomes:** Upon completing this lab, the student will be able to identify active hosts on a local network, interpret the output of the nmap port scanner, discover unpublished directories and user accounts, understand why a weak password undermines the overall security of a system, and extract credentials from configuration files such as `wp-config.php` as well as from MySQL database tables.

---

# Raven 1 CTF - Write-up (Explanation and Solution)

**Capture the Flag (CTF)** is a cybersecurity challenge in which the player attempts to find hidden "markers" (flags) within a system (the so-called "box"). These challenges offer an excellent opportunity for enjoyment, self-directed learning, and hands-on practice of new skills. Today, we will solve a CTF from the VulnHub platform called **Raven**.

**Raven 1** is an excellent example of a well-structured CTF with a highly practical approach. This challenging machine is of particular interest because it involves the use of many different tools and techniques.
You can download the machine from its official page: [VulnHub - Raven: 1](https://www.vulnhub.com/entry/raven-1,256/)

> **Reference write-up:** [Raven 1 CTF Write-up — Hariharan](https://corruptedprotocol.medium.com/raven-1-ctf-write-up-1ddaf2cff9e2)

The central objective of this CTF is to find **4 flags**, which are very well hidden within the system.

## Step 1: Network Reconnaissance

### Explanation of Step 1 — why ARP and not ping

**Key concepts.** A **local area network (LAN)** is a network of limited geographical extent in which devices communicate directly with one another. Each device has two addresses: a logical **IP address** (network layer — OSI L3), which may change, and a unique physical **MAC address** (data link layer — OSI L2), embedded in the network interface card. The **ARP (Address Resolution Protocol)** is the protocol that maps an IP address to the corresponding MAC address.

For this reason, within a LAN the most reliable way to verify that a host exists is not the **ICMP** protocol (used by the `ping` command), but ARP. Every active network interface is obliged to answer the question "which MAC address corresponds to this IP address?", because otherwise communication over Ethernet would be impossible. ICMP replies, by contrast, can be filtered or disabled by a firewall, which makes `ping` unreliable as the sole criterion.

| Method | Protocol | Easily blocked? | When it is used |
| :--- | :--- | :--- | :--- |
| `ping` sweep | ICMP | **Yes** (firewalls, configuration) | Quick check |
| `netdiscover` / `arp-scan` | ARP (L2) | Not within a LAN | Local lab network |
| `nmap -sn` | ICMP + ARP + TCP | Partially | General scan |
| `nmap -Pn` | TCP (no ping) | No | When ICMP is filtered |

**The `netdiscover` tool.** This is an active reconnaissance tool that sends ARP requests to a range of addresses and records the replies. The typical syntax is `netdiscover -r <range>`:

* The `-r` parameter defines the **scan range** in **CIDR (Classless Inter-Domain Routing)** notation. This notation expresses how many of the 32 bits of an IPv4 address are fixed (the "network prefix"). Thus, `/24` fixes the first 24 bits and leaves the remaining 8 free, i.e. 2⁸ = 256 addresses, of which 254 are usable for hosts (the first corresponds to the network itself and the last to the broadcast address).
* For each host that responds, the tool displays the IP address, the MAC address, and the **vendor** of the network interface card. The label "VMware, Inc." is a strong indication that the host is a **virtual machine (VM)**.

**The significance of the MAC address.** The first three octets (bytes) of a MAC address constitute the **OUI (Organizationally Unique Identifier)** and identify the manufacturer. In a real penetration test, the appearance of a "VMware" or "QEMU" MAC address within a corporate network indicates a virtual machine — often a test system or one that has already been compromised.

**Common mistake.** Scanning a `/24` range on the wrong subnet (e.g. scanning a different network while the lab resides on VirtualBox's NAT network `10.0.2.0/24`) will yield no results. It is advisable to first check the local address with the `ip a` command in order to determine the correct subnet to scan.
The first step is to identify the IP address of the machine running as a virtual machine on our local network. To achieve this, you may use the `netdiscover` tool (e.g. by running `netdiscover -r <network range>`) or any other network scanning tool of your choice (such as `nmap -sn`).

## Step 2: Enumeration & Port Scanning

### Explanation of Step 2 — interpreting nmap output

**What a port and a service are.** Every network service "listens" on a numbered **port** — a logical communication channel at the transport layer (OSI L4, TCP/UDP protocols). A single IP address can serve many services simultaneously, each on a different port (e.g. SSH on port 22, HTTP on port 80). Identifying the open ports therefore reveals which services are available and, by extension, what the system's **attack surface** is.

**nmap (Network Mapper)** is the standard tool for port scanning and service identification. The command `nmap -sC -sV <IP>` combines three distinct techniques:

| Parameter | What it does | What information it adds |
| :--- | :--- | :--- |
| (none) | TCP connect / SYN scan | Which ports are **open** |
| `-sV` | Version detection | Which **service and version** (e.g. Apache 2.4.10) |
| `-sC` | Default scripts | Banners, HTTP titles, indications of vulnerabilities |

**The three basic port states:**

* **open** — a service is listening and accepting connections.
* **closed** — nothing is listening, but the machine responds (RST).
* **filtered** — no response is received; the traffic is blocked by a firewall or dropped by the host.

**How the result translates into an attack plan:**

| Port | Service | First consideration |
| :--- | :--- | :--- |
| 22 | SSH | Are there known users? Weak passwords? |
| 80/443 | HTTP/S | CMS, versions, hidden directories, comments in code |
| 111 | rpcbind | NFS exports — often left "open" without access control |
| 3306 | MySQL | Credentials in configuration files |

**On Raven 1**, the three open ports (22, 80, 111) dictate a clear strategy: the **website** constitutes the primary attack surface, **SSH** will serve as the entry point once a valid password has been identified, while the presence of rpcbind is a reminder that no service should be ignored.

**Useful variations:**

```bash
nmap -sV -p- 10.0.2.6            # all 65535 ports (slow but thorough)
nmap -sV -p 22,80,443 10.0.2.6   # targeted
nmap -A 10.0.2.6                 # -sV + -sC + OS detection + traceroute
```

**Caution:** the `-sC` parameter executes automated scripts which, against a vulnerable or unstable service, may cause it to malfunction or even crash. In production environments it is used sparingly and only with explicit authorization.
It is always extremely important to proceed with "enumeration" of the machine rather than resorting to blind guessing. Understanding each step is critical. A basic network scan (usually with **Nmap**) is essential for every CTF, in order to reveal the "open ports" and to identify potential vulnerabilities using the tool's various scripts (e.g. with `nmap -sC -sV <IP>`).

![Nmap scan results showing the open ports](../../03.%20Raven-CTF/images/image.png)

*Figure 1 — Nmap scan: 22/tcp (ssh), 80/tcp (http) and 111/tcp (rpcbind) open.*

From the scan results we observe that there are **3 open ports**. Of these three, the **SSH** (usually on port 22) and **HTTP** (port 80) services are the most common.
We begin by entering the machine's IP address into our web browser to see exactly what the page is "serving"!

![Home page of the web server over HTTP](../../03.%20Raven-CTF/images/image-1.png)

*Figure 2 — The "Raven Security" home page on port 80.*

## Step 3: Analyzing the Website and Finding the 1st Flag

### Explanation of Step 3 — what we look for within a page

In a CTF, the first flag is usually hidden in a place that is **not visible** on screen but exists in the HTML. The classic locations to check are:

| Location | How to view it | What it may conceal |
| :--- | :--- | :--- |
| HTML comments | `Ctrl+U` or DevTools (F12) | Flags, usernames, paths |
| Meta tags | `<meta name="generator">` | CMS and version (e.g. WordPress 5.x) |
| `robots.txt` | `http://host/robots.txt` | Directories they "do not want" to be seen |
| `sitemap.xml` | `http://host/sitemap.xml` | Site structure |
| JavaScript files | DevTools → Sources | API endpoints, keys, comments |
| HTTP headers | `curl -I http://host` | Server, technologies |

**Developer Tools.** Every modern web browser includes built-in developer tools, accessible with the `F12` key or the `Ctrl+U` key combination (view page source). Through these, the analyst examines the **HTML** source code that the server sent to the browser — that is, everything delivered to the client side, regardless of whether it is visually rendered on the page.

**Why HTML comments are a frequent finding.** Comments of the form `<!-- ... -->` are not rendered visually, yet they remain fully visible to anyone who inspects the source code. Developers insert such comments during development and often neglect to remove them before deployment. This phenomenon is the simplest example of **information disclosure**: the exposure of sensitive data without even requiring the exploitation of a software vulnerability.

**Additional information to retain:** from the Raven Security page you learn that this is a corporate site built on WordPress — so the next step (directory fuzzing) has a specific target: `/wordpress/`.

**Checking from the terminal (useful when no browser is available):**

```bash
curl -s http://10.0.2.6/ | grep -i "flag\|comment"
```
The website contains many hyperlinks. The very next thing every security professional should check is the browser's **Developer Tools** (by pressing F12), and "Inspect" the page's HTML source code.
CTFs frequently hide useful clues in the form of HTML comments (`<!-- comment -->`).

![HTML comment containing the first flag in the source code](../../03.%20Raven-CTF/images/image-2.png)

*Figure 3 — The `<!--flag{...}-->` hidden in the HTML comments.*

And indeed, the **First Flag (Flag 1)** is very well hidden within the code, visible in the Developer Tools!

## Step 4: Directory Brute-Forcing (Directory Fuzzing)

### Explanation of Step 4 — the logic of directory fuzzing

**Definition.** **Directory fuzzing** (or directory brute-forcing) is the automated testing of a large number of possible directory and file names on a web server, with the aim of discovering resources that are not linked from the visible pages. The method does not rely on random guessing, but on **wordlists** containing thousands of common names (e.g. `common.txt`, `directory-list-2.3-medium.txt`). If the administrator has left directories such as `/backup/`, `/vendor/` or `/wordpress/` accessible without any public link to them, the tool discovers them through the server's responses.

**How to distinguish "exists" from "does not exist" — HTTP status codes:**

| Code | Meaning | What it means for us |
| :--- | :--- | :--- |
| **200** | OK | The page/directory exists and is accessible |
| **301/302** | Redirect | It exists, but at a different path (e.g. `/wordpress` → `/wordpress/`) |
| **403** | Forbidden | It **exists** but access is denied — information in its own right |
| **401** | Unauthorized | Requires login |
| **404** | Not found | Does not exist |

**Why a 403 is useful:** it confirms the existence of a file/directory. In a real audit, a `403` on `/admin/` tells you "there is an administration panel" — and you may be able to reach it by other means (a bypass, a known CVE).

**Tool comparison:**

| Tool | Advantage | Disadvantage |
| :--- | :--- | :--- |
| **DirBuster** | Graphical, easy for beginners | Slow, resource-heavy |
| **gobuster** | Fast, CLI, scriptable | Requires an appropriate wordlist |
| **ffuf** | Very fast, response-size filters | Steeper learning curve |

**Example with gobuster:**

```bash
gobuster dir -u http://10.0.2.6/ -w /usr/share/wordlists/dirb/common.txt -x php,txt
```

**On Raven 1**, the result `301 → /wordpress/` is the critical finding: it tells us that a CMS is present, and therefore that user and plugin management exists — in other words, an attack surface.
The next logical step, after examining the home page, is to discover unpublished directories on the server. For this purpose we use **DirBuster** (or equivalent tools such as Gobuster/Dirb). This is a very widely used technique that makes it possible to discover paths not reachable through links, without resorting to blind guessing.

![Results from running DirBuster](../../03.%20Raven-CTF/images/image-3.png)

*Figure 4 — DirBuster: the `/wordpress/` directory was discovered (HTTP 301).*

The tool's results reveal the existence of a directory named **WordPress**.

## Step 5: Vulnerability Assessment with WPScan

### Explanation of Step 5 — how WPScan works

**What a CMS is.** **WordPress** is a **Content Management System (CMS)**: a ready-made software platform that enables the creation and management of websites without specialized code development, with functionality that can be extended through **plugins** and **themes**. Precisely because of its widespread adoption, WordPress is a frequent target, since a known vulnerability in a popular plugin simultaneously affects a large number of websites.

**What WPScan is.** It is a scanner specifically designed for WordPress. It does not compromise the website; it **gathers** information (passive and active enumeration):

* The WordPress **version** (from meta tags, readme files, feeds).
* **Plugins/themes** and their versions → compared against known vulnerabilities (WPScan API).
* **Users** — via `?author=1` enumeration or the REST API.

**Breakdown of the lab command:**

```bash
wpscan --url http://10.0.2.6/wordpress -e u
```

| Component | Meaning |
| :--- | :--- |
| `--url` | The target (the WordPress path, not necessarily the web root) |
| `-e u` | *enumerate users* — list of users |
| `-e vp` | vulnerable plugins |
| `-e vt` | vulnerable themes |
| `--api-token` | for the full vulnerability database (free token) |

**Why user enumeration is critical:** WordPress authenticates with **username + password**. If you know the username (`michael`, `steven`), brute-forcing becomes feasible — you no longer need to guess both.

**How a site defends itself:** disabling author enumeration, 2FA, limiting login attempts (rate limiting), strong passwords, hiding the version. Any one of these would have stopped the progression of this CTF.

**Ethics note:** WPScan sends hundreds of requests; against a site you do not own, this constitutes an attack. Here we do it against a local VM.
The WordPress content management system has a rather poor track record with respect to vulnerabilities. By launching a scan with the specialized tool **WPScan**, we can discover the exact WordPress version, any vulnerable plugins, and also enumerate the users.

The syntax of the WPScan command may initially appear complicated (e.g. `wpscan --url http://<IP>/wordpress/ -e u`). Whenever difficulties arise, the tool's built-in documentation, accessible via `man wpscan` or `wpscan --help`, provides a complete explanation of the available parameters.

![Running WPScan in the terminal](../../03.%20Raven-CTF/images/image-4.png)

*Figure 5 — Running WPScan with user (`-eu`) and plugin/theme (`-ep -et`) enumeration.*

The scan may take a few minutes to complete.

Once the scan has completed, the tool's database reveals the **usernames**. Perhaps we can use these usernames to log in to the machine via SSH? Let us find out!

![Results showing the usernames found by WPScan](../../03.%20Raven-CTF/images/image-5.png)

*Figure 6 — WPScan: the users `steven` and `michael` were identified.*

## Step 6: Gaining System Access and Finding the 2nd Flag

### Explanation of Step 6 — why the "obvious" password is a realistic scenario

**Definition — brute-force attack.** This is the systematic, automated testing of many candidate passwords until the correct one is found. When the passwords tested come from a predefined list of common passwords, the technique is called a **dictionary attack**.

The combination `michael:michael` (a username identical to the password) may seem far-fetched, but it is an entirely realistic scenario. **Credential stuffing** attacks (testing credentials leaked from other services) and **password spraying** (testing a few very common passwords against many accounts) rely precisely on such insecure user habits: the username as the password, `Password1`, a season followed by `!`, and so on. Statistically, a small dictionary of carefully selected passwords — rather than the entire `rockyou.txt` — is sufficient for a significant success rate.

**The `hydra` tool and the speed of the attack.** `hydra` is a tool for automated credential testing against numerous protocols, including SSH. Brute-forcing SSH is inherently slow, because every individual attempt requires a full TCP **handshake** followed by an SSH cryptographic handshake. For this reason, `hydra` performs many attempts in parallel (the `-t` parameter). This very property, however, makes the attack conspicuous: servers counter such attempts with mechanisms such as **`fail2ban`** (temporarily banning an IP address after repeated failures) or with connection rate limiting.

```bash
hydra -l michael -P /usr/share/wordlists/rockyou.txt ssh://10.0.2.6
hydra -L users.txt -P pass.txt -t 4 ssh://10.0.2.6     # lists for both
```

**What we do after logging in — the logic of enumeration as `michael`:**

1. **Who am I?** `id`, `whoami`, `sudo -l` (what can I run as root?)
2. **Where am I?** `pwd`, `ls -la`, `cat /etc/passwd` (other users)
3. **What can I read?** `find / -name "flag*" 2>/dev/null`, `ls /var/www`
4. **Which services are running?** `ps aux`, `ss -tuln`

**Why `sudo -l` is the first thing we run:** if the user has `sudo` rights on some command (e.g. `less`, `find`, `vim`), privilege escalation is almost guaranteed. On Raven 1 the path is different (password reuse), but the habit remains.

**Flag 2** is located at `/var/www/flag2.txt`: it confirms that the web root is accessible to the user and that the site's "secrets" are not protected.
Here we face a problem. The password for the username we found (user `michael`) is unknown to us. In such cases, it is very good practice to always try the most "common" or obvious passwords before launching time-consuming brute-force attacks.

Specifically, the user runs the command `hydra -l michael -P /usr/share/wordlists/rockyou.txt ssh://10.0.2.6`, which targets the account named `michael` at the IP address `10.0.2.6`, using the well-known `rockyou.txt` password dictionary for the large-scale, automated testing of thousands of candidate passwords. As shown in the execution output at the bottom of the terminal, the attack completes successfully: the tool identifies a valid password, revealing that the password for user `michael` is likewise the word "michael".

![SSH brute-force with hydra](../../03.%20Raven-CTF/images/image-13.png)

*Figure 7 — `hydra -l michael -P rockyou.txt ssh://…`: successful brute-force, `login: michael  password: michael`.*

Upon testing, we confirm that the username and the password were identical: **"michael"**.
We therefore log in successfully via SSH (`ssh michael@<IP>`).

![SSH login as user michael](../../03.%20Raven-CTF/images/image-6.png)

*Figure 8 — Successful login as `michael` and access to MySQL as `root`.*

The next major step in every CTF, once we have gained access as an unprivileged user, is **Privilege Escalation**, in order to become the administrator (root). Before proceeding with this, however, let us take a look at the system files to which we already have access.

![Exploring files in the terminal](../../03.%20Raven-CTF/images/image-7.png)

*Figure 9 — Reading `flag2.txt` within `/var/www/`.*

While exploring the directories, we navigate to `/var/www/html/`. Within this directory, we locate the **Second Flag (Flag 2)**!

## Step 7: Exploring the Database (Finding Flags 3 & 4)

### Explanation of Step 7 — from a configuration file to the database

**What the database is.** WordPress stores its content and users in a **relational database** managed by **MySQL**. The application connects to it using credentials stored in the `wp-config.php` configuration file.

**The critical role of `wp-config.php`.** This file is a central security element of every WordPress installation: it contains the `DB_NAME`, `DB_USER`, `DB_PASSWORD` and `DB_HOST` parameters, as well as the **authentication keys**. If a user gains read access to it, they now possess the credentials for direct access to the database.

**The chain:**

```
/wp-content/../wp-config.php  →  credentials  →  mysql -u user -p  →  USE wordpress;
                                                                    →  SELECT * FROM wp_posts;
```

**Breakdown of the SQL commands:**

| Command | What it does |
| :--- | :--- |
| `SHOW DATABASES;` | List of databases — we look for `wordpress` |
| `USE wordpress;` | Selects the database (otherwise "No database selected") |
| `SHOW TABLES;` | Tables: `wp_posts`, `wp_users`, `wp_options`… |
| `SELECT * FROM wp_posts;` | Post contents (this is where the flags are hidden) |
| `SELECT user_login, user_pass FROM wp_users;` | Users and **hashes** |

**Why the flags are found in "drafts":** `wp_posts` has a `post_status` field (`publish`, `draft`, `inherit`). The flags appear in **draft/revision** records — that is, in content that was never published but remained in the database. This is realistic: deleted or hidden content often persists.

**Good auditing practice:**

```sql
SELECT ID, post_title, post_status FROM wp_posts WHERE post_status != 'publish';
SELECT user_login, user_pass FROM wp_users;
```

**Why this is serious in production:** the web user (`www-data`) should not have read access to all of the site's files, and the database should not expose hashes to anyone who has a shell.
Digging even deeper into the directories, we move to the WordPress folder (the path is `/var/www/html/wordpress/`). There we find the `wp-config.php` file, which is extremely important because it contains the database connection details (database credentials).

We use these credentials to connect to **MySQL** from the terminal (e.g. `mysql -u <user> -p`).

![Successful connection to the MySQL environment](../../03.%20Raven-CTF/images/image-8.png)

*Figure 10 — MySQL: the `mysql`, `performance_schema` and `wordpress` databases.*

We switch the active database to the WordPress one (by typing `USE wordpress;`). The database tables are now visible to us (`SHOW TABLES;`). We see some very interesting tables, such as `wp_users` and `wp_posts`.

![List of tables in the wordpress database](../../03.%20Raven-CTF/images/image-9.png)

*Figure 11 — `SHOW TABLES`: the tables `wp_users`, `wp_posts`, `wp_options`, etc.*

If we search within the `wp_posts` table (the site's posts), we will find the **Third Flag (Flag 3)** and the **Fourth Flag (Flag 4)**. In effect, this marks the official end of the CTF, since we have found all 4 flags!

![Contents of the wp_posts table showing flags 3 and 4](../../03.%20Raven-CTF/images/image-10.png)

*Figure 12 — `flag3` and `flag4` appear in `wp_posts`.*

But wait a moment... What about **root access** (full administrative access)?

![Privilege escalation on Raven](../../03.%20Raven-CTF/images/Odigos2.jpg)

## Step 8: Password Recovery with John the Ripper

### Explanation of Step 8 — recovering a hash and the dangers of password reuse

**What a hash function is.** It is a **one-way function** that transforms an input of arbitrary length (the password) into a fixed-length string (the hash). The process is irreversible: it is computationally infeasible to recover the original password from the hash. For this reason, systems do not store passwords in readable form, but rather their hashes. A **salt** is a random value added to the password before hashing, so that two identical passwords produce different hashes and the use of precomputed tables (rainbow tables) is prevented.

**The `$P$B...` format.** The `$P$` prefix denotes a **phpass** hash — the algorithm used by WordPress (based on MD5 with multiple iterations and a salt). Since the function is one-way, the hash is not "decrypted"; the password is **recovered** through exhaustive testing (a process known as *cracking*).

**How the dictionary attack works.** The **John the Ripper** tool (`john`) applies the following logic:

```
for each word in rockyou.txt:
    compute phpass(word, salt)
    if the result matches the stored hash  →  the password has been found
```

The process is time-consuming because the phpass algorithm is designed to be **deliberately slow** (it performs thousands of iterations per computation, increasing the cost of each attempt). `john` partially offsets this cost by running computations in parallel and through algorithmic optimizations.

**Breakdown of the command:**

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
john --show hashes.txt        # shows whatever has been cracked
```

**Why `steven`'s password also unlocks root:** because the administrator **reused** the password. This is the most common mistake in real-world incidents: a password leaked from an application becomes the key to the system.

**How it should have been:** different passwords per service, `sudo` with NOPASSWD only for specific commands, MFA for SSH, and regular auditing with `john --test` / a password policy.

**In practice, after obtaining `steven:pink84`:** with these credentials we log in as `steven` and obtain a regular (unprivileged) shell, from which the privilege escalation of the next step will begin.

```bash
su steven          # or: ssh steven@10.0.2.6   (password: pink84)
id                 # confirmation: uid=1000(steven) — not root yet
```

**Note:** flag 4 is not always located at `/root/flag4.txt` — on this machine the path to root is the objective, and flags 1–3 have already been found via the web/DB.
The `wp_users` table we saw earlier in the database contains the "hashes" (encrypted forms) of the WordPress users' passwords.

![The wp_users table with the password hashes](../../03.%20Raven-CTF/images/image-14.png)

*Figure 13 — `SELECT * FROM wp_users`: the hashes (phpass, `$P$…`) of `michael` and `steven`.*

We copy these hashes and use a password-cracking tool called **John The Ripper** (`john`) to crack them (e.g. by supplying it with a dictionary such as `rockyou.txt`).
This process successfully reveals the password of user **`steven`**. The most interesting aspect is that `steven`'s password happens to be **the same password required for root access** (either via `su root` or via `sudo` privileges)!

```bash
# Save the hashes to a file and run a dictionary attack
john hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

![Running John The Ripper on the hashes](../../03.%20Raven-CTF/images/image-11.png)

*Figure 14 — John The Ripper: 2 hashes (phpass), 1 remaining to be cracked.*

![Result: the password of user steven](../../03.%20Raven-CTF/images/image-12.png)

*Figure 15 — Successful credential recovery: `steven:pink84`.*

We now have valid credentials and access as `steven`. The final step is **privilege escalation** from this user to `root`.

## Step 9: Privilege Escalation with `sudo` + Python (GTFOBins)

Now that we have a shell as user `steven`, the first thing we examine for privilege escalation is **what we are permitted to execute as `root`**. This is shown by the `sudo -l` command, which lists the entries of the `/etc/sudoers` file that apply to the logged-in user.

![Output of the sudo -l command for user steven](../../03.%20Raven-CTF/images/imagea.png)

### Terminal output

```text
Matching Defaults entries for steven on raven:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\...

User steven may run the following commands on raven:
    (ALL) NOPASSWD: /usr/bin/python
```

We observe that `steven` can run **Python as `root` without a password** (`NOPASSWD`). This in itself is the vulnerability: Python is not a "safe" tool that performs one specific task — it is a **full interpreter** capable of executing arbitrary code, and therefore also of spawning a shell. We exploit this configuration by invoking an interactive `bash` with `root` privileges:

![Performing the binary escape and obtaining a root shell](../../03.%20Raven-CTF/images/image-1a.png)

```bash
sudo python -c 'import os; os.system("/bin/bash")'
```

This technique (documented in **GTFOBins**) leverages the right to run Python as `root` in order to "escape" from the interpreter environment (**binary escape**) and open a full root shell (`root@raven`).

### Explanation of Step 9 — why a simple `sudo python` yields root

**What GTFOBins is.** It is a public knowledge base (*Get The F\* Out Binaries*) that documents how "legitimate" Unix executables can be abused when they are granted elevated privileges — via `sudo`, `SUID`, capabilities, and so on. The logic behind every entry is the same: if a program running as `root` can execute **commands or code of our choosing**, then its privileges become ours.

**Breakdown of the command:**

* **`sudo python`** — launches the Python interpreter under the superuser account. Because of `NOPASSWD`, no password is even requested.
* **`-c`** (*command*) — allows Python code to be passed directly as an argument, without the need for a separate `.py` file.
* **`import os`** — imports the built-in `os` module, which provides interfaces to the operating system.
* **`os.system("/bin/bash")`** — calls the system function, executing the `/bin/bash` shell.

**The escalation mechanism.** When we run something with `sudo`, the process acquires **effective UID 0** (root). Python is now a parent process with root privileges. `os.system()` performs a `fork()` and launches `/bin/bash` as a **child process**; every child **inherits** the security context of its parent, so the new shell also starts as `root`. No security mechanism is "broken" here: an overly broad permission (`sudo` on an entire interpreter) is simply used for exactly what it technically allows.

**Verification.** Inside the new shell we run:

```bash
id        # uid=0(root) gid=0(root) groups=0(root)
whoami    # root
```

If `id` returns `uid=0(root)`, the escalation is complete and we have full control of the machine (as well as access to `/root/flag4.txt`, where it exists).

**Note — alternative form.** The same idea can also be written as `sudo python -c 'import pty; pty.spawn("/bin/bash")'`. The `pty` module provides a fully **interactive** pseudo-terminal (useful for job control, `sudo`, and interactive programs), whereas `os.system` suffices for a simple root shell. Both rely on the same configuration flaw.

**How it should have been (remediation).** The root cause of the problem is a violation of the **principle of least privilege**: never grant `sudo` on a general-purpose interpreter (Python, Perl, Ruby) or on tools capable of spawning a shell (`vi`, `less`, `find`, `awk`, `tar`). If automation is required, grant rights only to the **specific script**, with its full path and validated parameters, preferably without `NOPASSWD`.

### Summary

### Explanation — what to take away from this lab

| Phase | Tool | Key idea |
| :--- | :--- | :--- |
| Recon | `netdiscover` | ARP is not "filtered" within a LAN |
| Scan | `nmap -sC -sV` | Service versions lead to known vulnerabilities |
| Web | DevTools / `curl` | HTML comments leak information |
| Fuzzing | DirBuster / `gobuster` | 403s are findings, not failures |
| CMS | `wpscan -e u` | Users are half of the problem |
| Access | `hydra` + dictionary | "Obvious" passwords work |
| Post | `wp-config.php` | Credentials are written into files |
| Privesc | `sudo -l` + Python (GTFOBins) | `sudo` on an interpreter = root; `john` + password reuse is an alternative |
The methodology followed is, in its conception, relatively simple; the main difficulty for the novice analyst lies in identifying the correct first step and in moving from one phase to the next. These difficulties are to be expected when solving a CTF; with systematic practice, familiarity with the techniques and tools improves steadily.

The machine also has a sequel (**Raven 2**), which is covered in a corresponding lab of this course material (Lab 06).

---

## Appendix A — Lab Command Map

| Purpose | Command |
| :--- | :--- |
| Find target IP | `netdiscover -r 10.0.2.0/24` |
| Service scan | `nmap -sC -sV 10.0.2.6` |
| Check headers | `curl -I http://10.0.2.6/` |
| Discover directories | `dirb http://10.0.2.6/` or `gobuster dir -u … -w …` |
| WP enumeration | `wpscan --url http://10.0.2.6/wordpress -e u` |
| SSH brute-force | `hydra -l michael -P rocklyn.txt ssh://10.0.2.6` |
| Login | `ssh michael@10.0.2.6` |
| Privileges | `id`, `sudo -l`, `ls -la /var/www` |
| MySQL | `mysql -u root -p` → `USE wordpress; SHOW TABLES;` |
| Crack hashes | `john --wordlist=rockyou.txt hashes.txt` |

## Appendix B — Glossary

| Term | Meaning |
| :--- | :--- |
| **Reconnaissance** | Information gathering without direct contact/attack |
| **Enumeration** | Active enumeration (users, directories, versions) |
| **Fuzzing (dir)** | Testing a list of names to discover resources |
| **CMS** | Content management system (WordPress, Drupal) |
| **Hash** | One-way function that cannot be reversed — the password can only be recovered through exhaustive testing |
| **phpass** | WordPress hashing algorithm (`$P$`) |
| **Salt** | Random value that makes each hash unique |
| **Credential reuse** | The same password used across multiple systems |
| **Privilege escalation** | Going from an ordinary user to root |

## Appendix C — Self-Assessment Questions

1. Why does `netdiscover` find hosts that `ping` cannot see?
2. What is the difference between `open` / `closed` / `filtered` in an nmap scan?
3. Why does user enumeration in WordPress facilitate brute-forcing?
4. What does HTTP 403 mean for a directory, and why is it useful?
5. Why is a hash not "decrypted"? Describe the cracking process.
6. Which three measures would have stopped the progression of this CTF?
7. Where would you look first after obtaining a shell, and why (`sudo -l`, `id`, `ls /var/www`)?

### Suggested Answers

1. `netdiscover` operates at **layer 2 (ARP)**: every live host **must** respond to ARP for the network to function. `ping` uses **ICMP (layer 3)**, which many systems/firewalls block — so a host that ignores ICMP echo requests appears "dead" to ping but is detected by ARP. (This applies only within the same LAN, since ARP does not traverse routers.)
2. **open** = a service is listening and has responded (SYN-ACK). **closed** = the port is reachable but no service is listening (it responded with RST). **filtered** = no response was received, or an ICMP unreachable message was returned — something (usually a firewall) is filtering the packet, so we cannot tell whether the port is open or closed.
3. Brute-forcing requires a **valid username + password**. If we know the real users (e.g. `michael`, `steven` via `wpscan -e u`), the attacker no longer has to guess usernames and can focus every attempt on the password of a known account — drastically reducing the search space.
4. 403 Forbidden = the directory **exists** but access/listing is denied. It is a positive finding: it confirms a valid path (as opposed to a 404), so it is worth trying known file names within it. In fuzzing, 403s are "targets", not failures.
5. A hash is produced by a **one-way** function — there is no inverse operation. Cracking does not decrypt; it **guesses**: it takes candidate passwords (rockyou), computes the hash with the same salt/algorithm (phpass), and compares it with the stored one; when they match, the password has been found. This is why slow algorithms combined with a salt make cracking expensive.
6. (1) **Strong passwords + lockout**: `michael:michael` would not have been cracked with hydra under a password policy and an account lockout after X failed attempts. (2) **Credential protection**: a `wp-config.php` with strict permissions and MySQL bound to `127.0.0.1` keep the database credentials out of reach. (3) **Least privilege for sudo**: removing `NOPASSWD: /usr/bin/python` eliminates the final escalation to root.
7. First `id`/`whoami` (who am I, which groups do I belong to), then `sudo -l` (what can I run as root — often the fastest privesc path) and `ls -la /var/www` plus reading configuration files (`wp-config.php`) for credentials/flags. The rationale: we immediately map privileges and "easy" secrets before searching for more complex paths (SUID, cron, kernel).

## Appendix D — Checklist

- [ ] Confirm the target IP (ARP, not just ping)
- [ ] Scan **all** ports (at least `-sV` and, if necessary, `-p-`)
- [ ] Inspect the source code, headers, `robots.txt`
- [ ] Directory fuzzing with a wordlist
- [ ] Enumerate CMS users
- [ ] Check for weak/reused passwords
- [ ] Read configuration files for credentials
- [ ] Record the flags and the solution path (for the report)
