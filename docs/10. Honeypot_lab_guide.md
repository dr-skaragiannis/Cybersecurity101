# Operating a Low-Interaction Honeypot in an Isolated Lab

> **Authorized defensive use only.** A honeypot is a decoy service intended to observe unsolicited access attempts. Deploy it only in a controlled lab or in an organization with written approval, documented monitoring, data-retention rules, and strict network isolation. Do not retaliate against source addresses or use a honeypot as a platform for scanning or attacking others.

## Purpose and limitations

A low-interaction honeypot listens on a selected network port, records connection attempts, and may return a simple banner or warning message. It is useful for learning how internet scans and unauthorized probes appear in logs. It is not a full simulation of SSH, HTTP, FTP, or another protocol, and it must not be mistaken for a replacement for production intrusion detection, patching, authentication, or a firewall.

A safe design places the decoy on a dedicated virtual machine or isolated VLAN. Permit only the inbound ports you intentionally monitor, restrict outbound traffic to essential management destinations, and send logs to a separate collector if possible. The key security goal is containment: a compromised decoy must not provide a path to production systems or the wider Internet.

## Reference architecture

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

The test client should be a machine you control. The firewall should block the honeypot from reaching production networks and should deny unnecessary outbound connections. A separate log-analysis host reduces the chance that evidence is altered if the decoy is compromised.

## Prepare the laboratory host

First, identify the operating system, current account, IP address, and listening services:

```bash
cat /etc/os-release
id
ip -brief address
sudo ss -tulpn
```

Representative output:

```text
PRETTY_NAME="Kali GNU/Linux Rolling"
uid=1000(analyst) gid=1000(analyst) groups=1000(analyst),27(sudo)
eth0             UP             192.168.56.20/24
Netid State  Local Address:Port  Process
LISTEN 0      128    127.0.0.1:631  users:(('cupsd',pid=621,fd=6))
```

The operating-system information ensures that you use the correct package-management commands. The `id` output confirms the local user and whether they have administrative privileges. `ip -brief address` shows the interface and assigned lab address. `ss -tulpn` inventories services that already listen on TCP or UDP ports. Do not start a decoy on a port already used by a real service.

Update the underlying operating system before adding lab software:

```bash
sudo apt update
sudo apt upgrade -y
```

Representative output:

```text
Hit:1 http://http.kali.org/kali kali-rolling InRelease
Reading package lists... Done
Building dependency tree... Done
0 upgraded, 0 newly installed, 0 to remove
```

`apt update` retrieves package metadata. `apt upgrade -y` applies available updates without asking for each confirmation. The final line in this example indicates that no upgrades were waiting. Keep the operating system current even when the decoy service is intentionally simplistic; the host itself should not be intentionally vulnerable.

## Install and inspect the tool

A Ruby-based low-interaction honeypot tool may be distributed as source code. Clone only a reviewed, authorized training copy into a dedicated directory:

```bash
sudo apt install -y git ruby
mkdir -p ~/honeypot-lab
cd ~/honeypot-lab
git clone https://example.invalid/training/pentbox.git pentbox
cd pentbox
ruby --version
```

Representative output:

```text
Cloning into 'pentbox'...
done.
ruby 3.1.xp0 (202x-xx-xx revision ...) [x86_64-linux-gnu]
```

`git` retrieves version-controlled source code and `ruby` provides the interpreter. The example uses a deliberately non-routable placeholder address; substitute only a repository that your organization has reviewed and approved. The Ruby version confirms that the runtime is installed. Read the source code, verify its origin, and avoid executing unknown scripts as root.

Inspect files and the program's help before executing it:

```bash
find . -maxdepth 2 -type f | sort | head -30
sed -n '1,80p' pentbox.rb
ruby pentbox.rb --help
```

Representative output:

```text
./LICENSE
./README.md
./pentbox.rb
./other/log_honeypot.txt
Usage: pentbox.rb [options]
```

The `find` command gives a compact inventory. `sed` prints only the first portion of the script for review. The help output may differ between versions; if a script does not support `--help`, stop and consult its local documentation rather than guessing its behavior.

## Configure a minimal decoy

Choose a high, unused TCP port for initial testing, such as `8088`, rather than impersonating a privileged or production service. Confirm that it is unused:

```bash
sudo ss -ltnp '( sport = :8088 )'
```

Expected output before the decoy starts:

```text
State  Recv-Q Send-Q Local Address:Port Peer Address:Port Process
```

The header with no rows means no TCP process is listening on port 8088. If a row appears, select another port or stop the conflicting lab service only if you are authorized to do so.

A typical menu-based workflow for a low-interaction tool is shown below. The exact menu names and choices depend on the installed version:

```text
1) Network tools
2) Honeypot
3) Manual configuration

Listening port: 8088
Banner: Training decoy — access is monitored.
Log path: ./other/log_honeypot.txt
```

Use a clearly documented banner and a dedicated log path. In an internal lab, a banner can help distinguish the decoy from a production service. Do not claim to be a real bank, government service, or another organization.

After starting the listener, verify the port locally:

```bash
sudo ss -ltnp '( sport = :8088 )'
```

Representative output:

```text
State  Recv-Q Send-Q Local Address:Port Peer Address:Port Process
LISTEN 0      128    0.0.0.0:8088  0.0.0.0:*     users:(('ruby',pid=1842,fd=7))
```

`LISTEN` means the process is waiting for new TCP connections. `0.0.0.0:8088` means it accepts IPv4 connections on all network interfaces. For a strictly local test, prefer binding to `127.0.0.1` if the tool supports it. The PID and process name help identify the listener for monitoring and controlled shutdown.

## Firewall containment

A host firewall should allow only intended lab access and should restrict unnecessary outbound activity. The following is a conceptual UFW policy for an isolated subnet; adapt it to the actual management path and test from a console before disconnecting SSH.

```bash
sudo ufw default deny incoming
sudo ufw default deny outgoing
sudo ufw allow from 192.168.56.0/24 to any port 22 proto tcp
sudo ufw allow from 192.168.56.0/24 to any port 8088 proto tcp
sudo ufw allow out to 192.168.56.10 port 514 proto udp
sudo ufw enable
sudo ufw status numbered
```

Representative output:

```text
Status: active

     To                         Action      From
[ 1] 22/tcp                     ALLOW IN    192.168.56.0/24
[ 2] 8088/tcp                   ALLOW IN    192.168.56.0/24
[ 3] 514/udp                    ALLOW OUT   192.168.56.10
```

The default-deny policies reject unsolicited inbound and outbound traffic unless a later rule permits it. Rule 1 permits SSH management only from the lab subnet. Rule 2 permits access to the decoy only from that same subnet. Rule 3 permits UDP syslog forwarding only to a designated log collector. If the host needs DNS, NTP, package updates, or another management service, add narrowly scoped temporary rules, complete the task, then remove them.

## Test from an authorized client

From a separate client inside the isolated subnet, test only the listener that you created:

```bash
nc -vz 192.168.56.20 8088
```

Representative output:

```text
Connection to 192.168.56.20 8088 port [tcp/*] succeeded!
```

Netcat attempts a TCP connection. The success message confirms network reachability, firewall allowance, and a listener on the target port. It does not demonstrate that a full application protocol is implemented.

To observe a text banner, if the lab tool returns one:

```bash
nc 192.168.56.20 8088
```

Representative output:

```text
Training decoy — access is monitored.
```

This connects interactively and displays the decoy's banner. End the session with `Ctrl+C`. Do not send exploit payloads; a connection test is sufficient to validate logging and containment.

## Monitor and interpret logs

Watch the decoy log in real time:

```bash
tail -F ./other/log_honeypot.txt
```

Representative output:

```text
2026-10-08T20:25:14+03:00 connection source=192.168.56.30:49152 destination_port=8088
```

`tail -F` continues following the file even if it is rotated or recreated. The timestamp supports incident reconstruction. The source IP and ephemeral source port identify the connecting test client, while `destination_port` identifies the decoy service.

A typical console message can look like:

```text
CONNECTION ATTEMPT DETECTED from 192.168.56.30:49152
```

This indicates that a remote endpoint established or attempted a connection. It is an event, not proof of malicious intent: health checks, scanners, misconfigurations, and authorized tests can all generate similar entries. Preserve surrounding context before classifying an event.

Review recent entries without continuously following the file:

```bash
tail -n 50 ./other/log_honeypot.txt
```

Example output:

```text
2026-10-08T20:25:14+03:00 connection source=192.168.56.30:49152 destination_port=8088
2026-10-08T20:26:03+03:00 connection source=192.168.56.31:49411 destination_port=8088
```

The two records show separate connection attempts from different hosts and source ports. A source port normally changes between outbound connections and is not a reliable user or device identity by itself. Correlate it with firewall logs, DHCP leases, asset inventory, and approved test records.

## Basic log summary

Extract source addresses in a lab log format and count them:

```bash
awk -F'source=' '/source=/{split($2,a,":"); print a[1]}' ./other/log_honeypot.txt | sort | uniq -c | sort -nr
```

Representative output:

```text
      8 192.168.56.30
      2 192.168.56.31
```

`awk` selects records containing `source=` and isolates the source address before the colon. `sort` places identical addresses together, `uniq -c` counts them, and the final `sort -nr` orders the counts numerically from highest to lowest. This simple count helps identify frequent contact, but it does not establish malicious intent or attribution.

## Preserve evidence safely

Rotate a copy of the log before analysis and protect permissions:

```bash
install -d -m 0750 ~/honeypot-evidence
cp --preserve=mode,timestamps ./other/log_honeypot.txt ~/honeypot-evidence/log_$(date +%F_%H%M%S).txt
chmod 0640 ~/honeypot-evidence/*.txt
sha256sum ~/honeypot-evidence/*.txt
```

Representative output:

```text
c9a4...e17b  /home/analyst/honeypot-evidence/log_2026-10-08_202700.txt
```

The directory mode limits access to the owner and group. `cp --preserve` retains useful metadata. The SHA-256 digest records a fingerprint of the evidence file: if the file changes, its digest will normally change. A hash proves integrity relative to a recorded value; it does not prove who generated the event.

## Stop the service cleanly

If the decoy runs in the foreground, use `Ctrl+C` in its terminal. Verify that the port is no longer listening:

```bash
sudo ss -ltnp '( sport = :8088 )'
```

Expected output:

```text
State  Recv-Q Send-Q Local Address:Port Peer Address:Port Process
```

The empty result indicates that no TCP listener remains on port 8088. Then remove temporary firewall access if the lab no longer needs it:

```bash
sudo ufw status numbered
sudo ufw delete allow from 192.168.56.0/24 to any port 8088 proto tcp
```

The first command should always be run immediately before deletion, because rule numbers and rule sets may have changed. The second command removes the exact inbound exception for the decoy port.

## Operational guidance

- Keep the honeypot in an isolated VLAN, a private subnet, or a dedicated virtual switch.
- Use restrictive egress rules; do not allow the decoy to reach production systems or arbitrary Internet destinations.
- Keep the host operating system patched even if the decoy service is intentionally simple.
- Send logs off the decoy host when feasible and protect them with access controls and retention policies.
- Treat IP addresses as sensitive operational data and follow applicable privacy, retention, and organizational policies.
- Classify logged connections carefully; observed traffic may be automated scanning, monitoring, misconfiguration, or an authorized test.
- Never perform counterattacks or active retaliation based on events observed in a honeypot.

## Key takeaways

A safe low-interaction honeypot is primarily a controlled sensor: it listens on a deliberate port, records contact, and gives defenders a place to practice detection and log analysis. The quality of the deployment depends more on isolation, egress control, evidence handling, and careful interpretation than on making the decoy appear sophisticated. Start with a high, unused port in a private lab, verify one authorized connection, and document both the observed event and the firewall boundaries that contained it.
