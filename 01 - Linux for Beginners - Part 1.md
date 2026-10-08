# Linux for Beginners (Part 1): The Shell, Files, Text, Packages and Permissions

<<<<<<<< HEAD:docs/01 - Linux for Beginners - Part 1.md
*This is Part 1 of a three-part guide. It is followed by [Part 2: Networks, Processes and Environment Variables](02%20-%20Linux%20for%20Beginners%20-%20Part%202.md) and [Part 3: Bash Scripting, Automation and Services](03%20-%20Linux%20for%20Beginners%20-%20Part%203.md).*
========
*This is Part 1 of a three-part guide. Its continuations — networks, processes and environment variables, then scripting, scheduling and services — are in [Linux for Beginners (Part 2)](02-linux-for-beginners-part-02.md) and [Linux for Beginners (Part 3)](03-linux-for-beginners-part-03.md).*
>>>>>>>> origin/arena/52f71d52-attack-scripts:docs/01-linux-for-beginners-part-01.md

## Introduction

More often than not, certain operating systems tend to get tied to certain tasks, and when the task is penetration testing, a Linux-based operating system is almost always the platform of choice. This guide is written for someone who has never opened a Linux terminal before and wants to become comfortable with the fundamentals in a single sitting. Rather than dumping a list of commands on you, every section explains *why* a command exists, *what* it does to your system, and *how* you can verify with your own eyes that it actually did it.

The material follows a friendly difficulty curve. We begin with the two questions every beginner asks out loud ("where am I?" and "who am I?") and with the commands that move you around the file system and list what is inside it. From there we look at the built-in help systems, because no one memorises every flag of every utility, and learning to read a manual page is a skill that pays for itself immediately. Next comes file and directory manipulation, the everyday business of creating, copying, moving and deleting things; then text manipulation, which matters far more on Linux than on other systems because almost everything you administer here is a plain text file. The last two sections cover installing and removing software through the package manager and understanding the Unix permission model, the part of Linux that trips up beginners most often and the part that matters most when you later study privilege escalation.

<<<<<<<< HEAD:docs/01 - Linux for Beginners - Part 1.md
This is **Part 1** of three. Everything here is the layer you need before anything else makes sense: the shell, the file system, text, packages and permissions. **Part 2** carries on with the topics that sit just beneath everyday use — networks and interfaces, process management, and the environment variables that quietly decide which program runs, which resolver answers and which editor opens — and **Part 3** finishes with the automation layer: writing Bash scripts, scheduling them with `cron`, and running the services that keep a machine useful. Neither continuation assumes anything beyond the material below.
========
This is **Part 1** of three. Everything here is the layer you need before anything else makes sense: the shell, the file system, text, packages and permissions. **Part 2** carries on with the topics that sit just beneath everyday use — networks and interfaces, process management, and the environment variables that quietly decide which program runs, which resolver answers and which editor opens — and **Part 3** finishes with scripting, scheduling and services, the automation layer that multiplies everything before it. Both assume only that you are comfortable with the material below.
>>>>>>>> origin/arena/52f71d52-attack-scripts:docs/01-linux-for-beginners-part-01.md

Every step is presented as a console transcript: the command that was typed, followed by the output it produced on screen, followed by an explanation in plain English of what just happened. The terminal contents are always written out as text rather than shown as a picture, so that you can copy and paste, scroll, search and compare them against your own results. If you follow along on your own machine, by the end of the guide you will have created, inspected, edited, mangled, re-permissioned and deleted a small set of files, changed an interface's address and hardware identifier, watched and re-prioritised the processes running on the machine, and reshaped the environment your shell hands to every program it starts — using nothing but the command line.

> **A word of caution before you start.** The examples below use `apt-get`, `chown`, `chmod`, `ifconfig` and `nano` on system paths, and several of them require root privileges. Practise inside a throwaway virtual machine (a Debian or Kali image is ideal) and take a snapshot before you begin, so that a typo can never damage real data. Deleting files as root is permanent: Linux has no recycle bin on the command line, and reconfiguring the network or killing the wrong process can lock you out of the machine you are working on.

### How to read the transcripts

Throughout this guide, commands appear inside a transcript that looks like this:

```bash
root@kali:~# pwd
/root
```

A few conventions are worth learning straight away, because they are part of the grammar of the shell itself:

* `root@kali:~#` is the **prompt**. It tells you that the current user is `root`, that the machine is named `kali`, and that you are currently sitting in the directory `~` (the home directory of the user). The trailing `#` is a strong visual warning: it means you are running as the all-powerful root account. A normal, unprivileged user gets a `$` at the end of the prompt instead — if you ever see `#` and did not expect it, stop and check what you are about to type.
* Everything on the prompt line *after* the prompt is what you typed. Every line underneath it, up to the next prompt, is what the program printed back to you. Nothing you type is displayed unless you type it, and nothing is executed until you press Enter.
* Inside configuration files, a line starting with `#` is a **comment**, not a command. The same character therefore means two completely different things depending on whether it appears before or after a prompt — a distinction that confuses nearly every beginner at least once.
* Text inside angle brackets, for example `cp <source> <destination>`, is a placeholder you replace with a real path.
* Output shown here was captured on a Debian-based system and is representative rather than identical: package versions, file sizes, timestamps, interface names and the number of matching lines will differ on your own installation. What matters is the shape of the output, not the exact digits.

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

Linux offers a far higher level of control over the operating system than a typical desktop platform does, and the reason for that starts with its **open source** nature. Because the source code of the kernel, the shell and virtually every utility is available for inspection, the system is **transparent**: if you want to know exactly how a network socket is created, how a password hash is compared or how a file handle is checked against a permission bit, you can read the code that does it. Before you try to break or defend anything, you have to understand how it works, and transparency is the single biggest advantage Linux gives you in that pursuit.

A second practical reason is **tooling gravity**. Because Linux is so popular in the penetration-testing and security-research community, the vast majority of security tools and frameworks are written for it first, and many are never ported anywhere else. Learning the command line is therefore not a detour around the tools — it is the road that leads to them, since most of them are invoked from a shell and compose with other shell utilities.

Third, **maintenance** is comparatively easy. Software is installed from curated repositories with a single command, updates are applied system-wide, and dependencies are resolved automatically, which is why an installed Linux server can be kept current for years with very little effort. Linux also tends to be very stable under the workload of long-running tools such as network scanners or brute-force utilities, which is exactly the kind of work you will be doing with it.

None of this means that Linux is magically secure. It means that Linux gives you the controls and the visibility to see what is happening on the system, and that is the foundation of every technique that follows.

---

## 2. The terminal and your first commands

Just as you would use a graphical desktop to create folders, move files and copy things around, every one of those everyday operations has a command-line equivalent. The terminal — also called the shell, the console or the command-line interface — is the program that reads what you type, interprets it, and asks the operating system to perform the operation you requested.

When you open a terminal window you are dropped into a shell session that starts in your home directory, and every command you run afterwards runs from some *current working directory*. Almost every confusion a beginner experiences traces back to not knowing which directory that is, so the first two commands we learn are the ones that answer the two most fundamental questions: *where am I* and *who am I*.

### 2.1 `pwd` — print working directory

The `pwd` command answers the first question. It takes no arguments, prints a single absolute path, and exits; whatever directory that path names is the directory in which the commands you type next will operate, and it is also the directory in which any file you create without giving an explicit path will be created.

```bash
root@kali:~# pwd
/root
```

Reading the transcript: the shell prompt says we are in `~`, and `pwd` confirms that `~` expands to `/root`, the home directory of the root user. The tilde is a shorthand that the shell expands before the command ever sees it, so `cd ~` and `cd /root` are the same instruction when you are logged in as root. If you are logged in as a normal user called `student`, `pwd` will print `/home/student` instead, and the prompt's `~` will refer to that path. Notice that the output is a single line with no decoration whatsoever — this is one of the great strengths of the command line: output is meant to be read by other programs as much as by humans.

### 2.2 `whoami` — identify the current user

The second question, *who am I*, is answered by the `whoami` command. This matters enormously on Linux because the operating system decides what you are allowed to read, write and execute based on the identity of the process making the request, not on where you are sitting or which desktop you logged in to.

```bash
root@kali:~# whoami
root
```

Here we are logged in as `root`, which translates to the local administrator account in Windows terminology. The root account is not subject to the ordinary permission checks, which is precisely why it is so useful and so dangerous: a wrong command executed as root can delete an entire system, while the same command run as a normal user would simply be refused. A closely related command, `id`, prints the numeric user id, the primary group id and every supplementary group the account belongs to, which is the information the kernel actually uses when it evaluates permissions:

```bash
root@kali:~# id
uid=0(root) gid=0(root) groups=0(root)
```

Three numbers to remember: user id `0` is always root, and on Debian-based systems ordinary human accounts start at `1000`. If you ever see a process running as `uid=0` that has no business doing so, you should be suspicious — that is the essence of a privilege escalation bug.

### 2.3 `cd` — changing directories

To move around the file system from the terminal we use `cd`, short for *change directory*. You give it the directory you want to move to, and if the operation succeeds the shell prints nothing at all — on Linux, silence is success. The only feedback is the change in your prompt, which now shows the new location.

```bash
root@kali:~# cd Desktop/
root@kali:~/Desktop# pwd
/root/Desktop
```

The prompt changed from `~` to `~/Desktop`, and `pwd` confirms the move to `/root/Desktop`. Notice the trailing slash on `Desktop/` — it is optional and purely cosmetic; it tells a human reader that the name refers to a directory, and the shell treats `Desktop` and `Desktop/` identically. `cd` also understands a handful of special destinations that every Linux user memorises within a day:

```bash
root@kali:~/Desktop# cd ..
root@kali:~# cd /
root@kali:/# cd ~
root@kali:~# cd -
/root/Desktop
root@kali:~/Desktop#
```

* `cd ..` moves one level **up** the tree, from `/root/Desktop` to `/root`. The `..` entry exists in every directory and always points at its parent.
* `cd /` moves to the **root of the file system**, the single directory from which everything else hangs. Note that this is a different concept from the root *user*; the forward slash is the top of the directory tree, while `root` is the name of the administrator account.
* `cd ~` (or simply `cd` with nothing after it) returns you to your **home directory** from anywhere on the system, which is a lifesaver after you have wandered deep into `/usr/share/doc` looking for something.
* `cd -` returns you to the **previous directory**, the way a browser's back button works. As you can see from the transcript, `cd -` prints the directory it moved you to, which is the only case where `cd` is chatty.

One last note for beginners: if you try to `cd` into something that does not exist, the shell will print a clear error rather than doing nothing, for example `bash: cd: Downloads/: No such file or directory`. That wording is worth reading carefully — it tells you both the program that complained and the exact argument it could not find.

### 2.4 `ls` — listing the contents of a directory

Where `pwd` tells you where you stand, `ls` tells you what is standing around you. It lists the contents of a directory — the current one by default, or any directory you name as an argument. It is very similar in spirit to the `dir` command on Windows, but considerably more flexible.

```bash
root@kali:~# cd Desktop/
root@kali:~/Desktop# ls
practice-notes.txt  simple_bash.sh  tools
```

Three entries were returned: two regular files and one directory, separated by whitespace and sorted alphabetically. On a modern terminal these names are also colour-coded — directories typically appear in blue, executable files in green and archive files in red — which gives you a quick visual summary before you read a single character.

The plain `ls` output is nicely readable for humans but it hides an enormous amount of useful information, so in practice you will almost always reach for one of the flags. The most important of these is `-l`, the *long* format, which prints one entry per line with permissions, ownership, size and timestamp attached:

```bash
root@kali:~/Desktop# ls -l
total 12
-rw-r--r-- 1 root root   96 Oct  8 11:42 practice-notes.txt
-rwxr-xr-x 1 root root  118 Oct  8 11:40 simple_bash.sh
drwxr-xr-x 2 root root 4096 Oct  7 19:05 tools
```

Each line has the same seven fields, and learning to read them now will save you a great deal of head-scratching later — the whole of section 6 is essentially a study of the first field. From left to right: the permission string (`-rw-r--r--`), the number of hard links (`1`), the owner (`root`), the owning group (`root`), the size in bytes (`96`), the last modification time (`Oct 8 11:42`), and finally the name. The first character of the permission string tells you what kind of object the entry is: a leading `-` means a regular file, a leading `d` means a directory. So `tools` is a directory, while the two other entries are files.

A few more flags are worth memorising because they come up constantly: `-a` also shows *hidden* entries (files and directories whose names begin with a dot, such as `.bashrc` and `.ssh`, which `ls` hides by default); `-h` prints sizes in human-readable units such as `4.0K` and `2.3M` instead of raw bytes; `-t` sorts by modification time with the newest first; `-S` sorts by size with the largest first; and `-R` recurses into every subdirectory. Combining flags is normal and the order rarely matters — `ls -lah` is a perfectly ordinary thing to type.

```bash
root@kali:~/Desktop# ls -lah
total 12K
drwxr-xr-x 3 root root 4.0K Oct  8 11:44 .
drwx------ 1 root root 4.0K Oct  8 11:38 ..
-rw-r--r-- 1 root root   96 Oct  8 11:42 practice-notes.txt
-rwxr-xr-x 1 root root  118 Oct  8 11:40 simple_bash.sh
drwxr-xr-x 2 root root 4.0K Oct  7 19:05 tools
```

With `-a` the two familiar shortcuts appear at the top: `.` is the current directory and `..` is its parent, which is exactly what makes `cd ..` work. With `-h`, sizes are shown as `4.0K` rather than `4096`. Finally, note `total 12K` — the total size of the directory's contents in blocks, not the size of the directory itself; directories are tiny bookkeeping records, and their reported size (often 4096 bytes) is an implementation detail you can safely ignore for now.

### 2.5 `--help` — the built-in quick reference

Nearly every command, application or utility on Linux ships with a dedicated help file that describes its usage, and reaching for it should become a reflex rather than an admission of defeat. When you are stuck on the syntax of a tool, or you simply want to know which flags it accepts without leaving the terminal, almost every program will answer to `-h` or `--help`.

```bash
root@kali:~# volatility --help
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

The example above uses a memory-forensics framework because such tools are common on security workstations, but the behaviour is identical for every program you will ever run; if you do not have it installed, `ls --help` or `nmap --help` demonstrate exactly the same thing on any system. The exact content is not important here; the habit is. The first lines tell you what the tool is and what version you have, and the block underneath enumerates every option with a one-line description. Help output can easily be a hundred lines long, so in practice you often want to page through it rather than lose the first half off the top of your screen — the `| more` and `| less` techniques from section 4 exist precisely for this, and `tool --help | less` is a perfectly good way to explore an unfamiliar program.

Two practical notes. First, `--help` is not universal: some older or minimalist utilities only respond to `-h`, and a few ignore both, in which case `man` is your next stop. Second, some tools print their help to *standard error* rather than *standard output*, which matters if you try to pipe it into another program and see nothing — the section on `find` below explains the difference between the two streams in detail.

### 2.6 `man` — the manual pages

In addition to the quick help summary, most commands and applications also have a full manual page, which you access by typing `man` followed by the name of the command. A manual page is the authoritative reference written by the people who maintain the software, and it is structured in a consistent way: `NAME`, `SYNOPSIS` (the grammar of the command), `DESCRIPTION` (every option explained at length), and usually `EXAMPLES`, `FILES`, `SEE ALSO` and `AUTHOR` sections at the end.

```bash
root@kali:~# man ls
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

The screenful shown above is the *pager* that `man` opens automatically; the final line is its status bar. Navigation inside a manual page is the same as in `less`, which we cover later: press `q` to quit, the space bar to scroll down one screen, `b` to scroll back up, `/keyword` to search forward for a keyword and `n` to jump to the next match. If you are not sure whether a manual page exists for a topic, `man -k <keyword>` searches the short descriptions of every page on the system, which is an excellent way to discover a command whose name you do not yet know.

The `(1)` next to the name is the manual *section* number, and it tells you what kind of documentation you are reading: section 1 is user commands, section 5 is file formats and conventions (try `man 5 passwd` to see the structure of the password file rather than the command that edits it), section 8 is system administration commands, and section 7 is miscellaneous topics such as `man 7 capabilities`. When the same name exists in several sections, `man` picks the lowest-numbered one by default, and you can ask for a specific section explicitly as shown.

### 2.7 `locate` — searching for filenames by keyword

When you are looking for a specific file and you only remember part of its name, one of the easiest ways to find it is `locate`. You give it a keyword and it prints every path in its database that contains that string anywhere in the name, which makes it dramatically faster than walking the file system by hand — it answers in milliseconds even on a machine with millions of files.

```bash
root@kali:~# locate wordlists | more
/usr/share/wordlists
/usr/share/wordlists/dirb
/usr/share/wordlists/dirb/big.txt
/usr/share/wordlists/dirb/common.txt
/usr/share/wordlists/rockyou.txt.gz
--More--
```

The pipe into `more` is not strictly necessary but it is a good habit: a keyword like `wordlists` can easily match several hundred paths, and `more` gives you one screenful at a time so you can skim, press Enter for the next page, and press `q` to leave when you have seen enough. Note that `locate` matches the *string*, not the filename component, so a search for `passwd` will also return files such as `/etc/passwd-` and `/usr/share/doc/passwd.html` — a useful reminder that the pattern you give it should be chosen with a little thought.

`locate`'s great weakness is worth understanding rather than merely memorising: it does not actually look at the file system at all. It looks at a pre-built index, usually `/var/lib/mlocate/mlocate.db`, which the operating system refreshes from a daily cron job. The consequences are two. First, a file you created five minutes ago will not appear until the index is rebuilt, which is why you sometimes run `locate` for your own file and conclude that the command is broken. Second, because the results are unfiltered by permission, `locate` will happily print the path of a file you are not allowed to read — it is telling you where the name exists, not whether you can open it. (This is also why a lightweight scanning tool that does not check permissions can be a mild information leak on a shared machine.) The drug for the first problem is to refresh the index on demand with `sudo updatedb`, which re-walks the tree and takes a few seconds on a normal desktop, after which your new file is searchable.

### 2.8 `whereis` and `which` — finding the binaries behind a command

Let us begin this section with a definition. Files that can be *executed* — the rough equivalent of the `.exe` files on Windows — are referred to as *binaries* (or executables). On Linux they generally live in a small set of well-known directories such as `/usr/bin`, `/usr/sbin`, `/bin`, `/sbin` and `/usr/local/bin`, and the everyday utilities we have been using, `ls`, `cd`, `cat`, `ps` and friends, all live there alongside the tools you install later.

When you want to know where a particular program actually lives, `whereis` answers the question generously: it prints the path of the binary together with the paths of any manual pages and source files it can find for the same name.

```bash
root@kali:~# whereis git
git: /usr/bin/git /usr/share/man/man1/git.1.gz
```

The `which` command is stricter and more precise. It looks only at the directories listed in your `PATH` environment variable and returns the single binary that the shell would actually run if you typed the bare command name.

```bash
root@kali:~# which git
/usr/bin/git
```

That difference is exactly why both commands exist, and the reason `which` is the more useful of the two in day-to-day work: when several copies of a program are installed in different places, `which` tells you which one wins, while `whereis` lists all of them and leaves you to guess. The `PATH` variable it consults is simply a colon-separated list of directories that the shell searches, in order, from left to right, until it finds a match:

```bash
root@kali:~# echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

That ordering explains a behaviour that surprises many beginners — if you place your own copy of a program in `/usr/local/bin` and another exists in `/usr/bin`, yours will be run, because `/usr/local/bin` is searched first. `$PATH` is also the reason a fresh shell says `command not found` for a script sitting in your current directory: the current directory is deliberately *not* on the search path (a safety measure, so that a malicious `ls` dropped into a shared folder cannot be run by accident). To run something from the current directory you must name it explicitly, for example `./simple_bash.sh`. A third command, `type`, is the shell's own answer to the same question and additionally reveals whether a name is a built-in, an alias or a function rather than a real binary:

```bash
root@kali:~# type git
git is /usr/bin/git
root@kali:~# type cd
cd is a shell builtin
```

### 2.9 Filtering output with `grep`

Very often when working on the command line you do not want *more* output, you want *less* of it — one particular line out of a hundred. This is where `grep` comes in: it reads text, either from a file you name or from its standard input, and prints only the lines that match the pattern you give it. The name comes from the editor command `g/re/p`, *globally search a regular expression and print*, and it is arguably the single most used utility in the arsenal.

```bash
root@kali:~/Desktop# grep -i "echo" simple_bash.sh
echo "Starting the backup job..."
echo "Backup finished at $(date)"
```

Here the `-i` flag makes the search case-insensitive, so a line containing `Echo` or `ECHO` would have matched just as well; without it, `grep` is case-sensitive and `grep "echo"` would have returned only the exact lowercase matches. The command found two of the lines in the script and printed them verbatim, without line numbers and without any indication of which lines they were. If you want that context, `-n` prefixes every match with its line number, and `-r` walks an entire directory tree recursively — `grep -rn "password" /etc` is the sort of command that reveals how many plaintext credentials a badly configured server accumulates.

By far the most common use of `grep`, however, is not to search files at all but to *pipe* the output of another command into it and filter that output down to the lines you care about. As an illustration, the `ifconfig` utility prints a great deal of information about every network interface on the machine, and if all you want is the IP addresses, throwing the rest away is as simple as this:

```bash
root@kali:~# ifconfig | grep inet
        inet 10.0.2.15  netmask 255.255.255.0  broadcast 10.0.2.255
        inet6 fe80::a00:27ff:fed1:3c4a  prefixlen 64  scopeid 0x20<link>
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
```

The vertical bar is the *pipe*: it takes whatever the program on its left wrote to standard output and feeds it to the program on its right as standard input. Nothing is written to disk in between; the two processes run simultaneously and data flows through the pipe as it is produced. From the four surviving lines you can read off everything that matters: the machine's real address (`10.0.2.15`, the classic NAT address of a virtual machine), its IPv6 link-local address, and the loopback interface `127.0.0.1`, which is how a machine refers to itself and which is present on every Linux system. Two refinements are worth knowing: `grep -v inet` *inverts* the match and prints every line that does **not** contain the string, which is how you throw away noise instead of keeping it; and on modern systems `ip a` is the preferred replacement for the ageing `ifconfig`, with `ip a | grep inet` producing much the same filtered result.

### 2.10 Searching the file system with `find`

The `find` command is the most powerful and flexible of the searching utilities, and it is the one that rewards a little study. Unlike `locate`, it walks the file system live, examining every file as it goes, and that lets it filter on criteria far beyond the name: the date of creation or modification, the owner, the group, the permissions, the size, and the type of the file.

Its grammar is slightly unusual at first sight — `find <where to look> <what to look for>` — because each search criterion is written as a flag, and you can stack as many as you like. Here we use the `-type` flag to say that we only want regular files (`f`) and the `-name` flag to fix the filename. The leading forward slash tells `find` where to begin its walk: the root of the file system, meaning "search the whole machine".

```bash
student@kali:~$ find / -type f -name practice-notes.txt
find: '/etc/ssl/private': Permission denied
find: '/proc/tty/driver': Permission denied
/home/student/notes/practice-notes.txt
find: '/var/cache/private': Permission denied
find: '/var/lib/private': Permission denied
```

Note that the prompt has changed to `student@kali:~$`: for the rest of this section we have dropped out of the root account and are working as an ordinary user, because that is the situation a beginner normally faces and it is what makes the next paragraph meaningful. The search found the file, exactly as intended, in our own notes directory — but look at the other lines. Because we asked `find` to search the entire file system as an unprivileged account, it also walked into directories our account is not allowed to read, and every such entry produced a `Permission denied` message. On a real system these warnings can number in the hundreds and bury the one line you actually wanted. The important thing to realise is that the two kinds of lines are travelling on *different channels*: the found path is written to **standard output** (file descriptor 1), while the errors are written to **standard error** (file descriptor 2). Because they are separate, you can silence one without touching the other, which is what the `2>/dev/null` redirection in the next transcript does.

```bash
student@kali:~$ find / -type f -name practice-notes.txt 2>/dev/null
/home/student/notes/practice-notes.txt
```

Read that as "send file descriptor 2 to the null device", where `/dev/null` is a special file that discards everything written to it. The errors vanish, the result remains, and the output is now clean enough to be fed into another command. A common alternative you will see in older tutorials achieves a similar-looking result with `2>&1 | grep -v "Permission denied"`, which first merges the errors into the normal output stream and then filters the pieces that look like permission failures. Both produce a tidy transcript, but `2>/dev/null` is preferable: it does not depend on the exact wording of the error message, it does not quietly hide genuine errors that are not permission problems, and it costs nothing in CPU time.

With that plumbing understood, the real power of `find` becomes usable, because you can now filter on almost any attribute of a file:

```bash
student@kali:~$ find /home -type f -user student -mtime -7 2>/dev/null
/home/student/notes/week-1.txt
/home/student/notes/week-2.txt
```

That command means "under `/home`, find regular files owned by the user `student` that were modified within the last seven days" — the sort of query that takes seconds to write and replaces ten minutes of clicking. A few flags worth keeping in your notes: `-name` and its case-insensitive sibling `-iname` take wildcards such as `*.conf`, `-size +10M` finds files larger than ten megabytes, `-perm -4000` finds files with the SUID bit set (we will use this in section 6), and `-maxdepth 2` stops the walk from descending further than the second level so that you do not accidentally traverse the entire disk. Finally, `find` can *do* things to what it finds, not merely print it, thanks to the `-exec` flag: `find . -name "*.tmp" -exec rm {} \;` deletes every temporary file below the current directory. That construct is powerful enough to be genuinely dangerous, so test it first with `-print` in place of `-exec` and only then let it loose.

---

## 3. Everyday file and directory operations

Now that we can find our way around, help ourselves and search the file system, it is time to perform the operations that make up most of the actual work: reading a file, creating an empty one, making a directory, copying, moving, renaming and deleting. Because the concepts are simple, this section is short per command — but do type every one of them out rather than copying blindly, because muscle memory for paths is half of what a Linux user is.

If you are following along, let us first create a small working area so that the examples have somewhere to live, using the directory tools we are about to learn:

```bash
root@kali:~# mkdir Documents/linux-lab
root@kali:~# cd Documents/linux-lab
root@kali:~/Documents/linux-lab# pwd
/root/Documents/linux-lab
```

### 3.1 `cat` — printing the contents of a file

We use the `cat` command to output the contents of a file to the terminal. Its name is short for *concatenate*, because given several files it will print them one after another, glued together — but the single-file use case, "just show me what is in here", is by far the most common.

```bash
root@kali:~/Documents/linux-lab# cat practice-notes.txt
Week 1: permissions - chmod 644 vs 755, SUID bit explained
Week 2: pipelines - grep, cut, sort, uniq, tee
Week 3: processes - ps aux, top, kill -9
```

The file's contents are printed exactly as stored, with lines appearing in the order they occur. `cat` makes no attempt to format, number or page the output — it is the raw bytes, which is precisely why it is useful inside pipelines and scripts. That same rawness is why `cat` is the wrong tool for large files: print a ten-thousand-line log and you will have to scroll back through it for the rest of the session. For anything longer than a screenful, use `less` (section 4) instead. `cat` becomes more interesting when combined with redirection: `cat file1 file2 > combined` merges two files into a third, `cat /etc/hostname` lets you read a system file directly, and `cat -n practice-notes.txt` numbers the lines as it prints them. A related utility, `tac`, prints a file *backwards*, last line first — surprisingly handy when you are reading log files where the newest entries are at the bottom.

### 3.2 `touch` — creating an empty file

The `touch` command creates a new file. Simply specifying a filename after the command results in the creation of that file, and here we create a small document to practise on:

```bash
root@kali:~/Documents/linux-lab# touch lab-notes.txt
root@kali:~/Documents/linux-lab# ls -l
total 4
-rw-r--r-- 1 root root 0 Oct  8 11:47 lab-notes.txt
-rw-r--r-- 1 root root 82 Oct  8 11:45 practice-notes.txt
```

The file appears in the listing immediately, with a size of `0` bytes and the current date as its modification time. That name is slightly misleading: the primary purpose of `touch` is not creation at all, it is *updating the timestamps* of an existing file, and if you point it at a file that already exists it will change nothing but the dates. That behaviour is genuinely useful — build systems such as `make` decide whether to recompile by comparing timestamps, so touching a source file is a quick way to force a rebuild. As a side effect, when the target does not exist, the operating system creates it, which is how the command earned its reputation as a file-creation tool.

`touch` also accepts several filenames at once and understands brace expansion, so `touch {1..5}-note.md` creates five files named `1-note.md` through `5-note.md` in one go — a trick worth remembering, because the shell performs the expansion, not `touch`, which means it works with every command. Finally note the permissions on the new file: `-rw-r--r--`, meaning the owner may read and write it, and everyone else may read it. We did not ask for that; the system applied it because of the `umask` setting, which we will meet properly in section 6.

### 3.3 `mkdir` — creating a directory

To make a directory, or `mkdir` for short, you specify the name of the directory you want after the command. If the parent directory exists and the name is not already taken, `mkdir` creates it and says nothing, which as usual means success.

```bash
root@kali:~/Documents/linux-lab# mkdir tools
root@kali:~/Documents/linux-lab# ls -l
total 8
-rw-r--r-- 1 root root   0 Oct  8 11:47 lab-notes.txt
-rw-r--r-- 1 root root  82 Oct  8 11:45 practice-notes.txt
drwxr-xr-x 2 root root 4096 Oct  8 11:49 tools
```

The new directory is visible with a leading `d` in its permission string, confirming that it is a directory rather than a regular file, and with a size of 4096 bytes — that size is not the space its contents occupy but the space of its own bookkeeping record, and it will stay at that value no matter how many files you put inside. The `2` in the third column is the hard-link count, which for a directory equals two (its own entry plus its `.` entry) plus the number of subdirectories it contains; you can safely ignore it until you study file systems in detail.

Two flags are worth knowing from day one. `-p` creates intermediate directories as needed, so `mkdir -p tools/nmap/scans` builds the whole chain in a single command instead of failing because `tools/nmap` does not exist yet — and it also silently succeeds when the target already exists, which makes it the standard way to make a directory "if it is not there yet" inside a script. `-v` prints a line for each directory it creates, which is helpful when you want confirmation that a long chain really was built.
```bash
root@kali:~/Documents/linux-lab# mkdir -pv tools/nmap/scans
mkdir: created directory 'tools/nmap'
mkdir: created directory 'tools/nmap/scans'
```

### 3.4 `cp` — copying files

To copy a file we use `cp`, which creates a duplicate of the file in the location you specify. The general form is `cp <file you want to copy> <destination of the copy>` — source first, destination second, always in that order.

```bash
root@kali:~/Documents/linux-lab# cp lab-notes.txt tools/
root@kali:~/Documents/linux-lab# ls -l tools/
total 0
-rw-r--r-- 1 root root 0 Oct  8 11:51 lab-notes.txt
```

The original is untouched and a second file with the same name and the same contents now exists inside `tools/`. We then list the destination directory to confirm the copy really landed there, which is a habit worth keeping: `cp` prints nothing on success, so if you mistype a destination path you can copy something into a place you never intended and never notice. Note the trailing slash on `tools/` — with it, `cp` understands the destination to be a directory and places the file inside; without a destination directory, `cp file1 file2` simply creates `file2` as a copy of `file1`, and it will happily *overwrite* an existing `file2` without so much as a warning.

That quiet overwriting is the main hazard of `cp`, and it is why beginners are taught two flags very early. `-i` asks for confirmation before overwriting an existing target, and `-r` (or `-R`) copies directories *recursively*, which is required because `cp` refuses to copy a directory without it — copying a directory means copying its contents, their contents, and so on. Combining them gives you the safe and useful `cp -ri source-dir/ destination-dir/`, which copies a whole tree while prompting before it overwrites anything. A few more that come up regularly: `-v` prints each file as it is copied, `-p` preserves the original permissions, ownership and timestamps instead of applying fresh ones, `-u` copies only when the source is newer than the destination (a poor man's incremental backup), and `-a` is the archive mode that bundles `-r`, `-p` and more, and is the flag to use when you want an exact replica.

### 3.5 `mv` — moving and renaming files

The move command, `mv`, can be used not only to move a file to a different directory but also to rename it, because on Linux those two things are the same operation: you are changing the name by which an inode is reached, and moving a file between directories on the same file system does not copy any data at all, it merely relinks it. Here we move the file out of the `tools` directory back to its parent:

```bash
root@kali:~/Documents/linux-lab# mv tools/lab-notes.txt .
root@kali:~/Documents/linux-lab# ls -l
total 0
-rw-r--r-- 1 root root 0 Oct  8 11:47 lab-notes.txt
-rw-r--r-- 1 root root 82 Oct  8 11:45 practice-notes.txt
drwxr-xr-x 3 root root 4096 Oct  8 11:49 tools
```

The single dot at the end is the shortcut for "the current directory", so the command reads "move `tools/lab-notes.txt` to here". The file is gone from `tools/` and sits in `linux-lab/` with its original timestamp intact — the same inode, a new name in a new directory. Renaming uses the identical syntax: `mv lab-notes.txt linux-notes.txt` leaves you with one file called `linux-notes.txt` and no file called `lab-notes.txt`. Because the command cannot tell whether you meant to rename or to overwrite, `mv` overwrites an existing destination file without asking, so `mv -i` and its confirmation prompt are worth adopting as a default, especially for destructive shell scripts. `-n` is the opposite behaviour (never overwrite) and `-v` narrates what was moved, which is invaluable in a loop or a script where dozens of files move at once.

### 3.6 `rm` — removing files

To remove a file, you simply use the `rm` command with the name of the file. Here we delete the note we created a moment ago, and the following `ls` proves that it is gone:

```bash
root@kali:~/Documents/linux-lab# rm lab-notes.txt
root@kali:~/Documents/linux-lab# ls
practice-notes.txt  tools
```

There is no output on success and, more importantly, **there is no recycle bin**. The command unlinks the file's name from its inode, and once the last link to a file is removed the data blocks are marked free and will be overwritten by future activity; there is no built-in undo. This is the single most common way beginners destroy their own work, and there are only two sane defences: always think before pressing Enter on `rm`, and make sure you know which directory you are standing in when you use wildcards, because `rm *` deletes everything in the current directory and the shell expands the `*` before `rm` ever sees it. A decade of catastrophes has been caused by the pattern `rm -rf /` and by its subtler cousin `rm -rf /etc` typed from an unexpected prompt, so it is worth internalising the rule that the `#` in your prompt means "everything you say here is final".

The flags that matter are `-i`, which asks for confirmation for each file (excellent training wheels; set it as an alias for `rm` while you are learning), `-f`, which forces deletion without prompting and, in the same breath, suppresses the error you would otherwise get for a file that does not exist, `-v` for a running commentary, and `-r`, which is required to delete a directory and everything inside it.

```bash
root@kali:~/Documents/linux-lab# rm -r tools/
root@kali:~/Documents/linux-lab# ls
practice-notes.txt
```

That last command removed `tools/` and all of its contents with no confirmation whatsoever, which brings us neatly to the directory-removal command — and to the reason so many certificates and auditors want `rm -r` out of your muscle memory.

### 3.7 `rmdir` — removing a directory

In order to remove a directory we use `rmdir`, which stands for *remove directory*. It is deliberately cautious: it will only delete a directory that is **empty**, and it will refuse with a clear error if anything at all is inside, which makes it the safe choice for tidying up.

```bash
root@kali:~/Documents/linux-lab# mkdir old-backups
root@kali:~/Documents/linux-lab# rmdir old-backups/
root@kali:~/Documents/linux-lab# ls
practice-notes.txt
```

Here the directory was empty, so `rmdir` removed it silently. Had it contained anything, we would have seen `rmdir: failed to remove 'old-backups/': Directory not empty` and the directory would have survived untouched, which is exactly the behaviour you want while you are still learning to type paths correctly. For directories that do have content, the tool is `rm -r` as shown above, where the `r` stands for *recursive* and instructs the command to descend into the directory and delete everything it finds. `rm -ri` combines recursion with a confirmation prompt for each item, and `rm -rf` combines recursion with force, which is the combination to be genuinely careful with: it will delete hundreds of files without a single question and will not even complain about the ones that were not there in the first place.

Before moving on, one general warning that applies to every destructive command in this section. The shell expands wildcards, resolves `..` and performs substitutions *before* `rm`, `mv` or `cp` ever run, so a mistyped space can change the meaning of a command completely — `rm file.txt` is a harmless typo, while `rm file .txt` is the deletion of two different things, and `rm -r ./dir /` is a typo that ends careers. When a command is destructive, print the arguments first: run `echo rm -r ./dir` to see exactly what the shell intends to hand over, then remove the `echo` and press Enter only once the line reads correctly.

---

## 4. Text manipulation

On Linux almost everything you deal with is going to be a **file**, and more often than not a *plain text* file: configuration lives in text, logs are text, user and group databases are text, the mapping tables used by networking tools are text. Hence learning how to read, slice, number, search and modify text becomes crucial to managing Linux and the applications on it. This section covers the tools you will use every day for exactly that.

To have something realistic to work with, let us create a small mapping file and a couple of lines of content. The first command below uses a *here-document*, a technique for feeding several lines of text to a command's standard input directly from the shell — the `<<'EOF'` says "everything until the line `EOF` is input", and the quotes around `EOF` prevent the shell from expanding anything inside, which is what you almost always want for file contents:

```bash
root@kali:~/Documents/linux-lab# cat > dns-mappings.txt <<'EOF'
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
root@kali:~/Documents/linux-lab# wc -l dns-mappings.txt
12 dns-mappings.txt
```

The `wc -l` at the end confirms the file has twelve lines, which we will need to know in a moment. This construct is the standard way to create a text file from the command line without opening an editor, and it will serve you well for writing small scripts and configuration stubs.

### 4.1 `head` — grabbing the beginning of a file

When dealing with large files, we use `head`, which by default displays the first **ten** lines of whatever you point it at. That default is one of the most useful inventions in the Unix toolbox, because configuration files and logs frequently begin with a comment block that explains the format of everything below it.

```bash
root@kali:~/Documents/linux-lab# head dns-mappings.txt
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

Ten lines came back and the last two lines of the file were silently dropped, which is the entire contract of the command. The `-n` flag changes how many you get — `head -3 dns-mappings.txt` prints only the three lines you asked for, and `head -n -4` prints everything *except* the last four lines, which is occasionally the easiest way to strip a trailing trailer. `head` also accepts multiple filenames and will label each block of output with the name of the file it came from, and it is most often used with pipes rather than files: `ps aux | head` gives you the header line plus the first few processes instead of several hundred, and `du -h * | sort -h | tail` uses the mirror-image command to show the ten largest items in a directory.

### 4.2 `tail` — grabbing the end of a file

Similar to `head`, the `tail` command views the **last** lines of a file, ten by default. For log files this is the more useful of the pair, because whatever is happening now is written at the bottom.

```bash
root@kali:~/Documents/linux-lab# tail dns-mappings.txt
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

Because the file has twelve lines, the last ten overlap with the output of `head` and the header comments are no longer visible. `tail -n 3` gives you just the final three lines, and `tail -n +3` gives you everything *from* line three onwards, which is a handy way to skip a two-line header when feeding a file to another program.

`tail` has one genuinely magical flag that you will use constantly: `-f`, for *follow*. It does not exit after printing the last lines; it keeps the file open and prints new lines as they are appended, which turns it into a live monitor.

```bash
root@kali:~# tail -f /var/log/syslog
Oct  8 11:52:31 kali systemd[1]: Started Session 4 of user root.
Oct  8 11:52:44 kali kernel: [ 4312.117834] usb 1-1: new high-speed USB device number 3
^C
```

The terminal sits there printing as the system logs events — in a second window you can plug in a USB stick and watch the kernel announce it here. Press `Ctrl+C` to interrupt the command and get your prompt back; the `^C` in the transcript is the shell's way of showing the interrupt you sent. This is exactly how you would watch an authentication log while testing whether a login attempt succeeds, and `journalctl -f` on a systemd machine does the same thing for the system journal.

### 4.3 `nl` — numbering the lines

We can use `nl` to number the lines while it outputs them to the terminal window. Again using our mapping file, this time numbering every non-empty line:

```bash
root@kali:~/Documents/linux-lab# nl dns-mappings.txt
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

Every line now carries a right-aligned number in a seven-character field, which is not decoration: it is the format that most compilers and interpreters use when reporting an error, so numbering a file yourself is how you make sense of a message that says *line 41: syntax error*. Two details distinguish `nl` from its simpler cousin `cat -n`. First, `nl` does not number *blank* lines by default, so a file with paragraph breaks and no comments will skip numbers at those points; `-ba` forces every line to be numbered. Second, `-s` changes the separator between the number and the text (for example `-s": "` produces `1: www.example.com ...`), and `-w` changes the width of the number field, which is what you need when a file has more than 99999 lines. `grep -n` is a third way to get numbers, and you will probably reach for it more often than either, because it numbers only the lines that match your pattern.

### 4.4 `sed` — finding and replacing text

The `sed` command — the *stream editor* — lets you search for the occurrence of a word or a text pattern and then perform an action on it, without opening an editor and without you having to sit there pressing keys. It reads its input line by line and applies your instructions to each line as it flows past, which is why it is called a stream editor and why it works perfectly inside pipelines. The most common instruction by far is **substitution**, written `s/<what to find>/<what to put there>/`, with an optional trailing `g` that means *every* occurrence on the line rather than only the first.

```bash
root@kali:~/Documents/linux-lab# sed 's/example.com/example.net/g' dns-mappings.txt | head -5
# DNS mapping file used by local network analysis tools
# Format: <name> <address> <comment>
www.example.net     192.168.1.10    # primary web server
ftp.example.net     192.168.1.11    # file server
mail.example.net    192.168.1.12    # smtp relay
```

The `s` stands for *substitute* and the `g` for *global*. Working through the pieces: `s/` opens the substitution, `example.com` is the pattern to search for, `example.net` is the replacement, and the closing `/g` applies the change to every match on each line rather than stopping after the first. Notice that the change appears in the output but the file on disk is untouched — `sed` prints the modified stream to standard output and never writes to the input file by default, which makes it completely safe to experiment with. We piped the result through `head -5` simply to keep the transcript short; without it you would have seen all twelve translated lines.

To make the change permanent you add the `-i` flag, for *in place*, and the correct and careful way to use it is with a backup suffix so that the original survives:

```bash
root@kali:~/Documents/linux-lab# sed -i.bak 's/example.com/example.net/g' dns-mappings.txt
root@kali:~/Documents/linux-lab# ls
dns-mappings.txt  dns-mappings.txt.bak
root@kali:~/Documents/linux-lab# head -3 dns-mappings.txt
# DNS mapping file used by local network analysis tools
# Format: <name> <address> <comment>
www.example.net     192.168.1.10    # primary web server
```

With `-i.bak`, `sed` renames the original to `dns-mappings.txt.bak` and writes the edited version under the original name, so a mistake is one `mv` away from being undone. Used bare, `sed -i` keeps no copy at all and its edits are as final as an `rm` — inside a script run as root this is a classic way to lock yourself out of a machine by editing the wrong line of `/etc/fstab` or `/etc/ssh/sshd_config`. A few other forms are worth filing away: an address prefix restricts the edit to a range of lines, so `sed '5,10 s/192\.168/10.0/' file` rewrites only lines five through ten; `-n '5p'` suppresses the default printing and shows only line five; `'/^#/d'` deletes every line beginning with a hash, which strips comments from a configuration file in one stroke; and because the delimiter after `s` is your choice rather than a rule, `sed 's|/usr/bin|/usr/local/bin|g'` is how you substitute paths containing slashes without escaping them. Finally, `sed` treats several characters specially — the dot in `example.com` matches *any* character, which is why the careful form writes it `example\.com` — and that detail is the doorway into regular expressions, which every serious Linux user eventually learns.

### 4.5 `more` — controlling the display of a file

The `more` command displays one page of a file at a time and lets you scroll down with the `Enter` key or the space bar. It exists because a terminal only shows a fixed number of rows, and dumping four thousand lines of configuration at it is not reading, it is scrolling in reverse.

```bash
root@kali:~/Documents/linux-lab# more dns-mappings.txt
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

The prompt at the bottom left is the pager's control: pressing `Enter` advances one line, and pressing the space bar advances a full screen. The percentage tells you how far down the file you currently are, which is a useful orientation cue on a long file. Type `q` at the prompt to quit, `h` if you have forgotten the keys available, and `/pattern` if you want to jump forward to a match — although `more`'s searching is famously limited, which is precisely why the next command exists and why almost everyone abandons `more` after their first week.

### 4.6 `less` — displaying and filtering a file

`less` is very similar to `more` — the name is a deliberate joke — but it comes with the crucial added ability to move *backwards* as well as forwards, and to filter and search a file interactively. In practice `less` is what you should be using whenever you would have reached for `more`.

```bash
root@kali:~# less /etc/ssh/sshd_config
#	$OpenBSD: sshd_config,v 1.100 2016/08/15 12:32:04 naddy Exp $
# This is the sshd server system-wide configuration file.  See
# sshd_config(5) for more information.
...
PermitRootLogin prohibit-password
```

Once inside, press the forward slash `/` and type the keyword you want to find, then press Enter; here we searched for the string `PermitRootLogin` and the pager jumped straight to the line of the configuration that decides whether the root account may log in over SSH. Press `n` to jump to the next occurrence of that keyword and `N` to jump to the previous one — a habit worth developing, since a default configuration file usually mentions the same option twice, once commented out as documentation and once as the actual setting. The other keys worth memorising are `G` to jump to the very end of the file, `g` to jump back to the beginning, `q` to quit, and the arrow or `j`/`k` keys to move a line at a time.

`less` reads its input from a file *or* from a pipe, and that second ability is what makes it a general-purpose window onto command output. `history | less`, `ls -lR /usr/share | less` and `dmesg | less` are all everyday commands, because `less` keeps the entire stream in its buffer and lets you scroll back through it long after the producer has finished. Crucially, piping into `less` does **not** lose anything: the whole output is captured, so you can search backwards through the complete result of a slow command rather than only the screenful it printed first.

### 4.7 A few more text utilities worth knowing

Three more commands round out the set. `wc` counts lines, words and characters (`wc -l file` answers "how many entries does this list have?"). `sort` arranges lines alphabetically or numerically and `uniq` collapses adjacent duplicates, and together they turn a messy stream into a summarised one:

```bash
root@kali:~/Documents/linux-lab# cut -d' ' -f1 dns-mappings.txt | grep -v '^#' | sort
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

The `cut -d' ' -f1` takes the first whitespace-separated field of every line, the `grep -v '^#'` throws away the comment lines, and `sort` puts the remaining hostnames in alphabetical order. This *filter chain* is the characteristic way Linux users work: rather than a single tool with fifty options, each tiny tool does one thing well and the pipe glues them together. `tee`, finally, is the tool for when you want to see output *and* save it at the same time — `ifconfig | tee /tmp/net.txt | grep inet` writes the full command output to a file while the filtered part continues down the pipeline to your screen.

---

## 5. Installing and removing software

Sooner or later you will need software that did not come with your distribution, and later still you will want to remove something you no longer use. On Debian-based distributions such as Debian itself, Ubuntu, Kali Linux and their derivatives, the default software manager is the **Advanced Package Tool**, or `apt` for short. The mental model is exactly the one you already have from a phone or an app store: instead of downloading installers from random websites, you have **repositories** — servers that hold a curated catalogue of software, already compiled and packaged — and a tool that knows how to search that catalogue, download from it, verify the packages and resolve their dependencies. Learning to use the repository properly is what makes Linux maintenance as easy as it is.

### 5.1 `apt-cache search` — searching the repository for a package

Before downloading any software package, it is worth checking whether it is available in the repository at all, and under which name. The catalogue of every package your machine knows about is stored locally, and `apt-cache` queries it. The form is `apt-cache search` followed by the keyword you are looking for; here we search for a well-known login-cracking tool:

```bash
root@kali:~# apt-cache search hydra
hydra - very fast network logon cracker
hydra-gtk - GTK+ based tool to do fast network logon cracking
libhydra0 - Hydra toolkit library
...
```

Several packages matched, which is typical: the first line is the actual program, the second is its graphical front end, and the third is a support library that other software depends on. Each line consists of the package name, a dash, and a short description — and that short description is exactly what `apt-cache search` matched against, which is why a search for a tool's function ("password cracker") can turn up software whose name you did not know. This command changes nothing on your system: it is a read-only query against the local copy of the catalogue, so it is always safe to run. To see much more about a candidate before installing it, use `apt-cache show hydra`, which prints the version, the download size, the dependencies, the maintainer and the long description, or the friendlier `apt show hydra` which is the modern spelling of the same thing.

### 5.2 `apt-get install` — installing packages

Now let us install the packages we want. This time we use the `apt-get` command followed by the word `install` and the package name. Here we install `git`, which will later allow us to pull repositories down to install further tools:

```bash
root@kali:~# apt-get install git
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
root@kali:~#
```

This transcript is worth reading line by line, because `apt` is unusually honest about what it intends to do before it does it. It begins by building its picture of the installed system, then announces the **additional packages** it must install alongside your request — these are the *dependencies*, and pulling them down automatically is the single biggest reason package managers exist. It reports how much data must be downloaded and how much disk space the installation will occupy, and only then does it ask `Do you want to continue? [Y/n]`. Any answer other than `y` (or `Y`, or just Enter, since the capital letter shows the default) aborts the whole thing harmlessly. After the fetch, the unpacking and configuration steps run in sequence and give you Git, ready to use.

Three flags make `install` safer and quieter in day-to-day use. `-y` pre-answers "yes" to the confirmation prompt, which is essential inside scripts but removes your last chance to read the plan; get into the habit of running a new command *without* `-y` the first time. `--no-install-recommends` installs only what is strictly required and skips the "recommended but not essential" extras, which keeps a container or a virtual machine lean. And `apt-get -f install` (or `apt --fix-broken install`) repairs a half-configured state, which is what you reach for when an interrupt or an out-of-space error has left the package system unhappy — the error message `E: Unmet dependencies. Try 'apt-get -f install'` is practically an instruction manual.

### 5.3 `apt-get remove` — removing packages

To remove any package from your machine, simply type `remove` after `apt-get`, followed by the package name. Let us remove the package we just installed:

```bash
root@kali:~# apt-get remove git
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

Note what the plan says and, more importantly, what it does not say. Only `git` itself will be removed — the dependencies that were installed with it, `git-man` and `liberror-perl`, are left behind, on the sensible assumption that they never hurt anyone and might be needed by something else. This is why removing software on Linux does not leave the system leaner in the way beginners expect: the program goes, but its libraries and, crucially, its **configuration files** remain. That last point is what the next command addresses. (For the sake of the walkthrough this step was aborted by answering `n` — as the transcript shows, `apt-get` takes you at your word and stops immediately, which is a safe way to preview exactly what a removal would do without doing it.)

### 5.4 `apt-get purge` — removing packages completely

Sometimes the package we just removed leaves residual files behind, configuration files being the classic example. In order to wipe everything out cleanly, we use `purge` with `apt-get`. As before, the operation is previewed before it happens and can be aborted:

```bash
root@kali:~# apt-get purge git
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following packages will be REMOVED:
  git*
0 upgraded, 0 newly installed, 1 to remove and 0 not upgraded.
After this operation, 38.9 MB disk space will be freed.
Do you want to continue? [Y/n]
```

The star after the package name in the plan is `apt`'s way of signalling that configuration files — the items that `remove` deliberately spared — are included in the removal this time. That difference matters more than it sounds: a `purge` followed by a fresh `install` gives you a genuinely default configuration, whereas a `remove` followed by `install` gives you your old settings back, which is either exactly what you want or exactly the source of an afternoon of confusion, depending on the day. Two related commands complete the set: `apt-get autoremove` deletes the dependencies that were pulled in for software you have since removed and that nothing else needs any more, and `apt-get clean` empties the downloaded `.deb` files out of the package cache under `/var/cache/apt/archives`, which on a long-lived machine can be worth hundreds of megabytes.

### 5.5 `apt-get update` — refreshing the catalogue

It is good practice to update the repository information regularly, since the repositories are usually updated with new software and newer versions of existing software. These updates have to be requested, and that is done by typing `update` after `apt-get`:

```bash
root@kali:~# apt-get update
Hit:1 http://kali.download/kali kali-rolling InRelease
Get:2 http://kali.download/kali kali-rolling/main amd64 Packages [19.6 MB]
Get:3 http://kali.download/kali kali-rolling/main amd64 Contents (deb) [45.1 MB]
Fetched 64.7 MB in 12s (5,392 kB/s)
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
```

The key sentence to understand is that **`update` does not install anything at all**. What it does is re-download the *lists* — the catalogue of which packages exist, in which versions, with which dependencies — so that your machine's picture of the repositories matches reality. `Hit` means the file was already up to date and was not downloaded again, while `Get` means new data was fetched. This is why the command is quick and harmless, and also why it is the first step of *every* recommended installation procedure: without a fresh catalogue, `apt-get install` may try to fetch a version that has since been replaced, and you get the notorious `404 Not Found` from the mirror. Pairing it with the next command gives the standard `apt-get update && apt-get upgrade` one-liner, in which `&&` means "run the second command only if the first one succeeded".

### 5.6 `apt-get upgrade` — applying available updates

In order to actually apply the changes described by the catalogue you just refreshed, you run `apt-get` with `upgrade`. This installs the newer versions of everything that is already on the system:

```bash
root@kali:~# apt-get upgrade
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

Note the framing: the plan lists six *upgrades*, not installs, because `upgrade` never removes a package and never installs a new one — it only replaces what is already there with a newer version. Compare the list with the three that follow it in the sample above: `libc6` and `openssl` and `sudo` are precisely the kind of package a security practitioner cares about, because an out-of-date crypto library or authentication tool on a server is a finding waiting to be written up. Upgrading can be time-consuming, so plan for it rather than starting it five minutes before you need the machine: when the kernel itself is upgraded, the new version only takes effect after a reboot.

A related command is `apt-get dist-upgrade` (or the modern `apt full-upgrade`), which is allowed to *remove* and *add* packages as required to complete a complex upgrade, and `apt-get upgrade`'s refusal to do so is exactly why it occasionally reports that some packages are "kept back". And one caution that belongs in any discussion of updating: **never upgrade a machine you are in the middle of breaking into or assessing**. A package upgrade can restart services, change a library version that a tool depends on, or overwrite a configuration file you modified, all of which is fine on your own workstation and catastrophic mid-engagement. Snapshot the target, or work on a copy, before you touch the package manager.

### 5.7 Adding repositories to `/etc/apt/sources.list`

The servers that hold the software catalogue for a particular distribution are known as *repositories*, and the list of repositories your machine is allowed to use lives in a plain text file at `/etc/apt/sources.list` (with additional entries, on modern systems, in the directory `/etc/apt/sources.list.d/`). To change that list you open it in a text editor — `nano` is the beginner-friendly one — and add, remove or comment out the lines you want:

```bash
root@kali:~# nano /etc/apt/sources.list
```

Inside the editor you would see something like the following, with one repository per line, each line consisting of the archive type (`deb` for binary packages, `deb-src` for source code), the URL of the mirror, the distribution name and the components:

```
deb http://kali.download/kali kali-rolling main contrib non-free non-free-firmware
# deb-src http://kali.download/kali kali-rolling main contrib non-free
```

Working in `nano` is deliberately simple: type your text directly, use the arrow keys to move around, and use the shortcut bar along the bottom of the screen for everything else. `Ctrl+O` then Enter saves the file, `Ctrl+W` searches for a string (the fastest way to find the right line in a long file), `Ctrl+K` cuts the current line and `Ctrl+U` pastes it back, and `Ctrl+X` leaves the editor, asking whether you want to save if you have unsaved changes. Once the file is edited, `apt-get update` must be run again before the new repositories are usable, because the package lists are built from the lines you just changed. A word of warning that belongs with this section: add only repositories you actually trust. An experimental or third-party repository can push packages that conflict with your distribution's own, and a mistyped URL pointing at a stranger's mirror is a straightforward way to install someone else's code as root. One incorrect line in this file is also enough to make `apt-get update` fail with a GPG or `404` error, so when an update does go wrong, read the message carefully and check the line number it names.

---

## 6. Understanding and playing with permissions

Before we learn the commands that manipulate permissions, we need to know what a permission *is*, because Linux takes them far more seriously than a desktop operating system does. A beginner's mental model is usually enough to get started: every file and directory on the system has an **owner** (a user) and a **group**, and every access check the kernel performs is a question about the identity of the process asking, checked against those two attributes and one of three permission classes.

The **root** user is, as we have seen, all-powerful; the kernel skips the ordinary permission checks for it entirely, which is why it can read any file on the system. Every other user has limited capabilities, and users are usually collected into **groups** that share a function, for example a separate group for the developer team, the deployment team and the administrators, so that different levels of access can be granted to each. A file's permissions are then granted at three levels — the **owner** of the file, the **group** that owns it, and **everyone else** (often called *others* or *the world*) — and at each of those three levels, three specific rights can be granted or withheld:

* **read (`r`)** — permission to open and view the contents of the file. On a *directory*, read means the ability to list the names of the entries inside it.
* **write (`w`)** — permission to modify the file (or, in the case of a directory, to create, rename and delete entries inside it).
* **execute (`x`)** — permission to run the file as a program. A script or binary with read permission but no execute permission will be listed, can be read, and will simply refuse to run. On a *directory*, `x` means the right to traverse it, that is, to pass through it and reach files below it — which is why the common trouble-shooting case of "I can list the directory but I cannot `cd` into it" is a missing `x` rather than a missing `r`.

### 6.1 Reading a permission string with `ls -l`

As we saw earlier, the long listing `ls -l` is how you inspect these settings, so let us look at one again now that we know what we are looking for:

```bash
root@kali:~/Documents/linux-lab# ls -l
total 12
-rw-r--r-- 1 root root   82 Oct  8 11:45 practice-notes.txt
-rwxr-xr-x 1 root root 4096 Oct  8 12:02 backup.sh
drwxr-xr-x 2 root root 4096 Oct  8 12:01 documents
```

Take the second line apart character by character, because every character means something:

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

So `backup.sh` is readable, writable and executable by its owner (`rwx`), readable and executable by members of the owning group (`r-x`), and readable and executable by everyone else (`r-x`) — a normal state for a script that everyone on the machine is meant to be able to run. Compare it with `practice-notes.txt`, whose `-rw-r--r--` means the owner can read and write but nobody, including the owner, can execute it: it is a text file, and there is no reason for it to be executable at all. The directory `documents` shows the leading `d`, and directories almost always carry `x` for everyone, because without it nobody could descend into them.

### 6.2 `chown` — granting ownership to an individual user

We change the ownership of a file so that the new user who owns it gains control over its permissions, and `chown` is the command that does it: `chown <new owner> <file>`. Here we transfer ownership of our notes file from the root account to a normal user called `student`:

```bash
root@kali:~/Documents/linux-lab# ls -l practice-notes.txt
-rw-r--r-- 1 root root 82 Oct  8 11:45 practice-notes.txt
root@kali:~/Documents/linux-lab# chown student practice-notes.txt
root@kali:~/Documents/linux-lab# ls -l practice-notes.txt
-rw-r--r-- 1 student root 82 Oct  8 11:45 practice-notes.txt
```

The third column of the listing changed from `root` to `student`, and with that single field the *meaning* of the permission string changed as well. The `rw-` at the start of `-rw-r--r--` used to describe root's rights over the file; it now describes the rights of `student`, who can write to the file, while root retains the ability to do anything without needing a permission bit to say so. Only root (or a user with `CAP_CHOWN`) may give a file away, and that restriction is deliberate: if any user could hand ownership of their files to someone else, the accounting of who is responsible for what would be meaningless. Adding a colon also sets the group in the same operation, so `chown student:developers practice-notes.txt` changes owner and group at once, and `-R` applies the change recursively to an entire tree, which is the usual form when you move a web directory between accounts: `chown -R www-data:www-data /var/www/html`.

### 6.3 `chgrp` — granting ownership to a group

To transfer ownership of a file to a group we use `chgrp`, which has exactly the same shape as `chown` but takes a group name. To ensure that only members of a particular team can claim the owning group of the file, here we change the group to `developers`:

```bash
root@kali:~/Documents/linux-lab# chgrp developers practice-notes.txt
root@kali:~/Documents/linux-lab# ls -l practice-notes.txt
-rw-r--r-- 1 student developers 82 Oct  8 11:45 practice-notes.txt
```

The fourth column now reads `developers` where it previously read `root`. The practical value of group ownership is that it lets you share a file with a *set* of people without opening it to everyone: the middle triad `r--` means every member of `developers` may read the file, while the last triad continues to protect it from unrelated users. Two details are worth remembering. First, you may only change the group to one that you yourself are a member of unless you are root — otherwise you could grant access you do not hold. Second, when sharing a directory between several people, the **set-group-ID** bit on the directory (see 6.6) makes new files automatically inherit the directory's group, which is what saves you from having to `chgrp` everything by hand afterwards. To see which groups your own account belongs to, run `groups`, and to see the same with numeric IDs, `id`.

### 6.4 `chmod` — changing permissions

We use `chmod` — *change mode* — to alter the permissions of a file. It understands two notations, and you should be comfortable with both because other people's documentation mixes them freely.

The first is **symbolic**, where you name the class, the operation and the right: `u` for user/owner, `g` for group, `o` for others and `a` for all three; `+` to add, `-` to remove and `=` to set exactly; and `r`, `w`, `x` for the rights. Thus `chmod g+w report.txt` adds write permission for the group, `chmod o-r secret.txt` removes read permission from everyone outside the owner and group, and `chmod u+x,g+x script.sh` adds execute permission for two classes at once.

The second is **numeric (octal)**, in which each class is represented by a single digit built from the rights it holds: read is worth 4, write is worth 2 and execute is worth 1, and their sum is the digit. This table is worth memorising, because it is the form you will see most often in guides and scripts:

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

You then give three digits, one per class, in the order **owner, group, others**. So `chmod 644 file` produces `rw-r--r--` (6 = read+write for the owner, 4 = read for the group, 4 = read for others), `chmod 755 file` produces `rwxr-xr-x`, and `chmod 600 file` produces `rw-------`, which is the correct setting for anything containing private data such as an SSH private key or a credentials file. To give a file everything you can run `chmod 777 file`, which makes it readable, writable and executable by absolutely everyone, while `chmod 111 file` grants execute permission and nothing else — technically enough to run a binary, though the program may not be able to read its own data files, which is why `111` is much rarer in practice than `755`.

Another way of arriving at the same result, as seen below, is the symbolic `chmod +x`, with no class letter in front of it, which means "the same change for every class":

```bash
root@kali:~/Documents/linux-lab# cp backup.sh script-demo.sh
root@kali:~/Documents/linux-lab# ls -l script-demo.sh
-rw-r--r-- 1 root root 4096 Oct  8 12:04 script-demo.sh
root@kali:~/Documents/linux-lab# chmod +x script-demo.sh
root@kali:~/Documents/linux-lab# ls -l script-demo.sh
-rwxr-xr-x 1 root root 4096 Oct  8 12:05 script-demo.sh
```

Before the change, the file was `-rw-r--r--`: a text file you could read but not run. Afterwards it is `-rwxr-xr-x`, and the file's colour in the terminal listing changes from white to green, indicating that it is now executable. A useful refinement of the same idea is `chmod -R`, which applies a permission change to every file and directory below a path, and `chmod -v`, which prints a line for each object it modified — invaluable when a recursive change touches two thousand files and you want to be sure the command matched what you intended.

One more piece of the permission story deserves a paragraph here even though it is not a `chmod` switch: the **umask**. When a program creates a new file, the kernel starts from a default of `666` for files and `777` for directories and subtracts the umask to get the result — the typical umask is `022`, which is why new files appear as `644` and new directories as `755`. That is exactly why the `touch` in section 3 produced `-rw-r--r--` without anyone asking for it. Run `umask` to see your current value and `umask 077` to make everything you create from that point onwards private to you, which is a sensible setting on a shared machine.

### 6.5 Setting the SUID bit

The SUID bit — the *set user ID* bit — is a special permission that says something quite different from `rwx`. When it is set on an executable file, any user may execute that file, and while it runs, the process carries the *permissions of the file's owner* rather than those of the person who started it. Crucially, those elevated rights apply only to that particular program's execution and do not extend beyond it: the caller does not become root, they merely run this one binary with the owner's authority.

To set the SUID bit with the numeric notation, we enter **4 before the regular three permission digits**. So a file whose permissions were `644` — write and read for the owner, read for everyone else — becomes `4644` once SUID is added:

```bash
root@kali:~/Documents/linux-lab# ls -l script-demo.sh
-rwxr-xr-x 1 root root 4096 Oct  8 12:05 script-demo.sh
root@kali:~/Documents/linux-lab# chmod 4644 script-demo.sh
root@kali:~/Documents/linux-lab# ls -l script-demo.sh
-rwSr--r-- 1 root root 4096 Oct  8 12:06 script-demo.sh
```

Read the result carefully, because the SUID bit does not appear as an extra character — it *overwrites* the owner's execute slot, turning the `x` into an `s`. Three things follow from that. When the owner's execute bit is *also* set, you see a lowercase `s` as in `-rwsr-xr-x` (the normal appearance of a system binary such as `passwd` or `su`). When SUID is set but the owner's execute bit is **not** set, as in the `-rwSr--r--` above, you see an uppercase `S`, and the file is *not* executable at all — so the SUID bit changed nothing except the appearance, which is why `chmod 4644` on a text file is a harmless curiosity rather than an escalation. To actually combine SUID with an executable file you want `4755`, giving `-rwsr-xr-x`. In symbolic notation the same setting is written `chmod u+s file`, and it is removed with `chmod u-s file`, or cleared entirely by returning to three digits such as `chmod 755 file`.

One caveat specific to Linux deserves to be stated plainly, because many older tutorials skip it: the kernel honours SUID only on *compiled* binaries, and it deliberately ignores the bit on shell scripts and other interpreted programs. A text file or a `#!` script with the SUID bit set will run with the privileges of whoever started it, not of its owner — which is a safety feature, since a script's behaviour depends on a interpreter that an attacker might influence through `PATH`. So setting the bit on a script is a useful exercise in reading permission strings, while real SUID privilege-escalation targets are always binaries and shared objects.

The reason this bit matters so much in security work is that a root-owned SUID binary is a legitimate, administrator-created way for unprivileged users to perform privileged actions, and every such program is therefore a piece of code worth auditing: if it can be tricked into running a program of the attacker's choosing — a shell, for instance — that program inherits its root authority. That is the classic *SUID privilege escalation* path, and finding candidates is one command away:

```bash
root@kali:~# find / -perm -4000 -type f 2>/dev/null
/usr/bin/passwd
/usr/bin/su
/usr/bin/sudo
/usr/bin/mount
/usr/bin/newgrp
/usr/bin/chsh
```

The `-perm -4000` here means "has the SUID bit set among its permissions" — the same 4 prefix you used with `chmod` — and the listing above is what a healthy Debian-family system looks like. Any *unexpected* entry on that list is worth investigating immediately, because it is either a package you did not know you had or something that got there without your knowledge.

### 6.6 Setting the SGID bit

Similarly to SUID, the **SGID** bit — *set group ID* — grants temporarily elevated permission, but with respect to the file owner's **group** rather than the owner. On an executable it means the process runs with the privileges of the file's owning group; on a directory it has a quite different and very useful meaning, which is that new files created inside the directory inherit the directory's group instead of the creator's primary group, and new subdirectories inherit the bit as well.

To set SGID with the numeric notation, we enter **2 before the regular permissions**, so our example file with base permissions 644 becomes `2644`:

```bash
root@kali:~/Documents/linux-lab# ls -l script-demo.sh
-rwxr-xr-x 1 root root 4096 Oct  8 12:06 script-demo.sh
root@kali:~/Documents/linux-lab# chmod 2466 script-demo.sh
root@kali:~/Documents/linux-lab# ls -l script-demo.sh
-rw-rwSrw- 1 root root 4096 Oct  8 12:07 script-demo.sh
```

The same overwriting trick is at work here as with SUID, but in the middle triad: SGID occupies the *group's* execute slot and shows as a lowercase `s` when execute is also granted (`rwxr-sr-x`) and as an uppercase `S` when it is not (`rw-rwSrw-`, as above, where the file is not executable for its group). Symbolically the notation is `chmod g+s directory`, and the practical demonstration in a shared workspace is:

```bash
root@kali:~# chmod g+s /srv/share
root@kali:~# ls -ld /srv/share
drwxr-sr-x 4 root developers 4096 Oct  8 12:09 /srv/share
```

Any file that any member of `developers` now creates in `/srv/share` will belong to the group `developers` automatically, which is exactly what you want when a team has to work in the same directory and nobody wants to run `chgrp` after every save. It is worth knowing that SGID on directories is common and benign, while SGID on executables is rare and warrants the same audit question as SUID above; and the same kernel caveat applies, in that the bit is ignored on interpreted scripts and honoured on compiled binaries.

### 6.7 The third special bit: the sticky bit

There is one more special bit and it is worth naming for completeness, because it is the one you meet without noticing. The **sticky bit**, written as **1** before the regular permissions (`chmod 1777`), is irrelevant on files on modern Linux but very meaningful on directories: inside a sticky directory, users may create files and may delete only *their own* files, even if the directory itself is world-writable. The canonical example is the system's temporary directory:

```bash
root@kali:~# ls -ld /tmp
drwxrwxrwt 21 root root 4096 Oct  8 12:10 /tmp
```

The final `t` in `drwxrwxrwt` is the sticky bit in its open, executable form (it would appear as an uppercase `T` if the directory were not searchable). Without it, any user could delete any other user's temporary files in `/tmp`, which would make every program on the machine a victim of every other program. Symbolically the setting is `chmod +t directory`, and if you ever see a world-writable directory *without* the sticky bit, that is a genuine finding rather than a quirk: it means every user on the system can destroy every other user's work. You can find them with `find / -type d -perm -0002 ! -perm -1000 2>/dev/null`, which reads as "directories that are writable by others and do not have the sticky bit".

### 6.8 Putting it together: a worked permissions scenario

To see how the pieces combine, imagine a small shared workspace. We want a directory that members of a team can all write into, files created there to inherit the team group, a script that anyone may run, and a private note that only its owner may read:

```bash
root@kali:~# mkdir /srv/share
root@kali:~# chgrp developers /srv/share
root@kali:~# chmod 2775 /srv/share
root@kali:~# ls -ld /srv/share
drwxrwsr-x 2 root developers 4096 Oct  8 12:12 /srv/share
root@kali:~# cp backup.sh /srv/share/backup.sh
root@kali:~# chmod 755 /srv/share/backup.sh
root@kali:~# chown root:developers /srv/share/backup.sh
root@kali:~# touch /srv/share/private.key && chmod 600 /srv/share/private.key
root@kali:~# ls -l /srv/share
total 8
-rwxr-xr-x 1 root developers 4096 Oct  8 12:13 backup.sh
-rw------- 1 root developers    0 Oct  8 12:13 private.key
```

Reading the result from the outside in: the directory is `drwxrwsr-x` — group-writable so that the team can add files, and carrying the SGID bit so that everything they add stays in the `developers` group rather than picking up each user's personal group. The script is `755`, so anyone may run it but only its owner may rewrite it — the standard shape for a tool that other people are meant to use but not modify. The key file is `600`, readable and writable by its owner and by nobody else, which is the correct setting for anything secret and is also the setting that SSH will *insist* on before it agrees to use a private key. Four commands — `mkdir`, `chown`, `chmod` and `cp` — and a share is configured correctly. That is how much of Linux system administration actually works.

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

The commands in this guide only become yours when you have typed them yourself, so here is a short sequence that covers everything above in one coherent flow. Work on a virtual machine or in a container, and use a throwaway user account for the permission parts.

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

The command line rewards patience and punishes haste, and almost every mistake a beginner makes falls into one of three categories: not knowing which directory they are in, not knowing which user they are running as, and not reading the error message that would have explained the problem. The first two are answered by `pwd` and `whoami` before every risky command; the third is a habit of attention rather than a command. Everything else in this guide is syntax, and syntax is what a manual page is for.

Two habits are worth building from the first week. The first is to *read the plan* before you approve it — `apt-get` tells you exactly what it will remove, `find -exec` can be tested with `-print`, and destructive commands can be previewed by prefixing them with `echo` so the shell shows you what it intends to run. The second is to *write things down*: keep a plain text file of the commands you had to look up, and after a few weeks you will find that you have written your own cheat sheet, in your own words, which is worth more than any guide you can download.

<<<<<<<< HEAD:docs/01 - Linux for Beginners - Part 1.md
**Part 2** is the continuation and it picks up exactly where this guide leaves off. [Linux for Beginners (Part 2): Networks, Processes and Environment Variables](02%20-%20Linux%20for%20Beginners%20-%20Part%202.md) covers reading and reconfiguring the interfaces a machine uses to talk to the network, watching and controlling the processes it runs, and managing the environment variables that every program you launch inherits. **[Part 3: Bash Scripting, Automation and Services](03%20-%20Linux%20for%20Beginners%20-%20Part%203.md)** then turns all of it into automation: a Bash script that does a day's work in a second, a `cron` job that runs it at 23:55 without being asked, and the services — Apache, OpenSSH, FTP — that let a machine answer requests on its own. Beyond the three parts, natural next steps are **service management** with `systemctl` and the systemd journal, and a deeper study of the **permission and privilege model** that this guide has only introduced.
========
**Part 2** is the immediate continuation and it picks up exactly where this guide leaves off. [Linux for Beginners (Part 2): Networks, Processes and Environment Variables](02-linux-for-beginners-part-02.md) covers reading and reconfiguring the interfaces a machine uses to talk to the network, watching and controlling the processes it runs, and managing the environment variables that every program you launch inherits. After that, [Linux for Beginners (Part 3): Scripting, Scheduling and Services](03-linux-for-beginners-part-03.md) covers shell **scripting and automation** — conditions, loops, functions and variables, assembled into programs that do a day's work in a second — along with **scheduling** with `cron` and **service management** with `service`/`systemctl`, including the Apache web server, OpenSSH and FTP. Beyond the three parts, natural next steps are writing your own systemd units and timers, configuration management across more than one machine, and a deeper study of the **permission and privilege model** that this guide has only introduced.
>>>>>>>> origin/arena/52f71d52-attack-scripts:docs/01-linux-for-beginners-part-01.md
