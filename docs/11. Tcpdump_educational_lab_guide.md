# tcpdump: Practical Packet Capture and Analysis in an Authorized Lab

> **Authorized use only.** Packet capture can expose credentials, session tokens, personal data, internal addresses, and application content. Capture only on networks and systems you own or are explicitly authorized to monitor. Use a lab interface whenever possible, minimize capture scope and retention, and protect capture files as sensitive evidence.

## What tcpdump does

`tcpdump` captures and decodes packets observed by a network interface. It uses the libpcap capture framework and accepts a powerful filter language, commonly called BPF syntax, to limit what is collected. It is valuable for troubleshooting DNS, TCP handshakes, service reachability, routing, and protocol behavior. It is not an intrusion tool; its primary value is careful observation and interpretation.

A typical packet summary contains a timestamp, source endpoint, destination endpoint, protocol information, TCP flags where applicable, and packet length. The exact rendering varies with operating system, tcpdump version, link type, privileges, and traffic present at the time of capture.

## Safe lab architecture

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

Use a test client you control to generate predictable traffic, such as a DNS lookup, HTTP request to a local test server, or a ping inside the lab. Store captures on an encrypted or access-controlled location. Do not send captures containing sensitive traffic through unprotected chat, email, or public file-sharing services.

## Install and identify interfaces

On Debian-based systems, install tcpdump and list capture-capable interfaces:

```bash
sudo apt update
sudo apt install -y tcpdump
sudo tcpdump -D
```

Representative output:

```text
1.eth0 [Up, Running]
2.lo [Up, Running, Loopback]
3.any (Pseudo-device that captures on all interfaces)
```

`apt update` refreshes available package metadata; `apt install` installs the capture utility. `tcpdump -D` lists interfaces tcpdump can use. `eth0` is commonly an Ethernet-style network interface, `lo` is the local loopback interface, and `any` captures across multiple interfaces. Interface names may instead be `ens33`, `enp0s3`, `wlan0`, or another distribution-specific name.

View version and usage information:

```bash
tcpdump --version
tcpdump --help | head -25
```

Example output:

```text
tcpdump version 4.x
libpcap version 1.x
Usage: tcpdump [ -AbdDefhHIJKlLnNOpqStuUvxX# ] ...
```

The version output identifies the tcpdump and libpcap builds. The help summary lists command options. Features and supported protocol decoders can differ across releases, so verify syntax locally with `man tcpdump` when a command behaves differently.

## Start with a bounded capture

Capturing without limits can create very large files or expose more data than necessary. Begin with a count limit:

```bash
sudo tcpdump -i eth0 -c 5
```

Representative output:

```text
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
20:32:14.102431 IP 192.168.56.30.49152 > 192.168.56.20.80: Flags [S], seq 125430, win 64240, length 0
20:32:14.102580 IP 192.168.56.20.80 > 192.168.56.30.49152: Flags [S.], seq 987650, ack 125431, win 65160, length 0
5 packets captured
```

`-i eth0` selects the interface and `-c 5` stops after five packets. The first packet is a TCP SYN from the client ephemeral port `49152` to port 80. The second is SYN-ACK, the server's reply during a TCP three-way handshake. The final line confirms how many packets tcpdump captured. `verbose output suppressed` is normal; it suggests `-v` if more detail is needed.

## Names, numbers, and concise output

By default, tcpdump may resolve addresses or service names. Disable name resolution when you need faster, unambiguous output:

```bash
sudo tcpdump -i eth0 -c 3 -nn
```

Representative output:

```text
20:33:01.100001 IP 192.168.56.30.53530 > 192.168.56.20.53: 1234+ A? example.test. (30)
```

`-n` avoids hostname resolution and the second `n` avoids service-name conversion. Therefore, `53` remains a port number rather than being displayed as `domain`. This reduces side effects and delays caused by DNS lookups during analysis.

Use quick mode when a short flow summary is enough:

```bash
sudo tcpdump -i eth0 -c 3 -q -nn
```

Example output:

```text
IP 192.168.56.30.49152 > 192.168.56.20.80: tcp 0
IP 192.168.56.20.80 > 192.168.56.30.49152: tcp 0
```

`-q` prints less protocol detail. It is useful for a high-level view but can hide fields needed for diagnosis, such as TCP flags and sequence-related details.

## Filtering by protocol, port, host, and network

A BPF filter is placed after tcpdump options. Capture ten ICMP packets on the selected interface:

```bash
sudo tcpdump -i eth0 -c 10 -nn icmp
```

Representative output:

```text
20:34:10.000200 IP 192.168.56.30 > 192.168.56.20: ICMP echo request, id 4512, seq 1, length 64
20:34:10.000611 IP 192.168.56.20 > 192.168.56.30: ICMP echo reply, id 4512, seq 1, length 64
```

The first line is an ICMP echo request, often produced by `ping`; the second line is the matching reply. The identifier and sequence number help pair request and reply packets. A request without a reply can indicate a firewall rule, a routing problem, host unavailability, or packet loss; it is not automatically evidence of malicious activity.

Capture traffic involving a web port:

```bash
sudo tcpdump -i eth0 -c 10 -nn 'port 80'
```

Example output:

```text
IP 192.168.56.30.49152 > 192.168.56.20.80: Flags [S], length 0
IP 192.168.56.20.80 > 192.168.56.30.49152: Flags [S.], length 0
```

The filter `port 80` matches packets whose source or destination port is 80. To restrict to a server-side destination port, use `dst port 80`; to observe traffic originating from port 80, use `src port 80`.

```bash
sudo tcpdump -i eth0 -c 10 -nn host 192.168.56.30
sudo tcpdump -i eth0 -c 10 -nn net 192.168.56.0/24
```

The `host` filter matches either direction involving one address. The `net` filter matches traffic to or from the stated network. Narrow filters reduce unrelated collection and improve readability.

A port range can be useful when testing a group of related services:

```bash
sudo tcpdump -i eth0 -c 10 -nn 'portrange 8000-8100'
```

This matches packets whose source or destination port is between 8000 and 8100 inclusive. Use it only where the range is relevant; broad ranges increase noise.

## Direction and link-layer information

On platforms that support capture direction, restrict output to inbound or outbound packets:

```bash
sudo tcpdump -i eth0 -c 5 -nn -Q in
sudo tcpdump -i eth0 -c 5 -nn -Q out
```

`-Q in` displays packets received by the interface, while `-Q out` displays packets sent by it. Support is platform and driver dependent. If tcpdump reports that direction is unsupported, use a host, address, or port filter instead.

Include Ethernet-level headers:

```bash
sudo tcpdump -i eth0 -c 2 -nn -e
```

Representative output:

```text
20:35:44.200001 08:00:27:aa:bb:01 > 08:00:27:aa:bb:20, ethertype IPv4 (0x0800), length 74: 192.168.56.30.49152 > 192.168.56.20.80: Flags [S], length 0
```

`-e` prints link-layer information, including source and destination MAC addresses on Ethernet captures. The `ethertype IPv4 (0x0800)` identifies the network-layer protocol carried in the frame. MAC addresses are normally meaningful only on the local Layer-2 segment.

## Verbosity and timestamps

Increase decoding detail only when useful:

```bash
sudo tcpdump -i eth0 -c 2 -nn -v
sudo tcpdump -i eth0 -c 2 -nn -vv
sudo tcpdump -i eth0 -c 2 -nn -vvv
```

A more verbose packet may look like:

```text
IP (tos 0x0, ttl 64, id 22301, offset 0, flags [DF], proto TCP (6), length 60)
    192.168.56.30.49152 > 192.168.56.20.80: Flags [S], cksum 0xabcd (correct), seq 125430, win 64240, options [mss 1460,sackOK,TS val 100 ecr 0,nop,wscale 7], length 0
```

The IPv4 section includes type of service, time-to-live, packet ID, fragmentation information, protocol number, and total length. The TCP section includes flags, checksum status, sequence number, receive window, and negotiation options. Extra verbosity can make troubleshooting easier but creates more output to review.

Timestamp formats change presentation, not packet contents:

```bash
sudo tcpdump -i eth0 -c 2 -nn -t
sudo tcpdump -i eth0 -c 2 -nn -tt
sudo tcpdump -i eth0 -c 2 -nn -ttt
sudo tcpdump -i eth0 -c 2 -nn -tttt
```

`-t` omits timestamps. `-tt` prints an absolute Unix-style timestamp. `-ttt` prints the time difference from the previous packet, which is useful for latency and timing analysis. `-tttt` displays a date and time in a human-readable format. Exact layout varies by version.

## Payload rendering and snap length

Payload display can reveal plaintext application content and therefore requires particular care. In a lab, inspect a small number of packets in ASCII:

```bash
sudo tcpdump -i eth0 -c 2 -nn -A 'tcp port 8080'
```

Representative output:

```text
GET /health HTTP/1.1
Host: 192.168.56.20:8080
User-Agent: lab-client/1.0
```

`-A` prints packet payload data as ASCII where possible. The shown data is an HTTP request: `GET` requests the `/health` path, `Host` identifies the intended virtual host and port, and `User-Agent` identifies the client software string. Do not use this technique to collect passwords, cookies, authorization headers, or user content.

View packet bytes in hexadecimal and ASCII:

```bash
sudo tcpdump -i eth0 -c 1 -nn -X 'tcp port 8080'
```

Example output:

```text
0x0000:  4500 0054 1a2b 4000 4006 ...
0x0030:  4745 5420 2f68 6561 6c74 6820 4854 5450  GET /health HTTP
```

`-X` prints a hexadecimal representation alongside ASCII decoding. The left-most values are byte offsets. The ASCII on the right makes recognizable plaintext visible, while the hex is useful for analyzing non-printable fields. `-x` prints packet data in hexadecimal without the ASCII column; `-xx` includes the link-layer header in the hex output; `-XX` combines link-layer hex with ASCII.

The snapshot length limits how many bytes tcpdump retains from each packet. For a deliberately limited capture:

```bash
sudo tcpdump -i eth0 -c 2 -nn -s 96 'tcp port 8080'
```

Representative output:

```text
tcpdump: listening on eth0, link-type EN10MB (Ethernet), snapshot length 96 bytes
```

`-s 96` retains up to 96 bytes per packet. A smaller snapshot reduces stored data and capture volume but can truncate headers or payloads needed for analysis. Modern tcpdump commonly uses a much larger default snapshot length; check the startup line rather than assuming a fixed value.

## TCP flags and connection diagnosis

TCP flags explain connection state. Capture initial SYN packets with a bit-mask filter:

```bash
sudo tcpdump -i eth0 -nn -c 5 'tcp[tcpflags] & (tcp-syn|tcp-ack) == tcp-syn'
```

Representative output:

```text
IP 192.168.56.30.49152 > 192.168.56.20.80: Flags [S], seq 125430, win 64240, length 0
```

The expression matches packets with SYN set and ACK not set, which normally identifies initial connection attempts. Using a bit-mask is more reliable than strict equality because packets can contain additional flags.

Capture TCP reset packets:

```bash
sudo tcpdump -i eth0 -nn -c 5 'tcp[tcpflags] & tcp-rst != 0'
```

Example output:

```text
IP 192.168.56.20.80 > 192.168.56.30.49152: Flags [R.], seq 1, ack 1, win 0, length 0
```

A RST indicates an abrupt TCP reset. It can result from a closed port, an application aborting a connection, a firewall device, or a network middlebox. Correlate it with service state and logs before deciding on a cause.

Show absolute TCP sequence numbers when needed:

```bash
sudo tcpdump -i eth0 -nn -S -c 3 tcp
```

`-S` prints absolute sequence numbers rather than relative values. This can be useful for detailed protocol troubleshooting but is normally unnecessary for routine capture review.

## Read and write PCAP files

Write raw packets to a PCAP file for later analysis. Avoid recording an unlimited capture:

```bash
sudo tcpdump -i eth0 -c 100 -nn -w lab-web-test.pcap 'tcp port 8080'
ls -lh lab-web-test.pcap
```

Representative output:

```text
100 packets captured
-rw-r----- 1 root root 42K Oct  8 20:40 lab-web-test.pcap
```

`-w` writes binary packet records rather than readable lines. The resulting PCAP can be opened by tcpdump or a graphical analyzer. The permissions and owner shown by `ls -lh` should be reviewed because capture files may include sensitive payloads.

Read the file without needing live traffic:

```bash
tcpdump -nn -r lab-web-test.pcap
```

Example output:

```text
reading from file lab-web-test.pcap, link-type EN10MB (Ethernet), snapshot length 262144
20:40:01.000010 IP 192.168.56.30.49152 > 192.168.56.20.8080: Flags [S], length 0
```

`-r` reads packets from the saved PCAP. The header reports the link type and snap length recorded in the file. This repeatable workflow lets analysts share filtered, protected evidence rather than reproducing activity on a live network.

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

The top pane lists packet summaries, the middle pane expands protocol fields, and the bottom pane displays raw bytes. A graphical analyzer is particularly useful for following a TCP stream, applying display filters, and viewing protocol trees. Preserve the original PCAP and record any derived or filtered file separately.

## Remote capture with safeguards

For an authorized remote host, write a PCAP locally rather than sending a live binary stream to an untrusted destination:

```bash
sudo tcpdump -i eth0 -c 50 -nn -w /var/tmp/lab-capture.pcap 'tcp port 8080'
sudo chmod 0640 /var/tmp/lab-capture.pcap
```

The first command performs a bounded capture on the remote host. The second limits read access to the file owner and group. Transfer it only through an approved authenticated channel and delete it according to your evidence-retention policy. Avoid commands that expose passwords in shells, logs, command history, or capture data.

## Promiscuous mode

On many Ethernet interfaces, tcpdump requests promiscuous mode so the capture library can receive frames presented to the interface. In switched networks, this does not automatically expose all traffic from other hosts; switch port configuration, VLAN membership, and network hardware determine what frames reach the interface.

To avoid requesting promiscuous mode:

```bash
sudo tcpdump -i eth0 --no-promiscuous-mode -c 5 -nn
```

The command captures traffic delivered normally to the host interface without changing the interface into promiscuous mode. This is often appropriate when diagnosing the host's own traffic. Availability and behavior depend on the operating system and interface driver.

## Good capture practice

- Start with a precise question, such as “Does the test client receive a DNS reply?” or “Does the local test service complete a TCP handshake?”
- Select the correct interface and verify it with `tcpdump -D` and `ip -brief address`.
- Use narrow BPF filters, a packet count, and an appropriate snap length.
- Prefer `-nn` to avoid name-resolution delays and ambiguous labels.
- Treat `-A`, `-X`, and PCAP files as sensitive because they can expose plaintext content.
- Record time zone, host, interface, capture filter, tool version, and file hashes in the investigation notes.
- Correlate captured packets with service logs, firewall logs, routing configuration, and authorized test activity before drawing conclusions.

## Key lessons

tcpdump is most effective when it collects the minimum traffic needed to answer a defined troubleshooting or security question. Interface selection, BPF filtering, packet limits, and safe evidence handling matter as much as protocol knowledge. A SYN, an ICMP request, or a plaintext HTTP line is an observation whose cause must be confirmed with context—not a conclusion on its own.
