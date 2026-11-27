# Raven 1 — Hardening & Remediation Guide

The [README.md](README.md) demonstrated **how** Raven 1 was compromised. This guide explains **why** each compromise was possible and **how** to remediate it, using only configuration changes, upgrades, and permission changes on the system itself.

The reasoning behind this guide is as follows: each step of the attack was made possible by **a configuration error**. If we correct that error, the corresponding step can no longer be carried out. If we correct **more than one** error in the same chain, the attack is stopped even if one of the fixes fails. This principle is known as **Defense in Depth**.

> ⚠️ **Before you begin:** take a **snapshot** of the VM (see lab [00](../00.%20Virtualization/README.md)). This allows you to revert to the vulnerable state, rerun the attack, and confirm that it now fails. This "before/after" comparison is the most valuable part of the exercise.

> ℹ️ All commands are executed **on the Raven machine** as `root` (e.g., after `su -`). The IP addresses shown are illustrative.

---

## Contents

1. [Map: Attack → Cause → Remediation](#1-map-attack--cause--remediation)
2. [System Inventory (what we are working with)](#2-system-inventory)
3. [Upgrading the Operating System and Packages](#3-upgrading-the-operating-system-and-packages)
4. [Minimizing the Attack Surface: Services & Firewall](#4-minimizing-the-attack-surface)
5. [Hardening Apache (HTML comments, listing, banners)](#5-hardening-apache)
6. [Hardening WordPress (version, user enumeration, wp-config)](#6-hardening-wordpress)
7. [Hardening SSH & Password Policy (hydra, michael:michael)](#7-hardening-ssh--password-policy)
8. [Hardening MySQL (unrestricted root)](#8-hardening-mysql)
9. [Fixing sudo (NOPASSWD python)](#9-fixing-sudo)
10. [Password Hashes & Password Reuse (John the Ripper)](#10-password-hashes--password-reuse)
11. [Verification: Rerunning the Attack](#11-verification)
12. [Checklist & Self-Assessment Questions](#12-checklist--self-assessment-questions)

---

## 1. Map: Attack → Cause → Remediation

| Attack step (README) | What the attacker exploited | Root cause | Remediation (section) |
| :--- | :--- | :--- | :--- |
| 2. `nmap` | Ports 22, 80, 111 open to everyone | Unnecessary services, no firewall | §4 |
| 3. Flag 1 in the HTML | `<!-- flag -->` comment in the page source | Sensitive information sent to the client | §5 |
| 4. DirBuster | Discovery of `/wordpress/`, `/vendor/` | Directory listing, development files in the web root | §5 |
| 5. WPScan | WP version, users `michael`, `steven` | Outdated WordPress, unrestricted user enumeration | §3, §6 |
| 6. `hydra` → SSH | `michael:michael` | No limit on attempts, password login, weak password | §7 |
| 7. `wp-config.php` → MySQL `root` | Database root password in a world-readable file | The application uses the MySQL `root` account | §6, §8 |
| 8. `sudo python` | `(ALL) NOPASSWD: /usr/bin/python` | Misconfigured sudoers rule | §9 |
| 8b. `john` | `steven`'s hash cracked with `rockyou.txt` | Weak password, password reuse | §10 |

> 💡 Note that **none** of the steps required an "advanced" exploit. All of them were **misconfigurations**. For this reason, fixing them is primarily a matter of discipline rather than cost.

---

## 2. System Inventory

We cannot protect something we do not know exists. An administrator always begins with the same question the attacker asked: **what is running on this machine?**

```bash
# Which distribution and version are we running?
cat /etc/os-release
uname -r

# Which services are listening on network ports?
ss -tulpn
```

* `cat /etc/os-release`: shows the distribution. On Raven 1 it is **Debian 8 (Jessie)**, a release that no longer receives security updates.
* `uname -r`: the kernel version. An old kernel implies known local privilege escalation vulnerabilities.
* `ss -tulpn`: shows the same information as `nmap`, but **from the inside**. The flags mean: `-t` TCP, `-u` UDP, `-l` listening ports only, `-p` the process that owns the socket, `-n` numeric output instead of names.

Examine the `Local Address` column:

| Value | Meaning |
| :--- | :--- |
| `0.0.0.0:22` or `*:22` | Listens on **all** interfaces, and is therefore reachable from the network |
| `127.0.0.1:3306` | Listens **locally only** and is not visible from outside (this is what we want for MySQL) |

```bash
# Which users have a login shell?
grep -vE 'nologin|false' /etc/passwd

# Which sudo rules exist?
cat /etc/sudoers; ls -la /etc/sudoers.d/
```

Record the results. You will compare them with the state of the system after hardening.

---

## 3. Upgrading the Operating System and Packages

### 3.1 Why It Matters

WPScan identified the exact WordPress version and correlated it with known vulnerabilities (CVEs). The same applies to Apache, PHP, MySQL, and the kernel. **A known vulnerability in outdated software is the cheapest route to compromise**, because a public exploit already exists.

The lifecycle of a distribution is significant:

| Distribution | Status |
| :--- | :--- |
| Debian 8 Jessie (Raven 1) | **End of life**. Its repositories have been moved to `archive.debian.org` |
| Debian 12 Bookworm / 13 Trixie | Supported; they receive security updates |

### 3.2 Immediate Update Within the Same Release

Because Jessie has been archived, `apt update` fails with the original repositories. We point them to the archive instead:

```bash
cp /etc/apt/sources.list /etc/apt/sources.list.bak
cat > /etc/apt/sources.list <<'EOF'
deb http://archive.debian.org/debian jessie main
deb http://archive.debian.org/debian-security jessie/updates main
EOF

apt-get -o Acquire::Check-Valid-Until=false update
apt-get upgrade
```

* We keep a backup copy (`.bak`) before modifying a configuration file. This is good practice for every change.
* `Acquire::Check-Valid-Until=false`: archived repositories carry an "expired" timestamp signature. This option instructs `apt` to accept it. **This is acceptable only in a lab environment**.
* `apt-get upgrade`: installs the latest versions ever released for Jessie.

### 3.3 The Real Solution: Migrating to a Supported Release

Updating within Jessie closes only **some** security gaps, not all of them, because vulnerabilities discovered after the end of support will never receive a fix. In a production environment, the correct decision is a **fresh installation** of Debian 12/13 followed by migration of the application (files and a copy of the database — a database dump). A gradual, sequential upgrade (Jessie → Stretch → Buster → Bullseye → Bookworm) is feasible, but laborious and error-prone.

### 3.4 Automatic Security Updates (on a Supported System)

```bash
apt install unattended-upgrades
dpkg-reconfigure -plow unattended-upgrades
```

This package installs **only** security updates, on a daily basis. As a result, the gap between "a fix is published" and "the fix is applied" shrinks from months to hours.

---

## 4. Minimizing the Attack Surface

### 4.1 Disabling rpcbind (Port 111)

`rpcbind` is required only for RPC services such as **NFS**. The web server does not use them. Every unnecessary service is additional code that may contain a vulnerability and additional information for the attacker.

```bash
# Confirm that no NFS mounts depend on it
mount | grep nfs

# Disable it (Debian 8 uses systemd)
systemctl stop rpcbind.socket rpcbind.service
systemctl disable rpcbind.socket rpcbind.service
systemctl mask rpcbind.service
```

* `stop`: stops the service immediately.
* `disable`: prevents it from starting at the next boot.
* `mask`: "locks" it so that it cannot be started even as a dependency of another service.
* We also disable the `.socket` unit because, through socket activation, systemd would otherwise restart the service as soon as a connection arrived on port 111.

If the service is not needed at all: `apt purge rpcbind nfs-common`.

### 4.2 Firewall with a "Deny Unless Allowed" Policy

The correct philosophy is **default deny**: block everything and open only what is required.

```bash
apt install ufw

ufw default deny incoming       # All incoming traffic is rejected
ufw default allow outgoing      # Outgoing traffic is allowed (we will tighten this later if needed)

ufw allow 80/tcp                # HTTP for the site
ufw allow 443/tcp               # HTTPS (once a certificate has been configured)

# SSH only from the management subnet, NOT from everywhere
ufw allow from 192.168.56.0/24 to any port 22 proto tcp

ufw enable
ufw status verbose
```

> ⚠️ **Pay attention to the order of operations.** If the machine is administered over SSH, the rule for port 22 must be added **before** running `ufw enable`; otherwise, access to the machine will be lost (a lockout).

**Why restrict SSH to a subnet?** The attacker ran `hydra` from the same network as the website. Had SSH accepted connections only from a management network, the brute-force attack would never have started.

In addition, `ufw` provides built-in **rate limiting**:

```bash
ufw limit 22/tcp
```

The `limit` rule blocks an IP address that opens **6 or more connections within 30 seconds**. Since `hydra` opens dozens of parallel connections, it would be cut off immediately.

---

## 5. Hardening Apache

### 5.1 Sensitive Comments in the HTML Source (Flag 1)

**The problem:** anything the server sends to the browser can be viewed by **anyone** using `F12` or `Ctrl+U`. HTML comments (`<!-- -->`) are not hidden; they are simply not rendered on the page.

**Check:** search all static files for comments:

```bash
grep -rn '<!--' /var/www/html --include=*.html --include=*.php | less
```

**Remediation:** remove every comment that contains information (passwords, paths, usernames, developer notes). In a sound development process, comments are stripped automatically during the "build" stage, before deployment to the server.

### 5.2 Directory Listing (DirBuster)

**The problem:** when a directory has no `index.html`/`index.php`, Apache with the `Indexes` option displays **a list of all its files**. The attacker sees the entire structure without even having to guess filenames.

**Check:**

```bash
grep -rn "Indexes" /etc/apache2/
curl -s http://localhost/vendor/ | head
```

If `curl` returns `Index of /vendor`, directory listing is enabled.

**Remediation:** in `/etc/apache2/apache2.conf`, within the `/var/www/` block:

```apache
<Directory /var/www/>
    Options -Indexes +FollowSymLinks
    AllowOverride All
    Require all granted
</Directory>
```

* `-Indexes`: the `-` sign **removes** the option. Apache now responds with `403 Forbidden` instead of a file listing.
* `AllowOverride All`: required so that WordPress can use `.htaccess` for permalinks.

### 5.3 Development Files Inside the Web Root (`/vendor/`)

The `vendor/` directory (Composer libraries, e.g., PHPMailer) must **never** be accessible from the browser. The same applies to `.git/`, `composer.json`, `*.bak`, and `*.sql`.

```apache
# /etc/apache2/conf-available/deny-dev-files.conf
<DirectoryMatch "/(vendor|\.git|node_modules)/">
    Require all denied
</DirectoryMatch>

<FilesMatch "\.(bak|sql|old|swp|log|md|json|lock)$">
    Require all denied
</FilesMatch>
```

```bash
a2enconf deny-dev-files
```

An even better solution is to **move** `vendor/` outside `/var/www/html` and load it from the PHP code using an absolute path.

### 5.4 Hiding Versions (Banners)

`nmap -sV` and Apache's error pages disclose version information (`Apache/2.4.10 (Debian)`). Hiding it **does not fix** any vulnerability, but it makes reconnaissance slower and noisier.

```apache
# /etc/apache2/conf-available/security.conf
ServerTokens Prod        # Header: "Server: Apache" without version/OS
ServerSignature Off      # No signature at the bottom of error pages
TraceEnable Off          # Disables the HTTP TRACE method

Header always set X-Content-Type-Options "nosniff"
Header always set X-Frame-Options "SAMEORIGIN"
```

```bash
# Likewise for PHP: hides the "X-Powered-By: PHP/5.x" header
sed -i 's/^expose_php = On/expose_php = Off/' /etc/php5/apache2/php.ini

a2enmod headers
apache2ctl configtest      # ALWAYS check the syntax before restarting
systemctl reload apache2
```

> 💡 The `apache2ctl configtest` command must return `Syntax OK`. If a `reload` is performed with an invalid configuration, Apache may fail to start and the website may go offline.

---

## 6. Hardening WordPress

### 6.1 Upgrading Core, Plugins, and Themes

The WordPress installation on Raven 1 is version **4.8.x** (2017). WPScan reported dozens of known vulnerabilities for this version.

Using **WP-CLI** (the official command-line tool):

```bash
# Install WP-CLI
curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
chmod +x wp-cli.phar && mv wp-cli.phar /usr/local/bin/wp

cd /var/www/html/wordpress

# Always back up before upgrading
sudo -u www-data wp db export /root/wp-backup.sql
tar czf /root/wp-files-backup.tgz /var/www/html/wordpress

sudo -u www-data wp core version
sudo -u www-data wp core update
sudo -u www-data wp plugin update --all
sudo -u www-data wp theme update --all
sudo -u www-data wp core verify-checksums
```

* We run the commands as `www-data` (`sudo -u`) so that no files owned by root are created.
* `wp core verify-checksums`: compares every core file against the official checksums. If any file has been tampered with (e.g., the attacker planted a backdoor), it will show up here.

> ⚠️ Recent versions of WordPress require PHP 7.2.24+ (8.x is recommended). Debian 8 ships PHP 5.6. This is yet another reason why migrating to a supported distribution (§3.3) is necessary.

### 6.2 Preventing User Enumeration (WPScan `-e u`)

WPScan found `michael` and `steven` through two main methods:

1. **Author archives:** `/?author=1` redirects to `/author/michael/`.
2. **REST API:** `/wp-json/wp/v2/users` returns a list of users in JSON.

We block both in the WordPress `.htaccess` file (above the `# BEGIN WordPress` block):

```apache
<IfModule mod_rewrite.c>
RewriteEngine On
# 1. Block ?author=N
RewriteCond %{QUERY_STRING} (^|&)author=\d+ [NC]
RewriteRule ^ - [F,L]
# 2. Block the user list in the REST API
RewriteRule ^wp-json/wp/v2/users - [F,L]
</IfModule>
```

* `RewriteCond`: the condition. It checks whether the query string contains `author=` followed by a number.
* `RewriteRule ^ - [F,L]`: `-` means "do not change the URL", `F` returns `403 Forbidden`, and `L` stops further rule processing.

```bash
a2enmod rewrite && systemctl reload apache2
```

**In addition:** login names (`user_login`) should not be identical to public display names (`display_name`). If the public name is "Michael", the login name should be something unpredictable.

### 6.3 Protecting `wp-config.php`

**The problem:** `michael` (an unprivileged SSH user) was able to read `wp-config.php` because it was world-readable (`644`).

**Fixing the permissions:**

```bash
cd /var/www/html/wordpress
ls -l wp-config.php                 # Before: -rw-r--r-- (644) → readable by everyone

chown root:www-data wp-config.php
chmod 640 wp-config.php
ls -l wp-config.php                 # After: -rw-r----- root www-data
```

| Digit | Who | Permission | Why |
| :---: | :--- | :--- | :--- |
| `6` | owner (`root`) | read + write | Only the administrator modifies it |
| `4` | group (`www-data`) | read | Apache/PHP needs to read it |
| `0` | others | none | `michael` and every other user cannot see it |

**Do not use the MySQL `root` account in WordPress.** We will create a separate user with restricted privileges in §8.3.

**Additional settings in `wp-config.php`:**

```php
define('DISALLOW_FILE_EDIT', true);   // Disables the theme/plugin editor in the dashboard
define('FORCE_SSL_ADMIN', true);      // wp-admin over HTTPS only (once a certificate is in place)
define('WP_DEBUG', false);            // No error messages that reveal file paths
```

We also regenerate the **salts** (the keys that sign login cookies), so that all existing sessions are terminated, possibly including the attacker's:

```bash
sudo -u www-data wp config shuffle-salts
```

### 6.4 File Permissions for the Entire WordPress Installation

The rule is: **the web server reads, it does not write** (except for the uploads directory).

```bash
cd /var/www/html/wordpress
chown -R root:www-data .
find . -type d -exec chmod 750 {} \;
find . -type f -exec chmod 640 {} \;

# Only the uploads directory needs to be writable by the web server
chown -R www-data:www-data wp-content/uploads
```

We also forbid PHP execution inside the uploads directory, so that an uploaded `shell.php` file cannot run:

```apache
# /var/www/html/wordpress/wp-content/uploads/.htaccess
<FilesMatch "\.(php|phtml|php\d)$">
    Require all denied
</FilesMatch>
```

---

## 7. Hardening SSH & Password Policy

This was the **critical** entry point: `hydra` found `michael:michael`. Three independent errors contributed:

1. SSH accepted **passwords** instead of keys.
2. There was no **limit on failed attempts**.
3. The password was **identical to the username**.

### 7.1 Key-Based Authentication Instead of Passwords

An Ed25519 key is practically impossible to brute-force, unlike a password that may appear in `rockyou.txt`.

**On the administrator's workstation (not on Raven):**

```bash
ssh-keygen -t ed25519 -C "admin@lab"     # Generates a private/public key pair
ssh-copy-id michael@<RAVEN_IP>           # Copies the public key to ~/.ssh/authorized_keys
ssh michael@<RAVEN_IP>                   # Test: must log in WITHOUT a password
```

> ⚠️ **Password authentication must not be disabled until you have confirmed that key-based login works.** Otherwise, you risk losing access to the system entirely.

### 7.2 Configuring `sshd_config`

```bash
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
nano /etc/ssh/sshd_config
```

```text
PermitRootLogin no               # root cannot log in directly, only via su/sudo
PasswordAuthentication no        # Keys only: hydra has nothing to try
PubkeyAuthentication yes
PermitEmptyPasswords no
ChallengeResponseAuthentication no
MaxAuthTries 3                   # Maximum of 3 attempts per connection
LoginGraceTime 30                # 30 seconds to complete the login
MaxStartups 10:30:60             # Limits concurrent unauthenticated connections
AllowUsers michael steven        # Allowlist: only these users may log in
X11Forwarding no
ClientAliveInterval 300
ClientAliveCountMax 2            # Disconnect idle sessions after ~10 minutes
```

Important explanations:

* **`MaxStartups 10:30:60`**: once there are 10 open unauthenticated connections, `sshd` begins rejecting new ones with a probability of 30%, which increases linearly up to 100% at 60 connections. This mechanism drastically slows down tools that open many parallel connections, such as `hydra -t 16`.
* **`AllowUsers`**: even if an account with a weak password is created later (e.g., a service account), it will not be able to log in via SSH.

```bash
sshd -t                     # Syntax check. No output = OK
systemctl reload ssh
```

> 💡 Keep your current SSH session **open** and test logging in from a **second** terminal. If something goes wrong, you still have access to fix it.

### 7.3 Strong Password Policy (PAM)

Even with key-based SSH, passwords are still used for `sudo`, `su`, and the console. **PAM** (Pluggable Authentication Modules) validates every new password before accepting it.

```bash
apt install libpam-pwquality
```

In `/etc/security/pwquality.conf`:

```text
minlen = 14          # Minimum length of 14 characters
minclass = 3         # At least 3 character classes (lowercase, uppercase, digits, symbols)
maxrepeat = 3        # No more than 3 identical consecutive characters
usercheck = 1        # Rejects a password that contains the username (michael:michael!)
dictcheck = 1        # Rejects dictionary words
enforce_for_root
```

The `usercheck = 1` setting would have prevented **exactly** the password `michael`.

### 7.4 Account Lockout After Failed Attempts

PAM can temporarily lock an account after consecutive failures. On Debian 8 the module is `pam_tally2` (on newer systems, `pam_faillock`).

**Debian 8 — `/etc/pam.d/common-auth`** (at the top of the file):

```text
auth required pam_tally2.so onerr=fail deny=5 unlock_time=900
```

**Debian 11+ / Ubuntu 22.04+ — `/etc/security/faillock.conf`:**

```text
deny = 5
unlock_time = 900
fail_interval = 900
```

After 5 failures the account is locked for 15 minutes. At a rate of 5 attempts per 15 minutes, testing the 14 million passwords in `rockyou.txt` would take approximately 80 years.

```bash
pam_tally2 --user michael            # How many failures the user has (Debian 8)
pam_tally2 --user michael --reset    # Unlock by the administrator
```

### 7.5 Changing the Exposed Passwords

The passwords `michael` and `pink84` are now **known**. They must be changed:

```bash
passwd michael
passwd steven
passwd root
chage -d 0 michael     # Forces the user to change the password at the next login
```

---

## 8. Hardening MySQL

### 8.1 What Went Wrong

From `wp-config.php`, the attacker obtained the password of the **MySQL `root`** account (`R@v3nSecurity`). With it, they read **all** databases, extracted hashes, and (on Raven 2) wrote files to disk. WordPress does not need any of these privileges.

### 8.2 Basic Hardening with `mysql_secure_installation`

```bash
mysql_secure_installation
```

The tool performs the following steps in sequence:

1. Changes the password of the database `root` account (answer **Y** and set a **new** password).
2. Removes anonymous users.
3. Disallows `root` login from remote hosts.
4. Removes the `test` database.
5. Reloads the privilege tables.

### 8.3 A Dedicated User for the Application (Least Privilege)

```sql
-- Connect as the database root: mysql -u root -p
CREATE USER 'wp_app'@'localhost' IDENTIFIED BY 'A-Long-Random-Password-2026!';

GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, ALTER, INDEX, DROP
  ON wordpress.* TO 'wp_app'@'localhost';

FLUSH PRIVILEGES;
SHOW GRANTS FOR 'wp_app'@'localhost';
```

* `'wp_app'@'localhost'`: the user can connect **only** from the same server.
* `ON wordpress.*`: privileges **only** on the `wordpress` database, not on `mysql` (where the database users' passwords are stored).
* We do **not** grant `FILE` (reading/writing files on disk), `SUPER`, `GRANT OPTION`, or `PROCESS`.

We update `wp-config.php`:

```php
define('DB_USER', 'wp_app');
define('DB_PASSWORD', 'A-Long-Random-Password-2026!');
```

Even if the attacker reads `wp-config.php` again, they will have access only to the WordPress database, not to the entire database server.

### 8.4 `mysqld` Settings

In `/etc/mysql/my.cnf` (or `/etc/mysql/mysql.conf.d/mysqld.cnf`), under the `[mysqld]` section:

```ini
[mysqld]
user             = mysql      # The service runs as the mysql user, NOT as root
bind-address     = 127.0.0.1  # Listens locally only; not reachable from the network
local-infile     = 0          # Disables LOAD DATA LOCAL INFILE
secure-file-priv = /var/lib/mysql-files   # Restricts LOAD_FILE / INTO OUTFILE to a single directory
symbolic-links   = 0
```

```bash
mkdir -p /var/lib/mysql-files && chown mysql:mysql /var/lib/mysql-files && chmod 750 /var/lib/mysql-files
systemctl restart mysql

# Verification: which user is running MySQL?
ps -o user,cmd -C mysqld
```

The **`user = mysql`** setting is critical: on Raven, MySQL ran as `root`, so anything the database did (e.g., writing a file) was performed with root privileges. The exploitation of this flaw is analyzed in the [Raven 2 Hardening Guide](../06.%20Raven2-CTF/Hardening-Guide.md).

---

## 9. Fixing sudo

### 9.1 Why `NOPASSWD: /usr/bin/python` Is Equivalent to root

```text
User steven may run the following commands on raven:
    (ALL) NOPASSWD: /usr/bin/python
```

The administrator may have reasoned, "I am giving steven access to Python only." However, Python is a **full programming language**: it can spawn a shell, read/write any file, and change passwords. The same applies to `perl`, `ruby`, `vim`, `less`, `find`, `awk`, `nmap`, `tar`, and others. The **GTFOBins** website catalogs hundreds of such programs.

> 📌 **Rule:** grant sudo on a program **only** if you are certain that it cannot execute other commands or write arbitrary files.

### 9.2 Check

```bash
sudo -l -U steven               # What steven is allowed to run
grep -rn NOPASSWD /etc/sudoers /etc/sudoers.d/
```

### 9.3 Remediation

**Always** use `visudo`; never edit the file directly:

```bash
visudo                       # For /etc/sudoers
visudo -f /etc/sudoers.d/steven
```

`visudo` validates the syntax before saving. A syntax error in `sudoers` can disable `sudo` entirely.

**Remove** the line:

```text
steven ALL=(ALL) NOPASSWD: /usr/bin/python
```

If `steven` genuinely needs to perform a specific task as root (e.g., reloading Apache), grant him **exactly** that:

```text
steven ALL=(root) /bin/systemctl reload apache2
```

* Without `NOPASSWD`: the user must enter his password. Thus, if someone hijacks his session, they do not automatically gain privileges.
* A specific command **with** its arguments: he cannot run `systemctl` with any other arguments.

If he needs to run **his own** script as root, the script must be owned by root and must not be writable by `steven`:

```bash
chown root:root /usr/local/sbin/backup.sh
chmod 755 /usr/local/sbin/backup.sh     # steven can execute it, but NOT modify it
```

### 9.4 Additional sudo Hardening Settings

```text
Defaults    use_pty                         # sudo runs in its own pseudo-terminal
Defaults    logfile="/var/log/sudo.log"     # Logs every use of sudo
Defaults    timestamp_timeout=5             # The password is "remembered" for only 5 minutes
Defaults    passwd_tries=3
```

---

## 10. Password Hashes & Password Reuse

### 10.1 What John the Ripper Revealed

John cracked `steven`'s hash (`pink84`) because the password appeared in `rockyou.txt`. **The problem was not the algorithm; it was the password.** No hashing algorithm can protect a password that appears in a public wordlist: the attacker simply computes the hash of each word and compares.

The second problem was **reuse**: the same password was valid in WordPress **and** in the operating system.

### 10.2 Remediation

1. **Unique passwords per system.** WordPress password ≠ Linux password ≠ MySQL password ≠ root password. Use a password manager (e.g., KeePassXC).
2. **Stronger algorithms.** Since version 6.8, WordPress stores passwords using **bcrypt** instead of phpass, so upgrading (§6.1) helps here as well. Existing passwords are converted automatically at the user's next login.
3. **Strong hashing on Linux.** Check the algorithm in `/etc/shadow`:

   ```bash
   awk -F: '$2 ~ /^\$/ {print $1, substr($2,1,4)}' /etc/shadow
   ```

   | Prefix | Algorithm | Assessment |
   | :--- | :--- | :--- |
   | `$1$` | MD5-crypt | Unsuitable |
   | `$6$` | SHA-512-crypt | Acceptable |
   | `$y$` | yescrypt | Default on Debian 11+; the best option |

   In `/etc/login.defs`, set `ENCRYPT_METHOD SHA512` (or `YESCRYPT` on newer systems). This applies to **new** passwords, which is why the password changes in §7.5 should be made **after** this setting.

4. **A separate root password and a locked root account.** Since administrators use `sudo`, the direct root account can be locked:

   ```bash
   passwd -l root      # Locks the root password. `su -` no longer works
   ```

   > ⚠️ Do this **only** after confirming that at least one user has full sudo privileges; otherwise, you will lose administrative access.

---

## 11. Verification

A remediation is not considered complete until it has been **demonstrated** that the attack fails. From Kali, repeat each step:

```bash
# 1. Scan: only port 80 should be visible (and port 22 only from the management network)
nmap -sV -p- <RAVEN_IP>

# 2. Directory listing: should return 403
curl -I http://<RAVEN_IP>/vendor/

# 3. User enumeration: should return 403
curl -I "http://<RAVEN_IP>/wordpress/?author=1"
curl -I http://<RAVEN_IP>/wordpress/wp-json/wp/v2/users

# 4. SSH with a password: should return "Permission denied (publickey)"
ssh -o PreferredAuthentications=password michael@<RAVEN_IP>

# 5. Hydra: should fail immediately
hydra -l michael -P /usr/share/wordlists/rockyou.txt ssh://<RAVEN_IP>
```

On **Raven**:

```bash
# 6. michael cannot read wp-config
sudo -u michael cat /var/www/html/wordpress/wp-config.php     # → Permission denied

# 7. steven no longer has sudo python
sudo -l -U steven

# 8. MySQL runs as mysql and listens locally only
ps -o user,cmd -C mysqld
ss -tlnp | grep 3306                                         # → 127.0.0.1:3306
```

Record the results in a "Before / After" table. This table is the key deliverable of a remediation report.

---

## 12. Checklist & Self-Assessment Questions

### 12.1 Checklist

- [ ] Snapshot taken before making changes
- [ ] Packages updated / migration plan to a supported Debian release
- [ ] `rpcbind` disabled, firewall set to `default deny`, SSH only from the management network
- [ ] HTML comments removed, `Options -Indexes`, `/vendor` blocked
- [ ] `ServerTokens Prod`, `ServerSignature Off`, `expose_php Off`
- [ ] WordPress updated, user enumeration blocked, `DISALLOW_FILE_EDIT`
- [ ] `wp-config.php` set to `640 root:www-data`, new salts
- [ ] SSH: keys, `PasswordAuthentication no`, `PermitRootLogin no`, `MaxAuthTries 3`, `AllowUsers`
- [ ] PAM: `pwquality` (`usercheck=1`) and lockout after 5 failures
- [ ] All exposed passwords changed
- [ ] MySQL: new root password, `wp_app` user with minimal privileges, `bind-address 127.0.0.1`, `user = mysql`
- [ ] `sudoers`: `NOPASSWD: /usr/bin/python` removed
- [ ] All steps in §11 verified

### 12.2 Self-Assessment Questions

1. Why is hiding the Apache version (`ServerTokens Prod`) **not** considered a vulnerability fix? What does it ultimately provide?
2. `michael`'s password was identical to his username. Which **three** independent measures in this guide would each, on its own, have prevented `hydra` from succeeding?
3. Why should `wp-config.php` have `www-data` as its group rather than being owned entirely by `www-data`?
4. Explain why `sudo` on `/usr/bin/python` is equivalent to full root access. Name three other programs with the same property.
5. What is the difference between `systemctl disable` and `systemctl mask`?
6. If `steven`'s password had been stored with bcrypt instead of phpass, would John the Ripper have failed? Justify your answer.
7. Which MySQL privilege enables the `LOAD_FILE()` and `INTO DUMPFILE` commands, and why should the application user not hold it?

### Suggested Answers

1. It hides only the **version information** in the HTTP headers; it does not fix any bug — the server remains just as vulnerable if it is running an outdated version. What it provides is a reduction in **information disclosure** (it makes mapping version → known CVE more difficult); in other words, it is a noise-reduction measure, not remediation. The real fix is updating the software.
2. (1) A **password policy** (PAM `pwquality` with `usercheck=1`) that rejects a password identical to the username. (2) **Account lockout** (pam_faillock/tally, locking after 5 failures), which stops brute-forcing. (3) **`PasswordAuthentication no` in SSH** (keys only) — hydra no longer has a password to guess. Each one alone is sufficient.
3. The web server must be able to **read** the file but not **modify** it. With owner `root`, group `www-data`, and permissions `640`, www-data has read-only access. If the file were owned entirely by www-data, a compromise of the application (RCE as www-data) would allow the config to be **rewritten** (malicious code, altered credentials). Owner root = the compromised process cannot tamper with it.
4. `sudo` runs Python with EUID 0, and Python is a **full interpreter** that executes arbitrary code — so `os.system("/bin/bash")` yields a root shell; privileges over an interpreter = privileges over **everything**. Other examples (GTFOBins): `perl`, `ruby`, `awk` (as well as `find`, `vim`, `less`, `nmap`).
5. `disable` removes only the **automatic start at boot**; the service can still be started manually or as a dependency. `mask` links it to `/dev/null` so that it **cannot be started by any means** until `unmask`. For services we definitely want inactive (e.g., `rpcbind`), we use `mask`.
6. Not necessarily — John supports bcrypt. The difference is **speed**: bcrypt with a high cost factor is deliberately much slower per attempt, so the number of guesses per second drops sharply. A weak password (`pink84`, which is in rockyou) would be cracked regardless, whereas a strong password hashed with bcrypt would become practically infeasible to crack. bcrypt raises the cost; it does not make cracking weak passwords impossible.
7. The **`FILE`** privilege. It allows reading (`LOAD_FILE()`) and writing (`INTO OUTFILE/DUMPFILE`) files with the privileges of the mysqld process — the basis for UDF/webshell attacks. The application user needs only SELECT/INSERT/UPDATE/DELETE on its own database; `FILE` is global and dangerous (least privilege), and we additionally restrict it with `secure_file_priv`.

---

## Sources

- [Debian Security Manual — Securing Debian](https://www.debian.org/doc/manuals/securing-debian-manual/)
- [WordPress — Hardening WordPress](https://developer.wordpress.org/advanced-administration/security/hardening/)
- [OpenSSH — sshd_config(5)](https://man.openbsd.org/sshd_config)
- [MySQL — Security Guidelines](https://dev.mysql.com/doc/refman/8.0/en/security-guidelines.html)
- [GTFOBins — python](https://gtfobins.github.io/gtfobins/python/)
- [CIS Benchmarks — Debian Linux / Apache HTTP Server](https://www.cisecurity.org/cis-benchmarks)
