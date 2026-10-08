# Linux for Beginners (Part 3): Scripting, Scheduling and Services

*This is Part 3 of a three-part guide. It continues [Linux for Beginners (Part 1): The Shell, Files, Text, Packages and Permissions](01-linux-for-beginners-part-01.md) and [Linux for Beginners (Part 2): Networks, Processes and Environment Variables](02-linux-for-beginners-part-02.md), and it assumes you are comfortable with the material there — moving around the file system, reading permission strings, filtering text through pipes, managing processes and reading the environment.*

## Introduction

Part 1 covered the layer of Linux that everything else is built on: the shell, the file system, text, packages and permissions. Part 2 moved one level deeper, into the state a machine depends on: its network configuration, its running processes and the environment every program inherits. This part is about getting more out of the operating system than interactive typing can give you. We look at **bash scripting** — writing small programs that run commands for you, take input, make decisions and glue tools together. We look at **scheduling** — arranging for those programs to run automatically, at a fixed time through `cron` or at boot through the startup scripts. And we look at **services** — the background applications that do the actual work of a Linux machine, here the Apache web server, OpenSSH and FTP.

Every section keeps the same shape as Parts 1 and 2: a transcript of the commands that were typed and the output they produced, followed by a paragraph-by-paragraph explanation of what happened and why it matters. Where a traditional tool has a modern replacement — `service` against `systemctl`, `nmap -sP` against `nmap -sn`, classic runlevels against systemd targets — both are shown, because older documentation, interview questions and real machines mix them freely.

> **A word of caution before you start.** Everything in this part either runs code automatically or exposes a service to the network, and both of those are things to practise in a throwaway virtual machine rather than on a machine anyone depends on. A scheduled script runs whether you are watching or not, a web server answers anyone who can reach it, and FTP sends credentials in plain text. Take a snapshot first, work on an isolated host-only network, and read each section before typing anything from it.

### How to read the transcripts

The conventions are the same as in Parts 1 and 2:

* `root@kali:~#` is a shell running as the all-powerful root account, and the trailing `#` is a warning worth heeding. `student@kali:~$` is an ordinary user, and the `$` marks the difference.
* Everything on a prompt line after the prompt is what was typed; every line underneath, up to the next prompt, is output. Nothing runs until Enter is pressed.
* Text inside angle brackets, such as `service <name> start`, is a placeholder you replace with a real value.
* File contents shown without a prompt are the literal bytes of the file, as you would see them in an editor.
* Output was captured on Debian-family systems and is representative rather than byte-identical: addresses, PIDs, package versions and timestamps will differ on your own machine. What matters is the shape of the output.

### Table of contents

1. [Bash scripting basics](#1-bash-scripting-basics)
2. [Scheduling your tasks](#2-scheduling-your-tasks)
3. [Using services in Linux](#3-using-services-in-linux)
4. [Quick reference cheat sheet](#4-quick-reference-cheat-sheet)
5. [Practice exercises](#5-practice-exercises)

---

## 1. Bash scripting basics

Hackers, administrators and developers all eventually face the same problem: a sequence of commands that works perfectly when typed by hand, but that needs to run ten times a day, or fifty times in a row, or at 3 AM when nobody is awake. Typing it by hand does not scale, and retyping it invites mistakes. The answer on Linux is to write the sequence down once, as a small computer program, and let the machine repeat it faithfully forever. Such a program is called a **script**, and the language we write it in is the language of the shell itself — **bash**.

Going back to the basics for a moment: a **shell** is the interface between you and the operating system, the program that reads what you type and asks the kernel to do it. There are several shells available on Linux — `sh`, `bash`, `zsh`, `fish` — and the one we have been using throughout this guide is called **bash**, the Bourne-Again Shell. The bash shell can run any system command, utility or application, which means a bash script can do anything you can do at a prompt: run `nmap`, filter the result through `grep` and `cut`, write the answer to a file, and mail it to you. The only thing you need to get started is a text editor such as `nano` or `vim`; which one you choose makes no difference whatsoever to the script.

### 1.1 Creating your first script file

Let us begin where every script begins: an empty file. We will create it in a small working area, using the directory tools from Part 1, so that the examples have somewhere to live:

```bash
root@kali:~# mkdir -p scripts && cd scripts
root@kali:~/scripts# pwd
/root/scripts
root@kali:~/scripts# touch first_script
root@kali:~/scripts# ls -l
total 0
-rw-r--r-- 1 root root 0 Oct  8 13:01 first_script
```

The file exists and is empty. Two observations are worth making before we write a single line. First, the file has no extension — no `.sh`, no `.bash`. That is deliberate and idiomatic: on Linux, what makes a file executable is its **permission bit**, not its name, so extensions are a hint for humans rather than a requirement of the system. Many authors add `.sh` anyway to signal "this is a shell script" to future readers, and that is a fine convention; just understand that the kernel never looks at it. Second, the file is currently `-rw-r--r--`, readable and writable but not executable. Section 1.4 explains how that changes, and why the change matters.

### 1.2 The shebang: `#!`

To tell the operating system which interpreter should run our script, we use the **shebang** — the two characters `#!` as the very first bytes of the file — followed by the path to the interpreter. Open the file and type:

```
#!/bin/bash
```

That single line says: "when this file is executed, run its contents with the program at `/bin/bash`". The mechanism is worth understanding rather than memorising, because it explains several behaviours that otherwise look like magic. When you execute a file, the kernel reads its first two bytes. If they are `#!`, the kernel treats the rest of the first line as the path to an interpreter, and runs *that program* with the script file as its input. If the first two bytes are anything else, the kernel refuses with `Exec format error` and the shell falls back to running the file as shell commands itself — which usually works for bash syntax, but is slower, less predictable, and wrong for any other language.

Three details catch beginners every time, so let us name them. First, the shebang must be on **line one, column one**: a blank line or even a single space before `#!` breaks it, because the kernel only inspects the first two bytes of the file. Second, there must be **no trailing slash** after the interpreter path. Some older tutorials print `#! /bin/bash/` with a space after the `!` and a slash at the end; the space is harmless (the kernel skips it) but the trailing slash turns the path into a directory, and the script fails with `bad interpreter: Not a directory`. The correct forms are `#!/bin/bash` or `#! /bin/bash`, with nothing after the path. Third, the path must be **absolute and correct**: `#!/bin/bash` works on virtually every Linux system, while `#!/usr/bin/bash` or `#!/bin/sh` point at different programs with subtly different behaviour. The portable alternative `#!/usr/bin/env bash` asks the `env` program to find `bash` on the `PATH`, which is the form to prefer in scripts you intend to share between systems.

### 1.3 `echo` — saying Hello to the World

Like the name suggests, we use `echo` to echo back a message or text we want. Let us extend our script so that it prints "Hello World":

```
#!/bin/bash
echo "Hello World"
```

The script is now two lines: the shebang from the previous section, and one command. That is the entire anatomy of a shell script — there is no boilerplate, no function wrapper, no compilation step. Every line after the shebang is a command that bash reads and runs in order, exactly as if you had typed it at a prompt. `echo` itself simply prints its arguments to standard output followed by a newline, and the double quotes group the two words into a single argument while still allowing variable expansion inside (section 1.5 shows why that matters).

You can already verify that the *contents* are correct without executing anything, by handing the file to bash explicitly:

```bash
root@kali:~/scripts# cat first_script
#!/bin/bash
echo "Hello World"
root@kali:~/scripts# bash first_script
Hello World
```

`bash first_script` works because we are not asking the kernel to execute the file at all — we are starting bash ourselves and telling it which file to read. The shebang is ignored in this mode (bash treats it as a comment, since `#` starts a comment), the permission bits are irrelevant, and the output proves the script's logic is sound. This is the standard way to test a script while you are writing it, and it is also how you run scripts you are not allowed to make executable.

### 1.4 Running our script: `chmod +x` and `./`

Before we can run our script *as a program*, we need to give it permission to do so. As we learned in Part 1, `chmod` with the `+x` flag adds the execute permission:

```bash
root@kali:~/scripts# ls -l first_script
-rw-r--r-- 1 root root 32 Oct  8 13:04 first_script
root@kali:~/scripts# chmod +x first_script
root@kali:~/scripts# ls -l first_script
-rwxr-xr-x 1 root root 32 Oct  8 13:05 first_script
root@kali:~/scripts# ./first_script
Hello World
```

The permission string changed from `-rw-r--r--` to `-rwxr-xr-x`, and the filename in a colour terminal changed from white to green — the shell's way of saying "this can now be executed". Adding `./` before the filename tells the system that we want to execute *this* file, in the current directory. That prefix is not decoration: as Part 1 explained, the current directory is deliberately **not** on the `PATH`, so typing a bare `first_script` makes the shell search the system directories, fail to find it, and answer `command not found`. The `./` is an explicit path — "the file called `first_script` right here" — and it is required for every program you run from the directory you are standing in.

It is worth pausing on what happens when you press Enter on `./first_script`, because three separate mechanisms cooperate. First, the shell checks the execute bit: without it, you get `Permission denied`, which is the system's way of refusing to run a file nobody marked as runnable. Second, the kernel reads the shebang and starts `/bin/bash` with the script as input. Third, bash reads the file line by line and runs each command. If any of those three is broken you get a different, diagnostic error: no execute bit means `Permission denied`; a wrong shebang path means `bad interpreter: No such file or directory`; a Windows-style line ending (`\r\n` instead of `\n`) means `bad interpreter: /bin/bash\r: No such file or directory`, which is the classic symptom of a script edited on Windows and is fixed with `dos2unix` or `sed -i 's/\r$//'`.

A final note for scripts you intend to keep: `chmod +x` adds execute permission for *everyone* (owner, group and others). On a shared machine, `chmod u+x` or `chmod 755`/`700` expresses more precisely who should be allowed to run your code, and the reasoning is the same as for any other file in Part 1, section 6.

### 1.5 Variables and taking user input with `read`

To add more functionality to our bash script, we need to discuss **variables**. A variable is like a bucket: it holds some value inside memory, and that value can be any text (a *string*) or a number. Bash variables need no declaration and no type — you create one by assigning to it, and you read it back by prefixing its name with `$`.

Let us create another script where we learn how to take user input and declare variables:

```
#!/bin/bash
echo "What is your name?"
read name
echo "Welcome, $name"
```

Two new things appear here. `read name` pauses the script, waits for the user to type a line and press Enter, and stores what was typed in the variable called `name`. The last line then uses `$name` to substitute that value back into the output. Now we can finally see the magic of variables as we run the script — being sure to give it executable permission first:

```bash
root@kali:~/scripts# chmod +x greet.sh
root@kali:~/scripts# ./greet.sh
What is your name?
Karan
Welcome, Karan
```

The script printed a prompt, waited, accepted the input `Karan`, and interpolated it into the greeting. That round trip — prompt, read, use — is the skeleton of every interactive script you will ever write.

Four refinements are worth learning immediately, because the naive form has sharp edges. First, `read -p "What is your name? " name` prints the prompt and reads the answer on the *same line*, which is both tidier and one line shorter. Second, `read -s password` reads *silently*, without echoing the characters — the correct form for passphrases and secrets, since the naive `read` leaves a typed password visible on screen and in scrollback. Third, quoting matters on use: `echo "Welcome, $name"` is correct, while `echo Welcome, $name` without quotes breaks the moment the input contains spaces or wildcard characters, because the shell splits the unquoted value into words and expands globs *after* substituting it. The rule from Part 2 applies here with full force: **quote every expansion**. Fourth, the assignment itself must have **no spaces** around the equals sign — `name=Karan` assigns, while `name = Karan` tries to run a command called `name`, exactly as with environment variables in Part 2.

Variables also have a second, non-interactive source that matters more for automation: **positional parameters**, the arguments the caller passed on the command line. Inside a script, `$1` is the first argument, `$2` the second, `$#` is the count of arguments, `$@` is all of them, and `$0` is the script's own name:

```bash
root@kali:~/scripts# cat show_args.sh
#!/bin/bash
echo "Script name: $0"
echo "First argument: $1"
echo "Second argument: $2"
echo "Argument count: $#"
root@kali:~/scripts# chmod +x show_args.sh
root@kali:~/scripts# ./show_args.sh 192.168.1.0 scan
Script name: ./show_args.sh
First argument: 192.168.1.0
Second argument: scan
Argument count: 2
```

A script that takes its target as `$1` instead of prompting with `read` can be run from `cron` (section 2), from a loop, and from another script — which is precisely the difference between a demo and a tool. The scanner in the next section is written in the interactive style of the original tutorial, and exercise 2 at the end of this guide asks you to convert it to arguments.

### 1.6 Decisions and repetition: `if` and `for`

Two small additions turn scripts from fixed sequences into real programs: doing something *only if* a condition holds, and doing something *once for each* item in a list. Both are one-liners in bash, and both repay learning now because every script beyond a toy uses them.

An `if` statement tests a condition and runs a block only when it holds. The most common tests check files (`-f` exists, `-r` readable, `-x` executable), strings (`-z` empty, `-n` non-empty, `=` equal) and numbers (`-eq` equal, `-ne` not equal, `-gt` greater than):

```bash
root@kali:~/scripts# cat check_host.sh
#!/bin/bash
# Usage: ./check_host.sh <address>
if [ -z "$1" ]; then
    echo "Usage: $0 <address>"
    exit 1
fi
if ping -c 1 -W 2 "$1" > /dev/null 2>&1; then
    echo "$1 is up"
else
    echo "$1 is down or unreachable"
fi
root@kali:~/scripts# chmod +x check_host.sh
root@kali:~/scripts# ./check_host.sh
Usage: ./check_host.sh <address>
root@kali:~/scripts# ./check_host.sh 192.168.1.1
192.168.1.1 is up
root@kali:~/scripts# ./check_host.sh 192.168.1.250
192.168.1.250 is down or unreachable
```

Read the script from the top. The `[ -z "$1" ]` test asks "is the first argument empty?" — and if it is, the script prints a usage message and exits with status `1`, the conventional signal for failure (success is always `0`, and `$?` holds the exit status of the last command). The second `if` runs `ping` directly as its condition: an `if` statement tests the *exit status* of a command, so `if ping ...; then` means "if ping succeeded". The `> /dev/null 2>&1` discards ping's output because we only care whether it worked, not what it printed — the redirection technique from Part 1, section 2.10. Note the spaces inside the brackets: `[` is itself a command (a synonym for `test`), so `[ -z "$1" ]` with spaces works and `[-z "$1"]` without them fails with a confusing error.

A `for` loop runs a block once for each word in a list, which makes it the natural way to repeat a check across many targets:

```bash
root@kali:~/scripts# cat sweep.sh
#!/bin/bash
# Ping every address in 192.168.1.1-10 and report the live ones.
for i in 1 2 3 4 5 6 7 8 9 10; do
    if ping -c 1 -W 1 "192.168.1.$i" > /dev/null 2>&1; then
        echo "192.168.1.$i is up"
    fi
done
root@kali:~/scripts# chmod +x sweep.sh
root@kali:~/scripts# ./sweep.sh
192.168.1.1 is up
192.168.1.9 is up
192.168.1.17 is up
```

The loop variable `i` takes each value in turn, and `"192.168.1.$i"` builds the address. The brace form `for i in {1..10}` is equivalent and shorter — the shell expands it before the loop runs — and `for host in $(cat targets.txt)` reads the list from a file instead. This script is slow (ten sequential pings with one-second timeouts) but completely transparent, which is exactly what you want while learning; the `nmap` version in the next section does the same job in a fraction of the time by parallelising the probes.

### 1.7 Creating a simple network scanner

Let us create a script that would be more useful. We will make our script scan the entire network for all the active hosts connected to it and find out their IP addresses.

In order to do so, we will be using **nmap**. It is a simple and essential tool when it comes to network penetration testing. It is used to discover the open ports of a system and the services it is running, and it has the capability to detect the operating system as well. The syntax of nmap is `nmap <type of scan> <target>` — for example `nmap -sV 192.168.1.9` probes the target's ports and identifies versions, as demonstrated in the SSH walkthrough.

We will be creating a script that allows us to scan all the devices' IP addresses connected to our network. For this we use nmap's **ping scan**, which checks for all the alive hosts on the network without probing any ports. In older tutorials this flag is written `-sP` (or `-sp`); on every modern nmap the flag is `-sN`-style `-sn` — "no port scan, host discovery only" — and the old spelling is accepted only as a deprecated alias. Create a new file called `scanner` and let us get started:

```
#!/bin/bash
echo "Enter the IP address"
read ip
nmap -sn "$ip/24" | grep "scan report" | cut -d " " -f 5
```

Working through the pipeline piece by piece, because every stage uses a tool from Part 1. `nmap -sn "$ip/24"` takes the address the user typed — say `192.168.1.17` — appends `/24`, and ping-scans all 256 addresses in that subnet. The output contains one line per live host reading `Nmap scan report for 192.168.1.1`, plus summary lines we do not want. `grep "scan report"` keeps only those per-host lines. `cut -d " " -f 5` splits each surviving line on spaces and prints the fifth field, which is the address itself (`Nmap(1) scan(2) report(3) for(4) 192.168.1.1(5)`).

Two corrections to older versions of this script are worth stating plainly, because copying them verbatim is a rite of passage for beginners. First, the command is `nmap`, not `nma` — a single missing letter that produces `command not found`. Second, the delimiter in `cut -d " "` must be an actual space between the quotes; `cut -d ""` with nothing between them is an error (`cut: the delimiter must be a single character`), since `cut` needs to know what separates the fields. Some versions of the script also end the pipeline with `| head -n -1`, which prints everything *except* the last line — a leftover from nmap output formats that ended with a summary line matching the grep, and harmless but unnecessary today. If you keep it, understand what it does: `head -n -1` is "all but the last line", which silently drops a real result when the trailing summary is absent.

Let us give our new bash script executable permission, and run it. Enter your IP address when prompted:

```bash
root@kali:~/scripts# chmod +x scanner
root@kali:~/scripts# ./scanner
Enter the IP address
192.168.1.17
Starting Nmap 7.94 ( https://nmap.org )
192.168.1.1
192.168.1.9
192.168.1.17
```

Now we can see all the different devices and their IP addresses connected to the network: the gateway at `.1`, the lab target at `.9`, and our own machine at `.17`. (The `Starting Nmap` banner goes to standard error rather than standard output, which is why it passed through the `grep` unfiltered — the two-stream distinction from Part 1 working in our favour for once.)

Before leaving this script, three improvements turn it from a classroom demo into something you would actually keep. First, take the address as an argument instead of prompting, so the script can run unattended: `target="${1:?Usage: $0 <ip>}"` assigns `$1` or aborts with a usage message if it is missing — the `${var:?message}` form from Part 2. Second, validate the input, because passing an empty string to nmap makes it scan a default target you did not intend. Third, quote the expansion (`"$ip/24"`, as written above): an unquoted `$ip` containing spaces or wildcards would be split and globbed before nmap ever sees it. Exercise 2 asks you to apply all three.

### 1.8 Debugging and hardening your scripts

Every script you write will fail, usually at 3 AM, usually when run by `cron` with nobody watching. Three habits catch most failures before they matter.

The first is **tracing**. `bash -x script` runs the script with every command printed to standard error *before* it executes, each line prefixed with `+`, so you can see exactly what the shell thought you asked for:

```bash
root@kali:~/scripts# bash -x first_script
+ echo 'Hello World'
Hello World
```

When a variable expands to something unexpected, the `+` line shows the expanded form rather than the source, which is usually the whole diagnosis. Adding `set -x` on a line inside the script turns tracing on from that point, and `set +x` turns it back off, so you can trace one section of a long script.

The second habit is **strict mode**. By default, bash soldiers on past errors: a failed command, an unset variable, a broken pipe — all are ignored, and the script continues with whatever is left. Three options change that, and they belong at the top of every script you intend to keep, immediately after the shebang:

```
#!/bin/bash
set -euo pipefail
```

`set -e` aborts the script the moment any command fails (exits non-zero outside an `if` test). `set -u` aborts on any reference to an unset variable, turning a typo like `$targer` into an immediate, loud error instead of a silent empty string. `set -o pipefail` makes a pipeline fail if *any* stage fails, rather than reporting only the last stage's status — without it, `nmap ... | grep ... | cut ...` reports success even when nmap itself crashed. Together they convert the three commonest silent failures into immediate stops, which is what you want from automation: a script that halts and complains is infinitely better than one that continues with wrong data.

The third habit is **checking with ShellCheck**. ShellCheck (`apt-get install shellcheck`) is a static analyser that reads your script and reports quoting mistakes, portability problems and classic pitfalls with an explanation and a fix for each:

```bash
root@kali:~/scripts# shellcheck scanner
No issues detected!
root@kali:~/scripts# echo 'cat $file | grep foo' > bad.sh
root@kali:~/scripts# shellcheck bad.sh

In bad.sh line 1:
cat $file | grep foo
    ^---^ SC2086 (info): Double quote to prevent globbing and word splitting.
         ^--^ SC2002 (style): Useless cat. Consider 'cmd < file | ..' or 'cmd file | ..' instead.

```

Running it takes a second and it finds the quoting bug from section 1.5 every time. There is no reason not to use it.

---

## 2. Scheduling your tasks

At times one is required to schedule tasks, such as a backup of the system. In Linux, we schedule jobs we want to run without having to do them manually or even think about them. Here we will learn about the **cron** daemon and **crontab** to run our scripts automatically.

The **crond** daemon runs in the background and wakes up every minute to check the cron tables — the **crontabs** — for commands whose time has come. Altering a crontab is therefore all it takes to execute a task on a schedule: you describe *when* in five time fields, and *what* as the command to run. The scanner from section 1.7 is the perfect example — a network inventory that runs itself every night while you sleep.

### 2.1 The cron table format

The system-wide cron table lives at `/etc/crontab`. It has seven fields per line: the first five specify *when* the command runs, the sixth names the *user* to run it as, and the seventh is the *command* itself:

```
# m h dom mon dow user  command
55 23 * * *   root  /root/scanner
```

Personal crontabs — the ones you edit with `crontab -e` in section 2.4 — have only **six** fields, because the user is implicit (the job runs as whoever owns the crontab), so the user column is absent. That difference is the single commonest source of "my cron job works in one place but not the other": a six-field line pasted into `/etc/crontab` is read as "run as user `root/scanner`" and fails, while a seven-field line pasted into a personal crontab is read as a command starting with a username and fails differently. Know which file you are editing.

Here is a table summarising the five time fields, which are identical in both formats:

| Field | Unit | Values | Meaning of `*` |
| --- | --- | --- | --- |
| 1 | Minute | 0–59 | every minute |
| 2 | Hour | 0–23 | every hour |
| 3 | Day of the month | 1–31 | every day |
| 4 | Month | 1–12 (or jan–dec) | every month |
| 5 | Day of the week | 0–7 (or sun–sat; 0 and 7 are both Sunday) | every day of the week |

A few readings to fix the grammar. `55 23 * * *` means "at 23:55, every day" — minute 55, hour 23, and every day, month and weekday. `0 3 * * 0` means "at 03:00 every Sunday". `*/15 * * * *` means "every fifteen minutes" — the `/` is a step value, so `0 9-17 * * 1-5` means "at the top of every hour from 09:00 to 17:00, Monday to Friday". `0 0 1 * *` means "at midnight on the first of every month". Lists are allowed too: `0 8,20 * * *` runs at 08:00 and 20:00 daily. One subtlety deserves a warning: when *both* the day-of-month and day-of-week fields are restricted (neither is `*`), cron runs the job when *either* matches, not when both do — so `0 0 1 * 0` runs on the first of the month *and* every Sunday, which surprises everyone once.

### 2.2 Checking that the cron daemon is running

First, let us check whether the cron daemon is running or not:

```bash
root@kali:~# service cron status
● cron.service - Regular background program processing daemon
     Loaded: loaded (/lib/systemd/system/cron.service; enabled; vendor preset: enabled)
     Active: inactive (dead) since Wed 2026-10-07 18:12:02 UTC; 19h ago
```

Since it shows inactive, we can start the service:

```bash
root@kali:~# service cron start
root@kali:~# service cron status
● cron.service - Regular background program processing daemon
     Loaded: loaded (/lib/systemd/system/cron.service; enabled; vendor preset: enabled)
     Active: active (running) since Thu 2026-10-08 13:20:11 UTC; 3s ago
   Main PID: 2317 (cron)
      Tasks: 1 (limit: 2261)
     Memory: 512.0K
        CPU: 8ms
     CGroup: /system.slice/cron.service
             └─2317 /usr/sbin/cron -f -P
```

The `service` command is the traditional wrapper for managing services, and section 3.1 explains it in full. On a modern systemd machine it forwards to `systemctl`, so `service cron status` and `systemctl status cron` are the same operation spelled two ways, and `service cron start` is `systemctl start cron`. The status output tells you three things worth reading: `Loaded: ... enabled` means the service starts automatically at boot (section 2.6 explains how that is arranged); `Active: active (running)` means it is running *now*; and the `Main PID` line names the actual daemon process. A service can be enabled but stopped (starts at next boot, not running now) or running but disabled (running now, will not survive a reboot) — and a scheduled job needs it to be *both*.

If the daemon is missing entirely (`Unit cron.service could not be found`), install it with `apt-get install cron` and enable it with `systemctl enable --now cron`, which starts it now *and* arranges for it to start at every boot.

### 2.3 Editing your crontab with `crontab -e`

Now, open the cron table in order to edit it. Type `crontab` in the terminal, followed by the `-e` flag (`e` stands for edit):

```bash
root@kali:~# crontab -e
no crontab for root - using an empty one

Select an editor.  To change later, run 'select-editor'.
  1. /bin/nano        <---- easiest
  2. /usr/bin/vim.basic
  3. /usr/bin/vim.tiny
  4. /bin/ed

Choose 1-4 [1]:
```

It gives you an option to select any text editor. We will be choosing nano as we have been working with it so far — so enter `1`. (Your choice is remembered in `~/.selected_editor`, and `select-editor` lets you change it later. Setting `export EDITOR=nano` in `~/.bashrc`, as Part 2 suggests, skips this prompt entirely on most systems.)

The editor opens on your personal crontab, which starts as comments explaining the format. Now scroll down and simply enter the time fields we learned about, to schedule the task. Let us say we want to see all the devices connected to our network before we sleep, so we will execute our scanner script every day at 11:55 PM automatically. Type the following on its own line:

```
55 23 * * * /root/scripts/scanner >> /var/log/scanner.log 2>&1
```

Save and exit (`Ctrl+O`, Enter, `Ctrl+X` in nano), and cron confirms the installation:

```bash
crontab: installing new crontab
root@kali:~# crontab -l
# m h  dom mon dow   command
55 23 * * * /root/scripts/scanner >> /var/log/scanner.log 2>&1
```

`crontab -l` (*list*) prints the installed table, which is how you verify that what you saved is what cron will run. Note two things about the command as written. First, it uses the **absolute path** `/root/scripts/scanner`, because cron runs with a minimal `PATH` (`/usr/bin:/bin`) and no concept of "the directory I was standing in" — a bare `scanner` or `./scanner` will not be found. For the same reason the script itself should use absolute paths internally, or `cd` to its working directory first. Second, the output is appended to a log file with `>> /var/log/scanner.log 2>&1`, because a cron job has no terminal: anything it prints is either mailed to the job's owner (if mail is configured, which on most lab machines it is not) or lost. Logging to a file is how you find out *whether* the job ran and *what* it said — and the `>>` (append) rather than `>` (overwrite) preserves yesterday's evidence.

Do not wait until 23:55 to find out whether the job works. Test it first with a near-future time — `* * * * *` runs every minute, which is the standard way to verify a job end to end — watch `/var/log/scanner.log` appear with `tail -f`, and only then set the real schedule. And check the system log if nothing happens: `grep CRON /var/log/syslog` shows every job cron started, which distinguishes "cron never ran my job" (a schedule or daemon problem) from "my job ran and failed" (a script problem whose answer is in your log file).

### 2.4 Cron pitfalls worth knowing

Most cron failures fall into four categories, and knowing them saves hours. **Environment** is the first: cron provides almost nothing — a minimal `PATH`, no `HOME` quirks, no aliases, `/bin/sh` as the shell unless the crontab sets `SHELL=/bin/bash`. Anything your script inherits from your interactive shell (a `PATH` entry for `/usr/local/bin`, an `EDITOR`, an SSH agent) is absent at 23:55. The fix is to make scripts self-sufficient: absolute paths, explicit variables, and the strict mode from section 1.8 so failures are loud.

**Percent signs** are the second: inside a crontab, an unescaped `%` means "newline, and pipe the rest to the command's standard input". A `date +%F` in a crontab line does not print the date — it truncates the command. Escape them (`date +\%F`) or, better, put complex commands in a script file and call the script from cron.

**Permissions** are the third: the script must be executable (`chmod +x`) by the user the job runs as, and every file it writes must be writable by that user. A personal crontab runs as its owner; `/etc/crontab` runs as the user named in the sixth field. A job that works when you run it by hand but fails from cron is almost always an environment or permission difference, not a schedule problem.

**Overlapping runs** are the fourth: cron will happily start a second copy of a job while the first is still running. For a quick scanner that does not matter; for a slow backup it means two backups fighting over the same files. The standard guard is `flock`: `55 23 * * * /usr/bin/flock -n /tmp/scanner.lock /root/scripts/scanner` runs the scanner only if no other copy holds the lock.

Finally, a security note that belongs in any discussion of scheduled jobs. `cron` is a classic **persistence mechanism**: a line in a crontab re-runs an attacker's command every minute, survives reboots, and is invisible to anyone who never looks. Defensively, that means unfamiliar cron entries — in personal crontabs (`crontab -l` for each user), in `/etc/crontab`, and in `/etc/cron.d/`, `/etc/cron.hourly/` and friends — deserve the same suspicion as unfamiliar startup scripts. A job that downloads and runs a remote script, or that reappears after being removed, is a finding, not a quirk.

### 2.5 Starting jobs at boot: rc scripts and runlevels

Whenever you switch on your Linux machine, a number of processes run which help in setting up the environment that you will use. The scripts that run are known as **rc scripts**. When booting up, the kernel starts the **init** system — historically a daemon whose configuration lived under `/etc/init.d/` — which is responsible for running these scripts in the right order.

The next thing we should know about is **Linux runlevels**. Linux has multiple runlevels, which tell the system what services should be started at bootup. Here is a table indicating the standard meanings:

| Runlevel | Purpose |
| --- | --- |
| 0 | Halt the system |
| 1 | Single-user / minimal (rescue) mode |
| 2–5 | Multi-user modes (normal operation; Debian-family systems boot to runlevel 2 by default) |
| 6 | Reboot the system |

In the classic SysV init design, each runlevel has a directory (`/etc/rc2.d/`, `/etc/rc3.d/`, …) full of symbolic links to the real scripts in `/etc/init.d/`, and a link starting with `S` means "start this service when entering this runlevel" while `K` means "stop it". Managing those links by hand would be miserable, so Debian-family systems provide the `update-rc.d` command, which enables you to add or remove services from the rc sequence.

Let us add a service to the rc configuration now. We will enable MySQL to start every time we boot. Simply write the service name after `update-rc.d` and follow it with `defaults` (the available actions are `remove`, `defaults`, `disable` and `enable`):

```bash
root@kali:~# update-rc.d mysql defaults
Synchronizing state of mysql.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install enable mysql
```

Now restart the system and you will see MySQL has already been started. We can check for it using the `ps aux` and `grep` combination we learned in Part 2:

```bash
root@kali:~# ps aux | grep -v grep | grep mysql
mysql        812  0.1  2.4 1843200 96512 ?       Ssl  09:41   0:01 /usr/sbin/mysqld
```

One process, owned by the `mysql` user, running the `mysqld` daemon — the database started itself at boot with nobody logged in, which is the entire point of the exercise. (The `grep -v grep` in the middle discards the grep process itself, the self-match from Part 2, section 2.3.)

A modern footnote is unavoidable here, because the transcript above already shows it: on a systemd machine, `update-rc.d` is a compatibility shim that forwards to `systemctl enable`. The native commands are `systemctl enable mysql` (start at boot), `systemctl disable mysql` (do not start at boot), and `systemctl is-enabled mysql` (ask which is true now). Runlevels, likewise, have systemd equivalents called **targets**: runlevel 1 is `rescue.target`, runlevels 2–4 are `multi-user.target`, runlevel 5 is `graphical.target`, and `systemctl get-default` shows which one the machine boots into. Learn both vocabularies — `update-rc.d` and runlevels for reading older documentation and passing interviews, `systemctl enable` and targets for administering anything built in the last decade — and verify either way with `ps`, because the process list is the ground truth no abstraction can lie about.

---

## 3. Using services in Linux

A **service** in Linux is the common way to denote an application that runs in the background for you to use. Multiple services come pre-installed on a Linux machine. One of the most common is the **Apache web server**, which helps us create and deploy web servers; another is **OpenSSH**, which allows you to connect to another machine; a third is **FTP**, the classic file-transfer service. Let us dig deeper into these services to understand their inner workings — which, as ever, is what helps us use them properly and assess them critically.

### 3.1 Playing with services: start, stop, status, restart

Before we begin, we should know how to manage services. The basic syntax is:

```
service <service_name> <start|stop|restart|status>
```

Let us start the apache2 server:

```bash
root@kali:~# service apache2 start
root@kali:~# service apache2 status
● apache2.service - The Apache HTTP Server
     Loaded: loaded (/lib/systemd/system/apache2.service; enabled; vendor preset: enabled)
     Active: active (running) since Thu 2026-10-08 13:31:02 UTC; 4s ago
       Docs: https://httpd.apache.org/docs/2.4/
   Main PID: 3120 (apache2)
      Tasks: 55 (limit: 2261)
     Memory: 5.4M
        CPU: 42ms
     CGroup: /system.slice/apache2.service
             ├─3120 /usr/sbin/apache2 -k start
             ├─3121 /usr/sbin/apache2 -k start
             └─3122 /usr/sbin/apache2 -k start
```

Now we use the `status` argument to check whether the service is up or not — and it is: `Active: active (running)`, with a main process and two worker children, which is Apache's standard shape (a parent that manages the socket and children that answer requests). To stop this service, we type:

```bash
root@kali:~# service apache2 stop
root@kali:~# service apache2 status
○ apache2.service - The Apache HTTP Server
     Loaded: loaded (/lib/systemd/system/apache2.service; enabled; vendor preset: enabled)
     Active: inactive (dead) since Thu 2026-10-08 13:32:44 UTC; 2s ago
```

At times — when the service had a faulty start, or you have changed a configuration file — you want to restart it so the changes take effect. That is what the `restart` option does: it stops the service completely and starts it again:

```bash
root@kali:~# service apache2 restart
root@kali:~# service apache2 status
● apache2.service - The Apache HTTP Server
     Active: active (running) since Thu 2026-10-08 13:33:19 UTC; 3s ago
   Main PID: 3240 (apache2)
```

Note the new Main PID (3240 where 3120 was): `restart` really did replace the process rather than merely reloading it, which is why existing connections are dropped. Two refinements complete the vocabulary. `service apache2 reload` asks the running daemon to re-read its configuration *without* dropping connections — the polite form, and the right one after editing a config file on a production host. And every `service` invocation has a `systemctl` twin that does the same thing natively: `systemctl start apache2`, `systemctl stop apache2`, `systemctl restart apache2`, `systemctl reload apache2`, `systemctl status apache2` — plus `systemctl enable apache2` and `systemctl disable apache2`, which control whether the service starts at boot (the `update-rc.d` of section 2.5 in modern dress). If a service is missing (`Unit apache2.service could not be found`), it simply is not installed: `apt-get install apache2` provides it, following the package-manager procedure from Part 1.

### 3.2 Creating an HTTP web server with Apache

More than half of the world's web servers have at one time run Apache, and it remains one of the most commonly deployed services on Linux. As a penetration tester it is critical to understand how Apache works — its document root, its default page, its logs — because half the web servers you will ever assess are instances of it. So let us deploy our own web server and get familiar with Apache.

Start the apache2 service if you have not already (section 3.1), and confirm it is listening. A web server speaks HTTP on port 80 by default, and `ss` shows the socket:

```bash
root@kali:~# ss -tlnp | grep apache
LISTEN 0      511                *:80              *:*    users:(("apache2",pid=3240,fd=4))
```

Now we turn to the HTML file that gets displayed in the browser. Apache's default web page lives at `/var/www/html/index.html`. Let us open it with `nano` and look at the default contents:

```bash
root@kali:~# nano /var/www/html/index.html
```

We see the HTML code present by default — the Debian/Ubuntu "It works!" page, a full HTML document with styling, which is longer than it needs to be for learning. Replace the body with something of your own so that you can prove to yourself the page is really yours:

```html
<!DOCTYPE html>
<html>
<head><title>Lab web server</title></head>
<body>
<h1>Hello from Apache</h1>
<p>If you can read this, the web server is serving /var/www/html/.</p>
</body>
</html>
```

Save the file (`Ctrl+O`, Enter, `Ctrl+X`). No restart is needed: Apache reads the file from disk on every request, so the change is live the moment you save. Now, to see what the Apache server displays, go to the browser and type:

```
http://localhost
```

The page renders with your heading — and `localhost` works because it always means "this machine" (resolving to `127.0.0.1` via `/etc/hosts`, as Part 2 explains). Prefer the terminal? `curl` fetches the same page without a browser:

```bash
root@kali:~# curl -s http://localhost/
<!DOCTYPE html>
<html>
<head><title>Lab web server</title></head>
<body>
<h1>Hello from Apache</h1>
<p>If you can read this, the web server is serving /var/www/html/.</p>
</body>
</html>
root@kali:~# curl -s -o /dev/null -w "%{http_code}\n" http://localhost/
200
```

The `200` is the HTTP status code for success, which is the number to check in scripts: a `200` means "served", a `403` means "forbidden" (usually a file-permission problem — the Apache user `www-data` cannot read the file), and a `404` means the path does not exist under the document root.

Three things about this setup matter for everything that follows. First, the **document root** `/var/www/html/` is the directory Apache maps to `/` in URLs, so `http://localhost/report.html` is the file `/var/www/html/report.html` — and there is no way to fetch anything *above* it, which is what keeps `/etc/shadow` safe from browsers. Second, every request is **logged**: `/var/log/apache2/access.log` records who asked for what, and `/var/log/apache2/error.log` records what went wrong, which makes the logs the first place to look both when debugging your own server and when investigating someone else's. Third, the files must be **readable by `www-data`**, the unprivileged user Apache runs as: a page saved with `600` permissions owned by root returns `403 Forbidden`, and the fix is `chown`/`chmod` from Part 1, not running Apache as root. From a security-testing point of view, a default Apache page on a target also leaks the operating system family (the Debian default page is distinctive) and deserves the same attention as any other banner: check the version with `curl -sI http://target/ | grep -i server`, and note that production servers should hide it with `ServerTokens Prod`.

### 3.3 Getting familiar with OpenSSH

**Secure Shell**, or SSH, is basically what enables us to connect to a terminal on a remote system, securely. Unlike its ancestor **telnet**, which was used quite some years back and sent everything — including the password — in plain text for anyone on the network to read, the channel SSH uses for its communication is **encrypted**, and hence far more secure. There is no legitimate reason to use telnet for remote administration today, and finding it offered on a target is itself a finding.

Again, before we start using the SSH service, we have to make sure it is running:

```bash
root@kali:~# service ssh status
● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/lib/systemd/system/ssh.service; enabled; vendor preset: enabled)
     Active: active (running) since Thu 2026-10-08 09:52:41 UTC; 3h ago
   Main PID: 812 (sshd)
      Tasks: 1 (limit: 2261)
     Memory: 1.2M
        CPU: 12ms
     CGroup: /system.slice/ssh.service
             └─812 "sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups"
```

(If it is not running, `service ssh start` starts it; if it is not installed, `apt-get install openssh-server` provides it — the lab setup from the SSH walkthrough.) Now, to connect to a remote system and get access to its terminal, we type `ssh` followed by `<username>@<address>`. Let us connect to another lab machine:

```bash
root@kali:~# ssh ignite@192.168.1.11
The authenticity of host '192.168.1.11 (192.168.1.11)' can't be established.
ED25519 key fingerprint is SHA256:9xQ2rTgZ7mK1pRcVq4Yd1sB3nE6jWfH8aLoX0uMiPn.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.1.11' (ED25519) to the list of known hosts.
ignite@192.168.1.11's password:
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-91-generic x86_64)
ignite@ubuntu:~$ hostname
ubuntu
ignite@ubuntu:~$ whoami
ignite
ignite@ubuntu:~$ exit
logout
Connection to 192.168.1.11 closed.
```

We have successfully connected to another machine called **ubuntu** with the user **ignite**: the prompt changed to `ignite@ubuntu:~$`, and `hostname` and `whoami` confirm where we are and who we are there. The host-key prompt on first connection is the client's protection against interception — verify the fingerprint out of band on a real engagement, rather than typing `yes` blindly — and the password characters are not echoed as you type, which is normal.

This section is deliberately a first taste: SSH key authentication, port changes, tunnelling, file transfer and persistence all build directly on this connection, and they are covered end to end in the companion [SSH Penetration Testing walkthrough](04-ssh-pentest-guide.md). The habits to take from here are small but load-bearing: confirm the service is running before debugging the client, read the host-key prompt instead of clicking through it, and type `exit` (or `Ctrl+D`) to close a session rather than just closing the terminal.

### 3.4 Working with FTP

Let us talk about the **File Transfer Protocol**, or FTP. This protocol is generally used — as the name suggests — for the transfer of files, and here we will try connecting to an FTP server and downloading files from it via the **`ftp`** command.

To access an FTP server, we type `ftp` followed by the domain name or the IP address. Here is an example using a well-known public test server:

```bash
root@kali:~# ftp ftp.cesca.es
Connected to ftp.cesca.es.
220---------- Welcome to Pure-FTPd [privsep] [TLS] ----------
220-You are user number 3 of 50 allowed.
220-Local time is now 13:41. Server port: 21.
220-This is a private system - No anonymous login
220-IPv6 connections are also welcome on this server.
220 You will be disconnected after 15 minutes of inactivity.
Name (ftp.cesca.es:root):
```

The `220` lines are the server's greeting banner — note what it advertises (the software, the limits, the rules) before you have authenticated at all. Now it is going to ask you to enter a name. On servers that permit it, we can type **anonymous** here:

```
Name (ftp.cesca.es:root): anonymous
331 Anonymous login OK, send your complete email address as your password
Password:
```

Now it asks for the password, and the convention for anonymous FTP is to type your email address — though most servers accept literally **anonymous** as well:

```
Password:
230 Anonymous access granted, restrictions apply
Remote system type is UNIX.
Using binary mode to transfer files.
ftp>
```

As we can see, we have been logged in successfully, and the `ftp>` prompt is the file-transfer shell. Note the `Using binary mode` line: FTP has two transfer modes, and *binary* (also called *image*) transfers bytes exactly, while *ascii* translates line endings and corrupts anything that is not plain text. Binary is correct for executables, archives, images and effectively everything; a file that arrives with the right size but a wrong checksum was usually fetched in the wrong mode, and the `binary` and `ascii` commands switch between them.

Now, with the help of the basic navigation commands we learned in Part 1, we can use `ls` to list the contents — the FTP shell deliberately mirrors the Unix commands:

```
ftp> ls
229 Entering Extended Passive Mode (|||30122|)
150 Accepted data connection
drwxr-xr-x    2 0        0            4096 Jan 10  2024 debian
drwxr-xr-x    2 0        0            4096 Jan 10  2024 ubuntu
-rw-r--r--    1 0        0             220 Jan 10  2024 README
226-Options: -l
226 3 matches total
```

Navigate around for a file you want to download — `cd ubuntu/release` moves down, `cd ..` moves up, `pwd` shows where you are, exactly as in a shell. Let us download the file at `ubuntu/release/favicon.ico`. Simply type `get` followed by the file name:

```
ftp> cd ubuntu/release
250 OK. Current directory is /ubuntu/release
ftp> get favicon.ico
local: favicon.ico remote: favicon.ico
229 Entering Extended Passive Mode (|||30144|)
150 Accepted data connection
226-File successfully transferred
226 0.012 seconds (measured here), 123.45 Kbytes per second
1150 bytes received in 0.01 secs (123.4 kB/s)
```

To exit the FTP session, type **`bye`** (or `quit` — the two are synonyms). Now we can `ls` in our own shell and see the file we just downloaded:

```bash
ftp> bye
221-Goodbye. You uploaded 0 and downloaded 1 kbytes.
221 Logout.
root@kali:~# ls -l favicon.ico
-rw-r--r-- 1 root root 1150 Oct  8 13:44 favicon.ico
root@kali:~# file favicon.ico
favicon.ico: MS Windows icon resource - 1 icon, 16x16
```

The `file` command confirms the download is what it claims to be — a real icon file, not an error page saved under the wrong name, which is the standard check after any transfer.

Three closing notes, and the third is the important one. First, the `229 Entering Extended Passive Mode` lines are the client and server negotiating the *data connection*: FTP uses one connection for commands (port 21) and a second, negotiated connection for every listing and transfer, which is why FTP misbehaves behind firewalls and NAT in ways that SSH-based transfer never does. If listings hang while commands work, the data connection is being filtered — try toggling `passive`. Second, `mget` and `mput` transfer multiple files with wildcard matching (`mget *.txt`, with `prompt` toggling the per-file confirmation), and `put` uploads by the same rules as `get` downloads. Third — and this is why FTP appears in a security-oriented guide at all — **classic FTP sends the username, the password and every file in plain text**. Anyone on the network path reads all three with a packet capture. Anonymous download of public files is FTP's one remaining legitimate use; for anything authenticated, use SFTP or SCP over SSH instead (both covered in the [SSH walkthrough](04-ssh-pentest-guide.md)), which provide the same operations over an encrypted channel. Finding an internal FTP server that accepts anonymous *uploads* is a genuine finding: it is free, unauthenticated storage for anyone who finds it.

### 3.5 Seeing which services are listening

With three services started, stopped and restarted across this section, the natural closing question is: what is actually listening *now*? Two commands answer it from opposite directions. `ss -tlnp` asks the kernel which TCP sockets are listening and which processes own them — the ground truth:

```bash
root@kali:~# ss -tlnp
State  Recv-Q Send-Q Local Address:Port  Peer Address:Port Process
LISTEN 0      511                *:80             *:*     users:(("apache2",pid=3240,fd=4))
LISTEN 0      128          0.0.0.0:22             *:*     users:(("sshd",pid=812,fd=3))
LISTEN 0      128             [::]:22                *:*  users:(("sshd",pid=812,fd=4))
LISTEN 0      5          127.0.0.1:3306            *:*     users:(("mysqld",pid=812,fd=21))
```

Apache on port 80, SSH on port 22, MySQL on loopback port 3306 — each line names the address (a `*` or `0.0.0.0` means "reachable from anywhere"; `127.0.0.1` means "this machine only"), the port, and the owning process. `systemctl list-units --type=service --state=running` answers the same question from the service manager's point of view, naming the *units* rather than the sockets:

```bash
root@kali:~# systemctl list-units --type=service --state=running
  UNIT           LOAD   ACTIVE SUB     DESCRIPTION
  apache2.service loaded active running The Apache HTTP Server
  cron.service   loaded active running Regular background program processing daemon
  mysql.service  loaded active running MySQL Community Server
  ssh.service    loaded active running OpenBSD Secure Shell server
```

Between them there is no hiding place: a service that is running but not listening is waiting for a socket (or has failed to bind one — check its logs with `journalctl -u <name>`), and a listening socket with no corresponding service is a process someone started by hand. Running both commands is the thirty-second audit to perform on any machine you inherit — and the fastest way to confirm that the service you just configured is really the service that is answering.

---

## 4. Quick reference cheat sheet

### Bash scripting

| Construct | What it does | Typical use |
| --- | --- | --- |
| `#!/bin/bash` | shebang — runs the file with bash (line 1, column 1) | first line of every script |
| `#!/usr/bin/env bash` | portable shebang via `PATH` lookup | scripts shared between systems |
| `chmod +x script` | makes a file executable | `chmod +x scanner` |
| `./script` | runs a file in the current directory | `./scanner` |
| `bash script` | runs a file without needing `+x` | `bash -x script` to trace |
| `echo "text"` | prints text to standard output | `echo "Hello World"` |
| `name=value` | creates/changes a variable (no spaces) | `target=192.168.1.0` |
| `$name` / `"${name}"` | reads a variable back (quote it) | `echo "target is $target"` |
| `read name` | reads one line of input into `$name` | `read -p "Name? " name` |
| `read -s secret` | reads without echoing (passwords) | `read -s -p "Pass: " pw` |
| `$1`, `$2`, `$#`, `$@`, `$0` | arguments, argument count, all args, script name | `target="${1:?Usage: $0 <ip>}"` |
| `$?` | exit status of the last command (0 = success) | `ping -c1 h \|\| echo down` |
| `if cmd; then … fi` | runs a block when a command succeeds | `if ping -c1 "$h"; then …` |
| `[ -z "$s" ]` / `[ -f "$f" ]` | tests: empty string / file exists | `if [ -z "$1" ]; then exit 1; fi` |
| `for i in …; do … done` | repeats a block for each item | `for i in {1..10}; do … done` |
| `exit N` | ends the script with status N | `exit 1` on usage error |
| `set -euo pipefail` | strict mode: fail fast, loudly | line 2 of every kept script |
| `bash -x script` | traces execution (`+` lines) | debugging |
| `shellcheck script` | static analysis with fixes | before every commit |

### Scheduling and startup

| Command | What it does | Typical use |
| --- | --- | --- |
| `service cron status` / `start` | checks / starts the cron daemon | `service cron start` |
| `systemctl enable --now cron` | starts cron now and at every boot | as shown |
| `crontab -e` / `-l` / `-r` | edits / lists / removes your crontab | `crontab -l` |
| `55 23 * * * cmd` | runs `cmd` daily at 23:55 | `55 23 * * * /root/scripts/scanner` |
| `*/15 * * * * cmd` | runs `cmd` every 15 minutes | testing: `* * * * * cmd` |
| `cmd >> file 2>&1` | appends a job's output to a log | every cron line |
| `grep CRON /var/log/syslog` | shows which jobs cron started | debugging a silent job |
| `flock -n lock cmd` | skips a run while one is active | `flock -n /tmp/scan.lock …` |
| `update-rc.d svc defaults` | enables a SysV service at boot | `update-rc.d mysql defaults` |
| `systemctl enable svc` | enables a systemd service at boot | `systemctl enable mysql` |
| runlevels `0 1 2–5 6` | halt / single-user / multi-user / reboot | `systemctl get-default` |

### Services

| Command | What it does | Typical use |
| --- | --- | --- |
| `service <n> start\|stop\|restart\|status` | controls a service (classic) | `service apache2 restart` |
| `systemctl start\|stop\|restart\|reload\|status <n>` | controls a service (modern) | `systemctl reload apache2` |
| `systemctl enable\|disable <n>` | service starts at boot or not | `systemctl enable ssh` |
| `ss -tlnp` | listening TCP sockets + owners | `ss -tlnp \\| grep apache` |
| `systemctl list-units --type=service --state=running` | running services | as shown |
| `journalctl -u <n>` | a service's logs | `journalctl -u apache2` |
| `curl -s http://localhost/` | fetches the local web page | `curl -sI http://target/` |
| `ssh <user>@<host>` | encrypted remote shell | `ssh ignite@192.168.1.11` |
| `ftp <host>` | classic file-transfer shell | `ftp ftp.cesca.es` |
| FTP `ls cd get put bye` | list / move / download / upload / quit | `get favicon.ico` |
| FTP `binary` / `ascii` | byte-exact / text transfer mode | `binary` before any download |

---

## 5. Practice exercises

The material in this part only becomes yours when you have typed it yourself, so here is a short sequence that exercises everything above in one coherent flow. Work on a virtual machine or in a container, on an isolated network, and take a snapshot first.

1. **First script.** Write `first_script` from sections 1.1–1.4 from memory: shebang, one `echo`, `chmod +x`, run with `./`. Then deliberately break it three ways — remove the execute bit, add a trailing slash to the shebang, run it as a bare name with no `./` — and record the exact error each produces. Fix all three.
2. **Harden the scanner.** Rewrite the section 1.7 scanner so that it takes the target network as `$1` instead of prompting, aborts with a usage message when no argument is given, quotes every expansion, starts with `set -euo pipefail`, and passes `shellcheck` cleanly. Run it against your lab subnet and compare its output with the original.
3. **Decide and repeat.** Write a script that reads hostnames from a file (one per line), pings each once with a short timeout, and prints two summaries at the end: the hosts that answered and the hosts that did not. Use a `for` loop, an `if`, and exit status `0` only if every host answered.
4. **Schedule it.** Install your scanner from exercise 2 under `/usr/local/bin/`, schedule it with `crontab -e` to run every minute while you test, confirm with `tail -f` on its log file and `grep CRON /var/log/syslog` that it runs, then change the schedule to once nightly. Explain in writing why the cron line uses an absolute path and where the output goes.
5. **Boot persistence.** Enable a service of your choice to start at boot with both `update-rc.d` and `systemctl enable`, reboot, and prove with `ps` and `ss -tlnp` that it started without anyone logging in. Then disable it again and confirm it stays down after a second reboot.
6. **Serve a page.** Start Apache, replace `/var/www/html/index.html` with a page of your own, and fetch it with both a browser (`http://localhost`) and `curl`. Break the permissions deliberately (`chmod 600` as root), observe the `403`, read the explanation in `/var/log/apache2/error.log`, and fix it with `chown`/`chmod`.
7. **Connect and copy.** Start the SSH service, connect to a second lab machine with `ssh user@host`, and run one remote command. Then connect to a public anonymous FTP server, list a directory, download one small file in binary mode, verify it with `file`, and quit. Write one paragraph explaining why the FTP credentials and the SSH credentials had completely different exposure on the network.

If you can complete these seven exercises without looking anything up, you are comfortable with the automation layer of Linux — writing programs that do the typing for you, arranging for them to run on their own, and serving the network rather than merely using it — and you have covered everything in this three-part guide.

---

## Closing notes

The three topics in this part share a theme that runs through the whole guide: they are all about **leverage**. A script multiplies one careful thought into a thousand faithful executions; a schedule multiplies one working command into years of unattended operation; a service multiplies one configured machine into an answer for every client on the network. That is what makes them powerful, and it is what makes their failure modes worth respecting — a typo in an interactive command produces an error message, while the same typo in a scheduled script or a world-readable service produces wrong behaviour, silently, for everyone, until somebody notices.

Three habits from this part are worth carrying into everything you do next. The first is to *make scripts strict and testable*: a shebang, `set -euo pipefail`, quoted expansions, arguments instead of prompts, and a `shellcheck` run before anything is scheduled. The second is to *log and verify automation*: every cron line appends to a log file, every schedule is tested at one-minute intervals before it is set for real, and `grep CRON /var/log/syslog` is the first stop when a job is silent. The third is to *know what is listening*: `ss -tlnp` and the service list are the thirty-second audit for any machine you touch, and anything there that you cannot explain is the most important thing on the machine.

With this part complete, you have the full foundation the three guides build together: Part 1 gave you the shell, files, text, packages and permissions; Part 2 gave you networks, processes and the environment; this part gave you scripting, scheduling and services. Natural next steps from here are **service-specific deep dives** — the [SSH walkthrough](04-ssh-pentest-guide.md) in this same collection works the SSH service end to end, from reconnaissance through hardening — followed by **shell scripting at larger scale** (functions, argument parsing with `getopts`, and configuration management with tools like Ansible), **systemd in depth** (writing your own unit files and timers as the modern replacement for rc scripts and cron), and **log analysis and monitoring**, where the logs this part introduced become the raw material for detecting everything the earlier parts taught you to do.
