# tcpdump: Practical Packet Capture and Analysis in an Authorized Lab

> **Authorized use only.** Packet capture can expose credentials, session tokens, personal data, internal addresses, and application content. Capture only on networks and systems you own or are explicitly authorized to monitor. Use a lab interface whenever possible, minimize capture scope and retention, and protect capture files as sensitive evidence.

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

The paragraph after each output explains *what the tool does* and *why the step matters*, so you learn the method rather than only the syntax. Anything longer — extra flags, pitfalls, background theory — is tucked into a collapsible box.

<details>
<summary><strong>Read more</strong> — the conventions used throughout this guide</summary>

* **A command box is what you type; an output box is what the machine prints back.** A command that succeeds often prints nothing at all — on Linux, silence usually means success.
* **A command box may contain more than one line.** When several commands belong to a single idea they are grouped together; run them in order, one at a time, and read the output after each.
* **`sudo` runs the command as an administrator.** Capture normally needs the ability to open a raw network socket, so most commands here are prefixed with it.
* **Text inside angle brackets, for example `-i <interface>`, is a placeholder** that you replace with a real value. The angle brackets themselves are never typed.
* **Output is representative, not identical.** Interfaces, addresses, versions and packet counts differ on your own lab. What matters is the shape of the output, not the exact digits.

</details>

### Table of contents

1. [What tcpdump does](#what-tcpdump-does)
2. [Safe lab architecture](#safe-lab-architecture)
3. [Install and identify interfaces](#install-and-identify-interfaces)
4. [Start with a bounded capture](#start-with-a-bounded-capture)
5. [Names, numbers, and concise output](#names-numbers-and-concise-output)
6. [Filtering by protocol, port, host, and network](#filtering-by-protocol-port-host-and-network)
7. [Direction and link-layer information](#direction-and-link-layer-information)
8. [Verbosity and timestamps](#verbosity-and-timestamps)
9. [Payload rendering and snap length](#payload-rendering-and-snap-length)
10. [TCP flags and connection diagnosis](#tcp-flags-and-connection-diagnosis)
11. [Read and write PCAP files](#read-and-write-pcap-files)
12. [Inspect in a graphical analyzer](#inspect-in-a-graphical-analyzer)
13. [Remote capture with safeguards](#remote-capture-with-safeguards)
14. [Promiscuous mode](#promiscuous-mode)
15. [Good capture practice](#good-capture-practice)
16. [Key lessons](#key-lessons)

---

## What tcpdump does

`tcpdump` captures and decodes packets observed by a network interface. It uses the libpcap capture framework and accepts a powerful filter language, commonly called BPF syntax, to limit what is collected.

It is valuable for troubleshooting DNS, TCP handshakes, service reachability, routing, and protocol behavior. It is not an intrusion tool; its primary value is careful observation and interpretation.

A typical packet summary contains a timestamp, source endpoint, destination endpoint, protocol information, TCP flags where applicable, and packet length.

<details>
<summary><strong>Read more</strong> — why the same command can look different on two machines</summary>

The exact rendering varies with operating system, tcpdump version, link type, privileges, and traffic present at the time of capture. That is why two hosts can print slightly different layouts for the same packet, and why the startup line of every capture is worth reading before you trust the columns beneath it.

</details>

## Safe lab architecture

The diagram below shows the smallest useful lab: one client you control, one capture host, and an optional analysis workstation.

```text
+----------------------+        private network        +----------------------+
| Test client          | ----------------------------> | Capture host         |
| 192.168.56.30        |       authorized traffic      | 192.168.56.20        |
+----------------------+                                | interface: eth0      |
                                                        | tcpdump / pcap file  |
                                                        +----------+-----------+
                                                                   |
                                                                   | protected copy
                                                                   v
                                                        +----------------------+
                                                        | Analysis workstation |
                                                        | Wireshark optional   |
                                                        +----------------------+
```

Use a test client you control to generate predictable traffic, such as a DNS lookup, HTTP request to a local test server, or a ping inside the lab.

Store captures on an encrypted or access-controlled location. Do not send captures containing sensitive traffic through unprotected chat, email, or public file-sharing services.

<details>
<summary><strong>Read more</strong> — keeping capture inside the lab boundary</summary>

The client generates the traffic, the capture host observes it, and the analysis workstation only ever sees a *copy*. Keeping those three roles separate means a capture can be deleted from the capture host without losing the analysis, and it keeps captured payloads off shared or personal machines.

</details>

## Install and identify interfaces

On Debian-based systems, refresh the package index, install tcpdump, and list capture-capable interfaces. Start by updating the local package list:

```bash
sudo apt update
```

**Output:**

```text
Hit:1 http://deb.debian.org/debian stable InRelease
Reading package lists... Done
```

`apt update` refreshes the local package metadata so that later install commands see current package versions. It changes nothing else, and it normally ends with `Reading package lists... Done`.

Install the capture utility:

```bash
sudo apt install -y tcpdump
```

**Output:**

```text
Setting up tcpdump (4.x) ...
```

`apt install -y tcpdump` installs tcpdump and its dependencies, and `-y` answers the confirmation prompt automatically. When the command returns without an error, the `tcpdump` binary is available for the next step.

List the interfaces tcpdump is allowed to capture on:

```bash
sudo tcpdump -D
```

**Output:**

```text
1.eth0 [Up, Running]
2.lo [Up, Running, Loopback]
3.any (Pseudo-device that captures on all interfaces)
```

`tcpdump -D` lists the interfaces tcpdump can use. `eth0` is commonly an Ethernet-style network interface, `lo` is the local loopback interface, and `any` captures across multiple interfaces.

<details>
<summary><strong>Read more</strong> — interface names differ from one distribution to the next</summary>

Interface names may be `ens33`, `enp0s3`, `wlan0`, or another distribution-specific name. That is why you run `tcpdump -D` first and copy the exact name, rather than assuming the classic `eth0` exists.

</details>

View the installed version and the usage summary. First, the version:

```bash
tcpdump --version
```

**Output:**

```text
tcpdump version 4.x
libpcap version 1.x
```

The version output identifies the tcpdump and the libpcap builds it links against; the exact numbers differ per distribution.

Then print the first lines of the usage text:

```bash
tcpdump --help | head -25
```

**Output:**

```text
Usage: tcpdump [ -AbdDefhHIJKlLnNOpqStuUvxX# ] ...
```

The help summary lists the command options. Piping it through `head -25` keeps the output short, showing only the first 25 lines.

<details>
<summary><strong>Read more</strong> — features differ between releases</summary>

Features and supported protocol decoders can differ across releases, so verify syntax locally with `man tcpdump` when a command behaves differently from a guide.

</details>

## Start with a bounded capture

Capturing without limits can create very large files or expose more data than necessary. Begin with a count limit:

```bash
sudo tcpdump -i eth0 -c 5
```

**Output:**

```text
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
20:32:14.102431 IP 192.168.56.30.49152 > 192.168.56.20.80: Flags [S], seq 125430, win 64240, length 0
20:32:14.102580 IP 192.168.56.20.80 > 192.168.56.30.49152: Flags [S.], seq 987650, ack 125431, win 65160, length 0
5 packets captured
```

`-i eth0` selects the interface and `-c 5` stops after five packets. The first packet is a TCP SYN from the client ephemeral port `49152` to port 80. The second is SYN-ACK, the server's reply during a TCP three-way handshake.

The final line confirms how many packets tcpdump captured. `verbose output suppressed` is normal; it suggests `-v` if more detail is needed.

<details>
<summary><strong>Read more</strong> — the startup line, and why a count limit is a habit</summary>

The first two lines are not packets at all: they are the startup banner, telling you the interface, the link type (`EN10MB` is Ethernet) and the snapshot length in use. Everything after them is one line per packet.

A count limit (`-c`) is the simplest guard against the two real problems of capture — a file that grows without bound, and a stream you can no longer realistically review.

</details>

## Names, numbers, and concise output

By default, tcpdump may resolve addresses or service names. Disable name resolution when you need faster, unambiguous output:

```bash
sudo tcpdump -i eth0 -c 3 -nn
```

**Output:**

```text
20:33:01.100001 IP 192.168.56.30.53530 > 192.168.56.20.53: 1234+ A? example.test. (30)
```

`-n` avoids hostname resolution and the second `n` avoids service-name conversion. Therefore, `53` remains a port number rather than being displayed as `domain`.

This reduces side effects and delays caused by DNS lookups during analysis.

Use quick mode when a short flow summary is enough:

```bash
sudo tcpdump -i eth0 -c 3 -q -nn
```

**Output:**

```text
IP 192.168.56.30.49152 > 192.168.56.20.80: tcp 0
IP 192.168.56.20.80 > 192.168.56.30.49152: tcp 0
```

`-q` prints less protocol detail. It is useful for a high-level view but can hide fields needed for diagnosis, such as TCP flags and sequence-related details.

<details>
<summary><strong>Read more</strong> — why a lab should prefer numbers to names</summary>

On a lab network the resolver is often slow, unreachable, or deliberately isolated, and a single lookup that never answers can stall the capture. Numeric output is also unambiguous: an address can never be mistaken for a hostname that merely looks like one, which matters when you compare a capture against logs or a firewall table.

</details>

## Filtering by protocol, port, host, and network

A BPF filter is placed after the tcpdump options. Capture ten ICMP packets on the selected interface:

```bash
sudo tcpdump -i eth0 -c 10 -nn icmp
```

**Output:**

```text
20:34:10.000200 IP 192.168.56.30 > 192.168.56.20: ICMP echo request, id 4512, seq 1, length 64
20:34:10.000611 IP 192.168.56.20 > 192.168.56.30: ICMP echo reply, id 4512, seq 1, length 64
```

The first line is an ICMP echo request, often produced by `ping`; the second line is the matching reply. The identifier and sequence number help pair request and reply packets.

A request without a reply can indicate a firewall rule, a routing problem, host unavailability, or packet loss; it is not automatically evidence of malicious activity.

Capture traffic involving a web port:

```bash
sudo tcpdump -i eth0 -c 10 -nn 'port 80'
```

**Output:**

```text
IP 192.168.56.30.49152 > 192.168.56.20.80: Flags [S], length 0
IP 192.168.56.20.80 > 192.168.56.30.49152: Flags [S.], length 0
```

The filter `port 80` matches packets whose source or destination port is 80. To restrict to a server-side destination port, use `dst port 80`; to observe traffic originating from port 80, use `src port 80`.

Capture only traffic involving one host:

```bash
sudo tcpdump -i eth0 -c 10 -nn host 192.168.56.30
```

**Output:**

```text
IP 192.168.56.30.49152 > 192.168.56.20.80: Flags [S], length 0
IP 192.168.56.20.80 > 192.168.56.30.49152: Flags [S.], length 0
```

The `host` filter matches packets in either direction that involve the single address `192.168.56.30`. It is the simplest way to watch everything a specific host sends and receives.

Restrict the capture to a whole subnet:

```bash
sudo tcpdump -i eth0 -c 10 -nn net 192.168.56.0/24
```

**Output:**

```text
IP 192.168.56.30.49152 > 192.168.56.20.80: Flags [S], length 0
IP 192.168.56.20.80 > 192.168.56.30.49152: Flags [S.], length 0
```

The `net` filter matches traffic to or from the stated network. Narrow filters reduce unrelated collection and improve readability.

A port range can be useful when testing a group of related services:

```bash
sudo tcpdump -i eth0 -c 10 -nn 'portrange 8000-8100'
```

This matches packets whose source or destination port is between 8000 and 8100 inclusive. Use it only where the range is relevant; broad ranges increase noise.

<details>
<summary><strong>Read more</strong> — building a filter that says exactly what you mean</summary>

The filter language is a grammar rather than a list of keywords: it combines terms such as `host`, `net`, `port`, `src`, `dst` and `portrange` into a single condition. Because the filter is applied before packets reach tcpdump, a narrow filter also reduces the work the capture host has to do.

The quoting matters. `'port 80'` is passed to the filter compiler as one argument, whereas an unquoted `port 80` would be read by the shell as the word `port` followed by the separate argument `80`.

</details>

## Direction and link-layer information

On platforms that support capture direction, restrict output to inbound or outbound packets. Show only the packets the interface receives:

```bash
sudo tcpdump -i eth0 -c 5 -nn -Q in
```

**Output:**

```text
IP 192.168.56.20.80 > 192.168.56.30.49152: Flags [S.], seq 987650, ack 125431, win 65160, length 0
```

`-Q in` displays packets received by the interface, that is, inbound traffic.

Show only the packets the interface sends:

```bash
sudo tcpdump -i eth0 -c 5 -nn -Q out
```

**Output:**

```text
IP 192.168.56.30.49152 > 192.168.56.20.80: Flags [S], seq 125430, win 64240, length 0
```

`-Q out` displays packets sent by the interface. Support for both directions is platform and driver dependent; if it is unsupported, tcpdump reports an error.

Include Ethernet-level headers:

```bash
sudo tcpdump -i eth0 -c 2 -nn -e
```

**Output:**

```text
20:35:44.200001 08:00:27:aa:bb:01 > 08:00:27:aa:bb:20, ethertype IPv4 (0x0800), length 74: 192.168.56.30.49152 > 192.168.56.20.80: Flags [S], length 0
```

`-e` prints link-layer information, including source and destination MAC addresses on Ethernet captures. The `ethertype IPv4 (0x0800)` identifies the network-layer protocol carried in the frame.

<details>
<summary><strong>Read more</strong> — when direction and MAC addresses are worth having</summary>

If tcpdump reports that direction is unsupported, use a host, address, or port filter instead. Directional capture is most useful when you already know which side of a conversation you want to watch, for example a server's replies.

MAC addresses are normally meaningful only on the local Layer-2 segment: they change at every router, so a MAC in a capture tells you about the nearest switch or gateway, not about the original sender.

</details>

## Verbosity and timestamps

Increase decoding detail only when it is useful, starting with `-v`:

```bash
sudo tcpdump -i eth0 -c 2 -nn -v
```

**Output:**

```text
IP (tos 0x0, ttl 64, id 22301, offset 0, flags [DF], proto TCP (6), length 60)
    192.168.56.30.49152 > 192.168.56.20.80: Flags [S], seq 125430, win 64240, length 0
```

`-v` adds an IPv4 header line with type of service, time-to-live, packet ID, fragmentation information, protocol number, and total length.

Add more protocol detail with a second `v`:

```bash
sudo tcpdump -i eth0 -c 2 -nn -vv
```

**Output:**

```text
IP (tos 0x0, ttl 64, id 22301, offset 0, flags [DF], proto TCP (6), length 60)
    192.168.56.30.49152 > 192.168.56.20.80: Flags [S], cksum 0xabcd (correct), seq 125430, win 64240, length 0
```

`-vv` adds further protocol fields, including the checksum status of the TCP segment.

Reserve the third level for genuinely deep protocol debugging:

```bash
sudo tcpdump -i eth0 -c 2 -nn -vvv
```

**Output:**

```text
IP (tos 0x0, ttl 64, id 22301, offset 0, flags [DF], proto TCP (6), length 60)
    192.168.56.30.49152 > 192.168.56.20.80: Flags [S], cksum 0xabcd (correct), seq 125430:125430, win 64240, options [mss 1460,sackOK,TS val 100 ecr 0,nop,wscale 7], length 0
```

`-vvv` decodes the most fields, including the negotiated TCP options such as `mss`, `sackOK`, `wscale` and timestamps. Extra verbosity can make troubleshooting easier but creates more output to review.

Timestamp formats change presentation, not packet contents. Omit timestamps entirely with `-t`:

```bash
sudo tcpdump -i eth0 -c 2 -nn -t
```

**Output:**

```text
IP 192.168.56.30.49152 > 192.168.56.20.80: Flags [S], length 0
IP 192.168.56.20.80 > 192.168.56.30.49152: Flags [S.], length 0
```

`-t` omits the timestamp column, leaving only the packet summary.

Print an absolute Unix-style timestamp with `-tt`:

```bash
sudo tcpdump -i eth0 -c 2 -nn -tt
```

**Output:**

```text
1759951801.102431 IP 192.168.56.30.49152 > 192.168.56.20.80: Flags [S], length 0
1759951801.102580 IP 192.168.56.20.80 > 192.168.56.30.49152: Flags [S.], length 0
```

`-tt` prints an absolute Unix-style timestamp, measured in seconds since the epoch.

Measure the delay between packets with `-ttt`:

```bash
sudo tcpdump -i eth0 -c 2 -nn -ttt
```

**Output:**

```text
00:00:00.000000 IP 192.168.56.30.49152 > 192.168.56.20.80: Flags [S], length 0
00:00:00.000149 IP 192.168.56.20.80 > 192.168.56.30.49152: Flags [S.], length 0
```

`-ttt` prints the time difference from the previous packet, which is useful for latency and timing analysis.

Use a human-readable date with `-tttt`:

```bash
sudo tcpdump -i eth0 -c 2 -nn -tttt
```

**Output:**

```text
2026-10-08 20:40:01.000010 IP 192.168.56.30.49152 > 192.168.56.20.80: Flags [S], length 0
2026-10-08 20:40:01.000149 IP 192.168.56.20.80 > 192.168.56.30.49152: Flags [S.], length 0
```

`-tttt` displays a date and time in a human-readable format instead of a bare clock time.

<details>
<summary><strong>Read more</strong> — choosing the right level of detail</summary>

Each `v` adds a layer: `-v` adds IP and TCP detail, `-vv` adds more protocol fields, and `-vvv` is reserved for genuinely deep protocol debugging. Start at the lowest level that answers your question, because every extra field is another column to read.

Timestamp layouts vary by version too, so treat the exact format shown here as a guide rather than a guarantee.

</details>

## Payload rendering and snap length

Payload display can reveal plaintext application content and therefore requires particular care. In a lab, inspect a small number of packets in ASCII:

```bash
sudo tcpdump -i eth0 -c 2 -nn -A 'tcp port 8080'
```

**Output:**

```text
GET /health HTTP/1.1
Host: 192.168.56.20:8080
User-Agent: lab-client/1.0
```

`-A` prints packet payload data as ASCII where possible. The shown data is an HTTP request: `GET` requests the `/health` path, `Host` identifies the intended virtual host and port, and `User-Agent` identifies the client software string.

Do not use this technique to collect passwords, cookies, authorization headers, or user content.

View packet bytes in hexadecimal and ASCII:

```bash
sudo tcpdump -i eth0 -c 1 -nn -X 'tcp port 8080'
```

**Output:**

```text
0x0000:  4500 0054 1a2b 4000 4006 ...
0x0030:  4745 5420 2f68 6561 6c74 6820 4854 5450  GET /health HTTP
```

`-X` prints a hexadecimal representation alongside ASCII decoding. The left-most values are byte offsets.

The snapshot length limits how many bytes tcpdump retains from each packet. For a deliberately limited capture:

```bash
sudo tcpdump -i eth0 -c 2 -nn -s 96 'tcp port 8080'
```

**Output:**

```text
tcpdump: listening on eth0, link-type EN10MB (Ethernet), snapshot length 96 bytes
```

`-s 96` retains up to 96 bytes per packet. A smaller snapshot reduces stored data and capture volume but can truncate headers or payloads needed for analysis.

<details>
<summary><strong>Read more</strong> — the family of payload switches, and what a snapshot costs you</summary>

The ASCII on the right of `-X` makes recognizable plaintext visible, while the hex is useful for analysing non-printable fields. The related switches are `-x` (hex, no ASCII), `-xx` (adds the link-layer header to the hex) and `-XX` (link-layer hex plus ASCII).

Modern tcpdump commonly uses a much larger default snapshot length, so check the startup line rather than assuming a fixed value. A snapshot that is too small silently truncates the very headers you may need later, and the truncation is not obvious when you first read the capture.

</details>

## TCP flags and connection diagnosis

TCP flags explain connection state. Capture initial SYN packets with a bit-mask filter:

```bash
sudo tcpdump -i eth0 -nn -c 5 'tcp[tcpflags] & (tcp-syn|tcp-ack) == tcp-syn'
```

**Output:**

```text
IP 192.168.56.30.49152 > 192.168.56.20.80: Flags [S], seq 125430, win 64240, length 0
```

The expression matches packets with SYN set and ACK not set, which normally identifies initial connection attempts. Using a bit-mask is more reliable than strict equality because packets can contain additional flags.

Capture TCP reset packets:

```bash
sudo tcpdump -i eth0 -nn -c 5 'tcp[tcpflags] & tcp-rst != 0'
```

**Output:**

```text
IP 192.168.56.20.80 > 192.168.56.30.49152: Flags [R.], seq 1, ack 1, win 0, length 0
```

A RST indicates an abrupt TCP reset. It can result from a closed port, an application aborting a connection, a firewall device, or a network middlebox. Correlate it with service state and logs before deciding on a cause.

Show absolute TCP sequence numbers when needed:

```bash
sudo tcpdump -i eth0 -nn -S -c 3 tcp
```

`-S` prints absolute sequence numbers rather than relative values. This can be useful for detailed protocol troubleshooting but is normally unnecessary for routine capture review.

<details>
<summary><strong>Read more</strong> — reading flags as states, not conclusions</summary>

Each flag is a bit, and a packet can carry several at once: `S` is SYN, `S.` is SYN-ACK, a lone `.` means ACK is set with no other flag, and `R.` is a reset with ACK. Reading the brackets is the fastest way to follow a handshake.

A filter such as `tcp[tcpflags] & tcp-syn != 0` matches any packet with SYN set, while the `(tcp-syn|tcp-ack) == tcp-syn` form is stricter and selects the initial SYN only. Neither proves intent: a SYN flood and a legitimate connection begin with exactly the same packet.

</details>

## Read and write PCAP files

Write raw packets to a PCAP file for later analysis. Avoid recording an unlimited capture:

```bash
sudo tcpdump -i eth0 -c 100 -nn -w lab-web-test.pcap 'tcp port 8080'
```

**Output:**

```text
100 packets captured
```

`-w` writes binary packet records rather than readable lines, so no per-packet output appears. The resulting PCAP can be opened later by tcpdump or a graphical analyzer.

Inspect the file that was created:

```bash
ls -lh lab-web-test.pcap
```

**Output:**

```text
-rw-r----- 1 root root 42K Oct  8 20:40 lab-web-test.pcap
```

`ls -lh` shows the human-readable size, owner and permissions. The permissions and owner should be reviewed, because capture files may include sensitive payloads.

Read the file without needing live traffic:

```bash
tcpdump -nn -r lab-web-test.pcap
```

**Output:**

```text
reading from file lab-web-test.pcap, link-type EN10MB (Ethernet), snapshot length 262144
20:40:01.000010 IP 192.168.56.30.49152 > 192.168.56.20.8080: Flags [S], length 0
```

`-r` reads packets from the saved PCAP. The header reports the link type and snap length recorded in the file.

This repeatable workflow lets analysts share filtered, protected evidence rather than reproducing activity on a live network.

<details>
<summary><strong>Read more</strong> — why capture files are evidence, not scratch files</summary>

Because a PCAP is a byte-for-byte record, it should be handled like any other piece of evidence: keep the original untouched, record a hash, and create any filtered or derived file separately under a name that says so. A `-w` file with the wrong permissions is a leak waiting to be written up.

Reading with `-r` also means the same capture can be re-examined with different filters long after the event, which is exactly what a live-only approach cannot do.

</details>

## Inspect in a graphical analyzer

A PCAP can be transferred securely to an approved analysis workstation and opened in Wireshark. The following ASCII representation replaces a graphical screenshot:

```text
+--------------------------------------------------------------------------------+
| Packet list                                                                    |
| No.   Time        Source            Destination       Protocol  Info          |
| 1     20:40:01    192.168.56.30     192.168.56.20     TCP       49152 -> 8080 |
| 2     20:40:01    192.168.56.20     192.168.56.30     TCP       8080 -> 49152 |
+--------------------------------------------------------------------------------+
| Packet details                                                                 |
| Ethernet II > Internet Protocol Version 4 > Transmission Control Protocol     |
+--------------------------------------------------------------------------------+
| Packet bytes                                                                   |
| 0000  08 00 27 aa bb 20 08 00 27 aa bb 01 08 00 45 00 ...                     |
+--------------------------------------------------------------------------------+
```

The top pane lists packet summaries, the middle pane expands protocol fields, and the bottom pane displays raw bytes. A graphical analyzer is particularly useful for following a TCP stream, applying display filters, and viewing protocol trees.

Preserve the original PCAP and record any derived or filtered file separately.

<details>
<summary><strong>Read more</strong> — when a graphical analyzer beats the command line</summary>

tcpdump is fast and scriptable, but reconstructing a whole TCP stream by eye is tedious. A graphical analyzer reassembles the stream, lets you apply display filters interactively, and shows the protocol tree for a single frame — the same data tcpdump captures, presented differently.

Treat the two as complements: capture with tcpdump, then analyse the protected copy in the graphical tool.

</details>

## Remote capture with safeguards

For an authorized remote host, write a PCAP locally rather than sending a live binary stream to an untrusted destination. Perform a bounded capture on the remote host:

```bash
sudo tcpdump -i eth0 -c 50 -nn -w /var/tmp/lab-capture.pcap 'tcp port 8080'
```

**Output:**

```text
50 packets captured
```

This performs a bounded capture on the remote host, so the file stops growing after 50 packets.

Then limit read access to the file owner and group:

```bash
sudo chmod 0640 /var/tmp/lab-capture.pcap
```

**Output:**

```text
```

`chmod 0640` limits access to the owner and group, and it succeeds silently: as elsewhere in Linux, no output means the change was accepted.

Transfer it only through an approved authenticated channel and delete it according to your evidence-retention policy. Avoid commands that expose passwords in shells, logs, command history, or capture data.

<details>
<summary><strong>Read more</strong> — why not to stream a capture across the wire</summary>

Piping a live capture over the network turns the transport itself into a second, unprotected copy: the binary stream can be intercepted, logged, or replayed, and it may carry exactly the credentials you were trying to observe. Capturing locally and copying the finished file under your own control keeps the data inside one trust boundary at a time.

The `chmod 0640` is deliberate: read and write for the owner, read for the group, and nothing for everyone else, which matches the way a sensitive artifact should sit on disk while it waits to be collected.

</details>

## Promiscuous mode

On many Ethernet interfaces, tcpdump requests promiscuous mode so the capture library can receive frames presented to the interface. In switched networks, this does not automatically expose all traffic from other hosts.

To avoid requesting promiscuous mode:

```bash
sudo tcpdump -i eth0 --no-promiscuous-mode -c 5 -nn
```

The command captures traffic delivered normally to the host interface without changing the interface into promiscuous mode. This is often appropriate when diagnosing the host's own traffic.

<details>
<summary><strong>Read more</strong> — what promiscuous mode does and does not do</summary>

Switch port configuration, VLAN membership, and network hardware determine what frames actually reach the interface, so promiscuous mode alone does not turn a switched network into a hub. To observe other hosts' traffic you generally need a mirror port, a TAP, or a capture point already in the path.

Whether the flag is honoured depends on the operating system and interface driver, which is why availability and behaviour should be verified on the target rather than assumed.

</details>

## Good capture practice

- Start with a precise question, such as “Does the test client receive a DNS reply?” or “Does the local test service complete a TCP handshake?”
- Select the correct interface and verify it with `tcpdump -D` and `ip -brief address`.
- Use narrow BPF filters, a packet count, and an appropriate snap length.
- Prefer `-nn` to avoid name-resolution delays and ambiguous labels.
- Treat `-A`, `-X`, and PCAP files as sensitive because they can expose plaintext content.
- Record time zone, host, interface, capture filter, tool version, and file hashes in the investigation notes.
- Correlate captured packets with service logs, firewall logs, routing configuration, and authorized test activity before drawing conclusions.

<details>
<summary><strong>Read more</strong> — the whole checklist in one idea</summary>

Question first, interface second, filter third, boundaries always. Every item on the list above answers the same underlying need: to be able to say, later, *why* a particular packet was in scope and *what* it does and does not prove.

</details>

## Key lessons

tcpdump is most effective when it collects the minimum traffic needed to answer a defined troubleshooting or security question. Interface selection, BPF filtering, packet limits, and safe evidence handling matter as much as protocol knowledge.

A SYN, an ICMP request, or a plaintext HTTP line is an observation whose cause must be confirmed with context—not a conclusion on its own.

<details>
<summary><strong>Read more</strong> — the one sentence to remember</summary>

Collect the least you can, on a system you are allowed to watch, and treat every packet as an observation to be explained rather than a verdict to be delivered.

</details>
