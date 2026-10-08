# Anonymous Logins for Pentesters: FTP, SMB and NFS File-Share Walkthrough

## Introduction

File sharing protocols power the back office of nearly every enterprise: FTP for legacy data drops, SMB for Windows-compatible file servers, and NFS for Unix-to-Unix exports. Each protocol also ships with an anonymous or guest access mode, and administrators routinely enable one of those modes for convenience and forget to lock it down. That is why misconfigured shares appear again and again as high-impact internal findings. Unauthenticated access can expose source code, backup archives, configuration files, credentials, and personal data that no perimeter control can recover after the fact.

This guide builds a deliberately vulnerable lab on Ubuntu, exposes all three services to unauthenticated access, and then enumerates each one from an attacker machine with Nmap, the standard FTP client, `smbclient`, and NetExec. The workflow is intentionally repetitive: confirm the exposure, list what is available, retrieve a marker file, and then understand the server-side setting responsible for the exposure. Each protocol section therefore pairs attack output with the defensive control that would have stopped it.

**Lab topology.** The exercises assume two virtual machines on an isolated host-only network. The target runs Ubuntu 22.04 LTS with `vsftpd`, Samba, and the NFS kernel server installed. The attacker is a Debian-family security distribution with Nmap, FTP and SMB clients, NetExec, and NFS client tools installed. The IP addresses used throughout are `192.168.1.9` for the target and `192.168.1.17` for the attacker.

| Role | Operating system | Address | Notes |
| --- | --- | --- | --- |
| Target | Ubuntu 22.04 LTS server | `192.168.1.9` | `vsftpd`, Samba, NFS kernel server, hostname `ubuntu-lab` |
| Attacker | Debian-based security distro | `192.168.1.17` | nmap, `ftp`, `smbclient`, NetExec, NFS utilities |
| Network | Host-only / NAT lab segment | `192.168.1.0/24` | No route to the internet from the lab |

> **Authorisation and scope.** Everything described here is an attack technique. Run it only on machines you own or for which you hold written authorisation — a purpose-built virtual lab is the right place to practise, and it is the only place these commands are safe. Unauthenticated enumeration, guest-share access, file retrieval, NFS mounting, and privilege-escalation testing are clearly detectable, aggressive behaviours on a production network, and performing them without permission is a criminal offence in most jurisdictions. The defensive countermeasures are given alongside each technique precisely so that this material can also be read from the other side of the fight.

### How to read the transcripts

The terminal sessions in this guide follow the conventions of the shell itself, and a few of them are worth learning before the first command:

* A prompt ending in `#` means the shell is running as the root account (`root@kali:~#`), while a prompt ending in `$` means an ordinary user. Both the target setup and the attacker-side enumeration are shown as root because package installation, service management, NFS mounting, and raw-socket scanning require privileges.
* The transcript `root@ubuntu-lab:~#` is the target machine during lab construction. The transcript `root@kali:~#` is the attacker machine during enumeration.
* Output shown here was captured on Debian-family systems and is representative rather than byte-identical: package versions, port numbers, timestamps, MAC addresses, directory dates, and tool banners will differ on your own lab. The structure of the output is what matters.
* File contents shown without a prompt are the literal bytes of the file, as you would see them in an editor.
* When a command is long enough to be broken across two lines in third-party tool documentation, it is always shown here as a single line you can paste.

### Table of contents

1. [Why unauthenticated file shares matter](#1-why-unauthenticated-file-shares-matter)
2. [Lab setup — target, attacker and network](#2-lab-setup--target-attacker-and-network)
3. [Anonymous FTP — configuring vsftpd](#3-anonymous-ftp--configuring-vsftpd)
4. [Anonymous FTP — enumeration and retrieval](#4-anonymous-ftp--enumeration-and-retrieval)
5. [Guest SMB — configuring Samba](#5-guest-smb--configuring-samba)
6. [Guest SMB — enumeration and retrieval](#6-guest-smb--enumeration-and-retrieval)
7. [Insecure NFS — configuring exports](#7-insecure-nfs--configuring-exports)
8. [Insecure NFS — enumeration, download and mounting](#8-insecure-nfs--enumeration-download-and-mounting)
9. [Hardening summary](#9-hardening-summary)
10. [Quick reference cheat sheet](#10-quick-reference-cheat-sheet)
11. [Practice exercises](#11-practice-exercises)

---

## 1. Why unauthenticated file shares matter

The three protocols in this guide expose data in three different ways, but the underlying mistake is the same: a server trusts a network client without verifying who that client is. FTP exposes an anonymous root directory through the unprivileged FTP account. SMB exposes shares marked as guest-accessible. NFS exposes exported paths while trusting the UID claimed by the client. Each of those behaviours can be useful in a narrowly controlled situation, and each becomes a finding as soon as it is reachable from an untrusted network segment.

The operational consequence is that file-share enumeration belongs early in internal testing. Before password guessing, before exploitation, and before post-exploitation, an operator should ask whether any machine simply gives files away. The commands are fast, the evidence is unambiguous, and the retrieved files often contain the credentials or configuration details needed for the next phase. A single world-readable backup or deployment script can therefore shorten an engagement more than a long brute-force run.

It is also useful to distinguish availability from authorisation. A share may be intentionally available while still requiring authentication; that is normal. The vulnerability here is not that FTP, SMB, or NFS exists, but that an anonymous or guest identity can browse or retrieve data meant for authenticated users. Throughout this walkthrough, the decisive question is therefore not “is the port open?” but “what can an unauthenticated client do after connecting?”

---

## 2. Lab setup — target, attacker and network

Before testing begins, both virtual machines must be able to reach each other and nothing else. Put them on the same isolated host-only or NAT lab segment, assign the target `192.168.1.9` and the attacker `192.168.1.17`, and take a snapshot before installing vulnerable services. The snapshot matters because this guide deliberately weakens defaults; you want a clean restore point before and after the attack phase.

The target needs three server packages:

```bash
root@ubuntu-lab:~# apt update
Hit:1 http://archive.ubuntu.com/ubuntu jammy InRelease
Hit:2 http://security.ubuntu.com/ubuntu jammy-security InRelease
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
All packages are up to date.
```

Install the FTP, SMB, and NFS servers in the protocol sections below rather than all at once. That order keeps each misconfiguration associated with the service that introduced it, and it makes the later hardening checks easier to interpret.

The attacker needs the corresponding clients and enumeration tools. On a Debian-family security distribution these are commonly installed already; on a minimal system, install Nmap, a command-line FTP client, Samba client utilities, NetExec, and NFS client tools. Verify basic connectivity first, because every later failure is easier to diagnose once the network path is known to work:

```bash
root@kali:~# ping -c 2 192.168.1.9
PING 192.168.1.9 (192.168.1.9) 56(84) bytes of data.
64 bytes from 192.168.1.9: icmp_seq=1 ttl=64 time=0.52 ms
64 bytes from 192.168.1.9: icmp_seq=2 ttl=64 time=0.48 ms

--- 192.168.1.9 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.480/0.500/0.520/0.020 ms
```

A sub-millisecond round-trip time is the usual signature of two virtual machines on the same host or lab segment. If ping fails, stop and fix virtual networking before continuing; an unreachable target will make every service-specific error misleading.

---

## 3. Anonymous FTP — configuring vsftpd

`vsftpd`, the Very Secure FTP Daemon, is the default FTP server on many Debian-derived systems. Out of the box it listens on TCP port 21, supports local-user logins, and disables anonymous access. The lab reverses that last default and configures a passive anonymous share that any network neighbour can read.

### 3.1 Installing vsftpd

Install the daemon package:

```bash
root@ubuntu-lab:~# apt install vsftpd
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following NEW packages will be installed:
  vsftpd
0 upgraded, 1 newly installed, 0 to remove and 0 not upgraded.
Need to get 142 kB of archives.
After this operation, 385 kB of additional disk space will be used.
Do you want to continue? [Y/n] y
Get:1 http://archive.ubuntu.com/ubuntu jammy/main amd64 vsftpd amd64 3.0.5-0ubuntu1 [142 kB]
Fetched 142 kB in 1s (198 kB/s)
Preconfiguring packages ...
Selecting previously unselected package vsftpd.
Preparing to unpack .../vsftpd_3.0.5-0ubuntu1_amd64.deb ...
Unpacking vsftpd (3.0.5-0ubuntu1) ...
Setting up vsftpd (3.0.5-0ubuntu1) ...
Processing triggers for man-db (2.10.2-1) ...
```

The single-package footprint is one reason teams reach for `vsftpd`: the daemon has few dependencies and conservative defaults. That simplicity is also why configuration review matters. There is not much to audit, so an auditor should actually audit all of it.

### 3.2 Reviewing the default configuration

The server configuration lives in `/etc/vsftpd.conf`. Before changing anything, inspect the directives that control listeners and anonymous access:

```bash
root@ubuntu-lab:~# grep -nE '^(#)?(listen|listen_ipv6|anonymous_enable|local_enable)=' /etc/vsftpd.conf
12:listen=NO
18:listen_ipv6=YES
27:#anonymous_enable=NO
29:local_enable=YES
```

The important line is the anonymous-access default. With anonymous access disabled, none of the FTP enumeration in section 4 would work. The listener settings determine whether the daemon listens on IPv4, IPv6, or both; the packaged Ubuntu default uses the IPv6 listener while still accepting IPv4 connections through the dual-stack socket. Record the “before” state now, because section 9 reverses the lab change by restoring a restrictive value.

### 3.3 Enabling anonymous access

Open the configuration file:

```bash
root@ubuntu-lab:~# nano /etc/vsftpd.conf
```

Change the anonymous-access directive to an active `YES`:

```text
anonymous_enable=YES
```

Leave `local_enable=YES` unchanged for this lab. The resulting combination accepts both local Linux users with their normal passwords and the anonymous user without a password. In production, that combination is a critical exposure whenever port 21 is reachable: anyone who reaches the FTP port can list and download whatever the FTP account can read.

### 3.4 Creating the anonymous directory and marker file

The anonymous FTP user maps to the local `ftp` account. The lab creates a conventional public subdirectory and seeds it with a small retrieval marker:

```bash
root@ubuntu-lab:~# mkdir -p /var/ftp/pub
root@ubuntu-lab:~# chown nobody:nogroup /var/ftp/pub
root@ubuntu-lab:~# cd /var/ftp/pub
root@ubuntu-lab:/var/ftp/pub# echo "Lab FTP retrieval marker" > note.txt
root@ubuntu-lab:/var/ftp/pub# ls -l
total 4
-rw-r--r-- 1 root root 25 Feb 10 12:01 note.txt
root@ubuntu-lab:/var/ftp/pub# cat note.txt
Lab FTP retrieval marker
```

The `pub` directory name is conventional for an anonymous FTP drop folder, dating back to early Internet file archives, and it is exactly what an attacker expects to find. Ownership by `nobody:nogroup` follows the conventional unprivileged identity for public data. The marker file gives the enumeration workflow a tangible artefact to retrieve; its contents are deliberately boring because the point is retrieval, not the data itself.

### 3.5 Tuning anonymous mode

Append the anonymous-mode block to `/etc/vsftpd.conf`:

```text
# Point anonymous users at the directory created earlier.
anon_root=/var/ftp/

# Do not prompt for a password on the command line.
no_anon_password=YES

# Show the user and group as ftp:ftp, regardless of the owner.
hide_ids=YES

# Constrain the passive-mode port range for predictable firewalling.
pasv_min_port=40000
pasv_max_port=50000
```

Each directive has a distinct effect. `anon_root` roots anonymous sessions in `/var/ftp/`, so the share begins at the directory prepared above. `no_anon_password` removes the password prompt for anonymous logins. `hide_ids` masks the underlying UID and GID in directory listings, which prevents ownership enumeration but also hides useful forensic detail. The passive-port pair limits data-connection ports to `40000–50000`, making firewall rules predictable.

From a defensive perspective, `hide_ids` deserves special attention. It exists so that administrators can offer an anonymous share without disclosing which Linux account owns each file. That is a privacy improvement, not a security boundary: it does not prevent listing, downloading, or inferring the presence of sensitive files.

### 3.6 Restarting and verifying the service

Restart the daemon to apply the configuration:

```bash
root@ubuntu-lab:/var/ftp/pub# service vsftpd restart
root@ubuntu-lab:/var/ftp/pub# cd ~
root@ubuntu-lab:~# ss -tlnp | grep ':21'
LISTEN 0      32                 *:21              *:*    users:(("vsftpd",pid=1842,fd=3))
root@ubuntu-lab:~# systemctl status vsftpd --no-pager
● vsftpd.service - vsftpd FTP server
     Loaded: loaded (/lib/systemd/system/vsftpd.service; enabled; vendor preset: enabled)
     Active: active (running) since Sat 2026-10-10 12:04:11 UTC; 18s ago
   Main PID: 1842 (vsftpd)
      Tasks: 1 (limit: 2261)
     Memory: 1.1M
        CPU: 9ms
     CGroup: /system.slice/vsftpd.service
             └─1842 /usr/sbin/vsftpd /etc/vsftpd.conf
```

The `service` command prints no output on success; on a systemd-managed distribution, an empty result after `restart` is the expected success signal. The `ss` output proves the daemon is bound to port 21, and `systemctl status` confirms that the running process is using the edited configuration file. If port 21 is absent, recheck the listener directives before assuming that anonymous access is broken: a daemon that is not listening cannot be enumerated at all.

---

## 4. Anonymous FTP — enumeration and retrieval

With the lab server live, the attacker workflow has two canonical steps: confirm anonymous access with Nmap, then interact with the share through an FTP session and retrieve the marker.

### 4.1 Nmap service detection

Nmap’s `-A` option enables version detection, script scanning, OS detection, and traceroute in one pass. Against a single lab port, it is a compact way to confirm both the daemon and the misconfiguration:

```bash
root@kali:~# nmap -A -p 21 192.168.1.9
Starting Nmap 7.94 ( https://nmap.org ) at 2026-10-10 12:06 UTC
Nmap scan report for 192.168.1.9
Host is up (0.00052s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| drwxr-xr-x    2 ftp      ftp          4096 Feb 10 12:01 pub
|_End of status.
MAC Address: 00:0C:29:1B:2C:3D (VMware)
Service Info: OS: Unix

TRACEROUTE
HOP RTT     ADDRESS
1   0.52 ms 192.168.1.9

Nmap done: 1 IP address (1 host up) scanned in 8.42 seconds
```

Three facts drive the next step. First, the service is `vsftpd 3.0.5` on the expected port. Second, the `ftp-anon` script reports `Anonymous FTP login allowed (FTP code 230)`, which is direct confirmation that no password is required. Third, the script lists the `pub` directory inside the anonymous root, so the operator already knows where to look before opening an interactive session. On a local lab network, the one-hop traceroute and VMware MAC prefix also corroborate that this is a nearby virtual target rather than a routed production host.

The `-A` option is noisy and slow compared with `-sV`, so it is best used after a narrower scan has found something worth investigating. Here its value is completeness: one command establishes the version, the anonymous-access finding, and the initial directory inventory.

### 4.2 FTP session and file retrieval

Connect as `anonymous`, inspect the share, enter `pub`, and download `note.txt`:

```bash
root@kali:~# ftp 192.168.1.9
Connected to 192.168.1.9.
220 (vsFTPd 3.0.5)
Name (192.168.1.9:root): anonymous
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||40217|).
150 Here comes the directory listing.
drwxr-xr-x    2 ftp      ftp          4096 Feb 10 12:01 pub
226 Directory send OK.
ftp> cd pub
250 Directory successfully changed.
ftp> ls
229 Entering Extended Passive Mode (|||40245|).
150 Here comes the directory listing.
-rw-r--r--    1 ftp      ftp            25 Feb 10 12:01 note.txt
226 Directory send OK.
ftp> get note.txt
local: note.txt remote: note.txt
229 Entering Extended Passive Mode (|||40271|).
150 Opening BINARY mode data connection for note.txt (25 bytes).
100% |********************************|    25        0.01 KiB/s    00:00 ETA
226 Transfer complete.
25 bytes received in 0.00 secs (12.4 kB/s)
ftp> bye
221 Goodbye.
root@kali:~# cat note.txt
Lab FTP retrieval marker
```

The session authenticates as `anonymous`, receives code `230` for successful login, lists the anonymous root, navigates into `pub`, and downloads the marker. The `229` lines show passive-mode data connections being opened inside the configured `40000–50000` range. The transfer is tiny here, but the same workflow retrieves multi-gigabyte archives with no additional access required.

Two details are worth retaining for reporting. First, FTP transmits the username, commands, file listing, and file contents without encryption unless TLS has been configured. Second, `hide_ids=YES` is visible in the listing: everything appears to be owned by `ftp:ftp`, even though the marker file was created by root. Ownership masking therefore did not prevent disclosure; it only made the listing less informative.

---

## 5. Guest SMB — configuring Samba

Samba implements the SMB/CIFS protocol on Linux and bridges Linux servers to Windows-compatible clients. Like `vsftpd`, it ships with conservative defaults that this lab deliberately weakens. The resulting `[shares]` export is browsable, writable, and accessible to unauthenticated SMB clients.

### 5.1 Installing Samba

Install the server package:

```bash
root@ubuntu-lab:~# apt install samba
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following additional packages will be installed:
  python3-samba samba-common samba-common-bin samba-dsdb-modules tdb-tools
Suggested packages:
  bind9 bind9utils ctdb ldb-tools ntp smbldap-tools winbind heimdal-clients
The following NEW packages will be installed:
  python3-samba samba samba-common samba-common-bin samba-dsdb-modules tdb-tools
0 upgraded, 6 newly installed, 0 to remove and 0 not upgraded.
Need to get 2,981 kB of archives.
After this operation, 27.4 MB of additional disk space will be used.
Do you want to continue? [Y/n] y
```

The installer pulls Samba with its common libraries and supporting tools. The daemon is ready to configure after installation, but it does not yet expose the lab share because the default configuration only declares global behaviour and printer-related sections.

### 5.2 Inspecting the Samba configuration directory

Move to the configuration directory and inspect its contents:

```bash
root@ubuntu-lab:~# cd /etc/samba/
root@ubuntu-lab:/etc/samba# ls -al
total 32
drwxr-xr-x  3 root root  4096 Feb 10 12:10 .
drwxr-xr-x 96 root root  4096 Feb 10 11:58 ..
-rw-r--r--  1 root root    17 Feb 10 11:59 gdbcommands
-rw-r--r--  1 root root  9831 Feb 10 11:59 smb.conf
drwxr-xr-x  2 root root  4096 Feb 10 11:59 tls
root@ubuntu-lab:/etc/samba# nano smb.conf
```

The canonical file is `smb.conf`; `gdbcommands` supports debugging and `tls` holds certificate material. Editing `smb.conf` directly is clear for a lab, but production changes should always be validated with `testparm` before the new configuration is applied.

### 5.3 Defining the guest-accessible `[shares]` section

Append the lab share definition:

```text
[shares]
path = /var/www/
available = yes
read only = no
browsable = yes
public = yes
writable = yes
guest ok = yes
```

Every line contributes to the exposure. `guest ok = yes` maps unauthenticated clients to guest access. `public = yes` is legacy syntax for the same intent. `browsable = yes` makes the share visible during enumeration. `read only = no` and `writable = yes` express the same writable setting twice: the two directives are inverse spellings of one option, so either one alone would permit writes. Publishing `/var/www/` compounds the risk because that path commonly backs onto web content; a writable SMB share over web-served files can become a write-anywhere primitive behind an HTTP endpoint.

Validate the resulting configuration before restarting:

```bash
root@ubuntu-lab:/etc/samba# testparm -s
Loaded services file OK.
Weak crypto is allowed by default.

Server role: ROLE_STANDALONE

# Global parameters
[global]
	map to guest = Bad User
	obey pam restrictions = Yes
	pam password change = Yes
	unix password sync = Yes

[shares]
	path = /var/www/
	read only = No
```

`testparm` parses the effective configuration and reports syntax or semantic problems. Its normalised output is also useful evidence: it shows the share path and writable state without requiring the reader to interpret every redundant directive in the source file.

Restart the SMB daemon and confirm the expected listeners:

```bash
root@ubuntu-lab:/etc/samba# systemctl restart smbd
root@ubuntu-lab:/etc/samba# ss -tlnp | grep -E ':(139|445)'
LISTEN 0      50                 *:139             *:*    users:(("smbd",pid=2261,fd=34))
LISTEN 0      50                 *:445             *:*    users:(("smbd",pid=2261,fd=33))
root@ubuntu-lab:/etc/samba# cd ~
```

Ports 139 and 445 are the traditional SMB listeners. If only one is present, check whether another service owns the missing socket and whether the host firewall is interfering. If neither is present, the Samba configuration did not load as expected.

### 5.4 Seeding and inspecting the shared files

Create a marker in the shared web directory and confirm its contents locally:

```bash
root@ubuntu-lab:~# echo "Lab SMB retrieval marker" > /var/www/file.txt
root@ubuntu-lab:~# cd /var/www/
root@ubuntu-lab:/var/www# ls -l
total 12
-rw-r--r-- 1 root root   25 Feb 10 12:14 file.txt
drwxr-xr-x 2 root root 4096 Feb 10 11:47 html
root@ubuntu-lab:/var/www# cat file.txt
Lab SMB retrieval marker
root@ubuntu-lab:/var/www# cd ~
```

The `html` subdirectory is the normal web-content directory, while `file.txt` is the lab’s verifiable retrieval target. Seeding a known file matters because successful anonymous access should be proved by reading back exact bytes, not merely by listing a filename.

---

## 6. Guest SMB — enumeration and retrieval

Two complementary tools handle Samba enumeration. NetExec discovers shares quickly across one host or many hosts, while `smbclient` provides interactive session work against a single export. Use NetExec for discovery and `smbclient` for retrieval.

### 6.1 NetExec share discovery

Authenticate as `guest` with an empty password and enumerate shares:

```bash
root@kali:~# nxc smb 192.168.1.9 --shares -u 'guest' -p ''
[*] SMB         192.168.1.9    445    UBUNTU-LAB     [*] Unix - Samba 4.17.7-Ubuntu
[+] SMB         192.168.1.9    445    UBUNTU-LAB     UBUNTU-LAB\guest: (Guest)
[*] SMB         192.168.1.9    445    UBUNTU-LAB     Enumerated shares
[*] SMB         192.168.1.9    445    UBUNTU-LAB     Share           Permissions     Remark
[*] SMB         192.168.1.9    445    UBUNTU-LAB     -----           -----------     ------
[*] SMB         192.168.1.9    445    UBUNTU-LAB     print$                          Printer Drivers
[*] SMB         192.168.1.9    445    UBUNTU-LAB     shares          READ            Lab file share
[*] SMB         192.168.1.9    445    UBUNTU-LAB     IPC$                            IPC Service
```

The output confirms guest authentication with the `(Guest)` marker and lists three shares. `print$` is the default printer-driver share, `IPC$` is the interprocess-communication share, and `shares` is the lab export. The `READ` permission against `shares` is the authorisation to continue: an unauthenticated client can browse and download from that export.

NetExec’s marker convention is consistent across protocols. Informational lines begin with `[*]`, successes begin with `[+]`, and failures begin with `[-]`. That convention makes one-host output readable and multi-host output scannable.

### 6.2 `smbclient` share listing

The same inventory can be obtained directly with `smbclient`. The `-N` option suppresses the password prompt for guest access, while `-L` lists shares:

```bash
root@kali:~# smbclient -N -L //192.168.1.9
Anonymous login successful

	Sharename       Type      Comment
	---------       ----      -------
	print$          Disk      Printer Drivers
	shares          Disk      Lab file share
	IPC$            IPC       IPC Service (Samba 4.17.7-Ubuntu)
Reconnecting with SMB1 for workgroup listing.
smbXcli_negprot_smb1_done: No compatible protocol selected by client, err=NT_STATUS_INVALID_PARAMETER
Unable to connect with SMB1 -- no workgroup available
```

The share table matches the NetExec result. The trailing SMB1 messages are also informative: the client attempts an obsolete dialect only for workgroup listing, the negotiation fails, and the server does not fall back to SMB1. That failure is a welcome default. Modern clients should negotiate SMB2 or SMB3, and obsolete-dialect fallback should remain disabled.

### 6.3 `smbclient` interactive retrieval

Open the `shares` export without a password, list it, and retrieve the marker:

```bash
root@kali:~# smbclient //192.168.1.9/shares -N
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Sat Oct 10 12:14:22 2026
  ..                                  D        0  Sat Oct 10 11:47:09 2026
  file.txt                            A       25  Sat Oct 10 12:14:22 2026
  html                                D        0  Sat Oct 10 11:47:09 2026

		26632192 blocks of size 1024. 18874368 blocks available
smb: \> get file.txt
getting file \file.txt of size 25 as file.txt (1.2 KiloBytes/sec) (average 1.2 KiloBytes/sec)
smb: \> exit
root@kali:~# cat file.txt
Lab SMB retrieval marker
```

The `A` attribute marks a normal file and `D` marks a directory. The local `cat` proves that the retrieved bytes match the seeded marker. Because the export is writable, the same session could also upload or overwrite files; in an authorised test, that capability should be demonstrated carefully and minimally, preferably with a uniquely named marker that is removed immediately afterward.

---

## 7. Insecure NFS — configuring exports

NFS, the Network File System, is the canonical Unix-to-Unix file-sharing protocol. Unlike FTP and SMB, NFS does not authenticate users at the protocol level by default; it trusts the UID declared by the client. The export options below intentionally weaponise that trust model for the lab.

### 7.1 Installing the NFS kernel server

Install the server package:

```bash
root@ubuntu-lab:~# apt install nfs-kernel-server -y
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following additional packages will be installed:
  keyutils libnfsidmap1 nfs-common rpcbind
Suggested packages:
  watchdog
The following NEW packages will be installed:
  keyutils libnfsidmap1 nfs-common nfs-kernel-server rpcbind
0 upgraded, 5 newly installed, 0 to remove and 0 not upgraded.
Need to get 1,048 kB of archives.
After this operation, 3,862 kB of additional disk space will be used.
```

The supporting packages matter. `nfs-common` provides client-side utilities, `libnfsidmap1` handles identity mapping, `keyutils` supports key management, and `rpcbind` provides the portmapper service on TCP/UDP port 111. Clients query the portmapper to locate the NFS and mount daemons, so an open port 111 is a strong indicator that NFS infrastructure is reachable on the host.

### 7.2 Creating the export directory

Create a public export directory, make it world-writable, and seed a marker file:

```bash
root@ubuntu-lab:~# mkdir -p /srv/nfs/public
root@ubuntu-lab:~# chmod 777 /srv/nfs/public
root@ubuntu-lab:~# cd /srv/nfs/public
root@ubuntu-lab:/srv/nfs/public# echo "Lab NFS retrieval marker" > data.txt
root@ubuntu-lab:/srv/nfs/public# ls -l
total 4
-rw-r--r-- 1 root root 25 Feb 10 12:20 data.txt
root@ubuntu-lab:/srv/nfs/public# cat data.txt
Lab NFS retrieval marker
root@ubuntu-lab:/srv/nfs/public# cd ~
```

World-writable `777` permissions are dangerous on their own, and they become especially dangerous when combined with the `no_root_squash` export option below. That combination allows a remote root user to write files that retain root ownership on the server, which is a textbook persistence and privilege-escalation surface.

### 7.3 Defining the insecure export

NFS access control lives in `/etc/exports`. Open it:

```bash
root@ubuntu-lab:~# nano /etc/exports
```

Add the following lab-only export line:

```text
/srv/nfs/public *(rw,sync,no_subtree_check,no_root_squash,insecure)
```

The options combine four risky choices. The `*` wildcard exports the directory to every client address. `rw` grants read and write access. `no_root_squash` disables the default protection that maps remote root requests to the unprivileged `nobody` account; with this flag, a client acting as root can act as root on the exported files. `insecure` permits clients to connect from unprivileged source ports above 1024, removing the historical requirement that NFS clients use a privileged port.

Use commas between options. A missing comma or an accidental space can change parsing or prevent the export from loading, and the resulting failure can look like a network problem when it is actually a one-character configuration error.

### 7.4 Applying and verifying the export

Reload the export table, inspect the effective export, restart the NFS server, and verify the listeners:

```bash
root@ubuntu-lab:~# exportfs -a
root@ubuntu-lab:~# exportfs -v
/srv/nfs/public
		*(rw,wdelay,no_root_squash,no_subtree_check,sec=sys,rw,insecure,no_root_squash,no_all_squash)
root@ubuntu-lab:~# systemctl restart nfs-kernel-server
root@ubuntu-lab:~# ss -tlnp | grep -E ':(111|2049)'
LISTEN 0      4096               *:111             *:*    users:(("rpcbind",pid=2418,fd=4))
LISTEN 0      64                 *:2049            *:*    users:(("nfsd",pid=2431,fd=9))
```

`exportfs -a` exports all listed directories without requiring a daemon restart, while `exportfs -v` displays the effective options. Empty output from `exportfs -a` is normal. The `systemctl restart` cycles the kernel-server user-space components and picks up broader service changes. The final check confirms both the RPC portmapper on port 111 and NFS on port 2049; if either is missing, enumeration will fail before authentication or UID behaviour even matters.

---

## 8. Insecure NFS — enumeration, download and mounting

NFS enumeration has three useful levels: discover exports, inspect or download individual files remotely, and mount the export for full filesystem access. NetExec covers the first two without mounting anything. A traditional mount provides the third and exposes the full impact of `no_root_squash`.

### 8.1 Classic export discovery with `showmount` and `rpcinfo`

Before using NetExec, confirm the classic discovery path from the attacker. `showmount -e` asks the target which directories it exports, and `rpcinfo -p` lists registered RPC services:

```bash
root@kali:~# showmount -e 192.168.1.9
Export list for 192.168.1.9:
/srv/nfs/public *
root@kali:~# rpcinfo -p 192.168.1.9 | head -12
   program vers proto   port  service
    100000    4   tcp    111  portmapper
    100000    3   tcp    111  portmapper
    100000    2   tcp    111  portmapper
    100000    4   udp    111  portmapper
    100005    3   tcp  45231  mountd
    100005    3   udp  42768  mountd
    100003    4   tcp   2049  nfs
    100003    3   tcp   2049  nfs
    100003    4   udp   2049  nfs
    100003    3   udp   2049  nfs
```

The export list names the vulnerable path and its wildcard client specification. The RPC table confirms the portmapper, mount daemon, and NFS services. Dynamic `mountd` ports are normal unless the administrator pins them; their presence explains why NFS firewalling is more involved than opening port 2049 alone.

### 8.2 NetExec export enumeration and listing

NetExec’s NFS module provides the same discovery workflow in the interface already used for SMB:

```bash
root@kali:~# nxc nfs 192.168.1.9 --enum-shares
[*] NFS         192.168.1.9    2049   UBUNTU-LAB     [*] Enumerating NFS exports
[+] NFS         192.168.1.9    2049   UBUNTU-LAB     /srv/nfs/public (rw, no_root_squash, root escape: True)
root@kali:~# nxc nfs 192.168.1.9 --share '/srv/nfs/public' --ls '/'
[*] NFS         192.168.1.9    2049   UBUNTU-LAB     [*] Listing / on /srv/nfs/public
[*] NFS         192.168.1.9    2049   UBUNTU-LAB     drwxrwxrwx root root 4096 .
[*] NFS         192.168.1.9    2049   UBUNTU-LAB     -rw-r--r-- root root   25 data.txt
```

The first command discovers the export and flags its dangerous combination: writable access plus `no_root_squash`, summarised as `root escape: True`. The second command lists the export root without performing a local mount and reveals `data.txt`. The listing requires no credentials because the export itself imposes no authentication check.

### 8.3 One-shot file download with NetExec

NetExec can download one file directly from the export. The command takes the remote path relative to the share root and a local destination:

```bash
root@kali:~# nxc nfs 192.168.1.9 --share /srv/nfs/public/ --get-file data.txt data.txt
[*] NFS         192.168.1.9    2049   UBUNTU-LAB     [*] Downloading data.txt from /srv/nfs/public/
[+] NFS         192.168.1.9    2049   UBUNTU-LAB     File successfully downloaded to data.txt
root@kali:~# cat data.txt
Lab NFS retrieval marker
```

This is the fastest path from “the file exists” to “the file is on local disk.” No mount point is created, no persistent filesystem state is changed on the attacker, and no shell access to the target is required. For evidence handling, it is also the cleanest method: one remote file becomes one local file with no broader filesystem side effects.

### 8.4 Mounting the export with `mount -t nfs`

A mount provides full filesystem semantics: `ls`, `find`, `grep`, `cp`, permission inspection, and execution against the remote export as if it were a local directory. Create a mount point and mount the export:

```bash
root@kali:~# mkdir -p /tmp/nfs
root@kali:~# mount -t nfs 192.168.1.9:/srv/nfs/public /tmp/nfs
root@kali:~# ls -la /tmp/nfs
total 12
drwxrwxrwx  2 root root 4096 Feb 10 12:20 .
drwxrwxrwt 14 root root 4096 Feb 10 12:24 ..
-rw-r--r--  1 root root   25 Feb 10 12:20 data.txt
root@kali:~# cat /tmp/nfs/data.txt
Lab NFS retrieval marker
```

The mounted directory preserves the server-side ownership and permissions, including world-writable access and root ownership. Because of `no_root_squash`, operations performed as root through this mount retain root identity on the server-side files. That behaviour turns an ordinary file share into a privilege-escalation primitive: files written through the mount can carry ownership and permission bits that would normally require local root access to create.

Unmount the export when finished:

```bash
root@kali:~# umount /tmp/nfs
root@kali:~# ls /tmp/nfs
root@kali:~#
```

Cleaning up matters in a shared lab. A forgotten mount can confuse later tests, retain stale file handles after the export changes, and leave misleading directory contents in `/tmp`. Verify that the mount point is empty after unmounting.

---

## 9. Hardening summary

The lab has now been attacked end to end, and every protocol exposure came from a small number of configuration choices. This section gathers the corresponding controls in one place.

### 9.1 FTP hardening

**Disable anonymous access.** Set `anonymous_enable=NO` in `/etc/vsftpd.conf` unless a documented business case justifies anonymous retrieval. Even then, prefer SFTP over SSH for file transfers across untrusted networks, because SFTP provides authentication and encryption by default.

**Enforce TLS where FTP must remain.** Set `ssl_enable=YES` with `force_local_logins_ssl=YES` and `force_local_data_ssl=YES` so credentials and transferred content do not traverse the network in cleartext. Certificates, client compatibility, and passive-port firewall rules must be managed as part of the same deployment.

**Restrict network exposure.** Bind `vsftpd` to internal interfaces where possible, limit port 21 to known management subnets with a host firewall, and open only the configured passive-port range. A share that cannot be reached from an untrusted segment cannot be enumerated from that segment.

### 9.2 Samba hardening

**Disable guest access.** Remove `guest ok = yes` and `public = yes` from every share, set `map to guest = Never` in `[global]`, and require valid Linux accounts for every connection. Anonymous SMB access should be exceptional, documented, and regularly reviewed.

**Disable SMB1.** Set `min protocol = SMB2` in `[global]` to prevent fallback to the obsolete SMB1 dialect. Modern clients negotiate SMB2 or SMB3 by default, so this control rarely breaks legitimate access while removing a historically fragile protocol path.

**Restrict share paths.** Never publish `/var/www`, `/etc`, home directories, backup roots, or any path that backs onto another service. Use isolated share roots such as `/srv/samba/<sharename>` so that an access-control mistake has a smaller blast radius. Pair that layout with filesystem permissions that deny writes unless writes are explicitly required.

### 9.3 NFS hardening

**Enable `root_squash`.** Remove `no_root_squash` from every export line. The default `root_squash` behaviour maps remote root requests to the unprivileged `nobody` account and removes the file-ownership persistence vector demonstrated in section 8.

**Restrict export wildcards.** Replace `*` with explicit client addresses or CIDR ranges so only known hosts can mount the export. A backup server, application host, or management workstation should be named individually; an entire untrusted subnet should not be trusted by default.

**Authenticate NFS where the network is untrusted.** Use NFSv4 with Kerberos security such as `sec=krb5p` where authentication and encryption are required. Without Kerberos, NFS’s default UID-trust model is unsuitable for hostile or zero-trust networks.

**Firewall the RPC and NFS ports.** Block TCP/UDP port 111 for `rpcbind`, TCP/UDP port 2049 for NFS, and the mount-daemon port for every untrusted source. Pin dynamic RPC services to stable ports where firewall policy requires it, and verify exposure from an untrusted network segment rather than only from localhost.

### 9.4 Detection and auditing

**Audit configuration files directly.** Every major misconfiguration in this walkthrough is visible with one search per file. Useful starting checks include `anonymous_enable`, `guest ok`, `public`, `map to guest`, `min protocol`, `no_root_squash`, export wildcards, and world-writable export directories.

**Scan from the attacker’s perspective.** Periodically probe for ports 21, 111, 139, 445, and 2049 from untrusted segments. An open port is only the beginning; follow it with the same anonymous and guest checks used in sections 4, 6, and 8.

**Review share inventories and logs.** Retain FTP, Samba, and NFS logs, alert on anonymous or guest authentication, and review export and share inventories after every configuration change. A quarterly audit of `/etc/vsftpd.conf`, `/etc/samba/smb.conf`, and `/etc/exports`, paired with network-level scans, closes every primitive demonstrated in this guide.

### 9.5 Final analysis

FTP, SMB, and NFS collectively underpin much enterprise file sharing. Each protocol ships with conservative defaults, and each is routinely weakened for convenience. This lab reproduced the resulting misconfigurations: anonymous FTP with masked ownership, guest-accessible Samba shares over sensitive paths, and NFS exports with `no_root_squash` and wildcard clients. The matching enumeration workflows collapse discovery and exploitation into a small number of commands.

Defenders win by treating file-share configuration as security-critical infrastructure rather than background plumbing. Every fix is short, every exposure is detectable, and the same lab supports both sides of the engagement: the operator practising the attack chain and the defender validating that controls actually catch it.

---

## 10. Quick reference cheat sheet

| Phase | Command | What it does |
| --- | --- | --- |
| Setup | `apt update` | refreshes the package catalogue before installation |
| Setup | `apt install vsftpd` | installs the FTP daemon |
| Setup | `apt install samba` | installs Samba and its supporting libraries |
| Setup | `apt install nfs-kernel-server -y` | installs the NFS server, client libraries, and `rpcbind` |
| Setup | `ss -tlnp \| grep ':21'` | confirms that FTP is listening on port 21 |
| Setup | `ss -tlnp \| grep -E ':(139\|445)'` | confirms that SMB is listening on ports 139 and 445 |
| Setup | `ss -tlnp \| grep -E ':(111\|2049)'` | confirms that RPC and NFS listeners are present |
| FTP config | `nano /etc/vsftpd.conf` | edits the FTP server configuration |
| FTP config | `anonymous_enable=YES` | enables passwordless anonymous FTP logins |
| FTP config | `anon_root=/var/ftp/` | roots anonymous sessions in `/var/ftp/` |
| FTP config | `no_anon_password=YES` | removes the anonymous password prompt |
| FTP config | `hide_ids=YES` | masks UID/GID ownership in FTP listings |
| FTP config | `pasv_min_port=40000` and `pasv_max_port=50000` | constrains passive-mode data ports |
| FTP config | `service vsftpd restart` | applies FTP configuration changes |
| FTP files | `mkdir -p /var/ftp/pub` | creates the conventional anonymous subdirectory |
| FTP files | `chown nobody:nogroup /var/ftp/pub` | assigns the conventional unprivileged owner |
| FTP files | `echo "Lab FTP retrieval marker" > /var/ftp/pub/note.txt` | seeds a verifiable FTP marker |
| FTP recon | `nmap -A -p 21 <target>` | confirms FTP version and anonymous access |
| FTP access | `ftp <target>` then `anonymous` | opens a passwordless FTP session |
| FTP access | `ls`, `cd pub`, `ls`, `get note.txt`, `bye` | browses the anonymous share and retrieves a file |
| SMB config | `nano /etc/samba/smb.conf` | edits the Samba configuration |
| SMB config | `testparm -s` | validates the effective Samba configuration |
| SMB config | `guest ok = yes` | maps unauthenticated clients to guest access |
| SMB config | `public = yes` | legacy synonym for guest-accessible intent |
| SMB config | `browsable = yes` | makes the share visible during enumeration |
| SMB config | `read only = no` and `writable = yes` | makes the share writable |
| SMB config | `systemctl restart smbd` | applies Samba configuration changes |
| SMB files | `echo "Lab SMB retrieval marker" > /var/www/file.txt` | seeds a verifiable SMB marker |
| SMB recon | `nxc smb <target> --shares -u 'guest' -p ''` | enumerates shares as guest |
| SMB access | `smbclient -N -L //<target>` | lists shares without a password |
| SMB access | `smbclient //<target>/shares -N` | opens the guest-accessible share |
| SMB access | `ls`, `get file.txt`, `exit` | lists and retrieves a file over SMB |
| NFS config | `nano /etc/exports` | edits the NFS export table |
| NFS config | `/srv/nfs/public *(rw,sync,no_subtree_check,no_root_squash,insecure)` | publishes an insecure lab-only export |
| NFS config | `exportfs -a` | reloads every export-table entry |
| NFS config | `exportfs -v` | displays effective export options |
| NFS config | `systemctl restart nfs-kernel-server` | restarts NFS user-space services |
| NFS files | `mkdir -p /srv/nfs/public && chmod 777 /srv/nfs/public` | creates a world-writable export directory |
| NFS files | `echo "Lab NFS retrieval marker" > /srv/nfs/public/data.txt` | seeds a verifiable NFS marker |
| NFS recon | `showmount -e <target>` | lists directories exported to the network |
| NFS recon | `rpcinfo -p <target>` | lists registered RPC services |
| NFS recon | `nxc nfs <target> --enum-shares` | discovers NFS exports with NetExec |
| NFS recon | `nxc nfs <target> --share '/srv/nfs/public' --ls '/'` | lists an export root without mounting |
| NFS access | `nxc nfs <target> --share /srv/nfs/public/ --get-file data.txt data.txt` | downloads one NFS file without mounting |
| NFS access | `mount -t nfs <target>:/srv/nfs/public /tmp/nfs` | mounts the export locally |
| NFS access | `ls -la /tmp/nfs` and `cat /tmp/nfs/data.txt` | inspects and reads the mounted export |
| NFS access | `umount /tmp/nfs` | removes the local NFS mount |
| Defence | `grep -nE 'anonymous_enable\|guest ok\|public =\|map to guest\|min protocol' /etc/vsftpd.conf /etc/samba/smb.conf` | finds risky FTP/SMB directives |
| Defence | `grep -nE 'no_root_squash\|\*\(\|insecure' /etc/exports` | finds risky NFS export options |
| Defence | `nmap -sV -p 21,111,139,445,2049 <target>` | checks file-sharing exposure from the network |

---

## 11. Practice exercises

The following sequence is designed to be run in an isolated lab with two virtual machines and no route to the internet. Each exercise builds on the previous one, and several are intended to show how one defensive change alters the attacker’s output.

1. **Build the range.** Install `vsftpd`, Samba, and `nfs-kernel-server` on the Ubuntu target. Use `ss -tlnp` to confirm listeners on ports 21, 139, 445, 111, and 2049. Explain which process owns each listener and why the RPC portmapper must be included in the check.
2. **Prove anonymous FTP.** Run `nmap -A -p 21` against the target and identify the script output that proves anonymous access. Then open an FTP session as `anonymous`, retrieve `note.txt`, and compare its bytes with the marker created on the server.
3. **Disable anonymous FTP.** Set `anonymous_enable=NO`, restart `vsftpd`, and repeat both the Nmap check and the interactive login. Record the exact changed output and explain which server-side setting produced it.
4. **Map guest SMB.** Enumerate shares as `guest` with NetExec and then with `smbclient -N -L`. Compare the two inventories and identify every share that should not normally be available to an unauthenticated client.
5. **Retrieve over SMB.** Connect to the `shares` export with `smbclient`, list its contents, retrieve `file.txt`, and verify its contents locally. Describe what the SMB file attributes reveal about files versus directories.
6. **Remove guest access.** Remove `guest ok` and `public` from the lab share, set `map to guest = Never`, validate with `testparm`, restart Samba, and rerun both enumeration commands. Document which behaviour changes first from the attacker’s perspective.
7. **Discover NFS classically.** Use `showmount -e` and `rpcinfo -p` to identify the exported path and the RPC services supporting it. Explain why a firewall rule for port 2049 alone may be insufficient.
8. **Enumerate NFS with NetExec.** Run `--enum-shares` and then list the export root. Identify the dangerous option combination in the output and explain what `root escape: True` means for file ownership.
9. **Download without mounting.** Retrieve `data.txt` with NetExec `--get-file`, then mount the export and read the same file through `/tmp/nfs`. Compare the two retrieval methods for speed, forensic footprint, and available filesystem operations.
10. **Restore safe NFS defaults.** Replace the wildcard client with an explicit lab address, remove `no_root_squash` and `insecure`, reload with `exportfs -a`, and rerun NetExec enumeration and the mount test. Record which operations still work and which now fail.
11. **Audit all three services.** Write one `grep` check for each configuration file that would detect the corresponding lab misconfiguration. Then run a single Nmap command covering all relevant file-sharing ports and explain what the scan can and cannot prove about authorisation.
12. **Write the report.** Summarise the whole chain as a five-hundred-word finding: the initial weaknesses, the protocol-by-protocol exploitation path, the business impact, and three controls that would each have independently prevented unauthenticated retrieval.

Working through those twelve exercises end to end will have taken you through every command in this guide at least once and, more usefully, through the reasoning behind each of them — which is the part that transfers to a different target, a different service and a different engagement.
