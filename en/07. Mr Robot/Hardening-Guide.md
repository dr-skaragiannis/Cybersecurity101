# Mr. Robot — Hardening & Remediation Guide

The [README.md](README.md) (§10) recorded six weaknesses, classified by CWE, together with a brief list of countermeasures. This guide is the **practical implementation** of that list: for each weakness it explains **why** it exists, **how** to identify it from inside the system, **which configuration change** remediates it, and **how to demonstrate** that the remediation works.

The Mr. Robot attack chain is typical of WordPress compromises:

```
robots.txt (Key 1 + dictionary) → distinct error messages → hydra with no rate limit
   → elliot:ER28-0652 → Theme Editor → PHP reverse shell (daemon)
   → password.raw-md5 readable → su robot → SUID nmap 3.81 → root
```

**Not a single** step required a vulnerability in the WordPress code itself. Every step relied on **configuration** and **administrative practices**. This means that upgrading alone would **not** have stopped the attack. The configuration changes described in this guide are required.

> ⚠️ **Before you begin:** if you are working on a local VM, take a **snapshot**. If you are working on TryHackMe, your changes are lost when the machine terminates, so keep the commands in a file so that you can reapply them.

> ℹ️ The commands are executed **on the Mr. Robot machine as `root`** (after step 8 of the README). IP addresses are illustrative.

---

## Contents

1. [Map: Attack → Cause → Fix](#1-map-attack--cause--fix)
2. [Inventory: where things are located (Bitnami stack)](#2-inventory-where-things-are-located)
3. [Sanitizing `robots.txt` and the web root](#3-sanitizing-robotstxt-and-the-web-root)
4. [Preventing user enumeration (CWE-204)](#4-preventing-user-enumeration)
5. [Protecting the login page against brute force](#5-protecting-the-login-page-against-brute-force)
6. [Passwords and two-factor authentication (CWE-521)](#6-passwords-and-two-factor-authentication)
7. [Disabling the Theme Editor and locking down files (CWE-94)](#7-disabling-the-theme-editor-and-locking-down-files)
8. [PHP and outbound connections (preventing reverse shells)](#8-php-and-outbound-connections)
9. [The `daemon` service account](#9-the-daemon-service-account)
10. [User home directories and password storage (CWE-327)](#10-user-home-directories-and-password-storage)
11. [SUID on nmap (CWE-250)](#11-suid-on-nmap)
12. [Upgrading the operating system, WordPress, and HTTPS](#12-upgrading-the-operating-system-wordpress-and-https)
13. [Verification](#13-verification)
14. [Checklist & self-assessment questions](#14-checklist--self-assessment-questions)

---

## 1. Map: Attack → Cause → Fix

| Phase (README) | What the attacker exploited | CWE | Fix |
| :--- | :--- | :--- | :--- |
| 2. `robots.txt` | Paths to `key-1-of-3.txt` and `fsocity.dic` | CWE-200 | §3 |
| 3. `hydra -L fs-list` | "Invalid username" vs. "The password you entered for the username…" | CWE-204 | §4 |
| 3. `hydra -l elliot -P fs-list` | Thousands of attempts without any restriction | CWE-307 | §5 |
| 3. `elliot:ER28-0652` | A password contained in a dictionary hosted by the site itself | CWE-521 | §6 |
| 4. Theme Editor → `archive.php` | Modification of PHP code from the dashboard | CWE-94 | §7 |
| 4. `nc -lnvp 1234` | The server connected out to the attacker | — | §8 |
| 4. shell as `daemon` | Apache runs as a generic system account | CWE-250 | §9 |
| 5. `password.raw-md5` | World-readable hash, unsalted MD5 | CWE-327, CWE-732 | §10 |
| 6. `/usr/local/bin/nmap` SUID | `nmap --interactive` → `!sh` | CWE-250 | §11 |

---

## 2. Inventory: Where Things Are Located

The Mr. Robot machine does **not** use the distribution's Apache/PHP/MySQL packages. It uses the **Bitnami WordPress Stack**, a self-contained bundle installed under `/opt/bitnami`. This matters:

* `apt upgrade` does **not** upgrade Bitnami's Apache, PHP, or MySQL.
* The configuration files are **not** located in `/etc/apache2` but under `/opt/bitnami/...`.

Always confirm the paths before changing anything:

```bash
lsb_release -a                                   # Ubuntu 14.04 (Trusty)
ps -eo user,pid,cmd | grep -E '[h]ttpd|[a]pache|[m]ysqld'

# Where WordPress and the configuration files are located
find / -name wp-config.php 2>/dev/null
find / -name httpd.conf 2>/dev/null
find / -name php.ini 2>/dev/null
ls /opt/bitnami/ 2>/dev/null
```

The typical Bitnami paths (confirm them with the commands above):

| Component | Path |
| :--- | :--- |
| WordPress web root | `/opt/bitnami/apps/wordpress/htdocs/` |
| `wp-config.php` | `/opt/bitnami/apps/wordpress/htdocs/wp-config.php` |
| Apache configuration | `/opt/bitnami/apache2/conf/httpd.conf` |
| WordPress vhost configuration | `/opt/bitnami/apps/wordpress/conf/httpd-app.conf` |
| `php.ini` | `/opt/bitnami/php/etc/php.ini` |
| Service control | `/opt/bitnami/ctlscript.sh {start\|stop\|restart} [apache\|mysql]` |

Throughout the remainder of this guide we use the following variable:

```bash
export WP=/opt/bitnami/apps/wordpress/htdocs
```

---

## 3. Sanitizing `robots.txt` and the Web Root

### 3.1 What `robots.txt` Is (and What It Is Not)

`robots.txt` is a **polite request** to search engines: "do not index these paths." It is **not** an access-control mechanism. Anyone can read it, and attackers read it **first**, because it reveals precisely what the administrator wishes to hide.

```bash
cat $WP/robots.txt
# User-agent: *
# fsocity.dic
# key-1-of-3.txt
```

### 3.2 Remediation

**a) Anything that must not be public does not belong in the web root.** Move the files outside it:

```bash
mkdir -p /root/private && chmod 700 /root/private
mv $WP/fsocity.dic $WP/key-1-of-3.txt /root/private/
```

**b) `robots.txt` contains only what is relevant to search engines:**

```bash
cat > $WP/robots.txt <<'EOF'
User-agent: *
Disallow: /wp-admin/
Allow: /wp-admin/admin-ajax.php
EOF
```

> 💡 Even `Disallow: /wp-admin/` reveals that the site runs WordPress. This is acceptable, since WordPress can be fingerprinted from dozens of other indicators anyway. What matters is that the file does not point to **sensitive** files.

**c) Search for other sensitive files in the web root:**

```bash
find $WP -maxdepth 2 -type f \( -name '*.txt' -o -name '*.dic' -o -name '*.bak' \
     -o -name '*.sql' -o -name '*.zip' -o -name '*.tar*' -o -name '*.log' \) -ls
```

Every result that is not part of WordPress is assessed and removed (for example, `license.txt` and `readme.html` are part of WordPress, but they disclose its version).

```bash
rm -f $WP/readme.html $WP/license.txt     # They disclose the WordPress version
```

**d) Disable directory listing** — in WordPress's `httpd-app.conf`:

```apache
<Directory "/opt/bitnami/apps/wordpress/htdocs">
    Options -Indexes +FollowSymLinks
    AllowOverride All
    Require all granted
</Directory>
```

---

## 4. Preventing User Enumeration

### 4.1 The Problem (CWE-204: Observable Response Discrepancy)

WordPress (in that version) responded **differently** depending on the type of error:

| Input | Message |
| :--- | :--- |
| Non-existent user | `ERROR: Invalid username.` |
| Existing user, wrong password | `ERROR: The password you entered for the username elliot is incorrect.` |

This discrepancy allowed `hydra` to test **usernames only** first (11,451 attempts) and then **passwords only** for `elliot` (11,451 attempts). Total: approximately 23,000 attempts. Without the discrepancy, every **combination** would have had to be tested: 11,451 × 11,451 ≈ **131 million**.

> 📌 Unifying the error messages does not "close" a vulnerability, but it multiplies the cost of the attack by a factor of about **5,700**.

### 4.2 Remediation with a Must-Use Plugin

**Must-use plugins** (`mu-plugins`) are loaded automatically by WordPress and **cannot** be deactivated from the dashboard. They are the appropriate place for security settings, because an administrator (or an attacker holding an administrator account) cannot switch them off by accident.

```bash
mkdir -p $WP/wp-content/mu-plugins
cat > $WP/wp-content/mu-plugins/security-hardening.php <<'EOF'
<?php
/*
 * Plugin Name: Security Hardening
 * Description: Uniform login error message and prevention of user enumeration.
 */

// 1. A single, identical message for every failed login
add_filter('login_errors', function () {
    return 'Invalid login credentials.';
});

// 2. A uniform message on the "Lost your password" form as well
add_filter('lostpassword_errors', function ($errors) {
    return new WP_Error('generic', 'If the account exists, an email will be sent.');
});

// 3. Block /?author=N for anonymous visitors
add_action('template_redirect', function () {
    if (is_author() && !is_user_logged_in()) {
        wp_safe_redirect(home_url(), 301);
        exit;
    }
});

// 4. Hide the user list in the REST API from anonymous visitors
add_filter('rest_endpoints', function ($endpoints) {
    if (!is_user_logged_in()) {
        unset($endpoints['/wp/v2/users'], $endpoints['/wp/v2/users/(?P<id>[\d]+)']);
    }
    return $endpoints;
});
EOF

chown root:daemon $WP/wp-content/mu-plugins/security-hardening.php
chmod 640 $WP/wp-content/mu-plugins/security-hardening.php
```

* `add_filter('login_errors', ...)`: WordPress passes the error message through this "filter" before displaying it. We always replace it with the same text.
* The file is owned by `root`, and `daemon` can only **read** it; therefore it can neither modify nor delete it.

> ℹ️ The REST API (`/wp-json/`) was added to WordPress core in version 4.7. It does not exist in the 4.3 release running on Mr. Robot, but the rule will be needed after the upgrade (§12).

### 4.3 Response Time

A subtler form of the same problem: if the password check is performed **only** for existing users, the response for an existing user takes a few milliseconds longer (a timing attack). Newer versions of WordPress mitigate this internally. This is yet another reason to upgrade.

---

## 5. Protecting the Login Page Against Brute Force

`hydra` sent thousands of requests with `-t 30` (30 in parallel). The application did not react at all. We apply **three layers** of restriction.

### 5.1 Layer 1: Restricting Access by IP Address (the Strongest)

If the administrative interface is used only from a specific network, **nobody else** should even be able to see the login page. In `httpd-app.conf`:

```apache
<Files "wp-login.php">
    Require ip 192.168.56.0/24
</Files>

<Directory "/opt/bitnami/apps/wordpress/htdocs/wp-admin">
    <Files "admin-ajax.php">
        Require all granted
    </Files>
    <FilesMatch "^(?!admin-ajax\.php$).*$">
        Require ip 192.168.56.0/24
    </FilesMatch>
</Directory>
```

* Visitors from outside the network receive `403 Forbidden` before the request ever reaches PHP.
* `admin-ajax.php` is exempted, because themes and plugins also use it for public-facing functionality.

### 5.2 Layer 2: A Second HTTP Password (Basic Auth)

If access must be possible from anywhere, we add an **additional** password at the Apache level, in front of the WordPress login form:

```bash
/opt/bitnami/apache2/bin/htpasswd -c /opt/bitnami/apache2/conf/.wp-admin-htpasswd adminuser
chmod 640 /opt/bitnami/apache2/conf/.wp-admin-htpasswd
```

```apache
<Files "wp-login.php">
    AuthType Basic
    AuthName "Restricted"
    AuthUserFile "/opt/bitnami/apache2/conf/.wp-admin-htpasswd"
    Require valid-user
</Files>
```

`hydra http-post-form` now receives `401 Unauthorized` on **every** request, regardless of the WordPress password it tries.

> ⚠️ Basic Auth transmits the password Base64-encoded, not encrypted. It **must** be combined with HTTPS (§12.3).

### 5.3 Layer 3: Limiting Failed Attempts in the Application

Using WP-CLI, we install a login-attempt limiting plugin:

```bash
cd $WP
wp plugin install limit-login-attempts-reloaded --activate --allow-root
```

Settings (Settings → Limit Login Attempts): **3** allowed attempts, a lockout of **20 minutes**, and, after **4** lockouts, an increase to **24 hours**.

With 3 attempts per 20 minutes, the 11,451 passwords in `fs-list` would take more than **50 days** for a single user.

### 5.4 Disabling XML-RPC

The README (§9.2) notes that `wpscan` can use **`xmlrpc.php`** with the `system.multicall` method, which tests **hundreds of passwords in a single request**. This bypasses per-request limits. If it is not in use (for example, by the WordPress mobile app or by Jetpack), we disable it:

```apache
# httpd-app.conf (Apache 2.4 syntax)
<Files "xmlrpc.php">
    Require all denied
</Files>
```

and, in the mu-plugin from §4.2:

```php
add_filter('xmlrpc_enabled', '__return_false');
```

> 💡 The README (§10.2) gives the syntax `Order Allow,Deny / Deny from all`. That is **Apache 2.2** syntax. Apache 2.4 accepts it only if `mod_access_compat` is enabled. `Require all denied` is the correct syntax for 2.4.

### 5.5 Applying the Changes

```bash
/opt/bitnami/apache2/bin/apachectl -t           # Syntax check → Syntax OK
/opt/bitnami/ctlscript.sh restart apache
```

---

## 6. Passwords and Two-Factor Authentication

### 6.1 Why the Password Was Compromised

The password `ER28-0652` was contained **within** `fsocity.dic`, a dictionary hosted by the site itself. More generally, any password that can be found in a dictionary or in a data breach, or that can be inferred from information about the organization (names, dates, employee IDs), is considered weak.

### 6.2 Changing Passwords and Terminating Sessions

```bash
cd $WP
wp user list --role=administrator --fields=ID,user_login,user_email --allow-root

# New random password (32 characters)
NEWPASS=$(openssl rand -base64 24)
wp user update elliot --user_pass="$NEWPASS" --allow-root
echo "$NEWPASS"     # Store it in a password manager

# Log out all existing sessions (including the attacker's)
wp user session destroy --all --allow-root 2>/dev/null || wp config shuffle-salts --allow-root
```

The `wp user session destroy` command is available in newer versions of WP-CLI. Changing the **salts** in `wp-config.php` achieves the same result: all login cookies are invalidated.

### 6.3 Least Privilege for WordPress Accounts

```bash
wp user list --fields=user_login,roles --allow-root
```

An account that only writes posts needs the **Editor** or **Author** role, not **Administrator**. Only an Administrator has access to the Theme Editor (§7). Fewer administrators means fewer accounts that, if compromised, yield RCE.

### 6.4 Two-Factor Authentication (2FA)

Even if the attacker discovers the password, they cannot log in without the **second factor** (a one-time code generated by a mobile app, i.e., TOTP).

```bash
wp plugin install two-factor --activate --allow-root
```

Afterwards, each administrator enables 2FA from their profile (Users → Profile → Two-Factor Options → Authenticator App) by scanning the QR code.

---

## 7. Disabling the Theme Editor and Locking Down Files

### 7.1 The Problem (CWE-94: Code Injection)

The **Theme Editor** (`Appearance → Editor`) allows any Administrator to **write PHP code** that is executed on the server. From a security perspective: **"WordPress administrator password" = "code execution on the server"**.

For the Editor to work, two conditions must hold:

1. The feature must be enabled in WordPress.
2. The web server user (`daemon`) must be able to **write** to the theme files.

We remediate both.

### 7.2 Disabling It in `wp-config.php`

Before the line `/* That's all, stop editing! */`:

```php
define('DISALLOW_FILE_EDIT', true);   // Removes the Theme/Plugin Editor from the menu
define('DISALLOW_FILE_MODS', true);   // ALSO prohibits installing/updating plugins/themes from the dashboard
```

| Setting | What it prevents |
| :--- | :--- |
| `DISALLOW_FILE_EDIT` | Editing PHP files from the dashboard (the attack described in the README) |
| `DISALLOW_FILE_MODS` | Uploading a "plugin" that is in fact a web shell (an alternative RCE path) |

> ⚠️ With `DISALLOW_FILE_MODS`, plugin/theme updates can **no longer** be performed from the dashboard. They are performed by the administrator through **WP-CLI** (`wp plugin update --all`). This is desirable: code changes are made from the terminal, not from the browser.

### 7.3 Locking Down Files: the Web Server Reads, It Does Not Write

```bash
cd $WP
chown -R root:daemon .
find . -type d -exec chmod 750 {} \;
find . -type f -exec chmod 640 {} \;

# Only the uploads directory is writable by the web server
chown -R daemon:daemon wp-content/uploads

# wp-config.php: read-only for the web server
chmod 440 wp-config.php
```

Now, even if `DISALLOW_FILE_EDIT` is removed (e.g., by an attacker with database access who alters the settings), writing to `archive.php` fails: `daemon` has no write permission.

### 7.4 No PHP Execution in Uploads

```apache
# httpd-app.conf
<Directory "/opt/bitnami/apps/wordpress/htdocs/wp-content/uploads">
    <FilesMatch "\.(php|phtml|php\d|phar)$">
        Require all denied
    </FilesMatch>
</Directory>
```

### 7.5 Verifying the Integrity of WordPress Files

After a compromise, we check whether any core file has been modified:

```bash
wp core verify-checksums --allow-root
```

Themes, which are **not** covered by the core checksums, are reinstalled from a clean source:

```bash
grep -l 'fsockopen\|proc_open' $WP/wp-content/themes/*/*.php     # Traces of a reverse shell
wp theme install twentyfifteen --force --allow-root               # Reinstall (before DISALLOW_FILE_MODS)
```

---

## 8. PHP and Outbound Connections

### 8.1 How `php-reverse-shell` Works

The pentestmonkey script does two things:

1. `fsockopen($ip, $port)`: opens a TCP connection **to** the attacker.
2. `proc_open('/bin/sh -i', ...)`: launches a shell and connects its input/output to that connection.

We can block **both**.

### 8.2 Disabling Execution Functions in PHP

In `/opt/bitnami/php/etc/php.ini`:

```ini
disable_functions = exec,passthru,shell_exec,system,proc_open,popen,pcntl_exec,show_source
expose_php        = Off
display_errors    = Off
allow_url_include = Off
open_basedir      = /opt/bitnami/apps/wordpress/htdocs:/opt/bitnami/apps/wordpress/tmp:/tmp
```

* Without `proc_open`, `php-reverse-shell` fails at step 2.
* WordPress core does **not** require any of these functions. Some plugins may. Test the site after making the change.
* `fsockopen` is **not** disabled, because WordPress uses it as a fallback mechanism for HTTP requests. Step 1 is addressed at the network level (§8.3).

```bash
/opt/bitnami/ctlscript.sh restart apache
```

### 8.3 Filtering Outbound Connections

The web server has reason to initiate outbound connections only for specific purposes (DNS, HTTPS for updates). **Never** to port 1234 on an arbitrary host.

```bash
iptables -A OUTPUT -o lo -j ACCEPT
iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
# The web server is allowed to perform DNS and HTTPS (e.g., WordPress updates)
iptables -A OUTPUT -m owner --uid-owner daemon -p udp --dport 53  -j ACCEPT
iptables -A OUTPUT -m owner --uid-owner daemon -p tcp --dport 443 -j ACCEPT
# Anything else new from the web server: log and reject
iptables -A OUTPUT -m owner --uid-owner daemon -m state --state NEW -j LOG --log-prefix "WEB-EGRESS: "
iptables -A OUTPUT -m owner --uid-owner daemon -m state --state NEW -j REJECT

apt-get install iptables-persistent
iptables-save > /etc/iptables/rules.v4
```

A detailed explanation of each parameter is given in [DC-1 §8](../05.%20DC-CTF/Hardening-Guide.md) (restricting outbound connections).

> 💡 A stricter restriction is to allow HTTPS **only** to the IP addresses of `api.wordpress.org` and `downloads.wordpress.org`. Because these IP addresses change, an intermediate proxy with an allowlist of hostnames is typically used instead.

---

## 9. The `daemon` Service Account

### 9.1 The Problem

`daemon` is a **generic** system account that exists on every Linux system for historical reasons. If it is used by multiple services, compromising one of them grants access to the files of all of them. Correct practice is: **one dedicated user per service**.

### 9.2 Remediation

```bash
# Which files/processes currently belong to daemon?
ps -u daemon -o pid,cmd
find / -xdev -user daemon -not -path '/proc/*' 2>/dev/null | head -50

# Dedicated user for the web server, with no shell and no home directory
useradd --system --no-create-home --shell /usr/sbin/nologin wpweb
```

In `/opt/bitnami/apache2/conf/httpd.conf`:

```apache
User  wpweb
Group wpweb
```

Then update the permissions from §7.3, replacing `daemon` with `wpweb`:

```bash
chown -R root:wpweb $WP
chown -R wpweb:wpweb $WP/wp-content/uploads
/opt/bitnami/ctlscript.sh restart apache
ps -eo user,cmd | grep [h]ttpd
```

* `--system`: a system account (UID < 1000) with no password expiry.
* `--shell /usr/sbin/nologin`: if anyone attempts to log in as `wpweb` (e.g., via `su`), they receive "This account is currently not available".

In addition, we remove the shell from `daemon`, if it has one:

```bash
grep '^daemon' /etc/passwd
usermod -s /usr/sbin/nologin daemon
```

---

## 10. User Home Directories and Password Storage

### 10.1 The Problem (CWE-732 and CWE-327)

```text
drwxr-xr-x  robot robot  /home/robot
-rw-r--r--  robot robot  /home/robot/password.raw-md5
```

Two mistakes at once:

1. **Permissions:** the directory (`755`) and the file (`644`) were readable by **every** user, including `daemon`.
2. **Storage:** the password was stored as **unsalted MD5**. MD5 can be computed billions of times per second on a graphics card, and the hashes of simple passwords already exist in public databases (CrackStation). The password `abcdefghijklmnopqrstuvwxyz` was recovered instantly.

### 10.2 Correcting Permissions

```bash
chmod 700 /home/robot
ls -ld /home/*                 # Check all users
```

To ensure that **new** users automatically receive a private directory, set the following in `/etc/adduser.conf`:

```text
DIR_MODE=0700
```

and in `/etc/login.defs`:

```text
UMASK 077
```

The **umask** determines which permissions are **removed** from newly created files. With `077`, new files are created as `600` and directories as `700`, i.e., accessible only by their owner.

### 10.3 Never Store Passwords or Hashes in Text Files

`password.raw-md5` **should not exist**. Linux user passwords are stored **only** in `/etc/shadow`, which is readable only by root:

```bash
shred -u /home/robot/password.raw-md5     # Secure deletion (overwrites the contents before deleting)
ls -l /etc/shadow                          # → -rw-r----- root shadow
```

### 10.4 A New Password with a Modern Algorithm

```bash
grep ENCRYPT_METHOD /etc/login.defs        # Expected: SHA512 (or YESCRYPT on newer systems)
passwd robot
awk -F: '$1=="robot"{print substr($2,1,3)}' /etc/shadow    # → $6$ (salted SHA-512)
```

| Algorithm | Salt | GPU speed (approximate) | Suitable? |
| :--- | :---: | :--- | :---: |
| MD5 (raw) | ✗ | tens of billions/sec | ✗ |
| SHA-512-crypt (`$6$`) | ✓ | hundreds of thousands/sec | ✓ |
| yescrypt (`$y$`) / bcrypt / Argon2id | ✓ | thousands/sec (deliberately slow) | ✓✓ |

The difference between MD5 and yescrypt amounts to a factor of **millions** in the time the attacker requires.

---

## 11. SUID on nmap

### 11.1 Why It Was There and Why It Is Wrong

nmap requires **raw sockets** for certain scan types (e.g., SYN scan, `-sS`), and raw sockets require root privileges. An administrator who wanted users to be able to run SYN scans set the SUID root bit on nmap. Version 3.81, however, included an **interactive mode** (`--interactive`) with a `!sh` command, which opens a shell **with the privileges of the process**, i.e., as root.

The technique of SUID abuse is analyzed in [DC-1 §9](../05.%20DC-CTF/Hardening-Guide.md) (SUID: understanding, auditing, remediation).

### 11.2 Remediation

```bash
ls -l /usr/local/bin/nmap         # -rwsr-xr-x root root
/usr/local/bin/nmap --version     # 3.81

# 1. Remove SUID
chmod u-s /usr/local/bin/nmap

# 2. The tool was installed manually (/usr/local) and is very old: remove it
rm -f /usr/local/bin/nmap
rm -rf /usr/local/share/nmap
```

The nmap binary in `/usr/local/bin` is **not** managed by `apt`, so it was never updated. Software installed outside the package manager is a frequent source of forgotten, vulnerable versions.

### 11.3 If a User Genuinely Needs nmap

**Option A — capabilities instead of SUID:**

```bash
apt-get install nmap                         # Distribution version, without interactive mode
setcap cap_net_raw,cap_net_admin,cap_net_bind_service+eip /usr/bin/nmap
getcap /usr/bin/nmap

# Restriction: only members of the netscan group can execute it
groupadd netscan
chgrp netscan /usr/bin/nmap
chmod 750 /usr/bin/nmap
usermod -aG netscan robot
```

The user runs `nmap --privileged -sS <target>`. The process holds **only** the network capabilities, not full root. A `!sh` (if it existed) would yield a shell **as `robot`**.

> ⚠️ Even with capabilities, nmap can execute **Lua scripts** (NSE, `--script`). With `cap_net_admin`, a user can modify network settings. Grant the tool **only** to users who genuinely need it.

**Option B — a restricted sudo rule:**

```text
# visudo -f /etc/sudoers.d/netscan
robot ALL=(root) /usr/bin/nmap -sS -p 1-1024 192.168.56.0/24
```

The user can run **only** that specific command with those specific parameters.

### 11.4 Full SUID Audit

```bash
find / -xdev \( -perm -4000 -o -perm -2000 \) -type f -exec ls -ld {} \; 2>/dev/null
```

Compare the results with the list of expected SUID binaries in [DC-1 §9.2](../05.%20DC-CTF/Hardening-Guide.md), and establish a baseline with a daily check as described in [DC-1 §9.5](../05.%20DC-CTF/Hardening-Guide.md).

---

## 12. Upgrading the Operating System, WordPress, and HTTPS

### 12.1 Operating System

The machine runs **Ubuntu 14.04 (Trusty)**, whose standard support ended in 2019. The options are:

| Option | Description |
| :--- | :--- |
| **Ubuntu Pro / ESM** | Extended security updates for older releases (`pro attach <token>`), free for personal use on up to 5 machines |
| **Incremental upgrade** | `do-release-upgrade` 14.04 → 16.04 → 18.04 → 20.04 → 22.04 → 24.04, one LTS release at a time |
| **Fresh installation** (recommended) | Ubuntu 24.04 LTS, migrating the WordPress files and a database dump |

```bash
# Update the distribution's packages (does NOT touch Bitnami)
apt-get update && apt-get upgrade
```

### 12.2 The Bitnami Stack and WordPress

Because Apache, PHP, and MySQL reside within `/opt/bitnami`, they are **not** upgraded by `apt`. The Bitnami stack can be upgraded only by performing a **fresh installation** of a newer stack and migrating the data. On a new system, prefer the distribution's packages (`apt install apache2 php mariadb-server`) so that they receive security updates automatically.

For WordPress (4.3.1 on Mr. Robot):

```bash
cd $WP
wp db export /root/wp-backup.sql --allow-root
wp core update --allow-root
wp core update-db --allow-root
wp plugin update --all --allow-root
wp theme update --all --allow-root
```

> ⚠️ WordPress 6.x requires PHP ≥ 7.2.24. The PHP version in the old Bitnami stack may be insufficient, in which case WordPress core will be upgraded only up to the latest compatible version. This is another reason to perform a fresh installation.

### 12.3 HTTPS with a Valid Certificate

The site responded on port 443, but with a self-signed certificate. On a public site we use **Let's Encrypt**:

```bash
apt-get install certbot
certbot certonly --webroot -w $WP -d example.com
```

We also redirect HTTP to HTTPS, so that passwords (including the Basic Auth password from §5.2) are never transmitted in cleartext:

```apache
<VirtualHost *:80>
    ServerName example.com
    Redirect permanent / https://example.com/
</VirtualHost>
```

```php
// wp-config.php
define('FORCE_SSL_ADMIN', true);
```

### 12.4 Firewall

Mr. Robot exposed only ports 80/443 and had port 22 closed. This is **correct**. We enforce it with an explicit rule, so that a future SSH installation does not automatically open the port:

```bash
ufw default deny incoming
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow from 192.168.56.0/24 to any port 22 proto tcp     # Only if SSH is required
ufw enable
```

> ⚠️ `ufw` and the `iptables` rules from §8.3 must be combined correctly. On systems using `ufw`, per-user outbound rules belong in `/etc/ufw/before.rules`.

---

## 13. Verification

From **Kali**, we repeat the attack step by step:

```bash
# 1. robots.txt is clean, no sensitive files
curl -s http://<MR_ROBOT_IP>/robots.txt
curl -I http://<MR_ROBOT_IP>/fsocity.dic                     # → 404
curl -I http://<MR_ROBOT_IP>/key-1-of-3.txt                  # → 404

# 2. Identical message for non-existent and existing users
curl -s -d 'log=noone&pwd=x'  http://<MR_ROBOT_IP>/wp-login.php | grep -o 'Invalid login credentials'
curl -s -d 'log=elliot&pwd=x' http://<MR_ROBOT_IP>/wp-login.php | grep -o 'Invalid login credentials'

# 3. hydra does not even reach the form (403/401) or is locked out after 3 attempts
hydra -l elliot -P fs-list <MR_ROBOT_IP> http-post-form \
  "/wp-login.php:log=^USER^&pwd=^PASS^:F=Invalid" -t 30

# 4. XML-RPC is closed
curl -I http://<MR_ROBOT_IP>/xmlrpc.php                      # → 403
```

On the **Mr. Robot machine**:

```bash
# 5. The Theme Editor does not exist and the web server cannot write to the themes
grep DISALLOW_FILE $WP/wp-config.php
su -s /bin/sh wpweb -c "touch $WP/wp-content/themes/twentyfifteen/archive.php"   # → Permission denied

# 6. PHP does not execute commands
echo '<?php proc_open("id", [], $p);' > $WP/wp-content/uploads/t.php
curl -s http://localhost/wp-content/uploads/t.php            # → 403 (no PHP execution in uploads)
rm $WP/wp-content/uploads/t.php

# 7. The web server does not open outbound connections
su -s /bin/sh wpweb -c 'nc -w 3 <KALI_IP> 1234' ; dmesg | grep WEB-EGRESS | tail -n 2

# 8. robot's home directory is not readable
su -s /bin/sh wpweb -c 'ls /home/robot'                      # → Permission denied

# 9. No SUID nmap
ls -l /usr/local/bin/nmap 2>&1                              # → No such file
find / -xdev -perm -4000 -type f 2>/dev/null
```

---

## 14. Checklist & Self-Assessment Questions

### 14.1 Checklist

- [ ] Snapshot taken, or a file containing the hardening commands
- [ ] Bitnami paths confirmed (`find`)
- [ ] `fsocity.dic`, `key-1-of-3.txt`, `readme.html`, `license.txt` moved out of the web root
- [ ] `robots.txt` free of sensitive paths, `Options -Indexes`
- [ ] mu-plugin: uniform error message, blocking of `?author=` and REST users
- [ ] `wp-login.php`/`wp-admin` restricted by IP or protected with Basic Auth
- [ ] Login-attempt limiting plugin installed, XML-RPC disabled
- [ ] New random password for `elliot`, new salts, 2FA for administrators
- [ ] `DISALLOW_FILE_EDIT` and `DISALLOW_FILE_MODS`
- [ ] WordPress files owned by `root:<web user>`, `750/640`, only uploads writable
- [ ] No PHP execution in uploads, `wp core verify-checksums` clean, theme reinstalled
- [ ] `php.ini`: `disable_functions` (including `proc_open`), `open_basedir`, `expose_php Off`
- [ ] Egress filtering for the web server user
- [ ] Dedicated `wpweb` user instead of `daemon`
- [ ] `/home/robot` set to `700`, `password.raw-md5` deleted, new password (`$6$` or `$y$`), `UMASK 077`
- [ ] nmap 3.81 removed, no unexpected SUID binaries, SUID baseline established
- [ ] Upgrade plan (Ubuntu Pro or fresh installation), WordPress updated, HTTPS
- [ ] All steps in §13 verified

### 14.2 Self-Assessment Questions

1. Why is `robots.txt` **not** a security mechanism? What is the correct way to protect a file that must not be public?
2. Calculate how many attempts `hydra` would need **with** and **without** the discrepancy in error messages, for a dictionary of 11,451 words. What does this show about the value of §4?
3. `DISALLOW_FILE_EDIT` disables the Theme Editor. Why do we **additionally** need the file permissions from §7.3?
4. `php-reverse-shell` uses `fsockopen` and `proc_open`. Why do we disable only the latter in `php.ini`, and how do we deal with the former?
5. What is the difference between unsalted MD5 and SHA-512-crypt? Why does a salt render precomputed tables (rainbow tables) useless?
6. Explain why nmap with `setcap cap_net_raw+eip` is safer than nmap with SUID root. Is there nevertheless any residual risk?
7. Why does software installed under `/usr/local` or `/opt` require particular attention with regard to updates?

### Suggested Answers

1. `robots.txt` is public and merely **asks** crawlers not to index certain paths — it enforces nothing, and anyone can read it, thereby revealing the "hidden" locations. The correct approach is **authentication/authorization** (login, access control), restriction through file or network permissions, or simply not placing the file in the web root at all. Security must not rely on concealing the existence of a resource (security through obscurity).
2. **Without** the message discrepancy, a valid username cannot be distinguished, so every username × password combination must be tested: ≈ **11,451² ≈ 131 million**. **With** the discrepancy (CWE-204), the username is found first in ~11,451 attempts and the password in a further ~11,451 — a total of ~**22,902** (2 × 11,451). This shows that a uniform, non-distinguishing response (§4) reduces the feasibility of the attack by orders of magnitude.
3. `DISALLOW_FILE_EDIT` closes only the **built-in editor** in the WordPress UI. An administrator can still upload a malicious plugin/theme or, if the web server has **write** access to the files, modify them by other means (e.g., via uploads or RCE). The permissions from §7.3 (the web server must **not** be able to write to the code) constitute a second, independent line of defense that prevents a web shell from being written regardless of the UI.
4. `proc_open` (like `exec`, `system`, `shell_exec`, `passthru`, `popen`) executes **external commands** and is rarely needed, so it can be added to `disable_functions` without breaking functionality. `fsockopen` is frequently needed for **legitimate network communication** (APIs, SMTP), so disabling it globally would break applications. We address it with an **egress firewall** (default deny outbound) so that the reverse shell cannot connect outward, together with `open_basedir` and least privilege.
5. Unsalted MD5 is a fast, deterministic function: the same password always produces the same hash, so rainbow tables crack it immediately. SHA-512-crypt is **salted** and performs **many rounds** (it is deliberately slow). The **salt** (a random value per password) makes the hash unique even for identical passwords, so a precomputed table would have to be built **separately for each salt** — practically infeasible — which renders rainbow tables useless.
6. SUID root grants nmap **all** of root's privileges; if an escape is found (interactive `!sh`, GTFOBins), the attacker obtains root. `setcap cap_net_raw+eip` grants **only** the specific capability required (raw sockets for SYN scans) — least privilege. Residual risk: the capability itself permits raw packet crafting and sniffing (monitoring or spoofing traffic), and old versions with interactive mode may still execute commands — but **not as root**. Ideally, nmap should not be installed on production systems at all.
7. Software installed **manually** under `/usr/local` or `/opt` (not through the package manager) is **not updated** by `apt`/`yum` — it falls outside the distribution's security-update cycle. As a result, vulnerable versions are forgotten; manual tracking of versions/CVEs and manual upgrades are required, otherwise such software becomes the weakest point of the system.

---

## Sources

- [WordPress — Hardening WordPress](https://developer.wordpress.org/advanced-administration/security/hardening/)
- [WordPress — Editing wp-config.php: DISALLOW_FILE_EDIT / DISALLOW_FILE_MODS](https://developer.wordpress.org/advanced-administration/wordpress/wp-config/#disable-the-plugin-and-theme-file-editor)
- [WordPress — Must Use Plugins](https://developer.wordpress.org/advanced-administration/plugins/mu-plugins/)
- [WP-CLI — Commands](https://developer.wordpress.org/cli/commands/)
- [MITRE CWE-204 — Observable Response Discrepancy](https://cwe.mitre.org/data/definitions/204.html)
- [MITRE CWE-307 — Improper Restriction of Excessive Authentication Attempts](https://cwe.mitre.org/data/definitions/307.html)
- [OWASP — Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [OWASP — Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- [GTFOBins — nmap](https://gtfobins.github.io/gtfobins/nmap/)
- [Nmap — Running as non-root with capabilities](https://secwiki.org/w/Running_nmap_as_an_unprivileged_user)
- [Ubuntu — Ubuntu Pro / ESM](https://ubuntu.com/pro)
