# UFW Firewall Administration: An Educational Guide

> **Use only on systems you own or are authorized to administer.** Firewall changes can interrupt SSH access, web traffic, monitoring, and other production services. Keep a recovery console or out-of-band access available before applying changes remotely.

## What UFW does

UFW, or Uncomplicated Firewall, is a command-line management layer for Linux firewall rules. It makes common allow and deny operations easier to read than directly editing low-level packet-filtering rules. UFW can control access by source IP address, subnet, service profile, port, and protocol. It is commonly available on Debian-based distributions such as Ubuntu and Debian.

UFW rules are evaluated together with the host's network services, routing, cloud security groups, and any upstream firewall. A rule that allows SSH in UFW does not help if the cloud firewall blocks the connection, and a UFW rule does not replace application authentication or system hardening.

## Prerequisites and safety

You need administrative privileges. Connect through an authorized SSH session or a local console, and identify the SSH port before enabling the firewall. If the server uses a nonstandard port, substitute that port in the examples below.

A useful initial check is:

```bash
id
sudo ss -tulpn
```

Example output:

```text
uid=1000(admin) gid=1000(admin) groups=1000(admin)
Netid State  Local Address:Port  Process
LISTEN 0      128    0.0.0.0:22   users:(('sshd',pid=742,fd=3))
LISTEN 0      128    0.0.0.0:80   users:(('apache2',pid=911,fd=4))
```

The `id` command confirms the current identity and groups. The `ss` command lists listening TCP and UDP sockets. In this example, SSH listens on port 22 and HTTP listens on port 80. This inventory should guide the firewall rules; do not blindly allow every discovered service.

## Install UFW

On a Debian-based system, refresh package metadata and install UFW:

```bash
sudo apt update
sudo apt install ufw
```

Representative output:

```text
Reading package lists... Done
Building dependency tree... Done
The following NEW packages will be installed:
  ufw
Setting up ufw ...
```

`apt update` downloads current package metadata. `apt install ufw` installs the firewall management utility and its supporting files. The exact output varies by distribution and package version. Review package prompts carefully rather than accepting unexpected changes automatically.

Check the initial state:

```bash
sudo ufw status verbose
```

Typical output after installation:

```text
Status: inactive
```

An inactive status means UFW is installed but is not currently enforcing its rules. This initial disabled state is useful because it allows you to create an access policy before activating enforcement.

## Establish a safe baseline

Before enabling UFW, choose default policies. A common server baseline denies unsolicited incoming connections and permits outgoing connections:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

Representative output:

```text
Default incoming policy changed to 'deny'
Default outgoing policy changed to 'allow'
```

The incoming policy controls new connections initiated toward the server. The outgoing policy controls connections initiated by the server. These defaults do not replace specific service rules, and they do not necessarily block traffic that belongs to an already established connection.

## Allow SSH before enabling

Allow the actual SSH port before activating the firewall:

```bash
sudo ufw allow 22/tcp
```

Representative output:

```text
Rule added
Rule added (v6)
```

The rule permits TCP connections to port 22 for both IPv4 and IPv6 when both address families are configured. If SSH uses another port, use that port instead:

```bash
sudo ufw allow 7822/tcp
```

Do not add a rule for port 22 merely because it is conventional. Confirm the listening port with `ss`, the SSH daemon configuration, or your hosting provider's console. Keeping an existing SSH session open while testing a new rule helps reduce the risk of lockout.

## Allow application services

UFW can use named application profiles:

```bash
sudo ufw app list
```

Example output:

```text
Available applications:
  Apache
  Apache Full
  OpenSSH
```

The list is generated from installed UFW application profiles. A profile usually maps a friendly name to one or more ports and protocols. Inspect a profile before using it:

```bash
sudo ufw app info "Apache Full"
```

Example output:

```text
Profile: Apache Full
Title: Web Server (HTTP,HTTPS)
Ports: 80/tcp,443/tcp
```

The output explains which ports the profile opens. `Apache Full` generally permits both HTTP and HTTPS. If only encrypted web traffic should be reachable, prefer a profile or explicit rule that allows only port 443.

To allow the profile:

```bash
sudo ufw allow "Apache Full"
```

Representative output:

```text
Rule added
Rule added (v6)
```

Quotation marks are important when a profile name contains spaces. Avoid opening a service that is not installed, needed, monitored, and securely configured.

## Allow a specific IP address

To allow all supported traffic from one trusted source address:

```bash
sudo ufw allow from 192.168.1.1
```

Representative output:

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

Typical output:

```text
Rule deleted
```

The deletion command must match the rule's meaning. If the original rule included a port or protocol, delete the corresponding complete rule or use numbered rules.

## Allow an entire subnet

```bash
sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp
```

Representative output:

```text
Rule added
```

The `/24` network contains addresses in the range represented by `192.168.1.0` through `192.168.1.255`, subject to network conventions. This example permits SSH from that network only. Use subnet ranges carefully: an overly broad trusted network can expose administrative services to more hosts than intended.

## Inspect and delete numbered rules

```bash
sudo ufw status numbered
```

Example output:

```text
Status: active

     To                         Action      From
[ 1] 22/tcp                     ALLOW IN    192.168.1.1
[ 2] 80/tcp                     ALLOW IN    Anywhere
[ 3] 443/tcp                    ALLOW IN    Anywhere
```

The numbered listing shows rule order, destination, action, direction, and source. Review it before deleting anything. To remove the second rule:

```bash
sudo ufw delete 2
```

Representative output:

```text
Deleting:
 allow 80/tcp
Proceed with operation (y|n)? y
Rule deleted
```

Rule numbers can change after deletion. Re-run `ufw status numbered` before deleting another rule rather than relying on an old list.

## Deny an address or subnet

To block a source address:

```bash
sudo ufw deny from 192.168.1.1
```

Representative output:

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

Typical output:

```text
Rule deleted
```

## Enable and verify the firewall

After adding all required rules, enable UFW:

```bash
sudo ufw enable
```

Typical output:

```text
Command may disrupt existing ssh connections. Proceed with operation (y|n)? y
Firewall is active and enabled on system startup
```

The warning exists because an incorrect policy can terminate or prevent SSH access. Confirm that the SSH allow rule is present before answering `y`. The second line indicates that UFW is active now and configured to start during boot.

Review the complete configuration:

```bash
sudo ufw status verbose
sudo ufw show added
```

Example output:

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

The verbose status displays whether UFW is active, logging state, default policies, and configured rules. `show added` displays rules entered through UFW commands. The IPv6 entries are important when the server has IPv6 connectivity; otherwise an IPv4-only review may give a false sense of coverage.

## Logging and troubleshooting

Enable moderate logging when you need visibility into blocked traffic:

```bash
sudo ufw logging medium
sudo ufw status verbose
```

Representative output:

```text
Logging enabled
Status: active
Logging: on (medium)
```

Logging levels affect the amount of firewall event information recorded. More logging can help troubleshooting but may increase log volume. Review the system's firewall and kernel logs using the logging system used by your distribution, for example:

```bash
sudo journalctl -k --since "10 minutes ago" | grep -i ufw
```

Example output:

```text
kernel: [UFW BLOCK] IN=eth0 OUT= SRC=203.0.113.50 DST=198.51.100.10 PROTO=TCP DPT=23
```

`UFW BLOCK` indicates that a packet was denied. `SRC` is the source address, `DST` is the destination address, `PROTO` is the protocol, and `DPT` is the destination port. These events help identify unexpected scans or incorrectly blocked legitimate traffic. Public documentation examples use reserved documentation addresses; real logs will contain your actual network addresses.

## Reload, reset, and disable

After manual changes to supported UFW configuration, reload the rules:

```bash
sudo ufw reload
```

Typical output:

```text
Firewall reloaded
```

Reloading applies the configuration without a full system reboot. If the firewall behaves unexpectedly, inspect the rules and logs before making further changes.

To disable enforcement temporarily:

```bash
sudo ufw disable
```

Representative output:

```text
Firewall stopped and disabled on system startup
```

Disabling removes UFW's active enforcement and also prevents automatic activation at boot. Use it only when you understand the exposure created by the change. A safer alternative is often to correct or delete a specific rule.

To return UFW to a clean state:

```bash
sudo ufw reset
```

Typical output:

```text
Resetting all rules to installed defaults. Proceed with operation (y|n)? y
Backing up 'user.rules' to '/etc/ufw/user.rules.20261008_190000'
```

Reset removes user-defined UFW rules and restores defaults. It may also remove the SSH exception that keeps remote access possible, so use a console or ensure you can immediately recreate the required rules. Backups and exact output vary by distribution.

## ICMP and ping considerations

Ping uses ICMP, which is not TCP or UDP. Blocking ping does not make a server invisible and can interfere with monitoring, path MTU discovery, diagnostics, and IPv6 operation. In most environments, it is better to permit necessary ICMP types and rely on authentication, patching, least privilege, and monitoring for security.

If a documented policy requires changing pre-routing or pre-input ICMP behavior, back up the configuration first:

```bash
sudo cp -a /etc/ufw/before.rules /etc/ufw/before.rules.backup
sudoedit /etc/ufw/before.rules
sudo ufw reload
```

Expected reload output:

```text
Firewall reloaded
```

The backup allows recovery from a syntax or policy mistake. `sudoedit` opens the file through a safer editing workflow. After editing, `ufw reload` asks UFW to load the new configuration. Test both intended and unintended traffic afterward, and consider IPv6 configuration separately because IPv4 ICMP rules do not automatically describe IPv6 behavior.

## Verification checklist

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

## Key lessons

A firewall policy should be built from an inventory of services and trusted sources, not from copied commands alone. Start with a restrictive default, allow only necessary traffic, inspect the resulting rules, and test connectivity before closing the administrative session. Explain every command's output in the change record so another administrator can understand what was enabled, blocked, or removed.
