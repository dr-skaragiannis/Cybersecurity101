# Kioptrix: Level 1 — Hardening & Remediation Guide

The [README.md](README.md) demonstrated **how** Kioptrix 1 was compromised. This guide explains **why** each compromise was possible and **how** to remediate it, using only configuration changes, upgrades, and permission changes on the system itself.

The underlying logic of this guide is as follows: each step of the attack succeeded because of **a configuration error** or **an unpatched package**. If we fix the root cause, the corresponding step can no longer be carried out. If we fix **more than one** problem in the same chain, the attack is stopped even if one of the fixes fails. This principle is known as **Defense in Depth**.

> ⚠️ **Before you begin:** take a **snapshot** of the VM (see lab [00](../00.%20Virtualization/README.md)). This allows you to revert to the vulnerable state, rerun the attack, and observe that it now fails. This "before/after" comparison is the most valuable part of the exercise.

> ℹ️ All commands are executed **on the Kioptrix machine** as `root`. The system is **Red Hat Linux 7.2** (kernel 2.4, without `systemd`): it uses `service`, `chkconfig`, `rpm`, and `iptables`. IP addresses are examples only.

---

## Contents

1. [Map: Attack → Cause → Fix](#1-map-attack--cause--fix)
2. [System Inventory](#2-system-inventory)
3. [Upgrading the Operating System and Packages (mod_ssl / OpenSSL / Samba)](#3-upgrading-the-operating-system-and-packages)
4. [Minimizing the Attack Surface: Services & Firewall](#4-minimizing-the-attack-surface)
5. [Hardening Samba (null session, trans2open)](#5-hardening-samba)
6. [Hardening Apache & mod_ssl (banners, listing, TLS)](#6-hardening-apache--mod_ssl)
7. [Hardening SSH (OpenSSH 2.9, protocol 1)](#7-hardening-ssh)
8. [Verification: Rerunning the Attack](#8-verification)
9. [Checklist & Self-Assessment Questions](#9-checklist--self-assessment-questions)

---

## 1. Map: Attack → Cause → Fix

| Attack step (README) | What the attacker exploited | Root cause | Fix (section) |
| :--- | :--- | :--- | :--- |
| 1. `nmap` | Ports 22, 80, 111, 139, 443, 32768 open to everyone | Unnecessary services, no firewall | §4 |
| 2. `enum4linux` / `rpcclient` | Null session on Samba | `restrict anonymous` not enabled | §5 |
| 3. `dirb` against the web server | Default page & standard Apache directories | Unnecessary modules/files, listing enabled | §6 |
| 4. `searchsploit mod_ssl 2.8` | `mod_ssl 2.8.4 < 2.8.7` (CVE-2002-0082) | **Unpatched mod_ssl / OpenSSL** | §3 |
| 5. `OpenFuck` → **root** | Remote heap overflow in the SSL handshake | Vulnerable version exposed to the network | §3, §4, §6 |
| 5b. `trans2open` (alternative) | `Samba < 2.2.8a` (CVE-2003-0201) | **Unpatched Samba** | §3, §5 |
| — | Banners disclose `Apache/1.3.20`, `mod_ssl/2.8.4` | `ServerTokens Full` | §6 |
| — | `ptrace`/kmod local privilege escalation (second stage) | **Obsolete 2.4 kernel** | §3 |

> 💡 Unlike many CTFs, the central problem here is **not** a misconfiguration but **unpatched software**: a publicly known CVE in `mod_ssl`. The cheapest defense is also the most mundane one: **patch management**.

---

## 2. System Inventory

We cannot protect what we do not know exists. The administrator begins with the same question the attacker asked: **what is running on this machine?**

```bash
# Which distribution and kernel?
cat /etc/redhat-release
uname -r

# Which services are listening on network ports?
netstat -tulpn          # (ss did not exist in this era)

# Which versions of the critical packages are installed?
rpm -q httpd mod_ssl openssl samba openssh-server kernel
```

* `cat /etc/redhat-release`: here it shows **Red Hat Linux 7.2**, a release that has received no support whatsoever since around 2004.
* `uname -r`: kernel **2.4.x**, which has known local privilege escalation vulnerabilities (e.g. `ptrace`) that the second stage of the exploit relies on.
* `netstat -tulpn`: the same view that `nmap` provides, but **from the inside**. The flags: `-t` TCP, `-u` UDP, `-l` listening sockets only, `-p` owning process, `-n` numeric output.
* `rpm -q`: the exact versions. Compare them against public CVEs; this is precisely what an attacker does as well.

Examine the `Local Address` column:

| Value | Meaning |
| :--- | :--- |
| `0.0.0.0:443` or `*:443` | Listening on **all** interfaces, reachable from the network (here, the vulnerable mod_ssl) |
| `127.0.0.1:xxxx` | Listening **locally only**, not visible from outside |

Record the results. You will compare them with the state of the system after hardening.

---

## 3. Upgrading the Operating System and Packages

### 3.1 Why It Matters

`searchsploit` matched the **exact version** `mod_ssl/2.8.4` to CVE-2002-0082 and returned a ready-made exploit. The same applies to Samba and the kernel. **A known vulnerability in outdated software is the cheapest route to compromise**, because the exploit code is already publicly available and requires only… recompilation.

| Target component | Status |
| :--- | :--- |
| Red Hat Linux 7.2 (kernel 2.4) | **End of support** for decades; no security fixes whatsoever |
| `mod_ssl 2.8.4` / `OpenSSL 0.9.6b` | Vulnerable (CVE-2002-0082); `>= 2.8.7` required |
| `Samba < 2.2.8a` | Vulnerable to trans2open (CVE-2003-0201) |
| `Apache 1.3.20` | End-of-life; the 1.3 branch is no longer maintained |

### 3.2 Immediate Fix Within the Same Release (Emergency Patch)

For this specific vulnerability, the immediate action is to **update mod_ssl/OpenSSL** to the latest errata versions available for the distribution:

```bash
# Historically, with Red Hat's tool (requires valid repositories/errata)
up2date -u mod_ssl openssl

# Or directly with the errata RPM (when the repos are offline, as they are today)
rpm -Uvh mod_ssl-2.8.7-*.i386.rpm openssl-0.9.6b-*.i386.rpm
service httpd restart
```

* Target: `mod_ssl >= 2.8.7`, which closes the heap overflow exploited by `OpenFuck`.
* Because the official RH 7.2 repositories are no longer active, in practice an update within the same release is a **half-measure**: it addresses this one vulnerability, but not those discovered later.

### 3.3 The Real Solution: Migrating to a Supported System

An RH 7.2 system with a 2.4 kernel is **essentially unpatchable**: even if we patch mod_ssl, the vulnerable kernel (local privilege escalation via `ptrace`) and dozens of other EOL packages remain. The correct decision in production is a **fresh installation** of a modern, supported distribution (e.g. RHEL/Rocky/Alma 9, Debian 12/13) and **migration of the data only** (site files, database dump), never of the old operating system as a whole.

### 3.4 Automatic Security Updates (on a Supported System)

On a modern RHEL-family system:

```bash
dnf install dnf-automatic
systemctl enable --now dnf-automatic-install.timer
```

The gap between "a fix is published" and "the fix is applied" shrinks from months to hours, and that gap is precisely the window an attacker exploits.

---

## 4. Minimizing the Attack Surface

`nmap` found **six** open ports. Most of them are not needed. Every service that is not listening is a vulnerability that no one can exploit.

### 4.1 Disabling Unnecessary RPC Services (Ports 111 & 32768)

`portmap` (111) and `rpc.statd` (32768) are rarely needed without NFS:

```bash
# Ensure that there are no NFS mounts that depend on them
cat /etc/exports ; mount | grep nfs

# Disable in all runlevels (RH 7.2 / SysV init)
chkconfig portmap off
chkconfig nfslock off          # the rpc.statd daemon
service portmap stop
service nfslock stop
```

### 4.2 Disabling HTTPS If It Is Not Used (Port 443)

The vulnerable component was **mod_ssl**. If the website does not require TLS (or terminates it elsewhere), the simplest defense is not to load the module at all:

```bash
# In httpd.conf / ssl.conf: comment out the module loading and the VirtualHost:443
#   #LoadModule ssl_module    modules/libssl.so
#   #AddModule   mod_ssl.c
service httpd restart
```

> If TLS **is** required, we do not turn it off; we **update** it (§3) and configure it correctly (§6.4). Disabling is an option only when the service is unnecessary.

### 4.3 Firewall with a "Deny Unless Explicitly Allowed" Policy

Using `iptables`, apply default-deny on inbound traffic and explicitly whitelist only what is strictly necessary:

```bash
iptables -F
iptables -P INPUT   DROP
iptables -P FORWARD DROP
iptables -P OUTPUT  ACCEPT

iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# HTTP to everyone (if the web server must be public)
iptables -A INPUT -p tcp --dport 80 -j ACCEPT

# SSH ONLY from the management subnet, not from everywhere
iptables -A INPUT -p tcp -s 192.168.1.0/24 --dport 22 -j ACCEPT

service iptables save        # persist to /etc/sysconfig/iptables
```

Result: even if a vulnerable service remains active, it is **not reachable** from the attacking network.

---

## 5. Hardening Samba

### 5.1 What Went Wrong

`enum4linux` connected via a **null session** (empty username/password), and Samba was a version vulnerable to **trans2open** (CVE-2003-0201), an alternative path to root entirely independent of mod_ssl.

### 5.2 Closing the Null Session & Restricting Access

In `/etc/samba/smb.conf`, in the `[global]` section:

```ini
[global]
   restrict anonymous = 2
   # Bind only to the required interfaces, not to all of them
   interfaces = 192.168.1.10/24
   bind interfaces only = yes
   # Only the management network is allowed
   hosts allow = 192.168.1.
   hosts deny  = 0.0.0.0/0
   # Do not advertise the version
   server string =
```

```bash
testparm                 # check the syntax of smb.conf
service smb restart
```

### 5.3 Upgrading Samba

The configuration restricts access, but the **root cause** of trans2open is the outdated version. We upgrade to `>= 2.2.8a` (ideally to a modern 4.x series following the migration described in §3):

```bash
rpm -Uvh samba-2.2.8a-*.i386.rpm
service smb restart
```

> If the file-sharing role is **not** needed at all on this machine, the best hardening measure is to remove it entirely: `chkconfig smb off && service smb stop`.

---

## 6. Hardening Apache & mod_ssl

### 6.1 Hiding Versions (Banners)

`nmap` read `Apache/1.3.20 (Unix) mod_ssl/2.8.4 OpenSSL/0.9.6b` directly, and this is precisely what led to the exploit. In `httpd.conf`:

```apache
ServerTokens ProductOnly
ServerSignature Off
```

* `ServerTokens ProductOnly`: the banner becomes simply `Apache`, without module versions.
* `ServerSignature Off`: removes the version signature from error pages (404/403).

> ⚠️ This does **not** fix the vulnerability; it merely makes it harder to identify. It complements the patch (§3) and is never a substitute for it.

### 6.2 Disabling Directory Listing & Unnecessary Modules

```apache
<Directory />
    Options -Indexes -FollowSymLinks
    AllowOverride None
</Directory>
```

We also remove unused modules and content (e.g. `mod_autoindex`, `mod_userdir`, and the `/manual`, `/mrtg`, and `/usage` directories discovered by `dirb`), in order to reduce the attack surface and prevent information leakage.

### 6.3 Removing the Default Page & Demo Content

Apache's default "Test Page" betrays an unconfigured server. Replace the web root with the actual content (or an empty `index.html`) and delete the samples:

```bash
rm -rf /var/www/html/manual /var/www/mrtg /var/www/usage
```

### 6.4 Correct TLS Configuration (If HTTPS Remains)

After patching mod_ssl, disable obsolete protocols and ciphers in `ssl.conf`:

```apache
SSLProtocol all -SSLv2 -SSLv3
SSLCipherSuite HIGH:!aNULL:!MD5:!RC4
```

* `OpenFuck` exploits the **SSLv2 handshake**; explicitly disabling `SSLv2` closes this channel as well.

---

## 7. Hardening SSH

### 7.1 What Went Wrong

Port 22 exposed **OpenSSH 2.9p2** with support for **protocol 1**, a cryptographically broken protocol. Although it was not the attack path here, it is a serious exposed weakness.

### 7.2 Configuring `sshd_config`

In `/etc/ssh/sshd_config`:

```
Protocol 2
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
AllowUsers admin
```

```bash
service sshd restart
```

* `Protocol 2`: disables the vulnerable SSH protocol 1.
* `PermitRootLogin no`: `root` cannot log in directly; users must first log in as an unprivileged user and then use `su`/`sudo`.
* `PasswordAuthentication no` + keys: eliminates password brute-forcing.

### 7.3 Upgrading OpenSSH

As with mod_ssl, the real solution is to upgrade to a modern OpenSSH version (as part of the migration described in §3), which does not even support protocol 1 and ships with secure defaults.

---

## 8. Verification

A fix is not considered complete until it has been **demonstrated** that the attack fails. From Kali, we repeat each step:

```bash
# 1. Scan: only port 80 should be visible (and 22 only from the management network)
nmap -sV -p- <KIOPTRIX_IP>

# 2. Banner: must not disclose the mod_ssl version
curl -sI http://<KIOPTRIX_IP>/ | grep -i server

# 3. Null session: must be rejected
enum4linux -a <KIOPTRIX_IP>
rpcclient -U "" -N <KIOPTRIX_IP>          # → NT_STATUS_ACCESS_DENIED

# 4. The mod_ssl exploit: must fail (patched, or port 443 closed)
./OpenFuck 0x6b <KIOPTRIX_IP> -c 50       # → does not spawn a shell

# 5. SSH protocol 1: must be rejected
ssh -1 admin@<KIOPTRIX_IP>                # → Protocol major versions differ
```

On **Kioptrix**:

```bash
# 6. The RPC services are no longer running
chkconfig --list portmap nfslock
netstat -tulpn | egrep '111|32768'        # → empty

# 7. The critical versions have been updated
rpm -q mod_ssl samba openssh-server        # → mod_ssl >= 2.8.7, samba >= 2.2.8a

# 8. The firewall is active with default deny
iptables -L INPUT -n --line-numbers
```

Record the results in a "Before / After" table. This table is the principal deliverable of a remediation report.

---

## 9. Checklist & Self-Assessment Questions

### 9.1 Checklist

- [ ] Snapshot taken before making changes
- [ ] `mod_ssl >= 2.8.7` / `OpenSSL` updated (or port 443 closed); migration plan to a supported system in place
- [ ] `Samba >= 2.2.8a` (or uninstalled)
- [ ] `portmap` / `nfslock` disabled, firewall set to `default deny`, SSH only from the management network
- [ ] `restrict anonymous = 2`, `hosts allow`, `bind interfaces only` configured in Samba
- [ ] `ServerTokens ProductOnly`, `ServerSignature Off`, `Options -Indexes`
- [ ] Default Apache page & demo directories (`manual`, `mrtg`, `usage`) removed
- [ ] `SSLProtocol -SSLv2 -SSLv3` (if TLS remains)
- [ ] SSH: `Protocol 2`, `PermitRootLogin no`, `PasswordAuthentication no`, `MaxAuthTries 3`, `AllowUsers`
- [ ] All steps in §8 verified

### 9.2 Self-Assessment Questions

1. Why is hiding the version (`ServerTokens ProductOnly`) **not** considered a vulnerability fix? What does it actually offer?
2. Kioptrix 1 can be solved via **two** independent paths (mod_ssl and Samba). Which single action in this guide closes **both** simultaneously, and why?
3. Why does patching `mod_ssl` on its own **not** make the system secure? Which component remains vulnerable, and in which phase of the attack is it used?
4. Explain how a `default deny` firewall would have prevented the attack **without** any software upgrade. What is the drawback of relying on it alone?
5. What is the difference between **disabling** a service (`chkconfig off`) and **uninstalling** it (`rpm -e`)? When would you prefer each?
6. Why is an "update within the same release" (§3.2) described as a half-measure for an end-of-life system? What exactly does it fail to cover?
7. `OpenFuck` exploits the SSLv2 handshake. Apart from the patch, which **two** independent measures in this guide would each, on their own, prevent it from executing?

### Suggested Answers

1. It merely hides the **version** in the HTTP headers; it fixes no vulnerability, and the service remains just as vulnerable. It offers a modest reduction in **information disclosure** (it hinders quick version→CVE matching), but an attacker can still detect the vulnerability by other means (the mod_ssl banner, service behavior). The real fix is patching/upgrading.
2. A **host-based firewall with `default deny`** (e.g. iptables) that permits only what is necessary closes **both** paths simultaneously: it blocks access to port 443 (mod_ssl/HTTPS) **and** to ports 139/445 (Samba) from unauthorized networks, without each service having to be patched. It cuts off the common **attack vector (network reachability)**.
3. Patching mod_ssl closes only **one** initial access path (CVE-2002-0082). **Samba (CVE-2003-0201, trans2open)** remains vulnerable and serves as an alternative path to **initial access/RCE**. The attacker simply switches routes; **all** exposed vulnerable services must be patched or closed.
4. A `default deny` firewall permits only approved ports/sources; if the vulnerable services (443, 139/445) are not reachable, the exploit **never even reaches** the vulnerability, without any upgrade. Drawback: it **treats the symptom, not the cause**. The vulnerability still exists and will be exploited if the port is opened, the rule is changed, or the attack originates from a permitted/internal network (hence it belongs in a defense-in-depth strategy, not as the sole measure).
5. `chkconfig off` **disables** automatic startup, but the software **remains installed** (it can be restarted manually or by an update). `rpm -e` **uninstalls** the package entirely, so the vulnerability is removed from disk. Prefer `chkconfig off` when you may need the service later or when dependencies exist; prefer `rpm -e` when it is not needed at all (attack surface reduction, which is the best option for unnecessary vulnerable components).
6. On an end-of-life system, an update within the same (obsolete) release brings only whatever backports exist; but for an EOL system **security patches are no longer released**, so new and known vulnerabilities remain unfixed. It does not cover kernels/libraries that require a newer major version, nor CVEs that were fixed only in supported releases. The real solution is an **upgrade to a supported release** of the operating system.
7. (1) **Disabling SSLv2** (e.g. `SSLProtocol -ALL +TLSv1`, removing SSLv2 ciphers): OpenFuck requires the SSLv2 handshake, which is no longer offered, so it fails without any change to the vulnerable binary. (2) **A `default deny` firewall / closing port 443** (or allowing access only from the management network): the exploit never reaches the service. (Alternatively: disabling/removing mod_ssl if HTTPS is not needed.)

---

## References

- [CVE-2002-0082 — Apache mod_ssl buffer overflow](https://nvd.nist.gov/vuln/detail/CVE-2002-0082)
- [CVE-2003-0201 — Samba trans2open buffer overflow](https://nvd.nist.gov/vuln/detail/CVE-2003-0201)
- [Apache HTTP Server — Security Tips](https://httpd.apache.org/docs/current/misc/security_tips.html)
- [Samba — smb.conf(5)](https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html)
- [OpenSSH — sshd_config(5)](https://man.openbsd.org/sshd_config)
- [Red Hat — Security Hardening Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/security_hardening/index)
- [CIS Benchmarks — Linux / Apache HTTP Server](https://www.cisecurity.org/cis-benchmarks)
