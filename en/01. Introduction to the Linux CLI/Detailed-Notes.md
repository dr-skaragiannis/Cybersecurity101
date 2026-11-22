
# Detailed Notes — Lab 01: Introduction to the Linux CLI

This file is the **extended edition** of `README.md`.

**What the CLI is and why we learn it**

The **CLI (Command Line Interface)** is a text-based way of interacting with a computer: you type a command, the **shell** reads it, executes it, and returns output. The shell you will typically use is **bash** (Bourne Again SHell).

Why use a CLI rather than a graphical environment?

| Advantage | Explanation |
| :--- | :--- |
| **Automation** | A single command can run on 1,000 servers via a script or SSH |
| **Precision** | You see exactly what was executed — essential during security incidents |
| **Speed** | `grep` across 10,000 files in seconds |
| **Availability** | Every Linux server has a shell; not all have a graphical environment |
| **Audit trail** | Every command is recorded in the history file (`~/.bash_history`) |

**The structure of a command**

```
ls   -lh   /var/log   >  out.txt
│     │     │           │
│     │     │           └─ output redirection
│     │     └─ argument (what it acts on)
│     └─ option/flag (how it should act)
└─ command name
```

The general form is: **`command [options] [arguments]`**. Options begin with `-` (short form) or `--` (long form) and can often be combined: `ls -l -h` = `ls -lh`.

**How to use these notes:** open a terminal and execute each command as you read it. Proficiency with the CLI is not acquired by reading alone.

---

## Lab Exercise 0: Introduction to the Command Line Interface (CLI) & Basic Networking Commands

### Learning Objectives


### Explanation of Part A — the filesystem as a tree

Linux organizes everything into **a single unified tree** that starts at the root `/`. There are no lettered "drives" such as `C:` — disks are *mounted* onto directories.

```
/                 root directory
├── bin/          essential executables (ls, cp, bash)
├── etc/          configuration files
├── home/         users' home directories (/home/kali)
├── var/          variable data (logs, databases, web root)
├── tmp/          temporary files (cleared on reboot)
├── usr/          user programs (usr/bin, usr/share)
└── proc/         kernel virtual filesystem (exists only in memory)
```

**Absolute vs. relative path** — the fundamental distinction of this entire section:

* An **absolute** path begins with `/` and is valid from anywhere: `/home/kali/lab1_workspace`.
* A **relative** path starts from the *current* directory: `project/src` means "inside the directory I am currently in."

**Special symbols:**

| Symbol | Meaning |
| :--- | :--- |
| `.` | the current directory |
| `..` | the parent directory (one level up) |
| `~` | the user's home directory (`/home/kali`) |
| `-` | the *previous* directory (for `cd` only) |
| `*` | wildcard: "anything" |

Upon completion of this lab session, students are expected to:

1. Become familiar with the command line interface (CLI) environment and basic command syntax.
2. Gain a thorough understanding of navigation using absolute and relative paths (`cd`, `cd ..`, `cd ../..`, `cd -`).
3. Apply advanced options and search patterns with the `ls` listing command (`-lh`, `-lt`, `-R`, `*`).
4. Apply output redirection and piping techniques.
5. Manage file access permissions using symbolic and numeric (octal) notation.
6. Inspect and analyze basic network configuration and connectivity parameters (`ifconfig`, `ip`, `ping`, `ss`, `nslookup`).

---

## Introduction to the CLI: Lab Structure & Directory Hierarchy Setup


#### Explanation 1.1 — `pwd`, `mkdir -p`, and why structure matters

* **`pwd`** (*print working directory*) prints the **absolute** path of the current directory. It is the CLI's answer to "where am I?" and the first command to run whenever you lose your bearings.
* **`mkdir`** (*make directory*) creates a directory. Without options, it fails if the parent directory does not exist:

```bash
mkdir a/b/c      # error: a does not exist
mkdir -p a/b/c   # creates the entire path
```

* **`-p`** (*parents*) serves two purposes: it creates **all** intermediate directories **and** does not complain if the directory already exists (idempotent behavior). This is why it is used ubiquitously in scripts.

**What the exercise does in practice:** `lab1_workspace/project/{docs,src,build}` uses **brace expansion** — bash expands `{docs,src,build}` into three separate arguments *before* `mkdir` is executed. It is equivalent to:

```bash
mkdir -p lab1_workspace/project/docs lab1_workspace/project/src lab1_workspace/project/build
```

**Common mistake:** writing `mkdir -p lab1_workspace/project/{docs, src}` **with a space** after the comma — bash then creates a directory named `" src"` (with a space in the name). Spaces in filenames are legal but cause problems everywhere later on.

1. Determine your initial working directory:
   ```bash
   pwd
   ```

2. Create the following directory tree for the navigation exercises:
   ```bash
   mkdir -p lab1_workspace/project/{docs,src,build}
   mkdir -p lab1_workspace/backup
   ```

3. Change into the main exercise directory:
   ```bash
   cd lab1_workspace
   pwd
   ```

---

## Extended Navigation Examples with `cd` (Relative & Absolute Paths)


#### Explanation 1.2 — how the shell reasons when you type `cd ../..`

The shell does not "know" where you are in any deeper sense; it simply **combines** the path you supply with the current one. If you are in `/home/user/lab1_workspace/project/src`:

| Command | Result | Rationale |
| :--- | :--- | :--- |
| `cd ..` | `/home/user/lab1_workspace/project` | removes the last component |
| `cd ../..` | `/home/user/lab1_workspace` | removes two components |
| `cd .` | same directory | nothing changes (useful in scripts) |
| `cd ~` or `cd` | `/home/user` | the home directory |
| `cd -` | the previous one | the shell remembers `$OLDPWD` |
| `cd /var/log` | `/var/log` | absolute path; ignores where you are |

**Why `cd -` is useful:** when you alternate between two directories (e.g., `/var/www/html` and `/etc/apache2`), it saves you typing. It also prints the path you moved to, so you always know where you are.

**`cd` without an argument** always goes to `$HOME`. If `$HOME` is not set (rare, but possible in scripts), the result is unpredictable — which is why scripts use `cd "$HOME" || exit 1`.

**Edge case:** `cd ..` from the root `/` leaves you at `/` (there is nothing "above" the root).

The `cd` (*Change Directory*) command allows you to move through the hierarchical directory tree.

1. **Moving into a deeper subdirectory (relative path):**
   ```bash
   cd project/src
   pwd
   # Result: /home/user/lab1_workspace/project/src
   ```

2. **Moving up one level (`..`):**
   ```bash
   cd ..
   pwd
   # You are now in: /home/user/lab1_workspace/project
   ```

3. **Moving up multiple levels of the hierarchy at once (`../..`):**
   ```bash
   # From project/src/, move directly two levels up
   cd src
   cd ../..
   pwd
   # You are back in the root directory of the exercise: lab1_workspace
   ```

4. **Switching to the immediately previous directory (`cd -`):**
   The `-` argument automatically returns you to the directory you were in before the last change:
   ```bash
   cd project/build
   cd -
   # Return to lab1_workspace
   cd -
   # Return to project/build
   ```

5. **Moving to the user's home directory (`cd ~` or `cd`):**
   ```bash
   cd ~
   pwd
   # Return to the exercise working directory via a relative path:
   cd lab1_workspace
   ```

6. **Moving to a sibling directory via a relative path:**
   Move from `project/docs/` to `backup/` without first passing through the root:
   ```bash
   cd project/docs
   cd ../../backup
   pwd
   # You are in: lab1_workspace/backup
   cd ..
   ```

---

#### Step 1.3: Detailed Usage and Options of the `ls` Command


#### Explanation 1.3 — reading the output of `ls -l`

`ls -l` displays **7 fields**. Example:

```
-rwxr-xr--  1  kali  developers  15240  Oct 24 10:15  script.sh
│└┬┘└┬┘└┬┘  │   │        │         │         │          │
│ │  │  │   │   │        │         │         │          └─ name
│ │  │  │   │   │        │         │         └─ modification date
│ │  │  │   │   │        │         └─ size in bytes
│ │  │  │   │   │        └─ group
│ │  │  │   │   └─ owner
│ │  │  │   └─ number of hard links
│ │  │  └─ permissions for others
│ │  └─ permissions for group
│ └─ permissions for owner  →  rwx = read/write/execute
└─ type: - file, d directory, l link, c/b device
```

**The options you will actually need:**

| Option | What it does | When to use it |
| :--- | :--- | :--- |
| `-l` | long listing | whenever permissions/ownership matter |
| `-a` | also shows hidden entries (`.` `..` `.env`) | when looking for configuration files or secrets |
| `-h` | sizes in K/M/G | always together with `-l` |
| `-t` | sort by time (newest first) | "what changed most recently?" |
| `-r` | reverse the order | `-ltr`: oldest first |
| `-R` | recurse into subdirectories | mapping a directory structure |
| `-S` | sort by size | finding large files |

**Why `-ltr` is the "administrator's tool":** in a directory with 500 log files, the newest ones appear at the **bottom** — right next to the cursor, with no scrolling required. It is the classic combination for answering "what did the application just write?"

**Hidden files and security:** files beginning with `.` (e.g., `.env`, `.gitconfig`, `.ssh/`) are not shown by a plain `ls`. On a web server, running `ls -la` in `/var/www/html` may reveal `.htaccess` or backup files that should not be accessible — which is why it is a fundamental enumeration step.

**Wildcards (globbing):** `*` is expanded **by the shell**, not by `ls`. In other words, `ls *.c` becomes `ls main.c utils.c` before it is executed. If no file matches, bash passes `*.c` through literally and `ls` complains "No such file."

The `ls` (*List*) command displays the contents of a directory. Different flags modify the level of detail provided.

1. **Creating test files for observation:**
   ```bash
   cd lab1_workspace/project/src
   touch main.c utils.c config.h .env_secret
   echo "Linux CLI Lab" > notes.txt
   ```

2. **Simple listing versus long listing (`ls` vs `ls -l`):**
   ```bash
   ls
   ls -l
   ```
   *Observe the 7 fields of the long listing: file type/permissions, number of links, owner, group, size in bytes, modification timestamp, and name.*

3. **Displaying hidden files (`ls -a` and `ls -la`):**
   In Linux, files whose names begin with a dot (`.`) are hidden:
   ```bash
   ls -la
   ```
   *Identify the entries `.`, `..`, and the hidden file `.env_secret`.*

4. **Displaying sizes in human-readable form (`ls -lh`):**
   The `-h` (*human-readable*) option converts bytes into K, M, G:
   ```bash
   ls -lh
   ```

5. **Sorting by modification time (`ls -lt` and `ls -ltr`):**
   * `ls -lt`: Sorts from most recent to oldest.
   * `ls -ltr`: Reverse sorting (the most recent files appear at the bottom of the screen):
   ```bash
   ls -ltr
   ```

6. **Recursive listing of subdirectories (`ls -R`):**
   Displays the full contents of all subdirectories in a tree-like fashion:
   ```bash
   cd ../..
   ls -R lab1_workspace
   ```

7. **Filtering with wildcard characters (Wildcards / Globbing):**
   * Display only files with the `.c` extension:
     ```bash
     ls lab1_workspace/project/src/*.c
     ```
   * Display files whose names begin with `u`:
     ```bash
     ls lab1_workspace/project/src/u*
     ```

> **Question 1.1:** What is the difference between the command `cd ..` and the command `cd .`?
> **Question 1.2:** Which combination of `ls` options is preferable when trying to determine which file was modified most recently in a directory containing hundreds of files?

---

### Part B: Data Filtering and Pipelines


### Explanation of Part B — the three data streams

Every Linux program has **three default streams**:

| Stream | FD | Meaning | Default |
| :--- | :--- | :--- | :--- |
| **stdin** | 0 | input | keyboard |
| **stdout** | 1 | normal output | screen |
| **stderr** | 2 | error messages | screen |

The power of the CLI comes from the fact that these streams can be **redirected**:

```
command > file        # stdout to the file (overwrites)
command >> file       # stdout to the file (appends to the end)
command 2> err.txt    # stderr to a file
command 2>/dev/null   # discards errors (into the kernel's "sink")
command > out 2>&1    # both into the same file
command < file        # stdin from a file
command1 | command2   # stdout of the 1st becomes stdin of the 2nd
```

**Why `2>/dev/null` appears everywhere in security work:** when you run `find /` as an unprivileged user, you receive hundreds of "Permission denied" messages on stderr. Suppressing fd 2 leaves only the useful stdout — and it is faster than filtering with `grep -v`.

#### Step 2.1: Using `grep` and Redirection


#### Explanation 2.1 — `grep` as a line filter

**`grep`** (*global regular expression print*) reads lines and prints those that **match** a pattern.

```bash
grep "model name" /proc/cpuinfo        # searches within a file
cat file | grep -i error               # -i: case-insensitive
grep -c "Failed" /var/log/auth.log     # -c: counts matching lines
grep -v "INFO" app.log                 # -v: inverted (lines that do NOT match)
grep -rn "password" /var/www           # -r recursive, -n with line numbers
grep -E "error|warn" app.log           # -E: extended regex (alternation with |)
```

**What `/proc` is:** a virtual filesystem created by the **kernel** in memory. It does not exist on disk; every read generates data at that moment. `/proc/cpuinfo` provides CPU information, `/proc/meminfo` memory information, and `/proc/net/tcp` active connections.

**Why redirect rather than simply print:** `>` **creates/truncates** the file. Caution: `grep x file > file` destroys `file` (it is truncated before it is read). Correct usage: `grep x file > out.txt`.

**Security relevance:** `grep -rn "password"` is one of the first actions taken after gaining access to a machine — it searches for credentials in source code and configuration files (you will see this again in DC-1 with `wp-config.php`).

Search for processor information in the `/proc` virtual filesystem:
```bash
grep "model name" /proc/cpuinfo > cpu_spec.txt
head -n 2 cpu_spec.txt
```

#### Step 2.2: Pipelines


#### Explanation 2.2 — why `|` changes everything

The **pipe** `|` connects the stdout of the left-hand command to the stdin of the right-hand command **concurrently** (streaming): it neither waits for the first command to finish nor writes to a temporary file.

```bash
ps aux | wc -l            # how many processes
ps aux | grep sshd        # only sshd
ps aux | grep sshd | grep -v grep   # excluding the grep process itself
ls -l | sort -k5 -n | tail -3       # the 3 largest files
```

**Why `grep -v grep` is needed:** the `grep sshd` process itself appears in the process list and matches its own pattern. This is the classic "false positive" of pipelines.

**Analysis of the answer:** `ps aux | wc -l` counts **lines**, and the first line is the **header** (`USER PID %CPU ...`). The actual number of processes is therefore `result - 1`. This detail is frequently examined.

**The pipeline as a philosophy:** each command does **one** thing well and communicates via text. Small tools are thus combined into complex workflows — the same logic you will later encounter when using `grep` on SIEM logs.

Count the number of active processes on the system by combining the `ps` command with the `wc` counting command:
```bash
ps aux | wc -l
```

> **Question 2.1:** What is the role of the pipe character `|` in the flow of data between two commands?

---

### Part C: Basic Network Diagnostic Commands


### Explanation of Part C — the model you need

To understand these commands, keep three layers in mind:

| Layer | What it contains | Example |
| :--- | :--- | :--- |
| **L2 (Data Link)** | MAC addresses, switches | `00:0c:29:65:4d:49` |
| **L3 (Network)** | IP addresses, routers | `192.168.35.131` |
| **L4 (Transport)** | ports, TCP/UDP | `22/tcp`, `80/tcp` |

A **port** is a "door" on an IP address that distinguishes between **services**. The same IP address can run SSH (22), HTTP (80), and MySQL (3306) simultaneously; the port determines which one you are talking to.

**Port ranges:** 0–1023 are *well-known* ports (binding to them requires root), 1024–49151 are *registered*, and 49152–65535 are *dynamic*. This is why an unprivileged user cannot listen on port 80.

#### Step 3.1: Inspecting Network Interfaces (`ifconfig` / `ip`)


#### Explanation 3.1 — reading `ip addr`

`ifconfig` belongs to the `net-tools` package and has been **deprecated** in modern distributions; its replacement is `ip` (from the `iproute2` package). Useful equivalents:

| Legacy | Modern |
| :--- | :--- |
| `ifconfig` | `ip addr` or `ip a` |
| `ifconfig eth0 up` | `ip link set eth0 up` |
| `route -n` | `ip route` |
| `netstat -tuln` | `ss -tuln` |
| `arp -a` | `ip neigh` |

**The fields to look at:**

```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
    link/ether 08:00:27:aa:bb:cc brd ff:ff:ff:ff:ff:ff
    inet 192.168.35.129/24 brd 192.168.35.255 scope global eth0
```

* `UP` → the interface is active.
* `link/ether` → the **MAC** address (L2).
* `inet .../24` → the **IPv4** address and network size (`/24` = 256 addresses, 254 usable).
* `inet6` → IPv6 (if present).

**The `lo` (loopback) interface:**

* It is **internal** to the machine and never reaches the physical wire.
* It always has `127.0.0.1/8` (and `::1` for IPv6).
* It allows local services to communicate with each other using the same API they would use remotely.
* If a service listens **only** on `127.0.0.1`, it is not reachable from another machine — an important detail during enumeration.

**In the lab:** the VM's IP address (e.g., `192.168.35.129`) is the one you will use as the "attacker" address, and the target's IP address will be discovered using `netdiscover`/`nmap`.

1. Display the available network interfaces and their assigned IP addresses:
   ```bash
   ifconfig
   ```
   *(If the `ifconfig` command is not installed on the system, run `ip addr show` instead.)*

2. Identify the IPv4 address (`inet`) of the primary interface (e.g., `eth0`, `enp3s0`, or `wlan0`), as well as that of the local loopback interface (`lo`).

> **Question 3.1:** What is the role of the `lo` (loopback) interface, and what is the default IPv4 address assigned to it?

#### Step 3.2: Testing Connectivity with the `ping` Command


#### Explanation 3.2 — what a `ping` actually tells you

`ping` sends an **ICMP Echo Request** and waits for an **Echo Reply**. It is the simplest test of "does the host exist and respond?"

**Reading the output:**

```
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 0.043/0.055/0.062/0.008 ms
```

| Metric | Meaning |
| :--- | :--- |
| **packet loss** | % of packets lost — 0% on a healthy local network |
| **rtt** (round-trip time) | time to the destination and back, in ms |
| **mdev** | variation (jitter) — a high value indicates an unstable network |

**Interpretation on a local network:** `rtt < 1 ms` is normal. `> 10 ms` on a LAN indicates a problem (load, a faulty virtual network adapter, or the wrong network).

**Why `ping` is not a reliable test of "is it alive?":** many hosts and firewalls **block** ICMP. "100% packet loss" does not necessarily mean the machine is down — which is why in CTFs we use **ARP** (`netdiscover`) or TCP scans (`nmap -Pn`).

**`-c N`:** sends N packets and stops. Without it, ping runs **indefinitely** until `Ctrl+C` — never leave it in a script without a limit.

1. Verify that the local network stack is functioning:
   ```bash
   ping -c 3 127.0.0.1
   ```

2. Test connectivity to an external host (e.g., a DNS server) by sending exactly 4 ICMP echo request packets:
   ```bash
   ping -c 4 8.8.8.8
   ```

> **Question 3.2:** Record the packet loss percentage and the average round-trip time (RTT) from the `ping` output.

#### Step 3.3: Active Connections and Name Resolution (`ss`, `nslookup`, `traceroute`)


#### Explanation 3.3 — listening ports, names, and routes

**`ss -tuln`** — breakdown of the options:

| Flag | Meaning |
| :--- | :--- |
| `-t` | TCP |
| `-u` | UDP |
| `-l` | **listening** (ports waiting for a connection) |
| `-n` | numeric (no reverse DNS — faster) |

The output shows `Local Address:Port`. `0.0.0.0:22` means "listening on **all** interfaces" (reachable from the network), while `127.0.0.1:3306` means "local only" (MySQL is not exposed). This distinction is **critical** during an audit.

**`nslookup www.kernel.org`** — queries the **DNS resolver**:

```
Server:  192.168.1.1        ← who answered
Address: 192.168.1.1#53
Name:    www.kernel.org
Address: 139.178.84.217     ← the A record (IPv4)
```

If it fails, either no DNS server is configured (`/etc/resolv.conf`) or you have no network connectivity. Alternatives: `dig`, `host`, `getent hosts`.

**`traceroute 8.8.8.8`** — shows the **hops** (routers) on the way to the destination. It works by incrementing the **TTL**: each router that receives a packet with TTL=1 discards it and replies with "time exceeded," thereby revealing its address. It is useful for locating the point at which a connection breaks down, as well as for mapping the network.

**Note:** behind NAT (as in VirtualBox), traceroute often shows `* * *` because intermediate hops do not respond — this is not an error.

1. Inspect the open and listening TCP/UDP ports on your system:
   ```bash
   ss -tuln
   # Alternative legacy command: netstat -tuln
   ```

2. Perform domain name resolution (DNS resolution) for a website:
   ```bash
   nslookup www.kernel.org
   ```

3. Trace the intermediate routing hops to the destination:
   ```bash
   traceroute 8.8.8.8
   ```

---

### Part D: File Permission Management (`chmod`)


### Explanation of Part D — the Unix permission model

Every file has an **owner**, a **group**, and **three user categories**, each with three permissions:

| Letter | Permission | On a file | On a directory |
| :--- | :--- | :--- | :--- |
| `r` (4) | read | read contents | list names (`ls`) |
| `w` (2) | write | modify | create/delete files within it |
| `x` (1) | execute | execute | **enter** the directory (`cd`) |

The digits are summed: `rwx = 7`, `rw- = 6`, `r-x = 5`, `r-- = 4`.

**This is how `755` is derived:**

```
7 5 5
│ │ └─ others: r-x  (can read and execute, cannot write)
│ └─── group : r-x
└───── owner : rwx  (full control)
```

**Why we do NOT use `chmod 777`:** it grants **write access to everyone**. Any user can replace the file with malicious content — a classic privilege escalation vector. You will see exactly this in Raven 2: `chmod 777 LinEnum.sh` is bad practice.

**SUID / SGID / sticky bit:**

| Bit | On a file | On a directory |
| :--- | :--- | :--- |
| **SUID** (`4xxx`) | executes with the **owner's** privileges | — |
| **SGID** (`2xxx`) | executes with the **group's** privileges | new files inherit the directory's group |
| **sticky** (`1xxx`) | — | only the owner can delete their own files (`/tmp`) |

SUID appears as `s` in place of `x`: `-rwsr-xr-x`. It is a primary vehicle for privilege escalation (DC-1: `/usr/bin/find` with SUID → root).

**The numeric forms you will encounter:**

| Value | Meaning | Use |
| :--- | :--- | :--- |
| `600` | `rw-------` | private files (SSH keys) |
| `644` | `rw-r--r--` | ordinary files, web content |
| `700` | `rwx------` | personal directories/scripts |
| `755` | `rwxr-xr-x` | executables, directories |
| `777` | `rwxrwxrwx` | **to be avoided** |
| `4755` | SUID + 755 | executable with root privileges |

#### Step 4.1: Modifying Execute Permissions


#### Explanation 4.1 — why `./script.sh` requires `+x`

When you type `script.sh`, the shell searches for a program named `script.sh` in `$PATH` (directories such as `/usr/bin`). The **current directory is not in `$PATH`** (for security reasons — otherwise a malicious `ls` in your downloads directory would be executed). This is why we type `./script.sh`: the `./` means "here, in this directory."

Without `x`, the system refuses with `Permission denied`, even if you own the file. The `x` permission is **permission to execute**, not a statement that the content is "correct."

**What `chmod +x` does compared with `chmod 755`:** `+x` **adds** the execute permission for everyone while leaving the remaining permissions unchanged (`644` → `755`). `755` **sets** the state absolutely — if the file was `600`, it becomes `755` and becomes readable by everyone.

**The first line `#!/bin/bash` (shebang):** tells the kernel which interpreter should execute the script. Without it, the system uses the current shell (`sh`), which may not support bash syntax.

**Checking before and after:**

```bash
ls -l script.sh      # -rw-r--r--  (644)
chmod +x script.sh
ls -l script.sh      # -rwxr-xr-x  (755)
```

1. Create a simple shell script file:
   ```bash
   echo 'echo "Lab Script Executed Successfully"' > script.sh
   ```

2. Check the initial access permissions:
   ```bash
   ls -l script.sh
   ```

3. Add execute permission (`+x`) and run the script:
   ```bash
   chmod +x script.sh
   ./script.sh
   ```

4. Set the permissions using the numeric (octal) form `755`:
   ```bash
   chmod 755 script.sh
   ```

> **Question 4.1:** Explain which permissions the value `755` grants to the owner, the group, and others.

---


## Appendix A — Answers to the Exercise Questions

**Question 1.1 — Difference between `cd ..` and `cd .`**

`cd ..` moves to the **parent** directory (one level up); that is, it changes the current working directory. `cd .` moves to the **same** directory (`.` = current), so in practice nothing changes — it is used in scripts to confirm that a directory exists and is accessible, not for navigation.

**Question 1.2 — Which `ls` combination finds the most recently modified file**

`ls -lt` (or `ls -ltr`). The `-t` option sorts by **modification time**, newest first; `-r` reverses the order so that the newest file appears at the bottom of the screen, next to the cursor, in long listings. Adding `-h` (`ls -ltrh`) makes the sizes human-readable.

**Question 2.1 — The role of `|` (pipe)**

The pipe connects the **standard output (stdout)** of the left-hand command to the **standard input (stdin)** of the right-hand command, without an intermediate file. Data flows **live** (streaming), enabling small tools to be composed into a single processing pipeline (e.g., `ps aux | grep sshd | wc -l`).

**Question 3.1 — The role of `lo` and its default address**

`lo` (*loopback*) is an internal virtual interface that allows processes on the **same** machine to communicate with one another through the TCP/IP stack, without a physical network. Default IPv4 address: **`127.0.0.1/8`** (and `::1` for IPv6). It is used for testing services and for isolation: a service listening only on `127.0.0.1` is not reachable from the network.

**Question 3.2 — Packet loss and average RTT**

These are read from the last two lines of the output:

```
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 0.043/0.055/0.062/0.008 ms
```

On a local network, **0% loss** and an **average RTT < 1 ms** are expected. High loss (> 0%) or an RTT of tens of ms on a LAN indicates a network problem or an overloaded VM.

**Question 4.1 — Breakdown of `755`**

| Category | Binary | Permissions | In practice |
| :--- | :--- | :--- | :--- |
| owner | 111 | `rwx` | reads, writes, executes |
| group | 101 | `r-x` | reads, executes — **no** write |
| others | 101 | `r-x` | reads, executes — **no** write |

This is the standard value for executables and directories: the owner maintains the file, while everyone else can only execute it.

## Appendix B — Glossary

| Term | Meaning |
| :--- | :--- |
| **Shell** | Command interpreter (bash, zsh, sh) |
| **stdin / stdout / stderr** | Streams 0 / 1 / 2 |
| **Pipe (`\|`)** | Connects stdout → stdin |
| **Redirect (`>`, `>>`, `2>`)** | Redirection of a stream to a file |
| **Wildcard / glob** | Pattern expansion performed by the shell (`*`, `?`, `[abc]`) |
| **FHS** | Filesystem Hierarchy Standard (the `/bin`, `/etc`, `/var` layout) |
| **Mount** | Attaching a filesystem to a directory |
| **Loopback** | Internal interface 127.0.0.1 |
| **TTL** | Time To Live — the hop limit of a packet |
| **DNS resolver** | The server that translates a name → IP address |
| **SUID** | Execution with the owner's privileges |
| **Umask** | Default permissions for new files (typically 022 → 644/755) |

## Appendix C — Skills Checklist

- [ ] I know where I am (`pwd`) and how to navigate using absolute/relative paths
- [ ] I can read an `ls -l` line and explain its 7 fields
- [ ] I use `grep` with `-i`, `-v`, `-c`, `-r` and combine it with pipes
- [ ] I can explain the difference between `>` and `>>` and why `2>/dev/null` hides errors
- [ ] I can find my IP address, run ping, and interpret loss/RTT
- [ ] I can distinguish "listening on 0.0.0.0" from "listening on 127.0.0.1"
- [ ] I can convert permissions from `rwx` to numeric form and vice versa
- [ ] I can explain why `777` is dangerous and what SUID means
