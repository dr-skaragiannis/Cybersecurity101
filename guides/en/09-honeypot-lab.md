# Operating a Low-Interaction Honeypot in an Isolated Lab

> **Authorized defensive use only.** A honeypot is a decoy service intended to observe unsolicited access attempts. Deploy it only in a controlled lab or in an organization with written approval, documented monitoring, data-retention rules, and strict network isolation. Do not retaliate against source addresses or use a honeypot as a platform for scanning or attacking others.

## How to read this guide

> **Concept.** This guide is written as a transcript, not a summary: every claim is anchored to a command and the output it produced, so the reader can reproduce the observation instead of trusting a description. That evidence-first style is the same discipline a defender applies to logs and forensic artefacts, where a finding is only as good as the record behind it. Each step below therefore shows the command, the machine's answer, and a short reading of what that answer does — and does not — prove.

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

## Purpose and limitations

> **Concept.** Honeypots are classified first by interaction: a low-interaction decoy emulates just enough of a service to log a contact, while a high-interaction system exposes a real operating system for the adversary to use. They are classified again by purpose — production honeypots feed an organisation's detection pipeline, research honeypots study attacker behaviour — and by both axes the design goal is the same: attractive enough to draw automated probes, worthless enough to be contained if compromised. Deception infrastructure is catalogued defensively by MITRE D3FEND (Decoy Network Resource) and MITRE Engage, while the probing a decoy observes maps to ATT&CK T1595 (Active Scanning) and T1046 (Network Service Discovery). A low-interaction decoy is a low-noise sensor for early detection and indicator collection, not a replacement for an IDS, a firewall, authentication or patching.

A low-interaction honeypot listens on a selected network port, records connection attempts, and may return a simple banner or warning message. It is useful for learning how internet scans and unauthorized probes appear in logs.

It is not a full simulation of SSH, HTTP, FTP, or another protocol. Do not mistake it for a replacement for production intrusion detection, patching, authentication, or a firewall.

<details>
<summary><strong>Read more</strong> — the safe design and the containment goal</summary>

A safe design places the decoy on a dedicated virtual machine or isolated VLAN. Permit only the inbound ports you intentionally monitor, restrict outbound traffic to essential management destinations, and send logs to a separate collector if possible.

The key security goal is **containment**: a compromised decoy must not provide a path to production systems or the wider Internet.

</details>

## Reference architecture

> **Concept.** The diagram is a trust-boundary sketch: the decoy lives in its own segment — a VLAN, a DMZ, or a dedicated virtual switch — so that a compromise is confined to it, and the evidence lives on a third host that the decoy cannot write back to. Isolation, egress restriction and off-host logging are the controls that decide whether a deployment is safe, far more than the realism of the emulated service. The organising principle is containment: no path from the decoy to production systems, and no path from the decoy to the wider Internet.

The diagram below shows the intended shape of the lab: one isolated segment, one decoy, and one separate place to read the logs.

```text
                  Authorized test client
                           |
                           | TCP connection to selected port
                           v
+------------------+   firewall allowlist    +------------------------+
| Isolated lab     | ----------------------> | Honeypot virtual       |
| network / VLAN   |                         | machine                |
| 192.168.56.0/24  |                         | 192.168.56.20          |
+------------------+                         |                        |
                                             | Low-interaction decoy  |
                                             | log file / collector   |
                                             +-----------+------------+
                                                         |
                                                         | restricted logs only
                                                         v
                                             +------------------------+
                                             | Log analysis workstation|
                                             +------------------------+
```

The test client should be a machine you control. The firewall should block the honeypot from reaching production networks and should deny unnecessary outbound connections.

<details>
<summary><strong>Read more</strong> — why the evidence lives on a third machine</summary>

A separate log-analysis host reduces the chance that evidence is altered if the decoy is compromised. Only restricted logs travel there, so an attacker who owns the decoy still cannot touch the records of what they did.

</details>

## Prepare the laboratory host

> **Concept.** Every defensible change begins from a known state: which distribution, which account, and which services already listen. This is the asset-inventory step of a security baseline, and it prevents two routine lab mistakes — installing packages with the wrong package manager, and starting a decoy on a port a real service already owns. Reading the baseline first is also what lets you prove later that you left nothing listening behind.

Before installing anything, establish three facts about the host: which operating system it runs, which account you hold, and which services are already listening.

```bash
cat /etc/os-release
```

**Output:**

```text
PRETTY_NAME="Kali GNU/Linux Rolling"
```

`cat /etc/os-release` prints the distribution identity, which tells you that this host uses Debian-style package management.

```bash
id
```

**Output:**

```text
uid=1000(analyst) gid=1000(analyst) groups=1000(analyst),27(sudo)
```

`id` confirms the local user and whether they hold administrative privileges (here `27(sudo)`).

```bash
ip -brief address
```

**Output:**

```text
eth0             UP             192.168.56.20/24
```

`ip -brief address` shows the interface and the assigned lab address (`192.168.56.20/24`), which is the address the decoy will listen on.

```bash
sudo ss -tulpn
```

**Output:**

```text
Netid State  Local Address:Port  Process
LISTEN 0      128    127.0.0.1:631  users:(('cupsd',pid=621,fd=6))
```

`ss -tulpn` inventories services already listening on TCP or UDP ports.

<details>
<summary><strong>Read more</strong> — what each line tells you, and the port conflict to avoid</summary>

* `cat /etc/os-release` ensures that you use the correct package-management commands for the distribution.
* `id` confirms the local user and whether they hold administrative privileges (here `27(sudo)`).
* `ip -brief address` shows the interface and the assigned lab address (`192.168.56.20/24`).
* `ss -tulpn` inventories services already listening on TCP or UDP ports.

The last line matters most for the next step: a real service is already bound to `127.0.0.1:631`. Do not start a decoy on a port already used by a real service.

</details>

Update the underlying operating system before adding lab software:

```bash
sudo apt update
```

**Output:**

```text
Hit:1 http://http.kali.org/kali kali-rolling InRelease
Reading package lists... Done
Building dependency tree... Done
```

`apt update` retrieves fresh package metadata from the repositories. It changes nothing on disk; it only refreshes which versions are available.

```bash
sudo apt upgrade -y
```

**Output:**

```text
0 upgraded, 0 newly installed, 0 to remove
```

`apt upgrade -y` applies the available updates and answers yes to each confirmation automatically. The final line indicates that no upgrades were waiting.

<details>
<summary><strong>Read more</strong> — keep the host patched even when the decoy is simple</summary>

Keep the operating system current even when the decoy service is intentionally simplistic. The *service* is meant to be trivial; the *host* should not be intentionally vulnerable.

</details>

## Install and inspect the tool

> **Concept.** Installing a network service from source is a supply-chain decision: you are trusting code to open a socket on your behalf, so the discipline is to clone a reviewed copy, read it before executing it, and run it under the least privilege that works. The concrete tool used here is PentBox, a small, menu-driven Ruby security suite whose honeypot module is deliberately simple. Because the tool is interpreted, the runtime (`ruby`) is part of the trust boundary too, and it should be as current as the rest of the host.

A Ruby-based low-interaction honeypot tool may be distributed as source code. Clone only a reviewed, authorized training copy into a dedicated directory.

```bash
sudo apt install -y git ruby
```

**Output:**

```text
Setting up git ...
Setting up ruby ...
```

`apt install` brings in `git`, which retrieves version-controlled source code, and `ruby`, which provides the interpreter the tool needs.

```bash
mkdir -p ~/honeypot-lab
```

`mkdir -p` creates the dedicated project directory and succeeds silently if it already exists.

```bash
cd ~/honeypot-lab
```

`cd` moves the shell into the project directory. It prints nothing, so the clone lands in the right place.

```bash
git clone https://example.invalid/training/pentbox.git pentbox
```

**Output:**

```text
Cloning into 'pentbox'...
done.
```

`git clone` copies the reviewed repository into a directory named `pentbox`. The `example.invalid` host is a placeholder that will not resolve.

```bash
cd pentbox
```

`cd` enters the freshly cloned directory so the next commands run against the tool's own files.

```bash
ruby --version
```

**Output:**

```text
ruby 3.1.xp0 (202x-xx-xx revision ...) [x86_64-linux-gnu]
```

`ruby --version` confirms that the runtime is installed and available from the shell.

<details>
<summary><strong>Read more</strong> — the placeholder repository and the review rule</summary>

The example uses a deliberately non-routable placeholder address (`example.invalid`). Substitute only a repository that your organization has reviewed and approved.

Read the source code, verify its origin, and avoid executing unknown scripts as root.

</details>

The command chain above uses a non-routable placeholder to describe the shape of the step. This guide's concrete example is PentBox, a small, menu-driven Ruby security suite whose honeypot module is deliberately simple; its canonical repository is cloned and entered as follows.

```bash
git clone https://github.com/technicaldada/pentbox.git
```

**Output:**

```text
Cloning into 'pentbox'...
remote: Enumerating objects: 42, done.
remote: Counting objects: 100% (42/42), done.
remote: Compressing objects: 100% (38/38), done.
Receiving objects: 100% (42/42), 268.91 KiB | 2.10 MiB/s, done.
Resolving deltas: 100% (9/9), done.
```

`git clone` copies the PentBox repository into a local directory named `pentbox`; the `remote:` and `Receiving objects` lines are Git reporting the transfer, and the clone is finished when it prints `done`.

PentBox keeps its executable inside a versioned subdirectory, so move into it before launching anything.

```bash
cd pentbox/pentbox-1.8
```

`cd` enters the directory that holds `pentbox.rb`; it prints nothing, so silence here means it succeeded.

Inspect the files and the program's help before executing anything:

```bash
find . -maxdepth 2 -type f | sort | head -30
```

**Output:**

```text
./LICENSE
./README.md
./pentbox.rb
./other/log_honeypot.txt
```

The `find` command gives a compact inventory of the repository, and `sort | head -30` orders it and keeps the first thirty entries.

```bash
sed -n '1,80p' pentbox.rb
```

**Output:**

```text
#!/usr/bin/env ruby
# Training copy — reviewed before use.
require 'socket'
require 'optparse'
...
```

`sed -n '1,80p'` prints only the first eighty lines of the script so you can review it before running anything.

```bash
ruby pentbox.rb --help
```

**Output:**

```text
Usage: pentbox.rb [options]
```

Running the script with `--help` shows its intended options. The help output may differ between versions.

<details>
<summary><strong>Read more</strong> — if the tool does not support <code>--help</code></summary>

If a script does not support `--help`, stop and consult its local documentation rather than guessing its behaviour. Reading the first eighty lines before running anything is the whole discipline of this step.

</details>

## Configure a minimal decoy

> **Concept.** A decoy is a set of deliberate design choices: a high, unused port so it cannot collide with a real service, a banner that tells a legitimate operator what the port is, and a log destination you can protect. PentBox adds one more choice — fast auto-configuration versus manual configuration — where the manual path is what lets you set a specific port, a specific false message and a specific log file. The deception is deliberately shallow: the point is to look worth a scanner's attention, not to impersonate a real organisation.

Choose a high, unused TCP port for initial testing, such as `8088`, rather than impersonating a privileged or production service. Confirm that it is unused:

```bash
sudo ss -ltnp '( sport = :8088 )'
```

**Output:**

```text
State  Recv-Q Send-Q Local Address:Port Peer Address:Port Process
```

The header with no rows means that no TCP process is listening on port 8088.

<details>
<summary><strong>Read more</strong> — what to do if a row appears</summary>

If a row appears, select another port — or stop the conflicting lab service, but only if you are authorized to do so.

</details>

A typical menu-based workflow for a low-interaction tool looks like this:

```text
1) Network tools
2) Honeypot
3) Manual configuration

Listening port: 8088
Banner: Training decoy — access is monitored.
Log path: ./other/log_honeypot.txt
```

Use a clearly documented banner and a dedicated log path. In an internal lab, a banner helps distinguish the decoy from a production service.

<details>
<summary><strong>Read more</strong> — the menu is representative, and the banner is not a costume</summary>

The exact menu names and choices depend on the installed version.

Do not claim to be a real bank, government service, or another organization. The banner exists so that a legitimate operator recognizes the decoy for what it is.

</details>

PentBox is menu-driven, so the configuration is a sequence of numbered choices rather than command-line flags. Launch it from the directory that holds `pentbox.rb`.

```bash
ruby pentbox.rb
```

**Output:**

```text
╔══════════════════════════════════════════════╗
║                 PentBox 1.8                  ║
╚══════════════════════════════════════════════╝

 1 - Cryptography tools
 2 - Network tools
 3 - Web tools
 4 - System tools
 5 - Extra tools
 0 - Exit

Select an option:
```

The main menu groups the suite's tools, and the honeypot lives under **Network tools**, so type `2` at the prompt.

```text
Network tools
 1 - SSH reverse connection
 2 - Mail spoofing
 3 - Honeypot
 0 - Back

Select an option:
```

Type `3` to open the honeypot menu, which offers an automatic and a manual path.

```text
Honeypot
 1 - Fast Auto Configuration
 2 - Manual Configuration
 0 - Back

Select an option:
```

**Fast Auto Configuration** (option `1`) asks only for a listening port, while **Manual Configuration** (option `2`) additionally lets you set the false message, choose whether to save a log and enable a beep on intrusion. Choose `1` and give the high port selected earlier.

```text
Insert the port: 8088
```

Once the port is entered, PentBox confirms that the decoy is listening, with the activation time.

```text
HONEYPOT ACTIVATED ON PORT 8088 (2026-10-08 20:25:14 +0300)
```

Every contact then produces a console line naming the source address and port, alongside an appended record in the log file.

```text
INTRUSION ATTEMPT DETECTED! from 192.168.56.30:49152 (2026-10-08 20:26:03 +0300)
```

The activation line confirms that the sensor is live; the intrusion line is the event itself. "Intrusion" is PentBox's wording for any connection and is not a judgement about intent.

After starting the listener, verify the port locally:

```bash
sudo ss -ltnp '( sport = :8088 )'
```

**Output:**

```text
State  Recv-Q Send-Q Local Address:Port Peer Address:Port Process
LISTEN 0      128    0.0.0.0:8088  0.0.0.0:*     users:(('ruby',pid=1842,fd=7))
```

`LISTEN` means the process is waiting for new TCP connections, and `0.0.0.0:8088` means it accepts IPv4 connections on all network interfaces. The PID and process name help identify the listener for monitoring and controlled shutdown.

<details>
<summary><strong>Read more</strong> — binding to loopback instead of every interface</summary>

For a strictly local test, prefer binding to `127.0.0.1` if the tool supports it. Listening on `0.0.0.0` exposes the decoy on every interface, which is exactly what the firewall rules in the next section are there to contain.

</details>

## Firewall containment

> **Concept.** Containment is expressed as firewall policy: default-deny in both directions, then permit the single inbound port you monitor and the single outbound path your logs require. The outbound rule is the one that stops a decoy from becoming a pivot, which is the failure mode that turns a sensor into a liability. Rule order matters as much as rule content, because the first matching rule decides the packet's fate.

A host firewall should allow only intended lab access and should restrict unnecessary outbound activity. The policy below is a conceptual UFW example for an isolated subnet.

```bash
sudo ufw default deny incoming
```

**Output:**

```text
Default incoming policy changed to 'deny'
```

`ufw default deny incoming` sets the base policy so any unsolicited inbound connection is rejected unless a later rule permits it.

```bash
sudo ufw default deny outgoing
```

**Output:**

```text
Default outgoing policy changed to 'deny'
```

The matching outbound default denies traffic leaving the host, which is what keeps a compromised decoy from reaching the network.

```bash
sudo ufw allow from 192.168.56.0/24 to any port 22 proto tcp
```

**Output:**

```text
Rule added
```

This rule permits SSH management only from the lab subnet.

```bash
sudo ufw allow from 192.168.56.0/24 to any port 8088 proto tcp
```

**Output:**

```text
Rule added
```

This rule permits access to the decoy only from that same subnet.

```bash
sudo ufw allow out to 192.168.56.10 port 514 proto udp
```

**Output:**

```text
Rule added
```

This rule permits UDP syslog forwarding only to the designated log collector.

```bash
sudo ufw enable
```

**Output:**

```text
Firewall is active and enabled on system startup
```

`ufw enable` activates the ruleset and makes it persist across reboots.

```bash
sudo ufw status numbered
```

**Output:**

```text
Status: active

     To                         Action      From
[ 1] 22/tcp                     ALLOW IN    192.168.56.0/24
[ 2] 8088/tcp                   ALLOW IN    192.168.56.0/24
[ 3] 514/udp                    ALLOW OUT   192.168.56.10
```

`status numbered` lists the active rules with an index, so you can see exactly what is permitted and in which order.

<details>
<summary><strong>Read more</strong> — rule by rule, and the temporary exceptions</summary>

* Rule 1 permits SSH management only from the lab subnet.
* Rule 2 permits access to the decoy only from that same subnet.
* Rule 3 permits UDP syslog forwarding only to a designated log collector.

Adapt the policy to the actual management path and test from a console before disconnecting SSH. If the host needs DNS, NTP, package updates, or another management service, add narrowly scoped temporary rules, complete the task, then remove them.

</details>

## Test from an authorized client

> **Concept.** A sensor is only trustworthy once you have generated a known event and watched the sensor record it, so testing is a validation exercise rather than an attack. It is performed from a client you control, with the least intrusive connection that still proves reachability, firewall allowance and logging. A port that answers is not proof that a real protocol is implemented, and a successful test is not evidence of intent.

From a separate client inside the isolated subnet, test only the listener that you created:

```bash
nc -vz 192.168.56.20 8088
```

**Output:**

```text
Connection to 192.168.56.20 8088 port [tcp/*] succeeded!
```

Netcat attempts a TCP connection. The success message confirms network reachability, firewall allowance, and a listener on the target port.

<details>
<summary><strong>Read more</strong> — what a successful <code>nc</code> does and does not prove</summary>

It does not demonstrate that a full application protocol is implemented. A port that answers is not a service that behaves like the real thing.

Do not send exploit payloads; a connection test is sufficient to validate logging and containment.

</details>

To observe a text banner, if the lab tool returns one:

```bash
nc 192.168.56.20 8088
```

**Output:**

```text
Training decoy — access is monitored.
```

This connects interactively and displays the decoy's banner. End the session with `Ctrl+C`.

<details>
<summary><strong>Read more</strong> — why the banner matters to a defender</summary>

The banner is the decoy's only outward-facing claim, so it should be accurate and unthreatening. It tells a casual visitor what the port is, and it tells your own team that they are looking at the lab and not at a real system.

</details>

## Monitor and interpret logs

> **Concept.** The value of a honeypot is early detection: any contact on a port that advertises nothing is, by construction, unexpected, which makes the decoy a low-noise tripwire. Each entry is a candidate indicator of compromise — a source address, a port, a timestamp — and forwarding those records to a central collector (for example a syslog stream per RFC 5424) is what converts a local file into a detection signal a SIEM can act on. Interpretation stays cautious: an event is an observation to be correlated, not a verdict.

### Follow the log live

Watch the decoy log in real time:

```bash
tail -F ./other/log_honeypot.txt
```

**Output:**

```text
2026-10-08T20:25:14+03:00 connection source=192.168.56.30:49152 destination_port=8088
```

`tail -F` continues following the file even if it is rotated or recreated. The timestamp supports incident reconstruction, and the source IP and ephemeral source port identify the connecting test client.

The decoy also prints its own line on the console for each contact:

```text
CONNECTION ATTEMPT DETECTED from 192.168.56.30:49152
```

This indicates that a remote endpoint established or attempted a connection.

<details>
<summary><strong>Read more</strong> — an event is not proof of intent</summary>

It is an event, not proof of malicious intent: health checks, scanners, misconfigurations, and authorized tests can all generate similar entries. Preserve surrounding context before classifying an event.

</details>

### Review recent entries

Review recent entries without continuously following the file:

```bash
tail -n 50 ./other/log_honeypot.txt
```

**Output:**

```text
2026-10-08T20:25:14+03:00 connection source=192.168.56.30:49152 destination_port=8088
2026-10-08T20:26:03+03:00 connection source=192.168.56.31:49411 destination_port=8088
```

The two records show separate connection attempts from different hosts and source ports.

<details>
<summary><strong>Read more</strong> — why a source port is not an identity</summary>

A source port normally changes between outbound connections and is not a reliable user or device identity by itself. Correlate it with firewall logs, DHCP leases, asset inventory, and approved test records.

</details>

PentBox appends one record per line, so `log_honeypot.txt` is a simple, append-only event stream that ordinary text tools can read. The representative format is `timestamp event source=<ip>:<port> destination_port=<port>`, and the fields are positional — which is exactly why the summary command in the next section can split them.

```text
2026-10-08T20:25:14+03:00 connection source=192.168.56.30:49152 destination_port=8088
```

The first field is an ISO 8601 timestamp with the local UTC offset, the second is the event type, and the remaining `key=value` pairs name the source and the monitored port.

<details>
<summary><strong>Read more</strong> — the record read field by field</summary>

| Field | Example | Meaning |
| --- | --- | --- |
| Timestamp | `2026-10-08T20:25:14+03:00` | ISO 8601 local time with offset; the anchor for correlating with other logs |
| Event | `connection` | What the decoy recorded; the console also prints `INTRUSION ATTEMPT DETECTED!` |
| `source` | `192.168.56.30:49152` | Source address and ephemeral source port of the contact |
| `destination_port` | `8088` | The monitored port the contact reached |

PentBox writes this file under `pentbox/pentbox-1.8/other/log_honeypot.txt` by default, and the manual configuration path lets you keep the default or choose another file. Because it is line-oriented plain text, it can be parsed with `awk`, forwarded by a log agent, or ingested by a SIEM without a custom decoder.

</details>

## Basic log summary

> **Concept.** Individual lines describe events; summaries describe behaviour. Counting sources, ports and times converts a raw feed into the aggregates that triage actually consumes, and those aggregates are how a single decoy becomes a comparative view of who is knocking. An aggregate is a starting point for correlation with firewall, DHCP and asset records, never a conclusion on its own.

Extract source addresses in a lab log format and count them:

```bash
awk -F'source=' '/source=/{split($2,a,":"); print a[1]}' ./other/log_honeypot.txt | sort | uniq -c | sort -nr
```

**Output:**

```text
      8 192.168.56.30
      2 192.168.56.31
```

`awk` selects records containing `source=` and isolates the source address before the colon. `sort` places identical addresses together, `uniq -c` counts them, and the final `sort -nr` orders the counts numerically from highest to lowest.

<details>
<summary><strong>Read more</strong> — what the count does and does not say</summary>

This simple count helps identify frequent contact, but it does not establish malicious intent or attribution. Eight connections from one address may be a scanner, a monitoring probe, or a colleague running an approved test.

</details>

## Preserve evidence safely

> **Concept.** Digital evidence is only useful if you can show it has not changed, which is why an acquired copy is hashed, and its timestamps and permissions are preserved. ISO/IEC 27037 gives the governing principles — identification, collection, acquisition and preservation of digital evidence — and the chain of custody is the record that ties each hash to each handover. A hash proves integrity relative to a recorded value; it does not prove who generated the event.

Rotate a copy of the log before analysis and protect its permissions:

```bash
install -d -m 0750 ~/honeypot-evidence
```

`install -d -m 0750` creates the evidence directory with mode `0750`, which limits access to the owner and group. It prints nothing on success.

```bash
cp --preserve=mode,timestamps ./other/log_honeypot.txt ~/honeypot-evidence/log_$(date +%F_%H%M%S).txt
```

`cp` copies the log to a timestamped name, and `--preserve` retains its mode and timestamps. It prints nothing when the copy succeeds.

```bash
chmod 0640 ~/honeypot-evidence/*.txt
```

`chmod 0640` removes access for `others` while keeping the owner and group able to read. Silence means the modes were applied.

```bash
sha256sum ~/honeypot-evidence/*.txt
```

**Output:**

```text
c9a4...e17b  /home/analyst/honeypot-evidence/log_2026-10-08_202700.txt
```

The SHA-256 digest records a fingerprint of the evidence file: if the file changes, its digest will normally change.

<details>
<summary><strong>Read more</strong> — integrity is not authorship</summary>

A hash proves integrity relative to a recorded value; it does not prove who generated the event. Copy the file before you analyse it, so that whatever you do next cannot alter the original.

</details>

## Stop the service cleanly

> **Concept.** A lab is left safe by removing exactly what it added: stop the listener, confirm the port is closed, and delete the temporary firewall exception. This is attack-surface reduction in miniature — every forgotten listener and every stale allow rule is reachable long after the exercise that justified it. Verification comes before deletion, because rule sets and ports may have changed since they were created.

If the decoy runs in the foreground, use `Ctrl+C` in its terminal. Verify that the port is no longer listening:

```bash
sudo ss -ltnp '( sport = :8088 )'
```

**Output:**

```text
State  Recv-Q Send-Q Local Address:Port Peer Address:Port Process
```

The empty result indicates that no TCP listener remains on port 8088.

Then remove temporary firewall access if the lab no longer needs it:

```bash
sudo ufw status numbered
```

**Output:**

```text
Status: active

     To                         Action      From
[ 1] 22/tcp                     ALLOW IN    192.168.56.0/24
[ 2] 8088/tcp                   ALLOW IN    192.168.56.0/24
[ 3] 514/udp                    ALLOW OUT   192.168.56.10
```

Run this immediately before deleting anything, because rule numbers and rule sets may have changed.

```bash
sudo ufw delete allow from 192.168.56.0/24 to any port 8088 proto tcp
```

**Output:**

```text
Rule deleted
```

The `delete` subcommand removes the exact inbound exception for the decoy port by matching its rule text, not its index.

<details>
<summary><strong>Read more</strong> — leave nothing behind</summary>

Removing the inbound exception is how you return the host to its hardened state. Leaving a lab rule in place means that a forgotten decoy port stays reachable long after the exercise ended.

</details>

## Operational guidance

> **Concept.** Running a honeypot is a governance problem as much as a technical one: who authorised it, what may be retained, for how long, and how the data is protected. Legal and ethical constraints — consent, privacy, data minimisation and retention limits — apply to a decoy exactly as they apply to any other monitoring system. The technical controls (isolation, egress restriction, log forwarding) are what make the governance promises enforceable rather than aspirational.

- Keep the honeypot in an isolated VLAN, a private subnet, or a dedicated virtual switch.
- Use restrictive egress rules; do not allow the decoy to reach production systems or arbitrary Internet destinations.
- Keep the host operating system patched even if the decoy service is intentionally simple.
- Send logs off the decoy host when feasible and protect them with access controls and retention policies.
- Treat IP addresses as sensitive operational data and follow applicable privacy, retention, and organizational policies.
- Classify logged connections carefully; observed traffic may be automated scanning, monitoring, misconfiguration, or an authorized test.
- Never perform counterattacks or active retaliation based on events observed in a honeypot.

<details>
<summary><strong>Read more</strong> — from one decoy to a monitored fleet</summary>

A single PentBox listener is a teaching tool; a real deployment scales it deliberately.

* **Multiple ports.** PentBox's fast mode monitors one port per process, so a fleet is one process per port — for example 22, 80, 443 and 3389 — under a supervisor such as `systemd` or `supervisord`, each writing to its own log. Wider cover means more captured contacts, and also more noise to triage.
* **DMZ placement.** Put the decoy in a demilitarised zone or a dedicated VLAN with a one-way reachability model: the monitored ports receive traffic, but the decoy initiates nothing outward except the logging path.
* **Alerts.** Forward the log to a collector and alert on new source addresses. `rsyslog` and `syslog-ng` can each raise a mail alert, and the syslog wire format is standardised in RFC 5424; detection is only as timely as the path from the log to a human.
* **Central analysis.** Ship the logs to an ELK stack (Elasticsearch, Logstash, Kibana) or to Splunk and build a dashboard of sources, ports and time. The value is correlation: the same address seen in the honeypot log, the firewall log and the authentication log is a far stronger signal than any single file.

</details>

<details>
<summary><strong>Read more</strong> — outbound control, law and ethics</summary>

* **Limit outbound.** Keep `ufw default deny outgoing` in force and open only the logging path, so that a compromised decoy cannot be used as a pivot, a proxy or a cryptocurrency miner. Egress filtering is the difference between an isolated sensor and an open relay.
* **Authorisation.** Written approval should name the segment, the ports, the monitoring period and the accountable person; an unauthorised honeypot can breach an organisation's own acceptable-use rules even when it is well intentioned.
* **Data protection.** Logged addresses are personal data in many jurisdictions, so minimise what you collect, set a retention period, restrict access and delete on schedule. In the EU, honeypot logs fall within the GDPR and, for network operators, the ePrivacy rules.
* **Proportionality.** Because a decoy may record traffic from third parties, keep the recorded fields to what the purpose requires — typically the source address, port and time — and never escalate to counterattack.

</details>

## Key takeaways

> **Concept.** A low-interaction honeypot is best understood as a controlled sensor: it listens on a deliberate port, records contact, and gives defenders a safe place to practise detection and log analysis. Its quality depends far less on how convincing the decoy looks than on isolation, egress control, evidence handling and careful interpretation. Read every entry as an observation whose cause must be confirmed with context.

A safe low-interaction honeypot is primarily a controlled sensor: it listens on a deliberate port, records contact, and gives defenders a place to practice detection and log analysis.

The quality of the deployment depends more on isolation, egress control, evidence handling, and careful interpretation than on making the decoy appear sophisticated.

Start with a high, unused port in a private lab, verify one authorized connection, and document both the observed event and the firewall boundaries that contained it.
