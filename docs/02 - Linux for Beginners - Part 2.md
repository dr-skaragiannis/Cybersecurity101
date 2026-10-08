# Linux for Beginners (Part 2): Networks, Processes and Environment Variables

========
*This is Part 2 of a three-part guide. It continues [Linux for Beginners (Part 1): The Shell, Files, Text, Packages and Permissions](01-linux-for-beginners-part-01.md), and it assumes you are comfortable with the material there — moving around the file system, reading a long listing, editing text from the command line, using the package manager and reading permission strings. Its continuation — scripting, scheduling and services — is in [Linux for Beginners (Part 3)](03-linux-for-beginners-part-03.md).*
>>>>>>>> origin/arena/52f71d52-attack-scripts:docs/02-linux-for-beginners-part-02.md

## Introduction

Part 1 covered the layer of Linux that everything else is built on: the shell, the file system, text, packages and permissions. This part moves one level deeper, into the things that decide how a machine actually behaves. We look at the **network** — the interfaces it presents, the addresses and hardware identifiers it carries, the server that hands its addresses out, and the name service it consults before it can connect to anything. We look at **processes** — how to see everything that is running, read what it is consuming, decide which work deserves the machine's attention, stop what has gone wrong, and arrange for jobs to run later. And we look at **environment variables**, the quiet configuration layer that decides which program runs when you type a bare name, which resolver answers, and what a newly launched shell already knows.

Every section keeps the same shape as Part 1: a transcript of the commands that were typed and the output they produced, followed by a paragraph-by-paragraph explanation of what happened and why it matters. Where a traditional tool has a modern replacement — `ifconfig` against `ip`, `netstat` against `ss`, `top` against `htop` — both are shown, because older documentation, interview questions and real machines mix them freely.

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

Networking is a crucial topic for anyone heading towards security work, because a great deal of what you will eventually be asked to test lives on the network rather than on a single machine. Even for everyday administration, you cannot get far without knowing how to look at an interface, read the address it holds, understand which server is answering your name lookups, and change any of those things deliberately. This section works through the tools that do exactly that, in the order you would reach for them: inspect an interface, inspect a wireless interface, change an address, spoof a hardware address, ask a DHCP server for a lease, interrogate DNS, change which resolver you use, and override a name locally.

One note before we start, because it explains why some of the commands below look old-fashioned. The traditional tools are `ifconfig`, `iwconfig`, `route` and `netstat`, and they come from a package called `net-tools` that many modern distributions no longer install by default. They have been superseded by the `ip` command (from the `iproute2` suite) and by `ss`. Every traditional command in this section is therefore accompanied by its modern equivalent, and the habit worth forming is to read both: legacy documentation, older tutorials and interview questions are full of `ifconfig`, while every current distribution ships `ip`.

### 1.1 `ifconfig` — analysing the network interfaces

The `ifconfig` command is the most basic tool for interacting with active network interfaces. Run with no arguments it prints every interface the kernel knows about, which on a typical lab machine means a physical or virtual Ethernet adapter plus the loopback interface.

```bash
root@kali:~# ifconfig
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

Two interfaces are listed, and reading them properly is the skill this section teaches. The first line of each block names the interface and its flags: `UP` means the interface is administratively enabled, `RUNNING` means the driver reports a carrier (a cable is plugged in, or the virtual switch is connected), `BROADCAST` and `MULTICAST` describe the traffic the interface will accept, and `mtu 1500` is the maximum transmission unit — the largest payload the link will carry in one frame, which matters when you are diagnosing fragmentation problems. The loopback interface has an `mtu` of 65536 because it is not a real wire; nothing has to fit into an Ethernet frame when the packets never leave the machine.

The second line of each block carries the addressing. `inet 192.168.1.17 netmask 255.255.255.0 broadcast 192.168.1.255` is IPv4: the address, the mask that defines which part of the address is the network and which part the host, and the address that reaches every machine on the local segment. From those two numbers you can derive the rest of the subnet mentally, which is a skill worth practising: a `/24` mask (255.255.255.0) with the address 192.168.1.17 means the network is 192.168.1.0, the usable host range is .1 to .254, and the broadcast is .255. The `inet6` line is the automatically generated IPv6 link-local address, recognisable by its `fe80::` prefix, which is present even on networks where nobody has configured IPv6 — one of the reasons an IPv6-blind firewall is a real problem.

The third line is the link layer. The `ether` field is the MAC address, the hardware identifier burned into the network card (or, on a virtual machine, assigned by the hypervisor), and `00:0c:29` identifies this one as a VMware virtual adapter. MAC addresses are discussed in detail in section 1.4, because they can be changed. After it, `txqueuelen` is the length of the transmit queue, a tuning parameter you will rarely need to touch.

The rest of the block is statistics. `RX` and `TX` count packets and bytes received and transmitted since the interface came up, the `errors`, `dropped`, `overruns`, `carrier` and `collisions` counters track things that went wrong, and they are the first place to look when a link is flaky: a steadily climbing `errors` or `dropped` count on a physical interface usually means a bad cable, a failing port or a duplex mismatch. All of these counters reset when the interface is brought down and up again.

The modern equivalent of this command is `ip addr` (often typed `ip a`), which shows the same information in a different layout:

```bash
root@kali:~# ip -br addr
lo               UNKNOWN        127.0.0.1/8 ::1/128 
eth0             UP             192.168.1.17/24 fe80::20c:29ff:fe1b:2c3d/64
root@kali:~# ip route
default via 192.168.1.1 dev eth0 proto dhcp metric 100 
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.17 metric 100 
```

The `-br` flag asks for *brief* output, one line per interface, which is what you want when you only care about the address. The second command, `ip route`, prints the routing table — the set of rules that decides where a packet goes — and the key line is `default via 192.168.1.1 dev eth0`, which names the gateway: the router that packets are sent to when their destination is not on the local subnet. Knowing the gateway is not optional knowledge, because the default route is the thing that makes the difference between a machine that can reach the internet and a machine that can only talk to its neighbours even though `ifconfig` looks perfectly healthy.

### 1.2 `iwconfig` — checking wireless network devices

If the machine has a wireless adapter, `iwconfig` is the traditional tool for reading its radio-specific settings: the network it is associated with, the access point it is talking to, the frequency, the transmission rate and the signal quality.

```bash
root@kali:~# iwconfig
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

On a machine with no wireless hardware — which is the common case on a virtual machine — every interface reports `no wireless extensions`, and that is the expected output rather than an error: Ethernet adapters have no radio settings to report. On a machine that does have a wireless card, the block above is what you get, and three fields are worth understanding. `ESSID:"LAB-AP"` is the name of the wireless network the card has joined, and `Mode:Managed` means it is behaving as a normal client. The `Access Point` field is the MAC address of the wireless router the card is associated with, which is useful for physical-location work and for detecting a rogue access point that is impersonating your network. `Link Quality=58/70  Signal level=-52 dBm` is the radio health: signal strength in dBm where values closer to zero are better, so −52 dBm is a strong, comfortable connection and anything worse than about −75 dBm will start to produce dropped packets.

The security-relevant fields are `Encryption key:off`, which tells you whether the link is using WEP, WPA or nothing at all, and the counters at the bottom. `Rx invalid crypt` counts frames that failed their integrity check, which on an open network is simply noise, but on an encrypted one can indicate interference or an attack in progress. The `iw` command is the modern replacement (`iw dev wlan0 link` prints the association details), and `nmcli device wifi list` is the higher-level tool if NetworkManager is managing the radio.

### 1.3 Changing an IP address

An interface's address can be set by hand, and doing so is a one-line command: `ifconfig`, then the interface to change, then the address you want it to have. Here the Ethernet interface is given the address 192.168.1.13.

```bash
root@kali:~# ifconfig eth0 192.168.1.13
root@kali:~# ifconfig eth0
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.13  netmask 255.255.255.0  broadcast 192.168.1.255
        ether 00:0c:29:1b:2c:3d  txqueuelen 1000  (Ethernet)
        RX packets 10430  bytes 9012611 (8.5 MiB)
        TX packets 5121  bytes 774311 (756.1 KiB)
root@kali:~# ip -br addr show eth0
eth0             UP             192.168.1.13/24 fe80::20c:29ff:fe1b:2c3d/64
```

The address changed immediately and the second command confirms it: `inet 192.168.1.13` where the previous transcript showed 192.168.1.17. This works because the kernel does not ask permission from anything — an address is local state, so assigning one takes effect the moment you type it. There is no negotiation in the manual case, no DHCP lease, and no notification to the router, which has consequences worth spelling out.

The first consequence is that the change is **temporary**. Nothing has been written to a configuration file, so a reboot — or a restart of the networking service — returns the interface to whatever its configuration says. If the machine is managed by NetworkManager or systemd-networkd, those services may overwrite your change within seconds or on the next reconnect. The second consequence is that `ifconfig eth0 <address>` does not touch the routing table, so the default route may still point out of the interface with a source address that no longer exists, and connectivity can break in a confusing way: the interface looks correctly configured yet nothing routes. The safe pattern, and the one used on machines you administer permanently, is to edit the configuration (a Netplan YAML file on Ubuntu, a connection profile under `/etc/NetworkManager/system-connections/`, or `/etc/network/interfaces` on Debian systems without NetworkManager) and then reload the networking service so that addresses, routes and DNS are all recalculated together.

If you are working over SSH when you change an address, be aware that you are almost certainly cutting the branch you are sitting on: the moment the source address of the connection changes, the session dies. The standard safety measures are to work from a console, to add the new address *alongside* the old one with `ip addr add 192.168.1.13/24 dev eth0` (the modern command, which adds rather than replaces), or to schedule a rollback with `at` — a technique we will meet in section 2.9.

### 1.4 Spoofing a MAC address

The MAC address is not only an identifier; on many networks it *is* the access control. Wireless access points with MAC filtering, captive portals that remember devices, DHCP servers that hand out fixed leases by hardware address, and network access control systems that quarantine unknown machines all make decisions on the strength of the MAC address they see. Because that value is read from the frame's headers rather than from any hardware register at receive time, a modern network card will let the operating system present whatever value you ask for. Spoofing is therefore trivial, and it neutralises every one of those controls.

Changing it takes three commands, because most drivers will not accept a new hardware address while the interface is carrying traffic: bring the interface **down**, set the address, and bring it back **up**.

```bash
root@kali:~# ifconfig eth0 down
root@kali:~# ifconfig eth0 hw ether 00:11:22:33:44:55
root@kali:~# ifconfig eth0 up
root@kali:~# ifconfig eth0
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.17  netmask 255.255.255.0  broadcast 192.168.1.255
        ether 00:11:22:33:44:55  txqueuelen 1000  (Ethernet)
        RX packets 2  bytes 168 (168.0 B)
        TX packets 3  bytes 252 (252.0 B)
```

The `hw ether` keyword means "the hardware address of type Ethernet", and the argument that follows is the new value. The verification shows `ether 00:11:22:33:44:55` in place of the original VMware address, and note what else changed: the `RX` and `TX` counters have reset to almost nothing, because downing the interface clears them and starting it again begins new ones. It is also worth noting that the interface came back with its configured IPv4 address and its routes, because those are applied by the network manager when the link comes up — a manual configuration inside a live system behaves differently from one made while the interface is down.

A few practical points. The modern equivalent is `ip link set dev eth0 address 00:11:22:33:44:55`, with `ip link set eth0 down` and `up` around it. Not every driver accepts a spoofed address, and not every address is accepted: a multicast address (the least significant bit of the first octet set), or a value whose first octet is odd in the wrong place, may be rejected with `SIOCSIFHWADDR: Cannot assign requested address`. Choose a unicast address — the second-least-significant bit of the first octet must be clear — and, for anything that needs to look legitimate, an address with a real vendor prefix.

Changing your MAC is not invisibility. On a switched network the frame is delivered by MAC, but the switch *records* which port each address appears on, so spoofing an address that another device is already using creates a flapping-port condition that switch management software will log and many will block. Spanning-tree protection features such as port security and DHCP snooping, and any decent network access control deployment, are built precisely to catch this. And if your goal is to *defeat* MAC-based access control in an assessment, the honest finding to write up is not "I spoofed a MAC" but "the network treats a spoofable identifier as an authentication factor, which constitutes no authentication at all".

### 1.5 Getting a fresh address from DHCP with `dhclient`

Manual addressing is useful for a lab, but most machines do not stand still: they ask the local DHCP server for an address every time they connect. DHCP — the Dynamic Host Configuration Protocol — hands out addresses, and usually a subnet mask, a gateway and DNS servers along with them, from a pool that the administrator controls. On a client machine the request is made by the DHCP client daemon, which on many Linux systems is `dhclient`.

```bash
root@kali:~# dhclient eth0
root@kali:~# ifconfig eth0
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.17  netmask 255.255.255.0  broadcast 192.168.1.255
        ether 00:11:22:33:44:55  txqueuelen 1000  (Ethernet)
        RX packets 421  bytes 38112 (37.2 KiB)
        TX packets 388  bytes 74520 (72.8 KiB)
```

Running `dhclient` with an interface name starts the client against that interface, and the address has now returned to `192.168.1.17` — the value the server had allocated to this interface before we typed 192.168.1.13 over the top of it. The machine is back to a server-managed state, and the previously configured address is gone from the interface although the manually assigned address may still survive in the DHCP client's own configuration file until it is rewritten.

Most of the useful detail is not printed on the console but written to the lease file, which records exactly what the server offered and when it expires:

```bash
root@kali:~# grep -A3 "lease" /var/lib/dhcp/dhclient.eth0.leases | head -20
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

That block is the client's record of the conversation: the address it holds, the mask, the gateway (`option routers`), the DNS servers it was told to use, the length of the lease in seconds, and the times at which it will try to renew, rebind and eventually lose the address. Those three times define the DHCP lifecycle and they explain a common puzzle. A lease is not permanent, so a machine that has been off for a long time may return to a different address; and because the client renews at half the lease time, an address that "suddenly changed" usually means the server had already given it to somebody else.

Two commands complete the set. `dhclient -r eth0` *releases* the lease, telling the server that the address is no longer needed so it can be returned to the pool — useful when you want to force a completely fresh configuration. `dhclient -v eth0` runs the client verbosely and prints the four-step handshake as it happens (DISCOVER, OFFER, REQUEST, ACK), which is the clearest way to understand the protocol and the fastest way to prove that a DHCP server is reachable. On modern desktop systems all of this is managed by NetworkManager or systemd-networkd, and reaching for `dhclient` by hand on such a machine leads to the two systems fighting over the interface; `nmcli con up <connection>` and `networkctl reconfigure <link>` are the polite ways to ask.

### 1.6 Examining DNS with `dig`

DNS is the service that translates a domain name such as `example.com` into the IP address a packet can actually be sent to, and it does far more than that: it publishes which servers accept mail for a domain, which name servers are authoritative for the zone, which hosts exist, and where services such as web and VPN endpoints can be found. For anyone assessing a target, DNS is reconnaissance gold, and `dig` — the Domain Information Groper — is the tool you use to read it.

Run with a single argument, `dig` performs a lookup for an A record (an IPv4 address) and prints the whole answer, including the parts that are normally hidden:

```bash
root@kali:~# dig example.com

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

Reading from the top: the `HEADER` line reports the operation (`opcode: QUERY`), the result code (`status: NOERROR`, where NXDOMAIN means the name does not exist and SERVFAIL means the resolver itself failed), and the flags, of which `qr` (this is a response), `rd` (recursion desired) and `ra` (recursion available) are the ones you will see most. The `QUESTION SECTION` repeats the question that was asked, and the `ANSWER SECTION` holds the answer: the name, the TTL in seconds (3600, meaning resolvers may cache this for an hour), the class (`IN` for internet), the record type (`A`) and the data (`93.184.216.34`).

The last four lines are the ones beginners overlook and experts read first. `Query time: 24 msec` is how long the recursive resolution took. `SERVER: 192.168.1.1#53(192.168.1.1) (UDP)` names the resolver that actually answered *you* — here the lab gateway forwarding on your behalf, and noticing when this changes is how you confirm that your resolver configuration took effect. `WHEN` is the timestamp, and `MSG SIZE rcvd` is the size of the response, a number that becomes interesting when DNS responses grow large enough to trigger TCP fallback or fragmentation.

Different record types answer different questions, and `dig` asks for them by adding the type to the command. The `mx` type lists the mail exchangers for the domain, which is step one in mapping an organisation's mail infrastructure:

```bash
root@kali:~# dig example.com mx +noall +answer
example.com.		3600	IN	MX	0 .
```

That answer is a **null MX**, and it is worth understanding rather than skipping past. The single record `0 .` — priority zero, target the root — is a deliberate, standards-defined way of saying "this domain does not accept mail at all". A resolver that finds it must stop looking, which is why the record exists: it tells the world not to bother trying. A domain that *does* accept mail returns one or more records with a hostname and a preference number. Since the reserved domain used above deliberately accepts no mail, here is the shape of a mail-accepting answer taken from a lab zone in the `.example` top-level domain, which is reserved for exactly this purpose and therefore cannot collide with anything real:

```bash
root@kali:~# dig intranet.example mx +noall +answer
intranet.example.	300	IN	MX	10 mail1.intranet.example.
intranet.example.	300	IN	MX	20 mail2.intranet.example.
```

Here two mail servers are published, `mail1` preferred over `mail2` because the lower number wins, which is how a domain load-balances inbound mail and provides failover. Both hostnames are useful to an assessor: each is a machine that accepts untrusted input from the internet, and each must be resolved further before you know what you are dealing with.

The `ns` type names the authoritative name servers for the zone, which are the machines that hold the definitive copies of every record in it:

```bash
root@kali:~# dig example.com ns +noall +answer
example.com.		172800	IN	NS	a.iana-servers.net.
example.com.		172800	IN	NS	b.iana-servers.net.
```

For an assessment these are the servers you would query directly (`dig @a.iana-servers.net example.com AXFR`) on the off-chance that they permit a zone transfer, which would hand over the entire contents of the DNS zone — every hostname, every mail server, every subdomain the organisation has ever registered — in one request. Modern servers refuse it, but the ones that do not remain one of the best findings available.

A few shortcuts make `dig` pleasant to use in scripts and in a hurry. `+short` prints only the answer data, so `dig +short example.com` returns a single line that is trivial to feed into another command. `+noall +answer` drops the header and status blocks when you only want the records. `@1.1.1.1` sends the query to a specific resolver instead of the system default, which is how you compare what different resolvers see — and the two answers are not always the same, because an internal resolver may know names that public DNS does not, while a public one may be willing to answer for names that internal filtering hides. `-x` performs a reverse lookup, turning an address into a name (`dig -x 93.184.216.34`). And `+trace` follows the resolution step by step from the root servers down, which is the single best way to *learn* how DNS actually works rather than just reading about it.

### 1.7 Changing your DNS server

The resolver your machine uses by default is described in a plain text file, `/etc/resolv.conf`, and it is worth reading before you change it:

```bash
root@kali:~# cat /etc/resolv.conf
# Generated by NetworkManager
search lab.local
nameserver 192.168.1.1
```

Two directives appear in a typical file. The `nameserver` line names the resolver to ask — here the lab gateway, which forwards queries upstream — and the `search` line lists domains that are appended to unqualified names, which is why typing `ping fileserver` can work on a corporate network without a fully qualified name. A file may list up to three `nameserver` lines, and they are consulted in order: the resolver asks the first one, and moves down the list only if the one above does not answer in time. That ordering is why a stale entry at the top of the list causes the mysterious pauses everybody eventually experiences.

Changing the resolver is a matter of writing a new line into the file. The quickest way is with the shell's `echo` and the `>` redirect, which *overwrites* the file with whatever you supply:

```bash
root@kali:~# echo "nameserver 1.1.1.1" > /etc/resolv.conf
root@kali:~# cat /etc/resolv.conf
nameserver 1.1.1.1
root@kali:~# dig example.com +short @1.1.1.1
93.184.216.34
root@kali:~# dig example.com | grep "SERVER:"
;; SERVER: 1.1.1.1#53(1.1.1.1) (UDP)
```

The `>` operator is destructive and there is no undo, which is why it is worth noting that `cat /etc/resolv.conf` came *before* the change: with a single-line file it hardly matters, but the same habit applied to a configuration file you have not read is how people destroy settings they cannot reconstruct. After the write, the file contains exactly one line — the public resolver operated by Cloudflare at 1.1.1.1, which is easy to remember and widely used; Google's public resolver at 8.8.8.8 and Quad9 at 9.9.9.9 are the other two you will meet most often, and every one of them has a published privacy policy that is worth reading before you make it your resolver. The second `dig` confirms the change took effect in the way that matters: the `SERVER:` line now reports `1.1.1.1#53` instead of the local gateway, which proves that queries are actually leaving through the new resolver rather than merely that the file's contents changed.

There is one important caveat and it catches everybody. On most modern systems `/etc/resolv.conf` is *generated*, not authored: NetworkManager, `dhclient`, `systemd-resolved` or `resolvconf` rewrites it whenever a network event occurs, and may even replace the file with a symbolic link pointing at a runtime directory. A hand-edited file therefore survives only until the next DHCP lease renewal or reboot. The durable approaches are to set the DNS server in the tool that owns the file — `nmcli con mod <name> ipv4.dns 1.1.1.1` for NetworkManager, a Netplan `nameservers:` stanza on Ubuntu Server, or `/etc/systemd/resolved.conf` followed by `systemctl restart systemd-resolved` — or, for a machine you control completely, to make the file immutable with `chattr +i /etc/resolv.conf`. The last option works and is sometimes the right answer on a lab machine, but it is a trap on anything else: every later network change will fail, and the next administrator will spend an afternoon wondering why.

For security work the resolver matters for two reasons. First, the DNS server sees every name you look up, so on an untrusted network a rogue resolver reads your reconnaissance and can answer with forged addresses. Second, using a resolver that validates DNSSEC or filters malicious domains changes what you can reach, which is why assessments often begin by identifying which resolvers a target environment uses and whether those queries are encrypted with DNS-over-HTTPS or DNS-over-TLS.

### 1.8 Mapping hostnames in `/etc/hosts`

Before any DNS query is sent, the system consults a local file. `/etc/hosts` is a static table of name-to-address mappings, and every Unix-like system has one; it is consulted first because of the lookup order defined in `/etc/nsswitch.conf`, where the `hosts:` line reads `files dns`, meaning "check the files first, then ask DNS".

```bash
root@kali:~# cat /etc/hosts
127.0.0.1	localhost
127.0.1.1	kali
::1		localhost ip6-localhost ip6-loopback

# The following lines are desirable for IPv6 capable hosts
ff02::1		ip6-allnodes
ff02::2		ip6-allrouters
root@kali:~# nano /etc/hosts
root@kali:~# cat /etc/hosts
127.0.0.1	localhost
127.0.1.1	kali
::1		localhost ip6-localhost ip6-loopback

# The following lines are desirable for IPv6 capable hosts
ff02::1		ip6-allnodes
ff02::2		ip6-allrouters
192.168.1.9	portal.example.com
```

The file's format is deliberately primitive: an IP address in the first column, one or more names after it separated by whitespace, and a `#` starting a comment. The default entries map `localhost` to the loopback addresses and give the machine its own hostname (`kali`) at `127.0.1.1`, which is a Debian-family convention that keeps the hostname resolvable even with no network at all. The `::1` line shows that several names can be attached to the same address, which is how aliases are created.

Adding a line maps any name of your choosing to any address, and because the file wins over DNS, the effect is immediate and global to the machine:

```bash
root@kali:~# getent hosts portal.example.com
192.168.1.9     portal.example.com
root@kali:~# curl -s -o /dev/null -w "%{http_code} %{remote_ip}\n" http://portal.example.com/
200 192.168.1.9
root@kali:~# dig +short portal.example.com
93.184.216.34
```

The three commands show the override from three angles, and the third is the interesting one. `getent hosts` asks the system's own resolver — the same mechanism applications use — and reports the address from the hosts file. `curl` then connects to `192.168.1.9` while the URL still says `portal.example.com`, which is exactly the kind of redirection this file is used for. But `dig` bypasses `/etc/hosts` entirely, because `dig` speaks to DNS servers directly, and it still returns the real, public address. That discrepancy is a wonderful teaching example of the difference between *the system's view* and *DNS's view*, and it is also the reason a name that works in the browser can fail in a script that uses `dig` or `nslookup` — the two are answering different questions.

Legitimate uses of the file are everywhere: giving a lab machine a friendly name before DNS exists, testing a new web server before its DNS record is changed, pinning a hostname so that an internal application stops resolving to a wrong address, and blocking a domain by pointing it at `127.0.0.1` or `0.0.0.0`. The illegitimate use is what makes it relevant here, and it is the same mechanism in reverse: because the file is local and trusted, an attacker who can write to it — or who can run a rogue DHCP or DNS server that answers first — can send a user to a server of their choosing. That is the basis of DNS and ARP spoofing attacks, where a user types the correct address and lands on an attacker's copy of the site. The defensive answers follow from the mechanism: monitor the integrity of `/etc/hosts` and `/etc/resolv.conf` on servers, prefer DNSSEC and DNS-over-HTTPS where the environment supports them, and treat any user-visible redirect to an unexpected address as an incident rather than a glitch.

---

## 2. Process management

A process is simply a program that is running: the kernel has loaded it into memory, given it an identifier, scheduled it onto a CPU and accounted for the resources it uses. On a single-user laptop you might have two hundred processes running at any moment, most of them daemons — background services with no terminal attached — and on a server the number is larger still. Knowing how to list them, read what they are consuming, decide which of them deserve more of the machine, stop the ones that have gone wrong, and move the ones that should be out of the way is the difference between administering a system and guessing at it. For security work there is a second reason to learn this material, since the same commands are how you find the process that is listening on a port, how you identify a suspicious daemon among legitimate ones, and how you stop an anti-virus agent that is interfering with an authorised test.

### 2.1 `ps` — viewing your own processes

The primary tool for looking at processes is `ps`, short for *process status*. Typed with no arguments it lists the processes attached to the current terminal and owned by the current user, which keeps the output short enough to be useful while you are learning.

```bash
root@kali:~# ps
    PID TTY          TIME CMD
   4122 pts/0    00:00:00 bash
   4188 pts/0    00:00:00 ps
```

Only two processes appear, and that is not an error: `ps` is printing the processes that belong to this shell session, which is you plus the `ps` command itself. The three columns are the vocabulary the rest of the section builds on. `PID` is the process identifier, a number the kernel assigns to every process it creates and which is unique among the processes running at that moment — it is the handle you use to inspect, prioritise or kill anything. `TTY` names the terminal the process is attached to (`pts/0` is the first pseudo-terminal, which is what your terminal emulator gives you), and a `?` in that column means the process has no terminal at all, which is the signature of a daemon. `TIME` is the total CPU time the process has consumed since it started, not the wall-clock time it has existed — a distinction that matters, because a process that has been running for a week and used two seconds of CPU is behaving very differently from one that has used two hours.

Two useful variations on the basic command are worth trying immediately. `ps -f` adds a column showing the parent process ID (PPID) and the full command line, which begins the process of answering "who started this, and with what arguments?". And `ps -ef` — the System V style that administrators of other Unix systems reach for out of habit — shows every process on the system with the same detail, which brings us to the next command.

### 2.2 `ps aux` — every process, every user

Running `ps` with the three BSD-style options `aux` displays all running processes for all users along with the resource figures for each one. This is the single most typed command in this section, and the header line it prints is worth memorising.

```bash
root@kali:~# ps aux
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.6 168412 12984 ?        Ss   09:41   0:04 /sbin/init splash
root           2  0.0  0.0      0     0 ?        S    09:41   0:00 [kthreadd]
root         812  0.0  0.1  15420  9216 ?        Ss   09:41   0:00 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
root        1044  0.0  0.4 240128 34220 ?        Ssl  09:41   0:01 /usr/lib/snapd/snapd
www-data    1204  0.1  1.1 3310420 88144 ?       Ssl  09:44   0:06 python3 /opt/portal/app.py
student     3901  0.0  0.1  10344  5120 pts/0    Ss   10:12   0:00 -bash
student     4122  0.0  0.0  12440  4096 pts/0    R+   10:31   0:00 ps aux
```

Each column answers a different question. `USER` is the account the process is running as, which is the first thing to check when you are deciding what you are allowed to do to it: a process owned by your own user can be signalled by you, while anyone else's needs root. `PID` is the identifier. `%CPU` and `%MEM` are the percentage of total CPU and memory the process is currently using, and `VSZ` and `RSS` are the memory figures in kilobytes — `VSZ` is the virtual size of the address space the process has reserved, while `RSS` is the resident set size, the memory it actually has in physical RAM right now. `VSZ` is usually much larger than `RSS` and is the more misleading of the two, because reserving address space costs nothing. `TTY` is the terminal, with `?` for daemons, and `STAT` is a short code describing the process state.

The `STAT` codes are worth learning because they tell you whether a process is healthy. `R` means running or runnable — on the CPU or waiting its turn. `S` means interruptible sleep, the normal state of a process waiting for something such as a network packet or a keystroke; almost everything on a quiet system is in `S`. `D` is uninterruptible sleep, which usually means blocked on disk or network I/O, and a process stuck in `D` cannot be killed even with `SIGKILL` because it is not running any code that could handle the signal — a diagnosis, not an obstacle. `T` means stopped, which is what `Ctrl+Z` leaves behind. `Z` means zombie: the process has finished but its parent has not yet collected its exit status, so a row remains in the table doing nothing. Zombies consume no CPU and no memory beyond the table entry, they cannot be killed, and they disappear when the parent reads the exit status or exits itself.

`START` is the time the process began, `TIME` is cumulative CPU time and `COMMAND` is the command line — with the additional detail that a command wrapped in square brackets at the start is a kernel thread rather than a user process, which is why `[kthreadd]` has no memory and no terminal. A few variations earn their place in your notes: `ps aux --sort=-%mem | head` lists the biggest memory consumers, `ps -eo pid,ppid,user,%cpu,cmd --sort=-%cpu | head` picks the columns you care about and sorts by CPU, and `ps -ef --forest` draws the process tree, showing which daemon spawned which worker — a view that turns "there are twelve nginx processes" into "there is one master and eleven workers".

### 2.3 Filtering processes by name

A busy machine produces hundreds of rows from `ps aux`, so in practice you filter. The technique is the pipe introduced in Part 1 (section 4.7), applied here to process output, and it works exactly as it does with any other command: pipe `ps aux` into `grep` and keep the lines that mention the program you are curious about. Here we look for metasploit's console, `msfconsole`.

```bash
root@kali:~# ps aux | grep msfconsole
root        5210  1.3  2.4 612344 198772 pts/1   Sl+  10:33   0:02 ruby /usr/bin/msfconsole -q
root        5233  0.0  0.0  12440  4096 pts/1    S+   10:33   0:00 grep msfconsole
```

Two lines come back, and understanding why is a small but genuinely useful lesson. The first is the real process: a Ruby interpreter running the `msfconsole` script, consuming 1.3% of CPU and about 2.4% of memory. The second line is the `grep` command itself, which matched its own command line — a permanent piece of bycatch that confuses beginners into thinking there are two processes. The standard ways to get rid of it are to exclude the search term from the results (`ps aux | grep msfconsole | grep -v grep`) or to bracket one character of the pattern (`ps aux | grep "[m]sfconsole"`), since the bracket expression matches the real process but does not appear literally in the grep process's own command line.

There is a better tool for this job, and it is worth adopting as the default: `pgrep` was written to answer exactly the question "which processes match this name?", so it does not match itself and it prints only identifiers by default.

```bash
root@kali:~# pgrep -a msfconsole
5210 ruby /usr/bin/msfconsole -q
root@kali:~# pgrep -u student -l bash
3901 bash
root@kali:~# pgrep -f "app.py"
1204
```

The `-a` flag prints the full command line alongside the PID, `-u` filters by user, `-l` prints the name only, and `-f` matches against the whole command line rather than just the executable name — which is what you need for anything started as `python3 /opt/portal/app.py`, because the process name there is only `python3`. Its companion, `pkill`, sends a signal to every process that matches, and we will use it later in this section once the concept of signals has been introduced.

### 2.4 `top` — finding the greediest process

`ps` gives you a snapshot; `top` gives you a film. The `top` command displays processes ordered by resource consumption and refreshes the display every few seconds, which makes it the tool of choice when you want to watch what a machine is doing right now — which process is eating the CPU, whether the load is rising, and how much memory is left.

```bash
root@kali:~# top
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

The five header lines summarise the whole machine and are worth reading before the process list. The first line gives the time, the uptime, the number of logged-in users and the three **load averages**, which report the average number of runnable processes over the last one, five and fifteen minutes. A load average above the number of CPU cores means work is queueing — on this single-core-equivalent virtual machine, 0.42 means the machine is comfortable. The second line counts processes by state, and any non-zero `zombie` or `stopped` count is worth following up. The third line breaks CPU time down by category: `us` is user-space work, `sy` is kernel work, `id` is idle, `wa` is time waiting on I/O (a rising `wa` usually means a slow disk), and `st` is time *stolen* by the hypervisor — the figure that tells you another virtual machine on the same host is eating your CPU. The fourth and fifth lines describe memory: total, free, used, and the page cache sitting in `buff/cache`, which is memory the kernel is using to cache disk reads and will give back instantly if an application needs it. Reading `free` without reading `buff/cache` is the classic beginner error, since Linux deliberately uses otherwise-idle RAM as a cache: a machine with almost no `free` memory and a large `buff/cache` is working exactly as designed.

Below the header comes the process list, which repeats several `ps` columns with two additions. `PR` is the kernel's priority and `NI` is the nice value discussed in the next subsection, and both are in the list because they are the knobs you turn when a process needs more or less of the CPU than the others.

`top` is interactive, and half a dozen keys carry most of its value: `P` sorts by CPU usage (the default), `M` sorts by memory, `k` prompts for a PID and a signal so that you can kill a process without leaving the display, `r` prompts for a PID and a new nice value so that you can re-prioritise one, `1` expands the CPU line to show each core individually, `h` shows the full help, and `q` quits. If you would rather capture the output than watch it, `top -b -n 1 | head -20` runs it once in batch mode and prints the result, which is the form to use in a script or a report. And if you spend much time doing this, install `htop`: it is the same information with colour, mouse support and scrollable columns.

### 2.5 `nice` — setting the priority of a new process

On a machine with more work than CPU, the kernel scheduler decides who runs, and it makes that decision partly on the basis of each process's **nice value** — a number from −20 to 19 that expresses how willing the process is to give way to others. The scale runs backwards from intuition: −20 is the highest priority (least nice, most demanding), 19 is the lowest (extremely nice, always yielding), and the default is 0. A process started with `nice` carries the value you give it for its entire life, which is why the setting is applied at launch.

The `-n` flag introduces the adjustment, and here the ssh-agent daemon is started with a nicer-than-default value of −10, meaning it should be given preference by the scheduler.

```bash
root@kali:~# nice -n -10 /usr/bin/ssh-agent
SSH_AUTH_SOCK=/tmp/ssh-XXXXXXuP3n9g/agent.4412; export SSH_AUTH_SOCK;
SSH_AGENT_PID=4412; export SSH_AGENT_PID;
echo Agent pid 4412;
root@kali:~# ps -o pid,ni,cmd -C ssh-agent
    PID  NI CMD
   4412 -10 /usr/bin/ssh-agent
```

The program started normally — a process identifier was assigned, and the shell printed the environment variables a caller needs in order to talk to the agent. The verification command is the interesting part: `ps -o pid,ni,cmd -C ssh-agent` selects specific columns and names the process to match, and the `NI` column shows `-10`, confirming that the priority request was accepted. `-C` (match by command name) is a tidy alternative to the `grep`-and-pipe ritual of section 2.3 and it has the pleasant property of not matching itself.

One permission rule matters here: any user may make a process *nicer* (raise its number, give way more), but only root may lower it (make a process more demanding). If a normal user tries the command above, the shell replies `nice: cannot set niceness: Permission denied` and no process starts. That asymmetry is deliberate, because the whole point of the mechanism is to protect interactive work from being starved by background jobs, and it would be defeated if any user could claim the front of the queue.

The non-privileged and far more common use is the opposite direction — pushing a long, unimportant job out of the way so that it does not compete with the work you are actually doing:

```bash
root@kali:~# nice -n 19 ./backup-script.sh &
[1] 5487
root@kali:~# nice -n 19 /usr/bin/updatedb &
[2] 5490
```

Both jobs are running at the lowest possible priority, and the `[1]` and `[2]` job numbers with the PIDs after them are the shell's acknowledgement — the topic of section 2.8. This pattern is worth remembering, because "run this in the background at minimum priority" is the correct shape for almost every maintenance task on a machine someone else is using.

### 2.6 `renice` — changing the priority of a running process

The priority of a process that is already running can be changed without restarting it, using `renice`. It takes a nice value or an adjustment and one or more PIDs, and it works on processes you did not start, provided you have the privileges to change them.

```bash
root@kali:~# renice 20 6242
6242 (process ID) old priority 0, new priority 20
root@kali:~# ps -o pid,ni,cmd -p 6242
    PID  NI CMD
   6242  20 /usr/bin/updatedb
root@kali:~# renice -n -5 -p 6242
renice: failed to set priority for 6242 (process ID): Permission denied
```

The report line is unambiguous and names both values, which makes `renice` pleasantly self-documenting: the process went from a nice value of 0 to 20, meaning it now yields to everything else on the machine. A later `ps` confirms the new value. The third command demonstrates the permission rule from the previous subsection from the other side: as an unprivileged user, lowering the nice value (here to −5, which would raise the process's priority) is refused outright, with the process left at its previous value.

Three forms are worth keeping in your notes. `renice -n 10 -p 6242` sets an absolute value using the explicit flags, which is the form to prefer in scripts because it is unambiguous. `renice -u student` changes the priority of every process owned by a user, which is a good way to quiet a runaway build account without hunting for PIDs. And `renice 5 -g developers` applies to a process group. On a modern systemd machine there is a second, complementary mechanism — cgroups and the `systemd-run --slice` and `systemctl set-property` commands let you cap CPU and memory for a whole service rather than just hinting at the scheduler, and that is the approach to use when the requirement is "this service may never use more than one core" rather than "this service should be polite".

### 2.7 `kill` — stopping processes with signals

The `kill` command stops processes, and its name misleads in a useful way: what it actually sends is a *signal*, a numbered, asynchronous notification delivered to a process by the kernel, and the process may handle most signals however it likes — clean up, save state, close sockets, ignore them or exit. There are 64 signals in all, and the two that matter most are the polite one and the fatal one. Here the process with PID 6242 is sent signal number 1.

```bash
root@kali:~# kill -1 6242
root@kali:~# ps -p 6242
    PID TTY          TIME CMD
root@kali:~# kill -l
 1) SIGHUP	 2) SIGINT	 3) SIGQUIT	 4) SIGILL	 5) SIGTRAP
 6) SIGABRT	 7) SIGBUS	 8) SIGFPE	 9) SIGKILL	10) SIGUSR1
11) SIGSEGV	12) SIGUSR2	13) SIGPIPE	14) SIGALRM	15) SIGTERM
...
root@kali:~# kill -9 4378
```

`SIGHUP`, signal 1, originally meant "the terminal you were connected to has hung up", and daemons such as `sshd`, `nginx` and `rsyslog` reinterpret it as "re-read your configuration files and reopen your log files" — which is why `kill -1` is the traditional way to make a service reload without dropping its connections, and why the same signal can terminate a simple foreground program that has no handler for it. Because `updatedb` is a short-lived program with no handler, sending it `SIGHUP` ended it, and the following `ps` shows an empty table.

`kill -l` lists every signal the system knows, and the numbers are worth learning in pairs. Signal 15, `SIGTERM`, is the *default* when you type `kill` with no number, and it is the correct first attempt for any process: it politely asks the program to terminate, and a well-written program will flush its buffers, remove its temporary files and close its sockets on the way out. Signal 9, `SIGKILL`, cannot be caught, blocked or ignored — the kernel removes the process on the spot. It is the right tool for a process that has ignored `SIGTERM` and the wrong tool for everything else, because `SIGKILL` gives the program no chance to clean up, which is how databases are corrupted and stale lock files are left behind. Signal 2, `SIGINT`, is what `Ctrl+C` sends from the terminal; signal 3, `SIGQUIT`, is `Ctrl+\`; signal 18 stops a process and 19 resumes it, which is what `Ctrl+Z` and `bg` do under the hood.

That ordering — `SIGTERM` first, then escalate — is the professional habit and it is worth enforcing on yourself from the first day. A process that does not die to `SIGTERM` is telling you something: it may be blocked in uninterruptible I/O, it may be a zombie, or it may be deliberately ignoring the signal, and each of those deserves a moment's thought before you reach for `-9`. A quick sanity check for the zombie case is `ps -o pid,stat,cmd -p <pid>`, where a `Z` in the state column means no signal will ever work, because there is no running process left to receive it.

Two related commands make targeted killing easier. `pkill` sends a signal to every process matching a pattern, so `pkill -f app.py` stops the web application by its command line and `pkill -u student -TERM -f backup` stops a user's backup jobs without touching anything else. `killall` does the same by exact program name. Both are powerful enough to be dangerous — a pattern that is too broad will match something you did not intend, and `pkill -9 -f python` on a busy machine can take out a dozen innocent processes — so the safe workflow is always to run `pgrep` first, look at what would be affected, and only then send the signal with the same pattern.

### 2.8 Background jobs, `jobs`, `fg` and `nohup`

A command typed at a prompt owns the terminal until it finishes, which is inconvenient the moment you start something long-running. The shell provides **job control** for exactly this problem: appending a single ampersand to a command runs it in the background, immediately returning your prompt while the process keeps working.

```bash
root@kali:~# nano lab-notes.txt &
[1] 4312
root@kali:~# jobs
[1]+  Stopped                 nano lab-notes.txt
root@kali:~# fg %1
```

The shell replied with the job number `[1]` and the process identifier `4312`, and in a terminal application this is where an important detail appears. `nano` is an interactive full-screen program, and a program in that category will normally *stop itself* when it finds it has been put in the background without a terminal — which is what the `jobs` output shows: the job is listed as `Stopped` rather than `Running`, because the editor tried to read from a terminal it no longer controls. This is the general rule and it is worth internalising: backgrounding is for programs that do not need your keyboard, such as a compiler, a download, a scan or a long copy — not for interactive editors.

```bash
root@kali:~# ./backup-script.sh &
[2] 4701
root@kali:~# jobs -l
[1]+  4312 Stopped                 nano lab-notes.txt
[2]-  4701 Running                 ./backup-script.sh &
root@kali:~# bg %1
```

With a genuinely non-interactive program the behaviour is what you would hope: job `[2]` is `Running`, and `jobs -l` shows both jobs with their PIDs and — importantly — the marker in the second column. The `+` marks the *current* job, the one `fg` and `bg` act on when you do not name one, while `-` marks the previous job; that is why `fg` with no argument resumes whatever was most recently backgrounded, and why it is safer to name the job explicitly until the idea is second nature.

The full vocabulary of job control is small. `Ctrl+Z` *stops* the foreground process and returns you to the prompt, which is a pause rather than an exit. `bg %1` resumes the stopped job `1` in the background. `fg %1` brings it back to the foreground, and it then owns the terminal again. `jobs` lists them, `kill %1` sends a signal to a job by number rather than by PID, and `disown %1` removes a job from the shell's table so that it is not sent `SIGHUP` when the shell exits.

That last point explains the most common way people lose work. When a terminal closes, the shell sends `SIGHUP` to its jobs, and any process that does not handle it dies — so a download started with a plain `&` in a terminal window that you then close simply disappears. The traditional defence is `nohup`, which detaches the process from the hangup signal and redirects its output, because there is no longer a terminal to print it to:

```bash
root@kali:~# nohup ./scan.sh > scan.log 2>&1 &
[3] 4820
root@kali:~# ps -o pid,stat,cmd -p 4820
    PID STAT CMD
   4820 S    /bin/bash ./scan.sh
root@kali:~# nohup ./scan.sh &
[4] 4833
nohup: ignoring input and appending output to 'nohup.out'
```

The redirection `> scan.log 2>&1` sends both standard output and standard error into a file, which is why the first invocation is silent. Omit it, as in the second invocation, and `nohup` writes the output to `nohup.out` in the current directory and says so — the message appears on your terminal *after* the job line because it is written to standard error while the shell is still printing the job number. The process now survives the terminal, and the `STAT` column shows it sleeping normally. For anything you will want to come back to, a terminal multiplexer such as `tmux` or `screen` is the better tool, because it keeps a whole session alive on the server and lets you reattach to it from anywhere, with scrollback and multiple windows; `nohup` remains the lightweight answer when you only need one command to outlive your session.

### 2.9 Scheduling a process with `at`

Sometimes the right moment to run a job is in the future. The `at` command schedules a command to run **once**, at a specified time, and it is the tool to reach for when you want a task performed after you have gone home or after some other event has happened. It reads the commands to run from standard input, which is why the invocation looks interactive.

```bash
root@kali:~# at 9:00pm
warning: commands will be executed using /bin/sh
at> /root/simple_bash.sh
at> <EOT>
job 3 at Sat Oct 10 21:00:00 2026
```

The first line is a warning rather than a complaint, and it is worth heeding: `at` runs its commands with `/bin/sh`, not with your interactive shell, so anything that depends on bash-specific syntax, aliases or your environment variables will not work — write the job as you would write a script, with full paths. The `at>` prompt then accepts commands one per line, and pressing `Ctrl+D` — which the shell displays as `<EOT>`, for end of transmission — ends the job. The confirmation line reports the job number and the exact time it will run, and `at` interprets a variety of human formats: `at 9:00pm`, `at 14:30`, `at now + 10 minutes`, `at midnight`, `at 09:00 tomorrow`. Times without a date are taken as today if still in the future and otherwise tomorrow, which occasionally surprises people at 23:59.

The queue of pending jobs is managed with three short commands. `atq` lists them, `at -c <job>` prints the entire environment and command that a scheduled job will run — the definitive way to debug one that misbehaves — and `atrm <job>` removes it. The service behind the command is a daemon called `atd`, and on a minimal system it may not be running; `systemctl status atd` will say so, and `systemctl enable --now atd` fixes it. Access to `at` is controlled by `/etc/at.allow` and `/etc/at.deny`, the same allow-and-deny pattern used by `cron`, which is how administrators keep ordinary users from scheduling work on production machines.

For jobs that should run **repeatedly** — every night, every hour, every Monday — the tool is `cron` rather than `at`, and the difference is worth stating plainly: `at` is for one-offs, `cron` is for recurrence. A user's scheduled jobs live in a crontab, edited with `crontab -e`, and each line contains five time fields followed by the command to run:

```bash
root@kali:~# crontab -l
# m h  dom mon dow   command
  0 3  *   *   *     /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
 30 1  *   *   0     /usr/local/bin/weekly-report.sh
```

Reading the fields in order: minute, hour, day of month, month, day of week, where `*` means "every". So the first line runs the backup at 03:00 every day, and the second runs the weekly report at 01:30 every Sunday (day-of-week 0). The system-wide equivalents live in `/etc/crontab` and `/etc/cron.d/`, with the addition of a user field, while the directories `/etc/cron.hourly`, `/etc/cron.daily`, `/etc/cron.weekly` and `/etc/cron.monthly` accept plain scripts with no time fields at all. Modern distributions increasingly use systemd timers instead, which offer the same scheduling with better logging and the ability to trigger on events rather than only on the clock — `systemctl list-timers` shows what is scheduled on such a system.

For security work, scheduled jobs matter for two reasons. Defensively, `cron` is a classic persistence mechanism, so anything scheduled that you do not recognise deserves attention: a job that downloads and runs a script, or that reappears after being removed, is a finding. Offensively, `at` is the reliable way to schedule a rollback when you are about to make a change that might cut off your own access — a firewall reload, a network reconfiguration, an SSH daemon restart — since a job armed before the change will restore the old state even if your session dies with it. That single technique has rescued more remote administration sessions than any other habit in this section.

---

## 3. User environment variables

Variables are simply named values — key-value pairs held by a running shell — and understanding them is a must for getting the most out of a Linux system, because a great deal of what the shell and the programs it launches consider to be "the environment" is nothing more than a list of them. There are two families, and the distinction matters. A **shell variable** exists only inside the shell that created it; it is invisible to programs the shell starts. An **environment variable** has been *exported*, which means it is copied into the environment of every child process, and therefore into every program you run from that shell and every process those programs start in turn. Variables are inherited down the process tree and never flow back up, which explains most of the confusing behaviour beginners meet: a child process can read what you exported, but it cannot change what its parent sees.

The convention for names is upper case — `HOME`, `PATH`, `USER` — which is a convention rather than a rule, but a valuable one, because `HISTSIZE` and `histsize` are two entirely different variables and only one of them is the one the shell uses.

### 3.1 Viewing all the variables

Two commands list variables, and the difference between them is the distinction above made concrete. `set`, typed with no arguments, prints every shell variable and function the current shell knows about, including all the environment variables it inherited. `env` prints only the exported environment, which is the list that child processes will actually receive.

```bash
root@kali:~# set | more
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

The pipe into `more` is not optional in practice: on a normal desktop `set` prints several hundred lines, most of them shell-internal variables such as `BASH_ARGC` that you will never use. Among the noise, a handful of familiar names appear — `HOME` is your home directory, `HOSTNAME` is the machine's name, `HISTFILE` is the file your command history is written to, and `HISTSIZE` is the number of commands the shell keeps in memory to offer you when you press the up arrow. Because the output is ordinary text going to standard output, it can be filtered, sorted and folded like anything else, which is exactly what the next subsection does.

### 3.2 Filtering for a particular variable

Piping `set` into `grep` is the way to find one variable among hundreds, and the technique is identical to the process filtering of section 2.3. Here we look for `HISTSIZE`.

```bash
root@kali:~# set | grep HISTSIZE
HISTFILESIZE=2000
HISTSIZE=1000
root@kali:~# echo "my history holds $HISTSIZE commands"
my history holds 1000 commands
root@kali:~# printenv HISTSIZE

root@kali:~# printenv HOME
/root
```

Three commands, and the third one's silence is the most instructive thing in this subsection. The `grep` approach prints both history-related variables in the `name=value` form the shell keeps them in, and the `echo` form embeds the value in a sentence while introducing the sigil that matters: **`$` before a name means "substitute the value"**. Drop the dollar sign and you print the letters of the name back at yourself, which is the single most common mistake in this whole section.

`printenv` prints *only the value* of a variable, which is what you want inside a script — and here it printed nothing at all. That is not a mistake: `HISTSIZE` is set, as the first two commands prove, but it has never been **exported**, so it exists only inside this shell and the `printenv` process — a child of the shell — cannot see it. The exit status tells the same story, returning 1 for a variable that is not in the environment. `printenv HOME` works, and prints `/root`, because `HOME` *is* exported and therefore is handed to every process the shell starts. There is no better demonstration of the difference between a shell variable and an environment variable, and it explains the behaviour of every program you will ever wonder about: if a tool claims not to know a value that is plainly set in your terminal, the variable was never exported.

So the answer to the original question is that this shell remembers one thousand commands. That is comfortable for interactive work, and it is also a fact with security implications worth noting: your history file is a record of everything you have typed, including passwords passed as command-line arguments and hosts you have connected to, which is why `~/.bash_history` is among the first files an attacker reads on a compromised machine — and why clearing it, or disabling history for sensitive commands with a leading space, is part of an operator's routine.

### 3.3 Changing a value temporarily

Changing a variable is a matter of assigning it a new value, and the assignment is written with no spaces around the equals sign.

```bash
root@kali:~# HISTSIZE=0
root@kali:~# echo $HISTSIZE
0
root@kali:~# echo $HISTSIZE > ~/valueofHISTSIZE.txt
root@kali:~# cat ~/valueofHISTSIZE.txt
0
```

The first line sets the variable to zero and the second confirms it, and the effect is exactly what that value implies: the shell no longer keeps any commands in memory for the current session, so pressing the up arrow produces nothing and history recall stops working until you change the value back. It is a small demonstration of a large truth about configuration, which is that a harmless-looking one-character change can quietly remove a convenience you depend on without any error message at all.

Two details in this transcript are worth pausing on, because they are where beginners lose time. The first is the spacing rule: `HISTSIZE=0` assigns, while `HISTSIZE = 0` is interpreted by the shell as an attempt to run a command called `HISTSIZE` with two arguments, which fails with `HISTSIZE: command not found`. The equals sign must be glued to the name and to the value, and no other arrangement works. The second detail is the redirect in the third command. The intent — and a habit worth adopting — is to *save the old value somewhere before changing it*, so that you can always put things back: `echo $HISTSIZE > ~/valueofHISTSIZE.txt` writes the current value into a file in your home directory, and `cat` confirms what was written. Note the `>`, which sends the output to the file; without it, the command would print the value to the terminal and save nothing, which is a mistake easy to make and easy to detect, since the redirect creates the file as a side effect. In this case we saved the already-changed value of 0, so the file now records the experiment rather than the default — a reminder to save *before* you edit, not after.

### 3.4 Making the change permanent with `export`

The assignment above lasts as long as the session does. Opening a new terminal gives you a fresh shell with a fresh environment, and `HISTSIZE` is back to 1000 because the value was never written down anywhere. To make a variable persistent you must first place it into the environment with `export`, and then arrange for the assignment to happen in every new shell — which is done by adding the line to a startup file such as `~/.bashrc`.

```bash
root@kali:~# HISTSIZE=0
root@kali:~# export HISTSIZE
root@kali:~# bash -c 'echo "child shell sees HISTSIZE=$HISTSIZE"'
child shell sees HISTSIZE=0
root@kali:~# bash -c 'echo "child shell sees HISTFILE=$HISTFILE"'
child shell sees HISTFILE=
root@kali:~# echo 'export HISTSIZE=0' >> ~/.bashrc
```

The first two commands are the two-step ritual: assign the value, then mark the variable for export. Order matters — exporting a name that has no value yet exports nothing — and `export HISTSIZE=0` in a single command is the compact form that does both at once. The third command proves the point, and it is the demonstration that makes the shell-versus-environment distinction concrete: `bash -c` starts a *new* shell, and because `HISTSIZE` was exported, that child process inherited the value 0 and printed it. The fourth command is the control experiment, asking the same child about `HISTFILE`, which was never exported — the child reports an empty value even though the parent shell has it set. Nothing about that is a bug; it is the inheritance rule doing exactly what it says.

The final line makes the change survive future sessions by appending to `~/.bashrc`, the script that bash runs each time it starts an interactive shell. The `>>` operator appends rather than overwrites, which is the correct choice for startup files that already contain configuration; using `>` here would erase whatever was there, including the lines that define your prompt and aliases. Which file to edit depends on the shell and the moment: `~/.bashrc` for interactive bash shells, `~/.bash_profile` or `~/.profile` for login shells, `/etc/environment` or a file under `/etc/profile.d/` for system-wide settings that apply to every user. A change to `~/.bashrc` takes effect in the *next* shell you start, or immediately in the current one if you load it by hand with `source ~/.bashrc` — a command worth remembering, since it saves logging out and back in for every edit.

### 3.5 Creating and deleting your own variables

Variables do not have to be pre-defined by the system; you can create them, name them whatever you like and use them as shorthand for values you type often. Here a variable is created to hold a domain name.

```bash
root@kali:~# url_variable="example.com"
root@kali:~# echo $url_variable
example.com
root@kali:~# echo "the target is $url_variable"
the target is example.com
root@kali:~# url_variable=example.org
root@kali:~# echo $url_variable
example.org
root@kali:~# unset url_variable
root@kali:~# echo $url_variable

root@kali:~#
```

The assignment creates the variable and `echo $url_variable` reads it back; the second form shows the substitution working inside a longer string, which is how variables earn their keep in scripts. The re-assignment demonstrates that a variable can be changed at any time simply by assigning it again, and the last two commands do the opposite: `unset` removes the variable from the shell entirely, and the following `echo` prints an empty line, because an undefined variable expands to nothing rather than producing an error.

That silent expansion is a trap worth naming, because scripts that fail mysteriously usually fail here: if a variable is unset or misspelled, `$name` becomes an empty string and the command runs with a missing argument instead of complaining. Two defences are standard. Quoting, which turns a missing value into an empty argument rather than into no argument at all — `echo "$url_variable"` is safer than `echo $url_variable` in every context where the value might contain spaces or be empty. And the `${var:?message}` form, which makes the shell abort with your message if the variable is unset, which is the right way to check a required input in a script: `: "${url_variable:?please set url_variable}"`.

The other habit from this subsection is quoting on assignment. `url_variable="example.com"` needs no quotes for a value without spaces, but `url_variable="the example domain"` would be interpreted as an assignment of `the` followed by a command called `example domain` without them — a mistake that produces one of the more confusing shell errors you will meet. Quoting the right-hand side of an assignment is never wrong, so it is worth doing by default.

### 3.6 The variables worth knowing

The environment is not an academic topic; a handful of variables change how the system behaves for you, and knowing them turns several classes of confusing failure into one-line diagnoses.

`PATH` is the most important of them, and we met it in Part 1 (section 2.8): a colon-separated list of directories the shell searches, in order, for the command you typed. `echo $PATH` shows your list, and appending to it is a common operation — `export PATH="$PATH:/opt/tools/bin"` makes a directory of your own scripts runnable by name, and it is worth understanding that the *order* determines who wins when two directories hold a program with the same name. The security angle is real in both directions: a `PATH` that includes `.` (the current directory) lets a malicious `ls` in a shared folder be executed by an unsuspecting user, and a writable directory early in root's `PATH` is a classic privilege escalation.

`HOME` is your home directory and is what `~` expands to; every program that stores a configuration file uses it, so a wrong value for a user running a service is a common source of "the daemon cannot find its config" problems. `USER` and `LOGNAME` name the current account, and `SHELL` records your login shell. `PS1` is the primary prompt string — the format of your shell prompt, which is why the prompt in this guide reads `root@kali:~#` and why changing `PS1` changes it instantly, complete with colour codes if you want them. `HISTFILE`, `HISTSIZE` and `HISTFILESIZE` govern command history, as we have seen. `LANG` and `LC_ALL` set the language, character encoding and formatting conventions that programs use for messages and for sorting, and they are the reason a script can behave differently on two machines with identical software — `sort` orders letters differently under different locales, and dates are printed in the local format. `EDITOR` and `VISUAL` decide which editor tools such as `crontab -e`, `git commit` and `visudo` open for you, which is why setting `export EDITOR=nano` is one of the first things a beginner should do. And `LD_PRELOAD` deserves a mention by name, because it asks the dynamic loader to load a shared library into every process the shell starts — a legitimate debugging and compatibility tool, and simultaneously a well-known privilege-escalation primitive when it can be set for a SUID binary. Seeing it in an environment on a production system is worth an investigation.

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

<<<<<<<< HEAD:docs/02 - Linux for Beginners - Part 2.md
If you can complete these three exercises without looking anything up, you are comfortable with the layer of Linux that sits between using the command line and administering a system — and you have covered everything in the first two parts of this guide.
========
If you can complete these three exercises without looking anything up, you are comfortable with the layer of Linux that sits between using the command line and administering a system — and you are ready for Part 3.
>>>>>>>> origin/arena/52f71d52-attack-scripts:docs/02-linux-for-beginners-part-02.md

---

## Closing notes

The three topics in this part share a theme that is easy to miss while you are memorising flags: they are all about **state that something else depends on**. An address is depended upon by every connection the machine makes; a process is depended upon by whoever is using the service it provides; an environment variable is depended upon by the child processes that inherit it and never report what they were given. That is why the commands here are more consequential than the ones in Part 1 — a mistake does not produce an error message, it produces a machine that behaves differently from the way you expected, often for somebody else.

Two habits from Part 1 are worth carrying forward, because they cover most of the risk. The first is to *look before you change anything*: read a configuration file before you overwrite it, run `pgrep` before you run `pkill`, print an address table before you alter an interface, and save a value before you unset or reassign it. The second is to *have a way back*: a snapshot, a second console, or a rollback scheduled with `at` before you touch the network.

<<<<<<<< HEAD:docs/02 - Linux for Beginners - Part 2.md
**Part 3** is where those threads come together. [Linux for Beginners (Part 3): Bash Scripting, Automation and Services](03%20-%20Linux%20for%20Beginners%20-%20Part%203.md) takes the variables, the pipelines and the processes you have just met and turns them into programs — a Bash script that takes input and does something useful, a `cron` job that runs it on a schedule without anyone remembering to launch it, and the services (Apache, OpenSSH, FTP) that let a machine answer requests by itself. Beyond that, natural next steps are **service management** with `systemctl` and the systemd journal, **package and configuration management** across more than one machine, and a deeper study of the **permission and privilege model**. Each of them builds directly on the material above, and each of them is far less intimidating once moving around the file system, reading a long listing and inspecting a running process all feel like second nature.
========
**Part 3** is the continuation and it picks up exactly where this guide leaves off. [Linux for Beginners (Part 3): Scripting, Scheduling and Services](03-linux-for-beginners-part-03.md) covers shell **scripting and automation** — conditions, loops, functions and the variables you have just met, assembled into programs that do a day's work in a second — along with **scheduling** with `cron` and **service management** with `service`/`systemctl`, including the Apache web server, OpenSSH and FTP. Beyond the three parts, natural next steps are writing your own systemd units and timers, **package and configuration management** across more than one machine, and a deeper study of the **permission and privilege model**. Each of them builds directly on the material above, and each of them is far less intimidating once moving around the file system, reading a long listing and inspecting a running process all feel like second nature.
>>>>>>>> origin/arena/52f71d52-attack-scripts:docs/02-linux-for-beginners-part-02.md
