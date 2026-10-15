# tcpdump: Πρακτική Συλλογή και Ανάλυση Πακέτων σε Εξουσιοδοτημένο Εργαστήριο

> **Μόνο για εξουσιοδοτημένη χρήση.** Η συλλογή πακέτων μπορεί να αποκαλύψει κωδικούς, session tokens, προσωπικά δεδομένα, εσωτερικές διευθύνσεις και περιεχόμενο εφαρμογών. Καταγράφετε μόνο σε δίκτυα και συστήματα που σας ανήκουν ή για τα οποία έχετε ρητή άδεια παρακολούθησης. Προτιμήστε lab interface, περιορίστε scope και retention, και προστατεύστε τα capture files ως ευαίσθητα στοιχεία.

## Πώς να διαβάζετε αυτόν τον οδηγό

Κάθε βήμα εμφανίζεται με τον ίδιο τρόπο: η **εντολή** που πληκτρολογείτε, η **έξοδος** που παράγει και, στη συνέχεια, μια σύντομη επεξήγηση για το τι συνέβη και πώς να διαβάσετε το αποτέλεσμα. Για παράδειγμα:

```bash
id
```

**Έξοδος:**

```text
uid=1000(student) gid=1000(student) groups=1000(student),27(sudo)
```

Το πλαίσιο εντολής δείχνει τι ακριβώς εκτελείτε· το πλαίσιο εξόδου δείχνει τι απάντησε το μηχάνημα. Η έξοδος είναι ενδεικτική — διευθύνσεις, εκδόσεις και process IDs διαφέρουν στο δικό σας lab.

Η παράγραφος μετά από κάθε έξοδο εξηγεί *τι κάνει το εργαλείο* και *γιατί έχει σημασία το βήμα*, ώστε να μάθετε τη μέθοδο και όχι μόνο τη σύνταξη. Οτιδήποτε μεγαλύτερο — επιπλέον παράμετροι, παγίδες, θεωρία — βρίσκεται σε συμπτυσσόμενο πλαίσιο.

<details>
<summary><strong>Περισσότερα</strong> — οι συμβάσεις που ισχύουν σε όλο τον οδηγό</summary>

* **Το πλαίσιο εντολής είναι αυτό που πληκτρολογείτε· το πλαίσιο εξόδου είναι αυτό που εκτυπώνει το μηχάνημα.** Μια εντολή που πετυχαίνει συχνά δεν εκτυπώνει τίποτα — στο Linux, η σιωπή συνήθως σημαίνει επιτυχία.
* **Ένα πλαίσιο εντολής μπορεί να περιέχει πολλές γραμμές.** Όταν πολλές εντολές ανήκουν στην ίδια ιδέα ομαδοποιούνται· εκτελέστε τις με τη σειρά, μία κάθε φορά, και διαβάστε την έξοδο μετά από καθεμία.
* **Το `sudo` εκτελεί την εντολή ως διαχειριστής.** Η συλλογή απαιτεί συνήθως δικαίωμα ανοίγματος raw socket δικτύου, γι' αυτό οι περισσότερες εντολές το έχουν μπροστά.
* **Κείμενο μέσα σε γωνιώδεις αγκύλες, όπως `-i <interface>`, είναι δεσμευτικό θέσης** που αντικαθιστάτε με πραγματική τιμή. Τις ίδιες τις αγκύλες δεν τις πληκτρολογείτε ποτέ.
* **Η έξοδος είναι ενδεικτική, όχι πανομοιότυπη.** Διεπαφές, διευθύνσεις, εκδόσεις και πλήθη πακέτων διαφέρουν στο δικό σας lab. Μετράει η δομή της εξόδου, όχι τα ακριβή ψηφία.

</details>

### Πίνακας Περιεχομένων

1. [Τι κάνει το tcpdump](#τι-κάνει-το-tcpdump)
2. [Ασφαλής εργαστηριακή αρχιτεκτονική](#ασφαλής-εργαστηριακή-αρχιτεκτονική)
3. [Εγκατάσταση και εντοπισμός interfaces](#εγκατάσταση-και-εντοπισμός-interfaces)
4. [Περιορισμένη αρχική capture](#περιορισμένη-αρχική-capture)
5. [Ονόματα, αριθμοί και σύντομη έξοδος](#ονόματα-αριθμοί-και-σύντομη-έξοδος)
6. [Filters για protocol, port, host και network](#filters-για-protocol-port-host-και-network)
7. [Κατεύθυνση και link-layer πληροφορίες](#κατεύθυνση-και-link-layer-πληροφορίες)
8. [Verbosity και timestamps](#verbosity-και-timestamps)
9. [Payload rendering και snapshot length](#payload-rendering-και-snapshot-length)
10. [TCP flags και διάγνωση συνδέσεων](#tcp-flags-και-διάγνωση-συνδέσεων)
11. [PCAP αρχεία: εγγραφή και ανάγνωση](#pcap-αρχεία-εγγραφή-και-ανάγνωση)
12. [Γραφική ανάλυση πακέτων](#γραφική-ανάλυση-πακέτων)
13. [Remote capture με safeguards](#remote-capture-με-safeguards)
14. [Promiscuous mode](#promiscuous-mode)
15. [Καλές πρακτικές capture](#καλές-πρακτικές-capture)
16. [Βασικά μαθήματα](#βασικά-μαθήματα)

---

## Τι κάνει το tcpdump

Το `tcpdump` καταγράφει και αποκωδικοποιεί πακέτα που παρατηρεί ένα network interface. Χρησιμοποιεί το libpcap capture framework και δέχεται ισχυρή γλώσσα filters, γνωστή ως BPF syntax, ώστε να περιορίζεται η συλλογή.

Είναι χρήσιμο για troubleshooting DNS, TCP handshakes, service reachability, routing και protocol behavior. Δεν είναι intrusion tool· η κύρια αξία του είναι η προσεκτική παρατήρηση και ερμηνεία.

Ένα τυπικό packet summary περιέχει timestamp, source endpoint, destination endpoint, protocol πληροφορίες, TCP flags όπου υπάρχουν και packet length.

<details>
<summary><strong>Περισσότερα</strong> — γιατί η ίδια εντολή μπορεί να φαίνεται διαφορετική σε δύο μηχανήματα</summary>

Η ακριβής εμφάνιση διαφέρει ανά λειτουργικό, tcpdump version, link type, privileges και traffic της στιγμής. Γι' αυτό δύο hosts μπορεί να εκτυπώσουν ελαφρώς διαφορετική διάταξη για το ίδιο πακέτο, και γι' αυτό αξίζει να διαβάζετε τη γραμμή εκκίνησης κάθε capture πριν εμπιστευτείτε τις στήλες από κάτω.

</details>

## Ασφαλής εργαστηριακή αρχιτεκτονική

Το σχήμα παρακάτω δείχνει το μικρότερο χρήσιμο lab: έναν client που ελέγχετε, έναν capture host και προαιρετικά ένα analysis workstation.

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

Χρησιμοποιήστε test client που ελέγχετε για προβλέψιμο traffic, όπως DNS lookup, HTTP request σε local test server ή ping μέσα στο lab.

Αποθηκεύστε τα captures σε encrypted ή access-controlled θέση. Μην στέλνετε captures με ευαίσθητο traffic μέσω μη προστατευμένου chat, email ή public file-sharing.

<details>
<summary><strong>Περισσότερα</strong> — κρατήστε τη συλλογή μέσα στα όρια του lab</summary>

Ο client παράγει το traffic, ο capture host το παρατηρεί και το analysis workstation βλέπει μόνο ένα *αντίγραφο*. Ο διαχωρισμός των τριών ρόλων σημαίνει ότι ένα capture μπορεί να διαγραφεί από τον capture host χωρίς να χαθεί η ανάλυση, και κρατά τα captured payloads μακριά από κοινόχρηστα ή προσωπικά μηχανήματα.

</details>

## Εγκατάσταση και εντοπισμός interfaces

Σε Debian-based συστήματα, εγκαταστήστε το tcpdump και καταγράψτε τα interfaces που μπορούν να συλλέξουν:

```bash
sudo apt update
sudo apt install -y tcpdump
sudo tcpdump -D
```

**Έξοδος:**

```text
1.eth0 [Up, Running]
2.lo [Up, Running, Loopback]
3.any (Pseudo-device that captures on all interfaces)
```

Η `apt update` ανανεώνει package metadata και η `apt install` εγκαθιστά το capture utility. Η `tcpdump -D` εμφανίζει τα interfaces που μπορεί να χρησιμοποιήσει το tcpdump.

Το `eth0` είναι συνήθως Ethernet-style interface, το `lo` είναι local loopback και το `any` καταγράφει από πολλαπλά interfaces.

<details>
<summary><strong>Περισσότερα</strong> — τα ονόματα interfaces διαφέρουν από διανομή σε διανομή</summary>

Τα ονόματα μπορεί να είναι `ens33`, `enp0s3`, `wlan0` ή κάποιο άλλο ειδικό για τη διανομή. Γι' αυτό τρέχετε πρώτα `tcpdump -D` και αντιγράφετε το ακριβές όνομα, αντί να υποθέτετε ότι υπάρχει το κλασικό `eth0`.

</details>

Δείτε την έκδοση και τις πληροφορίες χρήσης:

```bash
tcpdump --version
tcpdump --help | head -25
```

**Έξοδος:**

```text
tcpdump version 4.x
libpcap version 1.x
Usage: tcpdump [ -AbdDefhHIJKlLnNOpqStuUvxX# ] ...
```

Η έξοδος έκδοσης δείχνει τα builds tcpdump και libpcap. Η σύνοψη βοήθειας εμφανίζει τα διαθέσιμα options.

<details>
<summary><strong>Περισσότερα</strong> — τα features διαφέρουν μεταξύ εκδόσεων</summary>

Features και protocol decoders μπορεί να διαφέρουν ανά release, επομένως ελέγχετε τη σύνταξη τοπικά με `man tcpdump` όταν μια εντολή συμπεριφέρεται διαφορετικά από όσο δείχνει ένας οδηγός.

</details>

## Περιορισμένη αρχική capture

Capture χωρίς όρια μπορεί να δημιουργήσει πολύ μεγάλα αρχεία ή να συλλέξει περισσότερα δεδομένα από όσα χρειάζεται. Ξεκινήστε με όριο πλήθους πακέτων:

```bash
sudo tcpdump -i eth0 -c 5
```

**Έξοδος:**

```text
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
20:32:14.102431 IP 192.168.56.30.49152 > 192.168.56.20.80: Flags [S], seq 125430, win 64240, length 0
20:32:14.102580 IP 192.168.56.20.80 > 192.168.56.30.49152: Flags [S.], seq 987650, ack 125431, win 65160, length 0
5 packets captured
```

Η `-i eth0` επιλέγει interface και η `-c 5` σταματά μετά από πέντε πακέτα. Το πρώτο πακέτο είναι TCP SYN από client ephemeral port `49152` προς port 80. Το δεύτερο είναι SYN-ACK, δηλαδή η απάντηση του server στη διάρκεια του TCP three-way handshake.

Η τελευταία γραμμή επιβεβαιώνει πόσα πακέτα κατέγραψε το tcpdump. Το `verbose output suppressed` είναι φυσιολογικό· προτείνει `-v` για περισσότερη λεπτομέρεια.

<details>
<summary><strong>Περισσότερα</strong> — η γραμμή εκκίνησης, και γιατί το όριο πλήθους είναι συνήθεια</summary>

Οι δύο πρώτες γραμμές δεν είναι πακέτα: είναι το banner εκκίνησης, που δηλώνει interface, link type (το `EN10MB` είναι Ethernet) και snapshot length. Ό,τι ακολουθεί είναι μία γραμμή ανά πακέτο.

Το όριο πλήθους (`-c`) είναι η απλούστερη προφύλαξη από τα δύο πραγματικά προβλήματα της συλλογής — ένα αρχείο που αυξάνεται χωρίς όριο και μια ροή που δεν μπορείτε πλέον να ελέγξετε ρεαλιστικά.

</details>

## Ονόματα, αριθμοί και σύντομη έξοδος

Από προεπιλογή το tcpdump μπορεί να επιλύει addresses ή service names. Απενεργοποιήστε το name resolution όταν χρειάζεστε γρηγορότερη και σαφή έξοδο:

```bash
sudo tcpdump -i eth0 -c 3 -nn
```

**Έξοδος:**

```text
20:33:01.100001 IP 192.168.56.30.53530 > 192.168.56.20.53: 1234+ A? example.test. (30)
```

Η `-n` αποφεύγει hostname resolution και η δεύτερη `n` αποφεύγει service-name conversion. Έτσι η `53` παραμένει αριθμός θύρας αντί για `domain`.

Αυτό μειώνει delays και side effects από DNS lookups κατά την ανάλυση.

Χρησιμοποιήστε quick mode όταν αρκεί μια σύντομη σύνοψη ροής:

```bash
sudo tcpdump -i eth0 -c 3 -q -nn
```

**Έξοδος:**

```text
IP 192.168.56.30.49152 > 192.168.56.20.80: tcp 0
IP 192.168.56.20.80 > 192.168.56.30.49152: tcp 0
```

Η `-q` εμφανίζει λιγότερες protocol λεπτομέρειες. Είναι χρήσιμη για high-level εικόνα, αλλά μπορεί να κρύψει στοιχεία που χρειάζονται για διάγνωση, όπως TCP flags και sequence-related λεπτομέρειες.

<details>
<summary><strong>Περισσότερα</strong> — γιατί ένα lab προτιμά αριθμούς αντί ονόματα</summary>

Σε lab δίκτυο ο resolver είναι συχνά αργός, μη προσβάσιμος ή σκόπιμα απομονωμένος, και ένα lookup που δεν απαντά μπορεί να κολλήσει ολόκληρη τη συλλογή. Η αριθμητική έξοδος είναι επίσης σαφής: μια διεύθυνση δεν μπορεί να συγχυστεί με hostname που απλώς της μοιάζει — κάτι που μετράει όταν συγκρίνετε capture με logs ή πίνακα firewall.

</details>

## Filters για protocol, port, host και network

Το BPF filter τοποθετείται μετά από τα options του tcpdump. Καταγράψτε δέκα ICMP πακέτα στο επιλεγμένο interface:

```bash
sudo tcpdump -i eth0 -c 10 -nn icmp
```

**Έξοδος:**

```text
20:34:10.000200 IP 192.168.56.30 > 192.168.56.20: ICMP echo request, id 4512, seq 1, length 64
20:34:10.000611 IP 192.168.56.20 > 192.168.56.30: ICMP echo reply, id 4512, seq 1, length 64
```

Η πρώτη γραμμή είναι ICMP echo request, που συχνά δημιουργεί το `ping`· η δεύτερη είναι η αντίστοιχη reply. Το identifier και το sequence number βοηθούν στο ταίριασμα request και reply.

Request χωρίς reply μπορεί να υποδηλώνει firewall rule, routing πρόβλημα, host unavailability ή packet loss· δεν αποτελεί αυτόματα κακόβουλη ενέργεια.

Καταγράψτε traffic που αφορά θύρα web:

```bash
sudo tcpdump -i eth0 -c 10 -nn 'port 80'
```

**Έξοδος:**

```text
IP 192.168.56.30.49152 > 192.168.56.20.80: Flags [S], length 0
IP 192.168.56.20.80 > 192.168.56.30.49152: Flags [S.], length 0
```

Το filter `port 80` ταιριάζει με πακέτα όπου source ή destination port είναι 80. Για server-side destination χρησιμοποιήστε `dst port 80`· για traffic που ξεκινά από port 80 χρησιμοποιήστε `src port 80`.

```bash
sudo tcpdump -i eth0 -c 10 -nn host 192.168.56.30
sudo tcpdump -i eth0 -c 10 -nn net 192.168.56.0/24
```

Το `host` ταιριάζει και στις δύο κατευθύνσεις μιας address. Το `net` ταιριάζει με traffic προς ή από το δηλωμένο network.

Τα στενά filters μειώνουν την άσχετη συλλογή και βελτιώνουν την αναγνωσιμότητα.

Ένα port range είναι χρήσιμο όταν δοκιμάζετε μια ομάδα σχετικών υπηρεσιών:

```bash
sudo tcpdump -i eth0 -c 10 -nn 'portrange 8000-8100'
```

Αυτό ταιριάζει με πακέτα όπου source ή destination port είναι μεταξύ 8000 και 8100 συμπεριλαμβανομένων. Χρησιμοποιήστε το μόνο όταν το range σχετίζεται· τα ευρέα ranges αυξάνουν τον θόρυβο.

<details>
<summary><strong>Περισσότερα</strong> — φτιάχνοντας ένα filter που λέει ακριβώς αυτό που εννοείτε</summary>

Η γλώσσα filters είναι γραμματική και όχι κατάλογος λέξεων-κλειδιών: συνδυάζει όρους όπως `host`, `net`, `port`, `src`, `dst` και `portrange` σε μία συνθήκη. Επειδή το filter εφαρμόζεται πριν φτάσουν τα πακέτα στο tcpdump, ένα στενό filter μειώνει και τη δουλειά του capture host.

Τα εισαγωγικά μετράνε: το `'port 80'` περνά στον compiler του filter ως ένα όρισμα, ενώ ένα άκλειδο `port 80` θα διαβαζόταν από το κέλυφος ως η λέξη `port` και μετά ως ξεχωριστό όρισμα το `80`.

</details>

## Κατεύθυνση και link-layer πληροφορίες

Σε συστήματα που υποστηρίζουν capture direction, περιορίστε την έξοδο σε εισερχόμενα ή εξερχόμενα πακέτα:

```bash
sudo tcpdump -i eth0 -c 5 -nn -Q in
sudo tcpdump -i eth0 -c 5 -nn -Q out
```

Η `-Q in` εμφανίζει πακέτα που λαμβάνει το interface, ενώ η `-Q out` πακέτα που στέλνει. Η υποστήριξη εξαρτάται από platform και driver.

Συμπεριλάβετε headers επιπέδου Ethernet:

```bash
sudo tcpdump -i eth0 -c 2 -nn -e
```

**Έξοδος:**

```text
20:35:44.200001 08:00:27:aa:bb:01 > 08:00:27:aa:bb:20, ethertype IPv4 (0x0800), length 74: 192.168.56.30.49152 > 192.168.56.20.80: Flags [S], length 0
```

Η `-e` εκτυπώνει link-layer πληροφορίες, όπως source και destination MAC addresses σε Ethernet captures. Το `ethertype IPv4 (0x0800)` αναγνωρίζει το network-layer protocol που μεταφέρει το frame.

<details>
<summary><strong>Περισσότερα</strong> — πότε αξίζουν η κατεύθυνση και οι MAC addresses</summary>

Αν το tcpdump αναφέρει ότι η direction δεν υποστηρίζεται, χρησιμοποιήστε host, address ή port filter. Η directional capture είναι πιο χρήσιμη όταν ξέρετε ήδη ποια πλευρά της συνομιλίας θέλετε να δείτε, για παράδειγμα τις απαντήσεις ενός server.

Οι MAC addresses έχουν συνήθως σημασία μόνο στο τοπικό Layer-2 segment: αλλάζουν σε κάθε router, οπότε μια MAC σε ένα capture σας λέει για το κοντινότερο switch ή gateway, όχι για τον αρχικό αποστολέα.

</details>

## Verbosity και timestamps

Αυξήστε το protocol detail μόνο όταν είναι χρήσιμο:

```bash
sudo tcpdump -i eth0 -c 2 -nn -v
sudo tcpdump -i eth0 -c 2 -nn -vv
sudo tcpdump -i eth0 -c 2 -nn -vvv
```

**Έξοδος (verbose packet):**

```text
IP (tos 0x0, ttl 64, id 22301, offset 0, flags [DF], proto TCP (6), length 60)
    192.168.56.30.49152 > 192.168.56.20.80: Flags [S], cksum 0xabcd (correct), seq 125430, win 64240, options [mss 1460,sackOK,TS val 100 ecr 0,nop,wscale 7], length 0
```

Το τμήμα IPv4 περιλαμβάνει type of service, time-to-live, packet ID, fragmentation πληροφορίες, protocol number και συνολικό μήκος. Το τμήμα TCP περιλαμβάνει flags, checksum status, sequence number, receive window και negotiation options.

Η επιπλέον verbosity κάνει το troubleshooting ευκολότερο αλλά αυξάνει τον όγκο εξόδου.

Οι μορφές timestamps αλλάζουν την παρουσίαση, όχι το περιεχόμενο των πακέτων:

```bash
sudo tcpdump -i eth0 -c 2 -nn -t
sudo tcpdump -i eth0 -c 2 -nn -tt
sudo tcpdump -i eth0 -c 2 -nn -ttt
sudo tcpdump -i eth0 -c 2 -nn -tttt
```

Η `-t` αποκρύπτει timestamps. Η `-tt` εμφανίζει απόλυτο Unix-style timestamp. Η `-ttt` εμφανίζει τη διαφορά χρόνου από το προηγούμενο πακέτο, χρήσιμη για latency και timing analysis. Η `-tttt` δείχνει ημερομηνία και ώρα σε human-readable μορφή.

<details>
<summary><strong>Περισσότερα</strong> — επιλέγοντας το σωστό επίπεδο λεπτομέρειας</summary>

Κάθε `v` προσθέτει ένα επίπεδο: η `-v` προσθέτει IP και TCP λεπτομέρειες, η `-vv` περισσότερα protocol fields, και η `-vvv` προορίζεται για πραγματικά βαθύ protocol debugging. Ξεκινήστε από το χαμηλότερο επίπεδο που απαντά στην ερώτησή σας, γιατί κάθε επιπλέον πεδίο είναι μια ακόμη στήλη για διάβασμα.

Οι διατάξεις timestamps ποικίλλουν και ανά έκδοση, οπότε δείτε την ακριβή μορφή εδώ ως οδηγό και όχι ως εγγύηση.

</details>

## Payload rendering και snapshot length

Η εμφάνιση payload μπορεί να αποκαλύψει plaintext application content και απαιτεί ιδιαίτερη προσοχή. Σε lab, ελέγξτε περιορισμένο αριθμό πακέτων σε ASCII:

```bash
sudo tcpdump -i eth0 -c 2 -nn -A 'tcp port 8080'
```

**Έξοδος:**

```text
GET /health HTTP/1.1
Host: 192.168.56.20:8080
User-Agent: lab-client/1.0
```

Η `-A` εκτυπώνει payload ως ASCII όπου είναι δυνατό. Τα δεδομένα που φαίνονται είναι HTTP request: το `GET` ζητά τη διαδρομή `/health`, το `Host` δηλώνει τον virtual host και το port, και το `User-Agent` δείχνει το client software string.

Μην χρησιμοποιείτε αυτή την τεχνική για συλλογή passwords, cookies, authorization headers ή user content.

Δείτε τα bytes των πακέτων σε hexadecimal και ASCII:

```bash
sudo tcpdump -i eth0 -c 1 -nn -X 'tcp port 8080'
```

**Έξοδος:**

```text
0x0000:  4500 0054 1a2b 4000 4006 ...
0x0030:  4745 5420 2f68 6561 6c74 6820 4854 5450  GET /health HTTP
```

Η `-X` εκτυπώνει hexadecimal παράσταση μαζί με ASCII decoding. Οι τιμές στα αριστερά είναι byte offsets.

Το snapshot length περιορίζει πόσα bytes διατηρεί το tcpdump από κάθε πακέτο. Για σκόπιμα περιορισμένη capture:

```bash
sudo tcpdump -i eth0 -c 2 -nn -s 96 'tcp port 8080'
```

**Έξοδος:**

```text
tcpdump: listening on eth0, link-type EN10MB (Ethernet), snapshot length 96 bytes
```

Η `-s 96` κρατά έως 96 bytes ανά πακέτο. Μικρότερο snapshot μειώνει τα αποθηκευμένα δεδομένα και τον όγκο της συλλογής, αλλά μπορεί να περικόψει headers ή payload που χρειάζονται για ανάλυση.

<details>
<summary><strong>Περισσότερα</strong> — η οικογένεια των payload switches, και τι κοστίζει ένα snapshot</summary>

Το ASCII δεξιά της `-X` κάνει ορατό το αναγνωρίσιμο plaintext, ενώ το hex είναι χρήσιμο για non-printable fields. Οι σχετικές παράμετροι είναι η `-x` (hex χωρίς ASCII), η `-xx` (προσθέτει link-layer header στο hex) και η `-XX` (link-layer hex μαζί με ASCII).

Το σύγχρονο tcpdump χρησιμοποιεί συνήθως πολύ μεγαλύτερο default snapshot length, οπότε ελέγχετε τη γραμμή εκκίνησης αντί να υποθέτετε σταθερή τιμή. Ένα πολύ μικρό snapshot κόβει σιωπηλά τα headers που μπορεί να χρειαστείτε αργότερα, και η περικοπή δεν είναι εμφανής με την πρώτη ματιά.

</details>

## TCP flags και διάγνωση συνδέσεων

Τα TCP flags εξηγούν την κατάσταση μιας σύνδεσης. Καταγράψτε αρχικά SYN πακέτα με bit-mask filter:

```bash
sudo tcpdump -i eth0 -nn -c 5 'tcp[tcpflags] & (tcp-syn|tcp-ack) == tcp-syn'
```

**Έξοδος:**

```text
IP 192.168.56.30.49152 > 192.168.56.20.80: Flags [S], seq 125430, win 64240, length 0
```

Η έκφραση ταιριάζει με πακέτα όπου το SYN είναι set και το ACK δεν είναι, κάτι που συνήθως αναγνωρίζει αρχικές προσπάθειες σύνδεσης. Το bit-mask είναι πιο αξιόπιστο από strict equality, γιατί ένα πακέτο μπορεί να περιέχει επιπλέον flags.

Καταγράψτε πακέτα TCP reset:

```bash
sudo tcpdump -i eth0 -nn -c 5 'tcp[tcpflags] & tcp-rst != 0'
```

**Έξοδος:**

```text
IP 192.168.56.20.80 > 192.168.56.30.49152: Flags [R.], seq 1, ack 1, win 0, length 0
```

Το RST δηλώνει απότομο TCP reset. Μπορεί να προκύψει από κλειστή θύρα, application abort, firewall device ή network middlebox. Συσχετίστε το με service state και logs πριν αποφασίσετε την αιτία.

Εμφανίστε απόλυτους TCP sequence numbers όταν χρειάζεται:

```bash
sudo tcpdump -i eth0 -nn -S -c 3 tcp
```

Η `-S` εκτυπώνει απόλυτους sequence numbers αντί για relative. Είναι χρήσιμη για λεπτομερές protocol troubleshooting, αλλά συνήθως δεν απαιτείται σε routine review.

<details>
<summary><strong>Περισσότερα</strong> — διαβάζοντας τα flags ως καταστάσεις, όχι ως συμπεράσματα</summary>

Κάθε flag είναι ένα bit και ένα πακέτο μπορεί να φέρει πολλά μαζί: το `S` είναι SYN, το `S.` είναι SYN-ACK, μια μόνη `.` σημαίνει ACK χωρίς άλλο flag, και το `R.` είναι reset με ACK. Το διάβασμα των αγκυλών είναι ο γρηγορότερος τρόπος να ακολουθήσετε ένα handshake.

Ένα filter όπως `tcp[tcpflags] & tcp-syn != 0` ταιριάζει με κάθε πακέτο που έχει SYN set, ενώ η μορφή `(tcp-syn|tcp-ack) == tcp-syn` είναι αυστηρότερη και διαλέγει μόνο το αρχικό SYN. Κανένα δεν αποδεικνύει πρόθεση: ένα SYN flood και μια νόμιμη σύνδεση ξεκινούν με ακριβώς το ίδιο πακέτο.

</details>

## PCAP αρχεία: εγγραφή και ανάγνωση

Γράψτε raw packets σε PCAP για μεταγενέστερη ανάλυση. Αποφύγετε unlimited capture:

```bash
sudo tcpdump -i eth0 -c 100 -nn -w lab-web-test.pcap 'tcp port 8080'
ls -lh lab-web-test.pcap
```

**Έξοδος:**

```text
100 packets captured
-rw-r----- 1 root root 42K Oct  8 20:40 lab-web-test.pcap
```

Η `-w` γράφει binary packet records αντί για readable γραμμές. Το PCAP που προκύπτει ανοίγει με tcpdump ή graphical analyzer.

Τα permissions και ο owner που δείχνει η `ls -lh` πρέπει να ελέγχονται, γιατί τα capture files μπορεί να περιέχουν ευαίσθητα payloads.

Διαβάστε το αρχείο χωρίς να χρειάζεστε live traffic:

```bash
tcpdump -nn -r lab-web-test.pcap
```

**Έξοδος:**

```text
reading from file lab-web-test.pcap, link-type EN10MB (Ethernet), snapshot length 262144
20:40:01.000010 IP 192.168.56.30.49152 > 192.168.56.20.8080: Flags [S], length 0
```

Η `-r` διαβάζει πακέτα από το αποθηκευμένο PCAP. Η κεφαλίδα δείχνει το link type και το snap length που καταγράφηκαν στο αρχείο.

Αυτή η επαναλήψιμη ροή επιτρέπει την ανταλλαγή φιλτραρισμένων, προστατευμένων στοιχείων αντί για την επανάληψη δραστηριότητας σε live δίκτυο.

<details>
<summary><strong>Περισσότερα</strong> — γιατί τα capture files είναι στοιχεία, όχι πρόχειρα αρχεία</summary>

Επειδή ένα PCAP είναι εγγραφή byte προς byte, πρέπει να αντιμετωπίζεται όπως κάθε άλλο στοιχείο: το πρωτότυπο μένει ανέπαφο, καταγράφεται hash, και κάθε φιλτραρισμένο ή παράγωγο αρχείο δημιουργείται χωριστά με όνομα που το δηλώνει. Ένα αρχείο `-w` με λάθος permissions είναι διαρροή που περιμένει να καταγραφεί.

Η ανάγνωση με `-r` σημαίνει επίσης ότι το ίδιο capture μπορεί να εξεταστεί ξανά με άλλα filters πολύ μετά το συμβάν — κάτι που μια προσέγγιση μόνο live δεν μπορεί να κάνει.

</details>

## Γραφική ανάλυση πακέτων

Ένα PCAP μπορεί να μεταφερθεί με ασφάλεια σε approved analysis workstation και να ανοιχθεί στο Wireshark. Η ακόλουθη ASCII παράσταση αντικαθιστά ένα GUI screenshot:

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

Το επάνω τμήμα εμφανίζει packet summaries, το μεσαίο αναπτύσσει protocol fields και το κάτω εμφανίζει raw bytes. Ένας graphical analyzer είναι ιδιαίτερα χρήσιμος για follow TCP stream, display filters και protocol trees.

Διατηρήστε το αρχικό PCAP και καταγράψτε χωριστά κάθε παράγωγο ή φιλτραρισμένο αρχείο.

<details>
<summary><strong>Περισσότερα</strong> — πότε ένα γραφικό εργαλείο κερδίζει τη γραμμή εντολών</summary>

Το tcpdump είναι γρήγορο και scriptable, αλλά η ανασύσταση μιας ολόκληρης TCP ροής με το μάτι είναι κουραστική. Ένας graphical analyzer επανασυνθέτει τη ροή, επιτρέπει διαδραστικά display filters και δείχνει το protocol tree ενός frame — τα ίδια δεδομένα που συλλέγει το tcpdump, με άλλη παρουσίαση.

Δείτε τα δύο ως συμπληρωματικά: συλλογή με tcpdump και ανάλυση του προστατευμένου αντιγράφου στο γραφικό εργαλείο.

</details>

## Remote capture με safeguards

Για εξουσιοδοτημένο remote host, γράψτε το PCAP τοπικά αντί να στέλνετε live binary stream σε μη έμπιστο προορισμό:

```bash
sudo tcpdump -i eth0 -c 50 -nn -w /var/tmp/lab-capture.pcap 'tcp port 8080'
sudo chmod 0640 /var/tmp/lab-capture.pcap
```

Η πρώτη εντολή κάνει bounded capture στον remote host. Η δεύτερη περιορίζει την read access στον owner και στην group του αρχείου.

Μεταφέρετέ το μόνο μέσω approved authenticated channel και διαγράψτε το σύμφωνα με την evidence-retention policy σας. Αποφεύγετε εντολές που εκθέτουν passwords σε shell, logs, command history ή capture data.

<details>
<summary><strong>Περισσότερα</strong> — γιατί να μη μεταδίδετε ένα capture μέσα από το δίκτυο</summary>

Η διοχέτευση ζωντανής συλλογής μέσα από το δίκτυο κάνει τη μεταφορά δεύτερο, απροστάτευτο αντίγραφο: το binary stream μπορεί να υποκλαπεί, να καταγραφεί ή να αναπαραχθεί, και μπορεί να μεταφέρει ακριβώς τα credentials που προσπαθούσατε να παρατηρήσετε. Η τοπική συλλογή και η αντιγραφή του τελικού αρχείου υπό τον έλεγχό σας κρατά τα δεδομένα σε ένα όριο εμπιστοσύνης τη φορά.

Το `chmod 0640` είναι σκόπιμο: read/write για τον owner, read για τη group, τίποτα για τους υπόλοιπους — που ταιριάζει με το πώς πρέπει να κάθεται στον δίσκο ένα ευαίσθητο τεκμήριο μέχρι να παραληφθεί.

</details>

## Promiscuous mode

Σε πολλά Ethernet interfaces το tcpdump ζητά promiscuous mode ώστε το capture library να λαμβάνει frames που παρουσιάζονται στο interface. Σε switched networks, αυτό δεν εκθέτει αυτόματα όλο το traffic άλλων hosts.

Για να μην ζητηθεί promiscuous mode:

```bash
sudo tcpdump -i eth0 --no-promiscuous-mode -c 5 -nn
```

Η εντολή καταγράφει traffic που παραδίδεται κανονικά στο host interface χωρίς να αλλάζει το interface σε promiscuous mode. Είναι συχνά κατάλληλη για τη διάγνωση του traffic του ίδιου του host.

<details>
<summary><strong>Περισσότερα</strong> — τι κάνει και τι δεν κάνει το promiscuous mode</summary>

Η switch port configuration, το VLAN membership και το hardware καθορίζουν ποια frames φτάνουν πραγματικά στο interface, οπότε το promiscuous mode μόνο του δεν μετατρέπει ένα switched δίκτυο σε hub. Για να δείτε traffic άλλων hosts χρειάζεστε συνήθως mirror port, TAP ή σημείο συλλογής που βρίσκεται ήδη στη διαδρομή.

Το αν η παράμετρος τηρείται εξαρτάται από το λειτουργικό και τον driver του interface, γι' αυτό η διαθεσιμότητα και η συμπεριφορά πρέπει να επιβεβαιώνονται στον στόχο αντί να υποθέτονται.

</details>

## Καλές πρακτικές capture

- Ξεκινήστε με συγκεκριμένο ερώτημα, όπως «Λαμβάνει ο test client DNS reply;» ή «Ολοκληρώνει το local test service TCP handshake;».
- Επιλέξτε το σωστό interface και επαληθεύστε το με `tcpdump -D` και `ip -brief address`.
- Χρησιμοποιήστε στενά BPF filters, όριο πλήθους πακέτων και κατάλληλο snap length.
- Προτιμήστε `-nn` για αποφυγή name-resolution delays και ασαφών labels.
- Αντιμετωπίστε τα `-A`, `-X` και τα PCAP ως ευαίσθητα, γιατί μπορεί να αποκαλύψουν plaintext content.
- Καταγράψτε time zone, host, interface, capture filter, tool version και file hashes στις σημειώσεις διερεύνησης.
- Συσχετίστε τα πακέτα με service logs, firewall logs, routing configuration και authorized test activity πριν εξάγετε συμπεράσματα.

<details>
<summary><strong>Περισσότερα</strong> — ολόκληρη η λίστα ελέγχου σε μία ιδέα</summary>

Πρώτα το ερώτημα, μετά το interface, μετά το filter, πάντα τα όρια. Κάθε στοιχείο της λίστας απαντά στην ίδια ανάγκη: να μπορείτε να πείτε αργότερα *γιατί* ένα πακέτο ήταν εντός πεδίου και *τι* αποδεικνύει ή δεν αποδεικνύει.

</details>

## Βασικά μαθήματα

Το tcpdump είναι πιο αποτελεσματικό όταν συλλέγει την ελάχιστη traffic που χρειάζεται για να απαντήσει σε ένα συγκεκριμένο troubleshooting ή security ερώτημα. Η επιλογή interface, τα BPF filters, τα όρια πακέτων και η ασφαλής διαχείριση στοιχείων μετράνε όσο και η γνώση πρωτοκόλλων.

Ένα SYN, ένα ICMP request ή μια plaintext HTTP γραμμή είναι παρατήρηση που απαιτεί επιβεβαίωση με context — όχι συμπέρασμα από μόνο του.

<details>
<summary><strong>Περισσότερα</strong> — η μία πρόταση που αξίζει να θυμάστε</summary>

Συλλέξτε τα λιγότερα δυνατά, σε σύστημα που επιτρέπεται να παρακολουθείτε, και αντιμετωπίστε κάθε πακέτο ως παρατήρηση προς εξήγηση και όχι ως ετυμηγορία προς ανακοίνωση.

</details>
