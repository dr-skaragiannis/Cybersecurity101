# Linux for Beginners: A Hands-On Guide to the Command Line

## Introduction

More often than not, certain operating systems tend to get tied to certain tasks, and when the task is penetration testing, a Linux-based operating system is almost always the platform of choice. This guide is written for someone who has never opened a Linux terminal before and wants to become comfortable with the fundamentals in a single sitting. Rather than dumping a list of commands on you, every section explains *why* a command exists, *what* it does to your system, and *how* you can verify with your own eyes that it actually did it.

The material follows a friendly difficulty curve. We begin with the two questions every beginner asks out loud ("where am I?" and "who am I?") and with the commands that move you around the file system and list what is inside it. From there we look at the built-in help systems, because no one memorises every flag of every utility, and learning to read a manual page is a skill that pays for itself immediately. Next comes file and directory manipulation, the everyday business of creating, copying, moving and deleting things; then text manipulation, which matters far more on Linux than on other systems because almost everything you administer here is a plain text file. After that we cover installing and removing software through the package manager and understanding the Unix permission model, the part of Linux that trips up beginners most often and the part that matters most when you later study privilege escalation. The second half of the guide then moves on to the layer that sits just beneath everyday use: **networks and interfaces**, **process management**, and the **environment variables** that quietly decide which program runs, which resolver answers and which editor opens. Each of those three topics is a doorway into system administration and into security work, and none of them require anything more than the commands already covered.

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
7. [Managing networks](#7-managing-networks)
8. [Process management](#8-process-management)
9. [User environment variables](#9-user-environment-variables)
10. [Quick reference cheat sheet](#10-quick-reference-cheat-sheet)
11. [Practice exercises](#11-practice-exercises)

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

## 7. Managing networks

Networking is a crucial topic for anyone heading towards security work, because a great deal of what you will eventually be asked to test lives on the network rather than on a single machine. Even for everyday administration, you cannot get far without knowing how to look at an interface, read the address it holds, understand which server is answering your name lookups, and change any of those things deliberately. This section works through the tools that do exactly that, in the order you would reach for them: inspect an interface, inspect a wireless interface, change an address, spoof a hardware address, ask a DHCP server for a lease, interrogate DNS, change which resolver you use, and override a name locally.

One note before we start, because it explains why some of the commands below look old-fashioned. The traditional tools are `ifconfig`, `iwconfig`, `route` and `netstat`, and they come from a package called `net-tools` that many modern distributions no longer install by default. They have been superseded by the `ip` command (from the `iproute2` suite) and by `ss`. Every traditional command in this section is therefore accompanied by its modern equivalent, and the habit worth forming is to read both: legacy documentation, older tutorials and interview questions are full of `ifconfig`, while every current distribution ships `ip`.

### 7.1 `ifconfig` — analysing the network interfaces

The `ifconfig` command is the most basic tool for interacting with active network interfaces. Run with no arguments it prints every interface the kernel knows about, which on a typical lab machine means a physical or virtual Ethernet adapter plus the loopback interface.

```bash
root@kali:~# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.17  netmask 255.255.255.0  broadcast 192.168.1.255
        inet6 fe80::20c:29ff:fe1b:2c3d  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:1b:2c:3d  txqueuelen 1000  (Ethernet)
        RX packets 10422  bytes 9012443 (8.5 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 5120  bytes 774211 (756.0 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loopback  txqueuelen 1000  (Local Loopback)
        RX packets 118  bytes 9412 (9.1 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 118  bytes 9412 (9.1 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

Two interfaces are listed, and reading them properly is the skill this section teaches. The first line of each block names the interface and its flags: `UP` means the interface is administratively enabled, `RUNNING` means the driver reports a carrier (a cable is plugged in, or the virtual switch is connected), `BROADCAST` and `MULTICAST` describe the traffic the interface will accept, and `mtu 1500` is the maximum transmission unit — the largest payload the link will carry in one frame, which matters when you are diagnosing fragmentation problems. The loopback interface has an `mtu` of 65536 because it is not a real wire; nothing has to fit into an Ethernet frame when the packets never leave the machine.

The second line of each block carries the addressing. `inet 192.168.1.17 netmask 255.255.255.0 broadcast 192.168.1.255` is IPv4: the address, the mask that defines which part of the address is the network and which part the host, and the address that reaches every machine on the local segment. From those two numbers you can derive the rest of the subnet mentally, which is a skill worth practising: a `/24` mask (255.255.255.0) with the address 192.168.1.17 means the network is 192.168.1.0, the usable host range is .1 to .254, and the broadcast is .255. The `inet6` line is the automatically generated IPv6 link-local address, recognisable by its `fe80::` prefix, which is present even on networks where nobody has configured IPv6 — one of the reasons an IPv6-blind firewall is a real problem.

The third line is the link layer. The `ether` field is the MAC address, the hardware identifier burned into the network card (or, on a virtual machine, assigned by the hypervisor), and `00:0c:29` identifies this one as a VMware virtual adapter. MAC addresses are discussed in detail in section 7.4, because they can be changed. After it, `txqueuelen` is the length of the transmit queue, a tuning parameter you will rarely need to touch.

The rest of the block is statistics. `RX` and `TX` count packets and bytes received and transmitted since the interface came up, the `errors`, `dropped`, `overruns`, `carrier` and `collisions` counters track things that went wrong, and they are the first place to look when a link is flaky: a steadily climbing `errors` or `dropped` count on a physical interface usually means a bad cable, a failing port or a duplex mismatch. All of these counters reset when the interface is brought down and up again.

The modern equivalent of this command is `ip addr` (often typed `ip a`), which shows the same information in a different layout:

```bash
root@kali:~# ip -br addr
lo               UNKNOWN        127.0.0.1/8 ::1/128 
eth0             UP             192.168.1.17/24 fe80::20c:29ff:fe1b:2c3d/64
root@kali:~# ip route
default via 192.168.1.1 dev eth0 proto dhcp metric 100 
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.17 metric 100 
```

The `-br` flag asks for *brief* output, one line per interface, which is what you want when you only care about the address. The second command, `ip route`, prints the routing table — the set of rules that decides where a packet goes — and the key line is `default via 192.168.1.1 dev eth0`, which names the gateway: the router that packets are sent to when their destination is not on the local subnet. Knowing the gateway is not optional knowledge, because the default route is the thing that makes the difference between a machine that can reach the internet and a machine that can only talk to its neighbours even though `ifconfig` looks perfectly healthy.

### 7.2 `iwconfig` — checking wireless network devices

If the machine has a wireless adapter, `iwconfig` is the traditional tool for reading its radio-specific settings: the network it is associated with, the access point it is talking to, the frequency, the transmission rate and the signal quality.

```bash
root@kali:~# iwconfig
lo        no wireless extensions.

eth0      no wireless extensions.

wlan0     IEEE 802.11  ESSID:"LAB-AP"  
          Mode:Managed  Frequency:2.437 GHz  Access Point: 3C:37:86:1A:2B:40   
          Bit Rate=72.2 Mb/s   Tx-Power=20 dBm   
          Retry short limit:7   RTS thr:off   Fragment thr:off
          Encryption key:off
          Power Management:off
          Link Quality=58/70  Signal level=-52 dBm  
          Rx invalid nwid:0  Rx invalid crypt:0  Rx invalid frag:0
          Tx excessive retries:0  Invalid misc:0   Missed beacon:0
```

On a machine with no wireless hardware — which is the common case on a virtual machine — every interface reports `no wireless extensions`, and that is the expected output rather than an error: Ethernet adapters have no radio settings to report. On a machine that does have a wireless card, the block above is what you get, and three fields are worth understanding. `ESSID:"LAB-AP"` is the name of the wireless network the card has joined, and `Mode:Managed` means it is behaving as a normal client. The `Access Point` field is the MAC address of the wireless router the card is associated with, which is useful for physical-location work and for detecting a rogue access point that is impersonating your network. `Link Quality=58/70  Signal level=-52 dBm` is the radio health: signal strength in dBm where values closer to zero are better, so −52 dBm is a strong, comfortable connection and anything worse than about −75 dBm will start to produce dropped packets.

The security-relevant fields are `Encryption key:off`, which tells you whether the link is using WEP, WPA or nothing at all, and the counters at the bottom. `Rx invalid crypt` counts frames that failed their integrity check, which on an open network is simply noise, but on an encrypted one can indicate interference or an attack in progress. The `iw` command is the modern replacement (`iw dev wlan0 link` prints the association details), and `nmcli device wifi list` is the higher-level tool if NetworkManager is managing the radio.

### 7.3 Changing an IP address

An interface's address can be set by hand, and doing so is a one-line command: `ifconfig`, then the interface to change, then the address you want it to have. Here the Ethernet interface is given the address 192.168.1.13.

```bash
root@kali:~# ifconfig eth0 192.168.1.13
root@kali:~# ifconfig eth0
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.13  netmask 255.255.255.0  broadcast 192.168.1.255
        ether 00:0c:29:1b:2c:3d  txqueuelen 1000  (Ethernet)
        RX packets 10430  bytes 9012611 (8.5 MiB)
        TX packets 5121  bytes 774311 (756.1 KiB)
root@kali:~# ip -br addr show eth0
eth0             UP             192.168.1.13/24 fe80::20c:29ff:fe1b:2c3d/64
```

The address changed immediately and the second command confirms it: `inet 192.168.1.13` where the previous transcript showed 192.168.1.17. This works because the kernel does not ask permission from anything — an address is local state, so assigning one takes effect the moment you type it. There is no negotiation in the manual case, no DHCP lease, and no notification to the router, which has consequences worth spelling out.

The first consequence is that the change is **temporary**. Nothing has been written to a configuration file, so a reboot — or a restart of the networking service — returns the interface to whatever its configuration says. If the machine is managed by NetworkManager or systemd-networkd, those services may overwrite your change within seconds or on the next reconnect. The second consequence is that `ifconfig eth0 <address>` does not touch the routing table, so the default route may still point out of the interface with a source address that no longer exists, and connectivity can break in a confusing way: the interface looks correctly configured yet nothing routes. The safe pattern, and the one used on machines you administer permanently, is to edit the configuration (a Netplan YAML file on Ubuntu, a connection profile under `/etc/NetworkManager/system-connections/`, or `/etc/network/interfaces` on Debian systems without NetworkManager) and then reload the networking service so that addresses, routes and DNS are all recalculated together.

If you are working over SSH when you change an address, be aware that you are almost certainly cutting the branch you are sitting on: the moment the source address of the connection changes, the session dies. The standard safety measures are to work from a console, to add the new address *alongside* the old one with `ip addr add 192.168.1.13/24 dev eth0` (the modern command, which adds rather than replaces), or to schedule a rollback with `at` — a technique we will meet in section 8.9.

### 7.4 Spoofing a MAC address

The MAC address is not only an identifier; on many networks it *is* the access control. Wireless access points with MAC filtering, captive portals that remember devices, DHCP servers that hand out fixed leases by hardware address, and network access control systems that quarantine unknown machines all make decisions on the strength of the MAC address they see. Because that value is read from the frame's headers rather than from any hardware register at receive time, a modern network card will let the operating system present whatever value you ask for. Spoofing is therefore trivial, and it neutralises every one of those controls.

Changing it takes three commands, because most drivers will not accept a new hardware address while the interface is carrying traffic: bring the interface **down**, set the address, and bring it back **up**.

```bash
root@kali:~# ifconfig eth0 down
root@kali:~# ifconfig eth0 hw ether 00:11:22:33:44:55
root@kali:~# ifconfig eth0 up
root@kali:~# ifconfig eth0
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.17  netmask 255.255.255.0  broadcast 192.168.1.255
        ether 00:11:22:33:44:55  txqueuelen 1000  (Ethernet)
        RX packets 2  bytes 168 (168.0 B)
        TX packets 3  bytes 252 (252.0 B)
```

The `hw ether` keyword means "the hardware address of type Ethernet", and the argument that follows is the new value. The verification shows `ether 00:11:22:33:44:55` in place of the original VMware address, and note what else changed: the `RX` and `TX` counters have reset to almost nothing, because downing the interface clears them and starting it again begins new ones. It is also worth noting that the interface came back with its configured IPv4 address and its routes, because those are applied by the network manager when the link comes up — a manual configuration inside a live system behaves differently from one made while the interface is down.

A few practical points. The modern equivalent is `ip link set dev eth0 address 00:11:22:33:44:55`, with `ip link set eth0 down` and `up` around it. Not every driver accepts a spoofed address, and not every address is accepted: a multicast address (the least significant bit of the first octet set), or a value whose first octet is odd in the wrong place, may be rejected with `SIOCSIFHWADDR: Cannot assign requested address`. Choose a unicast address — the second-least-significant bit of the first octet must be clear — and, for anything that needs to look legitimate, an address with a real vendor prefix.

Changing your MAC is not invisibility. On a switched network the frame is delivered by MAC, but the switch *records* which port each address appears on, so spoofing an address that another device is already using creates a flapping-port condition that switch management software will log and many will block. Spanning-tree protection features such as port security and DHCP snooping, and any decent network access control deployment, are built precisely to catch this. And if your goal is to *defeat* MAC-based access control in an assessment, the honest finding to write up is not "I spoofed a MAC" but "the network treats a spoofable identifier as an authentication factor, which constitutes no authentication at all".

### 7.5 Getting a fresh address from DHCP with `dhclient`

Manual addressing is useful for a lab, but most machines do not stand still: they ask the local DHCP server for an address every time they connect. DHCP — the Dynamic Host Configuration Protocol — hands out addresses, and usually a subnet mask, a gateway and DNS servers along with them, from a pool that the administrator controls. On a client machine the request is made by the DHCP client daemon, which on many Linux systems is `dhclient`.

```bash
root@kali:~# dhclient eth0
root@kali:~# ifconfig eth0
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.17  netmask 255.255.255.0  broadcast 192.168.1.255
        ether 00:11:22:33:44:55  txqueuelen 1000  (Ethernet)
        RX packets 421  bytes 38112 (37.2 KiB)
        TX packets 388  bytes 74520 (72.8 KiB)
```

Running `dhclient` with an interface name starts the client against that interface, and the address has now returned to `192.168.1.17` — the value the server had allocated to this interface before we typed 192.168.1.13 over the top of it. The machine is back to a server-managed state, and the previously configured address is gone from the interface although the manually assigned address may still survive in the DHCP client's own configuration file until it is rewritten.

Most of the useful detail is not printed on the console but written to the lease file, which records exactly what the server offered and when it expires:

```bash
root@kali:~# grep -A3 "lease" /var/lib/dhcp/dhclient.eth0.leases | head -20
lease {
  interface "eth0";
  fixed-address 192.168.1.17;
  option subnet-mask 255.255.255.0;
  option routers 192.168.1.1;
  option domain-name-servers 192.168.1.1;
  option dhcp-lease-time 3600;
  option dhcp-message-type 5;
  renew 4 2026/10/09 13:20:11;
  rebind 4 2026/10/09 13:50:11;
  expire 4 2026/10/09 14:05:11;
}
```

That block is the client's record of the conversation: the address it holds, the mask, the gateway (`option routers`), the DNS servers it was told to use, the length of the lease in seconds, and the times at which it will try to renew, rebind and eventually lose the address. Those three times define the DHCP lifecycle and they explain a common puzzle. A lease is not permanent, so a machine that has been off for a long time may return to a different address; and because the client renews at half the lease time, an address that "suddenly changed" usually means the server had already given it to somebody else.

Two commands complete the set. `dhclient -r eth0` *releases* the lease, telling the server that the address is no longer needed so it can be returned to the pool — useful when you want to force a completely fresh configuration. `dhclient -v eth0` runs the client verbosely and prints the four-step handshake as it happens (DISCOVER, OFFER, REQUEST, ACK), which is the clearest way to understand the protocol and the fastest way to prove that a DHCP server is reachable. On modern desktop systems all of this is managed by NetworkManager or systemd-networkd, and reaching for `dhclient` by hand on such a machine leads to the two systems fighting over the interface; `nmcli con up <connection>` and `networkctl reconfigure <link>` are the polite ways to ask.

### 7.6 Examining DNS with `dig`

DNS is the service that translates a domain name such as `example.com` into the IP address a packet can actually be sent to, and it does far more than that: it publishes which servers accept mail for a domain, which name servers are authoritative for the zone, which hosts exist, and where services such as web and VPN endpoints can be found. For anyone assessing a target, DNS is reconnaissance gold, and `dig` — the Domain Information Groper — is the tool you use to read it.

Run with a single argument, `dig` performs a lookup for an A record (an IPv4 address) and prints the whole answer, including the parts that are normally hidden:

```bash
root@kali:~# dig example.com

; <<>> DiG 9.18.24-1-Debian <<>> example.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 41233
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;example.com.			IN	A

;; ANSWER SECTION:
example.com.		3600	IN	A	93.184.216.34

;; Query time: 24 msec
;; SERVER: 192.168.1.1#53(192.168.1.1) (UDP)
;; WHEN: Fri Oct 09 12:34:56 UTC 2026
;; MSG SIZE  rcvd: 56
```

Reading from the top: the `HEADER` line reports the operation (`opcode: QUERY`), the result code (`status: NOERROR`, where NXDOMAIN means the name does not exist and SERVFAIL means the resolver itself failed), and the flags, of which `qr` (this is a response), `rd` (recursion desired) and `ra` (recursion available) are the ones you will see most. The `QUESTION SECTION` repeats the question that was asked, and the `ANSWER SECTION` holds the answer: the name, the TTL in seconds (3600, meaning resolvers may cache this for an hour), the class (`IN` for internet), the record type (`A`) and the data (`93.184.216.34`).

The last four lines are the ones beginners overlook and experts read first. `Query time: 24 msec` is how long the recursive resolution took. `SERVER: 192.168.1.1#53(192.168.1.1) (UDP)` names the resolver that actually answered *you* — here the lab gateway forwarding on your behalf, and noticing when this changes is how you confirm that your resolver configuration took effect. `WHEN` is the timestamp, and `MSG SIZE rcvd` is the size of the response, a number that becomes interesting when DNS responses grow large enough to trigger TCP fallback or fragmentation.

Different record types answer different questions, and `dig` asks for them by adding the type to the command. The `mx` type lists the mail exchangers for the domain, which is step one in mapping an organisation's mail infrastructure:

```bash
root@kali:~# dig example.com mx +noall +answer
example.com.		3600	IN	MX	0 .
```

That answer is a **null MX**, and it is worth understanding rather than skipping past. The single record `0 .` — priority zero, target the root — is a deliberate, standards-defined way of saying "this domain does not accept mail at all". A resolver that finds it must stop looking, which is why the record exists: it tells the world not to bother trying. A domain that *does* accept mail returns one or more records with a hostname and a preference number. Since the reserved domain used above deliberately accepts no mail, here is the shape of a mail-accepting answer taken from a lab zone in the `.example` top-level domain, which is reserved for exactly this purpose and therefore cannot collide with anything real:

```bash
root@kali:~# dig intranet.example mx +noall +answer
intranet.example.	300	IN	MX	10 mail1.intranet.example.
intranet.example.	300	IN	MX	20 mail2.intranet.example.
```

Here two mail servers are published, `mail1` preferred over `mail2` because the lower number wins, which is how a domain load-balances inbound mail and provides failover. Both hostnames are useful to an assessor: each is a machine that accepts untrusted input from the internet, and each must be resolved further before you know what you are dealing with.

The `ns` type names the authoritative name servers for the zone, which are the machines that hold the definitive copies of every record in it:

```bash
root@kali:~# dig example.com ns +noall +answer
example.com.		172800	IN	NS	a.iana-servers.net.
example.com.		172800	IN	NS	b.iana-servers.net.
```

For an assessment these are the servers you would query directly (`dig @a.iana-servers.net example.com AXFR`) on the off-chance that they permit a zone transfer, which would hand over the entire contents of the DNS zone — every hostname, every mail server, every subdomain the organisation has ever registered — in one request. Modern servers refuse it, but the ones that do not remain one of the best findings available.

A few shortcuts make `dig` pleasant to use in scripts and in a hurry. `+short` prints only the answer data, so `dig +short example.com` returns a single line that is trivial to feed into another command. `+noall +answer` drops the header and status blocks when you only want the records. `@1.1.1.1` sends the query to a specific resolver instead of the system default, which is how you compare what different resolvers see — and the two answers are not always the same, because an internal resolver may know names that public DNS does not, while a public one may be willing to answer for names that internal filtering hides. `-x` performs a reverse lookup, turning an address into a name (`dig -x 93.184.216.34`). And `+trace` follows the resolution step by step from the root servers down, which is the single best way to *learn* how DNS actually works rather than just reading about it.

### 7.7 Changing your DNS server

The resolver your machine uses by default is described in a plain text file, `/etc/resolv.conf`, and it is worth reading before you change it:

```bash
root@kali:~# cat /etc/resolv.conf
# Generated by NetworkManager
search lab.local
nameserver 192.168.1.1
```

Two directives appear in a typical file. The `nameserver` line names the resolver to ask — here the lab gateway, which forwards queries upstream — and the `search` line lists domains that are appended to unqualified names, which is why typing `ping fileserver` can work on a corporate network without a fully qualified name. A file may list up to three `nameserver` lines, and they are consulted in order: the resolver asks the first one, and moves down the list only if the one above does not answer in time. That ordering is why a stale entry at the top of the list causes the mysterious pauses everybody eventually experiences.

Changing the resolver is a matter of writing a new line into the file. The quickest way is with the shell's `echo` and the `>` redirect, which *overwrites* the file with whatever you supply:

```bash
root@kali:~# echo "nameserver 1.1.1.1" > /etc/resolv.conf
root@kali:~# cat /etc/resolv.conf
nameserver 1.1.1.1
root@kali:~# dig example.com +short @1.1.1.1
93.184.216.34
root@kali:~# dig example.com | grep "SERVER:"
;; SERVER: 1.1.1.1#53(1.1.1.1) (UDP)
```

The `>` operator is destructive and there is no undo, which is why it is worth noting that `cat /etc/resolv.conf` came *before* the change: with a single-line file it hardly matters, but the same habit applied to a configuration file you have not read is how people destroy settings they cannot reconstruct. After the write, the file contains exactly one line — the public resolver operated by Cloudflare at 1.1.1.1, which is easy to remember and widely used; Google's public resolver at 8.8.8.8 and Quad9 at 9.9.9.9 are the other two you will meet most often, and every one of them has a published privacy policy that is worth reading before you make it your resolver. The second `dig` confirms the change took effect in the way that matters: the `SERVER:` line now reports `1.1.1.1#53` instead of the local gateway, which proves that queries are actually leaving through the new resolver rather than merely that the file's contents changed.

There is one important caveat and it catches everybody. On most modern systems `/etc/resolv.conf` is *generated*, not authored: NetworkManager, `dhclient`, `systemd-resolved` or `resolvconf` rewrites it whenever a network event occurs, and may even replace the file with a symbolic link pointing at a runtime directory. A hand-edited file therefore survives only until the next DHCP lease renewal or reboot. The durable approaches are to set the DNS server in the tool that owns the file — `nmcli con mod <name> ipv4.dns 1.1.1.1` for NetworkManager, a Netplan `nameservers:` stanza on Ubuntu Server, or `/etc/systemd/resolved.conf` followed by `systemctl restart systemd-resolved` — or, for a machine you control completely, to make the file immutable with `chattr +i /etc/resolv.conf`. The last option works and is sometimes the right answer on a lab machine, but it is a trap on anything else: every later network change will fail, and the next administrator will spend an afternoon wondering why.

For security work the resolver matters for two reasons. First, the DNS server sees every name you look up, so on an untrusted network a rogue resolver reads your reconnaissance and can answer with forged addresses. Second, using a resolver that validates DNSSEC or filters malicious domains changes what you can reach, which is why assessments often begin by identifying which resolvers a target environment uses and whether those queries are encrypted with DNS-over-HTTPS or DNS-over-TLS.

### 7.8 Mapping hostnames in `/etc/hosts`

Before any DNS query is sent, the system consults a local file. `/etc/hosts` is a static table of name-to-address mappings, and every Unix-like system has one; it is consulted first because of the lookup order defined in `/etc/nsswitch.conf`, where the `hosts:` line reads `files dns`, meaning "check the files first, then ask DNS".

```bash
root@kali:~# cat /etc/hosts
127.0.0.1	localhost
127.0.1.1	kali
::1		localhost ip6-localhost ip6-loopback

# The following lines are desirable for IPv6 capable hosts
ff02::1		ip6-allnodes
ff02::2		ip6-allrouters
root@kali:~# nano /etc/hosts
root@kali:~# cat /etc/hosts
127.0.0.1	localhost
127.0.1.1	kali
::1		localhost ip6-localhost ip6-loopback

# The following lines are desirable for IPv6 capable hosts
ff02::1		ip6-allnodes
ff02::2		ip6-allrouters
192.168.1.9	portal.example.com
```

The file's format is deliberately primitive: an IP address in the first column, one or more names after it separated by whitespace, and a `#` starting a comment. The default entries map `localhost` to the loopback addresses and give the machine its own hostname (`kali`) at `127.0.1.1`, which is a Debian-family convention that keeps the hostname resolvable even with no network at all. The `::1` line shows that several names can be attached to the same address, which is how aliases are created.

Adding a line maps any name of your choosing to any address, and because the file wins over DNS, the effect is immediate and global to the machine:

```bash
root@kali:~# getent hosts portal.example.com
192.168.1.9     portal.example.com
root@kali:~# curl -s -o /dev/null -w "%{http_code} %{remote_ip}\n" http://portal.example.com/
200 192.168.1.9
root@kali:~# dig +short portal.example.com
93.184.216.34
```

The three commands show the override from three angles, and the third is the interesting one. `getent hosts` asks the system's own resolver — the same mechanism applications use — and reports the address from the hosts file. `curl` then connects to `192.168.1.9` while the URL still says `portal.example.com`, which is exactly the kind of redirection this file is used for. But `dig` bypasses `/etc/hosts` entirely, because `dig` speaks to DNS servers directly, and it still returns the real, public address. That discrepancy is a wonderful teaching example of the difference between *the system's view* and *DNS's view*, and it is also the reason a name that works in the browser can fail in a script that uses `dig` or `nslookup` — the two are answering different questions.

Legitimate uses of the file are everywhere: giving a lab machine a friendly name before DNS exists, testing a new web server before its DNS record is changed, pinning a hostname so that an internal application stops resolving to a wrong address, and blocking a domain by pointing it at `127.0.0.1` or `0.0.0.0`. The illegitimate use is what makes it relevant here, and it is the same mechanism in reverse: because the file is local and trusted, an attacker who can write to it — or who can run a rogue DHCP or DNS server that answers first — can send a user to a server of their choosing. That is the basis of DNS and ARP spoofing attacks, where a user types the correct address and lands on an attacker's copy of the site. The defensive answers follow from the mechanism: monitor the integrity of `/etc/hosts` and `/etc/resolv.conf` on servers, prefer DNSSEC and DNS-over-HTTPS where the environment supports them, and treat any user-visible redirect to an unexpected address as an incident rather than a glitch.

---

## 8. Process management

A process is simply a program that is running: the kernel has loaded it into memory, given it an identifier, scheduled it onto a CPU and accounted for the resources it uses. On a single-user laptop you might have two hundred processes running at any moment, most of them daemons — background services with no terminal attached — and on a server the number is larger still. Knowing how to list them, read what they are consuming, decide which of them deserve more of the machine, stop the ones that have gone wrong, and move the ones that should be out of the way is the difference between administering a system and guessing at it. For security work there is a second reason to learn this material, since the same commands are how you find the process that is listening on a port, how you identify a suspicious daemon among legitimate ones, and how you stop an anti-virus agent that is interfering with an authorised test.

### 8.1 `ps` — viewing your own processes

The primary tool for looking at processes is `ps`, short for *process status*. Typed with no arguments it lists the processes attached to the current terminal and owned by the current user, which keeps the output short enough to be useful while you are learning.

```bash
root@kali:~# ps
    PID TTY          TIME CMD
   4122 pts/0    00:00:00 bash
   4188 pts/0    00:00:00 ps
```

Only two processes appear, and that is not an error: `ps` is printing the processes that belong to this shell session, which is you plus the `ps` command itself. The three columns are the vocabulary the rest of the section builds on. `PID` is the process identifier, a number the kernel assigns to every process it creates and which is unique among the processes running at that moment — it is the handle you use to inspect, prioritise or kill anything. `TTY` names the terminal the process is attached to (`pts/0` is the first pseudo-terminal, which is what your terminal emulator gives you), and a `?` in that column means the process has no terminal at all, which is the signature of a daemon. `TIME` is the total CPU time the process has consumed since it started, not the wall-clock time it has existed — a distinction that matters, because a process that has been running for a week and used two seconds of CPU is behaving very differently from one that has used two hours.

Two useful variations on the basic command are worth trying immediately. `ps -f` adds a column showing the parent process ID (PPID) and the full command line, which begins the process of answering "who started this, and with what arguments?". And `ps -ef` — the System V style that administrators of other Unix systems reach for out of habit — shows every process on the system with the same detail, which brings us to the next command.

### 8.2 `ps aux` — every process, every user

Running `ps` with the three BSD-style options `aux` displays all running processes for all users along with the resource figures for each one. This is the single most typed command in this section, and the header line it prints is worth memorising.

```bash
root@kali:~# ps aux
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.6 168412 12984 ?        Ss   09:41   0:04 /sbin/init splash
root           2  0.0  0.0      0     0 ?        S    09:41   0:00 [kthreadd]
root         812  0.0  0.1  15420  9216 ?        Ss   09:41   0:00 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
root        1044  0.0  0.4 240128 34220 ?        Ssl  09:41   0:01 /usr/lib/snapd/snapd
www-data    1204  0.1  1.1 3310420 88144 ?       Ssl  09:44   0:06 python3 /opt/portal/app.py
student     3901  0.0  0.1  10344  5120 pts/0    Ss   10:12   0:00 -bash
student     4122  0.0  0.0  12440  4096 pts/0    R+   10:31   0:00 ps aux
```

Each column answers a different question. `USER` is the account the process is running as, which is the first thing to check when you are deciding what you are allowed to do to it: a process owned by your own user can be signalled by you, while anyone else's needs root. `PID` is the identifier. `%CPU` and `%MEM` are the percentage of total CPU and memory the process is currently using, and `VSZ` and `RSS` are the memory figures in kilobytes — `VSZ` is the virtual size of the address space the process has reserved, while `RSS` is the resident set size, the memory it actually has in physical RAM right now. `VSZ` is usually much larger than `RSS` and is the more misleading of the two, because reserving address space costs nothing. `TTY` is the terminal, with `?` for daemons, and `STAT` is a short code describing the process state.

The `STAT` codes are worth learning because they tell you whether a process is healthy. `R` means running or runnable — on the CPU or waiting its turn. `S` means interruptible sleep, the normal state of a process waiting for something such as a network packet or a keystroke; almost everything on a quiet system is in `S`. `D` is uninterruptible sleep, which usually means blocked on disk or network I/O, and a process stuck in `D` cannot be killed even with `SIGKILL` because it is not running any code that could handle the signal — a diagnosis, not an obstacle. `T` means stopped, which is what `Ctrl+Z` leaves behind. `Z` means zombie: the process has finished but its parent has not yet collected its exit status, so a row remains in the table doing nothing. Zombies consume no CPU and no memory beyond the table entry, they cannot be killed, and they disappear when the parent reads the exit status or exits itself.

`START` is the time the process began, `TIME` is cumulative CPU time and `COMMAND` is the command line — with the additional detail that a command wrapped in square brackets at the start is a kernel thread rather than a user process, which is why `[kthreadd]` has no memory and no terminal. A few variations earn their place in your notes: `ps aux --sort=-%mem | head` lists the biggest memory consumers, `ps -eo pid,ppid,user,%cpu,cmd --sort=-%cpu | head` picks the columns you care about and sorts by CPU, and `ps -ef --forest` draws the process tree, showing which daemon spawned which worker — a view that turns "there are twelve nginx processes" into "there is one master and eleven workers".

### 8.3 Filtering processes by name

A busy machine produces hundreds of rows from `ps aux`, so in practice you filter. The technique is the pipe from section 2.9 applied to process output, and it works exactly as it does with any other command: pipe `ps aux` into `grep` and keep the lines that mention the program you are curious about. Here we look for metasploit's console, `msfconsole`.

```bash
root@kali:~# ps aux | grep msfconsole
root        5210  1.3  2.4 612344 198772 pts/1   Sl+  10:33   0:02 ruby /usr/bin/msfconsole -q
root        5233  0.0  0.0  12440  4096 pts/1    S+   10:33   0:00 grep msfconsole
```

Two lines come back, and understanding why is a small but genuinely useful lesson. The first is the real process: a Ruby interpreter running the `msfconsole` script, consuming 1.3% of CPU and about 2.4% of memory. The second line is the `grep` command itself, which matched its own command line — a permanent piece of bycatch that confuses beginners into thinking there are two processes. The standard ways to get rid of it are to exclude the search term from the results (`ps aux | grep msfconsole | grep -v grep`) or to bracket one character of the pattern (`ps aux | grep "[m]sfconsole"`), since the bracket expression matches the real process but does not appear literally in the grep process's own command line.

There is a better tool for this job, and it is worth adopting as the default: `pgrep` was written to answer exactly the question "which processes match this name?", so it does not match itself and it prints only identifiers by default.

```bash
root@kali:~# pgrep -a msfconsole
5210 ruby /usr/bin/msfconsole -q
root@kali:~# pgrep -u student -l bash
3901 bash
root@kali:~# pgrep -f "app.py"
1204
```

The `-a` flag prints the full command line alongside the PID, `-u` filters by user, `-l` prints the name only, and `-f` matches against the whole command line rather than just the executable name — which is what you need for anything started as `python3 /opt/portal/app.py`, because the process name there is only `python3`. Its companion, `pkill`, sends a signal to every process that matches, and we will use it later in this section once the concept of signals has been introduced.

### 8.4 `top` — finding the greediest process

`ps` gives you a snapshot; `top` gives you a film. The `top` command displays processes ordered by resource consumption and refreshes the display every few seconds, which makes it the tool of choice when you want to watch what a machine is doing right now — which process is eating the CPU, whether the load is rising, and how much memory is left.

```bash
root@kali:~# top
top - 10:36:22 up 55 min,  1 user,  load average: 0.42, 0.31, 0.24
Tasks: 214 total,   1 running, 213 sleeping,   0 stopped,   0 zombie
%Cpu(s):  3.9 us,  1.0 sy,  0.0 ni, 94.8 id,  0.3 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :   3914.8 total,   2412.6 free,    681.0 used,    821.2 buff/cache
MiB Swap:   2048.0 total,   2047.7 free,      0.3 used.   2893.0 avail Mem 

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
   1204 www-data  20   0 3310420  88144  21412 S   4.3   2.2   0:06.42 python3
   1044 root      20   0  240128  34220  19840 S   0.7   0.9   0:01.19 snapd
   5210 root      20   0  612344 198772  42188 S   0.3   5.0   0:02.11 ruby
      1 root      20   0  168412  12984   9216 S   0.0   0.6   0:04.33 systemd
```

The five header lines summarise the whole machine and are worth reading before the process list. The first line gives the time, the uptime, the number of logged-in users and the three **load averages**, which report the average number of runnable processes over the last one, five and fifteen minutes. A load average above the number of CPU cores means work is queueing — on this single-core-equivalent virtual machine, 0.42 means the machine is comfortable. The second line counts processes by state, and any non-zero `zombie` or `stopped` count is worth following up. The third line breaks CPU time down by category: `us` is user-space work, `sy` is kernel work, `id` is idle, `wa` is time waiting on I/O (a rising `wa` usually means a slow disk), and `st` is time *stolen* by the hypervisor — the figure that tells you another virtual machine on the same host is eating your CPU. The fourth and fifth lines describe memory: total, free, used, and the page cache sitting in `buff/cache`, which is memory the kernel is using to cache disk reads and will give back instantly if an application needs it. Reading `free` without reading `buff/cache` is the classic beginner error, since Linux deliberately uses otherwise-idle RAM as a cache: a machine with almost no `free` memory and a large `buff/cache` is working exactly as designed.

Below the header comes the process list, which repeats several `ps` columns with two additions. `PR` is the kernel's priority and `NI` is the nice value discussed in the next subsection, and both are in the list because they are the knobs you turn when a process needs more or less of the CPU than the others.

`top` is interactive, and half a dozen keys carry most of its value: `P` sorts by CPU usage (the default), `M` sorts by memory, `k` prompts for a PID and a signal so that you can kill a process without leaving the display, `r` prompts for a PID and a new nice value so that you can re-prioritise one, `1` expands the CPU line to show each core individually, `h` shows the full help, and `q` quits. If you would rather capture the output than watch it, `top -b -n 1 | head -20` runs it once in batch mode and prints the result, which is the form to use in a script or a report. And if you spend much time doing this, install `htop`: it is the same information with colour, mouse support and scrollable columns.

### 8.5 `nice` — setting the priority of a new process

On a machine with more work than CPU, the kernel scheduler decides who runs, and it makes that decision partly on the basis of each process's **nice value** — a number from −20 to 19 that expresses how willing the process is to give way to others. The scale runs backwards from intuition: −20 is the highest priority (least nice, most demanding), 19 is the lowest (extremely nice, always yielding), and the default is 0. A process started with `nice` carries the value you give it for its entire life, which is why the setting is applied at launch.

The `-n` flag introduces the adjustment, and here the ssh-agent daemon is started with a nicer-than-default value of −10, meaning it should be given preference by the scheduler.

```bash
root@kali:~# nice -n -10 /usr/bin/ssh-agent
SSH_AUTH_SOCK=/tmp/ssh-XXXXXXuP3n9g/agent.4412; export SSH_AUTH_SOCK;
SSH_AGENT_PID=4412; export SSH_AGENT_PID;
echo Agent pid 4412;
root@kali:~# ps -o pid,ni,cmd -C ssh-agent
    PID  NI CMD
   4412 -10 /usr/bin/ssh-agent
```

The program started normally — a process identifier was assigned, and the shell printed the environment variables a caller needs in order to talk to the agent. The verification command is the interesting part: `ps -o pid,ni,cmd -C ssh-agent` selects specific columns and names the process to match, and the `NI` column shows `-10`, confirming that the priority request was accepted. `-C` (match by command name) is a tidy alternative to the `grep`-and-pipe ritual of section 8.3 and it has the pleasant property of not matching itself.

One permission rule matters here: any user may make a process *nicer* (raise its number, give way more), but only root may lower it (make a process more demanding). If a normal user tries the command above, the shell replies `nice: cannot set niceness: Permission denied` and no process starts. That asymmetry is deliberate, because the whole point of the mechanism is to protect interactive work from being starved by background jobs, and it would be defeated if any user could claim the front of the queue.

The non-privileged and far more common use is the opposite direction — pushing a long, unimportant job out of the way so that it does not compete with the work you are actually doing:

```bash
root@kali:~# nice -n 19 ./backup-script.sh &
[1] 5487
root@kali:~# nice -n 19 /usr/bin/updatedb &
[2] 5490
```

Both jobs are running at the lowest possible priority, and the `[1]` and `[2]` job numbers with the PIDs after them are the shell's acknowledgement — the topic of section 8.8. This pattern is worth remembering, because "run this in the background at minimum priority" is the correct shape for almost every maintenance task on a machine someone else is using.

### 8.6 `renice` — changing the priority of a running process

The priority of a process that is already running can be changed without restarting it, using `renice`. It takes a nice value or an adjustment and one or more PIDs, and it works on processes you did not start, provided you have the privileges to change them.

```bash
root@kali:~# renice 20 6242
6242 (process ID) old priority 0, new priority 20
root@kali:~# ps -o pid,ni,cmd -p 6242
    PID  NI CMD
   6242  20 /usr/bin/updatedb
root@kali:~# renice -n -5 -p 6242
renice: failed to set priority for 6242 (process ID): Permission denied
```

The report line is unambiguous and names both values, which makes `renice` pleasantly self-documenting: the process went from a nice value of 0 to 20, meaning it now yields to everything else on the machine. A later `ps` confirms the new value. The third command demonstrates the permission rule from the previous subsection from the other side: as an unprivileged user, lowering the nice value (here to −5, which would raise the process's priority) is refused outright, with the process left at its previous value.

Three forms are worth keeping in your notes. `renice -n 10 -p 6242` sets an absolute value using the explicit flags, which is the form to prefer in scripts because it is unambiguous. `renice -u student` changes the priority of every process owned by a user, which is a good way to quiet a runaway build account without hunting for PIDs. And `renice 5 -g developers` applies to a process group. On a modern systemd machine there is a second, complementary mechanism — cgroups and the `systemd-run --slice` and `systemctl set-property` commands let you cap CPU and memory for a whole service rather than just hinting at the scheduler, and that is the approach to use when the requirement is "this service may never use more than one core" rather than "this service should be polite".

### 8.7 `kill` — stopping processes with signals

The `kill` command stops processes, and its name misleads in a useful way: what it actually sends is a *signal*, a numbered, asynchronous notification delivered to a process by the kernel, and the process may handle most signals however it likes — clean up, save state, close sockets, ignore them or exit. There are 64 signals in all, and the two that matter most are the polite one and the fatal one. Here the process with PID 6242 is sent signal number 1.

```bash
root@kali:~# kill -1 6242
root@kali:~# ps -p 6242
    PID TTY          TIME CMD
root@kali:~# kill -l
 1) SIGHUP	 2) SIGINT	 3) SIGQUIT	 4) SIGILL	 5) SIGTRAP
 6) SIGABRT	 7) SIGBUS	 8) SIGFPE	 9) SIGKILL	10) SIGUSR1
11) SIGSEGV	12) SIGUSR2	13) SIGPIPE	14) SIGALRM	15) SIGTERM
...
root@kali:~# kill -9 4378
```

`SIGHUP`, signal 1, originally meant "the terminal you were connected to has hung up", and daemons such as `sshd`, `nginx` and `rsyslog` reinterpret it as "re-read your configuration files and reopen your log files" — which is why `kill -1` is the traditional way to make a service reload without dropping its connections, and why the same signal can terminate a simple foreground program that has no handler for it. Because `updatedb` is a short-lived program with no handler, sending it `SIGHUP` ended it, and the following `ps` shows an empty table.

`kill -l` lists every signal the system knows, and the numbers are worth learning in pairs. Signal 15, `SIGTERM`, is the *default* when you type `kill` with no number, and it is the correct first attempt for any process: it politely asks the program to terminate, and a well-written program will flush its buffers, remove its temporary files and close its sockets on the way out. Signal 9, `SIGKILL`, cannot be caught, blocked or ignored — the kernel removes the process on the spot. It is the right tool for a process that has ignored `SIGTERM` and the wrong tool for everything else, because `SIGKILL` gives the program no chance to clean up, which is how databases are corrupted and stale lock files are left behind. Signal 2, `SIGINT`, is what `Ctrl+C` sends from the terminal; signal 3, `SIGQUIT`, is `Ctrl+\`; signal 18 stops a process and 19 resumes it, which is what `Ctrl+Z` and `bg` do under the hood.

That ordering — `SIGTERM` first, then escalate — is the professional habit and it is worth enforcing on yourself from the first day. A process that does not die to `SIGTERM` is telling you something: it may be blocked in uninterruptible I/O, it may be a zombie, or it may be deliberately ignoring the signal, and each of those deserves a moment's thought before you reach for `-9`. A quick sanity check for the zombie case is `ps -o pid,stat,cmd -p <pid>`, where a `Z` in the state column means no signal will ever work, because there is no running process left to receive it.

Two related commands make targeted killing easier. `pkill` sends a signal to every process matching a pattern, so `pkill -f app.py` stops the web application by its command line and `pkill -u student -TERM -f backup` stops a user's backup jobs without touching anything else. `killall` does the same by exact program name. Both are powerful enough to be dangerous — a pattern that is too broad will match something you did not intend, and `pkill -9 -f python` on a busy machine can take out a dozen innocent processes — so the safe workflow is always to run `pgrep` first, look at what would be affected, and only then send the signal with the same pattern.

### 8.8 Background jobs, `jobs`, `fg` and `nohup`

A command typed at a prompt owns the terminal until it finishes, which is inconvenient the moment you start something long-running. The shell provides **job control** for exactly this problem: appending a single ampersand to a command runs it in the background, immediately returning your prompt while the process keeps working.

```bash
root@kali:~# nano lab-notes.txt &
[1] 4312
root@kali:~# jobs
[1]+  Stopped                 nano lab-notes.txt
root@kali:~# fg %1
```

The shell replied with the job number `[1]` and the process identifier `4312`, and in a terminal application this is where an important detail appears. `nano` is an interactive full-screen program, and a program in that category will normally *stop itself* when it finds it has been put in the background without a terminal — which is what the `jobs` output shows: the job is listed as `Stopped` rather than `Running`, because the editor tried to read from a terminal it no longer controls. This is the general rule and it is worth internalising: backgrounding is for programs that do not need your keyboard, such as a compiler, a download, a scan or a long copy — not for interactive editors.

```bash
root@kali:~# ./backup-script.sh &
[2] 4701
root@kali:~# jobs -l
[1]+  4312 Stopped                 nano lab-notes.txt
[2]-  4701 Running                 ./backup-script.sh &
root@kali:~# bg %1
```

With a genuinely non-interactive program the behaviour is what you would hope: job `[2]` is `Running`, and `jobs -l` shows both jobs with their PIDs and — importantly — the marker in the second column. The `+` marks the *current* job, the one `fg` and `bg` act on when you do not name one, while `-` marks the previous job; that is why `fg` with no argument resumes whatever was most recently backgrounded, and why it is safer to name the job explicitly until the idea is second nature.

The full vocabulary of job control is small. `Ctrl+Z` *stops* the foreground process and returns you to the prompt, which is a pause rather than an exit. `bg %1` resumes the stopped job `1` in the background. `fg %1` brings it back to the foreground, and it then owns the terminal again. `jobs` lists them, `kill %1` sends a signal to a job by number rather than by PID, and `disown %1` removes a job from the shell's table so that it is not sent `SIGHUP` when the shell exits.

That last point explains the most common way people lose work. When a terminal closes, the shell sends `SIGHUP` to its jobs, and any process that does not handle it dies — so a download started with a plain `&` in a terminal window that you then close simply disappears. The traditional defence is `nohup`, which detaches the process from the hangup signal and redirects its output, because there is no longer a terminal to print it to:

```bash
root@kali:~# nohup ./scan.sh > scan.log 2>&1 &
[3] 4820
root@kali:~# ps -o pid,stat,cmd -p 4820
    PID STAT CMD
   4820 S    /bin/bash ./scan.sh
root@kali:~# nohup ./scan.sh &
[4] 4833
nohup: ignoring input and appending output to 'nohup.out'
```

The redirection `> scan.log 2>&1` sends both standard output and standard error into a file, which is why the first invocation is silent. Omit it, as in the second invocation, and `nohup` writes the output to `nohup.out` in the current directory and says so — the message appears on your terminal *after* the job line because it is written to standard error while the shell is still printing the job number. The process now survives the terminal, and the `STAT` column shows it sleeping normally. For anything you will want to come back to, a terminal multiplexer such as `tmux` or `screen` is the better tool, because it keeps a whole session alive on the server and lets you reattach to it from anywhere, with scrollback and multiple windows; `nohup` remains the lightweight answer when you only need one command to outlive your session.

### 8.9 Scheduling a process with `at`

Sometimes the right moment to run a job is in the future. The `at` command schedules a command to run **once**, at a specified time, and it is the tool to reach for when you want a task performed after you have gone home or after some other event has happened. It reads the commands to run from standard input, which is why the invocation looks interactive.

```bash
root@kali:~# at 9:00pm
warning: commands will be executed using /bin/sh
at> /root/simple_bash.sh
at> <EOT>
job 3 at Sat Oct 10 21:00:00 2026
```

The first line is a warning rather than a complaint, and it is worth heeding: `at` runs its commands with `/bin/sh`, not with your interactive shell, so anything that depends on bash-specific syntax, aliases or your environment variables will not work — write the job as you would write a script, with full paths. The `at>` prompt then accepts commands one per line, and pressing `Ctrl+D` — which the shell displays as `<EOT>`, for end of transmission — ends the job. The confirmation line reports the job number and the exact time it will run, and `at` interprets a variety of human formats: `at 9:00pm`, `at 14:30`, `at now + 10 minutes`, `at midnight`, `at 09:00 tomorrow`. Times without a date are taken as today if still in the future and otherwise tomorrow, which occasionally surprises people at 23:59.

The queue of pending jobs is managed with three short commands. `atq` lists them, `at -c <job>` prints the entire environment and command that a scheduled job will run — the definitive way to debug one that misbehaves — and `atrm <job>` removes it. The service behind the command is a daemon called `atd`, and on a minimal system it may not be running; `systemctl status atd` will say so, and `systemctl enable --now atd` fixes it. Access to `at` is controlled by `/etc/at.allow` and `/etc/at.deny`, the same allow-and-deny pattern used by `cron`, which is how administrators keep ordinary users from scheduling work on production machines.

For jobs that should run **repeatedly** — every night, every hour, every Monday — the tool is `cron` rather than `at`, and the difference is worth stating plainly: `at` is for one-offs, `cron` is for recurrence. A user's scheduled jobs live in a crontab, edited with `crontab -e`, and each line contains five time fields followed by the command to run:

```bash
root@kali:~# crontab -l
# m h  dom mon dow   command
  0 3  *   *   *     /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
 30 1  *   *   0     /usr/local/bin/weekly-report.sh
```

Reading the fields in order: minute, hour, day of month, month, day of week, where `*` means "every". So the first line runs the backup at 03:00 every day, and the second runs the weekly report at 01:30 every Sunday (day-of-week 0). The system-wide equivalents live in `/etc/crontab` and `/etc/cron.d/`, with the addition of a user field, while the directories `/etc/cron.hourly`, `/etc/cron.daily`, `/etc/cron.weekly` and `/etc/cron.monthly` accept plain scripts with no time fields at all. Modern distributions increasingly use systemd timers instead, which offer the same scheduling with better logging and the ability to trigger on events rather than only on the clock — `systemctl list-timers` shows what is scheduled on such a system.

For security work, scheduled jobs matter for two reasons. Defensively, `cron` is a classic persistence mechanism, so anything scheduled that you do not recognise deserves attention: a job that downloads and runs a script, or that reappears after being removed, is a finding. Offensively, `at` is the reliable way to schedule a rollback when you are about to make a change that might cut off your own access — a firewall reload, a network reconfiguration, an SSH daemon restart — since a job armed before the change will restore the old state even if your session dies with it. That single technique has rescued more remote administration sessions than any other habit in this section.

---

## 9. User environment variables

Variables are simply named values — key-value pairs held by a running shell — and understanding them is a must for getting the most out of a Linux system, because a great deal of what the shell and the programs it launches consider to be "the environment" is nothing more than a list of them. There are two families, and the distinction matters. A **shell variable** exists only inside the shell that created it; it is invisible to programs the shell starts. An **environment variable** has been *exported*, which means it is copied into the environment of every child process, and therefore into every program you run from that shell and every process those programs start in turn. Variables are inherited down the process tree and never flow back up, which explains most of the confusing behaviour beginners meet: a child process can read what you exported, but it cannot change what its parent sees.

The convention for names is upper case — `HOME`, `PATH`, `USER` — which is a convention rather than a rule, but a valuable one, because `HISTSIZE` and `histsize` are two entirely different variables and only one of them is the one the shell uses.

### 9.1 Viewing all the variables

Two commands list variables, and the difference between them is the distinction above made concrete. `set`, typed with no arguments, prints every shell variable and function the current shell knows about, including all the environment variables it inherited. `env` prints only the exported environment, which is the list that child processes will actually receive.

```bash
root@kali:~# set | more
BASH=/usr/bin/bash
BASHOPTS=checkwinsize:cmdhist:complete_fullquote:expand_aliases:extquote:force_fignore:globasciiranges
BASH_ALIASES=()
BASH_ARGC=()
BASH_ARGV=()
BASH_CMDS=()
BASH_COMPLETION_VERSINFO=([0]="2" [1]="11")
BASH_LINENO=()
BASH_SOURCE=()
...
COLUMNS=80
DIRSTACK=()
EUID=0
GROUPS=([0]="0")
HISTFILE=/root/.bash_history
HISTFILESIZE=2000
HISTSIZE=1000
HOME=/root
HOSTNAME=kali
--More--
```

The pipe into `more` is not optional in practice: on a normal desktop `set` prints several hundred lines, most of them shell-internal variables such as `BASH_ARGC` that you will never use. Among the noise, a handful of familiar names appear — `HOME` is your home directory, `HOSTNAME` is the machine's name, `HISTFILE` is the file your command history is written to, and `HISTSIZE` is the number of commands the shell keeps in memory to offer you when you press the up arrow. Because the output is ordinary text going to standard output, it can be filtered, sorted and folded like anything else, which is exactly what the next subsection does.

### 9.2 Filtering for a particular variable

Piping `set` into `grep` is the way to find one variable among hundreds, and the technique is identical to the process filtering of section 8.3. Here we look for `HISTSIZE`.

```bash
root@kali:~# set | grep HISTSIZE
HISTFILESIZE=2000
HISTSIZE=1000
root@kali:~# echo "my history holds $HISTSIZE commands"
my history holds 1000 commands
root@kali:~# printenv HISTSIZE

root@kali:~# printenv HOME
/root
```

Three commands, and the third one's silence is the most instructive thing in this subsection. The `grep` approach prints both history-related variables in the `name=value` form the shell keeps them in, and the `echo` form embeds the value in a sentence while introducing the sigil that matters: **`$` before a name means "substitute the value"**. Drop the dollar sign and you print the letters of the name back at yourself, which is the single most common mistake in this whole section.

`printenv` prints *only the value* of a variable, which is what you want inside a script — and here it printed nothing at all. That is not a mistake: `HISTSIZE` is set, as the first two commands prove, but it has never been **exported**, so it exists only inside this shell and the `printenv` process — a child of the shell — cannot see it. The exit status tells the same story, returning 1 for a variable that is not in the environment. `printenv HOME` works, and prints `/root`, because `HOME` *is* exported and therefore is handed to every process the shell starts. There is no better demonstration of the difference between a shell variable and an environment variable, and it explains the behaviour of every program you will ever wonder about: if a tool claims not to know a value that is plainly set in your terminal, the variable was never exported.

So the answer to the original question is that this shell remembers one thousand commands. That is comfortable for interactive work, and it is also a fact with security implications worth noting: your history file is a record of everything you have typed, including passwords passed as command-line arguments and hosts you have connected to, which is why `~/.bash_history` is among the first files an attacker reads on a compromised machine — and why clearing it, or disabling history for sensitive commands with a leading space, is part of an operator's routine.

### 9.3 Changing a value temporarily

Changing a variable is a matter of assigning it a new value, and the assignment is written with no spaces around the equals sign.

```bash
root@kali:~# HISTSIZE=0
root@kali:~# echo $HISTSIZE
0
root@kali:~# echo $HISTSIZE > ~/valueofHISTSIZE.txt
root@kali:~# cat ~/valueofHISTSIZE.txt
0
```

The first line sets the variable to zero and the second confirms it, and the effect is exactly what that value implies: the shell no longer keeps any commands in memory for the current session, so pressing the up arrow produces nothing and history recall stops working until you change the value back. It is a small demonstration of a large truth about configuration, which is that a harmless-looking one-character change can quietly remove a convenience you depend on without any error message at all.

Two details in this transcript are worth pausing on, because they are where beginners lose time. The first is the spacing rule: `HISTSIZE=0` assigns, while `HISTSIZE = 0` is interpreted by the shell as an attempt to run a command called `HISTSIZE` with two arguments, which fails with `HISTSIZE: command not found`. The equals sign must be glued to the name and to the value, and no other arrangement works. The second detail is the redirect in the third command. The intent — and a habit worth adopting — is to *save the old value somewhere before changing it*, so that you can always put things back: `echo $HISTSIZE > ~/valueofHISTSIZE.txt` writes the current value into a file in your home directory, and `cat` confirms what was written. Note the `>`, which sends the output to the file; without it, the command would print the value to the terminal and save nothing, which is a mistake easy to make and easy to detect, since the redirect creates the file as a side effect. In this case we saved the already-changed value of 0, so the file now records the experiment rather than the default — a reminder to save *before* you edit, not after.

### 9.4 Making the change permanent with `export`

The assignment above lasts as long as the session does. Opening a new terminal gives you a fresh shell with a fresh environment, and `HISTSIZE` is back to 1000 because the value was never written down anywhere. To make a variable persistent you must first place it into the environment with `export`, and then arrange for the assignment to happen in every new shell — which is done by adding the line to a startup file such as `~/.bashrc`.

```bash
root@kali:~# HISTSIZE=0
root@kali:~# export HISTSIZE
root@kali:~# bash -c 'echo "child shell sees HISTSIZE=$HISTSIZE"'
child shell sees HISTSIZE=0
root@kali:~# bash -c 'echo "child shell sees HISTFILE=$HISTFILE"'
child shell sees HISTFILE=
root@kali:~# echo 'export HISTSIZE=0' >> ~/.bashrc
```

The first two commands are the two-step ritual: assign the value, then mark the variable for export. Order matters — exporting a name that has no value yet exports nothing — and `export HISTSIZE=0` in a single command is the compact form that does both at once. The third command proves the point, and it is the demonstration that makes the shell-versus-environment distinction concrete: `bash -c` starts a *new* shell, and because `HISTSIZE` was exported, that child process inherited the value 0 and printed it. The fourth command is the control experiment, asking the same child about `HISTFILE`, which was never exported — the child reports an empty value even though the parent shell has it set. Nothing about that is a bug; it is the inheritance rule doing exactly what it says.

The final line makes the change survive future sessions by appending to `~/.bashrc`, the script that bash runs each time it starts an interactive shell. The `>>` operator appends rather than overwrites, which is the correct choice for startup files that already contain configuration; using `>` here would erase whatever was there, including the lines that define your prompt and aliases. Which file to edit depends on the shell and the moment: `~/.bashrc` for interactive bash shells, `~/.bash_profile` or `~/.profile` for login shells, `/etc/environment` or a file under `/etc/profile.d/` for system-wide settings that apply to every user. A change to `~/.bashrc` takes effect in the *next* shell you start, or immediately in the current one if you load it by hand with `source ~/.bashrc` — a command worth remembering, since it saves logging out and back in for every edit.

### 9.5 Creating and deleting your own variables

Variables do not have to be pre-defined by the system; you can create them, name them whatever you like and use them as shorthand for values you type often. Here a variable is created to hold a domain name.

```bash
root@kali:~# url_variable="example.com"
root@kali:~# echo $url_variable
example.com
root@kali:~# echo "the target is $url_variable"
the target is example.com
root@kali:~# url_variable=example.org
root@kali:~# echo $url_variable
example.org
root@kali:~# unset url_variable
root@kali:~# echo $url_variable

root@kali:~#
```

The assignment creates the variable and `echo $url_variable` reads it back; the second form shows the substitution working inside a longer string, which is how variables earn their keep in scripts. The re-assignment demonstrates that a variable can be changed at any time simply by assigning it again, and the last two commands do the opposite: `unset` removes the variable from the shell entirely, and the following `echo` prints an empty line, because an undefined variable expands to nothing rather than producing an error.

That silent expansion is a trap worth naming, because scripts that fail mysteriously usually fail here: if a variable is unset or misspelled, `$name` becomes an empty string and the command runs with a missing argument instead of complaining. Two defences are standard. Quoting, which turns a missing value into an empty argument rather than into no argument at all — `echo "$url_variable"` is safer than `echo $url_variable` in every context where the value might contain spaces or be empty. And the `${var:?message}` form, which makes the shell abort with your message if the variable is unset, which is the right way to check a required input in a script: `: "${url_variable:?please set url_variable}"`.

The other habit from this subsection is quoting on assignment. `url_variable="example.com"` needs no quotes for a value without spaces, but `url_variable="the example domain"` would be interpreted as an assignment of `the` followed by a command called `example domain` without them — a mistake that produces one of the more confusing shell errors you will meet. Quoting the right-hand side of an assignment is never wrong, so it is worth doing by default.

### 9.6 The variables worth knowing

The environment is not an academic topic; a handful of variables change how the system behaves for you, and knowing them turns several classes of confusing failure into one-line diagnoses.

`PATH` is the most important of them, and we met it in section 2.8: a colon-separated list of directories the shell searches, in order, for the command you typed. `echo $PATH` shows your list, and appending to it is a common operation — `export PATH="$PATH:/opt/tools/bin"` makes a directory of your own scripts runnable by name, and it is worth understanding that the *order* determines who wins when two directories hold a program with the same name. The security angle is real in both directions: a `PATH` that includes `.` (the current directory) lets a malicious `ls` in a shared folder be executed by an unsuspecting user, and a writable directory early in root's `PATH` is a classic privilege escalation.

`HOME` is your home directory and is what `~` expands to; every program that stores a configuration file uses it, so a wrong value for a user running a service is a common source of "the daemon cannot find its config" problems. `USER` and `LOGNAME` name the current account, and `SHELL` records your login shell. `PS1` is the primary prompt string — the format of your shell prompt, which is why the prompt in this guide reads `root@kali:~#` and why changing `PS1` changes it instantly, complete with colour codes if you want them. `HISTFILE`, `HISTSIZE` and `HISTFILESIZE` govern command history, as we have seen. `LANG` and `LC_ALL` set the language, character encoding and formatting conventions that programs use for messages and for sorting, and they are the reason a script can behave differently on two machines with identical software — `sort` orders letters differently under different locales, and dates are printed in the local format. `EDITOR` and `VISUAL` decide which editor tools such as `crontab -e`, `git commit` and `visudo` open for you, which is why setting `export EDITOR=nano` is one of the first things a beginner should do. And `LD_PRELOAD` deserves a mention by name, because it asks the dynamic loader to load a shared library into every process the shell starts — a legitimate debugging and compatibility tool, and simultaneously a well-known privilege-escalation primitive when it can be set for a SUID binary. Seeing it in an environment on a production system is worth an investigation.

Taken together, the three topics of this instalment — networks, processes and the environment — are the layer of Linux that sits between "I can use the command line" and "I can administer or assess this machine". An interface with a wrong address, a process eating a core, a name resolving through the wrong resolver: these are the ordinary faults, and the commands above are the ordinary answers to them.

---

## 10. Quick reference cheat sheet

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

### Networking

| Command | What it does | Typical use |
| --- | --- | --- |
| `ifconfig` / `ip -br addr` | shows interfaces, addresses and MAC addresses | `ip -br addr` |
| `ip route` | shows the routing table and default gateway | `ip route` |
| `iwconfig` / `iw dev` | shows wireless-specific settings | `iw dev wlan0 link` |
| `ifconfig eth0 <ip>` | assigns an address (temporary) | `ifconfig eth0 192.168.1.13` |
| `ip addr add <ip>/<mask> dev eth0` | adds an address in the modern syntax | `ip addr add 192.168.1.13/24 dev eth0` |
| `ip link set dev eth0 address <mac>` | changes the MAC address | `ip link set dev eth0 address 00:11:22:33:44:55` |
| `dhclient eth0` / `dhclient -r eth0` | requests / releases a DHCP lease | `dhclient -v eth0` |
| `dig <name>` | full DNS answer for a name | `dig example.com` |
| `dig +short <name>` | answer data only, script friendly | `dig +short example.com` |
| `dig <name> mx` / `ns` | mail servers / name servers | `dig example.com mx` |
| `dig @1.1.1.1 <name>` | query a specific resolver | `dig @1.1.1.1 example.com` |
| `dig -x <address>` | reverse lookup | `dig -x 93.184.216.34` |
| `cat /etc/resolv.conf` | shows the resolvers in use | `cat /etc/resolv.conf` |
| `echo "nameserver 1.1.1.1" > /etc/resolv.conf` | changes the resolver (may be overwritten) | as shown |
| `getent hosts <name>` | resolves a name the way applications do | `getent hosts portal.example.com` |

### Processes

| Command | What it does | Typical use |
| --- | --- | --- |
| `ps` | your own processes on the current terminal | `ps -f` |
| `ps aux` | every process, every user, with resources | `ps aux --sort=-%mem \| head` |
| `ps -ef --forest` | full listing drawn as a process tree | `ps -ef --forest` |
| `pgrep -a <name>` | PIDs and command lines matching a name | `pgrep -a sshd` |
| `top` | live, resource-sorted view of processes | `top -b -n 1 \| head -20` |
| `nice -n <n> <cmd>` | starts a command with a given priority | `nice -n 19 ./backup.sh &` |
| `renice <n> -p <pid>` | changes the priority of a running process | `renice 20 -p 6242` |
| `kill <pid>` | sends SIGTERM (15) — the polite stop | `kill 6242` |
| `kill -9 <pid>` | sends SIGKILL — unstoppable, last resort | `kill -9 4378` |
| `kill -1 <pid>` | sends SIGHUP — daemons re-read config | `kill -1 812` |
| `kill -l` | lists all 64 signal names | `kill -l` |
| `pkill -f <pattern>` | signals every process matching a pattern | `pkill -f app.py` |
| `cmd &` | runs a command in the background | `./backup.sh &` |
| `jobs -l` / `fg %1` / `bg %1` | lists / foregrounds / backgrounds jobs | `fg %1` |
| `nohup ./job.sh > job.log 2>&1 &` | survives the terminal closing | as shown |
| `at 9:00pm` | schedules a one-off job (end with Ctrl+D) | `at now + 10 minutes` |
| `atq` / `atrm <job>` | lists / removes scheduled `at` jobs | `atq` |
| `crontab -e` / `crontab -l` | edits / lists recurring jobs | `crontab -l` |

### Environment variables

| Command | What it does | Typical use |
| --- | --- | --- |
| `set` | every shell variable and function | `set \| more` |
| `env` / `printenv` | only exported variables (what children inherit) | `env \| sort` |
| `printenv <NAME>` | prints one variable's value | `printenv PATH` |
| `echo $NAME` | expands a variable in a string | `echo "$HOME"` |
| `NAME=value` | assigns for the current shell only | `HISTSIZE=0` |
| `export NAME=value` | assigns and passes it to child processes | `export EDITOR=nano` |
| `echo $NAME > file` | saves a value before you change it | `echo $HISTSIZE > ~/histsize.bak` |
| `echo 'export NAME=value' >> ~/.bashrc` | makes a variable persistent | as shown |
| `source ~/.bashrc` | reloads the startup file in the current shell | `source ~/.bashrc` |
| `unset NAME` | deletes a variable | `unset url_variable` |
| `export PATH="$PATH:/opt/tools/bin"` | adds a directory to the command search path | as shown |

---

## 11. Practice exercises

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
10. **Networks.** Record your interface's address, mask, MAC address, gateway and resolver. Change the address by hand and prove that the change is temporary by restarting the networking service. Spoof the MAC address, then run `dhclient` and explain which value came back and why. Finally, ask `dig` for the A, MX and NS records of a domain you control, and say in one sentence what each record type is telling you.
11. **Processes.** Find the process that is using the most memory, and the one that is using the most CPU. Run a long job at the lowest priority in the background, list it with `jobs -l`, then stop it, background it again and foreground it. Schedule a harmless command with `at` to run two minutes from now and confirm it ran. Send `SIGTERM` to a process of your choosing, and only if that fails escalate to `SIGKILL`, explaining in writing why the order matters.
12. **Environment.** Print the value of `HISTSIZE`, save it to a file in your home directory, change it for the current shell only, and prove that a child shell does not see the change. Then export it, prove that a child shell does see it, and finally make it permanent in `~/.bashrc`. Create a variable of your own, use it in a command, and delete it with `unset` — noting what an unset variable expands to.

If you can complete these twelve exercises from memory alone, you have the command-line foundation that everything else in Linux administration and security testing is built on.

---

## Closing notes

The command line rewards patience and punishes haste, and almost every mistake a beginner makes falls into one of three categories: not knowing which directory they are in, not knowing which user they are running as, and not reading the error message that would have explained the problem. The first two are answered by `pwd` and `whoami` before every risky command; the third is a habit of attention rather than a command. Everything else in this guide is syntax, and syntax is what a manual page is for.

Two habits are worth building from the first week. The first is to *read the plan* before you approve it — `apt-get` tells you exactly what it will remove, `find -exec` can be tested with `-print`, and destructive commands can be previewed by prefixing them with `echo` so the shell shows you what it intends to run. The second is to *write things down*: keep a plain text file of the commands you had to look up, and after a few weeks you will find that you have written your own cheat sheet, in your own words, which is worth more than any guide you can download.

From here, natural next steps are shell **scripting and automation** — conditions, loops, functions and the variables you have just met, assembled into programs that do a day's work in a second — followed by **service management** with `systemctl` and the systemd journal, **package and configuration management** across more than one machine, and a deeper study of the **permission and privilege model** that this guide has only introduced. Each of those builds directly on the material above, and each of them is far less intimidating once moving around the file system, reading files, reading a long listing and inspecting a running process all feel like second nature.

One last piece of advice about the material in the second half of this guide. Networks, processes and environment variables are the topics where a beginner's commands can do real damage quickly: changing an address over SSH ends your session, `kill -9` on the wrong PID takes down a service somebody is using, and a careless line appended to `~/.bashrc` can break every new shell you open. Practise on a virtual machine, take a snapshot before each exercise, and keep the two habits from the paragraph above — read the plan before you approve it, and write down what you typed.
