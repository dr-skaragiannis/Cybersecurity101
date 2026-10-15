# UFW Firewall Administration: An Educational Guide

> **Use only on systems you own or are authorized to administer.** Firewall changes can interrupt SSH access, web traffic, monitoring, and other production services. Keep a recovery console or out-of-band access available before applying changes remotely.

## How to read this guide

Every step is shown the same way: the **command** you type, the **output** it produces, and then a short explanation of what happened and how to read the result. For example:

```bash
id
```

**Output:**

```text
uid=1000(student) gid=1000(student) groups=1000(student),27(sudo)
```

A command box shows exactly what to run; an output box shows what the machine answered. Output is representative — addresses, versions and process IDs differ on your own lab.

The paragraphs that follow each output explain *what the tool does* and *why the step matters*, so you learn the method, not just the syntax.

<details>
<summary><strong>Read more</strong> — the rhythm every section follows</summary>

Each section follows the same rhythm. First you read what the step is for. Then you see the exact command, in its own box. Then the output it produced, in a separate box. Finally a short explanation ties the two together and tells you what to look for next.

Anything longer — extra flags, pitfalls, background theory, alternatives and warnings — is tucked into a collapsible box like this one, so you can read the main path first and open the detail only when you want it.

</details>

## Table of contents

1. [What UFW does](#1-what-ufw-does)
2. [Prerequisites and safety](#2-prerequisites-and-safety)
3. [Install UFW](#3-install-ufw)
4. [Establish a safe baseline](#4-establish-a-safe-baseline)
5. [Allow SSH before enabling](#5-allow-ssh-before-enabling)
6. [Allow application services](#6-allow-application-services)
7. [Allow a specific IP address](#7-allow-a-specific-ip-address)
8. [Allow an entire subnet](#8-allow-an-entire-subnet)
9. [Inspect and delete numbered rules](#9-inspect-and-delete-numbered-rules)
10. [Deny an address or subnet](#10-deny-an-address-or-subnet)
11. [Enable and verify the firewall](#11-enable-and-verify-the-firewall)
12. [Logging and troubleshooting](#12-logging-and-troubleshooting)
13. [Reload, reset, and disable](#13-reload-reset-and-disable)
14. [ICMP and ping considerations](#14-icmp-and-ping-considerations)
15. [Verification checklist](#15-verification-checklist)
16. [Key lessons](#16-key-lessons)

---

## 1. What UFW does

UFW, or Uncomplicated Firewall, is a command-line management layer for Linux firewall rules.

It makes common allow and deny operations easier to read than directly editing low-level packet-filtering rules. UFW can control access by source IP address, subnet, service profile, port, and protocol. It is commonly available on Debian-based distributions such as Ubuntu and Debian.

<details>
<summary><strong>Read more</strong> — where a firewall sits, and what it does not replace</summary>

UFW rules are evaluated together with the host's network services, routing, cloud security groups, and any upstream firewall. A rule that allows SSH in UFW does not help if the cloud firewall blocks the connection.

A UFW rule does not replace application authentication or system hardening. A firewall decides which connections may reach a service; it does not decide who the service trusts once the connection is open.

It also does not describe traffic that belongs to a connection established before the rule was added, so review exposed services separately.

</details>

---

## 2. Prerequisites and safety

You need administrative privileges for every UFW command in this guide.

Connect through an authorized SSH session or a local console, and identify the SSH port before enabling the firewall. If the server uses a nonstandard port, substitute that port in the examples below.

A useful initial check is to confirm who you are and what is listening:

```bash
id
sudo ss -tulpn
```

**Output:**

```text
uid=1000(admin) gid=1000(admin) groups=1000(admin)
Netid State  Local Address:Port  Process
LISTEN 0      128    0.0.0.0:22   users:(('sshd',pid=742,fd=3))
LISTEN 0      128    0.0.0.0:80   users:(('apache2',pid=911,fd=4))
```

The `id` command confirms the current identity and groups. The `ss` command lists listening TCP and UDP sockets.

In this example, SSH listens on port 22 and HTTP listens on port 80. This inventory should guide the firewall rules; do not blindly allow every discovered service.

<details>
<summary><strong>Read more</strong> — reading <code>ss</code> output and avoiding a lockout</summary>

Each line of `ss -tulpn` is one listening socket. `Netid` tells you the protocol family (`tcp`, `udp`, `tcp6`). `Local Address:Port` is where the service accepts connections; `0.0.0.0:22` means every IPv4 interface on port 22, while `127.0.0.1:5432` would mean that service is reachable only from the machine itself. The `Process` column names the program and its PID, which is how you learn which service owns a port.

Before you enable any firewall remotely, keep an existing SSH session open, confirm the SSH port a second way (the daemon configuration, or your provider's console), and make sure a recovery path exists. A permissive rule added in a hurry is recoverable; a lockout on a remote server without console access is not.

</details>

---

## 3. Install UFW

On a Debian-based system, refresh package metadata and install UFW:

```bash
sudo apt update
sudo apt install ufw
```

**Output:**

```text
Reading package lists... Done
Building dependency tree... Done
The following NEW packages will be installed:
  ufw
Setting up ufw ...
```

`apt update` downloads current package metadata. `apt install ufw` installs the firewall management utility and its supporting files.

The exact output varies by distribution and package version. Review package prompts carefully rather than accepting unexpected changes automatically.

Check the initial state:

```bash
sudo ufw status verbose
```

**Output:**

```text
Status: inactive
```

An inactive status means UFW is installed but is not currently enforcing its rules.

This initial disabled state is useful because it allows you to create an access policy before activating enforcement.

<details>
<summary><strong>Read more</strong> — why UFW starts disabled, and what install does not do</summary>

Installing the package places the `ufw` command, its application profiles under `/etc/ufw/applications.d`, and a set of default configuration files on the system. It does not, by itself, change the running packet filter — that happens only when you enable UFW or load rules.

That separation is the whole reason the workflow in this guide exists: build the policy first, then switch enforcement on. Enabling a firewall with no SSH rule is the single most common way to lock yourself out of a remote machine.

</details>

---

## 4. Establish a safe baseline

Before enabling UFW, choose default policies. A common server baseline denies unsolicited incoming connections and permits outgoing connections:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

**Output:**

```text
Default incoming policy changed to 'deny'
Default outgoing policy changed to 'allow'
```

The incoming policy controls new connections initiated toward the server. The outgoing policy controls connections initiated by the server.

These defaults do not replace specific service rules, and they do not necessarily block traffic that belongs to an already established connection.

<details>
<summary><strong>Read more</strong> — why deny-incoming and allow-outgoing is the usual starting point</summary>

A server normally needs to answer requests it already knows how to serve, and it normally needs to reach out for updates, DNS and time synchronisation. Denying incoming by default and allowing outgoing by default matches that shape while still blocking unsolicited connections nobody asked for.

The defaults are the *fallback*: they describe what happens to a connection that no explicit rule matched. Order matters less for defaults than for rules, because defaults are consulted only when no rule applies.

</details>

---

## 5. Allow SSH before enabling

Allow the actual SSH port before activating the firewall:

```bash
sudo ufw allow 22/tcp
```

**Output:**

```text
Rule added
Rule added (v6)
```

The rule permits TCP connections to port 22 for both IPv4 and IPv6 when both address families are configured.

If SSH uses another port, use that port instead:

```bash
sudo ufw allow 7822/tcp
```

Do not add a rule for port 22 merely because it is conventional. Confirm the listening port with `ss`, the SSH daemon configuration, or your hosting provider's console. Keeping an existing SSH session open while testing a new rule helps reduce the risk of lockout.

<details>
<summary><strong>Read more</strong> — IPv4, IPv6 and the two confirmation lines</summary>

The `Rule added` line is the IPv4 rule; `Rule added (v6)` is its IPv6 twin. UFW adds both when the system has both stacks configured, which is why a single command produces two messages.

Keeping an existing session open matters because the session is already *established*: closing a firewall over a live connection usually does not cut it, but no *new* connection will be accepted until a rule allows one. That is why you test a second session before you close the first.

</details>

---

## 6. Allow application services

UFW can use named application profiles:

```bash
sudo ufw app list
```

**Output:**

```text
Available applications:
  Apache
  Apache Full
  OpenSSH
```

The list is generated from installed UFW application profiles. A profile usually maps a friendly name to one or more ports and protocols.

Inspect a profile before using it:

```bash
sudo ufw app info "Apache Full"
```

**Output:**

```text
Profile: Apache Full
Title: Web Server (HTTP,HTTPS)
Ports: 80/tcp,443/tcp
```

The output explains which ports the profile opens. `Apache Full` generally permits both HTTP and HTTPS.

If only encrypted web traffic should be reachable, prefer a profile or explicit rule that allows only port 443.

To allow the profile:

```bash
sudo ufw allow "Apache Full"
```

**Output:**

```text
Rule added
Rule added (v6)
```

Quotation marks are important when a profile name contains spaces. Avoid opening a service that is not installed, needed, monitored, and securely configured.

<details>
<summary><strong>Read more</strong> — what an application profile actually is</summary>

A profile is a small file, usually under `/etc/ufw/applications.d/`, that gives a service a name and a list of ports. `OpenSSH` ships with the package, which is why it appears in the list even on a machine that has never had a web server.

`app info` shows you the ports behind the friendly name. The habit worth building is to run `app info` before `app allow`: the profile is a convenience, not a promise that only the ports you expected will open.

</details>

---

## 7. Allow a specific IP address

To allow all supported traffic from one trusted source address:

```bash
sudo ufw allow from 192.168.1.1
```

**Output:**

```text
Rule added
```

This creates a source-based rule. It is broad because it can allow access to multiple local services unless combined with a destination port or service. A narrower rule is often safer:

```bash
sudo ufw allow from 192.168.1.1 to any port 22 proto tcp
```

This permits SSH only from the specified source IP. The `to any port 22` portion identifies the destination service, and `proto tcp` limits the rule to TCP.

Remove the broad rule when it is no longer needed:

```bash
sudo ufw delete allow from 192.168.1.1
```

**Output:**

```text
Rule deleted
```

The deletion command must match the rule's meaning. If the original rule included a port or protocol, delete the corresponding complete rule or use numbered rules.

<details>
<summary><strong>Read more</strong> — why "from an IP" is broader than it looks</summary>

`allow from 192.168.1.1` says "this host may reach *any* port on me". On a machine that runs only SSH that is harmless; on one that also runs a database or an admin panel it is a larger hole than intended. The moment you add `to any port 22 proto tcp`, the rule is bounded to one service and one protocol.

Because a rule's meaning depends on every part of it, UFW deletes a rule by finding one that matches the same terms. Deleting a *different* wording of the rule you meant can silently remove nothing, or remove something else — which is exactly why the numbered listing in the next sections exists.

</details>

---

## 8. Allow an entire subnet

```bash
sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp
```

**Output:**

```text
Rule added
```

The `/24` network contains addresses in the range represented by `192.168.1.0` through `192.168.1.255`, subject to network conventions. This example permits SSH from that network only.

Use subnet ranges carefully: an overly broad trusted network can expose administrative services to more hosts than intended.

<details>
<summary><strong>Read more</strong> — how a CIDR suffix is calculated</summary>

The number after the slash is the length of the network prefix. `/24` fixes the first 24 bits and leaves 8 bits free, which covers 256 addresses; `/16` leaves 16 bits free and covers 65,536; `/32` fixes all 32 bits and names exactly one host.

The rule of thumb is to make the trusted network as small as the requirement allows. A `/24` for an office subnet is a design decision; a `/16` "because it was easier" quietly trusts thousands of addresses.

</details>

---

## 9. Inspect and delete numbered rules

```bash
sudo ufw status numbered
```

**Output:**

```text
Status: active

     To                         Action      From
[ 1] 22/tcp                     ALLOW IN    192.168.1.1
[ 2] 80/tcp                     ALLOW IN    Anywhere
[ 3] 443/tcp                    ALLOW IN    Anywhere
```

The numbered listing shows rule order, destination, action, direction, and source. Review it before deleting anything.

To remove the second rule:

```bash
sudo ufw delete 2
```

**Output:**

```text
Deleting:
 allow 80/tcp
Proceed with operation (y|n)? y
Rule deleted
```

Rule numbers can change after deletion. Re-run `ufw status numbered` before deleting another rule rather than relying on an old list.

<details>
<summary><strong>Read more</strong> — why numbering beats quoting a rule back</summary>

Deleting by number is precise and shows you the rule text before it acts, which is why it is the safer of the two deletion styles. Deleting by wording depends on getting every term exactly right, and a near-miss can fail quietly.

The cost is that numbers are positional: remove rule 2 and the old rule 3 becomes rule 2. That is why the instruction to re-run `status numbered` before every deletion is not pedantry — it is the rule most people learn the hard way.

</details>

---

## 10. Deny an address or subnet

To block a source address:

```bash
sudo ufw deny from 192.168.1.1
```

**Output:**

```text
Rule added
```

To block a subnet:

```bash
sudo ufw deny from 192.168.1.0/24
```

A deny rule rejects matching traffic according to the firewall's processing behavior. Be careful with broad denies because they can block monitoring systems, administrators, health checks, or legitimate users. Prefer precise rules and document why each block exists.

Remove a deny rule with:

```bash
sudo ufw delete deny from 192.168.1.1
```

**Output:**

```text
Rule deleted
```

<details>
<summary><strong>Read more</strong> — deny versus reject, and why a block is a last resort</summary>

A `deny` rule drops matching traffic. Some firewall front-ends also offer a mode that answers with an explicit refusal instead of silence; the difference is mainly how quickly a scanner learns the port exists, and it is a configuration choice rather than something UFW decides for you.

The reason to be careful is that a deny is easy to write and hard to reason about later. A block aimed at one noisy host is fine; a block for a whole subnet can take an administrator, a monitoring agent or a health check offline, and the symptom appears somewhere else entirely. Record the reason next to every block so the next person does not have to guess.

</details>

---

## 11. Enable and verify the firewall

After adding all required rules, enable UFW:

```bash
sudo ufw enable
```

**Output:**

```text
Command may disrupt existing ssh connections. Proceed with operation (y|n)? y
Firewall is active and enabled on system startup
```

The warning exists because an incorrect policy can terminate or prevent SSH access. Confirm that the SSH allow rule is present before answering `y`.

The second line indicates that UFW is active now and configured to start during boot.

Review the complete configuration:

```bash
sudo ufw status verbose
sudo ufw show added
```

**Output:**

```text
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
80/tcp                     ALLOW IN    Anywhere
443/tcp                    ALLOW IN    Anywhere
22/tcp (v6)                ALLOW IN    Anywhere (v6)
```

The verbose status displays whether UFW is active, logging state, default policies, and configured rules. `show added` displays rules entered through UFW commands.

The IPv6 entries are important when the server has IPv6 connectivity; otherwise an IPv4-only review may give a false sense of coverage.

<details>
<summary><strong>Read more</strong> — reading the verbose header line by line</summary>

`Status: active` means enforcement is on. `Logging: on (low)` is the current logging level. `Default: deny (incoming), allow (outgoing), disabled (routed)` restates the baseline policies from section 4, and the `disabled (routed)` clause tells you UFW is not forwarding traffic between interfaces. `New profiles: skip` says what UFW does when it meets an application profile it has not seen before.

The final decision is always the rule table: if a port appears with `ALLOW IN`, new connections to it are accepted. If nothing matches a new incoming connection, the deny-incoming default decides its fate.

</details>

---

## 12. Logging and troubleshooting

Enable moderate logging when you need visibility into blocked traffic:

```bash
sudo ufw logging medium
sudo ufw status verbose
```

**Output:**

```text
Logging enabled
Status: active
Logging: on (medium)
```

Logging levels affect the amount of firewall event information recorded. More logging can help troubleshooting but may increase log volume.

Review the system's firewall and kernel logs using the logging system used by your distribution, for example:

```bash
sudo journalctl -k --since "10 minutes ago" | grep -i ufw
```

**Output:**

```text
kernel: [UFW BLOCK] IN=eth0 OUT= SRC=203.0.113.50 DST=198.51.100.10 PROTO=TCP DPT=23
```

`UFW BLOCK` indicates that a packet was denied. `SRC` is the source address, `DST` is the destination address, `PROTO` is the protocol, and `DPT` is the destination port.

These events help identify unexpected scans or incorrectly blocked legitimate traffic. Public documentation examples use reserved documentation addresses; real logs will contain your actual network addresses.

<details>
<summary><strong>Read more</strong> — from a single log line to a decision</summary>

A block line is a question, not a verdict. Read the source, destination and port, then decide whether the traffic was hostile (a scan against a port you do not run) or legitimate (a monitoring host you forgot to allow).

Logging levels run from `off` upward; the higher the level, the more detail per event and the faster the log grows. Keep a quiet level in normal operation and raise it only while you are investigating, so a busy log does not hide the events you care about.

</details>

---

## 13. Reload, reset, and disable

After manual changes to supported UFW configuration, reload the rules:

```bash
sudo ufw reload
```

**Output:**

```text
Firewall reloaded
```

Reloading applies the configuration without a full system reboot. If the firewall behaves unexpectedly, inspect the rules and logs before making further changes.

To disable enforcement temporarily:

```bash
sudo ufw disable
```

**Output:**

```text
Firewall stopped and disabled on system startup
```

Disabling removes UFW's active enforcement and also prevents automatic activation at boot. Use it only when you understand the exposure created by the change. A safer alternative is often to correct or delete a specific rule.

To return UFW to a clean state:

```bash
sudo ufw reset
```

**Output:**

```text
Resetting all rules to installed defaults. Proceed with operation (y|n)? y
Backing up 'user.rules' to '/etc/ufw/user.rules.20261008_190000'
```

Reset removes user-defined UFW rules and restores defaults. It may also remove the SSH exception that keeps remote access possible, so use a console or ensure you can immediately recreate the required rules. Backups and exact output vary by distribution.

<details>
<summary><strong>Read more</strong> — reload, disable and reset are not interchangeable</summary>

`reload` keeps the policy and re-applies it, which is what you want after editing a supported file. `disable` stops enforcement but leaves the rules saved, so the machine is exposed until you enable UFW again. `reset` throws the rule set away and returns to installation defaults, and because the SSH exception is part of "user rules", it is the one of the three most likely to lock you out.

The common thread is that all three change what is enforced, so all three belong to a moment when you have console access or a recovery plan.

</details>

---

## 14. ICMP and ping considerations

Ping uses ICMP, which is not TCP or UDP. Blocking ping does not make a server invisible and can interfere with monitoring, path MTU discovery, diagnostics, and IPv6 operation.

In most environments, it is better to permit necessary ICMP types and rely on authentication, patching, least privilege, and monitoring for security.

If a documented policy requires changing pre-routing or pre-input ICMP behavior, back up the configuration first:

```bash
sudo cp -a /etc/ufw/before.rules /etc/ufw/before.rules.backup
sudoedit /etc/ufw/before.rules
sudo ufw reload
```

**Output:**

```text
Firewall reloaded
```

The backup allows recovery from a syntax or policy mistake. `sudoedit` opens the file through a safer editing workflow. After editing, `ufw reload` asks UFW to load the new configuration.

Test both intended and unintended traffic afterward, and consider IPv6 configuration separately because IPv4 ICMP rules do not automatically describe IPv6 behavior.

<details>
<summary><strong>Read more</strong> — what <code>before.rules</code> is and why it is edited last</summary>

`/etc/ufw/before.rules` holds the raw rules UFW inserts ahead of its user rules — the ICMP handling, connection tracking and other low-level plumbing live there. Because it is closer to the packet filter than anything you type with `ufw`, a syntax mistake can stop `ufw` from loading at all.

That is why the backup-with-`cp -a` step comes first, why the edit runs through `sudoedit` rather than a plain root editor, and why the reload is a separate, deliberate step you can watch. Treat this file as the exception you touch deliberately, not part of everyday rule management.

</details>

---

## 15. Verification checklist

After configuration, validate from an authorized management host and from the server itself:

```bash
sudo ufw status numbered
sudo ss -tulpn
```

Confirm that:

- The SSH port is allowed from the required administrative source.
- Public web ports are open only when the service is intended to be public.
- Unused services are not exposed.
- IPv4 and IPv6 rules match the deployment design.
- Default incoming and outgoing policies are documented.
- Logging is enabled at an appropriate level.
- A rollback path or console access exists.

<details>
<summary><strong>Read more</strong> — how to actually test, not just read, the policy</summary>

Reading the rule table confirms what you *intended*. A test confirms what the machine *does*. Connect to the tool from a host that should be allowed and from one that should not, and confirm the two answers differ in the expected direction.

Combine the two commands above for a reason: the first is your policy, the second is what is genuinely listening. A listening port with no matching rule is either a service you forgot to publish or one you forgot to close, and the only way to tell them apart is to check the design.

</details>

---

## 16. Key lessons

A firewall policy should be built from an inventory of services and trusted sources, not from copied commands alone.

Start with a restrictive default, allow only necessary traffic, inspect the resulting rules, and test connectivity before closing the administrative session.

Explain every command's output in the change record so another administrator can understand what was enabled, blocked, or removed.

<details>
<summary><strong>Read more</strong> — the habits worth taking away</summary>

Build from the inventory outward: know what listens before you decide what is allowed. Let the default deny do the heavy lifting and add rules only where a service genuinely needs to be reachable.

Then verify, twice: once on paper, by reading `ufw status numbered`, and once in practice, by testing a connection that should succeed and one that should fail. Document the result, because a firewall change nobody recorded is a firewall change nobody can safely undo.

</details>
