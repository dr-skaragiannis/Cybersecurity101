# tcpdump: Πρακτική Συλλογή και Ανάλυση Πακέτων σε Εξουσιοδοτημένο Εργαστήριο

> **Μόνο για εξουσιοδοτημένη χρήση.** Η συλλογή πακέτων μπορεί να αποκαλύψει κωδικούς, session tokens, προσωπικά δεδομένα, εσωτερικές διευθύνσεις και περιεχόμενο εφαρμογών. Καταγράφετε μόνο σε δίκτυα και συστήματα που σας ανήκουν ή για τα οποία έχετε ρητή άδεια παρακολούθησης. Προτιμήστε lab interface, περιορίστε scope και retention, και προστατεύστε τα capture files ως ευαίσθητα στοιχεία.

## Τι κάνει το tcpdump

Το `tcpdump` καταγράφει και αποκωδικοποιεί πακέτα που παρατηρεί ένα network interface. Χρησιμοποιεί το libpcap capture framework και δέχεται ισχυρή γλώσσα filters, γνωστή ως BPF syntax, ώστε να περιορίζεται η συλλογή. Είναι χρήσιμο για troubleshooting DNS, TCP handshakes, service reachability, routing και protocol behavior. Δεν είναι intrusion tool· η κύρια αξία του είναι η προσεκτική παρατήρηση και ερμηνεία.

Ένα τυπικό packet summary περιέχει timestamp, source endpoint, destination endpoint, protocol πληροφορίες, TCP flags όπου υπάρχουν και packet length. Η ακριβής εμφάνιση διαφέρει ανά λειτουργικό, tcpdump version, link type, privileges και traffic.

## Ασφαλής εργαστηριακή αρχιτεκτονική

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

Χρησιμοποιήστε test client που ελέγχετε για προβλέψιμο traffic, όπως DNS lookup, HTTP request σε local test server ή ping μέσα στο lab. Αποθηκεύστε captures σε encrypted ή access-controlled θέση. Μην στέλνετε PCAP που περιέχει ευαίσθητο traffic σε μη προστατευμένο chat, email ή public file-sharing.

## Εγκατάσταση και εντοπισμός interfaces

Σε Debian-based συστήματα:

```bash
sudo apt update
sudo apt install -y tcpdump
sudo tcpdump -D
```

Ενδεικτική έξοδος:

```text
1.eth0 [Up, Running]
2.lo [Up, Running, Loopback]
3.any (Pseudo-device that captures on all interfaces)
```

Η `apt update` ανανεώνει package metadata και η `apt install` εγκαθιστά το capture utility. Η `tcpdump -D` εμφανίζει interfaces που μπορεί να χρησιμοποιήσει το tcpdump. Το `eth0` είναι συνήθως Ethernet-style interface, το `lo` είναι local loopback και το `any` καταγράφει από πολλαπλά interfaces. Τα ονόματα μπορεί να είναι `ens33`, `enp0s3`, `wlan0` ή άλλα.

```bash
tcpdump --version
tcpdump --help | head -25
```

Παράδειγμα εξόδου:

```text
tcpdump version 4.x
libpcap version 1.x
Usage: tcpdump [ -AbdDefhHIJKlLnNOpqStuUvxX# ] ...
```

Η έκδοση δείχνει τα tcpdump και libpcap builds. Η βοήθεια εμφανίζει διαθέσιμα options. Features και protocol decoders μπορεί να διαφέρουν ανά release, επομένως ελέγχετε `man tcpdump` όταν η συμπεριφορά διαφέρει.

## Περιορισμένη αρχική capture

Capture χωρίς όρια μπορεί να δημιουργήσει πολύ μεγάλα files ή να συλλέξει περισσότερο data από όσο χρειάζεται. Ξεκινήστε με packet count:

```bash
sudo tcpdump -i eth0 -c 5
```

Ενδεικτική έξοδος:

```text
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
20:32:14.102431 IP 192.168.56.30.49152 > 192.168.56.20.80: Flags [S], seq 125430, win 64240, length 0
20:32:14.102580 IP 192.168.56.20.80 > 192.168.56.30.49152: Flags [S.], seq 987650, ack 125431, win 65160, length 0
5 packets captured
```

Η `-i eth0` επιλέγει interface και η `-c 5` σταματά μετά από πέντε πακέτα. Το πρώτο πακέτο είναι TCP SYN από client ephemeral port `49152` προς port 80. Το δεύτερο είναι SYN-ACK, δηλαδή server reply στη διάρκεια TCP three-way handshake. Η τελευταία γραμμή επιβεβαιώνει πόσα packets καταγράφηκαν. Το `verbose output suppressed` είναι φυσιολογικό μήνυμα και προτείνει `-v` για περισσότερη λεπτομέρεια.

## Ονόματα, αριθμοί και σύντομη έξοδος

Από προεπιλογή το tcpdump μπορεί να επιλύει addresses ή service names. Απενεργοποιήστε name resolution για γρηγορότερη και σαφή έξοδο:

```bash
sudo tcpdump -i eth0 -c 3 -nn
```

Ενδεικτική έξοδος:

```text
20:33:01.100001 IP 192.168.56.30.53530 > 192.168.56.20.53: 1234+ A? example.test. (30)
```

Η πρώτη `-n` αποφεύγει hostname resolution και η δεύτερη service-name conversion. Έτσι η `53` παραμένει αριθμός θύρας αντί για `domain`. Αυτό μειώνει delays και side effects από DNS lookups κατά την ανάλυση.

```bash
sudo tcpdump -i eth0 -c 3 -q -nn
```

Παράδειγμα εξόδου:

```text
IP 192.168.56.30.49152 > 192.168.56.20.80: tcp 0
IP 192.168.56.20.80 > 192.168.56.30.49152: tcp 0
```

Η `-q` εμφανίζει λιγότερες protocol λεπτομέρειες. Είναι χρήσιμη για high-level flow summary, αλλά μπορεί να κρύψει στοιχεία που απαιτούνται για διάγνωση, όπως TCP flags.

## Filters για protocol, port, host και network

BPF filter τοποθετείται μετά από tcpdump options. Καταγράψτε δέκα ICMP packets:

```bash
sudo tcpdump -i eth0 -c 10 -nn icmp
```

Ενδεικτική έξοδος:

```text
20:34:10.000200 IP 192.168.56.30 > 192.168.56.20: ICMP echo request, id 4512, seq 1, length 64
20:34:10.000611 IP 192.168.56.20 > 192.168.56.30: ICMP echo reply, id 4512, seq 1, length 64
```

Η πρώτη γραμμή είναι ICMP echo request, που συχνά δημιουργεί το `ping`, και η δεύτερη είναι η αντίστοιχη reply. Το identifier και sequence number βοηθούν στο pairing request και reply. Request χωρίς reply μπορεί να υποδηλώνει firewall, routing πρόβλημα, host unavailability ή packet loss· δεν αποτελεί αυτόματα κακόβουλη ενέργεια.

```bash
sudo tcpdump -i eth0 -c 10 -nn 'port 80'
```

Παράδειγμα εξόδου:

```text
IP 192.168.56.30.49152 > 192.168.56.20.80: Flags [S], length 0
IP 192.168.56.20.80 > 192.168.56.30.49152: Flags [S.], length 0
```

Το `port 80` ταιριάζει με πακέτα όπου source ή destination port είναι 80. Για server-side destination χρησιμοποιήστε `dst port 80`. Για traffic που ξεκινά από port 80 χρησιμοποιήστε `src port 80`.

```bash
sudo tcpdump -i eth0 -c 10 -nn host 192.168.56.30
sudo tcpdump -i eth0 -c 10 -nn net 192.168.56.0/24
```

Το `host` ταιριάζει και στις δύο κατευθύνσεις μιας συγκεκριμένης address. Το `net` ταιριάζει με traffic προς ή από το δηλωμένο network. Τα στενά filters μειώνουν άσχετη συλλογή και βελτιώνουν αναγνωσιμότητα.

```bash
sudo tcpdump -i eth0 -c 10 -nn 'portrange 8000-8100'
```

Το `portrange` ταιριάζει με packets όπου source ή destination port είναι μεταξύ 8000 και 8100. Χρησιμοποιήστε το μόνο όταν το range σχετίζεται με το συγκεκριμένο test, γιατί τα ευρεία ranges αυξάνουν θόρυβο.

## Κατεύθυνση και link-layer πληροφορίες

Σε συστήματα που υποστηρίζουν capture direction:

```bash
sudo tcpdump -i eth0 -c 5 -nn -Q in
sudo tcpdump -i eth0 -c 5 -nn -Q out
```

Η `-Q in` εμφανίζει πακέτα που λαμβάνει το interface, ενώ η `-Q out` πακέτα που στέλνει. Η υποστήριξη εξαρτάται από platform και driver. Αν το tcpdump αναφέρει ότι η direction δεν υποστηρίζεται, χρησιμοποιήστε host, address ή port filter.

```bash
sudo tcpdump -i eth0 -c 2 -nn -e
```

Ενδεικτική έξοδος:

```text
20:35:44.200001 08:00:27:aa:bb:01 > 08:00:27:aa:bb:20, ethertype IPv4 (0x0800), length 74: 192.168.56.30.49152 > 192.168.56.20.80: Flags [S], length 0
```

Η `-e` προσθέτει link-layer πληροφορίες, όπως source και destination MAC addresses. Το `ethertype IPv4 (0x0800)` αναγνωρίζει το network-layer protocol του Ethernet frame. Οι MAC addresses έχουν συνήθως σημασία μόνο στο τοπικό Layer-2 segment.

## Verbosity και timestamps

Αυξήστε protocol detail μόνο όταν χρειάζεται:

```bash
sudo tcpdump -i eth0 -c 2 -nn -v
sudo tcpdump -i eth0 -c 2 -nn -vv
sudo tcpdump -i eth0 -c 2 -nn -vvv
```

Πιο verbose packet μπορεί να εμφανίζεται ως:

```text
IP (tos 0x0, ttl 64, id 22301, offset 0, flags [DF], proto TCP (6), length 60)
    192.168.56.30.49152 > 192.168.56.20.80: Flags [S], cksum 0xabcd (correct), seq 125430, win 64240, options [mss 1460,sackOK,TS val 100 ecr 0,nop,wscale 7], length 0
```

Το IPv4 τμήμα περιλαμβάνει type of service, time-to-live, packet ID, fragmentation στοιχεία, protocol number και συνολικό μήκος. Το TCP τμήμα περιλαμβάνει flags, checksum status, sequence number, receive window και negotiation options. Η επιπλέον verbosity βοηθά troubleshooting αλλά αυξάνει τον όγκο εξόδου.

```bash
sudo tcpdump -i eth0 -c 2 -nn -t
sudo tcpdump -i eth0 -c 2 -nn -tt
sudo tcpdump -i eth0 -c 2 -nn -ttt
sudo tcpdump -i eth0 -c 2 -nn -tttt
```

Η `-t` αποκρύπτει timestamps. Η `-tt` εμφανίζει absolute Unix-style timestamp. Η `-ttt` εμφανίζει διαφορά χρόνου από το προηγούμενο packet και βοηθά σε latency/timing analysis. Η `-tttt` δείχνει ημερομηνία και ώρα σε human-readable μορφή. Η ακριβής εμφάνιση εξαρτάται από version.

## Payload rendering και snapshot length

Η εμφάνιση payload μπορεί να αποκαλύψει plaintext application data και απαιτεί ιδιαίτερη προσοχή. Σε lab, ελέγξτε περιορισμένο αριθμό packets ως ASCII:

```bash
sudo tcpdump -i eth0 -c 2 -nn -A 'tcp port 8080'
```

Ενδεικτική έξοδος:

```text
GET /health HTTP/1.1
Host: 192.168.56.20:8080
User-Agent: lab-client/1.0
```

Η `-A` εμφανίζει payload ως ASCII όπου είναι δυνατό. Το παράδειγμα είναι HTTP request: το `GET` ζητά το `/health`, το `Host` δηλώνει virtual host και port, και το `User-Agent` δείχνει client software string. Μην χρησιμοποιείτε αυτή την τεχνική για συλλογή passwords, cookies, authorization headers ή user content.

```bash
sudo tcpdump -i eth0 -c 1 -nn -X 'tcp port 8080'
```

Παράδειγμα εξόδου:

```text
0x0000:  4500 0054 1a2b 4000 4006 ...
0x0030:  4745 5420 2f68 6561 6c74 6820 4854 5450  GET /health HTTP
```

Η `-X` εμφανίζει hexadecimal και ASCII decoding. Οι τιμές αριστερά είναι byte offsets. Το ASCII δεξιά εμφανίζει αναγνωρίσιμο plaintext, ενώ το hex είναι χρήσιμο για non-printable fields. Η `-x` εμφανίζει hex χωρίς ASCII, η `-xx` περιλαμβάνει link-layer header και η `-XX` συνδυάζει link-layer hex με ASCII.

Το snapshot length περιορίζει πόσα bytes διατηρεί το tcpdump ανά packet:

```bash
sudo tcpdump -i eth0 -c 2 -nn -s 96 'tcp port 8080'
```

Ενδεικτική έξοδος:

```text
tcpdump: listening on eth0, link-type EN10MB (Ethernet), snapshot length 96 bytes
```

Η `-s 96` κρατά έως 96 bytes ανά packet. Μικρότερο snapshot μειώνει stored data και capture volume, αλλά μπορεί να περικόψει headers ή payload απαραίτητα για ανάλυση. Ελέγχετε το startup line αντί να θεωρείτε σταθερό default snap length.

## TCP flags και διάγνωση συνδέσεων

Τα TCP flags εξηγούν connection state. Καταγράψτε initial SYN packets με bit-mask filter:

```bash
sudo tcpdump -i eth0 -nn -c 5 'tcp[tcpflags] & (tcp-syn|tcp-ack) == tcp-syn'
```

Ενδεικτική έξοδος:

```text
IP 192.168.56.30.49152 > 192.168.56.20.80: Flags [S], seq 125430, win 64240, length 0
```

Η έκφραση ταιριάζει με πακέτα όπου SYN είναι set και ACK δεν είναι set, δηλαδή συνήθως initial connection attempts. Το bit-mask είναι πιο αξιόπιστο από strict equality επειδή ένα packet μπορεί να περιέχει επιπλέον flags.

```bash
sudo tcpdump -i eth0 -nn -c 5 'tcp[tcpflags] & tcp-rst != 0'
```

Παράδειγμα εξόδου:

```text
IP 192.168.56.20.80 > 192.168.56.30.49152: Flags [R.], seq 1, ack 1, win 0, length 0
```

Το RST δηλώνει απότομο TCP reset. Μπορεί να προκύψει από κλειστή θύρα, application abort, firewall device ή network middlebox. Συσχετίστε το με service state και logs πριν αποφασίσετε αιτία.

```bash
sudo tcpdump -i eth0 -nn -S -c 3 tcp
```

Η `-S` εμφανίζει absolute sequence numbers αντί relative. Είναι χρήσιμη για λεπτομερές protocol troubleshooting, αλλά συνήθως δεν απαιτείται σε routine review.

## PCAP αρχεία: εγγραφή και ανάγνωση

Γράψτε raw packets σε PCAP για μεταγενέστερη ανάλυση. Αποφύγετε unlimited capture:

```bash
sudo tcpdump -i eth0 -c 100 -nn -w lab-web-test.pcap 'tcp port 8080'
ls -lh lab-web-test.pcap
```

Ενδεικτική έξοδος:

```text
100 packets captured
-rw-r----- 1 root root 42K Oct  8 20:40 lab-web-test.pcap
```

Η `-w` γράφει binary packet records αντί για readable lines. Το PCAP ανοίγει με tcpdump ή graphical analyzer. Τα permissions και owner της `ls -lh` πρέπει να ελέγχονται, επειδή τα capture files μπορεί να περιέχουν ευαίσθητα payloads.

```bash
tcpdump -nn -r lab-web-test.pcap
```

Παράδειγμα εξόδου:

```text
reading from file lab-web-test.pcap, link-type EN10MB (Ethernet), snapshot length 262144
20:40:01.000010 IP 192.168.56.30.49152 > 192.168.56.20.8080: Flags [S], length 0
```

Η `-r` διαβάζει πακέτα από saved PCAP. Η κεφαλίδα δείχνει link type και snap length που καταγράφηκαν στο αρχείο. Αυτή η repeatable ροή επιτρέπει ανάλυση protected evidence αντί για επανάληψη δραστηριότητας σε live δίκτυο.

## Γραφική ανάλυση πακέτων

Ένα PCAP μπορεί να μεταφερθεί με ασφάλεια σε approved analysis workstation και να ανοιχθεί στο Wireshark. Το ακόλουθο ASCII αντικαθιστά GUI screenshot:

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

Το επάνω τμήμα εμφανίζει packet summaries, το μεσαίο αναπτύσσει protocol fields και το κάτω raw bytes. Graphical analyzer είναι χρήσιμο για follow TCP stream, display filters και protocol trees. Διατηρείτε το αρχικό PCAP και καταγράφετε χωριστά κάθε derived ή filtered file.

## Remote capture με safeguards

Σε authorized remote host, γράψτε PCAP τοπικά αντί να στέλνετε live binary stream σε μη έμπιστο προορισμό:

```bash
sudo tcpdump -i eth0 -c 50 -nn -w /var/tmp/lab-capture.pcap 'tcp port 8080'
sudo chmod 0640 /var/tmp/lab-capture.pcap
```

Η πρώτη εντολή κάνει bounded capture στο remote host. Η δεύτερη περιορίζει read access σε owner και group. Μεταφέρετε το μόνο μέσω approved authenticated channel και διαγράψτε το σύμφωνα με evidence-retention policy. Αποφεύγετε εντολές που εκθέτουν passwords σε shell, logs, command history ή capture data.

## Promiscuous mode

Σε πολλά Ethernet interfaces το tcpdump ζητά promiscuous mode ώστε το capture library να λαμβάνει frames που παρουσιάζονται στο interface. Σε switched networks, αυτό δεν εκθέτει αυτόματα όλη την traffic άλλων hosts· η switch port configuration, VLAN membership και hardware καθορίζουν ποια frames φτάνουν στο interface.

```bash
sudo tcpdump -i eth0 --no-promiscuous-mode -c 5 -nn
```

Η εντολή καταγράφει traffic που παραδίδεται κανονικά στο host interface χωρίς να ζητά promiscuous mode. Είναι συχνά κατάλληλη για διάγνωση της traffic του ίδιου host. Η διαθεσιμότητα και συμπεριφορά εξαρτώνται από OS και driver.

## Καλές πρακτικές capture

- Ξεκινήστε με συγκεκριμένο ερώτημα, όπως «Λαμβάνει ο test client DNS reply;» ή «Ολοκληρώνει το local test service TCP handshake;».
- Επιλέξτε σωστό interface και επαληθεύστε το με `tcpdump -D` και `ip -brief address`.
- Χρησιμοποιήστε narrow BPF filters, packet count και κατάλληλο snap length.
- Προτιμήστε `-nn` για αποφυγή name-resolution delays και ασαφών labels.
- Αντιμετωπίστε `-A`, `-X` και PCAP ως ευαίσθητα επειδή μπορεί να αποκαλύψουν plaintext content.
- Καταγράψτε time zone, host, interface, capture filter, tool version και file hashes στις σημειώσεις διερεύνησης.
- Συσχετίστε packets με service logs, firewall logs, routing configuration και authorized test activity πριν εξάγετε συμπεράσματα.

## Βασικά μαθήματα

Το tcpdump είναι πιο αποτελεσματικό όταν συλλέγει την ελάχιστη traffic που χρειάζεται για να απαντήσει σε συγκεκριμένο troubleshooting ή security ερώτημα. Η επιλογή interface, τα BPF filters, τα packet limits και η ασφαλής διαχείριση στοιχείων είναι εξίσου σημαντικά με τη γνώση πρωτοκόλλων. Ένα SYN, ένα ICMP request ή μια plaintext HTTP γραμμή είναι παρατήρηση που απαιτεί επιβεβαίωση με context — όχι συμπέρασμα από μόνο του.
