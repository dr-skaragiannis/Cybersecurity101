# Linux for Beginners (Part 3): Bash Scripting, Automation and Services

*This is Part 3 of a three-part guide. It continues [Linux for Beginners (Part 2): Networks, Processes and Environment Variables](02%20-%20Linux%20for%20Beginners%20-%20Part%202.md), which in turn follows [Linux for Beginners (Part 1): The Shell, Files, Text, Packages and Permissions](01%20-%20Linux%20for%20Beginners%20-%20Part%201.md). It assumes you are comfortable with both: the shell, the file system, text, packages and permissions from Part 1, and interfaces, processes and environment variables from Part 2.*

## Introduction

Parts 1 and 2 were about *using* a Linux machine: finding your way around it, editing what is on it, reading its network configuration, seeing what it is running and shaping the environment it hands to your programs. This part is about making the machine do the work. It has three parts, and they build on one another neatly.

First, **Bash scripting**. Everything you have typed so far has been a conversation with the shell: one command at a time, each one waiting for you to type the next. A script turns a sequence of those commands into a file that the machine can run without you — which is how anything repetitive gets done reliably, and how you stop being the slowest and most error-prone component in your own workflow. You will write a script that takes input, makes a decision, and calls another tool.

Second, **scheduling**. A script that exists is useful; a script that runs itself at 23:55 every night without anyone remembering to launch it is a different kind of useful. That is what the `cron` daemon is for, and it is also how a great deal of legitimate administration — and a fair number of compromises — survive reboots. We will also look at the older machinery underneath: the `rc` scripts and runlevels that decide what starts when a machine boots, and how those map onto the systemd model used by every current distribution.

Third, **services**. A service is a program that keeps running in the background, waiting for something to talk to it. Apache is a service that serves web pages, OpenSSH is a service that accepts remote shells, FTP is a service that moves files. Learning to start, stop, inspect and configure them is the last big gap between "I can use Linux" and "I understand what this machine is doing" — and it is also where the security-relevant surface of a host lives, because a service is a listening socket with a program behind it, and every misconfiguration in that program is reachable from the network.

> **A word of caution before you start.** This part asks you to install and expose network services. Do it on an isolated lab network, never on a machine that shares a network with anything you do not own: an Apache server left running with default content, an FTP server that accepts anonymous logins, and a cron job that scans a subnet are all things you want on a disposable virtual machine and nowhere else. Take a snapshot before you begin.

### How to read the transcripts

The conventions are the same as in Parts 1 and 2, with one addition that matters here: because this part is about writing files, scripts are shown in full, exactly as you would type them into an editor, followed by a transcript of the script being run. When you see a block labelled as a script, type it into the file named in the surrounding paragraph; when you see a transcript with a prompt, that is you at the shell.

* `root@kali:~#` is a shell running as root, and the trailing `#` is the warning that comes with it. `student@kali:~$` is an ordinary user.
* Everything on a prompt line after the prompt is what was typed; the lines underneath, up to the next prompt, are output.
* Script lines beginning with `#` are comments *inside a script*, with two exceptions you are about to meet: `#!` on the first line, and `#` inside quotes. The distinction between a comment, a prompt, and a comment character inside a string is the thing beginners trip on most in this part.
* Text inside angle brackets, such as `service <name> start`, is a placeholder you replace with a real value.
* Output was captured on Debian-family systems. Versions, PIDs, addresses and timestamps will differ on your machine; the shape of the output is what matters.

### Table of contents

1. [Bash scripting basics](#1-bash-scripting-basics)
2. [Scheduling your tasks](#2-scheduling-your-tasks)
3. [Using services in Linux](#3-using-services-in-linux)
4. [Quick reference cheat sheet](#4-quick-reference-cheat-sheet)
5. [Practice exercises](#5-practice-exercises)

---

## 1. Bash scripting basics

When you have to run the same set of commands repeatedly, or compile the output of several tools into one result, the answer is to write a small program. On Linux the natural language for that is the one you have been speaking all along: the shell. A shell is the interface between you and the operating system, and there are several to choose from — `sh`, `bash`, `zsh`, `dash`, `fish` — each with its own dialect. The one used throughout this guide is **bash**, the Bourne Again SHell, which is the default interactive shell on most Linux distributions and on virtually every security distribution.

Learning bash has a specific advantage over learning a general-purpose language for this job: a bash script can run any system command, utility or application directly, with no bindings, libraries or wrappers in between. The tools you already know how to use on the command line — `nmap`, `grep`, `find`, `curl`, `ssh` — are the same tools you write scripts with. The only thing needed to get started is a text editor; `nano` is used here because Part 1 introduced it, and the choice makes no difference to the script itself.

### 1.1 The shebang

A script is just a text file containing commands, but the operating system needs to be told which program should interpret those commands. That is the purpose of the first line, which begins with a two-character sequence called the **shebang** — `#` followed by `!` — and continues with the path to the interpreter.

Create a new file called `first_script` and put this on the first line:

```bash
#!/bin/bash
```

The shebang is not a comment, even though it starts with the character that usually introduces one. When the kernel is asked to execute a file directly, it reads the first two bytes; if they are `#!`, it takes the rest of the line as the path to an interpreter and runs the file *with that program*. So the line above means "execute everything below with `/bin/bash`", and it is why a script can be run simply by typing its name once it is executable, rather than being prefixed with `bash` every time.

Two variants are worth knowing. `#!/bin/bash` names one specific interpreter, which is correct on any system where bash is installed at that path — true on every mainstream distribution. `#!/usr/bin/env bash` asks the `env` utility to find bash in the current `PATH`, which makes the script portable to systems where bash lives somewhere else (a BSD, a macOS machine, a container with an unusual layout) at the cost of a slightly less predictable interpreter. For scripts that will travel, `env` is usually the better choice; for scripts that stay on one Debian-family machine, `/bin/bash` is fine. What you must not do is write the path with a stray character in it — a trailing slash in `#!/bin/bash/` produces the confusing error `bash: ./first_script: /bin/bash/: bad interpreter: Not a directory`, which is exactly the class of typo that the shebang's unusual position makes easy and that costs beginners an hour.

There is one more detail that matters when your fingers are used to the command line: a plain `#` later in the file *is* an ordinary comment, and everything after it on that line is ignored by the shell. Comments cost nothing to write and are the only documentation a future reader of your script — usually you, six months later — will have.

### 1.2 `echo` — printing a message

The first command in nearly every scripting tutorial is `echo`, which writes its arguments to standard output, followed by a newline. Extend the script so that it looks like this:

```bash
#!/bin/bash
# first_script - the traditional first step in any language
echo "Hello World"
```

Nothing about `echo` is surprising here: it prints the string it was given, in double quotes so that the shell treats the spaces as part of a single argument rather than as separators. Those quotes are not decoration — without them, `echo Hello World` also prints `Hello World`, because `echo` joins its arguments with spaces, but the difference becomes visible the moment you want more than one space, a variable that might be empty, or a literal `*` that should not be expanded into filenames. Quoting is the single most important habit in shell scripting, and starting with it early is the cheapest way to avoid learning it the hard way.

`echo` has two limitations worth knowing about before you rely on it. First, its behaviour with escape sequences such as `\n` varies between implementations and is controlled by the `-e` flag and the shell's `xpg_echo` setting, so when you need precise control of the output, `printf` is the tool to reach for: `printf "user: %s\n" "$USER"` is unambiguous and works identically everywhere. Second, `echo` cannot easily print an unknown number of arguments with separators, which `printf` handles naturally.

### 1.3 Making the script executable and running it

Before the script can be run it needs permission to execute — the same executable bit from Part 1, section 6.4 — and the command that grants it is the same `chmod` you already know.

```bash
root@kali:~# chmod +x first_script
root@kali:~# ls -l first_script
-rwxr-xr-x 1 root root 97 Oct  9 14:02 first_script
root@kali:~# ./first_script
Hello World
```

The permission string changed from `-rw-r--r--` to `-rwxr-xr-x`, and the file name may now be shown in green in your terminal — the usual visual cue for an executable. Then the script ran and printed its one line of output. If you forget the `chmod`, the shell will answer `bash: ./first_script: Permission denied`, which is not an error about the script's contents but about the file's mode.

The `./` in front of the name is the second thing beginners find confusing, and the reason is in Part 1, section 2.8 as well. Current directory is **not** on the `PATH`, deliberately, so that a file called `ls` sitting in a shared folder cannot be run by accident when somebody types `ls`. Writing `./first_script` means "the file called `first_script` in the current directory", which is unambiguous. Typing just `first_script` produces `bash: first_script: command not found` even though the file is right there — the shell searched every directory in `PATH`, and the current one was not among them.

There is a way to run a script without touching its permissions at all, and it is worth knowing for the times you want to test something quickly: hand the file to the interpreter as an argument.

```bash
root@kali:~# chmod -x first_script
root@kali:~# bash first_script
Hello World
```

`bash first_script` works regardless of the executable bit, because `bash` opens and reads the file itself — the kernel is never asked to execute it, so the permission check and the shebang are both irrelevant. This is also why a script with a broken shebang will still run when invoked this way, which makes it a useful diagnostic: if `bash myscript.sh` works and `./myscript.sh` fails, the problem is in the first line or the permissions, not in the script.

### 1.4 Variables and user input

A variable is a named place in memory that holds a value — a string, a number, a path — and it is how a script keeps track of something for longer than one command. Bash variables are created by assigning them, with no `declare`, no types and no spaces around the equals sign, exactly as in Part 2, section 3.3. What makes them useful in a script is that they can be filled with **input from the user**, which turns a fixed sequence of commands into a small tool.

Create a second file, `greeting`, containing this:

```bash
#!/bin/bash
# greeting - ask for a name and greet the person
echo "What is your name?"
read name
echo "Welcome, $name"
```

Save it, make it executable, and run it:

```bash
root@kali:~# chmod +x greeting
root@kali:~# ./greeting
What is your name?
Analyst
Welcome, Analyst
```

The script paused after printing its question, the word `Analyst` was typed and echoed by the terminal (the shell's line editor shows what you type even though the program has not read it yet), and the last line used the value. The mechanics are worth spelling out because two of the three lines are doing something specific. `echo` prints the question, with no trailing special characters — it just happens to end in a question mark. `read name` then *blocks*, waiting for a line of input from standard input, and stores everything up to the newline in the variable `name`. The final `echo` contains `$name`, and the shell substitutes the variable's value before `echo` ever sees the string, exactly like the environment variables in Part 2 — the difference being that this variable is a plain shell variable belonging to the script, not an exported one.

`read` has a few forms that are worth having in your fingers. `read -p "What is your name? " name` prints the prompt itself and avoids a separate `echo`; note the trailing space inside the quotes, which is what keeps the cursor one character away from the question mark. `read -r line` disables the interpretation of backslashes, which is what you almost always want when reading arbitrary text such as a path or a URL. `read -s -p "Password: " pw` reads without echoing the characters, which is the right way to accept a secret. And `read a b c` splits the typed line on whitespace into three variables, with the last one receiving everything that is left.

One habit to build immediately: reference variables with quotes. `"$name"` is correct in every context; `$name` works only while the value contains no spaces and no glob characters. The classic way this bites is a file with a space in its name, where an unquoted variable silently becomes two arguments and the command acts on two paths instead of one.

### 1.5 A script that does something useful: a network scanner

The mechanics are now in place — a shebang, some commands, a variable and an input — so the next step is to assemble them into something you would actually use. The example here is a script that sweeps a network for live hosts, which is a reasonable first genuinely useful tool: it is short enough to understand completely, it exercises every idea in this section, and it is the first thing an assessor runs on an unfamiliar network.

The tool it drives is `nmap`, the standard network mapper. Its job is to discover what is on a network, which ports are open on each host, which services are behind those ports, and often which operating system the host runs. Its general syntax is `nmap <type of scan> <target>`, and for the job of finding live hosts the scan type is a **ping scan**, written `-sn`. Part 2's interface work gives you the reason a ping scan is meaningful: on a local network, a host that answers an ARP request or a ping is up, and the set of hosts that answer is the set of machines you can talk to.

```bash
#!/bin/bash
# sweep - scan a /24 network and print the addresses that answer
read -p "Enter the network prefix (for example 192.168.1): " net
nmap -sn "$net.0/24" | grep "Nmap scan report" | cut -d " " -f 5
```

Save that as `sweep`, make it executable, and run it against the lab network:

```bash
root@kali:~# chmod +x sweep
root@kali:~# ./sweep
Enter the network prefix (for example 192.168.1): 192.168.1
192.168.1.1
192.168.1.9
192.168.1.17
192.168.1.20
```

Four addresses came back, and they are the lab gateway, the Ubuntu target, the attacker machine and one other host. The interesting content is in the pipeline, so it is worth unpacking stage by stage, because this is the pattern that almost every useful script is built from. `nmap -sn "192.168.1.0/24"` performs the scan; writing `/24` after the network address tells nmap to sweep all 256 addresses in that range. The raw output of the command is a short paragraph per live host, most of which is not the address:

```bash
root@kali:~# nmap -sn 192.168.1.0/24
Starting Nmap 7.94 ( https://nmap.org ) at 2026-10-09 14:11 UTC
Nmap scan report for 192.168.1.1
Host is up (0.0021s latency).
Nmap scan report for 192.168.1.9
Host is up (0.00051s latency).
Nmap scan report for 192.168.1.17
Host is up (0.00060s latency).
Nmap scan report for server.lab (192.168.1.20)
Host is up (0.0013s latency).
Nmap done: 256 IP addresses (4 hosts up) scanned in 2.61 seconds
```

The `grep "Nmap scan report"` keeps only the lines that name a host, dropping the latency lines and the summary. The `cut -d " " -f 5` then splits each surviving line on spaces and takes the fifth field: for `Nmap scan report for 192.168.1.1` the fields are `Nmap`, `scan`, `report`, `for`, `192.168.1.1`, so field five is the address. That works — for three of the four hosts. Look again at the fourth: `Nmap scan report for server.lab (192.168.1.20)` is what nmap prints when it manages to reverse-resolve an address to a name, and field five of that line is `server.lab`, not an address. The script's output above happens to be correct because the lab resolver did not know about `server.lab`; on a network with working reverse DNS, the same script would print hostnames mixed in with addresses, and a later script that expected addresses would break.

The robust fix is to stop relying on field positions and extract only what actually looks like an address. Replacing the last two stages with a single pattern match does exactly that:

```bash
#!/bin/bash
# sweep - scan a /24 network and print the addresses that answer
read -p "Enter the network prefix (for example 192.168.1): " net
nmap -sn "$net.0/24" | grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}'
```

The `-o` flag tells `grep` to print only the matched part of the line rather than the whole line, and `-E` enables extended regular expressions. The pattern `([0-9]{1,3}\.){3}[0-9]{1,3}` says "one to three digits, then a dot, repeated three times, then one to three digits" — which is everything that looks like an IPv4 address and nothing else. Running it against the same network now yields a clean list regardless of whether names resolve, and it is a good illustration of a general principle: a script that depends on *where* something appears in an output format breaks when the tool changes its formatting or adds an alternative form, while a script that depends on *what* the thing looks like tends to survive.

### 1.6 Making a script robust

The scripts above work when they are used as intended, which is exactly when they are least tested. A few additions make the difference between a script that is a convenience and a script you can rely on.

```bash
#!/usr/bin/env bash
# sweep - scan a network and print the addresses that answer
# usage: ./sweep 192.168.1
set -euo pipefail

if [ $# -ne 1 ]; then
    echo "usage: $(basename "$0") <network prefix, for example 192.168.1>" >&2
    exit 1
fi

command -v nmap >/dev/null 2>&1 || { echo "nmap is not installed" >&2; exit 1; }

net="$1"
nmap -sn "$net.0/24" | grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}'
```

This version takes its input as an **argument** (`$1`) instead of prompting, which is what makes a script usable from cron, from another script, or in a loop over many networks — and it is worth noting the three special variables that come with arguments: `$1`, `$2` and so on are the positional parameters, `$#` is how many were given, and `$0` is the name of the script itself. The `[ $# -ne 1 ]` test therefore means "if the script was not given exactly one argument", and the message it prints goes to **standard error** (`>&2`) rather than standard output, so that a caller redirecting the output to a file does not capture the error message into the data. `basename "$0"` strips the directory from the script's own name so the usage message reads `sweep` rather than `./sweep`.

The `set -euo pipefail` line at the top is three settings in one and it changes the script's behaviour in ways that prevent a large family of silent failures. `-e` makes the script exit immediately if any command fails, instead of ploughing on with the wreckage of a failed step. `-u` makes referencing an unset variable an error rather than a silent empty substitution — the trap from Part 2, section 3.5. `-o pipefail` makes a pipeline report failure if *any* stage in it failed, not just the last one; without it, `nmap ... | grep ...` can report success because `grep` was happy, even when `nmap` never ran. The `command -v nmap` test checks that a required tool exists before trying to use it, which produces a clear message instead of a `command not found` halfway through.

Two more habits belong in this list because they are what separate script authors from script maintainers. Write comments that say *why*, not *what*: `# -sn is a ping scan; -sP is the same thing under the old name` is valuable, `# run nmap` is not. And test a script with tools that analyse it. `shellcheck` is available in every distribution's repository, understands both bash and POSIX `sh`, and will point out unquoted variables, misused tests, and the other hundred ways shell syntax quietly does something different from what you meant. Running it habitually is the closest thing shell scripting has to a compiler.

---

## 2. Scheduling your tasks

There are times when a task needs to happen on a schedule rather than when you remember it: a nightly backup, a weekly report, a check that runs every hour. Linux has had an answer to this for decades in the **cron** daemon, and it remains the mechanism you will meet on almost every server. This section covers the modern way (a user's crontab), then the older machinery underneath that decides what starts when the machine boots, and finally how both map onto the systemd model that current distributions actually use — because a security practitioner needs to recognise all three.

### 2.1 The cron daemon and the crontab

The `cron` daemon runs in the background, wakes up once a minute, and reads one or more **crontabs** — plain text tables of jobs — to see whether anything is due. If something is, it runs it. Nothing else happens; there is no interactive prompt, no confirmation, and no output unless you arrange for it.

Two kinds of crontab exist and they look slightly different, which is a common source of confusion. The user crontab is edited with `crontab -e` and contains five time fields followed by the command to run, because the user is implied by whose crontab it is. The system crontab, at `/etc/crontab`, has **seven** fields: the five time fields, then the user the job should run as, then the command. The example below is the system file, which is why the sixth field is there:

```
# /etc/crontab: system-wide crontab
# m h dom mon dow user  command
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || run-parts --report /etc/cron.daily
47 6    * * 7   root    run-parts --report /etc/cron.weekly
52 6    1 * *   root    run-parts --report /etc/cron.monthly
```

The first five fields are the schedule, and each one restricts the matching jobs to a set of values:

| Field | Unit | Allowed values |
| --- | --- | --- |
| 1 | Minute | 0–59 |
| 2 | Hour | 0–23 |
| 3 | Day of the month | 1–31 |
| 4 | Month | 1–12 |
| 5 | Day of the week | 0–7 (0 and 7 are both Sunday) |

An asterisk means "every value", so the first job above runs at minute 17 of *every* hour. A number means exactly that value, `25 6` means 06:25, and a `7` in the last field means Sunday. More than one value can be given with a comma (`1,15`), a range with a dash (`1-5` for weekdays), and a step with a slash (`*/10` for every tenth minute, `0-30/5` for every five minutes in the first half hour). The last two fields in `/etc/crontab` are the account to run as and the command line itself, and it is here that the difference from a user crontab shows: in your own crontab you would write the same schedule and then just the command.

### 2.2 Checking and starting the cron service

Before scheduling anything it is worth confirming that the daemon is actually running, because cron is not enabled on every minimal installation and it is trivially easy to add a job to a table that nothing reads. The traditional way to ask is the `service` command, and the modern way is `systemctl`; both are shown because you will meet both.

```bash
root@kali:~# service cron status
○ cron.service - Regular background program processing daemon
     Loaded: loaded (/lib/systemd/system/cron.service; enabled; vendor preset: enabled)
     Active: inactive (dead) since Thu 2026-10-09 14:20:11 UTC; 3min ago
       Docs: man:cron(8)
   Main PID: 918 (code=exited, status=0/SUCCESS)
root@kali:~# systemctl is-active cron
inactive
root@kali:~# service cron start
root@kali:~# systemctl is-active cron
active
```

The status output is a systemd report, which is what `service` wraps on a modern distribution. Reading it from the top: `Loaded` tells you whether the unit file exists and whether the service is *enabled* — that is, whether it will start at boot — while `Active:` tells you whether it is running right now. In this case the two disagree, which is exactly the situation the section warns about: the service is enabled but not currently running, so a job scheduled now would sit in the table until the machine was rebooted. The `systemctl is-active` form is the machine-readable version of the same question and is what you would use in a script, since it prints a single word and returns an exit status; `service cron start` then started the daemon, and the follow-up check confirms `active`.

On a systemd machine the same operations are available directly, and the pairs are worth learning side by side: `systemctl start cron` and `systemctl stop cron` start and stop it now, `systemctl enable cron` makes it start at boot, `systemctl status cron` shows both facts at once, and `systemctl list-timers` shows any systemd timers, which are the modern alternative to cron and are covered at the end of this section. On Debian-family systems the unit is called `cron`; on Red Hat-family systems it is `crond`, which is why a `service crond status` copied from another tutorial may report that no such service exists.

### 2.3 Editing your crontab

With the daemon running, jobs are added by editing your crontab. The `crontab` command is the tool, and the `-e` flag means *edit* — it opens the table in an editor, validates what you save, and installs it.

```bash
root@kali:~# crontab -e
no crontab for root - using an empty one

Select an editor.  To change later, run 'select-editor'.
  1. /bin/nano        <---- easiest
  2. /usr/bin/vim.basic
  3. /usr/bin/vim.tiny
  4. /usr/bin/ed
  5. /bin/ed

Choose 1-5 [1]: 1
crontab: installing new crontab
```

The editor menu appears the first time only, on Debian-family systems, and the choice is remembered in `~/.selected_editor`; `1` selects `nano`, which is the sensible answer for anyone following this guide. In the editor, the file starts with a short comment block explaining the field layout, which is worth reading once. Scroll to the bottom, past the comments, and add the job:

```
# m h  dom mon dow   command
55 23 * * * /root/sweep 192.168.1
```

Save and exit (`Ctrl+O`, Enter, `Ctrl+X` in nano), and crontab reports `installing new crontab`. The schedule reads: at minute 55 of hour 23, every day of every month, whatever the day of the week — in other words, 23:55 every night — run the script `/root/sweep` with the argument `192.168.1`. The `crontab -l` command lists the installed table without opening an editor, and `crontab -r` removes it entirely, which is a command to type carefully because there is no confirmation and no undo.

That job line is worth studying for two details that catch almost everyone. The first is the **absolute path**. `cron` does not run your shell, does not read your startup files, and does not have your `PATH`; it runs the command in a minimal environment where `PATH` is often just `/usr/bin:/bin`. Writing `/root/sweep` rather than `sweep` removes the ambiguity, and any tool the script itself calls should be referenced the same way or added to a `PATH=` line at the top of the crontab. The second detail is the **missing output**: whatever the job prints goes nowhere you will see it. The daemon captures stdout and stderr and mails them to the local user, which on a machine with no mail configured means the text is discarded — so a job that fails on the first night fails silently every night. The defence is to redirect the job's output to a file yourself, with the time and a separator so the log stays readable:

```
55 23 * * * /root/sweep 192.168.1 >> /var/log/sweep.log 2>&1
```

Once jobs are running, their execution is recorded in the system log even when their output is not, which is the first place to look when a schedule seems not to fire:

```bash
root@kali:~# grep CRON /var/log/syslog | tail -5
Oct  9 23:55:01 kali CRON[7123]: (root) CMD (/root/sweep 192.168.1 >> /var/log/sweep.log 2>&1)
Oct  9 23:55:03 kali CRON[7122]: (root) CMD (cd / && run-parts --report /etc/cron.hourly)
```

Those lines answer the two questions that matter: did cron *start* the job, and as which user? If a line appears but the job produced no result, the problem is inside the command — a tool that is missing from the minimal `PATH`, a script that assumed it was being run from its own directory, or a relative path that resolved somewhere unexpected. If no line appears at all, the problem is in the schedule, the crontab, or the daemon — and the checks in section 2.2 are the way to narrow it down. On Ubuntu, `cron` logs to `/var/log/syslog`; on systemd-focused systems you may need `journalctl -u cron`, and on Red Hat-family systems it is `/var/log/cron`.

A few last refinements are worth having. The `%` character is special in a crontab — it terminates the command and everything after it becomes standard input to it — so a literal percent sign must be escaped as `\%`, which matters when you schedule anything involving `date +%Y%m%d`. Cron runs jobs with a minimal environment, so anything that depends on a variable set in `/etc/environment` or `~/.bashrc` will not have it. And if you need a job to run at a time that has already passed today, cron will not run it until tomorrow; that is what the `at` command from Part 2, section 2.9 is for.

Systemd timers are the modern equivalent and are worth knowing by name. Where cron expresses a schedule as five fields, a timer pairs a `.timer` unit with a `.service` unit and can express things cron cannot — "run this 10 minutes after boot", "run this once, three hours from now", "run this an hour after the system was last active". `systemctl list-timers --all` lists what is scheduled on such a system, and the classic entries for `apt-daily` and `fstrim` show up there on any current Ubuntu machine. Timers are not a replacement for cron in every case, but on a distribution that already manages services with systemd they are the more consistent choice, and they log to the journal in the same way as everything else.

### 2.4 Boot-time startup: rc scripts, init and runlevels

Scheduling a job is one form of automation; the other is deciding what runs when the machine starts, and that machinery is older than cron. When a Linux machine is switched on, a large number of processes have to run in order to bring the system to a usable state — mounting file systems, starting the network, launching the daemons that will accept connections. The scripts that perform these steps are known as **rc scripts**, short for *run commands*, and the traditional layout keeps them in `/etc/init.d/`, one file per service, each with a standard set of subcommands (`start`, `stop`, `restart`, `status`).

The daemon responsible for running them was **init**, the first process the kernel starts (process ID 1), which read the scripts and executed them in the order the current **runlevel** dictated. A runlevel is a number that tells the system which set of services should be running, and the traditional set is small:

| Runlevel | Meaning |
| --- | --- |
| 0 | Halt the system |
| 1 | Single-user / minimal mode — for maintenance and recovery |
| 2–5 | Multiuser modes — normal operation, with 5 traditionally meaning "with a graphical login" |
| 6 | Reboot the system |

Zero and six are the two you must not confuse: `init 0` powers the machine off and `init 6` reboots it, so a mistyped digit on a remote server is the difference between an outage and an outage that comes back. Runlevel 1 is the one worth remembering for its usefulness rather than its danger: in single-user mode the machine comes up with a root shell, no network and almost no services, which is why it is the standard way back into a system whose normal boot has broken.

The runlevel a machine starts in is decided by a set of symbolic links, one directory per runlevel — `/etc/rc2.d`, `/etc/rc3.d` and so on — each containing links named `Snn<service>` or `Knn<service>`. The `S` links are started and the `K` links are killed when that runlevel is entered, and the two-digit numbers set the order: `S01networking` starts before `S20apache2`. The whole scheme is a way of declaring, declaratively, which services belong to which operating mode.

Adding a service to that scheme is what `update-rc.d` is for: it creates and removes the appropriate links so that you do not have to manage them by hand.

```bash
root@kali:~# update-rc.d mysql defaults
update-rc.d: warning: start and stop actions are no longer supported; falling back to defaults
Adding system startup for /etc/init.d/mysql ...
   /etc/rc0.d/K01mysql -> ../init.d/mysql
   /etc/rc1.d/K01mysql -> ../init.d/mysql
   /etc/rc6.d/K01mysql -> ../init.d/mysql
   /etc/rc2.d/S01mysql -> ../init.d/mysql
   /etc/rc3.d/S01mysql -> ../init.d/mysql
   /etc/rc4.d/S01mysql -> ../init.d/mysql
   /etc/rc5.d/S01mysql -> ../init.d/mysql
```

The word `defaults` is one of the four actions the command accepts — `defaults`, `remove`, `disable` and `enable` — and it means "use the standard sequence for this service". The output shows exactly what the sections above described: one link per runlevel, `S` links for the multiuser runlevels 2 to 5 so MySQL starts when they are entered, and `K` links for runlevels 0, 1 and 6 so it stops cleanly before the machine halts, drops to single-user mode or reboots. After a reboot the service is running, and the way to confirm that is a tool from Part 2:

```bash
root@kali:~# ps aux | grep -v grep | grep mysql
mysql       1204  0.4  5.1 3310420 208144 ?      Ssl  14:31   0:02 /usr/sbin/mysqld
```

One process, owned by the `mysql` account, with no controlling terminal — a daemon, started automatically, running as a non-privileged user, which is exactly what a well-behaved service looks like. (The `grep -v grep` stage removes the search command's own line from the output, the bycatch explained in Part 2, section 2.3.)

The parts of this that have changed matter more than the parts that have not, so here is the modern picture. On a systemd machine, `/sbin/init` is a link to systemd, and the runlevels are implemented as **targets** with aliases that keep the old numbers working:

| Runlevel | systemd target |
| --- | --- |
| 0 | `poweroff.target` |
| 1 | `rescue.target` |
| 2, 3, 4 | `multi-user.target` |
| 5 | `graphical.target` |
| 6 | `reboot.target` |

`systemctl get-default` reports which target the machine boots into (`multi-user.target` or `graphical.target`, normally), `systemctl set-default graphical.target` changes it, and `systemctl enable <service>` is the modern equivalent of `update-rc.d <service> defaults`, with `disable` to undo it. The `update-rc.d` command above still works on such a system because of a compatibility layer that translates the old interface into unit changes — which is why you will see it in older documentation and why it is worth recognising rather than being confused by. Two smaller tools complete the picture: `runlevel` prints the previous and current runlevel numbers, and `who -r` prints the same information in a wordier form; both are thin wrappers over the systemd state.

For security work, the whole of this subsection matters for one practical reason: **a service that starts at boot is a service that is running when you arrive**. Enumerating what is enabled — `systemctl list-unit-files --type=service --state=enabled`, or the contents of `/etc/rc*.d/` — tells you what will be listening before you have run a single scan, and it is how you spot a service that an administrator disabled by hand but which came back because its links were never removed.

---

## 3. Using services in Linux

A **service** is a program that runs in the background and waits to be used: it holds a socket open, or keeps a queue moving, or watches a file, and it does this whether or not anyone is logged in. Most of what a server does is services. Apache serves web pages, OpenSSH accepts remote shells, FTP transfers files, MySQL answers database queries, and a mail server sits quietly queueing messages. Several come preinstalled, and the rest are one package away.

Learning to manage them — start, stop, restart, inspect, enable — is the last piece of the picture this guide has been building. It is also where the security-relevant surface of a host lives, so the section treats each of the three example services from both directions: how to run it, and what an assessor sees when they find it misconfigured. That second view is not a digression. The most common findings on a real engagement are not exotic exploits; they are a service that was installed for a demo and never removed, an FTP server that still allows anonymous logins, or a web directory whose permissions allow more than the administrator intended.

### 3.1 Managing a service

The traditional interface is the `service` command, and its grammar is small enough to memorise in one line: `service <name> <action>`, where the actions are `start`, `stop`, `restart` and `status`. The same actions exist in systemd as `systemctl <action> <name>`, and on a modern distribution `service` is a wrapper around `systemctl`, so the two are interchangeable for these four verbs. The differences appear for the operations that systemd added — `enable`, `disable`, `mask` — which have no equivalent in the older command.

```bash
root@kali:~# service apache2 start
root@kali:~# service apache2 status
● apache2.service - The Apache HTTP Server
     Loaded: loaded (/lib/systemd/system/apache2.service; enabled; vendor preset: enabled)
     Active: active (running) since Fri 2026-10-09 14:35:02 UTC; 2s ago
       Docs: https://httpd.apache.org/docs/2.4/
   Main PID: 3312 (apache2)
     Status: "Processing requests..."
      Tasks: 55 (limit: 2261)
     Memory: 6.2M
        CPU: 42ms
     CGroup: /system.slice/apache2.service
             ├─3312 /usr/sbin/apache2 -k start
             ├─3313 /usr/sbin/apache2 -k start
             └─3314 /usr/sbin/apache2 -k start
root@kali:~# service apache2 stop
root@kali:~# service apache2 status
● apache2.service - The Apache HTTP Server
     Loaded: loaded (/lib/systemd/system/apache2.service; enabled; vendor preset: enabled)
     Active: inactive (dead) since Fri 2026-10-09 14:35:41 UTC; 3s ago
```

The status output is worth reading properly because it answers four separate questions. `Loaded` and its `enabled` word tell you whether the service will start at boot — so this Apache will come back after a reboot even though it is stopped right now. `Active: active (running)` is the present state, along with the time it has been that way. `Main PID` and the `CGroup` tree show the processes involved, and the tree is instructive: Apache is not one process but a parent and a set of children, which is how it handles several connections at once — the master process binds the port and the children serve requests. `Status: "Processing requests..."` is the service's own self-report, which comes from Apache rather than from systemd.

Some services take a moment to come up, and a status polled immediately after `start` can legitimately read `activating` rather than `active`; `systemctl is-active` returns `activating` as a distinct state for exactly that reason, and a script that tests for the word `active` will misread it. The complementary command `systemctl is-enabled` answers the boot question on its own, printing `enabled`, `disabled`, `static` or `masked`.

`restart` is the fourth verb and the one with the most subtlety, because there are two ways to apply a configuration change and they are not equivalent. A **restart** stops the service and starts it again, which guarantees the new configuration is in effect but drops every connection that was open — acceptable for a lab web server, unacceptable for a database or an SSH daemon somebody is logged into. A **reload** asks the running process to re-read its configuration without stopping: `service apache2 reload`, or `systemctl reload apache2`, or for a daemon that has no reload support at all, the `kill -1` signal from Part 2, section 2.7. The habit worth forming is to check the configuration before either one, because a service that fails to start is an outage while a service that was never restarted is not:

```bash
root@kali:~# apache2ctl configtest
Syntax OK
root@kali:~# service apache2 reload
```

Every service has an equivalent validation step — `sshd -t` for SSH, `nginx -t` for Nginx, `named-checkconf` for BIND — and running it costs a second while skipping it costs the outage.

### 3.2 Apache: deploying a web server

Apache is the most widely deployed web server in the world, and for anyone heading into security work it is worth running once by hand, because the vast majority of engagements involve a web application. Installing it is one package, and it is worth noting that the package is named `apache2` on Debian-family systems and `httpd` on Red Hat-family systems — a difference that breaks a surprising number of copied commands.

```bash
root@kali:~# apt-get install -y apache2
...
Setting up apache2 (2.4.62-1~deb12u2) ...
Enabling module mpm_event.
Enabling module authz_core.
Enabling module authz_host.
Enabling module mime.
Enabling module alias.
...
Created symlink /etc/systemd/system/multi-user.target.wants/apache2.service → /lib/systemd/system/apache2.service.
Processing triggers for man-db (2.11.2-1) ...
root@kali:~# service apache2 start
root@kali:~# ss -tlnp | grep :80
LISTEN 0      511          0.0.0.0:80        0.0.0.0:*    users:(("apache2",pid=3312,fd=4))
LISTEN 0      511             [::]:80           [::]:*    users:(("apache2",pid=3312,fd=4))
```

Two lines of that output deserve attention. The `Created symlink ... multi-user.target.wants/apache2.service` line is systemd recording that the service is now *enabled*: it will start at boot, which is what the packaging system decided is right for a web server, and which is worth knowing before you install one on a machine you did not intend to turn into a web server. The `ss -tlnp` output — the socket-listing tool that Part 2 names as the modern replacement for `netstat`, and that the [SSH guide](04%20-%20SSH%20Penetration%20Testing%20-%20Port%2022.md) demonstrates against a live daemon — confirms the practical consequence: something is now listening on port 80, on every interface (`0.0.0.0` and `[::]`), as the `apache2` process. A port bound to all addresses is reachable by anything that can route to the machine, so this is the exact moment a lab machine becomes visible on the network.

The page that Apache serves by default lives at `/var/www/html/index.html` within the **document root**, the directory whose contents are published. Opening it in an editor shows a small forest of HTML comments and a placeholder page:

```bash
root@kali:~# nano /var/www/html/index.html
```

The file that is there out of the box begins like this, and the comments are worth reading rather than deleting blindly, because they document where the rest of the configuration lives:

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Welcome to Apache2</title>
    <style>...default styling...</style>
  </head>
  <body>
    <div class="main_page">
      <div class="page_header floating_element">
        <img src="/icons/logo.png" alt="Apache logo" class="floating_element" />
        <span class="floating_element">Apache2 Default Page</span>
      </div>
      ...
      <div class="content_section">
        <p>This is the default welcome page used to test the correct operation of
        the Apache2 server after installation on Debian systems. If you can read
        this page, the Apache2 server is installed correctly, and is now working.</p>
      </div>
    </div>
  </body>
</html>
```

Replacing the body with something of your own is the shortest possible demonstration that the server really is serving *your* files:

```html
<!DOCTYPE html>
<html>
  <head><title>Lab Server</title></head>
  <body>
    <h1>It works</h1>
    <p>Served by Apache on 192.168.1.9 at 14:40.</p>
  </body>
</html>
```

With the file saved, the change is visible immediately — Apache reads the file from disk for each request, so no restart is needed for a content edit — and the simplest way to see it is to ask for it from the command line, which produces exactly what a browser would receive:

```bash
root@kali:~# curl -s http://localhost/
<!DOCTYPE html>
<html>
  <head><title>Lab Server</title></head>
  <body>
    <h1>It works</h1>
    <p>Served by Apache on 192.168.1.9 at 14:40.</p>
  </body>
</html>
root@kali:~# curl -s -o /dev/null -w "%{http_code}\n" http://192.168.1.9/
200
```

`curl http://localhost/` sends the request to the loopback interface, which Apache answers because it is bound to every interface, and prints the response body. The second command asks the same question but discards the body and prints only the HTTP status code: `200`. That one-liner is the standard way to test whether a web service is reachable, and it is worth adding to your notes because it is equally useful in a script — a `200` means the page is there, `404` means the server is up but the path is wrong, and a connection failure means nothing is listening at all.

Three details of the layout are worth knowing before moving on, because they come up constantly. The **configuration** lives in `/etc/apache2/`, with `apache2.conf` as the main file and `sites-available/` and `sites-enabled/` holding the per-site virtual host definitions; enabling a site is a matter of `a2ensite` creating a symbolic link, and `a2enmod` does the same job for modules. The **logs** are at `/var/log/apache2/access.log` and `/var/log/apache2/error.log`, the first recording every request with the client's address, the method, the path and the status code, and the second recording the server's own complaints; between them they are the single best record of what has been done to a web server, and the first thing to read after any suspected compromise. And the **permissions** on `/var/www/html` are a classic source of findings: the directory is usually owned by `root` with group `www-data`, and an administrator who "fixes" a broken upload by running `chmod -R 777 /var/www` has created a document root that any local user can rewrite — which converts a low-privileged account on the box into control of the website it serves.

That last observation is the bridge to the security view of this section. When you meet an Apache server on an engagement, the questions worth asking are the same ones that make it useful here: what is in the document root beyond the intended application (backup files, `.git` directories, configuration files with credentials), whether directory listing is enabled and reveals a file tree, which modules are loaded, and what the access and error logs say about who has been there before you. Almost none of those are exploits; all of them are findings.

### 3.3 OpenSSH as a service

The second example service is one you have already used. SSH — the Secure Shell — is what lets you open a terminal on a remote machine, and it is the subject of the companion guide in this repository, [SSH Penetration Testing - Port 22](04%20-%20SSH%20Penetration%20Testing%20-%20Port%2022.md), which covers the service from an offensive and defensive angle in detail. Here the interest is only in what makes it a *service*: it is a daemon that listens on a port, it has a name in the service manager, and it can be started, stopped and inspected like any other. The one thing worth saying in this guide's context is what it replaced.

SSH's ancestor was **telnet**, which performed the same job of giving you a remote terminal and did it in cleartext — every keystroke, including the password, crossing the network in a form anybody on the path could read. SSH encrypts the whole channel and authenticates the server with a host key, which is why it displaced telnet entirely for administration; you will still meet telnet in lab environments as an intentionally vulnerable service, and its presence on a production network is a finding in itself.

Starting the service and connecting are two commands, and the prompt tells you when it worked:

```bash
pentest@ubuntu-lab:~$ sudo systemctl start ssh
pentest@ubuntu-lab:~$ sudo systemctl enable ssh
Synchronizing state of ssh.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install enable ssh
pentest@ubuntu-lab:~$ ss -tlnp | grep :22
LISTEN 0      128          0.0.0.0:22        0.0.0.0:*    users:(("sshd",pid=812,fd=3))
root@kali:~# ssh pentest@192.168.1.9
pentest@192.168.1.9's password: 
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-91-generic x86_64)
Last login: Fri Oct  9 14:44:10 2026 from 192.168.1.9
pentest@ubuntu-lab:~$ 
```

The `enable` output is a small piece of archaeology worth one sentence: systemd reports that it is synchronising state with a SysV init script, which means the unit is being translated through the compatibility layer described in section 2.4 — the same layer that lets `update-rc.d` still work. Then `ss` shows the daemon listening, and the `ssh` command from the attacker machine authenticates, prints the target's login banner and hands over a prompt on the remote host. The syntax is `ssh <username>@<address>`, and everything about what happens next — host keys, passwords against keys, port forwarding — is covered in the companion guide.

### 3.4 Working with FTP

The **File Transfer Protocol** is the third example, and it is included because it is instructive rather than because it is good practice. FTP was the standard way to move files between machines for decades: a client connects to a server on port 21, authenticates, and can then list, upload and download. Its design predates any notion of security — credentials and data both travel in cleartext — and it has largely been replaced by SFTP and by HTTPS-based transfers. It remains in this guide for two reasons: it is still installed on a surprising number of internal networks, and its failure modes (anonymous access, cleartext credentials, and a data channel that most firewalls and monitors handle badly) are the sort of thing you will be asked to look for.

Setting up a server to practise against takes one package, and it should be done on the isolated lab network only. The example server is the Ubuntu target from the earlier parts, at `192.168.1.9`:

```bash
pentest@ubuntu-lab:~$ sudo apt-get install -y vsftpd
...
Setting up vsftpd (3.0.5-0ubuntu1) ...
Created symlink /etc/systemd/system/multi-user.target.wants/vsftpd.service → /lib/systemd/system/vsftpd.service.
pentest@ubuntu-lab:~$ sudo systemctl start vsftpd
pentest@ubuntu-lab:~$ ss -tlnp | grep :21
LISTEN 0      32           0.0.0.0:21        0.0.0.0:*    users:(("vsftpd",pid=4211,fd=3))
```

The server is now listening on port 21, and the client side is a single command. Connecting is where the protocol's age shows most clearly, because everything happens in plain text and in a strict order: server greeting, then a name, then a password.

```bash
root@kali:~# ftp 192.168.1.9
Connected to 192.168.1.9.
220 (vsFTPd 3.0.5)
Name (192.168.1.9:root): anonymous
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||43795|)
150 Here comes the directory listing.
-rw-r--r--    1 0        0            1024 Oct 09 14:02 readme.txt
drwxr-xr-x    2 0        0            4096 Oct 09 14:02 pub
226 Directory send OK.
ftp> cd pub
250 Directory successfully changed.
ftp> get readme.txt
local: readme.txt remote: readme.txt
229 Entering Extended Passive Mode (|||51423|)
150 Opening BINARY mode data connection for readme.txt (1024 bytes).
1024 bytes received in 0.0012 secs (853.3 Kbytes/sec)
226 Transfer complete.
ftp> bye
221 Goodbye.
root@kali:~# ls -l readme.txt
-rw-r--r-- 1 root root 1024 Oct  9 14:02 readme.txt
```

Reading the transcript in order: `220 (vsFTPd 3.0.5)` is the server's greeting, and it hands you the product and version without being asked, which is reconnaissance for free. The `Name` prompt was answered with `anonymous` — the conventional account for a public file server, which requests a password only as a formality (any address is accepted by a polite server, and `anonymous` typed again works). `230 Login successful` means the server accepted it, and everything after that is a conversation in the same numbered-reply style: `ls` lists, `cd` changes directory, `get` downloads, and `bye` closes the session. Each transfer is preceded by a `229 Entering Extended Passive Mode` line, which is the server telling the client to open a *second* connection to an ephemeral port for the data — a design decision that is the reason FTP is so awkward to firewall and to monitor, and the reason passive and active modes exist as a topic.

Two things about this example are worth stating plainly, because the transcript above could be mistaken for a recommendation. The first is that **anonymous FTP is an access-control failure by definition**. A server that lets anyone in, even read-only, publishes whatever is in its reachable directories to anyone who can route to port 21, and the classic finding is that the directory tree above the intended public folder is also reachable. The second is that even an authenticated FTP session sends its password in cleartext, so anyone who can observe the network — a colleague on the same switch, a compromised router, a wireless access point — has the credentials. The secure replacements are worth knowing by name: `sftp`, which speaks the file-transfer protocol *over* SSH and therefore inherits its encryption and its authentication (and is simply the `sftp` subcommand of a normal SSH client); `scp` for simple copies; `lftp` and `curl` for scripted transfers over SFTP, FTPS or HTTPS; and `rsync` for anything involving large trees or repeated synchronisation. If FTP must be kept for a legacy client, the mitigations are to disable anonymous access, force TLS with `ssl_enable=YES` in the configuration, restrict the reachable directory tree with `chroot_local_user=YES`, and firewall port 21 to the hosts that genuinely need it.

The security view of this whole section, then, is one idea applied three times. A service is a listening socket with a program behind it, and the questions that matter are the same for all of them: what is listening (`ss -tlnp`), who started it and with what configuration (`systemctl status`, then the configuration file), does it accept input from the network that it should not (`anonymous` FTP, an Apache directory listing, a password prompt), and will it come back after a reboot whether or not anyone wants it (`systemctl is-enabled`). Enumerating services is the first substantive step of both administering a machine and assessing one, and the command is the same in both cases.

---

## 4. Quick reference cheat sheet

### Writing scripts

| Item | What it does | Example |
| --- | --- | --- |
| `#!/bin/bash` | shebang — declares the interpreter for a script | first line of the file |
| `#!/usr/bin/env bash` | the same, finding bash through `PATH` (portable) | first line of the file |
| `# comment` | a comment anywhere except the shebang | `# usage: sweep <prefix>` |
| `chmod +x script` | makes a script executable | `chmod +x sweep` |
| `./script` | runs a script in the current directory | `./sweep 192.168.1` |
| `bash script` | runs a script without the executable bit | `bash sweep` |
| `echo "text"` | prints text, quoting preserves spaces | `echo "Hello World"` |
| `printf "%s\n" "$x"` | prints with explicit formatting | `printf "user: %s\n" "$USER"` |
| `read name` | reads a line of input into a variable | `read name` |
| `read -p "prompt " name` | reads with a prompt printed first | `read -p "Name: " name` |
| `read -r line` | reads without interpreting backslashes | `read -r path` |
| `read -s -p "Password: " pw` | reads a secret without echoing it | as shown |
| `"$variable"` | expands a variable, safely quoted | `echo "$name"` |
| `$1`, `$2`, `$#` | first argument, second argument, argument count | `if [ $# -ne 1 ]` |
| `$0` | the name the script was invoked with | `basename "$0"` |
| `set -euo pipefail` | fail fast, error on unset variables, fail on any pipeline stage | first lines of the script |
| `command -v tool` | checks whether a required tool exists | `command -v nmap` |
| `>&2` | sends a message to standard error | `echo "usage: ..." >&2` |
| `exit 1` | exits with a failure status | after a usage message |
| `shellcheck script` | analyses a script for common mistakes | `shellcheck sweep` |

### Scheduling

| Command | What it does | Example |
| --- | --- | --- |
| `crontab -e` | edits your crontab (5 time fields + command) | `crontab -e` |
| `crontab -l` | lists the installed crontab | `crontab -l` |
| `crontab -r` | removes the crontab entirely, with no confirmation | `crontab -r` |
| `55 23 * * * /path/cmd` | minute, hour, day, month, weekday, then the command | see section 2.3 |
| `*/10 * * * *` | every ten minutes | steps with `/` |
| `1-5` / `1,15` | a range / a list of values | weekdays |
| `>> /var/log/job.log 2>&1` | captures a job's output, which cron would otherwise mail | as shown |
| `service cron status` | asks whether the cron daemon is running | also `systemctl status cron` |
| `service cron start` | starts the daemon now | `systemctl start cron` |
| `systemctl enable cron` | makes it start at boot | `systemctl is-enabled cron` |
| `grep CRON /var/log/syslog` | shows job execution (with the user and command) | as shown |
| `journalctl -u cron` | the same on a journal-based system | as shown |
| `/etc/crontab` | the system crontab (7 fields: adds the user) | `cat /etc/crontab` |
| `at 23:55` | a one-off job instead of a recurring one (Part 2) | end with `Ctrl+D` |

### Boot, runlevels and services

| Command | What it does | Example |
| --- | --- | --- |
| `update-rc.d <svc> defaults` | adds init-script links so the service starts at boot | `update-rc.d mysql defaults` |
| `update-rc.d <svc> enable\|disable\|remove` | the other three actions | `update-rc.d mysql disable` |
| `/etc/init.d/<svc> start` | runs the legacy init script directly | `/etc/init.d/apache2 status` |
| `runlevel` / `who -r` | prints the current (and previous) runlevel | `runlevel` |
| `systemctl get-default` | the target the machine boots into | `systemctl get-default` |
| `systemctl set-default <target>` | changes the boot target | `systemctl set-default graphical.target` |
| `service <svc> start\|stop\|restart\|status` | the four verbs, traditional interface | `service apache2 status` |
| `systemctl <verb> <svc>` | the same under systemd | `systemctl restart ssh` |
| `systemctl reload <svc>` | re-reads configuration without dropping connections | `systemctl reload apache2` |
| `systemctl enable\|disable <svc>` | chooses whether it starts at boot | `systemctl enable ssh` |
| `systemctl is-active` / `is-enabled` | machine-readable state, for scripts | `systemctl is-active cron` |
| `systemctl list-unit-files --type=service --state=enabled` | everything that starts at boot | as shown |
| `ss -tlnp` | what is listening now, and which process owns it | `ss -tlnp \| grep :80` |
| `apache2ctl configtest` | validates the web server configuration before a reload | as shown |
| `curl -s -o /dev/null -w "%{http_code}\n" <url>` | tests whether a web service answers | `curl ... http://192.168.1.9/` |
| `tail -f /var/log/apache2/access.log` | watches every request a web server receives | as shown |
| `ftp <host>` | connects to an FTP server (`ls`, `cd`, `get`, `put`, `bye`) | `ftp 192.168.1.9` |
| `sftp <user>@<host>` | the secure replacement, over SSH | `sftp pentest@192.168.1.9` |
| `scp <src> <dst>` | simple encrypted copies (Part 1 and the SSH guide) | `scp file.txt user@host:/tmp/` |

---

## 5. Practice exercises

These exercises assume the two machines from the earlier parts — a disposable target and a testing machine on an isolated network — and they build directly on the scripts and services above. Take a snapshot before exercise 3, which schedules jobs and enables services.

1. **First scripts.** Write `first_script` with a shebang and an `echo`, make it executable, and run it both as `./first_script` and as `bash first_script`. Then remove the executable bit and explain which invocation still works and why. Write a second script that asks for a name with `read` and greets the person, and a third that takes the name as an argument instead — explaining in one sentence when each style is appropriate.
2. **A useful script.** Write a script that sweeps a `/24` for live hosts with `nmap -sn` and prints only the addresses. Run it on a network where at least one host has reverse DNS, compare the naive `cut -d " " -f 5` version with the `grep -oE` version, and explain the difference in the output. Then run `shellcheck` on it, fix what it reports, and add `set -euo pipefail` plus a usage message for the wrong number of arguments.
3. **Scheduling and boot.** Confirm whether the cron daemon is running and whether it is enabled, then schedule your scanner to run every night at 23:55 with its output logged to a file. Prove it ran by looking in the syslog for the `CRON` line, and prove the log file has content. Then enable a service of your choice to start at boot, reboot the machine, and confirm it came back — using `systemctl is-enabled`, the `S` links in `/etc/rc*.d/`, and a `ps aux` listing. Finally, explain in writing what runlevel 1 is for and how it is reached on a modern system.
4. **Running a service.** Install Apache, start it, and confirm with `ss -tlnp` that it is listening. Replace the default page with one of your own and verify it with `curl`. Start the service, stop it, and note what changes in `ss` output each time. Then install an FTP server, connect anonymously from the other machine, download a file, and write down three reasons why running that service on a production network would be a finding rather than a convenience.

If you can complete these four exercises without looking anything up, you have worked through the whole of this three-part guide: the shell and the file system, networks and processes, and finally the scripts and services that let a machine do its work without you.

---

## Closing notes

Scripting, scheduling and services are the three mechanisms by which a Linux machine stops being something you operate and starts being something that operates. Everything in this part is really one idea seen from three angles: *the machine can run your commands without you*. A script is that idea applied to a sequence; a cron job is that idea applied to a moment in time; a service is that idea applied to a condition — "whenever someone connects to this port". Understanding all three is what makes the difference between a machine you administer by hand and a machine that administers itself.

Two habits from the earlier parts matter more than ever here, because the material in this part runs unattended. The first is to *give everything a way to be observed*: log a cron job's output, read the access log of the web server you just built, check `systemctl is-active` rather than assuming, and know where each service's error messages go. Something that runs without a terminal produces no visible evidence unless you arrange for it, and the most expensive mistakes in this material are silent ones. The second is to *assume that anything you enable is permanent*: a service enabled at boot comes back after every reboot, a cron job keeps running after you have forgotten writing it, and a script copied to `/usr/local/bin` is a command from then on. The craft of automation is as much about being able to turn things off as it is about turning them on, so keep notes — what you enabled, where, and why.

And one habit specific to this part, which follows from the three services you ran. Anything that listens on a socket has an audience, and the audience is not the one you chose. Anonymous FTP, a default Apache page, a web directory somebody chmod'ed to `777` — these are not clever attacks, and they are what a large proportion of real findings actually look like. Whether you go on to administer systems, assess them, or both, the useful instinct to carry away is the one this part has been building all along: before asking whether something *works*, ask what it *exposes*.
