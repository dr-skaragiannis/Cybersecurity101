# Linux for Beginners (Part 1): The Shell, Files, Text, Packages and Permissions

## Introduction

This guide is written for someone who has never opened a Linux terminal before and wants to become comfortable with the fundamentals in a single sitting. Rather than dumping a list of commands on you, every section explains *why* a command exists, *what* it does to your system, and *how* you can verify with your own eyes that it actually did it.

Every step follows the same rhythm: the **command** you type, the **output** it produced, then a short explanation of what just happened. Anything longer — extra flags, pitfalls, background theory — is tucked into a collapsible box, so you can read the main path first and open the detail only when you want it.

<details>
<summary><strong>Read more</strong> — the shape of the course and what comes next</summary>

The material follows a friendly difficulty curve. We begin with the two questions every beginner asks out loud ("where am I?" and "who am I?") and with the commands that move you around the file system and list what is inside it. From there we look at the built-in help systems, because no one memorises every flag of every utility, and learning to read a manual page is a skill that pays for itself immediately. Next comes file and directory manipulation, the everyday business of creating, copying, moving and deleting things; then text manipulation, which matters far more on Linux than on other systems because almost everything you administer here is a plain text file. The last two sections cover installing and removing software through the package manager and understanding the Unix permission model, the part of Linux that trips up beginners most often and the part that matters most when you later study privilege escalation.

This is **Part 1** of three. Everything here is the layer you need before anything else makes sense: the shell, the file system, text, packages and permissions. **Part 2** carries on with the topics that sit just beneath everyday use — networks and interfaces, process management, and the environment variables that quietly decide which program runs, which resolver answers and which editor opens — and **Part 3** finishes with scripting, scheduling and services, the automation layer that multiplies everything before it. Both assume only that you are comfortable with the material below.

The terminal contents are always written out as text rather than shown as a picture, so that you can copy and paste, scroll, search and compare them against your own results. If you follow along on your own machine, by the end of the guide you will have created, inspected, edited, re-permissioned and deleted a small set of files, understood a permission string character by character, and met the package manager — using nothing but the command line.

</details>

> **A word of caution before you start.** The examples below use `apt-get`, `chown`, `chmod`, `nano` on system paths, and several of them require root privileges. Practise inside a throwaway virtual machine (a Parrot or Kali image is ideal) and take a snapshot before you begin, so that a typo can never damage real data. Deleting files as root is permanent: Linux has no recycle bin on the command line, and reconfiguring the network or killing the wrong process can lock you out of the machine you are working on.

### How to read this guide

Every command is shown in its own box, exactly as you would type it (with no shell prompt), and the result it produces is shown in a separate **Output** box immediately after it. For example:

```bash
pwd
```

**Output:**

```text
/root
```

A few conventions run through the whole guide and are worth learning straight away:

* **A command box is what you type; an output box is what the machine prints back.** Nothing is executed until you press Enter, and a command that succeeds often prints nothing at all — on Linux, silence usually means success.
* **A command box may contain more than one line.** When several commands belong to a single idea they are grouped together; run them in order, one at a time, and read the output after each.
* **A `#` at the start of a line inside a configuration file is a comment, not a command.** (On many systems a `#` at the end of a *shell prompt* means something quite different — you are running as the administrator.)
* **Text inside angle brackets, for example `cp <source> <destination>`, is a placeholder** that you replace with a real path or value. Do not type the angle brackets themselves.
* **Output is representative, not identical.** Package versions, file sizes, timestamps and the number of matching lines will differ on your own machine. What matters is the shape of the output, not the exact digits.

> **About the root account.** Many examples run as `root`, the Linux administrator (user id `0`), which is why they are allowed to change system files. Where a prompt or an output line would end in `#` rather than `$`, the command ran with full privileges. That is powerful and dangerous in equal measure: a mistyped command as `root` can destroy a system, while the same command run as an ordinary user is simply refused.

### Table of contents

1. [Why Linux for security work](#1-why-linux-for-security-work)
2. [The terminal and your first commands](#2-the-terminal-and-your-first-commands)
3. [Everyday file and directory operations](#3-everyday-file-and-directory-operations)
4. [Text manipulation](#4-text-manipulation)
5. [Installing and removing software](#5-installing-and-removing-software)
6. [Understanding and playing with permissions](#6-understanding-and-playing-with-permissions)
7. [Quick reference cheat sheet](#7-quick-reference-cheat-sheet)
8. [Practice exercises](#8-practice-exercises)

---

## 1. Why Linux for security work

Three practical reasons explain why Linux is the platform security work happens on: it is **transparent** (its source code can be read), it has **tool gravity** (the security tools are written for it first), and it is **cheap to maintain** (updates and dependencies are handled by the system).

None of this means that Linux is magically secure. It means that Linux gives you the controls and the visibility to see what is happening on the system, and that is the foundation of every technique that follows.

<details>
<summary><strong>Read more</strong> — what each of those three reasons actually means</summary>

Because the source code of the kernel, the shell and virtually every utility is available for inspection, the system is **transparent**: if you want to know exactly how a network socket is created, how a password hash is compared or how a file handle is checked against a permission bit, you can read the code that does it. Before you try to break or defend anything, you have to understand how it works, and transparency is the single biggest advantage Linux gives you in that pursuit.

**Tooling gravity.** Because Linux is so popular in the penetration-testing and security-research community, the vast majority of security tools and frameworks are written for it first, and many are never ported anywhere else. Learning the command line is therefore not a detour around the tools — it is the road that leads to them, since most of them are invoked from a shell and compose with other shell utilities.

**Maintenance** is comparatively easy. Software is installed from curated repositories with a single command, updates are applied system-wide, and dependencies are resolved automatically, which is why an installed Linux server can be kept current for years with very little effort. Linux also tends to be very stable under the workload of long-running tools such as network scanners or brute-force utilities, which is exactly the kind of work you will be doing with it.

</details>

---

## 2. The terminal and your first commands

A terminal is the program that reads what you type, interprets it, and asks the operating system to perform the operation you requested. It is the command-line equivalent of the everyday graphical operations — creating folders, moving files, copying things around.

When you open a terminal you start a shell session in your **home directory**, and every command you run afterwards runs from some *current working directory*. Almost every confusion a beginner experiences traces back to not knowing which directory that is, so the first two commands answer the two most fundamental questions: *where am I* and *who am I*.

### 2.1 `pwd` — print working directory

`pwd` takes no arguments, prints a single absolute path, and exits. That path is the directory your next commands will operate in.

```bash
pwd
```

**Output:**

```text
/root
```

We are logged in as `root`, and `pwd` tells us the home directory of that account is `/root`. The output is a single line with no decoration — command-line output is meant to be read by other programs as much as by humans.

<details>
<summary><strong>Read more</strong> — the <code>~</code> shortcut and why it is everywhere</summary>

In the transcript the prompt says we are in `~`, and `pwd` confirms that `~` expands to `/root`. The tilde is a shorthand that the shell expands *before* the command ever sees it, so `cd ~` and `cd /root` are the same instruction when you are logged in as root. If you are logged in as a normal user called `student`, `pwd` will print `/home/student` instead, and the prompt's `~` will refer to that path.

A related habit: when you create a file without giving an explicit path, it is created in whatever directory `pwd` just printed.

</details>

### 2.2 `whoami` — identify the current user

The second question, *who am I*, is answered by `whoami`. Linux decides what you are allowed to read, write and execute based on the **identity of the process making the request**, not on where you are sitting or which desktop you logged in to.

```bash
whoami
```

**Output:**

```text
root
```

`root` is the local administrator account — the equivalent of "Administrator" on Windows. It is not subject to the ordinary permission checks, which is exactly why it is so useful and so dangerous.

The related command `id` prints the numeric ids the kernel actually uses when it evaluates permissions:

```bash
id
```

**Output:**

```text
uid=0(root) gid=0(root) groups=0(root)
```

User id `0` is always root; on Debian-based systems ordinary human accounts start at `1000`. Seeing a process running as `uid=0` that has no business doing so is the essence of a privilege-escalation bug.

<details>
<summary><strong>Read more</strong> — why identity matters more than location</summary>

A root command run from the terminal is the same operation as a root command run by a service at 3 a.m.: the kernel checks the uid and the group memberships, nothing else. That is why the same typo is harmless as an ordinary user and catastrophic as root, and why the very first habit in security work is to ask "which user am I running as?" before anything risky.

The `id` output also lists *supplementary* groups — every additional group the account belongs to. Access is often granted by adding a user to a group rather than by touching each file's permissions individually, so a strange entry in that list is worth investigating.

</details>

### 2.3 `cd` — changing directories

`cd` stands for *change directory* and moves your shell session from the current working directory to the path you pass as an argument. Here we move into the `Desktop/` directory inside our home folder:

```bash
cd Desktop/
```

Because `cd` prints nothing when it succeeds — on Linux, silence is success and the only immediate feedback is the change in your prompt — we run `pwd` to verify our new location:

```bash
pwd
```

**Output:**

```text
/root/Desktop
```

The move worked, and `pwd` proves that our working directory is now `/root/Desktop`. The trailing slash on `Desktop/` is optional and purely cosmetic; the shell treats `Desktop` and `Desktop/` identically.

`cd` also understands four special destination shortcuts that every Linux user memorises within a day. First, `..` always points to the parent directory one level up the tree:

```bash
cd ..
```

The shell silently moves us one level up from `/root/Desktop` back to `/root`. Next, a single forward slash `/` represents the root of the entire file system — the top-level directory from which every other directory hangs:

```bash
cd /
```

Again, `cd` succeeds silently and places us at the top of the directory tree (note that the `/` directory is different from `/root`, the administrator's personal home directory). To jump straight back to your home directory from anywhere on the system, pass the tilde shortcut `~` (or run `cd` with no arguments at all):

```bash
cd ~
```

The shell expands `~` to `/root` and returns us home without printing anything. Finally, a single dash `-` acts like a browser's back button and returns you to whichever directory you were in immediately before:

```bash
cd -
```

**Output:**

```text
/root/Desktop
```

Unlike the other forms of `cd`, `cd -` prints the path of the directory it moved you back to (`/root/Desktop` in the transcript), confirming where you landed. If you ever try to `cd` into a path that does not exist, the shell prints a clear error such as `bash: cd: Downloads/: No such file or directory`, naming both the command that failed and the exact argument it could not find.

### 2.4 `ls` — listing the contents of a directory

Where `pwd` tells you where you stand, `ls` tells you what is standing around you. It lists the contents of the current directory by default, or of any directory you name. Let us first make sure we are inside `Desktop/`:

```bash
cd Desktop/
```

With our working directory set to `Desktop/`, we run `ls` with no arguments to list the visible files and folders inside it:

```bash
ls
```

**Output:**

```text
practice-notes.txt  simple_bash.sh  tools
```

Three entries came back: two regular files and one directory, separated by whitespace and sorted alphabetically. On a modern terminal these names are colour-coded — directories blue, executables green — which gives you a quick visual summary before you read a single character.

The plain `ls` is readable but hides most of the useful information, so in practice you almost always reach for `-l`, the *long* format:

```bash
ls -l
```

**Output:**

```text
total 12
-rw-r--r-- 1 root root   96 Oct  8 11:42 practice-notes.txt
-rwxr-xr-x 1 root root  118 Oct  8 11:40 simple_bash.sh
drwxr-xr-x 2 root root 4096 Oct  7 19:05 tools
```

Every line has the same seven fields. From left to right: the permission string, the number of hard links, the owner, the owning group, the size in bytes, the last modification time, and the name. The first character of the permission string tells you what kind of object the entry is — a leading `-` means a regular file, a leading `d` means a directory. Section 6 is essentially a study of that first field.

Combining flags is normal, and `ls -lah` is a perfectly ordinary thing to type:

```bash
ls -lah
```

**Output:**

```text
total 12K
drwxr-xr-x 3 root root 4.0K Oct  8 11:44 .
drwx------ 1 root root 4.0K Oct  8 11:38 ..
-rw-r--r-- 1 root root   96 Oct  8 11:42 practice-notes.txt
-rwxr-xr-x 1 root root  118 Oct  8 11:40 simple_bash.sh
drwxr-xr-x 2 root root 4.0K Oct  7 19:05 tools
```

With `-a` the two hidden shortcuts appear at the top: `.` is the current directory and `..` is its parent, which is exactly what makes `cd ..` work. With `-h`, sizes are shown as `4.0K` rather than `4096`.

<details>
<summary><strong>Read more</strong> — the <code>ls</code> flags worth memorising</summary>

* `-a` also shows *hidden* entries — files and directories whose names begin with a dot, such as `.bashrc` and `.ssh`, which `ls` hides by default.
* `-h` prints sizes in human-readable units (`4.0K`, `2.3M`) instead of raw bytes.
* `-t` sorts by modification time with the newest first; `-S` sorts by size with the largest first.
* `-R` recurses into every subdirectory.

The order of flags rarely matters — `ls -lah` and `ls -alh` are the same command.

One trap for later: `total 12K` is the total size of the directory's *contents* in blocks, not the size of the directory itself. Directories are tiny bookkeeping records, and their reported size (often 4096 bytes) is an implementation detail you can safely ignore for now.

</details>

### 2.5 `--help` — the built-in quick reference

Nearly every utility on Linux ships with a short help text, and reaching for it should become a reflex rather than an admission of defeat. Almost every program answers to `-h` or `--help`.

```bash
volatility --help
```

**Output:**

```text
Volatility Foundation Volatility Framework 2.6
Usage: Volatility - A memory forensics analysis platform.

Options:
  -h, --help            list all available options and their default values.
                        Default values may be set in the configuration file
                        (/etc/volatilityrc)
  --conf-file=<file>    User based configuration file
  -d, --debug           Debug Volatility
  -i, --info            Print information about all registered objects
  -o, --output=<file>   Output in a specific file (default is stdout)
  ...
```

The first lines tell you what the tool is and which version you have; the block underneath lists every option with a one-line description. The exact content is not important here — **the habit is**. If you do not have this tool installed, `ls --help` or `nmap --help` demonstrate exactly the same thing on any system.

<details>
<summary><strong>Read more</strong> — two limitations of <code>--help</code> worth knowing</summary>

First, `--help` is not universal: some older or minimalist utilities only respond to `-h`, and a few ignore both, in which case `man` is your next stop.

Second, some tools print their help to *standard error* rather than *standard output*, which matters if you try to pipe it into another program and see nothing. Section 2.10 explains the difference between the two streams in detail.

Help output can easily be a hundred lines long, so `tool --help | less` is a perfectly good way to explore an unfamiliar program without losing the first half off the top of your screen.

</details>

### 2.6 `man` — the manual pages

The full reference for most commands is the manual page, which you open with `man` followed by the command name. Manual pages have a consistent structure: `NAME`, `SYNOPSIS` (the grammar of the command), `DESCRIPTION`, and usually `EXAMPLES`, `FILES`, `SEE ALSO` and `AUTHOR`.

```bash
man ls
```

**Output:**

```text
LS(1)                            User Commands                           LS(1)

NAME
       ls - list directory contents

SYNOPSIS
       ls [OPTION]... [FILE]...

DESCRIPTION
       List information about the FILEs (the current directory by default).
       Sort entries alphabetically if none of -cftuvSUX nor --sort is specified.

       Mandatory arguments to long options are mandatory for short options too.

       -a, --all
              do not ignore entries starting with .

       -A, --almost-all
              do not list implied . and ..

       -l     use a long listing format

 Manual page ls(1) line 1 (press h for help or q to quit)
```

That last line is the status bar of the *pager* that `man` opened. Press `q` to quit, the space bar to scroll down a screen, `b` to scroll back up, `/keyword` to search forward and `n` to jump to the next match. To discover a command whose name you do not yet know, `man -k <keyword>` searches the short description of every manual on the system.

<details>
<summary><strong>Read more</strong> — what the <code>(1)</code> means: manual sections</summary>

The `(1)` next to the name is the manual *section* number, and it tells you what kind of documentation you are reading:

* **1** — user commands (what you type)
* **5** — file formats and conventions (try `man 5 passwd` to see the structure of the password file rather than the command that edits it)
* **7** — miscellaneous topics, such as `man 7 capabilities`
* **8** — system administration commands

When the same name exists in several sections, `man` picks the lowest-numbered one by default, and you can ask for a specific section explicitly, for example `man 8 tcpdump`.

</details>

### 2.7 `locate` — searching for filenames by keyword

When you remember only part of a filename, `locate` is the fastest way to find it. You give it a keyword and it prints every path in its database that contains that string anywhere in the name — milliseconds, even on a machine with millions of files.

```bash
locate wordlists | more
```

**Output:**

```text
/usr/share/wordlists
/usr/share/wordlists/dirb
/usr/share/wordlists/dirb/big.txt
/usr/share/wordlists/dirb/common.txt
/usr/share/wordlists/rockyou.txt.gz
--More--
```

The pipe into `more` is a good habit: a keyword can match several hundred paths, and `more` gives you one screenful at a time. Note that `locate` matches the *string*, not just the filename, so a search for `passwd` will also return `/etc/passwd-` and `/usr/share/doc/passwd.html`.

<details>
<summary><strong>Read more</strong> — why <code>locate</code> sometimes lies to you</summary>

`locate` does not look at the file system at all. It looks at a pre-built index, usually `/var/lib/mlocate/mlocate.db`, which the operating system refreshes from a daily cron job. Two consequences follow.

First, a file you created five minutes ago will not appear until the index is rebuilt, which is why beginners sometimes run `locate` for their own file and conclude the command is broken. `sudo updatedb` refreshes the index on demand.

Second, the results are not filtered by permission: `locate` will happily print the path of a file you are not allowed to read. It is telling you where the name exists, not whether you can open it — a mild information leak on a shared machine.

</details>

### 2.8 `whereis` and `which` — finding the binaries behind a command

Let us begin with a definition. Files that can be *executed* — the rough equivalent of `.exe` files on Windows — are called *binaries* (or executables). On Linux they live in a small set of well-known directories such as `/usr/bin`, `/usr/sbin`, `/bin`, `/sbin` and `/usr/local/bin`.

`whereis` answers the question generously: it prints the path of the binary together with any manual pages and source files it can find for the same name.

```bash
whereis git
```

**Output:**

```text
git: /usr/bin/git /usr/share/man/man1/git.1.gz
```

The output shows two paths after `git:`: the executable binary itself at `/usr/bin/git` and its compressed Section 1 manual page at `/usr/share/man/man1/git.1.gz`.

`which` is stricter. It looks only at the directories listed in your `PATH` environment variable and returns the single binary the shell would actually run if you typed the bare command name:

```bash
which git
```

**Output:**

```text
/usr/bin/git
```

The single line `/usr/bin/git` tells us the exact executable that wins when we type `git`. That difference is why both commands exist, and why `which` is the more useful of the two in day-to-day work when several copies of a program are installed. To see the search path itself, we print the `$PATH` variable with `echo`:

```bash
echo $PATH
```

**Output:**

```text
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

The output is a colon-separated list of six directories that the shell searches from left to right whenever you type a command name.

A third command, `type`, is the shell's own built-in query and reveals whether a command name resolves to an external binary on disk, an alias, a function, or a shell built-in. Here we check `git` with `type`:

```bash
type git
```

**Output:**

```text
git is /usr/bin/git
```

The shell confirms that `git` is an external binary located at `/usr/bin/git`. Now let us run `type` against `cd` to see why `which cd` often produces nothing:

```bash
type cd
```

**Output:**

```text
cd is a shell builtin
```

The output tells us that `cd` is not a separate file on disk at all, but a command built directly into the shell itself (because only the shell process can change its own working directory).

<details>
<summary><strong>Read more</strong> — two surprises explained by <code>PATH</code></summary>

Because `/usr/local/bin` is searched before `/usr/bin`, a copy of a program you place in `/usr/local/bin` will be run instead of the system one. That is how you override a tool without touching the distribution's files.

The current directory is deliberately *not* on the search path — a safety measure, so that a malicious `ls` dropped into a shared folder cannot be run by accident. To run something from the current directory you must name it explicitly, for example `./simple_bash.sh`.

</details>

### 2.9 Filtering output with `grep`

Often you do not want *more* output, you want *less* of it — one particular line out of a hundred. `grep` reads text, either from a file you name or from its standard input, and prints only the lines matching the pattern you give it. Here we search `simple_bash.sh` for the string `echo`, passing `-i` to ignore case:

```bash
grep -i "echo" simple_bash.sh
```

**Output:**

```text
echo "Starting the backup job..."
echo "Backup finished at $(date)"
```

Only the two lines from `simple_bash.sh` that contain `echo` are printed, while every other line in the script is hidden. Because we used `-i`, a line containing `Echo` or `ECHO` would have matched too; without `-i`, `grep` is strictly case-sensitive.

By far the most common use of `grep` is not to search files directly, but to *pipe* another program's output into it using the vertical bar `|`. The `ifconfig` utility prints a great deal of detail about every network interface; here we pipe its output into `grep inet` to keep only the lines that show IP addresses:

```bash
ifconfig | grep inet
```

**Output:**

```text
        inet 10.0.2.15  netmask 255.255.255.0  broadcast 10.0.2.255
        inet6 fe80::a00:27ff:fed1:3c4a  prefixlen 64  scopeid 0x20<link>
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
```

The four surviving lines tell you everything that matters: the machine's IPv4 address (`10.0.2.15`, the classic NAT address of a virtual machine) with its netmask and broadcast address, its IPv6 link-local address (`fe80::...`), and the IPv4 (`127.0.0.1`) and IPv6 (`::1`) loopback addresses that the machine uses to refer to itself.

<details>
<summary><strong>Read more</strong> — how the pipe works and useful <code>grep</code> flags</summary>

The vertical bar `|` is the *pipe*: it takes whatever the program on its left wrote to standard output and feeds it to the program on its right as standard input. Nothing is written to disk in between; the two processes run at the same time and data flows through as it is produced.

Useful flags to combine with `grep`: `-n` prefixes every matching line with its line number, `-r` walks a whole directory tree recursively (for example, `grep -rn "password" /etc` reveals how many plaintext credentials a badly configured server accumulates), and `-v` *inverts* the match to print every line that does **not** contain the pattern. On modern systems `ip a` is the preferred replacement for the ageing `ifconfig`, with `ip a | grep inet` producing much the same result.

</details>

### 2.10 Searching the file system with `find`

`find` is the most powerful of the searching utilities, and the one that rewards a little study. Unlike `locate` it walks the file system live, examining every file as it goes, which lets it filter on far more than the name: creation or modification date, owner, group, permissions, size and type.

Its grammar is `find <where to look> <what to look for>`. Here we search from the root directory `/` for a regular file (`-type f`) named `practice-notes.txt` (`-name practice-notes.txt`):

```bash
find / -type f -name practice-notes.txt
```

**Output:**

```text
find: '/etc/ssl/private': Permission denied
find: '/proc/tty/driver': Permission denied
/home/student/notes/practice-notes.txt
find: '/var/cache/private': Permission denied
find: '/var/lib/private': Permission denied
```

The file was found at `/home/student/notes/practice-notes.txt` — but look at the surrounding lines. Because we asked `find` to search the whole machine as an unprivileged account, it also tried to enter restricted directories, and every such attempt produced a `Permission denied` message. Those two kinds of line travel on **different channels**: the actual results on standard output (file descriptor 1), and the errors on standard error (file descriptor 2).

Silencing the error channel without touching the results is what redirecting file descriptor `2` to `/dev/null` (`2>/dev/null`) does:

```bash
find / -type f -name practice-notes.txt 2>/dev/null
```

**Output:**

```text
/home/student/notes/practice-notes.txt
```

`/dev/null` is a special kernel device that discards everything written to it, so all `Permission denied` lines vanish and only the clean match `/home/student/notes/practice-notes.txt` remains.

With that redirection in place, we can stack multiple criteria in a single `find` command. Here we search `/home` for regular files (`-type f`) owned by the user `student` (`-user student`) that were modified within the last 7 days (`-mtime -7`):

```bash
find /home -type f -user student -mtime -7 2>/dev/null
```

**Output:**

```text
/home/student/notes/week-1.txt
/home/student/notes/week-2.txt
```

The output lists the two files under `/home/student/notes/` that match all three conditions simultaneously (`week-1.txt` and `week-2.txt`), demonstrating how `find` narrows thousands of files down to exact recent activity in seconds.

<details>
<summary><strong>Read more</strong> — standard output vs standard error, and the flags that make <code>find</code> dangerous</summary>

Because the two streams are separate, you can redirect them independently. A common alternative you will see in older tutorials is `2>&1 | grep -v "Permission denied"`, which first merges the errors into the normal output and then filters out the permission failures. Both produce a tidy transcript, but `2>/dev/null` is preferable: it does not depend on the exact wording of the error message, it does not hide genuine errors that are not permission problems, and it costs nothing.

Flags worth keeping in your notes:

* `-name` and its case-insensitive sibling `-iname` take wildcards such as `*.conf`
* `-size +10M` finds files larger than ten megabytes
* `-maxdepth 2` stops the walk from descending further than the second level
* `-perm -4000` finds files with the SUID bit set — we will use this in section 6

Finally, `find` can *do* things to what it finds, not merely print them, thanks to `-exec`: `find . -name "*.tmp" -exec rm {} \;` deletes every temporary file below the current directory. That is powerful enough to be genuinely dangerous, so test it first with `-print` in place of `-exec`.

</details>

---

## 3. Everyday file and directory operations

Now that we can move around, ask for help and search, it is time for the operations that make up most of the actual work: reading a file, creating one, making a directory, copying, moving, renaming and deleting. The concepts are simple, but type every one of them out rather than copying blindly — muscle memory for paths is half of what a Linux user is.

Let us first create a dedicated `linux-lab` directory inside `Documents/` for our examples using `mkdir`:

```bash
mkdir Documents/linux-lab
```

`mkdir` creates the new folder silently without printing any output. Next, we move into that new directory with `cd`:

```bash
cd Documents/linux-lab
```

Again, `cd` succeeds silently, so we run `pwd` to confirm we are standing inside our new workspace before creating or deleting files:

```bash
pwd
```

**Output:**

```text
/root/Documents/linux-lab
```

The output `/root/Documents/linux-lab` confirms that our working directory is now the isolated lab folder, so every relative filename we use below will live safely inside it.

### 3.1 `cat` — printing the contents of a file

`cat` prints the contents of a file to the terminal. Its name is short for *concatenate*, because given several files it prints them one after another, glued together.

```bash
cat practice-notes.txt
```

**Output:**

```text
Week 1: permissions - chmod 644 vs 755, SUID bit explained
Week 2: pipelines - grep, cut, sort, uniq, tee
Week 3: processes - ps aux, top, kill -9
```

The lines appear in the order they occur, exactly as stored. `cat` makes no attempt to format, number or page the output, which is precisely why it is useful inside pipelines and scripts — and precisely why it is the wrong tool for large files.

<details>
<summary><strong>Read more</strong> — when <code>cat</code> is the right tool and when it is not</summary>

For anything longer than a screenful, use `less` instead (section 4) or you will be scrolling back through it for the rest of the session.

`cat` becomes more interesting combined with redirection:

* `cat file1 file2 > combined` merges two files into a third
* `cat /etc/hostname` reads a system file directly
* `cat -n practice-notes.txt` numbers the lines as it prints them

A related utility, `tac`, prints a file *backwards*, last line first — surprisingly handy when reading log files where the newest entries are at the bottom.

</details>

### 3.2 `touch` — creating an empty file

`touch` with a filename that does not exist creates it, which is how it earned its reputation as a file-creation tool. Here we create a small document to practise on:

```bash
touch lab-notes.txt
```

```bash
ls -l
```

**Output:**

```text
total 4
-rw-r--r-- 1 root root 0 Oct  8 11:47 lab-notes.txt
-rw-r--r-- 1 root root 82 Oct  8 11:45 practice-notes.txt
```

The file appears immediately with a size of `0` bytes and today's date. The name is slightly misleading, though: the primary purpose of `touch` is *updating timestamps*.

<details>
<summary><strong>Read more</strong> — why timestamps matter, and the <code>umask</code></summary>

If you point `touch` at a file that already exists it changes nothing but the dates. That is genuinely useful — build systems such as `make` decide whether to recompile by comparing timestamps, so touching a source file is a quick way to force a rebuild.

`touch` also accepts several filenames at once and understands brace expansion, so `touch {1..5}-note.md` creates five files in one go. The expansion is performed by the *shell*, not by `touch`, which means the trick works with every command.

Finally, note the permissions on the new file: `-rw-r--r--`. We did not ask for that; the system applied it because of the `umask` setting, which we meet properly in section 6.4.

</details>

### 3.3 `mkdir` — creating a directory

`mkdir` takes the name of the directory you want. If the parent exists and the name is not taken, it creates the directory and says nothing — as usual, silence is success.

```bash
mkdir tools
```

```bash
ls -l
```

**Output:**

```text
total 8
-rw-r--r-- 1 root root   0 Oct  8 11:47 lab-notes.txt
-rw-r--r-- 1 root root  82 Oct  8 11:45 practice-notes.txt
drwxr-xr-x 2 root root 4096 Oct  8 11:49 tools
```

The leading `d` confirms it is a directory rather than a file. Two flags are worth knowing from day one: `-p` creates intermediate directories as needed, and `-v` reports what it created.

```bash
mkdir -pv tools/nmap/scans
```

**Output:**

```text
mkdir: created directory 'tools/nmap'
mkdir: created directory 'tools/nmap/scans'
```

<details>
<summary><strong>Read more</strong> — why <code>-p</code> is the default in scripts</summary>

`mkdir tools/nmap/scans` on its own would fail, because `tools/nmap` does not exist yet. With `-p` the whole chain is built in one command — and, importantly, `-p` also silently succeeds when the target already exists. That combination is exactly what you want inside a script that must be safe to run twice.

The `4096` next to the directory is not the space its contents occupy but the size of its own bookkeeping record, and it stays at that value no matter how many files you put inside. The `2` in the third column is the hard-link count, which for a directory equals two (its own entry plus its `.` entry) plus the number of subdirectories it contains; you can safely ignore it until you study file systems in detail.

</details>

### 3.4 `cp` — copying files

`cp` creates a duplicate of a file at the location you specify. The general form is `cp <file you want to copy> <destination>` — source first, destination second, always in that order.

```bash
cp lab-notes.txt tools/
```

```bash
ls -l tools/
```

**Output:**

```text
total 0
-rw-r--r-- 1 root root 0 Oct  8 11:51 lab-notes.txt
```

The original is untouched and a copy now exists inside `tools/`. We listed the destination to confirm, which is a habit worth keeping: `cp` prints nothing on success, so a mistyped destination sends the copy somewhere you never intended and you may never notice.

<details>
<summary><strong>Read more</strong> — the quiet-overwrite hazard and the flags that prevent it</summary>

The trailing slash on `tools/` tells `cp` the destination is a directory and the file goes inside it. Without a destination directory, `cp file1 file2` creates `file2` as a copy of `file1`, and it will happily *overwrite* an existing `file2` without so much as a warning. That quiet overwriting is the main hazard of `cp`, and it is why beginners are taught two flags very early:

* `-i` asks for confirmation before overwriting an existing target
* `-r` (or `-R`) copies directories *recursively* — required, because copying a directory means copying its contents, their contents, and so on

Combined, `cp -ri source-dir/ destination-dir/` copies a whole tree while prompting before it overwrites anything. A few more that come up regularly: `-v` prints each file as it is copied, `-p` preserves the original permissions, ownership and timestamps, `-u` copies only when the source is newer than the destination (a poor man's incremental backup), and `-a` is the archive mode that bundles `-r`, `-p` and more, the flag to use when you want an exact replica.

</details>

### 3.5 `mv` — moving and renaming files

`mv` moves a file to a different directory *and* renames it, because on Linux those are the same operation: you are changing the name by which an inode is reached, and moving within the same file system copies no data at all, it merely relinks.

```bash
mv tools/lab-notes.txt .
```

```bash
ls -l
```

**Output:**

```text
total 0
-rw-r--r-- 1 root root 0 Oct  8 11:47 lab-notes.txt
-rw-r--r-- 1 root root 82 Oct  8 11:45 practice-notes.txt
drwxr-xr-x 3 root root 4096 Oct  8 11:49 tools
```

The single dot is the shortcut for "the current directory", so the command reads "move `tools/lab-notes.txt` to here". The file is gone from `tools/` and sits in `linux-lab/` with its timestamp intact — the same inode, a new name in a new directory. Renaming uses the identical syntax: `mv lab-notes.txt linux-notes.txt`.

<details>
<summary><strong>Read more</strong> — why <code>mv</code> overwrites without asking</summary>

The command cannot tell whether you meant to rename or to overwrite, so `mv` overwrites an existing destination file without asking. For that reason `mv -i` — which adds a confirmation prompt — is worth adopting as a default, especially in destructive shell scripts.

`-n` is the opposite behaviour (never overwrite), and `-v` narrates what was moved, which is invaluable in a loop or a script where dozens of files move at once.

</details>

### 3.6 `rm` — removing files

`rm` removes a file by name. Here we delete the note we created a moment ago, and the following `ls` proves it is gone:

```bash
rm lab-notes.txt
```

```bash
ls
```

**Output:**

```text
practice-notes.txt  tools
```

There is no output on success and, more importantly, **there is no recycle bin**. The command unlinks the file's name from its inode, and once the last link is gone the data blocks are marked free for future use.

<details>
<summary><strong>Read more</strong> — the two defences every beginner needs</summary>

This is the single most common way beginners destroy their own work, and there are only two sane defences: always think before pressing Enter on `rm`, and make sure you know which directory you are standing in when you use wildcards, because `rm *` deletes everything in the current directory and the shell expands the `*` before `rm` ever sees it.

A decade of catastrophes has been caused by `rm -rf /` and by its subtler cousin `rm -rf /etc` typed from an unexpected prompt, so it is worth internalising the rule that the `#` in your prompt means "everything you say here is final".

The flags that matter: `-i` asks for confirmation for each file (excellent training wheels — set it as an alias while you are learning), `-f` forces deletion without prompting and suppresses the "no such file" error, `-v` gives a running commentary, and `-r` is required to delete a directory and everything inside it.

```bash
rm -r tools/
```

```bash
ls
```

**Output:**

```text
practice-notes.txt
```

That last command removed `tools/` and all of its contents with no confirmation whatsoever — which is exactly the behaviour auditors want out of your muscle memory.

</details>

### 3.7 `rmdir` — removing a directory

`rmdir` removes a directory, and it is deliberately cautious: it will only delete one that is **empty**.

```bash
mkdir old-backups
```

```bash
rmdir old-backups/
```

```bash
ls
```

**Output:**

```text
practice-notes.txt
```

Had the directory contained anything, we would have seen `rmdir: failed to remove 'old-backups/': Directory not empty` and it would have survived untouched — exactly the behaviour you want while you are still learning to type paths correctly.

<details>
<summary><strong>Read more</strong> — destructive commands and the preview habit</summary>

For directories that do have content, the tool is `rm -r`, where the `r` stands for *recursive* and instructs the command to descend and delete everything it finds. `rm -ri` combines recursion with a confirmation prompt, and `rm -rf` combines recursion with force — the combination to be genuinely careful with, since it will delete hundreds of files without a single question and will not even complain about the ones that were not there in the first place.

One general warning that applies to every destructive command in this section: the shell expands wildcards, resolves `..` and performs substitutions *before* `rm`, `mv` or `cp` ever run, so a mistyped space can change the meaning of a command completely — `rm file.txt` is a harmless typo, while `rm file .txt` is the deletion of two different things, and `rm -r ./dir /` is a typo that ends careers.

When a command is destructive, print the arguments first: run `echo rm -r ./dir` to see exactly what the shell intends to hand over, then remove the `echo` and press Enter only once the line reads correctly.

</details>

---

## 4. Text manipulation

On Linux almost everything you deal with is a **file**, and more often than not a *plain text* file: configuration lives in text, logs are text, user and group databases are text, the tables used by networking tools are text. Learning how to read, slice, number, search and modify text is therefore essential.

To have something realistic to work with, let us create a small mapping file using a *here-document* — a way of feeding several lines of text straight into a command's standard input. The `<<'EOF'` says "everything until the line `EOF` is input", and the quotes prevent the shell from expanding anything inside.

```bash
cat > dns-mappings.txt <<'EOF'
# DNS mapping file used by local network analysis tools
# Format: <name> <address> <comment>
www.example.com     192.168.1.10    # primary web server
ftp.example.com     192.168.1.11    # file server
mail.example.com    192.168.1.12    # smtp relay
ns1.example.com     192.168.1.13    # primary nameserver
ns2.example.com     192.168.1.14    # secondary nameserver
vpn.example.com     192.168.1.20    # remote access gateway
git.example.com     192.168.1.21    # code repository
cloud.example.com   192.168.1.22    # storage endpoint
lab.example.com     192.168.1.30    # test environment
# end of file
EOF
```

```bash
wc -l dns-mappings.txt
```

**Output:**

```text
12 dns-mappings.txt
```

Twelve lines, which we will need to know in a moment. This is the standard way to create a text file from the command line without opening an editor, and it serves well for small scripts and configuration stubs.

### 4.1 `head` — grabbing the beginning of a file

`head` displays the first **ten** lines of whatever you point it at. That default is one of the most useful inventions in the toolbox, because configuration files and logs frequently begin with a comment block explaining the format of everything below it.

```bash
head dns-mappings.txt
```

**Output:**

```text
# DNS mapping file used by local network analysis tools
# Format: <name> <address> <comment>
www.example.com     192.168.1.10    # primary web server
ftp.example.com     192.168.1.11    # file server
mail.example.com    192.168.1.12    # smtp relay
ns1.example.com     192.168.1.13    # primary nameserver
ns2.example.com     192.168.1.14    # secondary nameserver
vpn.example.com     192.168.1.20    # remote access gateway
git.example.com     192.168.1.21    # code repository
cloud.example.com   192.168.1.22    # storage endpoint
```

Ten lines came back and the last two of the file were dropped — that is the entire contract of the command.

<details>
<summary><strong>Read more</strong> — the <code>-n</code> flag and how <code>head</code> is usually really used</summary>

`head -3 dns-mappings.txt` prints only the three lines you asked for, and `head -n -4` prints everything *except* the last four lines, which is occasionally the easiest way to strip a trailing trailer.

`head` also accepts multiple filenames and labels each block of output with the name it came from, but it is most often used with pipes rather than files: `ps aux | head` gives you the header line plus the first few processes instead of several hundred, and `du -h * | sort -h | tail` uses the mirror-image command to show the largest items in a directory.

</details>

### 4.2 `tail` — grabbing the end of a file

`tail` is the mirror of `head`: the **last** lines of a file, ten by default. For log files this is the more useful of the pair, because whatever is happening now is written at the bottom.

```bash
tail dns-mappings.txt
```

**Output:**

```text
ftp.example.com     192.168.1.11    # file server
mail.example.com    192.168.1.12    # smtp relay
ns1.example.com     192.168.1.13    # primary nameserver
ns2.example.com     192.168.1.14    # secondary nameserver
vpn.example.com     192.168.1.20    # remote access gateway
git.example.com     192.168.1.21    # code repository
cloud.example.com   192.168.1.22    # storage endpoint
lab.example.com     192.168.1.30    # test environment
# end of file
```

Because the file has twelve lines, the last ten overlap with the output of `head` and the header comments are no longer visible.

<details>
<summary><strong>Read more</strong> — <code>tail -f</code>: watching a log live</summary>

`tail -n 3` gives you just the final three lines, and `tail -n +3` gives you everything *from* line three onwards, which is a handy way to skip a two-line header when feeding a file to another program.

The genuinely magical flag is `-f`, for *follow*. It does not exit after printing; it keeps the file open and prints new lines as they are appended, turning it into a live monitor:

```bash
tail -f /var/log/syslog
```

**Output:**

```text
Oct  8 11:52:31 kali systemd[1]: Started Session 4 of user root.
Oct  8 11:52:44 kali kernel: [ 4312.117834] usb 1-1: new high-speed USB device number 3
^C
```

The terminal sits there printing as the system logs events — in a second window you can plug in a USB stick and watch the kernel announce it here. Press `Ctrl+C` to get your prompt back; the `^C` in the transcript is the shell showing the interrupt you sent. This is exactly how you watch an authentication log while testing whether a login attempt succeeds, and `journalctl -f` does the same for the system journal on a systemd machine.

</details>

### 4.3 `nl` — numbering the lines

`nl` prints a file with its lines numbered.

```bash
nl dns-mappings.txt
```

**Output:**

```text
     1  # DNS mapping file used by local network analysis tools
     2  # Format: <name> <address> <comment>
     3  www.example.com     192.168.1.10    # primary web server
     4  ftp.example.com     192.168.1.11    # file server
     5  mail.example.com    192.168.1.12    # smtp relay
     6  ns1.example.com     192.168.1.13    # primary nameserver
     7  ns2.example.com     192.168.1.14    # secondary nameserver
     8  vpn.example.com     192.168.1.20    # remote access gateway
     9  git.example.com     192.168.1.21    # code repository
    10  cloud.example.com   192.168.1.22    # storage endpoint
    11  lab.example.com     192.168.1.30    # test environment
    12  # end of file
```

The numbers are not decoration: they are the format most compilers and interpreters use when reporting an error, so numbering a file yourself is how you make sense of a message that says *line 41: syntax error*.

<details>
<summary><strong>Read more</strong> — <code>nl</code> vs <code>cat -n</code> vs <code>grep -n</code></summary>

Two details distinguish `nl` from its simpler cousin `cat -n`. First, `nl` does not number *blank* lines by default, so a file with paragraph breaks will skip numbers at those points; `-ba` forces every line to be numbered. Second, `-s` changes the separator between number and text (`-s": "` produces `1: www.example.com ...`) and `-w` changes the width of the number field, which is what you need when a file has more than 99999 lines.

`grep -n` is a third way to get numbers, and you will probably reach for it more often than either, because it numbers only the lines that match your pattern.

</details>

### 4.4 `sed` — finding and replacing text

`sed` — the *stream editor* — searches for a word or a pattern and applies an action to it, without opening an editor. It reads its input line by line and applies your instructions as the text flows past, which is why it works perfectly inside pipelines. The most common instruction is **substitution**: `s/<what to find>/<what to put there>/`.

```bash
sed 's/example.com/example.net/g' dns-mappings.txt | head -5
```

**Output:**

```text
# DNS mapping file used by local network analysis tools
# Format: <name> <address> <comment>
www.example.net     192.168.1.10    # primary web server
ftp.example.net     192.168.1.11    # file server
mail.example.net    192.168.1.12    # smtp relay
```

The `s` is *substitute* and the `g` is *global* — every occurrence on the line, not just the first. Crucially, the change appears in the output while **the file on disk is untouched**: `sed` prints the modified stream and never writes to the input file by default, which makes it safe to experiment with.

To make the change permanent you add `-i`, for *in place*, and the careful way to use it includes a backup suffix:

```bash
sed -i.bak 's/example.com/example.net/g' dns-mappings.txt
```

```bash
ls
```

**Output:**

```text
dns-mappings.txt  dns-mappings.txt.bak
```

```bash
head -3 dns-mappings.txt
```

**Output:**

```text
# DNS mapping file used by local network analysis tools
# Format: <name> <address> <comment>
www.example.net     192.168.1.10    # primary web server
```

With `-i.bak`, `sed` renames the original to `dns-mappings.txt.bak` and writes the edited version under the original name, so a mistake is one `mv` away from being undone.

<details>
<summary><strong>Read more</strong> — the other <code>sed</code> forms, and the regex trap</summary>

Used bare, `sed -i` keeps no copy at all and its edits are as final as an `rm` — inside a script run as root this is a classic way to lock yourself out of a machine by editing the wrong line of `/etc/fstab` or `/etc/ssh/sshd_config`.

Forms worth filing away:

* an address prefix restricts the edit to a range of lines: `sed '5,10 s/192\.168/10.0/' file`
* `-n '5p'` suppresses the default printing and shows only line five
* `'/^#/d'` deletes every line beginning with a hash, stripping comments in one stroke
* the delimiter after `s` is your choice, so `sed 's|/usr/bin|/usr/local/bin|g'` substitutes paths containing slashes without escaping them

And one trap: `sed` treats several characters specially. The dot in `example.com` matches *any* character, which is why the careful form writes it `example\.com`. That detail is the doorway into regular expressions, which every serious Linux user eventually learns.

</details>

### 4.5 `more` — controlling the display of a file

`more` displays one page of a file at a time. It exists because a terminal only shows a fixed number of rows, and dumping four thousand lines of configuration at it is not reading, it is scrolling in reverse.

```bash
more dns-mappings.txt
```

**Output:**

```text
# DNS mapping file used by local network analysis tools
# Format: <name> <address> <comment>
www.example.com     192.168.1.10    # primary web server
ftp.example.com     192.168.1.11    # file server
mail.example.com    192.168.1.12    # smtp relay
ns1.example.com     192.168.1.13    # primary nameserver
ns2.example.com     192.168.1.14    # secondary nameserver
vpn.example.com     192.168.1.20    # remote access gateway
git.example.com     192.168.1.21    # code repository
cloud.example.com   192.168.1.22    # storage endpoint
--More--(58%)
```

The prompt at the bottom left is the pager's control: `Enter` advances one line, the space bar a full screen, and the percentage tells you how far down the file you are. Type `q` to quit.

<details>
<summary><strong>Read more</strong> — why almost everyone switches to <code>less</code> within a week</summary>

`/pattern` jumps forward to a match, but `more`'s searching is famously limited — it cannot search backwards, and it cannot re-search the same pattern easily. That limitation is precisely why the next command exists.

</details>

### 4.6 `less` — displaying and filtering a file

`less` is very similar to `more` — the name is a deliberate joke — but it can move *backwards* as well as forwards, and it can search interactively. In practice `less` is what you should be using whenever you would have reached for `more`.

```bash
less /etc/ssh/sshd_config
```

**Output:**

```text
#	$OpenBSD: sshd_config,v 1.100 2016/08/15 12:32:04 naddy Exp $
# This is the sshd server system-wide configuration file.  See
# sshd_config(5) for more information.
...
PermitRootLogin prohibit-password
```

Press the forward slash, type the keyword, press Enter: here we searched for `PermitRootLogin` and jumped straight to the line that decides whether root may log in over SSH. Press `n` for the next occurrence and `N` for the previous one.

<details>
<summary><strong>Read more</strong> — the keys worth memorising, and why <code>less</code> beats <code>cat</code></summary>

The keys worth memorising: `G` jumps to the very end of the file, `g` back to the beginning, `q` quits, and the arrow or `j`/`k` keys move a line at a time.

A default configuration file usually mentions the same option twice — once commented out as documentation, once as the actual setting — which is why jumping between matches is a habit worth developing.

`less` also reads from a pipe, and that second ability is what makes it a general-purpose window onto command output: `history | less`, `ls -lR /usr/share | less` and `dmesg | less` are all everyday commands. Piping into `less` does **not** lose anything: the whole output is captured in its buffer, so you can search backwards through the complete result of a slow command rather than only the screenful it printed first.

</details>

### 4.7 A few more text utilities worth knowing

Three commands round out the set: `wc` counts lines, words and characters (`wc -l` answers "how many entries does this list have?"), while `sort` arranges lines and `uniq` collapses adjacent duplicates. Together with `cut`, they turn a messy stream into a summarised one:

```bash
cut -d' ' -f1 dns-mappings.txt | grep -v '^#' | sort
```

**Output:**

```text
cloud.example.com
ftp.example.com
git.example.com
lab.example.com
mail.example.com
ns1.example.com
ns2.example.com
vpn.example.com
www.example.com
```

`cut -d' ' -f1` takes the first whitespace-separated field of every line, `grep -v '^#'` throws away the comment lines, and `sort` puts the remaining hostnames in alphabetical order.

<details>
<summary><strong>Read more</strong> — the filter-chain habit, and <code>tee</code></summary>

That *filter chain* is the characteristic way Linux users work: rather than a single tool with fifty options, each tiny tool does one thing well and the pipe glues them together. The same three commands against `/etc/passwd` give you an alphabetical list of every username on the machine.

`tee`, finally, is for when you want to see output *and* save it at the same time: `ifconfig | tee /tmp/net.txt | grep inet` writes the full command output to a file while the filtered part continues down the pipeline to your screen.

</details>

---

## 5. Installing and removing software

On Debian-based distributions such as Debian, Ubuntu, Kali and Parrot, the software manager is the **Advanced Package Tool**, or `apt`. The mental model is the one you already have from a phone: instead of downloading installers from random websites, you have **repositories** — servers holding a curated catalogue of software, already compiled and packaged — and a tool that searches it, downloads from it, verifies packages and resolves their dependencies.

### 5.1 `apt-cache search` — searching the repository for a package

Before installing anything, it is worth checking whether it is available and under which name. The catalogue of every package your machine knows about is stored locally, and `apt-cache` queries it.

```bash
apt-cache search hydra
```

**Output:**

```text
hydra - very fast network logon cracker
hydra-gtk - GTK+ based tool to do fast network logon cracking
libhydra0 - Hydra toolkit library
...
```

Several packages matched, which is typical: the program, its graphical front end, and a support library. Each line is the package name, a dash, and a short description — and that description is exactly what the search matched against, which is why searching for a *function* ("password cracker") can turn up software whose name you did not know.

<details>
<summary><strong>Read more</strong> — inspecting a package before you trust it</summary>

This command changes nothing on your system: it is a read-only query against the local copy of the catalogue, so it is always safe to run.

To see much more about a candidate before installing it, `apt-cache show hydra` prints the version, the download size, the dependencies, the maintainer and the long description — or the friendlier `apt show hydra`, which is the modern spelling of the same thing.

</details>

### 5.2 `apt-get install` — installing packages

`apt-get` followed by `install` and a package name installs it. Here we install `git`, which later lets us pull repositories down to install further tools:

```bash
apt-get install git
```

**Output:**

```text
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following additional packages will be installed:
  git-man liberror-perl
Suggested packages:
  git-daemon-run | git-daemon-sysvinit git-doc git-email git-gui gitk
The following NEW packages will be installed:
  git git-man liberror-perl
0 upgraded, 3 newly installed, 0 to remove and 0 not upgraded.
Need to get 5,608 kB of archives.
After this operation, 38.9 MB of additional disk space will be used.
Do you want to continue? [Y/n] y
Get:1 http://ftp.debian.org/debian stable/main amd64 liberror-perl all 0.17029-1 [31.0 kB]
Get:2 http://ftp.debian.org/debian stable/main amd64 git-man all 1:2.39.2-1.1 [2,150 kB]
Get:3 http://ftp.debian.org/debian stable/main amd64 git amd64 1:2.39.2-1.1 [3,427 kB]
Fetched 5,608 kB in 1s (4,318 kB/s)
Selecting previously unselected package liberror-perl.
(Reading database ... 145321 files and directories currently installed.)
Preparing to unpack .../liberror-perl_0.17029-1_all.deb ...
Unpacking liberror-perl (0.17029-1) ...
Setting up git (1:2.39.2-1.1) ...
```

`apt` is unusually honest: it tells you what it intends to do *before* doing it. It lists the **additional packages** it must install alongside your request — the *dependencies* — reports how much will be downloaded and how much disk it will use, and only then asks `Do you want to continue? [Y/n]`. Any answer other than `y`, `Y` or Enter aborts harmlessly.

<details>
<summary><strong>Read more</strong> — the three flags that make <code>install</code> safer</summary>

* `-y` pre-answers "yes" to the confirmation prompt. Essential inside scripts, but it removes your last chance to read the plan — so get into the habit of running a new command *without* `-y` the first time.
* `--no-install-recommends` installs only what is strictly required and skips the "recommended but not essential" extras, which keeps a container or a virtual machine lean.
* `apt-get -f install` (or `apt --fix-broken install`) repairs a half-configured state. When you see `E: Unmet dependencies. Try 'apt-get -f install'`, that error message is practically an instruction manual.

</details>

### 5.3 `apt-get remove` — removing packages

`remove` takes the package name back off the system. Here we ask to remove the package we just installed:

```bash
apt-get remove git
```

**Output:**

```text
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following packages will be REMOVED:
  git
0 upgraded, 0 newly installed, 1 to remove and 0 not upgraded.
After this operation, 38.9 MB disk space will be freed.
Do you want to continue? [Y/n] n
Abort.
```

Note what the plan says — and what it does *not* say. Only `git` itself will go; the dependencies installed with it stay behind, on the sensible assumption that something else might need them. This is why removing software on Linux does not make the system leaner in the way beginners expect: the program goes, but its libraries and, crucially, its **configuration files** remain.

<details>
<summary><strong>Read more</strong> — previewing a removal without doing it</summary>

For the sake of this walkthrough the step was aborted by answering `n`. As the transcript shows, `apt-get` takes you at your word and stops immediately — which makes answering `n` a safe way to preview exactly what a removal would do without doing it.

</details>

### 5.4 `apt-get purge` — removing packages completely

`purge` removes the package *and* the configuration files that `remove` deliberately spared.

```bash
apt-get purge git
```

**Output:**

```text
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following packages will be REMOVED:
  git*
0 upgraded, 0 newly installed, 1 to remove and 0 not upgraded.
After this operation, 38.9 MB disk space will be freed.
Do you want to continue? [Y/n]
```

The star after the package name is `apt`'s way of signalling that configuration files are included in the removal this time.

<details>
<summary><strong>Read more</strong> — why that star matters, and the two clean-up commands</summary>

That difference matters more than it sounds: a `purge` followed by a fresh `install` gives you a genuinely default configuration, whereas a `remove` followed by `install` gives you your old settings back — which is either exactly what you want or exactly the source of an afternoon of confusion, depending on the day.

Two related commands complete the set: `apt-get autoremove` deletes the dependencies that were pulled in for software you have since removed and that nothing else needs any more, and `apt-get clean` empties the downloaded `.deb` files out of the package cache under `/var/cache/apt/archives`, which on a long-lived machine can be worth hundreds of megabytes.

</details>

### 5.5 `apt-get update` — refreshing the catalogue

The repositories gain new software constantly, and your machine has to be told about it. `update` is the command that asks.

```bash
apt-get update
```

**Output:**

```text
Hit:1 http://kali.download/kali kali-rolling InRelease
Get:2 http://kali.download/kali kali-rolling/main amd64 Packages [19.6 MB]
Get:3 http://kali.download/kali kali-rolling/main amd64 Contents (deb) [45.1 MB]
Fetched 64.7 MB in 12s (5,392 kB/s)
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
```

**`update` does not install anything at all.** It re-downloads the *lists* — which packages exist, in which versions, with which dependencies — so that your machine's picture of the repositories matches reality. `Hit` means the file was already current; `Get` means new data was fetched.

<details>
<summary><strong>Read more</strong> — why <code>update</code> comes first in every procedure</summary>

Without a fresh catalogue, `apt-get install` may try to fetch a version that has since been replaced, and you get the notorious `404 Not Found` from the mirror. That is why `update` is the first step of *every* recommended installation procedure.

Pair it with the next command for the standard one-liner `apt-get update && apt-get upgrade`, in which `&&` means "run the second command only if the first one succeeded".

</details>

### 5.6 `apt-get upgrade` — applying available updates

`upgrade` installs the newer versions of everything already on the system.

```bash
apt-get upgrade
```

**Output:**

```text
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
Calculating upgrade... Done
The following packages will be upgraded:
  libc6  openssl  python3  sudo  tzdata  vim
6 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
Need to get 12.4 MB of archives.
After this operation, 1,024 kB of additional disk space will be used.
Do you want to continue? [Y/n] y
...
Setting up openssl (3.0.11-1~deb12u2) ...
Setting up sudo (1.9.13p3-1+deb12u1) ...
Processing triggers for libc-bin (2.36-9+deb12u3) ...
```

The plan lists six *upgrades*, not installs, because `upgrade` never removes a package and never adds a new one — it only replaces what is already there with a newer version. Note the entries: `libc6`, `openssl` and `sudo` are precisely the packages a security practitioner cares about, because an out-of-date crypto library or authentication tool is a finding waiting to be written up.

<details>
<summary><strong>Read more</strong> — <code>dist-upgrade</code>, "kept back" packages, and one golden rule</summary>

`apt-get dist-upgrade` (or the modern `apt full-upgrade`) is allowed to *remove* and *add* packages as required to complete a complex upgrade. `upgrade`'s refusal to do so is exactly why it occasionally reports that some packages are "kept back".

Upgrading can be time-consuming, so plan for it rather than starting it five minutes before you need the machine. When the kernel itself is upgraded, the new version only takes effect after a reboot.

And one caution that belongs in any discussion of updating: **never upgrade a machine you are in the middle of breaking into or assessing.** A package upgrade can restart services, change a library version a tool depends on, or overwrite a configuration file you modified — all fine on your own workstation, catastrophic mid-engagement. Snapshot the target, or work on a copy, before you touch the package manager.

</details>

### 5.7 Adding repositories to `/etc/apt/sources.list`

The list of repositories your machine is allowed to use lives in a plain text file at `/etc/apt/sources.list`, with extra entries in `/etc/apt/sources.list.d/`. You edit it with a text editor — `nano` is the beginner-friendly one.

```bash
nano /etc/apt/sources.list
```

Inside the editor you would see something like the following, one repository per line: the archive type (`deb` for binaries, `deb-src` for source code), the URL of the mirror, the distribution name and the components.

```
deb http://kali.download/kali kali-rolling main contrib non-free non-free-firmware
# deb-src http://kali.download/kali kali-rolling main contrib non-free
```

<details>
<summary><strong>Read more</strong> — working in <code>nano</code>, and the warning that goes with this file</summary>

Working in `nano` is deliberately simple: type your text directly, use the arrow keys to move around, and use the shortcut bar along the bottom for everything else. `Ctrl+O` then Enter saves, `Ctrl+W` searches (the fastest way to find the right line in a long file), `Ctrl+K` cuts the current line and `Ctrl+U` pastes it back, and `Ctrl+X` leaves the editor, asking whether to save if you have unsaved changes.

After the edit you must run `apt-get update` again, because the package lists are built from the lines you just changed.

A word of warning: add only repositories you actually trust. An experimental or third-party repository can push packages that conflict with your distribution's own, and a mistyped URL pointing at a stranger's mirror is a straightforward way to install someone else's code as root. One incorrect line in this file is also enough to make `apt-get update` fail with a GPG or `404` error, so when an update does go wrong, read the message carefully and check the line number it names.

</details>

---

## 6. Understanding and playing with permissions

Linux takes permissions far more seriously than a desktop operating system does, and the model is small enough to learn in one sitting.

Every file and directory has an **owner** (a user) and a **group**, and every access check the kernel performs is a question about the identity of the process asking. Permissions are granted at three levels — the **owner**, the **group**, and **everyone else** (often called *others* or *the world*) — and at each level three rights can be granted or withheld:

* **read (`r`)** — open and view the contents. On a *directory*, read means the ability to list the names of the entries inside it.
* **write (`w`)** — modify the file (or, for a directory, create, rename and delete entries inside it).
* **execute (`x`)** — run the file as a program. On a *directory*, `x` means the right to *traverse* it — to pass through and reach files below it, which is why "I can list the directory but I cannot `cd` into it" is a missing `x` rather than a missing `r`.

<details>
<summary><strong>Read more</strong> — why root is exempt, and how groups are used in practice</summary>

The **root** user is all-powerful: the kernel skips the ordinary permission checks for it entirely, which is why it can read any file on the system. Every other user has limited capabilities.

Users are usually collected into **groups** that share a function — a group for the developers, one for the deployment team, one for the administrators — so that different levels of access can be granted to each without touching individual files. Access is then granted by adding a user to a group, which is why the membership list of an account is worth auditing.

</details>

### 6.1 Reading a permission string with `ls -l`

The long listing is how you inspect these settings, so let us look at one again now that we know what we are looking for:

```bash
ls -l
```

**Output:**

```text
total 12
-rw-r--r-- 1 root root   82 Oct  8 11:45 practice-notes.txt
-rwxr-xr-x 1 root root 4096 Oct  8 12:02 backup.sh
drwxr-xr-x 2 root root 4096 Oct  8 12:01 documents
```

Take the second line apart character by character:

```
-  rwx  r-x  r-x   1  root  root  4096  Oct  8 12:02  backup.sh
│   │    │    │    │   │     │     │        │            │
│   │    │    │    │   │     │     │        │            └─ name of the file
│   │    │    │    │   │     │     │        └────────────── last modified time
│   │    │    │    │   │     │     └─────────────────────── size in bytes
│   │    │    │    │   │     └───────────────────────────── owning group
│   │    │    │    │   └─────────────────────────────────── owner (user)
│   │    │    │    └─────────────────────────────────────── number of hard links
│   │    │    └──────────────────────────────────────────── permissions for others
│   │    └───────────────────────────────────────────────── permissions for the group
│   └────────────────────────────────────────────────────── permissions for the owner
└────────────────────────────────────────────────────────── type: - file, d directory, l symlink
```

So `backup.sh` is readable, writable and executable by its owner (`rwx`), readable and executable by members of the owning group (`r-x`), and the same for everyone else (`r-x`) — the normal shape for a script everyone on the machine should be able to run.

<details>
<summary><strong>Read more</strong> — reading the other two lines</summary>

`practice-notes.txt` is `-rw-r--r--`: the owner can read and write, but nobody, including the owner, can execute it. It is a text file, and there is no reason for it to be executable at all.

The directory `documents` shows the leading `d`, and directories almost always carry `x` for everyone, because without it nobody could descend into them.

</details>

### 6.2 `chown` — granting ownership to an individual user

`chown <new owner> <file>` changes who owns a file, and therefore who the `rwx` triad is describing. Here we transfer our notes file from root to a normal user called `student`:

```bash
ls -l practice-notes.txt
```

**Output:**

```text
-rw-r--r-- 1 root root 82 Oct  8 11:45 practice-notes.txt
```

```bash
chown student practice-notes.txt
```

```bash
ls -l practice-notes.txt
```

**Output:**

```text
-rw-r--r-- 1 student root 82 Oct  8 11:45 practice-notes.txt
```

The third column changed from `root` to `student`, and with it the *meaning* of the permission string: the `rw-` now describes what `student` may do to the file. Root retains the ability to do anything, with or without a permission bit.

<details>
<summary><strong>Read more</strong> — who may change ownership, and the two useful variants</summary>

Only root (or a user holding `CAP_CHOWN`) may give a file away, and that restriction is deliberate: if any user could hand ownership of their files to someone else, the accounting of who is responsible for what would be meaningless.

Adding a colon sets the group in the same operation, so `chown student:developers practice-notes.txt` changes owner and group at once, and `-R` applies the change recursively to an entire tree. That recursive form is the usual one when you move a web directory between accounts: `chown -R www-data:www-data /var/www/html`.

</details>

### 6.3 `chgrp` — granting ownership to a group

`chgrp` has exactly the same shape as `chown` but takes a group name. Here we change the group to `developers`:

```bash
chgrp developers practice-notes.txt
```

```bash
ls -l practice-notes.txt
```

**Output:**

```text
-rw-r--r-- 1 student developers 82 Oct  8 11:45 practice-notes.txt
```

The fourth column now reads `developers`. The practical value is that it shares the file with a *set* of people without opening it to everyone: the middle triad `r--` means every member of `developers` may read the file, while the last triad still protects it from unrelated users.

<details>
<summary><strong>Read more</strong> — two rules that catch people out</summary>

First, you may only change the group to one that you yourself are a member of, unless you are root — otherwise you could grant access you do not hold.

Second, when sharing a directory between several people, the **set-group-ID** bit on the directory (section 6.6) makes new files automatically inherit the directory's group, which saves you from having to `chgrp` everything by hand afterwards.

To see which groups your own account belongs to, run `groups`, and for the same list with numeric IDs, `id`.

</details>

### 6.4 `chmod` — changing permissions

`chmod` — *change mode* — alters the permissions of a file. It understands two notations, and you should be comfortable with both, because other people's documentation mixes them freely.

The **symbolic** notation names the class, the operation and the right: `u` for user/owner, `g` for group, `o` for others, `a` for all three; `+` to add, `-` to remove, `=` to set exactly; and `r`, `w`, `x` for the rights. So `chmod g+w report.txt` adds write permission for the group, and `chmod o-r secret.txt` removes read from everyone outside the owner and group.

The **numeric (octal)** notation represents each class by a single digit built from the rights it holds — read is worth 4, write 2, execute 1 — and their sum is the digit:

| Digit | Binary | Rights | Meaning |
| --- | --- | --- | --- |
| 0 | 000 | `---` | no permissions at all |
| 1 | 001 | `--x` | execute only |
| 2 | 010 | `-w-` | write only |
| 3 | 011 | `-wx` | write and execute |
| 4 | 100 | `r--` | read only |
| 5 | 101 | `r-x` | read and execute |
| 6 | 110 | `rw-` | read and write |
| 7 | 111 | `rwx` | read, write and execute |

You then give three digits, in the order **owner, group, others**. So `chmod 644 file` produces `rw-r--r--`, `chmod 755 file` produces `rwxr-xr-x`, and `chmod 600 file` produces `rw-------` — the correct setting for anything containing private data such as an SSH private key.

Another way of arriving at the same result is the symbolic `chmod +x`, with no class letter, which means "the same change for every class":

```bash
cp backup.sh script-demo.sh
```

```bash
ls -l script-demo.sh
```

**Output:**

```text
-rw-r--r-- 1 root root 4096 Oct  8 12:04 script-demo.sh
```

```bash
chmod +x script-demo.sh
```

```bash
ls -l script-demo.sh
```

**Output:**

```text
-rwxr-xr-x 1 root root 4096 Oct  8 12:05 script-demo.sh
```

Before, a text file you could read but not run; afterwards, executable — and the terminal usually turns it green to say so.

<details>
<summary><strong>Read more</strong> — the two variants of <code>chmod</code> that bite, and the <code>umask</code></summary>

`chmod -R` applies a permission change to every file and directory below a path, and `chmod -v` prints a line for each object it modified — invaluable when a recursive change touches two thousand files and you want to be sure the command matched what you intended.

One more piece of the permission story deserves a mention even though it is not a `chmod` switch: the **umask**. When a program creates a new file, the kernel starts from a default of `666` for files and `777` for directories and subtracts the umask to get the result — the typical umask is `022`, which is why new files appear as `644` and new directories as `755`. That is exactly why the `touch` in section 3.2 produced `-rw-r--r--` without anyone asking for it. Run `umask` to see your current value, and `umask 077` to make everything you create from that point onwards private to you — a sensible setting on a shared machine.

`chmod 777 file` gives it to everyone, readable, writable and executable by absolutely everyone; `chmod 111` grants execute and nothing else, which is technically enough to run a binary but may leave the program unable to read its own data files — which is why `111` is much rarer in practice than `755`.

</details>

### 6.5 Setting the SUID bit

The SUID bit — *set user ID* — says something quite different from `rwx`. When it is set on an executable, any user may run that file, and while it runs the process carries the **permissions of the file's owner** rather than those of the person who started it. Those elevated rights apply only to that one execution: the caller does not become root, they merely run this one binary with the owner's authority.

Numerically you add **4 before the three regular digits**, so a `644` file becomes `4644`:

```bash
ls -l script-demo.sh
```

**Output:**

```text
-rwxr-xr-x 1 root root 4096 Oct  8 12:05 script-demo.sh
```

```bash
chmod 4644 script-demo.sh
```

```bash
ls -l script-demo.sh
```

**Output:**

```text
-rwSr--r-- 1 root root 4096 Oct  8 12:06 script-demo.sh
```

Read the result carefully: the SUID bit does not appear as an extra character, it *overwrites* the owner's execute slot, turning `x` into `s`. Because the owner's execute bit was **not** set here, we see an uppercase `S` — and the file is not executable at all, so the bit changed nothing except the appearance.

<details>
<summary><strong>Read more</strong> — lowercase <code>s</code>, the script caveat, and why this matters in security</summary>

When the owner's execute bit *is* also set, you see a lowercase `s`, as in `-rwsr-xr-x` — the normal appearance of a system binary such as `passwd` or `su`. To actually combine SUID with an executable file you want `4755`. In symbolic notation the same setting is `chmod u+s file`, removed with `chmod u-s file`.

One caveat many older tutorials skip: the kernel honours SUID only on *compiled* binaries, and deliberately ignores the bit on shell scripts and other interpreted programs. A script with SUID set will run with the privileges of whoever started it, not of its owner — a safety feature, since a script's behaviour depends on an interpreter an attacker might influence through `PATH`. So setting the bit on a script is a useful exercise in reading permission strings, while real SUID privilege-escalation targets are always binaries and shared objects.

The reason the bit matters so much in security work is that a root-owned SUID binary is a legitimate, administrator-created way for unprivileged users to perform privileged actions — and therefore a piece of code worth auditing. If it can be tricked into running a program of the attacker's choosing, that program inherits its root authority. That is the classic *SUID privilege escalation* path, and finding candidates is one command away:

```bash
find / -perm -4000 -type f 2>/dev/null
```

**Output:**

```text
/usr/bin/passwd
/usr/bin/su
/usr/bin/sudo
/usr/bin/mount
/usr/bin/newgrp
/usr/bin/chsh
```

`-perm -4000` means "has the SUID bit set" — the same `4` prefix you used with `chmod`. The listing above is what a healthy Debian-family system looks like; any *unexpected* entry is worth investigating immediately, because it is either a package you did not know you had or something that got there without your knowledge.

</details>

### 6.6 Setting the SGID bit

**SGID** — *set group ID* — is the group's version of the same idea. On an executable it means the process runs with the privileges of the file's owning **group**; on a directory it means something different and very useful: new files created inside inherit the directory's group instead of the creator's primary group.

Numerically you add **2 before the regular permissions**, so a `644` file becomes `2644`:

```bash
ls -l script-demo.sh
```

**Output:**

```text
-rwxr-xr-x 1 root root 4096 Oct  8 12:06 script-demo.sh
```

```bash
chmod 2466 script-demo.sh
```

```bash
ls -l script-demo.sh
```

**Output:**

```text
-rw-rwSrw- 1 root root 4096 Oct  8 12:07 script-demo.sh
```

The same overwriting trick is at work, but in the middle triad: SGID occupies the *group's* execute slot and shows as a lowercase `s` when execute is also granted (`rwxr-sr-x`) and as an uppercase `S` when it is not, as above.

The practically useful case is SGID on a directory, symbolically `chmod g+s directory`:

```bash
chmod g+s /srv/share
```

```bash
ls -ld /srv/share
```

**Output:**

```text
drwxr-sr-x 4 root developers 4096 Oct  8 12:09 /srv/share
```

Any file that any member of `developers` now creates in `/srv/share` will belong to the group `developers` automatically — exactly what a team working in the same directory wants, so that nobody has to run `chgrp` after every save.

<details>
<summary><strong>Read more</strong> — which SGID you are likely to meet</summary>

SGID on directories is common and benign; SGID on executables is rare and warrants the same audit question as SUID above. The same kernel caveat applies: the bit is ignored on interpreted scripts and honoured on compiled binaries.

</details>

### 6.7 The third special bit: the sticky bit

The **sticky bit** is written as **1** before the regular permissions (`chmod 1777`). It is irrelevant on files on modern Linux, but very meaningful on directories: inside a sticky directory, users may create files and may delete only *their own* files, even if the directory is world-writable.

```bash
ls -ld /tmp
```

**Output:**

```text
drwxrwxrwt 21 root root 4096 Oct  8 12:10 /tmp
```

The final `t` in `drwxrwxrwt` is the sticky bit in its open form (it would appear as an uppercase `T` if the directory were not searchable). Without it, any user could delete any other user's temporary files in `/tmp`.

<details>
<summary><strong>Read more</strong> — a missing sticky bit is a real finding</summary>

Symbolically the setting is `chmod +t directory`. If you ever see a world-writable directory *without* the sticky bit, that is a genuine finding rather than a quirk: it means every user on the system can destroy every other user's work.

You can hunt for them with `find / -type d -perm -0002 ! -perm -1000 2>/dev/null`, which reads as "directories that are writable by others and do not have the sticky bit".

</details>

### 6.8 Putting it together: a worked permissions scenario

Imagine a small shared workspace. We want a directory the team can all write into, files created there to inherit the team group, a script anyone may run, and a private note only its owner may read:

```bash
mkdir /srv/share
```

```bash
chgrp developers /srv/share
```

```bash
chmod 2775 /srv/share
```

```bash
ls -ld /srv/share
```

**Output:**

```text
drwxrwsr-x 2 root developers 4096 Oct  8 12:12 /srv/share
```

```bash
cp backup.sh /srv/share/backup.sh
```

```bash
chmod 755 /srv/share/backup.sh
```

```bash
chown root:developers /srv/share/backup.sh
```

```bash
touch /srv/share/private.key && chmod 600 /srv/share/private.key
```

```bash
ls -l /srv/share
```

**Output:**

```text
total 8
-rwxr-xr-x 1 root developers 4096 Oct  8 12:13 backup.sh
-rw------- 1 root developers    0 Oct  8 12:13 private.key
```

Reading the result: the directory is `drwxrwsr-x` — group-writable, and carrying the SGID bit so everything the team adds stays in the `developers` group. The script is `755`, so anyone may run it but only its owner may rewrite it. The key is `600`, readable and writable by its owner and by nobody else — which is also the setting SSH will *insist* on before it agrees to use a private key.

<details>
<summary><strong>Read more</strong> — why this is the shape of real system administration</summary>

Four commands — `mkdir`, `chgrp`, `chmod` and `cp` — and a share is configured correctly. That is how much of Linux system administration actually works: a handful of small, composable commands, each doing one thing well, verified with `ls -l` after every step.

The pattern to take away is the verification step itself. In security work you will rarely be trusted on your word that a permission change took effect; you will be asked to show the `ls -l` line that proves it.

</details>

---

## 7. Quick reference cheat sheet

### Navigation and orientation

| Command | What it does | Typical use |
| --- | --- | --- |
| `pwd` | prints the current working directory | `pwd` |
| `whoami` / `id` | shows the current user / user, group and IDs | `id` |
| `cd <dir>` | changes directory | `cd /var/log` |
| `cd ..` / `cd -` / `cd ~` | up one level / previous directory / home | `cd -` |
| `ls` | lists directory contents | `ls -lah` |
| `tree` | draws the directory structure as a tree | `tree -L 2` |

### Finding things

| Command | What it does | Typical use |
| --- | --- | --- |
| `locate <keyword>` | searches the pre-built filename index (fast, may be stale) | `locate wordlists` |
| `whereis <name>` | path of a binary plus its man pages | `whereis git` |
| `which <name>` | the binary the shell would actually run | `which python3` |
| `type <name>` | whether a name is a binary, alias, function or builtin | `type cd` |
| `find <where> <what>` | live search by name, type, size, owner, time, permissions | `find / -type f -name "*.conf" 2>/dev/null` |
| `grep <pattern> <file>` | prints matching lines | `grep -rin "password" /etc` |
| `grep -v <pattern>` | prints non-matching lines | `ifconfig \| grep -v inet6` |

### Files and directories

| Command | What it does | Typical use |
| --- | --- | --- |
| `touch <file>` | creates an empty file or updates timestamps | `touch notes.txt` |
| `cat <file>` | prints a file (short files only) | `cat /etc/hostname` |
| `mkdir <dir>` | creates a directory | `mkdir -p tools/nmap/scans` |
| `cp <src> <dst>` | copies a file | `cp -r dir/ /backup/` |
| `mv <src> <dst>` | moves or renames | `mv old.txt new.txt` |
| `rm <file>` | deletes a file, permanently | `rm -i *.tmp` |
| `rm -r <dir>` | deletes a directory and everything inside | `rm -ri old-dir/` |
| `rmdir <dir>` | deletes an empty directory only | `rmdir empty-dir/` |

### Text handling

| Command | What it does | Typical use |
| --- | --- | --- |
| `head -n N <file>` | first N lines (default 10) | `head -n 20 config.conf` |
| `tail -n N <file>` | last N lines (default 10) | `tail -n 50 /var/log/syslog` |
| `tail -f <file>` | live-follow a file as it grows | `tail -f /var/log/auth.log` |
| `nl <file>` | prints a file with line numbers | `nl script.sh` |
| `wc -l <file>` | counts lines | `wc -l access.log` |
| `sed 's/a/b/g' <file>` | search and replace in a stream | `sed 's/old/new/g' file` |
| `sed -i.bak 's/a/b/g' <file>` | the same, editing in place with a backup | `sed -i.bak 's/80/8080/' conf` |
| `more <file>` / `less <file>` | one screenful at a time | `less /etc/ssh/sshd_config` |
| `sort` / `uniq` / `cut -d -f` | sort, de-duplicate, extract columns | `cut -d: -f1 /etc/passwd \| sort` |
| `tee <file>` | copies output to a file *and* the screen | `cmd \| tee log.txt` |

### Software management (Debian-family)

| Command | What it does | Typical use |
| --- | --- | --- |
| `apt-cache search <term>` | searches the package catalogue | `apt-cache search scanner` |
| `apt-get install <pkg>` | installs a package and its dependencies | `apt-get install -y git` |
| `apt-get remove <pkg>` | removes a package, keeps its configuration | `apt-get remove git` |
| `apt-get purge <pkg>` | removes a package *and* its configuration | `apt-get purge git` |
| `apt-get autoremove` | removes orphaned dependencies | `apt-get autoremove` |
| `apt-get update` | refreshes the package lists (installs nothing) | `apt-get update` |
| `apt-get upgrade` | applies available upgrades | `apt-get upgrade` |
| `nano /etc/apt/sources.list` | edits the repository list | `nano /etc/apt/sources.list.d/local.list` |

### Permissions

| Command | What it does | Typical use |
| --- | --- | --- |
| `ls -l` | shows permissions, owner, group, size, date | `ls -l /srv/share` |
| `chown <user> <file>` | changes the owner | `chown -R www-data:www-data /var/www` |
| `chgrp <group> <file>` | changes the owning group | `chgrp developers report.txt` |
| `chmod 644 <file>` | `rw-r--r--` — owner writes, everyone reads | `chmod 644 index.html` |
| `chmod 755 <file>` | `rwxr-xr-x` — everyone may run it | `chmod 755 tool.sh` |
| `chmod 600 <file>` | `rw-------` — owner only | `chmod 600 id_rsa` |
| `chmod +x <file>` | adds execute permission to all classes | `chmod +x install.sh` |
| `chmod u+s` / `4755` | sets SUID (run as the file's owner) | `chmod u+s binary` |
| `chmod g+s` / `2775` | sets SGID (inherit the directory's group) | `chmod g+s /srv/share` |
| `chmod +t` / `1777` | sets the sticky bit (delete only your own files) | `chmod +t /srv/share` |
| `umask` | shows/sets the default permission mask | `umask 077` |
| `find / -perm -4000 -type f 2>/dev/null` | lists SUID files | as shown |

---

## 8. Practice exercises

The commands in this guide only become yours when you have typed them yourself. Work on a virtual machine or in a container, and use a throwaway user account for the permission parts.

1. **Orientation.** Open a terminal and report, in order: the current directory, your username, your user and group IDs, and the contents of your home directory including hidden files. Then move to `/etc`, come back to your home directory in two different ways, and use `cd -` to bounce between `/etc` and `/var/log` three times.
2. **Reading help.** Without using the internet, find out what the `-t` flag does for `ls`, what the `-p` flag does for `mkdir`, and which pager keys move backwards. Use `--help` for one and `man` for the others, and page through at least one manual with `less` instead of dumping it to the screen.
3. **Searching.** Use `locate` to find every file whose name contains `hosts`, and then use a single `find` command to locate every regular file in `/etc` that is larger than 100 KB, discarding the permission errors. Explain in your own words why the two commands can disagree.
4. **Files.** Create a directory `practice` in your home directory, create five empty files inside it using brace expansion, copy the whole directory to `/tmp`, rename one file inside the copy, then delete the copy and the original with `rm -r` — checking with `ls` after each step.
5. **Text.** Write a ten-line file from the command line using a here-document. Print its first three lines, its last three lines, all of it with line numbers, and then use `sed` to replace a word throughout *without* modifying the original, and afterwards *with* modification and a backup file. Verify the claim that `sed` alone changes nothing on disk.
6. **Searching inside text.** Use `grep -n` to find every line of `/etc/passwd` that mentions `nologin`, and use `cut` and `sort` to produce an alphabetical list of every username on the system. Then count them with `wc -l`.
7. **Packages.** Search the catalogue for a text editor of your choice, inspect it with `apt-cache show`, install it, run it once, then remove it with `purge`. Watch what `autoremove` reports afterwards and explain why the list of packages is not empty.
8. **Permissions.** As root, create a file, note its default permissions and explain them in terms of the umask. Change its owner to a normal user, change its group, then set `644`, `755`, `600`, `4755` and `2755` in turn, running `ls -l` after each change and writing down what the permission string looks like. Finally, log in as the ordinary user and note which of those operations you are allowed to repeat without `sudo`.
9. **Audit.** Produce a list of every SUID file on the machine, and a list of every world-writable directory that lacks the sticky bit. For each entry on either list, decide whether it is expected, and be able to say why.

If you can complete these nine exercises from memory alone, you have the command-line foundation that everything else in Linux administration and security testing is built on, and you are ready for Part 2.

---

## Closing notes

Almost every mistake a beginner makes falls into one of three categories: not knowing which directory they are in, not knowing which user they are running as, and not reading the error message that would have explained the problem. The first two are answered by `pwd` and `whoami` before every risky command; the third is a habit of attention rather than a command. Everything else in this guide is syntax, and syntax is what a manual page is for.

<details>
<summary><strong>Read more</strong> — the two habits worth building in the first week</summary>

**Read the plan before you approve it.** `apt-get` tells you exactly what it will remove, `find -exec` can be tested with `-print`, and destructive commands can be previewed by prefixing them with `echo` so the shell shows you what it intends to run.

**Write things down.** Keep a plain text file of the commands you had to look up, and after a few weeks you will find that you have written your own cheat sheet, in your own words — worth more than any guide you can download.

</details>

**Part 2** is the continuation and it picks up exactly where this guide leaves off. [Linux for Beginners (Part 2): Networks, Processes and Environment Variables](02-linux-fundamentals-part-2.md) covers reading and reconfiguring the interfaces a machine uses to talk to the network, watching and controlling the processes it runs, and managing the environment variables that every program you launch inherits. **[Part 3: Bash Scripting, Automation and Services](03-linux-fundamentals-part-3.md)** then turns all of it into automation: a Bash script that does a day's work in a second, a `cron` job that runs it at 23:55 without being asked, and the services — Apache, OpenSSH, FTP — that let a machine answer requests on its own.
