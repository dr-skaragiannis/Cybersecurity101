# Bandit — OverTheWire Wargame Walkthrough

**Bandit** is a **34-level** educational wargame provided by the **OverTheWire** community and hosted on a real Linux server. At each level you connect via **SSH** as a different user (`bandit0`, `bandit1`, …) and must locate the **password of the next user**. Each level introduces a new Linux command or concept, which makes Bandit the natural continuation of the lab [01. Introduction to the Linux CLI](../01.%20Introduction%20to%20the%20Linux%20CLI/README.md): there you learn the commands; here you apply them on a remote system to solve specific problems.

This guide is an expanded English adaptation of the write-up by Alexandre Cheron.

> - **Platform:** [OverTheWire — Bandit](https://overthewire.org/wargames/bandit/)
> - **Original write-up (English):** [[OTW] Write-up for the Bandit Wargame — A. Cheron](https://axcheron.github.io/writeups/otw/bandit/)
> - **Lab manual (PDF):** [Lab 02 - Bandit.pdf (Greek)](../../02.%20Bandit/Lab%2002%20-%20Bandit.pdf)

#### **Caution! The passwords shown in the command outputs date from 2018. OverTheWire rotates them periodically, so they are no longer valid. The educational value lies in the solution method, not in the string itself. Attempt each level on your own first, and consult the solution only if you cannot make further progress.**

---

## Lab Objectives

Upon completing Bandit, you will be able to:

1. Connect to remote systems via `ssh` or PuTTY and transfer files via `scp`, using either a password or a private key.
2. Locate files by name, size, owner, and permissions (`find`, `file`, `ls -la`).
3. Filter and transform text using pipelines (`grep`, `sort`, `uniq`, `tr`, `base64`).
4. Interact with network services via `nc` and `openssl s_client`, and perform port scanning.
5. Understand how **SUID** executables and **cron** jobs can lead to privilege escalation.
6. Leverage **git** history to recover data that has been removed.
7. Escape from restricted shells (`more` → `vi`, `$0`).

---

## Game Structure

| Topic area | Levels and content |
| :--- | :--- |
| **Files and names** | 0–6: `ls`, `cat`, hidden files, special filenames, `file`, `find` |
| **Text processing** | 7–12: `grep`, `sort \| uniq`, `strings`, Base64, ROT13, compression |
| **Networking and keys** | 13–20: SSH keys, `nc`, `openssl`, port scanning, `diff`, SUID |
| **Automation** | 21–24: `cron`, bash scripts, brute force |
| **Escapes and Git** | 25–33: escape via `more` / `vi`, history, branches, tags, pushing changes, `$0` |

```
[ SSH connection ]
       │
       ▼
[ 0–6   Files & names ]           ls, cat, file, find
       │
       ▼
[ 7–12  Text processing ]         grep, sort|uniq, strings, base64, tr, xxd
       │
       ▼
[ 13–20 Networking & keys ]       ssh -i, nc, openssl, diff, SUID
       │
       ▼
[ 21–24 Automation ]              cron, bash scripts, brute force
       │
       ▼
[ 25–33 Escapes & Git ]           more → vi, git log/branch/tag/push, $0
```

---

## Connecting to the Server via SSH

The connection is made via the **SSH (Secure Shell)** protocol, which provides an encrypted channel and shell access to a remote system. Whatever you type in your local terminal is executed on the server.

| Parameter | Value |
| :--- | :--- |
| **Server (host)** | `bandit.labs.overthewire.org` |
| **Port** | `2220` (not the default SSH port 22) |
| **Username** | `bandit0` for the first level, `banditN` for level N |
| **Password** | `bandit0` for the first level; subsequent ones are discovered as you solve the levels |

```
ssh   bandit0  @  bandit.labs.overthewire.org   -p 2220
│     │           │                             │
│     │           │                             └─ port (not the default 22)
│     │           └─ the remote server (host)
│     └─ the user, i.e. the level you are connecting to
└─ the SSH client program
```

### A. From a Linux, macOS, or Windows 10/11 terminal

Linux, macOS, and modern versions of Windows (PowerShell / Command Prompt) include a built-in OpenSSH client:

```bash
$ ssh bandit0@bandit.labs.overthewire.org -p 2220
The authenticity of host '[bandit.labs.overthewire.org]:2220' can't be established.
ED25519 key fingerprint is SHA256:C2ihUBV7ihnV1wUXRb4RrEcLfXC5CXlhmAAM/urerLY.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
bandit0@bandit.labs.overthewire.org's password:     ← type bandit0
bandit0@bandit:~$
```

1. **Host verification:** on the first connection, answer `yes`. The key is stored in `~/.ssh/known_hosts`.
2. **Password:** no characters are displayed while you type. This is expected behavior and serves as a security measure.
3. **Success:** the prompt `bandit0@bandit:~$` indicates that you are on the server.
4. **Disconnecting:** `exit` or `Ctrl + D`.

### B. From Windows using PuTTY

Download PuTTY **only** from the [official website](https://www.chiark.greenend.org.uk/~sgtatham/putty/), since tampered versions containing malware are in circulation. The package also includes **PuTTYgen** and **PSCP**, which are needed in later levels.

1. Under **Session**, set Host Name: `bandit.labs.overthewire.org`, Port: `2220`, Connection type: **SSH**.
2. (Optional) Saved Sessions: `bandit` → **Save**.
3. **Open** → in the *PuTTY Security Alert* dialog, select **Accept**.
4. `login as: bandit0`, followed by the password.

> **Copy/paste in PuTTY:** selecting text with the mouse copies it; right-clicking pastes it. `Ctrl + C` interrupts the running command.

**Connecting with a private key in PuTTY (levels 13 and 16):** PuTTY requires keys in `.ppk` format. In PuTTYgen: *Conversions → Import key* → *Save private key* (e.g. `bandit14.ppk`); then in PuTTY: *Connection → SSH → Auth → Credentials → Private key file*. To transfer files, use `pscp -P 2220 bandit13@bandit.labs.overthewire.org:sshkey.private .` or WinSCP.

### Common connection errors

| Message | Cause and remedy |
| :--- | :--- |
| `Connection timed out` / `Connection refused` | Wrong port (usually the `-p 2220` option was omitted) or blocking by a firewall. |
| `Permission denied, please try again.` | Wrong password or username. Passwords are case-sensitive. |
| `UNPROTECTED PRIVATE KEY FILE!` | The key is readable by other users. Run `chmod 400 <file>`. |
| `REMOTE HOST IDENTIFICATION HAS CHANGED!` | The server's key has changed. If the change is expected: `ssh-keygen -R "[bandit.labs.overthewire.org]:2220"`. |

---

## Part A: Files and Names (Levels 0–6)

### Level 0 → 1: Initial connection
**Commands:** `ssh` · `ls -la` · `cat`

**Goal:** the password is stored in the file `readme` in the home directory.

```bash
bandit0@bandit:~$ ls -la
-rw-r-----  1 bandit1 bandit0   33 Oct 16 14:00 readme
bandit0@bandit:~$ cat readme
boJ9jbbUNNfktd78OOpsqOltutMc3MY1
```

**Explanation — reading a line of `ls -l` output:**

```
-rw-r-----   bandit1   bandit0   33   readme
│└┬┘└┬┘└┬┘   │         │         │
│ │  │  │    │         │         └─ size in bytes
│ │  │  │    │         └─ group
│ │  │  │    └─ owner
│ │  │  └─ others: ---  (no permissions)
│ │  └─ group: r--  (read) ← the user bandit0
│ └─ owner: rw- (read/write)
└─ type: - file, d directory, l link
```

### Level 1 → 2: A file named "-"
**Commands:** `cat` · stdin · relative path

```bash
bandit1@bandit:~$ cat ./-
CV1DtqXWVFXTvM2F0k09SHz0YwRINYA9
```

For many commands, the argument `-` means "read from standard input (stdin)", so `cat -` waits for input from the keyboard. The path `./-` explicitly states that the argument refers to a file in the current directory.

### Level 2 → 3: Filenames containing spaces
**Commands:** quoting · escaping · tab completion

```bash
bandit2@bandit:~$ cat "spaces in this filename"
UmHadQclWmgdLOKQ3YNgjWxGoRMb5luK
```

The shell splits arguments on whitespace, so without quotation marks `cat` would look for four separate files. Equivalently: `cat spaces\ in\ this\ filename`. In practice: type `cat sp` and press `Tab`.

### Level 3 → 4: Hidden files
**Commands:** `cd` · `ls -a` · dotfiles

```bash
bandit3@bandit:~$ cd inhere/
bandit3@bandit:~/inhere$ ls -la
-rw-r----- 1 bandit4 bandit3   33 Dec 28 14:34 .hidden
bandit3@bandit:~/inhere$ cat .hidden
pIwrPrtPN36QITSp3EQaw936yaFoFgAB
```

In Linux, a "hidden file" is merely a convention: any name beginning with a dot is omitted by a plain `ls`. The `-a` (all) option displays such files.

> 🛡️ **Security:** `ls -la` is among the very first steps when analyzing a system. Malicious files are frequently concealed under names such as `.cache` or `...`.

### Level 4 → 5: Locating a text file by type
**Commands:** `file` · wildcards

```bash
bandit4@bandit:~/inhere$ file ./-file0*
./-file00: data
./-file07: ASCII text
...
bandit4@bandit:~/inhere$ cat ./-file07
koReBOKuIDDepwhWk7jZC0RTdopnAYKh
```

The `file` command inspects the first bytes of a file (its "magic bytes") and identifies its type regardless of its name or extension.

### Level 5 → 6: Searching by file attributes
**Commands:** `find` · `-size` · `-readable` · `! -executable`

**Goal:** a file that is readable, 1033 bytes in size, and not executable.

```bash
bandit5@bandit:~$ find ./inhere/ -type f -readable ! -executable -size 1033c
./inhere/maybehere07/.file2
bandit5@bandit:~$ cat ./inhere/maybehere07/.file2
DXjZPULLxYr17uwoI01bNLQbtFemEgo7
```

| Option | Meaning |
| :--- | :--- |
| `-type f` | regular files only (not directories) |
| `-readable` | readable by the current user |
| `! -executable` | the `!` negates the filter: not executable |
| `-size 1033c` | exactly 1033 bytes (`c` = bytes, `k` = KiB, `M` = MiB) |

### Level 6 → 7: Searching the entire file system
**Commands:** `find /` · `-user` · `-group` · `2>/dev/null`

**Goal:** a file owned by user `bandit7` and group `bandit6`, 33 bytes in size.

```bash
bandit6@bandit:~$ find / -type f -size 33c -group bandit6 -user bandit7 2>/dev/null
/var/lib/dpkg/info/bandit7.password
bandit6@bandit:~$ cat /var/lib/dpkg/info/bandit7.password
HKBPTKQnIay4Fw76bEy8PVxKEDQRKTzs
```

When the search starts from the root directory `/`, `find` produces hundreds of `Permission denied` messages. These are written to **stderr** (stream 2), not to **stdout** (stream 1). The redirection `2>/dev/null` discards them. Alternatively, `2>&1 | grep -v "Permission denied"` merges them into stdout and then filters them out.

---

## Part B: Text Processing (Levels 7–12)

### Level 7 → 8: Searching for a string
**Commands:** `grep` · `find -exec`

```bash
bandit7@bandit:~$ grep millionth data.txt
millionth    cvX2JJa4CFALtqS87jk27qwqGhBM9plV
```

The original write-up uses `find / -name "data.txt" -exec grep -H 'millionth' {} \; 2>/dev/null`. The `-exec` option runs a command for every match: `{}` is replaced by the filename, and `\;` terminates the command.

### Level 8 → 9: The unique line
**Commands:** `sort` · `uniq` · pipeline

```bash
bandit8@bandit:~$ sort data.txt | uniq -u
UsvVyFSfZZWbi6wgC7dAFyFuR6jQQUhR
```

```
data.txt ──> sort ──> uniq -c ──> grep "1 " ──> password
             │        │           │
             │        │           └─ keep only the lines with a count of 1
             │        └─ count the occurrences of each line
             └─ group identical lines into adjacent positions
```

`uniq` detects only **adjacent** identical lines, which is why `sort` is applied first.

### Level 9 → 10: Readable strings in a binary file
**Commands:** `strings` · `grep ^`

```bash
bandit9@bandit:~$ strings data.txt | grep "^=="
========== password
========== isa
========== truKLdjsbJ5g7yyJ2X2R0o3a5HQJFuLk
```

`strings` extracts only the sequences of printable characters. In regular expressions, `^` anchors the match to the start of the line and `$` to the end.

> 🛡️ **Security:** `strings` is a first step in analyzing unknown executables and malware, because URLs, IP addresses, and passwords are often directly visible.

### Level 10 → 11: Base64 decoding
**Commands:** `base64 -d`

```bash
bandit10@bandit:~$ base64 -d data.txt
The password is IFukwKGsFW8MOq3IRFqrxE1hxTNEbUPR
```

| Concept | Function |
| :--- | :--- |
| **Encoding** (Base64, hex) | Changes the representation for transport. Requires no key and can be reversed by anyone. |
| **Encryption** (AES, RSA) | Conceals the content. Cannot be reversed without the key. |
| **Hashing** (MD5, SHA-256) | A one-way, fixed-length digest. Cannot be reversed. |

**Recognizing Base64:** only the characters `A–Z a–z 0–9 + /`, a length that is a multiple of 4, and frequently a trailing `=` or `==` (padding).

### Level 11 → 12: ROT13
**Commands:** `tr` · Caesar cipher

```bash
bandit11@bandit:~$ cat data.txt | tr 'A-Za-z' 'N-ZA-Mn-za-m'
The password is 5Te8Y4drgCRfCx8ugdwuEX8KFC6k2EUu
```

```
before: A B C D E F G H I J K L M N O P Q R S T U V W X Y Z
        │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │
after:  N O P Q R S T U V W X Y Z A B C D E F G H I J K L M
```

Because the alphabet has 26 letters, applying ROT13 twice restores the original text. The same command therefore both encodes and decodes.

### Level 12 → 13: Repeated decompression
**Commands:** `xxd -r` · `gzip` · `bzip2` · `tar` · `/tmp`

**Goal:** `data.txt` is a hexdump of a file that has been compressed repeatedly.

```bash
# The home directory is not writable, so we work in /tmp
bandit12@bandit:~$ mkdir /tmp/ax && cp data.txt /tmp/ax && cd /tmp/ax
bandit12@bandit:/tmp/ax$ xxd -r data.txt data.out
bandit12@bandit:/tmp/ax$ file data.out
data.out: gzip compressed data, was "data2.bin", ...
bandit12@bandit:/tmp/ax$ mv data.out data.gz && gzip -d data.gz
bandit12@bandit:/tmp/ax$ file data
data: bzip2 compressed data, block size = 900k
bandit12@bandit:/tmp/ax$ bzip2 -d data
...                                    # repeat: file → decompress
bandit12@bandit:/tmp/ax$ cat data8
The password is 8ZjyCRiBWFYkneahHwxCv3wb2a1ORpYL
```

| Output of `file` | Decompression command |
| :--- | :--- |
| `gzip compressed data` | `mv x x.gz && gzip -d x.gz` (gzip requires the `.gz` extension) |
| `bzip2 compressed data` | `bzip2 -d x` (produces `x.out` if it does not recognize the extension) |
| `POSIX tar archive` | `tar -xf x` |
| `ASCII text` | `cat x`: end of the process |

```
hexdump ─xxd -r─> gzip ─> bzip2 ─> gzip ─> tar ─> tar ─> bzip2 ─> tar ─> gzip ─> ASCII
```

---

## Part C: Networking and Keys (Levels 13–20)

### Level 13 → 14: Logging in with a private key
**Commands:** `ssh -i` · `scp` · `chmod 400`

```bash
# On the local machine (Linux, macOS, or PowerShell)
$ scp -P 2220 bandit13@bandit.labs.overthewire.org:sshkey.private .
$ chmod 400 sshkey.private
$ ssh -i sshkey.private bandit14@bandit.labs.overthewire.org -p 2220
bandit14@bandit:~$
```

The server holds the public key of `bandit14` in `~/.ssh/authorized_keys`. Anyone in possession of the corresponding private key can log in without a password.

> ⚠️ **Common mistakes:** (a) `scp` requires an **uppercase** `-P` for the port, whereas `ssh` uses a **lowercase** `-p`. (b) Without `chmod 400`, the client rejects the key with `UNPROTECTED PRIVATE KEY FILE`.

### Level 14 → 15: Sending data to a TCP port
**Commands:** `nc` · TCP · localhost

```bash
bandit14@bandit:~$ cat /etc/bandit_pass/bandit14 | nc localhost 30000
Correct!
BfMYroe26WYalil77FoDi9qh59eK5xNr
```

`nc` (netcat) opens a plain TCP connection: whatever is fed to its stdin is sent to the server, and the response is printed to stdout.

### Level 15 → 16: Communicating over SSL/TLS
**Commands:** `openssl s_client`

```bash
bandit15@bandit:~$ cat /etc/bandit_pass/bandit15 | openssl s_client -connect localhost:30001 -quiet
verify error:num=18:self signed certificate
Correct!
cluFn7wTiGryunymYOu4RcffSxQluehd
```

A TLS port first requires a handshake, so plain `nc` fails. `openssl s_client` completes the handshake and then behaves like `nc`. The `self signed certificate` message is expected in a lab environment.

### Level 16 → 17: Port scanning
**Commands:** `/dev/tcp` · `openssl` · `nmap`

**Goal:** one port in the range 31000–32000 speaks SSL and returns the credentials.

```bash
bandit16@bandit:~$ for i in {31000..32000}; do
>   (echo > /dev/tcp/localhost/$i) >& /dev/null && echo "Port $i open"
> done
Port 31518 open
Port 31790 open
bandit16@bandit:~$ cat /etc/bandit_pass/bandit16 | openssl s_client -connect localhost:31790 -quiet
Correct!
-----BEGIN RSA PRIVATE KEY-----
...
-----END RSA PRIVATE KEY-----
```

```
31000 ───────────────────── 31518 ─────────── 31790 ──────── 32000
                              │                 │
                              echo              SSL ok ──> RSA private key
```

Bash supports the special path `/dev/tcp/HOST/PORT`: writing to it attempts a TCP connection. Save the key (e.g. as `sshkey`), apply `chmod 400`, and log in as `bandit17`. With a dedicated tool: `nmap -sV -p 31000-32000 localhost`.

### Level 17 → 18: Comparing files
**Commands:** `diff`

```bash
bandit17@bandit:~$ diff passwords.old passwords.new
42c42
< 6vcSC74ROI95NqkKaeEC2ABVMDX9TyUr
---
> kfBf3eYk5BPBRzwjqutbbfE887SVc5Yd
```

`42c42` means that line 42 has changed (c = change). `<` refers to the first (old) file and `>` to the second (new) file.

### Level 18 → 19: Bypassing a modified `.bashrc`
**Commands:** `ssh <command>`

```bash
$ ssh bandit18@bandit.labs.overthewire.org -p 2220
Byebye !
Connection to bandit.labs.overthewire.org closed.
$ ssh bandit18@bandit.labs.overthewire.org -p 2220 "cat readme"
IueksS7Ubh8G3DCwVzrTd8rAVOwq3M5x
```

`.bashrc` is loaded only by an **interactive** shell. When the command is passed as an argument to `ssh`, it is executed without an interactive shell, so the modification is never triggered. In PuTTY: *Connection → SSH → Remote command*.

### Level 19 → 20: SUID executable
**Commands:** setuid · privilege escalation

```bash
bandit19@bandit:~$ ls -l bandit20-do
-rwsr-x--- 1 bandit20 bandit19 7296 Oct 16 14:00 bandit20-do
bandit19@bandit:~$ ./bandit20-do cat /etc/bandit_pass/bandit20
GbKksEFF4yrVs6il55v6gwY5aVje5f0j
```

The `s` in place of the owner's `x` (`-rwsr-x---`) is the **SUID** bit: the file runs with the privileges of **its owner** (`bandit20`), not those of the user who launches it.

> 🛡️ **Security:** once access has been obtained, `find / -perm -4000 2>/dev/null` is one of the first steps when searching for privilege escalation vectors. You will encounter it again in [DC-CTF](../04.%20DC-CTF/README.md), [Raven2-CTF](../05.%20Raven2-CTF/README.md), and [Mr Robot](../06.%20Mr%20Robot/README.md).

### Level 20 → 21: Listening on a port with netcat
**Commands:** `nc -l` · listener · setuid

**Goal:** the SUID binary `suconnect` connects to a port on localhost, reads one line and, if it matches the password of `bandit20`, sends back the password of `bandit21`.

```bash
# Terminal 1: the listener
bandit20@bandit:~$ nc -lp 31337 < /etc/bandit_pass/bandit20
gE269g2h3mw3pwgrj0Ha9Uoqen1c9DGr

# Terminal 2: the client
bandit20@bandit:~$ ./suconnect 31337
Read: GbKksEFF4yrVs6il55v6gwY5aVje5f0j
Password matches, sending next password
```

```
  Terminal 1                                    Terminal 2
┌──────────────────┐  ① connect to 31337     ┌────────────────────┐
│  nc -lp 31337    │ <────────────────────── │  ./suconnect 31337 │
│  (listener)      │ ② bandit20 password ──> │  (runs as          │
│                  │ <── ③ bandit21 password │   bandit21)        │
└──────────────────┘                         └────────────────────┘
```

With a single terminal, run the listener in the background: `nc -lp 31337 < /etc/bandit_pass/bandit20 &`, followed by `./suconnect 31337`.

---

## Part D: Automation (Levels 21–24)

### Level 21 → 22: Cron jobs (I): reading the configuration
**Commands:** `cron` · `/etc/cron.d`

```bash
bandit21@bandit:~$ cat /etc/cron.d/cronjob_bandit22
* * * * * bandit22 /usr/bin/cronjob_bandit22.sh &> /dev/null
bandit21@bandit:~$ cat /usr/bin/cronjob_bandit22.sh
#!/bin/bash
chmod 644 /tmp/t7O6lds9S0RqQh9aMcz6ShpAoZKF7fgv
cat /etc/bandit_pass/bandit22 > /tmp/t7O6lds9S0RqQh9aMcz6ShpAoZKF7fgv
bandit21@bandit:~$ cat /tmp/t7O6lds9S0RqQh9aMcz6ShpAoZKF7fgv
Yk7owGAcWjwMVRwrTesJEwB7WVOiILLI
```

```
*    *    *    *    *    bandit22   /usr/bin/cronjob_bandit22.sh
│    │    │    │    │    │          │
│    │    │    │    │    │          └─ the command to be executed
│    │    │    │    │    └─ the user under which it runs
│    │    │    │    └─ day of week (0–7)
│    │    │    └─ month (1–12)
│    │    └─ day of month (1–31)
│    └─ hour (0–23)
└─ minute (0–59)          * = "every"  →  here: every minute
```

### Level 22 → 23: Cron jobs (II): analyzing a script
**Commands:** bash script · `md5sum` · `whoami`

```bash
bandit22@bandit:~$ cat /usr/bin/cronjob_bandit23.sh
#!/bin/bash
myname=$(whoami)
mytarget=$(echo I am user $myname | md5sum | cut -d ' ' -f 1)
cat /etc/bandit_pass/$myname > /tmp/$mytarget
bandit22@bandit:~$ echo "I am user bandit23" | md5sum
8ca319486bfbbc3663ea0fbe81326349  -
bandit22@bandit:~$ cat /tmp/8ca319486bfbbc3663ea0fbe81326349
jc1udXuA1tiHqjIsL8yaapX5XIAI6i0n
```

The script is executed by cron as `bandit23`, so `$myname = bandit23`. It suffices to compute the same MD5 hash.

> ⚠️ **Common mistake:** running the script as `bandit22`. In that case `whoami` returns `bandit22` and the resulting hash is wrong.

### Level 23 → 24: Cron jobs (III): writing a script
**Commands:** shell script · `/var/spool` · directory permissions

**Goal:** the cron job of `bandit24` executes and then deletes every script in `/var/spool/bandit24`.

```bash
bandit23@bandit:~$ mkdir /tmp/alex1234 && cd /tmp/alex1234
bandit23@bandit:/tmp/alex1234$ cat > script.sh <<'EOF'
#!/bin/sh
cat /etc/bandit_pass/bandit24 >> /tmp/alex1234/bandit24pass
EOF
bandit23@bandit:/tmp/alex1234$ chmod 777 script.sh /tmp/alex1234
bandit23@bandit:/tmp/alex1234$ cp script.sh /var/spool/bandit24/
# Wait up to one minute
bandit23@bandit:/tmp/alex1234$ cat bandit24pass
UoMYTrfrBFHyQXmg6gzctqAwOmw1IohZ
```

```
cron ──(every minute)──> cronjob_bandit24.sh ──(as bandit24)──> /var/spool/bandit24/script.sh
                                                                     │
                                                                     ▼
                                                     /tmp/alex1234/bandit24pass
```

The `chmod 777` on the directory is necessary because the script runs as `bandit24`: without write permission, the output file cannot be created.

> 🛡️ **Security:** writable directories or scripts executed by `root` via cron are a common privilege escalation path on real-world systems.

### Level 24 → 25: Brute-forcing a four-digit PIN
**Commands:** brute force · `seq` · `nc`

```bash
bandit24@bandit:~$ PASS=$(cat /etc/bandit_pass/bandit24)
bandit24@bandit:~$ for pin in $(seq -w 0000 9999); do echo "$PASS $pin"; done \
>     | nc localhost 30002 | grep -v Wrong
Correct!
The password of user bandit25 is uNG9O58gUE7snukf3bvZ0rxhtnjzSGzG
```

`seq -w 0000 9999` generates every PIN with leading zeros. All attempts are sent over a **single** connection, which is far faster than 10,000 separate ones. If the service closes the connection prematurely, split the range (e.g. `0000–4999` and `5000–9999`).

> 🛡️ **Security:** 10,000 combinations can be exhausted within seconds. This is why real systems enforce **rate limiting** or account lockout.

---

## Part E: Shell Escapes and Git (Levels 25–33)

### Level 25 → 26: Escaping via `more`
**Commands:** `/etc/passwd` · `more` · `vi`

```bash
bandit25@bandit:~$ grep bandit26 /etc/passwd
bandit26:x:11026:11026:bandit level 26:/home/bandit26:/usr/bin/showtext
bandit25@bandit:~$ cat /usr/bin/showtext
#!/bin/sh
export TERM=linux
more ~/text.txt
exit 0
bandit25@bandit:~$ ssh -i bandit26.sshkey bandit26@localhost
# Shrink the window (to about 6 lines) and connect again:
--More--(83%)
# Press v to open vi. Inside vi:
:e /etc/bandit_pass/bandit26
5czgV9L3Xx8JPOyRbXh6lQbmIOWvPT6Z
```

```
ssh bandit26 ──> showtext ──> more ──[ v ]──> vi ──[ :set shell / :! ]──> bash
                              │
                              └─ pauses at --More-- only if the text does not fit
```

Any program capable of launching another program is a potential escape route. With a small window, `more` pauses at the `--More--` prompt, where pressing `v` opens `vi`.

### Level 26 → 27: Shell commands from `vi`
**Commands:** `:set shell` · `:!` · setuid

```
:set shell=/bin/bash
:!ls -la
-rwsr-x---  1 bandit27 bandit26 7296 Oct 16 14:00 bandit27-do
:!./bandit27-do cat /etc/bandit_pass/bandit27
3ba3118a22e93127a4ed485be72ef5ea
```

`vi` executes `:!` commands through the user's login shell (here, `showtext`). Setting `:set shell=/bin/bash` selects bash instead. Then, as in level 19, the SUID binary `bandit27-do` reads the password. The `:shell` command opens a full interactive shell.

> 🛡️ **Security:** such escape techniques are systematically catalogued in [GTFOBins](https://gtfobins.github.io/), a key resource for penetration testing.

### Level 27 → 28: Git (I): cloning
**Commands:** `git clone`

```bash
bandit27@bandit:~$ mkdir /tmp/repo123 && cd /tmp/repo123
bandit27@bandit:/tmp/repo123$ git clone ssh://bandit27-git@localhost/home/bandit27-git/repo
bandit27@bandit:/tmp/repo123$ cat repo/README
The password to the next level is: 0ef186ac70e04ea33b4c1853d2526fa2
```

The password of `bandit27-git` is the same as that of `bandit27`. The next four levels follow the same procedure (`bandit28-git`, `bandit29-git`, …).

### Level 28 → 29: Git (II): recovery from history
**Commands:** `git log` · `git checkout`

```bash
bandit28@bandit:/tmp/repo1337/repo$ cat README.md
- password: xxxxxxxxxx
bandit28@bandit:/tmp/repo1337/repo$ git log --oneline
073c27c fix info leak
186a103 add missing data
b67405d initial commit of README.md
bandit28@bandit:/tmp/repo1337/repo$ git checkout 186a103
bandit28@bandit:/tmp/repo1337/repo$ cat README.md
- password: bbc96594b4e001778eee9975372716b2
```

```
b67405d ─────────> 186a103 ─────────────> 073c27c  (HEAD)
initial commit     add missing data       fix info leak
                   ▲ contains the password ▲ removed it from the README
```

Alternatively: `git log -p` or `git show 073c27c`.

> 🛡️ **Security:** passwords and API keys that have been committed remain in the history even if they are removed later. Tools such as **trufflehog** and **gitleaks** detect such leaks.

### Level 29 → 30: Git (III): branches
**Commands:** `git branch -r` · `git checkout`

```bash
bandit29@bandit:/tmp/plop123/repo$ cat README.md
- password: <no passwords in production!>
bandit29@bandit:/tmp/plop123/repo$ git branch -r
  origin/dev
  origin/master
  origin/sploits-dev
bandit29@bandit:/tmp/plop123/repo$ git checkout dev && cat README.md
- password: 5b90576bedb2cc04c86a9e924ce42faf
```

The phrase "no passwords in production" is a hint: `master` is the production branch, so we search the other branches.

### Level 30 → 31: Git (IV): tags
**Commands:** `git tag` · `git show`

```bash
bandit30@bandit:/tmp/plop1234/repo$ git tag
secret
bandit30@bandit:/tmp/plop1234/repo$ git show secret
47e603bb428404d265f59c42920d81e5
```

Neither the history nor the branches contain the password. **Tags** are named references to objects in the repository—in this case, to a blob.

### Level 31 → 32: Git (V): pushing changes
**Commands:** `git add -f` · `.gitignore` · `git push` · hooks

```bash
bandit31@bandit:/tmp/plop12345/repo$ echo "May I come in?" > key.txt
bandit31@bandit:/tmp/plop12345/repo$ git add -f key.txt
bandit31@bandit:/tmp/plop12345/repo$ git commit -m "key.txt"
bandit31@bandit:/tmp/plop12345/repo$ git push origin master
remote: Well done! Here is the password for the next level:
remote: 56a9bf19c63d650ce78e6ec0354ee45e
 ! [remote rejected] master -> master (pre-receive hook declined)
```

The `.gitignore` file contains the rule `*.txt`, so `add -f` is required. On the server, a **pre-receive hook** displays the password and then rejects the push. The rejection is expected.

### Level 32 → 33: Escaping an uppercase shell
**Commands:** `$0`

```
WELCOME TO THE UPPERCASE SHELL
>> ls
sh: 1: LS: not found
>> $0
$ whoami
bandit33
$ cat /etc/bandit_pass/bandit33
c9c3199ddf4121b10cf581a98d51caee
```

The shell converts all input to uppercase, and Linux is case-sensitive (`LS` ≠ `ls`). The variable `$0`, however, contains no letters: it holds the name of the current shell (`sh`), so executing it spawns a regular shell. Because the program is SUID, the new shell runs as `bandit33`.

### Level 33: Completion

```bash
bandit33@bandit:~$ cat README.txt
Congratulations on solving the last level of this game!
```

As a next step, the **Leviathan** wargame (short, with a focus on executables) and **Natas** (web application security) are recommended.

---

## Appendix A: Command Summary by Level

| Level | Key command | Concept |
| :--- | :--- | :--- |
| 0–3 | `ssh`, `ls -la`, `cat ./-` | Connecting, special filenames, hidden files |
| 4–6 | `file`, `find` | Searching by type, size, and owner |
| 7–9 | `grep`, `sort \| uniq`, `strings` | Text filtering with pipelines |
| 10–12 | `base64 -d`, `tr`, `xxd -r` | Encodings and compression |
| 13 | `scp`, `ssh -i` | Key-based authentication |
| 14–16 | `nc`, `openssl s_client`, `/dev/tcp` | TCP, TLS, port scanning |
| 17–18 | `diff`, `ssh host "cmd"` | File comparison, non-interactive shell |
| 19–20 | `./bandit20-do`, `nc -l` | SUID, listening on a port |
| 21–23 | `cat /etc/cron.d/*`, `md5sum` | Analyzing and exploiting cron jobs |
| 24 | `seq -w` + `nc` | Brute force |
| 25–26 | `more` → `vi` → `:!` | Escaping a restricted shell |
| 27–31 | `git log / branch / tag / push` | Sensitive data in git repositories |
| 32 | `$0` | Bypassing an input filter |

## Appendix B: Glossary

| Term | Meaning |
| :--- | :--- |
| **SSH** | Protocol for encrypted remote shell access (port 22; here 2220) |
| **PuTTY / PuTTYgen** | Graphical SSH client for Windows / tool for managing and converting keys |
| **stdin / stdout / stderr** | Standard input / output / error output (streams 0 / 1 / 2) |
| **Pipeline (`\|`)** | Connects the stdout of one command to the stdin of the next |
| **Dotfile** | A file whose name begins with `.`; not shown by a plain `ls` |
| **Base64** | Encoding of binary data as text (not encryption) |
| **ROT13** | Shifting letters by 13 positions; a variant of the Caesar cipher |
| **Hexdump** | Representation of bytes in hexadecimal form (`xxd`) |
| **TLS / SSL** | Protocols for encrypting a TCP connection |
| **Port scanning** | Checking which ports on a system accept connections |
| **SUID** | Execution of a file with the privileges of its owner |
| **Cron** | Scheduler for periodic jobs |
| **Brute force** | Trying every possible value (exhaustive search) |
| **Shell escape** | Obtaining a full shell from within a restricted program |
| **Git hook** | A script that runs automatically on git events (e.g. push) |

## Appendix C: Skills Checklist

- [ ] I can connect with `ssh` or PuTTY on a non-default port, using a password or a private key
- [ ] I can convert an OpenSSH key to `.ppk` format with PuTTYgen
- [ ] I can handle files with unusual names (`-`, spaces, a leading dot)
- [ ] I can construct a `find` command with size, owner, and permission filters
- [ ] I can explain the difference between `2>/dev/null` and `2>&1`
- [ ] I can compose pipelines with `sort`, `uniq`, `grep`, and `tr`
- [ ] I can distinguish between encoding, encryption, and hashing
- [ ] I can interact with a network service via `nc` and `openssl s_client`
- [ ] I can explain the meaning of the `s` in `-rwsr-x---` and the risks it entails
- [ ] I can interpret a crontab line and identify the user under which it runs
- [ ] I can search for sensitive data in the history, branches, and tags of a git repository
- [ ] I can escape from a restricted shell via `vi` or `$0`

### Summary
Bandit requires no offensive tooling: every level can be solved with basic Linux commands. The same techniques (hidden files, `find`, SUID, cron, shell escapes, git leaks) reappear in the CTF machines that follow, beginning with [03. Raven-CTF](../03.%20Raven-CTF/README.md).

**Source:** A. Cheron, "[OTW] Write-up for the Bandit Wargame", [axcheron.github.io](https://axcheron.github.io/writeups/otw/bandit/). English adaptation and explanatory notes for the CSEC 101 course.
