# Linux for Beginners (Part 2): Networks, Processes and Environment Variables

*This is Part 2 of a three-part guide. It continues [Linux for Beginners (Part 1): The Shell, Files, Text, Packages and Permissions](01-linux-fundamentals-part-1.md), and it assumes you are comfortable with the material there — moving around the file system, reading a long listing, editing text from the command line, using the package manager and reading permission strings. Its continuation — scripting, scheduling and services — is in [Linux for Beginners (Part 3)](03-linux-fundamentals-part-3.md).*

## Introduction

Part 1 covered the layer of Linux that everything else is built on: the shell, the file system, text, packages and permissions. This part moves one level deeper, into the things that decide how a machine actually behaves: its **network**, its **processes** and its **environment variables**.

Every section keeps the same shape as Part 1: a transcript of the commands that were typed and the output they produced, followed by a short explanation of what happened and why it matters. Anything longer — extra flags, pitfalls, background theory — is tucked into a collapsible box, so you can read the main path first and open the detail only when you want it.

<details>
<summary><strong>Read more</strong> — the three topics this part covers</summary>

First, the **network**: the interfaces a machine presents, the addresses and hardware identifiers it carries, the server that hands its addresses out, and the name service it consults before it can connect to anything. Second, **processes**: how to see everything that is running, read what it is consuming, decide which work deserves the machine's attention, stop what has gone wrong, and arrange for jobs to run later. Third, **environment variables**: the quiet configuration layer that decides which program runs when you type a bare name, which resolver answers, and what a newly launched shell already knows.

Where a traditional tool has a modern replacement — `ifconfig` against `ip`, `netstat` against `ss`, `top` against `htop` — both are shown, because older documentation, interview questions and real machines mix them freely.

</details>

> **A word of caution before you start.** Two of the three topics here can lock you out of the machine you are working on. Changing an address over SSH ends your session, and killing the wrong process takes down a service somebody is using. Work on a virtual machine, take a snapshot first, and keep a second console open while you experiment with the network.

### How to read the transcripts

The conventions are the same as in Part 1, and they are worth restating because the prompts in this part change from one moment to the next:

* `root@kali:~#` is a shell running as the all-powerful root account, and the trailing `#` is a warning worth heeding. `student@kali:~$` is an ordinary user, and the `$` marks the difference.
* Everything on a prompt line after the prompt is what was typed; every line underneath, up to the next prompt, is output. Nothing runs until Enter is pressed.
* Text inside angle brackets, such as `dig <name>`, is a placeholder you replace with a real value.
* Output was captured on Debian-family systems and is representative rather than byte-identical: addresses, MAC addresses, PIDs, package versions and timestamps will differ on your own machine. What matters is the shape of the output.

### Table of contents

1. [Managing networks](#1-managing-networks)
2. [Process management](#2-process-management)
3. [User environment variables](#3-user-environment-variables)
4. [Quick reference cheat sheet](#4-quick-reference-cheat-sheet)
5. [Practice exercises](#5-practice-exercises)

---

## 1. Managing networks

> **Concept.** A host's networking is a stack of layers — the link, internet, transport and application layers of the TCP/IP model that the seven-layer OSI model refines — and each layer wraps the one above in its own header, a process called *encapsulation* [RFC 1122]. Addressing lives at several layers at once: an IPv4 address is split into a network and a host part by its prefix length or netmask, so CIDR notation such as `192.168.1.0/24` decides whether a destination is local or must be handed to a router [RFC 791, RFC 4632], while a MAC address identifies a card on the local segment and the loopback interface (`127.0.0.1`, `::1`) names the host itself so that "local" traffic never leaves the machine. Neighbours, dynamic addresses and names are each answered by a dedicated protocol — ARP resolves an address to a MAC on the local link [RFC 826], DHCP hands out leases [RFC 2131], and DNS resolves a name to an address and publishes a zone's records [RFC 1034/1035]. When a destination is not local, the kernel consults its routing table and forwards the packet to the default gateway, which is why reading `ip route` is the first step in almost every connectivity problem.

Networking is a crucial topic for anyone heading towards security work, because a great deal of what you will eventually be asked to test lives on the network rather than on a single machine.

Even for everyday administration you cannot get far without knowing how to look at an interface, read the address it holds, and change any of it deliberately. This section works through the tools that do exactly that, in the order you would reach for them: inspect an interface, inspect a wireless interface, change an address, spoof a hardware address, ask a DHCP server for a lease, interrogate DNS, change which resolver you use, and override a name locally.

<details>
<summary><strong>Read more</strong> — why some of these commands look old-fashioned</summary>

The traditional tools are `ifconfig`, `iwconfig`, `route` and `netstat`, and they come from a package called `net-tools` that many modern distributions no longer install by default. They have been superseded by the `ip` command (from the `iproute2` suite) and by `ss`. Every traditional command in this section is therefore accompanied by its modern equivalent, and the habit worth forming is to read both: legacy documentation, older tutorials and interview questions are full of `ifconfig`, while every current distribution ships `ip`.

</details>

### 1.1 `ifconfig` — analysing the network interfaces

The `ifconfig` command is the most basic tool for interacting with active network interfaces. Run with no arguments it prints every interface the kernel knows about, which on a typical lab machine means a physical or virtual Ethernet adapter plus the loopback interface.

```bash
ifconfig
```

**Output:**

```text
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

Two interfaces are listed. The first line of each block names the interface and its flags, the second carries the addressing, the third the link-layer identity, and the rest are statistics — and reading them properly is the skill this section teaches.

The modern equivalent of this command is `ip addr` (often typed `ip a`), which shows the same information in a different layout. Passing `-br` asks for *brief* output — one line per interface — which is ideal when you only care about the address:

```bash
ip -br addr
```

**Output:**

```text
lo               UNKNOWN        127.0.0.1/8 ::1/128 
eth0             UP             192.168.1.17/24 fe80::20c:29ff:fe1b:2c3d/64
```

This compact view confirms that `eth0` is `UP` and holds the IPv4 address `192.168.1.17/24`. To see where traffic goes when it leaves the local subnet, run `ip route` to print the kernel's routing table:

```bash
ip route
```

**Output:**

```text
default via 192.168.1.1 dev eth0 proto dhcp metric 100 
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.17 metric 100 
```

The key line is `default via 192.168.1.1 dev eth0`, which names the default gateway: the router that packets are sent to when their destination is not on the local `192.168.1.0/24` network.

The kernel also keeps a table of *neighbours*: the link-layer addresses it has learned for hosts on the local segment. That table is the ARP cache, and `ip neigh` prints it (the legacy equivalent is `arp -n` from `net-tools`):

```bash
ip neigh
```

**Output:**

```text
192.168.1.1 dev eth0 lladdr c8:3a:35:1f:2a:40 REACHABLE
192.168.1.9 dev eth0 lladdr 00:0c:29:aa:bb:cc STALE
192.168.1.20 dev eth0 lladdr 00:50:56:c0:00:08 DELAY
```

Reading the columns in order: the first field is the neighbour's IP address, `dev eth0` is the interface it was seen on, and `lladdr` is the MAC address the kernel resolved for it through ARP — the protocol defined in [RFC 826](https://www.rfc-editor.org/rfc/rfc826) that maps a known IP address to an unknown hardware address on the local link. The trailing word is the entry's state: `REACHABLE` means it was confirmed recently, `STALE` means the value is still usable but unverified, and `DELAY` means the kernel is waiting a moment before re-checking it. A neighbour entry is what makes "the gateway is up" or "that host is on this segment" concrete, and it is the table that an ARP-spoofing attack poisons.

Knowing the gateway is not the same as knowing the path beyond it. To see every router a packet passes through on its way to a destination, use `traceroute`, which sends probes with steadily increasing TTL values and reads the ICMP "time exceeded" replies that come back:

```bash
traceroute example.com
```

**Output:**

```text
traceroute to example.com (93.184.216.34), 30 hops max, 60 byte packets
 1  _gateway (192.168.1.1)  0.482 ms  0.511 ms  0.489 ms
 2  10.10.0.1 (10.10.0.1)  3.204 ms  3.187 ms  3.411 ms
 3  ae-1.edge.lab (203.0.113.9)  8.771 ms  8.690 ms  9.013 ms
 4  93.184.216.34 (93.184.216.34)  24.105 ms  24.228 ms  24.060 ms
```

Each line is one hop: its number, the name and address of the router that answered, and three round-trip times in milliseconds, one per probe. A line of `* * *` means a router that declined to answer, which is common in the middle of the internet; the cases worth attention are a path that never leaves your own network (a routing problem) and one that reaches the destination by a surprising route.

`mtr` — "my traceroute" — is the same probe measured repeatedly and summarised. In report mode it prints one line per hop with packet-loss and latency statistics, which is easier to read and to paste into a report than a single `traceroute` run:

```bash
mtr --report --report-cycles 5 example.com
```

**Output:**

```text
HOST: kali                       Loss%   Snt   Last   Avg  Best  Wrst StDev
  1.|-- _gateway                 0.0%     5    0.5   0.5   0.4   0.6   0.1
  2.|-- 10.10.0.1                0.0%     5    3.2   3.3   3.1   3.6   0.2
  3.|-- ae-1.edge.lab            0.0%     5    8.8   8.9   8.7   9.3   0.2
  4.|-- 93.184.216.34            0.0%     5   24.1  24.2  24.0  24.5   0.2
```

The `Loss%` column is what earns `mtr` its place: a hop that drops a few percent of probes while everything after it is clean is usually rate-limiting rather than a fault, whereas loss that persists to the final hop is a real problem. `Snt` is how many probes were sent, and the remaining columns summarise the round-trip time across them.

Finally, the same machine that *sends* packets is usually also *offering* services, and `ss -tulpn` lists the sockets it is listening on: TCP and UDP sockets (`-tu`), listening only (`-l`), with numeric ports (`-n`) and the owning process shown (`-p`):

```bash
ss -tulpn
```

**Output:**

```text
Netid State  Recv-Q Send-Q Local Address:Port  Peer Address:Port Process
udp   UNCONN 0      0          127.0.0.54:53         0.0.0.0:*     users:(("systemd-resolve",pid=712,fd=13))
tcp   LISTEN 0      128          0.0.0.0:22           0.0.0.0:*     users:(("sshd",pid=812,fd=3))
tcp   LISTEN 0      511          0.0.0.0:80           0.0.0.0:*     users:(("apache2",pid=3312,fd=4))
tcp   LISTEN 0      4096       127.0.0.53:53           0.0.0.0:*     users:(("systemd-resolve",pid=712,fd=14))
tcp   LISTEN 0      128             [::]:22              [::]:*     users:(("sshd",pid=812,fd=4))
```

The flags are the whole command: `-t` for TCP, `-u` for UDP, `-l` for listening sockets only, `-p` to show the owning process, and `-n` to print numbers instead of names. `State` is `LISTEN` for TCP and `UNCONN` for UDP; `Local Address:Port` shows what is bound, so `0.0.0.0:22` means "every interface on this host, port 22" while `127.0.0.53:53` means the loopback address only; and `Process` names the daemon that owns the socket, which is why the command needs root to show it. That last column is the single most useful one for security work: it is the shortest path from "a port is open" to "this program opened it".

<details>
<summary><strong>Read more</strong> — reading the <code>ifconfig</code> output line by line</summary>

The first line of each block names the interface and its flags: `UP` means the interface is administratively enabled, `RUNNING` means the driver reports a carrier (a cable is plugged in, or the virtual switch is connected), `BROADCAST` and `MULTICAST` describe the traffic the interface will accept, and `mtu 1500` is the maximum transmission unit — the largest payload the link will carry in one frame, which matters when you are diagnosing fragmentation problems. The loopback interface has an `mtu` of 65536 because it is not a real wire; nothing has to fit into an Ethernet frame when the packets never leave the machine.

The second line carries the addressing. `inet 192.168.1.17 netmask 255.255.255.0 broadcast 192.168.1.255` is IPv4: the address, the mask that defines which part of the address is the network and which part the host, and the address that reaches every machine on the local segment. From those two numbers you can derive the rest of the subnet mentally, which is a skill worth practising: a `/24` mask (255.255.255.0) with the address 192.168.1.17 means the network is 192.168.1.0, the usable host range is .1 to .254, and the broadcast is .255. The `inet6` line is the automatically generated IPv6 link-local address, recognisable by its `fe80::` prefix, which is present even on networks where nobody has configured IPv6 — one of the reasons an IPv6-blind firewall is a real problem.

The third line is the link layer. The `ether` field is the MAC address, the hardware identifier burned into the network card (or, on a virtual machine, assigned by the hypervisor), and `00:0c:29` identifies this one as a VMware virtual adapter. MAC addresses are discussed in detail in section 1.4, because they can be changed.

The rest of the block is statistics. `RX` and `TX` count packets and bytes received and transmitted since the interface came up, and the `errors`, `dropped`, `overruns`, `carrier` and `collisions` counters track things that went wrong — the first place to look when a link is flaky. A steadily climbing `errors` or `dropped` count on a physical interface usually means a bad cable, a failing port or a duplex mismatch. All of these counters reset when the interface is brought down and up again.

Knowing the gateway is not optional knowledge: the default route is the thing that makes the difference between a machine that can reach the internet and one that can only talk to its neighbours even though `ifconfig` looks perfectly healthy.

</details>

### 1.2 `iwconfig` — checking wireless network devices

If the machine has a wireless adapter, `iwconfig` is the traditional tool for reading its radio-specific settings: the network it is associated with, the access point it is talking to, the frequency, the transmission rate and the signal quality.

```bash
iwconfig
```

**Output:**

```text
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

On a machine with no wireless hardware — the common case on a virtual machine — every interface reports `no wireless extensions`, and that is the expected output rather than an error. On a machine that does have a wireless card, three fields are worth understanding.

<details>
<summary><strong>Read more</strong> — the wireless fields that matter, and the modern replacement</summary>

`ESSID:"LAB-AP"` is the name of the wireless network the card has joined, and `Mode:Managed` means it is behaving as a normal client. The `Access Point` field is the MAC address of the wireless router the card is associated with, which is useful for physical-location work and for detecting a rogue access point that is impersonating your network. `Link Quality=58/70  Signal level=-52 dBm` is the radio health: signal strength in dBm where values closer to zero are better, so −52 dBm is a strong, comfortable connection and anything worse than about −75 dBm will start to produce dropped packets.

The security-relevant fields are `Encryption key:off`, which tells you whether the link is using WEP, WPA or nothing at all, and the counters at the bottom. `Rx invalid crypt` counts frames that failed their integrity check, which on an open network is simply noise, but on an encrypted one can indicate interference or an attack in progress.

The `iw` command is the modern replacement (`iw dev wlan0 link` prints the association details), and `nmcli device wifi list` is the higher-level tool if NetworkManager is managing the radio.

</details>

### 1.3 Changing an IP address

An interface's address can be set by hand, and doing so is a one-line command: `ifconfig`, then the interface to change, then the address you want it to have. Here the Ethernet interface is given the address `192.168.1.13`.

```bash
ifconfig eth0 192.168.1.13
```

The command succeeds silently because the kernel applies local address changes immediately without confirmation. We now run `ifconfig eth0` to inspect the interface and verify that the new address is active:

```bash
ifconfig eth0
```

**Output:**

```text
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.13  netmask 255.255.255.0  broadcast 192.168.1.255
        ether 00:0c:29:1b:2c:3d  txqueuelen 1000  (Ethernet)
        RX packets 10430  bytes 9012611 (8.5 MiB)
        TX packets 5121  bytes 774311 (756.1 KiB)
```

The `inet` field now shows `192.168.1.13` in place of `192.168.1.17`, with the default `/24` netmask applied automatically. We can confirm the exact same state using the modern `ip` tool restricted to `eth0`:

```bash
ip -br addr show eth0
```

**Output:**

```text
eth0             UP             192.168.1.13/24 fe80::20c:29ff:fe1b:2c3d/64
```

Both tools report `192.168.1.13/24` on `eth0`. This works immediately because the kernel does not ask permission from anything — an address is local state, so assigning one takes effect the moment you type it, with no DHCP lease and no notification to the router.

<details>
<summary><strong>Read more</strong> — why a hand-set address is temporary, and how to avoid cutting yourself off</summary>

The change is **temporary**: nothing has been written to a configuration file, so a reboot — or a restart of the networking service — returns the interface to whatever its configuration says. If the machine is managed by NetworkManager or systemd-networkd, those services may overwrite your change within seconds or on the next reconnect. A second consequence is that `ifconfig eth0 <address>` does not touch the routing table, so the default route may still point out of the interface with a source address that no longer exists, and connectivity can break in a confusing way: the interface looks correctly configured yet nothing routes. The safe pattern, and the one used on machines you administer permanently, is to edit the configuration (a Netplan YAML file on Ubuntu, a connection profile under `/etc/NetworkManager/system-connections/`, or `/etc/network/interfaces` on Debian systems without NetworkManager) and then reload the networking service so that addresses, routes and DNS are all recalculated together.

If you are working over SSH when you change an address, be aware that you are almost certainly cutting the branch you are sitting on: the moment the source address of the connection changes, the session dies. The standard safety measures are to work from a console, to add the new address *alongside* the old one with `ip addr add 192.168.1.13/24 dev eth0` (the modern command, which adds rather than replaces), or to schedule a rollback with `at` — a technique we will meet in section 2.9.

</details>

### 1.4 Spoofing a MAC address

The MAC address is not only an identifier; on many networks it *is* the access control. Wireless access points with MAC filtering, captive portals that remember devices, DHCP servers that hand out fixed leases by hardware address, and network access control systems that quarantine unknown machines all make decisions on the strength of the MAC address they see. Because that value is read from the frame's headers rather than from any hardware register at receive time, a modern network card will let the operating system present whatever value you ask for. Spoofing is therefore trivial, and it neutralises every one of those controls.

Changing it takes three commands, because most drivers will not accept a new hardware address while the interface is carrying traffic. First, bring the interface **down** so it stops transmitting:

```bash
ifconfig eth0 down
```

The interface is now administratively disabled and silent. Next, use the `hw ether` keyword ("hardware address of type Ethernet") to assign the new MAC address:

```bash
ifconfig eth0 hw ether 00:11:22:33:44:55
```

Again, a successful change produces no output. Now bring the interface back **up** so it can carry traffic under its new identity:

```bash
ifconfig eth0 up
```

With the link active again, inspect `eth0` to verify that the spoofed hardware address is in place:

```bash
ifconfig eth0
```

**Output:**

```text
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.17  netmask 255.255.255.0  broadcast 192.168.1.255
        ether 00:11:22:33:44:55  txqueuelen 1000  (Ethernet)
        RX packets 2  bytes 168 (168.0 B)
        TX packets 3  bytes 252 (252.0 B)
```

The verification shows `ether 00:11:22:33:44:55` in place of the original VMware address, and note what else changed: the `RX` and `TX` counters have reset to almost nothing, because downing the interface clears them and starting it again begins fresh counts.

<details>
<summary><strong>Read more</strong> — choosing a valid address, and why spoofing is not invisibility</summary>

The interface came back with its configured IPv4 address and its routes, because those are applied by the network manager when the link comes up — a manual configuration inside a live system behaves differently from one made while the interface is down.

The modern equivalent is `ip link set dev eth0 address 00:11:22:33:44:55`, with `ip link set eth0 down` and `up` around it. Not every driver accepts a spoofed address, and not every address is accepted: a multicast address (the least significant bit of the first octet set), or a value whose first octet is odd in the wrong place, may be rejected with `SIOCSIFHWADDR: Cannot assign requested address`. Choose a unicast address — the second-least-significant bit of the first octet must be clear — and, for anything that needs to look legitimate, an address with a real vendor prefix.

Changing your MAC is not invisibility. On a switched network the frame is delivered by MAC, but the switch *records* which port each address appears on, so spoofing an address that another device is already using creates a flapping-port condition that switch management software will log and many will block. Spanning-tree protection features such as port security and DHCP snooping, and any decent network access control deployment, are built precisely to catch this. And if your goal is to *defeat* MAC-based access control in an assessment, the honest finding to write up is not "I spoofed a MAC" but "the network treats a spoofable identifier as an authentication factor, which constitutes no authentication at all".

</details>

### 1.5 Getting a fresh address from DHCP with `dhclient`

Manual addressing is useful for a lab, but most machines do not stand still: they ask the local DHCP server for an address every time they connect. DHCP — the Dynamic Host Configuration Protocol — hands out addresses, and usually a subnet mask, a gateway and DNS servers along with them, from a pool that the administrator controls. On a client machine the request is made by the DHCP client daemon, `dhclient`:

```bash
dhclient eth0
```

Running `dhclient` with an interface name negotiates a lease in the background and returns to the prompt without printing the new address. To see the address that the DHCP server assigned, inspect `eth0` with `ifconfig`:

```bash
ifconfig eth0
```

**Output:**

```text
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.17  netmask 255.255.255.0  broadcast 192.168.1.255
        ether 00:11:22:33:44:55  txqueuelen 1000  (Ethernet)
        RX packets 421  bytes 38112 (37.2 KiB)
        TX packets 388  bytes 74520 (72.8 KiB)
```

The address has now returned to `192.168.1.17` — the value the server had allocated to this interface before we typed `192.168.1.13` over the top of it. The machine is back to a server-managed state, and the manually configured address is gone from the interface.

Most of the useful detail is not printed on the console but written to the lease file, which records exactly what the server offered and when it expires:

```bash
grep -A3 "lease" /var/lib/dhcp/dhclient.eth0.leases | head -20
```

**Output:**

```text
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

That block is the client's record of the conversation: the address it holds, the mask, the gateway (`option routers`), the DNS servers it was told to use, the length of the lease in seconds, and the times at which it will try to renew, rebind and eventually lose the address.

Those three times define the DHCP lifecycle, and they explain a common puzzle: a lease is not permanent, so a machine that has been off for a long time may return to a different address.

<details>
<summary><strong>Read more</strong> — releasing a lease, and the modern replacement</summary>

Because the client renews at half the lease time, an address that "suddenly changed" usually means the server had already given it to somebody else.

`dhclient -r eth0` *releases* the lease, telling the server that the address is no longer needed so it can be returned to the pool — useful when you want to force a completely fresh configuration. `dhclient -v eth0` runs the client verbosely and prints the four-step handshake as it happens (DISCOVER, OFFER, REQUEST, ACK), which is the clearest way to understand the protocol and the fastest way to prove that a DHCP server is reachable. On modern desktop systems all of this is managed by NetworkManager or systemd-networkd, and reaching for `dhclient` by hand on such a machine leads to the two systems fighting over the interface; `nmcli con up <connection>` and `networkctl reconfigure <link>` are the polite ways to ask.

</details>

### 1.6 Examining DNS with `dig`

DNS is the service that translates a domain name such as `example.com` into the IP address a packet can actually be sent to, and it does far more than that: it publishes which servers accept mail for a domain, which name servers are authoritative for the zone, which hosts exist, and where services such as web and VPN endpoints can be found. For anyone assessing a target, DNS is reconnaissance gold, and `dig` — the Domain Information Groper — is the tool you use to read it.

Run with a single argument, `dig` performs a lookup for an A record (an IPv4 address) and prints the whole answer, including the parts that are normally hidden:

```bash
dig example.com
```

**Output:**

```text

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

Reading from the top: the `HEADER` line reports the operation (`opcode: QUERY`), the result code (`status: NOERROR`, where NXDOMAIN means the name does not exist and SERVFAIL means the resolver itself failed), and the flags — of which `qr` (this is a response), `rd` (recursion desired) and `ra` (recursion available) are the ones you will see most. The `ANSWER SECTION` holds the answer: the name, the TTL in seconds (3600, meaning resolvers may cache this for an hour), the class (`IN` for internet), the record type (`A`) and the data (`93.184.216.34`).

The last four lines are the ones beginners overlook and experts read first. `SERVER: 192.168.1.1#53(192.168.1.1) (UDP)` names the resolver that actually answered *you*, and noticing when this changes is how you confirm that your resolver configuration took effect.

Different record types answer different questions, and `dig` asks for them by adding the type to the command. The `mx` type lists the mail exchangers for the domain, which is step one in mapping an organisation's mail infrastructure:

```bash
dig example.com mx +noall +answer
```

**Output:**

```text
example.com.		3600	IN	MX	0 .
```

That answer is a **null MX**: the single record `0 .` is a deliberate, standards-defined way of saying "this domain does not accept mail at all". Since the reserved `example.com` domain deliberately accepts no mail, here is the shape of a mail-accepting answer taken from a lab zone in the reserved `.example` top-level domain:

```bash
dig intranet.example mx +noall +answer
```

**Output:**

```text
intranet.example.	300	IN	MX	10 mail1.intranet.example.
intranet.example.	300	IN	MX	20 mail2.intranet.example.
```

Here two mail servers are published, `mail1` preferred over `mail2` because the lower number wins, which is how a domain load-balances inbound mail and provides failover.

The `ns` type names the authoritative name servers for the zone, which are the machines that hold the definitive copies of every record in it:

```bash
dig example.com ns +noall +answer
```

**Output:**

```text
example.com.		172800	IN	NS	a.iana-servers.net.
example.com.		172800	IN	NS	b.iana-servers.net.
```

For an assessment these are the servers you would query directly (`dig @a.iana-servers.net example.com AXFR`) on the off-chance that they permit a zone transfer.

A few shortcuts make `dig` pleasant to use in scripts and in a hurry: `+short` for the answer data only, `+noall +answer` to drop the header and status blocks, `@1.1.1.1` to query a specific resolver, `-x` for a reverse lookup, and `+trace` to follow the resolution step by step from the root servers down.

The record types you will reach for most often are `A` (IPv4), `AAAA` (IPv6), `MX` (mail exchangers), `NS` (name servers), `CNAME` (an alias), `PTR` (reverse) and `TXT` (free-form text, which is where mail policies and, often, useful leakage live). Asking for `TXT` is a single added word:

```bash
dig example.com txt +short
```

**Output:**

```text
"v=spf1 -all"
```

That one `TXT` record is a policy statement in a single string: `v=spf1` announces a Sender Policy Framework record and `-all` says "no host is authorised to send mail for this domain", the mail-side twin of the null MX seen above.

The `nslookup` tool predates `dig` and is still found everywhere, including on systems where `dig` is not installed. Given a name it reports the responding server and then the answer, using the same query types:

```bash
nslookup example.com
```

**Output:**

```text
Server:		192.168.1.1
Address:	192.168.1.1#53

Non-authoritative answer:
Name:	example.com
Address: 93.184.216.34
```

The first two lines name the resolver that answered and the port it answered on — the same information `dig` prints as `SERVER:`. The phrase `Non-authoritative answer` is `nslookup`'s way of saying the data came from a cache rather than from the zone's own servers. In interactive mode (`nslookup` with no argument) you can change the record type with `set type=TXT` or `set type=AAAA` and query several names in one session, which is convenient when you are enumerating what a domain publishes. Both tools query DNS directly and therefore bypass `/etc/hosts`, which is why a name that resolves in your browser can still answer differently here.

<details>
<summary><strong>Read more</strong> — the whole <code>dig</code> answer, the null MX, and the shortcuts in full</summary>

The last four lines are the ones beginners overlook and experts read first. `Query time: 24 msec` is how long the recursive resolution took, and `SERVER: 192.168.1.1#53(192.168.1.1) (UDP)` names the resolver that actually answered *you* — here the lab gateway forwarding on your behalf. `WHEN` is the timestamp, and `MSG SIZE rcvd` is the size of the response, a number that becomes interesting when DNS responses grow large enough to trigger TCP fallback or fragmentation.

The null MX is worth understanding rather than skipping past: priority zero, target the root, is a deliberate way of saying "this domain does not accept mail at all". A resolver that finds it must stop looking, which is why the record exists: it tells the world not to bother trying. A domain that *does* accept mail returns one or more records with a hostname and a preference number, and the `.example` top-level domain is reserved for exactly this kind of illustration and therefore cannot collide with anything real. Both hostnames in a mail-accepting answer are useful to an assessor: each is a machine that accepts untrusted input from the internet, and each must be resolved further before you know what you are dealing with.

A zone transfer, if the authoritative servers allowed one, would hand over the entire contents of the DNS zone — every hostname, every mail server, every subdomain the organisation has ever registered — in one request. Modern servers refuse it, but the ones that do not remain one of the best findings available.

`+short` prints only the answer data, so `dig +short example.com` returns a single line that is trivial to feed into another command. `+noall +answer` drops the header and status blocks when you only want the records. `@1.1.1.1` sends the query to a specific resolver instead of the system default, which is how you compare what different resolvers see — and the two answers are not always the same, because an internal resolver may know names that public DNS does not, while a public one may be willing to answer for names that internal filtering hides. `-x` performs a reverse lookup, turning an address into a name (`dig -x 93.184.216.34`). And `+trace` follows the resolution step by step from the root servers down, which is the single best way to *learn* how DNS actually works rather than just reading about it.

</details>

### 1.7 Changing your DNS server

The resolver your machine uses by default is described in a plain text file, `/etc/resolv.conf`, and it is worth reading before you change it:

```bash
cat /etc/resolv.conf
```

**Output:**

```text
# Generated by NetworkManager
search lab.local
nameserver 192.168.1.1
```

Two directives appear in a typical file. The `nameserver` line names the resolver to ask — here the lab gateway, which forwards queries upstream — and the `search` line lists domains that are appended to unqualified names, which is why typing `ping fileserver` can work on a corporate network without a fully qualified name.

Changing the resolver is a matter of writing a new line into the file. The quickest way is with the shell's `echo` and the `>` redirect, which *overwrites* the file with whatever you supply:

```bash
echo "nameserver 1.1.1.1" > /etc/resolv.conf
```

Because `>` writes directly to the file rather than the screen, the command produces no output. Always read the file back with `cat` to verify that the overwrite succeeded as intended:

```bash
cat /etc/resolv.conf
```

**Output:**

```text
nameserver 1.1.1.1
```

The file now contains a single line pointing to Cloudflare's public resolver at `1.1.1.1`. Before relying on the system-wide setting, we can query `1.1.1.1` directly with `dig` and `+short` to confirm it is reachable from this network:

```bash
dig example.com +short @1.1.1.1
```

**Output:**

```text
93.184.216.34
```

The resolver answered with `93.184.216.34`, proving that outbound DNS traffic to `1.1.1.1` is permitted. Finally, run `dig` without specifying a server and filter for the `SERVER:` line to confirm that the system uses the new resolver by default:

```bash
dig example.com | grep "SERVER:"
```

**Output:**

```text
;; SERVER: 1.1.1.1#53(1.1.1.1) (UDP)
```

The `SERVER:` line now reports `1.1.1.1#53` instead of the local gateway, which proves that queries are actually leaving through the new resolver rather than merely that the file's contents changed. Note also why `cat /etc/resolv.conf` came *before* we used `>`: the `>` operator is destructive and has no undo, so reading a configuration file before overwriting it is a vital habit.

There is one important caveat and it catches everybody: on most modern systems `/etc/resolv.conf` is *generated*, not authored.

<details>
<summary><strong>Read more</strong> — why the file is generated, and why the resolver matters</summary>

NetworkManager, `dhclient`, `systemd-resolved` or `resolvconf` rewrites `/etc/resolv.conf` whenever a network event occurs, and may even replace the file with a symbolic link pointing at a runtime directory. A hand-edited file therefore survives only until the next DHCP lease renewal or reboot.

The durable approaches are to set the DNS server in the tool that owns the file — `nmcli con mod <name> ipv4.dns 1.1.1.1` for NetworkManager, a Netplan `nameservers:` stanza on Ubuntu Server, or `/etc/systemd/resolved.conf` followed by `systemctl restart systemd-resolved` — or, for a machine you control completely, to make the file immutable with `chattr +i /etc/resolv.conf`. The last option works and is sometimes the right answer on a lab machine, but it is a trap on anything else: every later network change will fail, and the next administrator will spend an afternoon wondering why.

Other public resolvers you will meet are Google's at 8.8.8.8 and Quad9 at 9.9.9.9, and every one of them has a published privacy policy worth reading before you make it your resolver. A file may list up to three `nameserver` lines, and they are consulted in order: the resolver asks the first one, and moves down the list only if the one above does not answer in time. That ordering is why a stale entry at the top of the list causes the mysterious pauses everybody eventually experiences.

For security work the resolver matters for two reasons. First, the DNS server sees every name you look up, so on an untrusted network a rogue resolver reads your reconnaissance and can answer with forged addresses. Second, using a resolver that validates DNSSEC or filters malicious domains changes what you can reach, which is why assessments often begin by identifying which resolvers a target environment uses and whether those queries are encrypted with DNS-over-HTTPS or DNS-over-TLS.

</details>

### 1.8 Mapping hostnames in `/etc/hosts`

Before any DNS query is sent, the system consults a local file. `/etc/hosts` is a static table of name-to-address mappings, and every Unix-like system has one; it is consulted first because of the lookup order defined in `/etc/nsswitch.conf`, where the `hosts:` line reads `files dns`, meaning "check the files first, then ask DNS".

```bash
cat /etc/hosts
```

**Output:**

```text
127.0.0.1	localhost
127.0.1.1	kali
::1		localhost ip6-localhost ip6-loopback

# The following lines are desirable for IPv6 capable hosts
ff02::1		ip6-allnodes
ff02::2		ip6-allrouters
```

The file's format is deliberately simple: an IP address in the first column, one or more names after it separated by whitespace, and a `#` starting a comment. The default entries map `localhost` to the loopback addresses and give the machine its own hostname (`kali`) at `127.0.1.1`. To add a custom mapping, open the file in `nano`:

```bash
nano /etc/hosts
```

After appending a new line for `portal.example.com` and saving the file, `nano` exits silently back to the prompt. Run `cat /etc/hosts` again to confirm the new entry is in place:

```bash
cat /etc/hosts
```

**Output:**

```text
127.0.0.1	localhost
127.0.1.1	kali
::1		localhost ip6-localhost ip6-loopback

# The following lines are desirable for IPv6 capable hosts
ff02::1		ip6-allnodes
ff02::2		ip6-allrouters
192.168.1.9	portal.example.com
```

The bottom line now maps `portal.example.com` to the local address `192.168.1.9`. Because `/etc/hosts` wins over DNS, the effect is immediate and global to the machine, which we can verify with `getent hosts`:

```bash
getent hosts portal.example.com
```

**Output:**

```text
192.168.1.9     portal.example.com
```

`getent hosts` queries the system's own name service switch — the exact mechanism applications use — and returns `192.168.1.9` from the hosts file. Next, we use `curl` with `-w` to print the HTTP status code and the remote IP address it actually connects to when requesting `http://portal.example.com/`:

```bash
curl -s -o /dev/null -w "%{http_code} %{remote_ip}\n" http://portal.example.com/
```

**Output:**

```text
200 192.168.1.9
```

`curl` received a `200` OK response from `192.168.1.9` while the URL still said `portal.example.com`. Contrast that with what happens when we ask `dig` for the same name:

```bash
dig +short portal.example.com
```

**Output:**

```text
93.184.216.34
```

Unlike `getent` and `curl`, `dig` bypasses `/etc/hosts` entirely because it speaks to DNS servers directly, so it still returns the real, public address `93.184.216.34`.

That discrepancy is a clear demonstration of the difference between *the system's view* and *DNS's view*.

<details>
<summary><strong>Read more</strong> — legitimate uses, and the same mechanism turned against you</summary>

The discrepancy is also why a name that works in the browser can fail in a script that uses `dig` or `nslookup` — the two are answering different questions.

Legitimate uses of the file are everywhere: giving a lab machine a friendly name before DNS exists, testing a new web server before its DNS record is changed, pinning a hostname so that an internal application stops resolving to a wrong address, and blocking a domain by pointing it at `127.0.0.1` or `0.0.0.0`.

The illegitimate use is what makes it relevant here, and it is the same mechanism in reverse: because the file is local and trusted, an attacker who can write to it — or who can run a rogue DHCP or DNS server that answers first — can send a user to a server of their choosing. That is the basis of DNS and ARP spoofing attacks, where a user types the correct address and lands on an attacker's copy of the site.

The defensive answers follow from the mechanism: monitor the integrity of `/etc/hosts` and `/etc/resolv.conf` on servers, prefer DNSSEC and DNS-over-HTTPS where the environment supports them, and treat any user-visible redirect to an unexpected address as an incident rather than a glitch.

</details>

---

## 2. Process management

> **Concept.** A process is a running program with an identity and a state of its own: the kernel gives it a PID and records its parent's PID (PPID), creates it from an existing process with `fork` and then replaces its image with a new program via `exec`, and moves it between states such as running, sleeping and zombie. Processes are steered with signals, the asynchronous notifications standardised by POSIX (`man 7 signal`): `SIGTERM` (15) politely asks a program to terminate so that it can clean up, whereas `SIGKILL` (9) cannot be caught, blocked or ignored and removes it on the spot. The scheduler decides which process runs next and weights that decision by each process's nice value, from −20 (highest priority) to 19 (lowest). The rule that any user may lower a process's priority but only root may raise it is the principle of least privilege applied to the CPU [Saltzer & Schroeder, 1975].

A process is simply a program that is running: the kernel has loaded it into memory, given it an identifier, scheduled it onto a CPU and accounted for the resources it uses. On a single-user laptop you might have two hundred processes running at any moment, most of them daemons — background services with no terminal attached — and on a server the number is larger still.

Knowing how to list them, read what they are consuming, decide which of them deserve more of the machine, stop the ones that have gone wrong, and move the ones that should be out of the way is the difference between administering a system and guessing at it.

<details>
<summary><strong>Read more</strong> — why this matters for security work</summary>

The same commands are how you find the process that is listening on a port, how you identify a suspicious daemon among legitimate ones, and how you stop an anti-virus agent that is interfering with an authorised test.

A process is also the unit of *accounting* on Linux: it carries the identity that every permission check is made against, so "which user is this process running as?" is one of the most useful questions you can ask about a system you are assessing.

</details>

### 2.1 `ps` — viewing your own processes

The primary tool for looking at processes is `ps`, short for *process status*. Typed with no arguments it lists the processes attached to the current terminal and owned by the current user, which keeps the output short enough to be useful while you are learning.

```bash
ps
```

**Output:**

```text
    PID TTY          TIME CMD
   4122 pts/0    00:00:00 bash
   4188 pts/0    00:00:00 ps
```

Only two processes appear, and that is not an error: `ps` is printing the processes that belong to this shell session, which is you plus the `ps` command itself. The three columns are the vocabulary the rest of the section builds on: `PID` is the process identifier, a number the kernel assigns to every process it creates and which is unique among the processes running at that moment — the handle you use to inspect, prioritise or kill anything. `TTY` names the terminal the process is attached to, and a `?` in that column means the process has no terminal at all, which is the signature of a daemon. `TIME` is the total CPU time the process has consumed since it started, not the wall-clock time it has existed.

<details>
<summary><strong>Read more</strong> — the flags that add detail, and why <code>TIME</code> is not wall-clock</summary>

`ps -f` adds a column showing the parent process ID (PPID) and the full command line, which begins the process of answering "who started this, and with what arguments?". `ps -ef` — the System V style that administrators of other Unix systems reach for out of habit — shows every process on the system with the same detail, which brings us to the next command.

That distinction between CPU time and wall-clock matters: a process that has been running for a week and used two seconds of CPU is behaving very differently from one that has used two hours.

</details>

### 2.2 `ps aux` — every process, every user

Running `ps` with the three BSD-style options `aux` displays all running processes for all users along with the resource figures for each one. This is the single most typed command in this section, and the header line it prints is worth memorising.

```bash
ps aux
```

**Output:**

```text
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.6 168412 12984 ?        Ss   09:41   0:04 /sbin/init splash
root           2  0.0  0.0      0     0 ?        S    09:41   0:00 [kthreadd]
root         812  0.0  0.1  15420  9216 ?        Ss   09:41   0:00 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
root        1044  0.0  0.4 240128 34220 ?        Ssl  09:41   0:01 /usr/lib/snapd/snapd
www-data    1204  0.1  1.1 3310420 88144 ?       Ssl  09:44   0:06 python3 /opt/portal/app.py
student     3901  0.0  0.1  10344  5120 pts/0    Ss   10:12   0:00 -bash
student     4122  0.0  0.0  12440  4096 pts/0    R+   10:31   0:00 ps aux
```

Each column answers a different question. `USER` is the account the process is running as, which is the first thing to check when you are deciding what you are allowed to do to it: a process owned by your own user can be signalled by you, while anyone else's needs root. `PID` is the identifier. `%CPU` and `%MEM` are the percentage of total CPU and memory the process is currently using, and `VSZ` and `RSS` are the memory figures in kilobytes — `VSZ` is the virtual size of the address space the process has reserved, while `RSS` is the memory it actually has in physical RAM right now. `TTY` is the terminal, with `?` for daemons, and `STAT` is a short code describing the process state.

The `STAT` codes are worth learning because they tell you whether a process is healthy: `R` means running or runnable — on the CPU or waiting its turn; `S` means interruptible sleep, the normal state of a process waiting for something such as a network packet or a keystroke, and almost everything on a quiet system is in `S`; `D` is uninterruptible sleep, which usually means blocked on disk or network I/O; `T` means stopped, which is what `Ctrl+Z` leaves behind; and `Z` means zombie, a process that has finished but whose parent has not yet collected its exit status.

`START` is the time the process began, `TIME` is cumulative CPU time and `COMMAND` is the command line — with the additional detail that a command wrapped in square brackets at the start is a kernel thread rather than a user process, which is why `[kthreadd]` has no memory and no terminal.

Where `ps` lists processes as rows, `pstree` draws them as a tree, which makes parentage obvious: every daemon that starts workers, and every shell that spawns children, appears as a branch rather than as two rows far apart. Adding `-p` prints each process's PID alongside its name:

```bash
pstree -p
```

**Output:**

```text
systemd(1)─┬─NetworkManager(645)─┬─{NetworkManager}(662)
           │                     └─{NetworkManager}(663)
           ├─apache2(3312)─┬─apache2(3313)
           │               └─apache2(3314)
           ├─cron(918)
           ├─sshd(812)───sshd(3901)───bash(4122)───pstree(4402)
           └─systemd-resolve(712)───{systemd-resolve}(714)
```

Every process sits under the one that started it, and the braces (`{...}`) mark threads rather than processes — the workers of NetworkManager and systemd-resolved are drawn that way because they share the PID of their leader. The line for `sshd` is the clearest illustration of the whole model: the daemon `sshd(812)` accepted a connection, forked a child `sshd(3901)` to handle it, and that child started the shell `bash(4122)` in which `pstree` itself is running.

<details>
<summary><strong>Read more</strong> — reading <code>STAT</code>, and the variations worth keeping</summary>

A process stuck in `D` cannot be killed even with `SIGKILL`, because it is not running any code that could handle the signal — a diagnosis, not an obstacle. A zombie consumes no CPU and no memory beyond the table entry, cannot be killed, and disappears when the parent reads the exit status or exits itself.

`VSZ` is usually much larger than `RSS` and is the more misleading of the two, because reserving address space costs nothing.

A few variations earn their place in your notes: `ps aux --sort=-%mem | head` lists the biggest memory consumers, `ps -eo pid,ppid,user,%cpu,cmd --sort=-%cpu | head` picks the columns you care about and sorts by CPU, and `ps -ef --forest` draws the process tree, showing which daemon spawned which worker — a view that turns "there are twelve nginx processes" into "there is one master and eleven workers".

</details>

### 2.3 Filtering processes by name

A busy machine produces hundreds of rows from `ps aux`, so in practice you filter. The technique is the pipe introduced in Part 1 (section 4.7), applied here to process output, and it works exactly as it does with any other command: pipe `ps aux` into `grep` and keep the lines that mention the program you are curious about. Here we look for metasploit's console, `msfconsole`.

```bash
ps aux | grep msfconsole
```

**Output:**

```text
root        5210  1.3  2.4 612344 198772 pts/1   Sl+  10:33   0:02 ruby /usr/bin/msfconsole -q
root        5233  0.0  0.0  12440  4096 pts/1    S+   10:33   0:00 grep msfconsole
```

Two lines come back, and understanding why is a small but genuinely useful lesson. The first is the real process: a Ruby interpreter running the `msfconsole` script, consuming 1.3% of CPU and about 2.4% of memory. The second line is the `grep` command itself, which matched its own command line — a permanent piece of bycatch that confuses beginners into thinking there are two processes.

There is a better tool for this job, and it is worth adopting as the default: `pgrep` was written to answer exactly the question "which processes match this name?", so it does not match itself and it prints only identifiers by default. Passing `-a` prints the full command line alongside the PID:

```bash
pgrep -a msfconsole
```

**Output:**

```text
5210 ruby /usr/bin/msfconsole -q
```

Only the real `msfconsole` process (`5210`) is returned, with no `grep` bycatch. You can also combine `-u` to filter by owner and `-l` to print the process name alongside the PID:

```bash
pgrep -u student -l bash
```

**Output:**

```text
3901 bash
```

This isolates the `bash` session (`3901`) belonging specifically to `student`. Finally, the `-f` flag matches against the entire command line rather than just the executable name, which is essential when a script is run through an interpreter like `python3`:

```bash
pgrep -f "app.py"
```

**Output:**

```text
1204
```

Because the executable name for `/opt/portal/app.py` is `python3`, a plain `pgrep app.py` would find nothing, whereas `pgrep -f "app.py"` inspects the arguments and returns PID `1204`.

<details>
<summary><strong>Read more</strong> — losing the <code>grep</code> bycatch, and matching by command line</summary>

The standard ways to get rid of the `grep` line are to exclude the search term from the results (`ps aux | grep msfconsole | grep -v grep`) or to bracket one character of the pattern (`ps aux | grep "[m]sfconsole"`), since the bracket expression matches the real process but does not appear literally in the grep process's own command line.

`pgrep` was written to answer exactly the question "which processes match this name?", so it does not match itself and it prints only identifiers by default.

Its companion, `pkill`, sends a signal to every process that matches, and we will use it later in this section once the concept of signals has been introduced.

</details>

### 2.4 `top` — finding the greediest process

`ps` gives you a snapshot; `top` gives you a film. The `top` command displays processes ordered by resource consumption and refreshes the display every few seconds, which makes it the tool of choice when you want to watch what a machine is doing right now — which process is eating the CPU, whether the load is rising, and how much memory is left.

```bash
top
```

**Output:**

```text
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

The five header lines summarise the whole machine and are worth reading before the process list. The first line gives the time, the uptime, the number of logged-in users and the three **load averages**, which report the average number of runnable processes over the last one, five and fifteen minutes. The second line counts processes by state. The third line breaks CPU time down by category. The fourth and fifth lines describe memory: total, free, used, and the page cache sitting in `buff/cache`.

Below the header comes the process list, which repeats several `ps` columns with two additions: `PR`, the kernel's priority, and `NI`, the nice value discussed in the next subsection.

`top` is interactive, and half a dozen keys carry most of its value.

If you would rather have colour, mouse support and scrollable columns, install `htop`, which presents the same information as `top` in a friendlier layout and adds a per-core percentage scale that many people find easier to read:

```bash
htop
```

**Output:**

```text
    0[|||||              35.0%]   Tasks: 87, 214 thr; 1 running
    1[|||                21.4%]   Load average: 0.42 0.31 0.24
  Mem[||||######     2.41G/3.82G]   Uptime: 00:55:12
  Swp[                   0K/2.00G]

  PID USER      PRI  NI  VIRT   RES  S CPU% MEM%   TIME+  Command
 1204 www-data   20   0 3233M 86.1M S  4.3  2.2  0:06.42 python3 /opt/portal/app.py
 1044 root       20   0  234M 33.4M S  0.7  0.9  0:01.19 /usr/lib/snapd/snapd
 5210 root       20   0  598M  194M S  0.3  5.0  0:02.11 ruby /usr/bin/msfconsole -q
    1 root       20   0  164M 12.7M S  0.0  0.6  0:04.33 /sbin/init splash
```

The header is the same information as `top`'s — per-core CPU bars, the load average, and memory and swap — and each process row repeats the fields you already know, with `CPU%` and `MEM%` on the same 100-per-cent scale. The keys are also familiar: `F6` chooses the sort column, `F9` sends a signal, `F10` quits, and `htop -u www-data` starts it filtered to one user, which is a quick way to watch a single service.

<details>
<summary><strong>Read more</strong> — reading the header, and the keys that matter</summary>

A load average above the number of CPU cores means work is queueing — on this single-core-equivalent virtual machine, 0.42 means the machine is comfortable, and any non-zero `zombie` or `stopped` count is worth following up. In the CPU line, `us` is user-space work, `sy` is kernel work, `id` is idle, `wa` is time waiting on I/O (a rising `wa` usually means a slow disk), and `st` is time *stolen* by the hypervisor — the figure that tells you another virtual machine on the same host is eating your CPU. `buff/cache` is memory the kernel is using to cache disk reads and will give back instantly if an application needs it; reading `free` without reading `buff/cache` is the classic beginner error, since Linux deliberately uses otherwise-idle RAM as a cache, so a machine with almost no `free` memory and a large `buff/cache` is working exactly as designed.

The keys: `P` sorts by CPU usage (the default), `M` sorts by memory, `k` prompts for a PID and a signal so that you can kill a process without leaving the display, `r` prompts for a PID and a new nice value so that you can re-prioritise one, `1` expands the CPU line to show each core individually, `h` shows the full help, and `q` quits.

If you would rather capture the output than watch it, `top -b -n 1 | head -20` runs it once in batch mode and prints the result, which is the form to use in a script or a report. And if you spend much time doing this, install `htop`: it is the same information with colour, mouse support and scrollable columns.

</details>

### 2.5 `nice` — setting the priority of a new process

On a machine with more work than CPU, the kernel scheduler decides who runs, and it makes that decision partly on the basis of each process's **nice value** — a number from −20 to 19 that expresses how willing the process is to give way to others. The scale runs backwards from intuition: −20 is the highest priority (least nice, most demanding), 19 is the lowest (extremely nice, always yielding), and the default is 0. A process started with `nice` carries the value you give it for its entire life, which is why the setting is applied at launch.

The `-n` flag introduces the adjustment, and here the `ssh-agent` daemon is started with a higher-priority nice value of `-10`, meaning it should be given preference by the scheduler:

```bash
nice -n -10 /usr/bin/ssh-agent
```

**Output:**

```text
SSH_AUTH_SOCK=/tmp/ssh-XXXXXXuP3n9g/agent.4412; export SSH_AUTH_SOCK;
SSH_AGENT_PID=4412; export SSH_AGENT_PID;
echo Agent pid 4412;
```

The program started normally — PID `4412` was assigned, and `ssh-agent` printed the environment variables a caller needs in order to talk to it. To verify that the kernel applied the requested priority, run `ps` with `-o pid,ni,cmd` to select specific columns and `-C ssh-agent` to match by command name:

```bash
ps -o pid,ni,cmd -C ssh-agent
```

**Output:**

```text
    PID  NI CMD
   4412 -10 /usr/bin/ssh-agent
```

The `NI` column shows `-10`, confirming that the priority request was accepted.

One permission rule matters here: any user may make a process *nicer* (raise its number, give way more), but only root may lower it (make a process more demanding). If a normal user tries the command above, the shell replies `nice: cannot set niceness: Permission denied` and no process starts.

The non-privileged and far more common use is the opposite direction — pushing a long, unimportant job out of the way with `-n 19` and `&` so that it runs in the background without competing with interactive work:

```bash
nice -n 19 ./backup-script.sh &
```

**Output:**

```text
[1] 5487
```

The shell starts `./backup-script.sh` in the background at the lowest possible priority (`19`) and reports job number `[1]` with PID `5487`. We can launch a second maintenance task, `updatedb`, the same way:

```bash
nice -n 19 /usr/bin/updatedb &
```

**Output:**

```text
[2] 5490
```

Both jobs are now running at the lowest possible priority, and the `[1]` and `[2]` job numbers with the PIDs after them are the shell's acknowledgement — the topic of section 2.8.

<details>
<summary><strong>Read more</strong> — the permission rule, and when to reach for <code>nice</code></summary>

The asymmetry is deliberate, because the whole point of the mechanism is to protect interactive work from being starved by background jobs, and it would be defeated if any user could claim the front of the queue.

The pattern is worth remembering, because "run this in the background at minimum priority" is the correct shape for almost every maintenance task on a machine someone else is using.

`-C` (match by command name, used above) is also a tidy alternative to the `grep`-and-pipe ritual of section 2.3, and it has the pleasant property of not matching itself.

</details>

### 2.6 `renice` — changing the priority of a running process

The priority of a process that is already running can be changed without restarting it, using `renice`. It takes a nice value or an adjustment and one or more PIDs, and it works on processes you did not start, provided you have the privileges to change them:

```bash
renice 20 6242
```

**Output:**

```text
6242 (process ID) old priority 0, new priority 20
```

The report line is unambiguous and names both values: PID `6242` went from a nice value of `0` to `20` (clamped by the kernel to the maximum nice value), meaning it now yields to everything else on the machine. We can verify the updated value in the process table with `ps`:

```bash
ps -o pid,ni,cmd -p 6242
```

**Output:**

```text
    PID  NI CMD
   6242  20 /usr/bin/updatedb
```

The `NI` column confirms the new priority for `/usr/bin/updatedb`. Now see what happens when an unprivileged user tries to lower the nice value (to `-5`, which would raise the process's priority):

```bash
renice -n -5 -p 6242
```

**Output:**

```text
renice: failed to set priority for 6242 (process ID): Permission denied
```

This demonstrates the permission rule from the previous subsection from the other side: as an unprivileged user, lowering the nice value is refused outright with `Permission denied`.

Three forms are worth keeping in your notes.

<details>
<summary><strong>Read more</strong> — the three forms, and the modern alternative</summary>

`renice -n 10 -p 6242` sets an absolute value using the explicit flags, which is the form to prefer in scripts because it is unambiguous. `renice -u student` changes the priority of every process owned by a user, which is a good way to quiet a runaway build account without hunting for PIDs. And `renice 5 -g developers` applies to a process group.

On a modern systemd machine there is a second, complementary mechanism — cgroups and the `systemd-run --slice` and `systemctl set-property` commands let you cap CPU and memory for a whole service rather than just hinting at the scheduler, and that is the approach to use when the requirement is "this service may never use more than one core" rather than "this service should be polite".

</details>

### 2.7 `kill` — stopping processes with signals

The `kill` command stops processes, and its name misleads in a useful way: what it actually sends is a *signal*, a numbered, asynchronous notification delivered to a process by the kernel, and the process may handle most signals however it likes — clean up, save state, close sockets, ignore them or exit. There are 64 signals in all, and the two that matter most are the polite one and the fatal one. Here the process with PID `6242` is sent signal number `1` (`SIGHUP`):

```bash
kill -1 6242
```

`kill` returns silently because sending a signal only delivers the notification to the target PID. To check whether process `6242` is still running, query its PID with `ps -p`:

```bash
ps -p 6242
```

**Output:**

```text
    PID TTY          TIME CMD
```

`SIGHUP` (signal 1) originally meant "the terminal you were connected to has hung up", and while daemons such as `sshd`, `nginx` and `rsyslog` reinterpret it as "reload your configuration", a simple program like `updatedb` has no handler for it and terminates — leaving the `ps` table empty. To see the names and numbers of all signals the system supports, run `kill -l`:

```bash
kill -l
```

**Output:**

```text
 1) SIGHUP	 2) SIGINT	 3) SIGQUIT	 4) SIGILL	 5) SIGTRAP
 6) SIGABRT	 7) SIGBUS	 8) SIGFPE	 9) SIGKILL	10) SIGUSR1
11) SIGSEGV	12) SIGUSR2	13) SIGPIPE	14) SIGALRM	15) SIGTERM
...
```

Signal 15 (`SIGTERM`) is the *default* when you type `kill` with no number, and it is the correct first attempt for any process: it politely asks the program to terminate so it can flush its buffers, remove temporary files and close sockets. When a process ignores `SIGTERM` and refuses to exit, you escalate to signal 9 (`SIGKILL`):

```bash
kill -9 4378
```

Signal 9 (`SIGKILL`) cannot be caught, blocked or ignored — the kernel removes process `4378` on the spot and returns silently to the prompt.

That ordering — `SIGTERM` first, then escalate — is the professional habit and it is worth enforcing on yourself from the first day. A process that does not die to `SIGTERM` is telling you something: it may be blocked in uninterruptible I/O, it may be a zombie, or it may be deliberately ignoring the signal, and each of those deserves a moment's thought before you reach for `-9`.

Two related commands make targeted killing easier.

<details>
<summary><strong>Read more</strong> — the rest of the signal list, and why the order matters</summary>

Signal 9 is the right tool for a process that has ignored `SIGTERM` and the wrong tool for everything else, because `SIGKILL` gives the program no chance to clean up, which is how databases are corrupted and stale lock files are left behind. Signal 2, `SIGINT`, is what `Ctrl+C` sends from the terminal; signal 3, `SIGQUIT`, is `Ctrl+\`; signal 18 stops a process and 19 resumes it, which is what `Ctrl+Z` and `bg` do under the hood.

A quick sanity check for the zombie case is `ps -o pid,stat,cmd -p <pid>`, where a `Z` in the state column means no signal will ever work, because there is no running process left to receive it.

`pkill` sends a signal to every process matching a pattern, so `pkill -f app.py` stops the web application by its command line and `pkill -u student -TERM -f backup` stops a user's backup jobs without touching anything else. `killall` does the same by exact program name. Both are powerful enough to be dangerous — a pattern that is too broad will match something you did not intend, and `pkill -9 -f python` on a busy machine can take out a dozen innocent processes — so the safe workflow is always to run `pgrep` first, look at what would be affected, and only then send the signal with the same pattern.

</details>

### 2.8 Background jobs, `jobs`, `fg` and `nohup`

A command typed at a prompt owns the terminal until it finishes, which is inconvenient the moment you start something long-running. The shell provides **job control** for exactly this problem: appending a single ampersand (`&`) to a command runs it in the background, immediately returning your prompt while the process keeps working.

```bash
nano lab-notes.txt &
```

**Output:**

```text
[1] 4312
```

The shell replies with the job number `[1]` and the process identifier `4312`. To check the status of backgrounded tasks in the current shell session, run `jobs`:

```bash
jobs
```

**Output:**

```text
[1]+  Stopped                 nano lab-notes.txt
```

Because `nano` is an interactive full-screen editor, it immediately *stops itself* when placed in the background without a terminal, so `jobs` lists it as `Stopped` rather than `Running`. To bring job `1` back to the foreground where it can use your keyboard, run `fg %1`:

```bash
fg %1
```

Running `fg %1` hands control of the terminal back to `nano` (where you can either exit or press `Ctrl+Z` to suspend it again). Now compare what happens when we background a genuinely non-interactive script:

```bash
./backup-script.sh &
```

**Output:**

```text
[2] 4701
```

The script starts in the background as job `[2]` with PID `4701`. Run `jobs -l` to list all jobs in the current shell alongside their PIDs:

```bash
jobs -l
```

**Output:**

```text
[1]+  4312 Stopped                 nano lab-notes.txt
[2]-  4701 Running                 ./backup-script.sh &
```

With a non-interactive program the behaviour is what you would hope: job `[2]` is `Running` in the background, while `+` marks the current default job and `-` marks the previous one. If a stopped job does not need keyboard input, you can resume it in the background with `bg`:

```bash
bg %1
```

The full vocabulary of job control is small: `Ctrl+Z` *stops* the foreground process and returns you to the prompt, `bg %1` resumes a stopped job in the background, `fg %1` brings it back to the foreground, `jobs` lists them, `kill %1` signals a job by number rather than by PID, and `disown %1` removes a job from the shell's table.

That last point explains the most common way people lose work. When a terminal closes, the shell sends `SIGHUP` to its jobs, and any process that does not handle it dies — so a download started with a plain `&` in a terminal window that you then close simply disappears. The traditional defence is `nohup`, which detaches the process from the hangup signal and redirects its output:

```bash
nohup ./scan.sh > scan.log 2>&1 &
```

**Output:**

```text
[3] 4820
```

The redirection `> scan.log 2>&1` sends both standard output and standard error into `scan.log`, so `nohup` starts job `[3]` (PID `4820`) without printing a warning. We can verify that the process is running normally with `ps`:

```bash
ps -o pid,stat,cmd -p 4820
```

**Output:**

```text
    PID STAT CMD
   4820 S    /bin/bash ./scan.sh
```

The `STAT` column shows `S` (interruptible sleep), confirming that `./scan.sh` is executing cleanly in the background. If you omit the file redirection, `nohup` automatically redirects output to `nohup.out` for you:

```bash
nohup ./scan.sh &
```

**Output:**

```text
[4] 4833
nohup: ignoring input and appending output to 'nohup.out'
```

Here job `[4]` (PID `4833`) is launched and `nohup` informs you that all output is being appended to `nohup.out` in the current directory, ensuring the job survives even if the terminal closes.

<details>
<summary><strong>Read more</strong> — why backgrounding is for compilers, not editors</summary>

The general rule from the first transcript is worth internalising: backgrounding is for programs that do not need your keyboard, such as a compiler, a download, a scan or a long copy — not for interactive editors.

It is also safer to name the job explicitly (`fg %1`) until the idea is second nature, since `fg` with no argument resumes whatever was most recently backgrounded. `disown %1` in particular removes a job from the shell's table so that it is not sent `SIGHUP` when the shell exits.

Omit the redirection, as in the second `nohup` invocation, and `nohup` writes the output to `nohup.out` in the current directory and says so — the message appears on your terminal *after* the job line because it is written to standard error while the shell is still printing the job number. The process now survives the terminal, and the `STAT` column shows it sleeping normally.

For anything you will want to come back to, a terminal multiplexer such as `tmux` or `screen` is the better tool, because it keeps a whole session alive on the server and lets you reattach to it from anywhere, with scrollback and multiple windows. `nohup` remains the lightweight answer when you only need one command to outlive your session.

</details>

### 2.9 Scheduling a process with `at`

Sometimes the right moment to run a job is in the future. The `at` command schedules a command to run **once**, at a specified time, and it is the tool to reach for when you want a task performed after you have gone home or after some other event has happened. It reads the commands to run from standard input, which is why the invocation looks interactive.

```bash
at 9:00pm
```

**Output:**

```text
warning: commands will be executed using /bin/sh
at> /root/simple_bash.sh
at> <EOT>
job 3 at Sat Oct 10 21:00:00 2026
```

The first line is a warning rather than a complaint, and it is worth heeding: `at` runs its commands with `/bin/sh`, not with your interactive shell, so anything that depends on bash-specific syntax, aliases or your environment variables will not work — write the job as you would write a script, with full paths. The `at>` prompt then accepts commands one per line, and pressing `Ctrl+D` — which the shell displays as `<EOT>`, for end of transmission — ends the job. The confirmation line reports the job number and the exact time it will run.

The queue of pending jobs is managed with three short commands: `atq` lists them, `at -c <job>` prints the entire environment and command that a scheduled job will run — the definitive way to debug one that misbehaves — and `atrm <job>` removes it.

The service behind `at` is a daemon called `atd`, and on a minimal system it may not be running — which is the classic reason a job is accepted but never fires. The service manager reports both whether it is running now and whether it will start at boot:

```bash
systemctl status atd
```

**Output:**

```text
● atd.service - Debian atd, a job scheduler daemon
     Loaded: loaded (/lib/systemd/system/atd.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2026-10-13 09:41:02 UTC; 1h 12min ago
   Main PID: 733 (atd)
      Tasks: 1 (limit: 2261)
     Memory: 1.1M
        CPU: 12ms
     CGroup: /system.slice/atd.service
             └─733 /usr/sbin/atd -f
```

Reading the same two fields as everywhere else: `Loaded … enabled` means `atd` starts at boot, and `Active: active (running)` means it is alive right now, with `Main PID 733 (atd)` naming the process. If `Active` read `inactive (dead)`, you would run `systemctl enable --now atd` to start it immediately and at every boot.

For jobs that should run **repeatedly** — every night, every hour, every Monday — the tool is `cron` rather than `at`, and the difference is worth stating plainly: `at` is for one-offs, `cron` is for recurrence. A user's scheduled jobs live in a crontab, edited with `crontab -e`, and each line contains five time fields followed by the command to run:

```bash
crontab -l
```

**Output:**

```text
# m h  dom mon dow   command
  0 3  *   *   *     /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
 30 1  *   *   0     /usr/local/bin/weekly-report.sh
```

Reading the fields in order: minute, hour, day of month, month, day of week, where `*` means "every". So the first line runs the backup at 03:00 every day, and the second runs the weekly report at 01:30 every Sunday (day-of-week 0).

For security work, scheduled jobs matter for two reasons.

<details>
<summary><strong>Read more</strong> — <code>at</code> formats, access control, and the security angle</summary>

`at` interprets a variety of human formats: `at 9:00pm`, `at 14:30`, `at now + 10 minutes`, `at midnight`, `at 09:00 tomorrow`. Times without a date are taken as today if still in the future and otherwise tomorrow, which occasionally surprises people at 23:59.

The service behind `at` is a daemon called `atd`, and on a minimal system it may not be running; `systemctl status atd` will say so, and `systemctl enable --now atd` fixes it. Access to `at` is controlled by `/etc/at.allow` and `/etc/at.deny`, the same allow-and-deny pattern used by `cron`, which is how administrators keep ordinary users from scheduling work on production machines.

The system-wide equivalents live in `/etc/crontab` and `/etc/cron.d/`, with the addition of a user field, while the directories `/etc/cron.hourly`, `/etc/cron.daily`, `/etc/cron.weekly` and `/etc/cron.monthly` accept plain scripts with no time fields at all. Modern distributions increasingly use systemd timers instead, which offer the same scheduling with better logging and the ability to trigger on events rather than only on the clock — `systemctl list-timers` shows what is scheduled on such a system.

Defensively, `cron` is a classic persistence mechanism, so anything scheduled that you do not recognise deserves attention: a job that downloads and runs a script, or that reappears after being removed, is a finding. Offensively, `at` is the reliable way to schedule a rollback when you are about to make a change that might cut off your own access — a firewall reload, a network reconfiguration, an SSH daemon restart — since a job armed before the change will restore the old state even if your session dies with it. That single technique has rescued more remote administration sessions than any other habit in this section.

</details>

---

## 3. User environment variables

> **Concept.** Every process carries an *environment block* — a private array of `NAME=value` strings that the kernel copies into a child when it is created by `fork`/`exec` — so environment variables flow down the process tree and never back up. A shell variable stays inside the shell that defined it, and `export` is what marks it for inclusion in the environment of children; this is why a value set with `HISTSIZE=0` is invisible to another program until it is exported, and why `env` and `printenv` show only the exported set. The environment decides concrete behaviour: `PATH` is the ordered, colon-separated list of directories searched for a bare command name (POSIX), and `EDITOR`/`VISUAL` choose the editor that tools such as `crontab -e` and `visudo` launch. Because a child inherits its parent's environment without re-checking it, a writable directory early in `PATH` or an attacker-controlled `LD_PRELOAD` is a classic privilege-escalation vector — an environment variable is trusted input, and trusted input is where least privilege [Saltzer & Schroeder] must be enforced.

Variables are simply named values — key-value pairs held by a running shell — and understanding them is a must for getting the most out of a Linux system, because a great deal of what the shell and the programs it launches consider to be "the environment" is nothing more than a list of them. There are two families, and the distinction matters: a **shell variable** exists only inside the shell that created it, while an **environment variable** has been *exported*, which means it is copied into the environment of every child process.

The convention for names is upper case — `HOME`, `PATH`, `USER` — which is a convention rather than a rule, but a valuable one, because `HISTSIZE` and `histsize` are two entirely different variables and only one of them is the one the shell uses.

<details>
<summary><strong>Read more</strong> — how variables flow down the process tree</summary>

An exported variable is therefore present in every program you run from that shell and every process those programs start in turn, while a plain shell variable is invisible to all of them.

Variables are inherited down the process tree and never flow back up, which explains most of the confusing behaviour beginners meet: a child process can read what you exported, but it cannot change what its parent sees.

</details>

### 3.1 Viewing all the variables

Two commands list variables, and the difference between them is the distinction above made concrete: `set` shows everything the current shell knows, while `env` shows only the exported environment that child processes will actually receive.

```bash
set | more
```

**Output:**

```text
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

The pipe into `more` is not optional in practice: on a normal desktop `set` prints several hundred lines, most of them shell-internal variables such as `BASH_ARGC` that you will never use. Among the noise a handful of familiar names appear, and because the output is ordinary text going to standard output, it can be filtered, sorted and folded like anything else — which is exactly what the next subsection does.

<details>
<summary><strong>Read more</strong> — what <code>set</code> shows that <code>env</code> cannot</summary>

`set`, typed with no arguments, prints every shell variable and function the current shell knows about, including all the environment variables it inherited. `env` prints only the exported environment, which is the list that child processes will actually receive — so when the two disagree in the next subsection, the difference is not a bug but the shell-versus-environment distinction made observable.

Among the noise, a handful of familiar names appear: `HOME` is your home directory, `HOSTNAME` is the machine's name, `HISTFILE` is the file your command history is written to, and `HISTSIZE` is the number of commands the shell keeps in memory to offer you when you press the up arrow.

</details>

### 3.2 Filtering for a particular variable

Piping `set` into `grep` is the way to find one variable among hundreds, and the technique is identical to the process filtering of section 2.3. Here we look for `HISTSIZE`:

```bash
set | grep HISTSIZE
```

**Output:**

```text
HISTFILESIZE=2000
HISTSIZE=1000
```

`grep` returns both variables whose names contain `HISTSIZE`: `HISTFILESIZE=2000` (lines saved on disk) and `HISTSIZE=1000` (commands kept in memory). To expand `HISTSIZE` inside a string, prefix its name with `$`:

```bash
echo "my history holds $HISTSIZE commands"
```

**Output:**

```text
my history holds 1000 commands
```

**`$` before a name means "substitute the value"**, so the shell replaces `$HISTSIZE` with `1000` before `echo` runs (dropping the dollar sign would print the literal word `HISTSIZE`). Now compare what happens when we ask `printenv` — an external program — for `HISTSIZE`:

```bash
printenv HISTSIZE
```

`printenv` prints *only the value* of an environment variable, and here it printed nothing at all because `HISTSIZE` is a shell variable that has never been **exported** to child processes. By contrast, `HOME` *is* exported, so `printenv HOME` prints its value:

```bash
printenv HOME
```

**Output:**

```text
/root
```

Because `HOME` is part of the exported environment, the child `printenv` process receives `/root` and prints it.

<details>
<summary><strong>Read more</strong> — why <code>printenv HISTSIZE</code> prints nothing, and what history records</summary>

The exit status of the silent `printenv HISTSIZE` tells the same story, returning `1` for a variable that is not in the environment. There is no better demonstration of the difference between a shell variable and an environment variable, and it explains the behaviour of every program you will ever wonder about: if a tool claims not to know a value that is plainly set in your terminal, the variable was never exported.

Keeping 1000 commands in history is comfortable for interactive work, and it is also a fact with security implications worth noting: your history file is a record of everything you have typed, including passwords passed as command-line arguments and hosts you have connected to, which is why `~/.bash_history` is among the first files an attacker reads on a compromised machine — and why clearing it, or disabling history for sensitive commands with a leading space, is part of an operator's routine.

</details>

### 3.3 Changing a value temporarily

Changing a variable is a matter of assigning it a new value, and the assignment is written with no spaces around the equals sign:

```bash
HISTSIZE=0
```

A variable assignment succeeds silently in the current shell. Run `echo $HISTSIZE` to confirm that the new value is in place:

```bash
echo $HISTSIZE
```

**Output:**

```text
0
```

The output `0` confirms the change, and the effect is immediate: the shell no longer keeps any commands in memory for the current session, so pressing the up arrow produces nothing until you change the value back. You can also redirect a variable's value into a file to save it:

```bash
echo $HISTSIZE > ~/valueofHISTSIZE.txt
```

Because `>` redirects standard output into `~/valueofHISTSIZE.txt`, nothing is printed to the terminal. Read the file back with `cat` to verify what was saved:

```bash
cat ~/valueofHISTSIZE.txt
```

**Output:**

```text
0
```

`cat` shows `0`, confirming the value was written to disk. Two details in this transcript are worth pausing on, because they are where beginners lose time: the spacing rule, and the habit of saving a value *before* you change it.

<details>
<summary><strong>Read more</strong> — the spacing rule, and saving a value before you change it</summary>

The first is the spacing rule: `HISTSIZE=0` assigns, while `HISTSIZE = 0` is interpreted by the shell as an attempt to run a command called `HISTSIZE` with two arguments, which fails with `HISTSIZE: command not found`. The equals sign must be glued to the name and to the value, and no other arrangement works.

The second detail is the redirect in the third command. The intent — and a habit worth adopting — is to *save the old value somewhere before changing it*, so that you can always put things back: `echo $HISTSIZE > ~/valueofHISTSIZE.txt` writes the current value into a file in your home directory, and `cat` confirms what was written. In this case we saved the already-changed value of 0, so the file now records the experiment rather than the default — a reminder to save *before* you edit, not after.

</details>

### 3.4 Making the change permanent with `export`

The assignment above lasts as long as the session does. Opening a new terminal gives you a fresh shell with a fresh environment, and `HISTSIZE` is back to 1000 because the value was never written down anywhere. To make a variable pass to child processes, assign it and then mark it with `export`:

```bash
HISTSIZE=0
```

The assignment sets `HISTSIZE` to `0` in the current shell. Next, run `export` so the shell copies `HISTSIZE` into the environment of any child process it launches:

```bash
export HISTSIZE
```

`export` succeeds silently (you can also combine both steps as `export HISTSIZE=0`). To prove that child processes now inherit the variable, launch a new child shell with `bash -c` and ask it to print `$HISTSIZE`:

```bash
bash -c 'echo "child shell sees HISTSIZE=$HISTSIZE"'
```

**Output:**

```text
child shell sees HISTSIZE=0
```

Because `HISTSIZE` was exported, the child `bash` process inherited `0` and printed `child shell sees HISTSIZE=0`. Now run the control experiment by asking a child shell about `HISTFILE`, which is set in our parent shell but was never exported:

```bash
bash -c 'echo "child shell sees HISTFILE=$HISTFILE"'
```

**Output:**

```text
child shell sees HISTFILE=
```

The child reports an empty value (`HISTFILE=`) even though the parent shell has it set — the inheritance rule doing exactly what it says. Finally, to make the exported setting survive future logins and new terminal windows, append the `export` line to `~/.bashrc`:

```bash
echo 'export HISTSIZE=0' >> ~/.bashrc
```

The `>>` operator appends the line to the end of `~/.bashrc` without overwriting existing settings, so bash will run `export HISTSIZE=0` automatically each time it starts an interactive shell.

<details>
<summary><strong>Read more</strong> — which startup file to edit, and reloading it without logging out</summary>

Using `>` instead of `>>` here would erase whatever was in `~/.bashrc`, including the lines that define your prompt and aliases.

Which file to edit depends on the shell and the moment: `~/.bashrc` for interactive bash shells, `~/.bash_profile` or `~/.profile` for login shells, `/etc/environment` or a file under `/etc/profile.d/` for system-wide settings that apply to every user.

A change to `~/.bashrc` takes effect in the *next* shell you start, or immediately in the current one if you load it by hand with `source ~/.bashrc` — a command worth remembering, since it saves logging out and back in for every edit.

</details>

### 3.5 Creating and deleting your own variables

Variables do not have to be pre-defined by the system; you can create them, name them whatever you like and use them as shorthand for values you type often. Here a variable is created to hold a domain name:

```bash
url_variable="example.com"
```

The assignment creates `url_variable` silently in the current shell. Use `echo` with `$` to read its value back:

```bash
echo $url_variable
```

**Output:**

```text
example.com
```

The shell expands `$url_variable` and prints `example.com`. You can also embed the variable inside a double-quoted string, which is how variables earn their keep in scripts and longer commands:

```bash
echo "the target is $url_variable"
```

**Output:**

```text
the target is example.com
```

Inside double quotes, `$url_variable` is substituted seamlessly to produce `the target is example.com`. A variable can be changed at any time simply by assigning a new value to the same name:

```bash
url_variable=example.org
```

The new assignment overwrites the old value silently. Print `$url_variable` again to confirm the update:

```bash
echo $url_variable
```

**Output:**

```text
example.org
```

The variable now holds `example.org`. When you no longer need a variable, remove it completely from the shell with `unset`:

```bash
unset url_variable
```

`unset` deletes the variable silently (note that you pass the bare name `url_variable`, without `$`). If you now try to print `$url_variable`, see what happens:

```bash
echo $url_variable
```

The `echo` command prints only a blank line, because an undefined variable expands to nothing at all rather than producing an error.

That silent expansion is a trap worth naming, because scripts that fail mysteriously usually fail here.

<details>
<summary><strong>Read more</strong> — the silent-expansion trap, and why quoting is the default defence</summary>

If a variable is unset or misspelled, `$name` becomes an empty string and the command runs with a missing argument instead of complaining. Two defences are standard. Quoting turns a missing value into an empty argument rather than into no argument at all — `echo "$url_variable"` is safer than `echo $url_variable` in every context where the value might contain spaces or be empty. And the `${var:?message}` form makes the shell abort with your message if the variable is unset, which is the right way to check a required input in a script: `: "${url_variable:?please set url_variable}"`.

The other habit from this subsection is quoting on assignment. `url_variable="example.com"` needs no quotes for a value without spaces, but `url_variable="the example domain"` would be interpreted as an assignment of `the` followed by a command called `example domain` without them — a mistake that produces one of the more confusing shell errors you will meet. Quoting the right-hand side of an assignment is never wrong, so it is worth doing by default.

</details>

### 3.6 The variables worth knowing

The environment is not an academic topic; a handful of variables change how the system behaves for you, and knowing them turns several classes of confusing failure into one-line diagnoses.

`PATH` is the most important of them, and we met it in Part 1 (section 2.8): a colon-separated list of directories the shell searches, in order, for the command you typed. `echo $PATH` shows your list, and appending to it is a common operation — `export PATH="$PATH:/opt/tools/bin"` makes a directory of your own scripts runnable by name.

`HOME` is your home directory and is what `~` expands to; `USER` and `LOGNAME` name the current account, and `SHELL` records your login shell. `PS1` is the primary prompt string, which is why the prompt in this guide reads `root@kali:~#` and why changing `PS1` changes it instantly. `HISTFILE`, `HISTSIZE` and `HISTFILESIZE` govern command history, as we have seen, and `LANG` and `LC_ALL` set the language, encoding and formatting conventions that programs use.

<details>
<summary><strong>Read more</strong> — the variables worth knowing, and the one with a privilege-escalation angle</summary>

The *order* of `PATH` determines who wins when two directories hold a program with the same name, and the security angle is real in both directions: a `PATH` that includes `.` (the current directory) lets a malicious `ls` in a shared folder be executed by an unsuspecting user, and a writable directory early in root's `PATH` is a classic privilege escalation.

Every program that stores a configuration file uses `HOME`, so a wrong value for a user running a service is a common source of "the daemon cannot find its config" problems. `LANG` and `LC_ALL` are the reason a script can behave differently on two machines with identical software — `sort` orders letters differently under different locales, and dates are printed in the local format. `EDITOR` and `VISUAL` decide which editor tools such as `crontab -e`, `git commit` and `visudo` open for you, which is why setting `export EDITOR=nano` is one of the first things a beginner should do. And `LD_PRELOAD` deserves a mention by name, because it asks the dynamic loader to load a shared library into every process the shell starts — a legitimate debugging and compatibility tool, and simultaneously a well-known privilege-escalation primitive when it can be set for a SUID binary. Seeing it in an environment on a production system is worth an investigation.

</details>

Taken together, the three topics of this part — networks, processes and the environment — are the layer of Linux that sits between "I can use the command line" and "I can administer or assess this machine". An interface with a wrong address, a process eating a core, a name resolving through the wrong resolver: these are the ordinary faults, and the commands above are the ordinary answers to them.

---

## 4. Quick reference cheat sheet

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
| `ip neigh` | shows the ARP/neighbour cache | `ip neigh` |
| `ss -tulpn` | lists listening TCP/UDP sockets and their processes | `ss -tulpn` |
| `traceroute <host>` | prints the router-by-router path to a host | `traceroute example.com` |
| `mtr --report <host>` | path plus per-hop loss and latency statistics | `mtr --report example.com` |
| `nslookup <name>` | quick DNS lookup and interactive query mode | `nslookup example.com` |

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
| `pstree -p` | draws the process tree with PIDs | `pstree -p` |
| `htop` | interactive process viewer with colour and mouse | `htop` |
| `systemctl status <unit>` | running state and boot state of a service | `systemctl status atd` |

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

## 5. Practice exercises

The commands in this part only become yours when you have typed them yourself, so here is a short sequence that builds on the work you did in Part 1 and exercises everything above. Work on a virtual machine or in a container, and take a snapshot first.

1. **Networks.** Record your interface's address, mask, MAC address, gateway and resolver. Change the address by hand and prove that the change is temporary by restarting the networking service. Spoof the MAC address, then run `dhclient` and explain which value came back and why. Finally, ask `dig` for the A, MX and NS records of a domain you control, and say in one sentence what each record type is telling you.
2. **Processes.** Find the process that is using the most memory, and the one that is using the most CPU. Run a long job at the lowest priority in the background, list it with `jobs -l`, then stop it, background it again and foreground it. Schedule a harmless command with `at` to run two minutes from now and confirm it ran. Send `SIGTERM` to a process of your choosing, and only if that fails escalate to `SIGKILL`, explaining in writing why the order matters.
3. **Environment.** Print the value of `HISTSIZE`, save it to a file in your home directory, change it for the current shell only, and prove that a child shell does not see the change. Then export it, prove that a child shell does see it, and finally make it permanent in `~/.bashrc`. Create a variable of your own, use it in a command, and delete it with `unset` — noting what an unset variable expands to.

If you can complete these three exercises without looking anything up, you are comfortable with the layer of Linux that sits between using the command line and administering a system — and you have covered everything in the first two parts of this guide.

---

## Closing notes

The three topics in this part share a theme that is easy to miss while you are memorising flags: they are all about **state that something else depends on**. An address is depended upon by every connection the machine makes; a process is depended upon by whoever is using the service it provides; an environment variable is depended upon by the child processes that inherit it and never report what they were given. That is why the commands here are more consequential than the ones in Part 1 — a mistake does not produce an error message, it produces a machine that behaves differently from the way you expected, often for somebody else.

Two habits from Part 1 are worth carrying forward, because they cover most of the risk. The first is to *look before you change anything*; the second is to *have a way back*.

<details>
<summary><strong>Read more</strong> — the two habits that cover most of the risk</summary>

The first is to *look before you change anything*: read a configuration file before you overwrite it, run `pgrep` before you run `pkill`, print an address table before you alter an interface, and save a value before you unset or reassign it.

The second is to *have a way back*: a snapshot, a second console, or a rollback scheduled with `at` before you touch the network.

</details>

**Part 3** is where those threads come together. [Linux for Beginners (Part 3): Bash Scripting, Automation and Services](03-linux-fundamentals-part-3.md) takes the variables, the pipelines and the processes you have just met and turns them into programs — a Bash script that takes input and does something useful, a `cron` job that runs it on a schedule without anyone remembering to launch it, and the services (Apache, OpenSSH, FTP) that let a machine answer requests by itself. Beyond that, natural next steps are **service management** with `systemctl` and the systemd journal, **package and configuration management** across more than one machine, and a deeper study of the **permission and privilege model**. Each of them builds directly on the material above, and each of them is far less intimidating once moving around the file system, reading a long listing and inspecting a running process all feel like second nature.
