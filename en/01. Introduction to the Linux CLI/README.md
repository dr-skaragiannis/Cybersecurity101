## Lab Exercise 0: Introduction to the Command Line Interface (CLI) & Basic Networking Commands

### Learning Objectives

Upon completion of this lab session, students are expected to:

1. Become familiar with the command line interface (CLI) environment and basic command syntax.
2. Gain a thorough understanding of navigation using absolute and relative paths (`cd`, `cd ..`, `cd ../..`, `cd -`).
3. Apply advanced options and search patterns with the `ls` listing command (`-lh`, `-lt`, `-R`, `*`).
4. Apply output redirection and piping techniques.
5. Manage file access permissions using symbolic and numeric (octal) notation.
6. Inspect and analyze basic network configuration and connectivity parameters (`ifconfig`, `ip`, `ping`, `ss`, `nslookup`).

---

## Introduction to the CLI: Lab Structure & Directory Hierarchy Setup

Run the commands below. In addition, follow the guide to set up the VPN connection for HackTheBox: https://help.hackthebox.com/en/articles/5185687-introduction-to-lab-access
. Afterwards, you may also work through the lab on basic Linux commands: [https://ctf.hackthebox.com/pack/linux-fundamentals-essentials](https://academy.hackthebox.com/app/module/18)

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

#### Step 2.1: Using `grep` and Redirection

Search for processor information in the `/proc` virtual filesystem:
```bash
grep "model name" /proc/cpuinfo > cpu_spec.txt
head -n 2 cpu_spec.txt
```

#### Step 2.2: Pipelines

Count the number of active processes on the system by combining the `ps` command with the `wc` counting command:
```bash
ps aux | wc -l
```

> **Question 2.1:** What is the role of the pipe character `|` in the flow of data between two commands?

---

### Part C: Basic Network Diagnostic Commands

#### Step 3.1: Inspecting Network Interfaces (`ifconfig` / `ip`)

1. Display the available network interfaces and their assigned IP addresses:
   ```bash
   ifconfig
   ```
   *(If the `ifconfig` command is not installed on the system, run `ip addr show` instead.)*

2. Identify the IPv4 address (`inet`) of the primary interface (e.g., `eth0`, `enp3s0`, or `wlan0`), as well as that of the local loopback interface (`lo`).

> **Question 3.1:** What is the role of the `lo` (loopback) interface, and what is the default IPv4 address assigned to it?

#### Step 3.2: Testing Connectivity with the `ping` Command

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

#### Step 4.1: Modifying Execute Permissions

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
