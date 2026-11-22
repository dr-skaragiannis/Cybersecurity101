# Raven 2 — Hardening & Remediation Guide

Raven 2 is the "patched" edition of Raven 1: its author closed the obvious gaps (weak SSH passwords, `sudo python`). The attack described in [README.md](README.md), however, found **a different** path:

```
/vendor/ exposed → PHPMailer 5.2.x → RCE as www-data → wp-config.php → MySQL root
       → MySQL runs as root → UDF → chmod u+s /usr/bin/find → root
```

**The central lesson of Raven 2:** fixing **only** the points exploited by the previous attack is not enough. Systematic hardening of **every** layer is required: code libraries, the web server, the database, and the operating system.

This guide focuses on what is **new** relative to Raven 1. For the shared topics (firewall, SSH, WordPress, `wp-config.php`) it refers to the [Raven 1 Hardening Guide](../03.%20Raven-CTF/Hardening-Guide.md).

> ⚠️ **Before you begin:** take a **snapshot** of the VM. Carry out the attack first, so that you can see the traces it leaves behind (§3), and only then apply the hardening.

> ℹ️ The commands are executed **on Raven 2 as `root`**. The IP addresses are illustrative.

---

## Contents

1. [Map: Attack → Cause → Fix](#1-map-attack--cause--fix)
2. [System inventory](#2-system-inventory)
3. [Post-compromise cleanup (Incident Cleanup)](#3-post-compromise-cleanup)
4. [Understanding the PHPMailer RCE (CVE-2016-10033)](#4-understanding-the-phpmailer-rce)
5. [Upgrading PHPMailer and secure form code](#5-upgrading-phpmailer-and-secure-code)
6. [Web root: permissions that defeat the exploit](#6-web-root-permissions-that-defeat-the-exploit)
7. [Apache: `/vendor/` and directory listing](#7-apache-vendor-and-directory-listing)
8. [MySQL: running as root (the root cause of the escalation)](#8-mysql-running-as-root)
9. [MySQL: UDF, `FILE`, `secure_file_priv`, `plugin_dir`](#9-mysql-udf-file-secure_file_priv-plugin_dir)
10. [MySQL: upgrade and CVE-2016-6662](#10-mysql-upgrade-and-cve-2016-6662)
11. [Operating system: tools, `/tmp`, outbound connections, SUID](#11-operating-system-tools-tmp-outbound-connections-suid)
12. [Verification](#12-verification)
13. [Checklist & self-assessment questions](#13-checklist--self-assessment-questions)

---

## 1. Map: Attack → Cause → Fix

| Attack step (README) | What the attacker exploited | Root cause | Fix |
| :--- | :--- | :--- | :--- |
| 3. `dirb` → `/vendor/` | File listing, `PATH` (Flag 1), `VERSION` | Directory listing, libraries inside the web root | §7 |
| 4. Exploit-DB 40974 | Command execution via the `contact.php` form | PHPMailer < 5.2.18 | §4, §5 |
| 4. `backdoor.php` written to the web root | Persistent web shell | `www-data` can write to `/var/www/html` | §6 |
| 4. `nc -lvp 4444` | Reverse shell to the attacker | No outbound connection policy | §11.3 |
| 5.1 `wp-config.php` | MySQL **root** password | The application uses the database root account | §9.4, [Raven 1 §8](../03.%20Raven-CTF/Hardening-Guide.md#8-hardening-mysql) |
| 5.1 `wget LinEnum.sh` | Downloading tools from Kali | Unrestricted egress, writable/executable `/tmp` | §11 |
| 5.2 `gcc ... 1518.so` | Compiling a malicious library | (on Kali or on the target) compiler available | §11.1 |
| 5.2 `INTO DUMPFILE '/usr/lib/mysql/plugin/'` | Writing a file to the plugins folder | `FILE` privilege, empty `secure_file_priv`, writable `plugin_dir` | §9 |
| 5.2 `do_system('chmod u+s ...')` | System command **as root** | The `mysqld` service runs as `root` | §8 |
| 6. `find raj -exec /bin/sh \;` | Root shell | SUID on `find` (set by the attacker) | §3, §11.4 |

---

## 2. System Inventory

```bash
cat /etc/os-release                         # Debian 8 Jessie
ss -tulpn                                   # Ports 22, 80, 111 and 3306 (local only?)

# Which user runs each service?
ps -eo user,pid,cmd | grep -E 'mysqld|apache2' | grep -v grep
```

The last command is the most important one for Raven 2. If you see:

```text
root      612  /usr/sbin/mysqld --basedir=/usr --datadir=/var/lib/mysql --user=root ...
```

then **every** action performed by the database is carried out with root privileges. This is the critical finding.

```bash
# Which PHPMailer version does the site use?
cat /var/www/html/vendor/VERSION
grep -rn "Version" /var/www/html/vendor/class.phpmailer.php | head -3

# Which MySQL version?
mysql --version
```

---

## 3. Post-Compromise Cleanup

If the system has already been attacked (as in the lab), hardening **is not sufficient**. The attacker has left **three return mechanisms** (persistence) that will continue to work even after the upgrade:

| Artifact | Where | Why it is dangerous |
| :--- | :--- | :--- |
| `backdoor.php` (or `rev.php`) | `/var/www/html/` | Web shell: command execution through the browser |
| `1518.so` + `do_system` function | `/usr/lib/mysql/plugin/`, table `mysql.func` | Command execution via SQL |
| SUID on `/usr/bin/find` | File system | Immediate root for any local user |

### 3.1 Detection

```bash
# PHP files in the web root modified within the last 30 days
find /var/www/html -name '*.php' -mtime -30 -ls

# Files owned by www-data (those created by the web server)
find /var/www/html -user www-data -type f -ls

# Contents of the backdoor
head -c 400 /var/www/html/backdoor.php
```

```sql
-- Inside MySQL (mysql -u root -p)
SELECT * FROM mysql.func;      -- The registered User Defined Functions
SHOW TABLES FROM mysql LIKE 'foo';
```

```bash
ls -la /usr/lib/mysql/plugin/
ls -l /usr/bin/find
ls -la /tmp
```

### 3.2 Removal

```bash
# Keep copies for analysis (forensics) before deleting
mkdir -p /root/evidence && chmod 700 /root/evidence
cp -a /var/www/html/backdoor.php /usr/lib/mysql/plugin/1518.so /tmp/LinEnum.sh /root/evidence/ 2>/dev/null

rm -f /var/www/html/backdoor.php /var/www/html/rev.php
rm -f /tmp/1518.so /tmp/LinEnum.sh /tmp/raj
```

```sql
DROP FUNCTION IF EXISTS do_system;
DROP TABLE IF EXISTS mysql.foo;
```

```bash
rm -f /usr/lib/mysql/plugin/1518.so
chmod u-s /usr/bin/find
```

> 💡 **Why keep copies?** In a real incident, the attacker's files reveal what they did, which IP they connected from (the reverse shell contains their IP), and whether they moved on to other systems.

> ⚠️ In a real environment, after a compromise involving root access, the only **reliable** recovery is a **reinstallation** from a clean source. An attacker with root may have left hidden mechanisms that are not easily detected (e.g., modified executables, cron jobs, SSH keys).

Additional persistence checks:

```bash
cat /root/.ssh/authorized_keys /home/*/.ssh/authorized_keys 2>/dev/null
ls -la /etc/cron.* /var/spool/cron/crontabs/
awk -F: '$3 == 0 {print $1}' /etc/passwd     # Users with UID 0 (should be root only)
```

---

## 4. Understanding the PHPMailer RCE

### 4.1 How PHP Sends Email

PHP's `mail()` function does **not** speak SMTP directly. It invokes the external program **`sendmail`** (or a compatible equivalent such as Exim/Postfix) through the operating system. It accepts a **5th parameter** containing additional arguments for `sendmail`:

```php
mail($to, $subject, $body, $headers, "-f sender@example.com");
//                                     ↑ arguments passed on the sendmail command line
```

The `-f` option sets the sender address (envelope sender).

### 4.2 The Flaw

PHPMailer < 5.2.18 took the sender address **from the contact form** (whatever the visitor typed in the "Email" field) and placed it in `-f`. Validation of the address was inadequate: an address such as the following was accepted as "valid":

```text
"attacker\" -oQ/tmp/ -X/var/www/html/backdoor.php  some"@email.com
```

The quotation marks and the space split the argument, with the result that `sendmail` receives **additional parameters**:

* **`-X /var/www/html/backdoor.php`**: "write all SMTP traffic to this log file".

The message body (which the attacker also controls) contains PHP code, e.g., `<?php system($_GET['cmd']); ?>`. `sendmail` writes it into the "log file", which resides inside the web root and has a `.php` extension. Apache executes it when the attacker opens it in the browser.

### 4.3 Three Preconditions, Three Opportunities for Defense

For the attack to succeed, all of the following are required **simultaneously**:

| Precondition | How we neutralize it |
| :--- | :--- |
| 1. A vulnerable PHPMailer that passes user input into `-f` | Upgrade + fixed sender (§5) |
| 2. `www-data` can **write** to the web root | File permissions (§6) |
| 3. Apache **executes** any `.php` file located there | Restricting PHP execution (§6.3) |

It suffices for **one** of these to be absent for the attack to fail. That is why we fix **all** of them.

---

## 5. Upgrading PHPMailer and Secure Code

### 5.1 Which Version

| Version | Status |
| :--- | :--- |
| < 5.2.18 | Vulnerable to CVE-2016-10033 |
| 5.2.18 – 5.2.19 | The fix can be bypassed (CVE-2016-10045) |
| 5.2.20 – 5.2.27 | Fixed, but the 5.2 branch is no longer supported |
| **6.x (latest)** | **Recommended**. Requires PHP ≥ 5.5 |

### 5.2 Upgrading with Composer (Recommended Method)

**Composer** is PHP's package manager. It knows which libraries the project uses and at which version, so upgrades are performed with a single command.

```bash
apt install composer        # or install from getcomposer.org

# Layout: libraries OUTSIDE the web root
mkdir -p /var/www/app && cd /var/www/app
composer require phpmailer/phpmailer:^6.9

# Check all project libraries for known vulnerabilities
composer audit
```

`composer audit` compares the installed versions against the PHP Security Advisories vulnerability database. On Raven 2 it would have reported CVE-2016-10033 immediately.

We remove the old directory from the web root:

```bash
mv /var/www/html/vendor /root/evidence/vendor-old
```

### 5.3 Secure Code in the Contact Form

The upgrade fixes the library, but the code in `contact.php` must also follow sound practices. The **sender** of an email originating from a contact form is **always** the server. The visitor's address goes only into **Reply-To**:

```php
<?php
require '/var/www/app/vendor/autoload.php';    // Absolute path, outside the web root

use PHPMailer\PHPMailer\PHPMailer;

$email = filter_input(INPUT_POST, 'email', FILTER_VALIDATE_EMAIL);
if ($email === false || $email === null) {
    http_response_code(400);
    exit('Invalid email address.');
}

$mail = new PHPMailer(true);
$mail->setFrom('no-reply@raven.local', 'Raven Security');   // FIXED sender
$mail->addReplyTo($email);                                   // The user's address ONLY here
$mail->addAddress('contact@raven.local');
$mail->Subject = 'New message from the contact form';
$mail->Body    = strip_tags($_POST['message'] ?? '');
$mail->send();
```

* `FILTER_VALIDATE_EMAIL`: rejects addresses containing quotation marks, spaces, and other special characters.
* `setFrom()` with a **fixed** value: nothing supplied by the user ends up on the `sendmail` command line.

### 5.4 Sending via SMTP Instead of `sendmail`

It is even safer not to invoke an external program **at all**. PHPMailer can speak SMTP directly:

```php
$mail->isSMTP();
$mail->Host = '127.0.0.1';      // Local mail server or external relay
$mail->Port = 25;
```

Without a call to `sendmail`, the entire class of "command-line parameter injection" attacks disappears.

---

## 6. Web Root: Permissions That Defeat the Exploit

### 6.1 The Problem

```bash
ls -ld /var/www/html
# drwxrwxrwx  or  drwxr-xr-x www-data www-data
```

If `www-data` is the **owner** of `/var/www/html`, or the folder is `777`, then `sendmail` (which runs as `www-data` when invoked by PHP) can create `backdoor.php` there.

### 6.2 Fix

```bash
cd /var/www/html
chown -R root:www-data .
find . -type d -exec chmod 750 {} \;
find . -type f -exec chmod 640 {} \;

# Exception only for the WordPress uploads
chown -R www-data:www-data wordpress/wp-content/uploads
```

Verification:

```bash
su -s /bin/sh www-data -c 'touch /var/www/html/test.php'
# → touch: cannot touch '/var/www/html/test.php': Permission denied
```

With this configuration, even the **vulnerable** PHPMailer would fail: `sendmail -X /var/www/html/backdoor.php` would be unable to create the file.

### 6.3 Prohibiting PHP Execution in Writable Folders

The attacker could target the **only** writable folder, the uploads. There we prohibit PHP execution:

```apache
# /etc/apache2/conf-available/no-php-uploads.conf
<Directory /var/www/html/wordpress/wp-content/uploads>
    <FilesMatch "\.(php|phtml|php\d|phar)$">
        Require all denied
    </FilesMatch>
    php_admin_flag engine off
</Directory>
```

```bash
a2enconf no-php-uploads
apache2ctl configtest && systemctl reload apache2
```

* `php_admin_flag engine off`: disables the PHP engine for the folder. Unlike `php_flag`, `php_admin_*` **cannot** be overridden by an `.htaccess` file.
* The setting is placed in the Apache configuration, not in `.htaccess`, so that it cannot be removed by someone who gains write access to the folder.

---

## 7. Apache: `/vendor/` and Directory Listing

Flag 1 was found in the file `/vendor/PATH`, and the PHPMailer version in `/vendor/VERSION`. These are two separate problems:

1. **Directory listing**: Apache displayed a file listing.
2. **Libraries in the web root**: the `vendor/` folder was accessible.

The first is fixed as in [Raven 1 §5.2](../03.%20Raven-CTF/Hardening-Guide.md#52-directory-listing-dirbuster) (`Options -Indexes`). The second was resolved in §5.2 by moving the libraries to `/var/www/app/`. As a **second line of defense**, we deny access to such folders in general:

```apache
# /etc/apache2/conf-available/deny-dev-files.conf
<DirectoryMatch "/(vendor|\.git|\.svn|node_modules)(/|$)">
    Require all denied
</DirectoryMatch>
<FilesMatch "^(composer\.(json|lock)|PATH|VERSION|.*\.(bak|old|sql|log|md))$">
    Require all denied
</FilesMatch>
```

```bash
a2enconf deny-dev-files
apache2ctl configtest && systemctl reload apache2
```

Also apply the `ServerTokens Prod`, `ServerSignature Off`, and `expose_php = Off` settings from [Raven 1 §5.4](../03.%20Raven-CTF/Hardening-Guide.md#54-hiding-versions-banners).

---

## 8. MySQL: Running as Root

### 8.1 Why This Is the Most Serious Mistake in Raven 2

A web server is continuously exposed to the risk of compromise; for this reason it runs as **`www-data`**, an unprivileged user. If it is compromised, the attacker obtains the privileges of `www-data`, not those of root. This constitutes **privilege separation**.

On Raven 2, MySQL was running as **root**. The `do_system()` function executes **inside** the `mysqld` process, and therefore with its privileges. Through SQL, the attacker became root.

Had MySQL been running as `mysql` (the normal configuration), `do_system('chmod u+s /usr/bin/find')` would have failed with `Operation not permitted`, because only the file's owner (root) can change its permissions.

### 8.2 Where the User Is Defined

```bash
grep -rn "^user" /etc/mysql/
```

We look for `user = root` in any file. The fix in `/etc/mysql/my.cnf`:

```ini
[mysqld]
user = mysql
```

### 8.3 Fixing File Ownership

Because MySQL was running as root, its data files may have been created as root. If they are left as they are, the service will not start under the new user.

```bash
systemctl stop mysql

chown -R mysql:mysql /var/lib/mysql          # Database data
chown -R mysql:adm   /var/log/mysql 2>/dev/null
chown mysql:mysql    /var/run/mysqld

# Configuration files belong to root and are NOT writable by mysql (see §10)
chown -R root:root /etc/mysql
find /etc/mysql -type f -exec chmod 644 {} \;

systemctl start mysql
ps -eo user,cmd | grep [m]ysqld
# → mysql  /usr/sbin/mysqld ... --user=mysql
```

> 💡 `grep [m]ysqld` is a useful technique: the regular expression `[m]ysqld` matches "mysqld" but not the line of the `grep` command itself (which contains the literal "[m]ysqld"). This avoids the need for `grep -v grep`.

---

## 9. MySQL: UDF, `FILE`, `secure_file_priv`, `plugin_dir`

### 9.1 What User Defined Functions Are

MySQL can be extended with **functions written in C**, which are loaded from `.so` files (shared objects) in the `plugin_dir` folder. This is a legitimate feature. The attack combined **four** weaknesses:

| # | Attack step | What was required | How we check it |
| :--- | :--- | :--- | :--- |
| 1 | `load_file('/tmp/1518.so')` | `FILE` privilege, empty `secure_file_priv` | `SELECT @@secure_file_priv;` |
| 2 | `INTO DUMPFILE '/usr/lib/mysql/plugin/1518.so'` | `mysqld` can write to `plugin_dir` | `ls -ld $(mysql -NBe 'select @@plugin_dir')` |
| 3 | `CREATE FUNCTION do_system ...` | `INSERT` privilege on the `mysql` database | `SHOW GRANTS;` |
| 4 | `do_system('chmod ...')` executed as root | The service runs as root | §8 |

### 9.2 Checking the Current State

```sql
SELECT @@version, @@plugin_dir, @@secure_file_priv;
SELECT user, host, File_priv, Super_priv FROM mysql.user;
```

| Value of `secure_file_priv` | Meaning |
| :--- | :--- |
| empty (`''`) | **Dangerous**: `LOAD_FILE`/`INTO OUTFILE` work in **any** folder |
| `/var/lib/mysql-files/` | They work **only** inside this folder |
| `NULL` | Completely disabled |

### 9.3 Fix

**a) Restricting file operations** — `/etc/mysql/my.cnf`, section `[mysqld]`:

```ini
[mysqld]
user             = mysql
bind-address     = 127.0.0.1
secure-file-priv = /var/lib/mysql-files
local-infile     = 0
skip-symbolic-links
```

```bash
mkdir -p /var/lib/mysql-files
chown mysql:mysql /var/lib/mysql-files
chmod 750 /var/lib/mysql-files
```

**b) The plugins folder is not writable by MySQL:**

```bash
PLUGIN_DIR=$(mysql -u root -p -NBe 'SELECT @@plugin_dir')
chown -R root:root "$PLUGIN_DIR"
chmod 755 "$PLUGIN_DIR"
find "$PLUGIN_DIR" -type f -exec chmod 644 {} \;
```

MySQL needs to **read** plugins, not write new ones. New plugins are installed only by the operating system administrator, as root.

```bash
systemctl restart mysql
```

### 9.4 A Separate User for WordPress, Without `FILE`

As in [Raven 1 §8.3](../03.%20Raven-CTF/Hardening-Guide.md#83-a-dedicated-user-for-the-application-least-privilege):

```sql
CREATE USER 'wp_app'@'localhost' IDENTIFIED BY 'A-Long-Random-Password-2026!';
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, ALTER, INDEX, DROP
  ON wordpress.* TO 'wp_app'@'localhost';

-- The database root account receives a NEW password (R@v3nSecurity has been exposed)
SET PASSWORD FOR 'root'@'localhost' = PASSWORD('Another-Unique-Root-Password!');

-- Remove FILE from every user except root
SELECT user, host FROM mysql.user WHERE File_priv = 'Y';
-- REVOKE FILE ON *.* FROM '<user>'@'<host>';   for any that are listed
FLUSH PRIVILEGES;
```

The `wp_app` user:

* Does **not** have `FILE`, and therefore cannot perform `LOAD_FILE` or `INTO DUMPFILE` (steps 1-2).
* Has **no** privileges on the `mysql` database, and therefore cannot perform `CREATE FUNCTION` (step 3).

Even if the attacker reads `wp-config.php`, **none** of the UDF steps is possible.

---

## 10. MySQL: Upgrade and CVE-2016-6662

### 10.1 What CVE-2016-6662 Is

The README mentions CVE-2016-6662. This vulnerability is directly related to the settings we have applied, so it merits an explanation.

The **`mysqld_safe`** startup script runs as **root** and reads `my.cnf`. `my.cnf` may contain the `malloc-lib=` option, which specifies a library to be loaded at startup. If an attacker manages to **write** to `my.cnf` (e.g., via MySQL's `general_log_file`, which allows the database to write its log to any file you specify), they can add `malloc-lib=/tmp/malicious.so`. On the next restart the library is loaded **as root**, even if `mysqld` runs as `mysql`.

### 10.2 Fix

1. **Upgrade** to MySQL ≥ 5.5.52 / 5.6.33 / 5.7.15 (or the corresponding MariaDB release). These versions reject `malloc-lib` from unsafe paths.

   ```bash
   apt-get update && apt-get install --only-upgrade mysql-server
   mysql --version
   ```

2. **Configuration files belong to root** and are not writable by `mysql` (we did this in §8.3). Confirmation:

   ```bash
   ls -l /etc/mysql/my.cnf /etc/mysql/conf.d/
   ls -la /var/lib/mysql/my.cnf 2>/dev/null     # Must not exist
   ```

   MySQL also reads `my.cnf` from the data directory (`/var/lib/mysql/`), which is **owned** by `mysql`. If a `my.cnf` exists there, it is deleted, and we create an empty, locked file so that it cannot be created again:

   ```bash
   touch /var/lib/mysql/my.cnf
   chown root:root /var/lib/mysql/my.cnf
   chmod 644 /var/lib/mysql/my.cnf
   ```

3. **Only the database root account** can change `general_log_file` (it requires `SUPER`). This already holds if the application user does not have `SUPER` (§9.4).

### 10.3 Long Term

Debian 8 ships MySQL 5.5, which is no longer supported. On a supported Debian 12/13 the default is **MariaDB 10.11 / 11.x**, where `secure_file_priv` and the `mysql` user are configured correctly from the outset.

---

## 11. Operating System: Tools, `/tmp`, Outbound Connections, SUID

### 11.1 Removing Development Tools from the Production Server

A web server **does not need** a compiler. The presence of `gcc` allows the attacker to compile exploits **on the target itself**, for the correct architecture and against the correct libraries.

```bash
dpkg -l | grep -E '^ii\s+(gcc|g\+\+|make|cc|build-essential)'
apt-get purge gcc g++ make build-essential
apt-get autoremove
```

If removal is not possible (e.g., another package depends on them), we restrict their execution to root only:

```bash
chmod 700 /usr/bin/gcc* /usr/bin/cc /usr/bin/make 2>/dev/null
```

### 11.2 `/tmp` Without Execution and SUID

The attack used `/tmp` for `LinEnum.sh` and `1518.so`. We apply the `nosuid,nodev,noexec` options as explained in [DC-1 §10](../04.%20DC-CTF/Hardening-Guide.md#10-mount-options):

```text
# /etc/fstab
tmpfs  /tmp      tmpfs  defaults,nosuid,nodev,noexec,size=512M  0 0
tmpfs  /dev/shm  tmpfs  defaults,nosuid,nodev,noexec            0 0
```

```bash
mount -o remount /tmp ; mount -o remount /dev/shm
```

In addition, with `secure-file-priv = /var/lib/mysql-files` (§9.3), MySQL can **no longer** read `/tmp/1518.so`.

### 11.3 Restricting Outbound Connections

The web server initiated **two** outbound connections: the reverse shell (`nc` on port 4444) and the download of `LinEnum.sh` (`wget`). Per-user egress filtering is explained in detail in [DC-1 §8](../04.%20DC-CTF/Hardening-Guide.md#8-restricting-outbound-connections). On Raven 2 we add a specific rule: the contact form needs to send email, so we allow **only** SMTP to the mail server:

```bash
iptables -A OUTPUT -o lo -j ACCEPT
iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
# Only SMTP to the internal mail relay (illustrative IP)
iptables -A OUTPUT -m owner --uid-owner www-data -p tcp -d 192.168.56.10 --dport 25 -j ACCEPT
iptables -A OUTPUT -m owner --uid-owner www-data -m state --state NEW -j LOG --log-prefix "WWW-EGRESS: "
iptables -A OUTPUT -m owner --uid-owner www-data -m state --state NEW -j REJECT

# mysql has NO reason whatsoever to open outbound connections
iptables -A OUTPUT -m owner --uid-owner mysql -m state --state NEW -j REJECT

apt-get install iptables-persistent && iptables-save > /etc/iptables/rules.v4
```

### 11.4 SUID: Baseline and Daily Audit

On Raven 2, the SUID bit on `find` **was added by the attacker**. A daily audit against a baseline would have detected it. See the script in [DC-1 §9.5](../04.%20DC-CTF/Hardening-Guide.md#95-periodic-auditing-against-a-baseline).

### 11.5 The Remaining Measures (Shared with Raven 1)

Also apply the following from the [Raven 1 Hardening Guide](../03.%20Raven-CTF/Hardening-Guide.md):

* §3 — package updates / migration to a supported Debian release
* §4 — disabling `rpcbind`, firewall `default deny`
* §6 — WordPress: upgrade, user enumeration, `wp-config.php` set to `640`, `DISALLOW_FILE_EDIT`
* §7 — key-based SSH, `MaxAuthTries`, PAM

---

## 12. Verification

Restore the **vulnerable snapshot** on a second VM for comparison, or repeat the attack against the hardened machine.

From **Kali**:

```bash
# 1. /vendor/ is not accessible
curl -I http://<RAVEN2_IP>/vendor/              # → 403 or 404
curl -I http://<RAVEN2_IP>/vendor/PATH          # → 403 or 404

# 2. Exploit 40974 fails
python 40974.py
curl -I http://<RAVEN2_IP>/backdoor.php         # → 404 (the file was not created)
```

On **Raven 2**:

```bash
# 3. www-data cannot write to the web root
su -s /bin/sh www-data -c 'touch /var/www/html/x.php'          # → Permission denied

# 4. MySQL runs as mysql
ps -eo user,cmd | grep [m]ysqld

# 5. File operations are restricted
mysql -u root -p -e "SELECT @@secure_file_priv;"                 # → /var/lib/mysql-files/
mysql -u root -p -e "SELECT load_file('/etc/passwd');"           # → NULL

# 6. The plugins folder is not writable
mysql -u root -p -e "SELECT 'x' INTO DUMPFILE '/usr/lib/mysql/plugin/test.so';"
# → ERROR 1290: The MySQL server is running with the --secure-file-priv option

# 7. The application user cannot create functions
mysql -u wp_app -p -e "CREATE FUNCTION x RETURNS INTEGER SONAME 'x.so';"
# → ERROR 1044: Access denied

# 8. No UDFs, no SUID find
mysql -u root -p -e "SELECT * FROM mysql.func;"                  # → empty
ls -l /usr/bin/find                                              # → -rwxr-xr-x
```

---

## 13. Checklist & Self-Assessment Questions

### 13.1 Checklist

- [ ] Snapshot taken, copies of evidence stored in `/root/evidence`
- [ ] Removed `backdoor.php`, `1518.so`, `do_system`, the `foo` table, and SUID on `find`
- [ ] Checked for persistence (SSH keys, cron, UID 0 users)
- [ ] PHPMailer 6.x via Composer, outside the web root, `composer audit` clean
- [ ] `contact.php`: fixed `setFrom`, user address only in `Reply-To`, `FILTER_VALIDATE_EMAIL`
- [ ] Web root set to `root:www-data`, `750/640`; `www-data` cannot write
- [ ] No PHP execution in uploads (`php_admin_flag engine off`)
- [ ] `Options -Indexes`, blocking of `vendor/`, `.git/`, `PATH`, `VERSION`
- [ ] MySQL: `user = mysql`, `bind-address = 127.0.0.1`, `secure-file-priv`, `local-infile = 0`
- [ ] `plugin_dir` set to `root:root 755`, `/etc/mysql` not writable by `mysql`
- [ ] `wp_app` user without `FILE`/`SUPER`, new database root password
- [ ] MySQL ≥ 5.5.52 (or a plan to migrate to MariaDB on a supported distribution)
- [ ] `gcc`/`make` removed, `/tmp` mounted with `noexec,nosuid,nodev`
- [ ] Egress filtering for `www-data` and `mysql`
- [ ] SUID baseline and daily audit
- [ ] Raven 1 measures (firewall, SSH, WordPress)
- [ ] Verification of all steps in §12

### 13.2 Self-Assessment Questions

1. PHP's `mail()` invokes an external program. Explain how this allowed CVE-2016-10033 to write a file to disk.
2. Name three **independent** measures from §5 and §6, each of which is sufficient on its own to make exploit 40974 fail.
3. Why must the visitor's email address be placed in `Reply-To` and not in `From`/`Sender`?
4. The UDF attack required four preconditions (§9.1). Which of them is the **most critical**, i.e., the one whose absence renders the other three harmless to the operating system?
5. What exactly does the `secure_file_priv` setting control, and what is the difference between an empty value and the value `NULL`?
6. Why can MySQL be at risk from CVE-2016-6662 **even if** it runs as the `mysql` user? Which measure in §10 addresses this?
7. Why is simply upgrading PHPMailer **not sufficient** on a system that has already been compromised?

### Suggested Answers

1. `mail()` does not send email by itself; it invokes the external binary **`/usr/sbin/sendmail`**, passing arguments to it. PHPMailer placed the sender address into the **additional parameters** (`-f...`) without adequate sanitization, so the attacker could "break out of" the argument and append `-X /path/shell.php`. As a result, sendmail wrote a file with attacker-controlled content to disk — a webshell.
2. (1) **Upgrading PHPMailer** to ≥5.2.18 (fixed sanitizer) — the injection no longer succeeds. (2) **Strict server-side validation of the email address** (`filter_var`/regex) so that values containing spaces/`-X` are rejected. (3) **A webroot that is not writable by the web server** (permissions + `open_basedir`) — even if the injection succeeds, no `.php` file is written to the accessible directory. Each of these on its own stops 40974.
3. `From`/`Sender` are used by PHPMailer to construct the arguments passed to sendmail (envelope/`-f`), so a malicious value there becomes exploitable (injection). `Reply-To` is a plain message header that is **not** passed as an execution parameter. By placing the visitor's address in `Reply-To`, we retain the functionality (replying to the visitor) without exposing the dangerous path.
4. The most critical one is that **MySQL runs as root** on the OS. If it runs as the ordinary `mysql` user, then even if the attacker has the `FILE` privilege, writes the `.so`, and loads the UDF, command execution occurs with the privileges of `mysql` — **not root** — so there is no privilege escalation on the operating system. The other three preconditions become harmless to the OS.
5. `secure_file_priv` restricts **the directory in which** file operations (`LOAD_FILE`, `INTO OUTFILE/DUMPFILE`) are permitted. An **empty value (`''`)** = **no restriction** (dangerous). A **path** = operations are permitted only inside that folder. **`NULL`** = file operations are **completely disabled** (most secure). Hence empty ≠ NULL: the former opens everything, the latter closes everything.
6. CVE-2016-6662 allows an attacker with database access to make MySQL load a **malicious configuration file** (via `general_log` → writing to `my.cnf`), injecting a `malloc_lib`/plugin that is executed **at mysqld startup** — and because startup is often performed by init/`mysqld_safe` **as root**, root is obtained even if the regular process runs as `mysql`. Mitigation (§10): configuration files (`my.cnf`) must **be owned by root and not be writable** by `mysql`.
7. The upgrade closes the **entry path**, but if the system has already been compromised, the attacker has most likely left **persistence** mechanisms (webshells, cron jobs, backdoor users, stolen credentials/keys) that remain after the patch. Incident response is required: identification/removal of artifacts, rotation of all passwords/keys and, ideally, a **rebuild** from a clean image.

---

## References

- [NVD — CVE-2016-10033 (PHPMailer)](https://nvd.nist.gov/vuln/detail/CVE-2016-10033), [CVE-2016-10045](https://nvd.nist.gov/vuln/detail/CVE-2016-10045)
- [PHPMailer — GitHub & Security notes](https://github.com/PHPMailer/PHPMailer/blob/master/SECURITY.md)
- [NVD — CVE-2016-6662 (MySQL)](https://nvd.nist.gov/vuln/detail/CVE-2016-6662)
- [MySQL — secure_file_priv](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_secure_file_priv)
- [MySQL — Security Guidelines](https://dev.mysql.com/doc/refman/8.0/en/security-guidelines.html)
- [Composer — audit command](https://getcomposer.org/doc/03-cli.md#audit)
- [Apache HTTP Server — Security Tips](https://httpd.apache.org/docs/2.4/misc/security_tips.html)
