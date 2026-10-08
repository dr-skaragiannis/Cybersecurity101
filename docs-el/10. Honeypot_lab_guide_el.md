# Λειτουργία Low-Interaction Honeypot σε Απομονωμένο Εργαστήριο

> **Μόνο για εξουσιοδοτημένη αμυντική χρήση.** Ένα honeypot είναι υπηρεσία-δόλωμα που παρατηρεί μη αναμενόμενες προσπάθειες πρόσβασης. Χρησιμοποιήστε το μόνο σε ελεγχόμενο εργαστήριο ή σε οργανισμό με γραπτή άδεια, τεκμηριωμένο monitoring, κανόνες διατήρησης δεδομένων και αυστηρή απομόνωση δικτύου. Μην κάνετε αντίποινα σε διευθύνσεις πηγής και μην χρησιμοποιείτε honeypot ως πλατφόρμα σάρωσης ή επίθεσης.

## Σκοπός και περιορισμοί

Ένα low-interaction honeypot ακούει σε επιλεγμένη θύρα δικτύου, καταγράφει απόπειρες σύνδεσης και μπορεί να επιστρέφει ένα απλό banner ή προειδοποιητικό μήνυμα. Είναι χρήσιμο για εκμάθηση του τρόπου με τον οποίο εμφανίζονται στα logs οι σαρώσεις Internet και τα μη εξουσιοδοτημένα probes. Δεν αποτελεί πλήρη προσομοίωση SSH, HTTP, FTP ή άλλου πρωτοκόλλου και δεν αντικαθιστά IDS, patching, authentication ή firewall παραγωγικού περιβάλλοντος.

Μια ασφαλής σχεδίαση τοποθετεί το decoy σε dedicated VM ή isolated VLAN. Επιτρέψτε μόνο τις inbound θύρες που θέλετε να παρακολουθείτε, περιορίστε το outbound traffic σε απαραίτητους management προορισμούς και στείλτε logs σε ξεχωριστό collector όταν είναι εφικτό. Ο κύριος στόχος ασφάλειας είναι το containment: ακόμη και αν το decoy παραβιαστεί, δεν πρέπει να παρέχει πρόσβαση σε παραγωγικά συστήματα ή στο ευρύτερο Internet.

## Αρχιτεκτονική αναφοράς

```text
                  Εξουσιοδοτημένος test client
                           |
                           | TCP σύνδεση στην επιλεγμένη θύρα
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
                                                         | μόνο περιορισμένα logs
                                                         v
                                             +------------------------+
                                             | Log analysis workstation|
                                             +------------------------+
```

Ο test client πρέπει να είναι μηχάνημα που ελέγχετε. Το firewall πρέπει να αποτρέπει το honeypot από το να φτάσει σε production δίκτυα και να αρνείται περιττές outbound συνδέσεις. Ξεχωριστό log-analysis host μειώνει τον κίνδυνο αλλοίωσης στοιχείων αν παραβιαστεί το decoy.

## Προετοιμασία εργαστηριακού host

Εντοπίστε λειτουργικό, τρέχοντα λογαριασμό, IP και υπηρεσίες που ήδη ακούν:

```bash
cat /etc/os-release
id
ip -brief address
sudo ss -tulpn
```

Ενδεικτική έξοδος:

```text
PRETTY_NAME="Kali GNU/Linux Rolling"
uid=1000(analyst) gid=1000(analyst) groups=1000(analyst),27(sudo)
eth0             UP             192.168.56.20/24
Netid State  Local Address:Port  Process
LISTEN 0      128    127.0.0.1:631  users:(('cupsd',pid=621,fd=6))
```

Οι πληροφορίες λειτουργικού εξασφαλίζουν σωστές εντολές package management. Η `id` επιβεβαιώνει τον local user και τα administrative privileges. Η `ip -brief address` εμφανίζει interface και lab IP. Η `ss -tulpn` κάνει inventory των TCP/UDP υπηρεσιών που ήδη ακούν. Μην ξεκινάτε decoy σε port που χρησιμοποιείται από πραγματική υπηρεσία.

Ενημερώστε το host πριν προσθέσετε εργαστηριακό λογισμικό:

```bash
sudo apt update
sudo apt upgrade -y
```

Ενδεικτική έξοδος:

```text
Hit:1 http://http.kali.org/kali kali-rolling InRelease
Reading package lists... Done
Building dependency tree... Done
0 upgraded, 0 newly installed, 0 to remove
```

Η `apt update` ανακτά package metadata. Η `apt upgrade -y` εφαρμόζει διαθέσιμες ενημερώσεις χωρίς prompt για κάθε πακέτο. Η τελευταία γραμμή δείχνει ότι δεν υπήρχαν εκκρεμείς αναβαθμίσεις. Ακόμη και σε lab, ο host πρέπει να είναι ενημερωμένος· δεν πρέπει να είναι σκόπιμα ευάλωτος.

## Εγκατάσταση και έλεγχος εργαλείου

Ένα Ruby-based low-interaction honeypot μπορεί να διανέμεται ως source code. Κλωνοποιήστε μόνο reviewed και authorized training copy σε dedicated directory:

```bash
sudo apt install -y git ruby
mkdir -p ~/honeypot-lab
cd ~/honeypot-lab
git clone https://example.invalid/training/pentbox.git pentbox
cd pentbox
ruby --version
```

Ενδεικτική έξοδος:

```text
Cloning into 'pentbox'...
done.
ruby 3.1.xp0 (202x-xx-xx revision ...) [x86_64-linux-gnu]
```

Η `git` ανακτά version-controlled source και η `ruby` παρέχει τον interpreter. Το URL είναι σκόπιμα μη δρομολογήσιμο placeholder· αντικαταστήστε το μόνο με repository που έχει εγκριθεί και εξεταστεί από τον οργανισμό σας. Η Ruby version επιβεβαιώνει ότι υπάρχει runtime. Διαβάστε τον κώδικα και μην εκτελείτε άγνωστα scripts ως root.

Ελέγξτε αρχεία και βοήθεια πριν εκτελέσετε πρόγραμμα:

```bash
find . -maxdepth 2 -type f | sort | head -30
sed -n '1,80p' pentbox.rb
ruby pentbox.rb --help
```

Ενδεικτική έξοδος:

```text
./LICENSE
./README.md
./pentbox.rb
./other/log_honeypot.txt
Usage: pentbox.rb [options]
```

Η `find` εμφανίζει σύντομο inventory. Η `sed` εμφανίζει μόνο το αρχικό τμήμα του script για review. Η βοήθεια διαφέρει ανά έκδοση. Αν ένα script δεν υποστηρίζει `--help`, σταματήστε και διαβάστε την τοπική τεκμηρίωση, αντί να μαντέψετε τη συμπεριφορά του.

## Ρύθμιση ελάχιστου decoy

Επιλέξτε αρχικά υψηλή, αχρησιμοποίητη TCP θύρα, όπως `8088`, αντί να μιμηθείτε privileged ή production service. Επιβεβαιώστε ότι είναι ελεύθερη:

```bash
sudo ss -ltnp '( sport = :8088 )'
```

Αναμενόμενη έξοδος πριν ξεκινήσει το decoy:

```text
State  Recv-Q Send-Q Local Address:Port Peer Address:Port Process
```

Η κεφαλίδα χωρίς γραμμές δεδομένων σημαίνει ότι κανένα TCP process δεν ακούει στη θύρα 8088. Αν εμφανιστεί γραμμή, επιλέξτε άλλη θύρα ή σταματήστε το συγκρουόμενο lab service μόνο αν έχετε εξουσιοδότηση.

Η τυπική menu-based ροή low-interaction εργαλείου μπορεί να μοιάζει με:

```text
1) Network tools
2) Honeypot
3) Manual configuration

Listening port: 8088
Banner: Training decoy — access is monitored.
Log path: ./other/log_honeypot.txt
```

Χρησιμοποιήστε σαφές banner και dedicated log path. Σε internal lab, το banner βοηθά να ξεχωρίζετε το decoy από production service. Μην ισχυρίζεστε ότι η υπηρεσία είναι πραγματική τράπεζα, κυβερνητική υπηρεσία ή άλλος οργανισμός.

Μετά την εκκίνηση listener, επιβεβαιώστε τοπικά τη θύρα:

```bash
sudo ss -ltnp '( sport = :8088 )'
```

Ενδεικτική έξοδος:

```text
State  Recv-Q Send-Q Local Address:Port Peer Address:Port Process
LISTEN 0      128    0.0.0.0:8088  0.0.0.0:*     users:(('ruby',pid=1842,fd=7))
```

Το `LISTEN` σημαίνει ότι process περιμένει νέες TCP συνδέσεις. Το `0.0.0.0:8088` σημαίνει ότι δέχεται IPv4 συνδέσεις σε όλα τα interfaces. Για αυστηρά τοπικό test, προτιμήστε bind στο `127.0.0.1` αν το εργαλείο το υποστηρίζει. Το PID και το process name βοηθούν σε monitoring και ελεγχόμενο shutdown.

## Firewall containment

Το host firewall πρέπει να επιτρέπει μόνο intended lab access και να περιορίζει περιττή outbound δραστηριότητα. Η ακόλουθη UFW policy είναι εννοιολογικό παράδειγμα για isolated subnet. Προσαρμόστε τη στη πραγματική management path και δοκιμάστε από console πριν τερματίσετε SSH:

```bash
sudo ufw default deny incoming
sudo ufw default deny outgoing
sudo ufw allow from 192.168.56.0/24 to any port 22 proto tcp
sudo ufw allow from 192.168.56.0/24 to any port 8088 proto tcp
sudo ufw allow out to 192.168.56.10 port 514 proto udp
sudo ufw enable
sudo ufw status numbered
```

Ενδεικτική έξοδος:

```text
Status: active

     To                         Action      From
[ 1] 22/tcp                     ALLOW IN    192.168.56.0/24
[ 2] 8088/tcp                   ALLOW IN    192.168.56.0/24
[ 3] 514/udp                    ALLOW OUT   192.168.56.10
```

Οι default-deny πολιτικές απορρίπτουν inbound και outbound traffic, εκτός αν μεταγενέστερος κανόνας το επιτρέπει. Ο κανόνας 1 επιτρέπει SSH management μόνο από lab subnet. Ο κανόνας 2 επιτρέπει πρόσβαση στο decoy μόνο από το ίδιο subnet. Ο κανόνας 3 επιτρέπει UDP syslog forwarding μόνο προς τον log collector. Αν χρειάζονται DNS, NTP, package updates ή άλλη management υπηρεσία, προσθέστε στενούς προσωρινούς κανόνες και αφαιρέστε τους μετά.

## Δοκιμή από εξουσιοδοτημένο client

Από ξεχωριστό client μέσα στο isolated subnet, ελέγξτε μόνο το listener που δημιουργήσατε:

```bash
nc -vz 192.168.56.20 8088
```

Ενδεικτική έξοδος:

```text
Connection to 192.168.56.20 8088 port [tcp/*] succeeded!
```

Το Netcat επιχειρεί TCP σύνδεση. Το success message επιβεβαιώνει network reachability, firewall allowance και listener στη target port. Δεν αποδεικνύει ότι υλοποιείται πλήρες application protocol.

Για παρατήρηση text banner, εφόσον το εργαλείο επιστρέφει banner:

```bash
nc 192.168.56.20 8088
```

Ενδεικτική έξοδος:

```text
Training decoy — access is monitored.
```

Η εντολή δημιουργεί interactive σύνδεση και εμφανίζει το banner του decoy. Τερματίστε με `Ctrl+C`. Μην στέλνετε exploit payloads· μια απλή connection test αρκεί για validation logging και containment.

## Παρακολούθηση και ερμηνεία logs

Παρακολουθήστε το decoy log σε πραγματικό χρόνο:

```bash
tail -F ./other/log_honeypot.txt
```

Ενδεικτική έξοδος:

```text
2026-10-08T20:25:14+03:00 connection source=192.168.56.30:49152 destination_port=8088
```

Η `tail -F` συνεχίζει να παρακολουθεί το αρχείο ακόμη και αν γίνει rotate ή αναδημιουργηθεί. Το timestamp βοηθά στην ανακατασκευή γεγονότων. Η source IP και η ephemeral source port αναγνωρίζουν τον connecting test client, ενώ η `destination_port` αναφέρει την decoy service.

Ένα console μήνυμα μπορεί να μοιάζει με:

```text
CONNECTION ATTEMPT DETECTED from 192.168.56.30:49152
```

Αυτό υποδηλώνει ότι ένα απομακρυσμένο endpoint έκανε ή επιχείρησε σύνδεση. Είναι event, όχι απόδειξη κακόβουλης πρόθεσης: health checks, scanners, misconfigurations και authorized tests μπορούν να δημιουργήσουν παρόμοια entries. Διατηρήστε το περιβάλλον γύρω από το γεγονός πριν το ταξινομήσετε.

Ελέγξτε πρόσφατες εγγραφές:

```bash
tail -n 50 ./other/log_honeypot.txt
```

Παράδειγμα εξόδου:

```text
2026-10-08T20:25:14+03:00 connection source=192.168.56.30:49152 destination_port=8088
2026-10-08T20:26:03+03:00 connection source=192.168.56.31:49411 destination_port=8088
```

Οι δύο εγγραφές δείχνουν ξεχωριστές προσπάθειες από διαφορετικούς hosts και source ports. Η source port συνήθως αλλάζει μεταξύ outbound συνδέσεων και δεν είναι αξιόπιστη ταυτότητα χρήστη ή συσκευής. Συσχετίστε τη με firewall logs, DHCP leases, asset inventory και approved test records.

## Βασική περίληψη logs

Εξαγάγετε source addresses και μετρήστε τες:

```bash
awk -F'source=' '/source=/{split($2,a,":"); print a[1]}' ./other/log_honeypot.txt | sort | uniq -c | sort -nr
```

Ενδεικτική έξοδος:

```text
      8 192.168.56.30
      2 192.168.56.31
```

Η `awk` επιλέγει entries με `source=` και απομονώνει source address πριν από το colon. Η `sort` ομαδοποιεί ίδιες διευθύνσεις, η `uniq -c` μετρά occurrences και η τελική `sort -nr` ταξινομεί αριθμητικά από τη μεγαλύτερη συχνότητα. Η μέτρηση αναδεικνύει συχνή επικοινωνία, αλλά δεν αποδεικνύει κακόβουλη πρόθεση ή attribution.

## Ασφαλής διατήρηση στοιχείων

Κρατήστε αντίγραφο log πριν την ανάλυση και περιορίστε permissions:

```bash
install -d -m 0750 ~/honeypot-evidence
cp --preserve=mode,timestamps ./other/log_honeypot.txt ~/honeypot-evidence/log_$(date +%F_%H%M%S).txt
chmod 0640 ~/honeypot-evidence/*.txt
sha256sum ~/honeypot-evidence/*.txt
```

Ενδεικτική έξοδος:

```text
c9a4...e17b  /home/analyst/honeypot-evidence/log_2026-10-08_202700.txt
```

Το directory mode περιορίζει πρόσβαση σε owner και group. Η `cp --preserve` διατηρεί χρήσιμα metadata. Το SHA-256 digest είναι fingerprint του evidence file: αν το αρχείο αλλάξει, συνήθως αλλάζει και το digest. Ένα hash τεκμηριώνει ακεραιότητα σε σχέση με καταγεγραμμένη τιμή, αλλά δεν αποδεικνύει ποιος δημιούργησε το event.

## Ελεγχόμενος τερματισμός

Αν το decoy τρέχει στο foreground, χρησιμοποιήστε `Ctrl+C`. Επιβεβαιώστε ότι δεν ακούει πλέον η θύρα:

```bash
sudo ss -ltnp '( sport = :8088 )'
```

Αναμενόμενη έξοδος:

```text
State  Recv-Q Send-Q Local Address:Port Peer Address:Port Process
```

Η κενή έξοδος δεδομένων σημαίνει ότι δεν υπάρχει πλέον TCP listener στη θύρα 8088. Στη συνέχεια αφαιρέστε το προσωρινό firewall access αν δεν χρειάζεται:

```bash
sudo ufw status numbered
sudo ufw delete allow from 192.168.56.0/24 to any port 8088 proto tcp
```

Η πρώτη εντολή πρέπει να εκτελείται αμέσως πριν από διαγραφή, επειδή τα rule numbers ή το rule set μπορεί να έχουν αλλάξει. Η δεύτερη αφαιρεί την ακριβή inbound εξαίρεση για τη decoy port.

## Λειτουργικές οδηγίες

- Κρατήστε το honeypot σε isolated VLAN, private subnet ή dedicated virtual switch.
- Χρησιμοποιήστε restrictive egress rules και μην επιτρέπετε στο decoy πρόσβαση σε production ή αυθαίρετους Internet προορισμούς.
- Διατηρείτε το host OS ενημερωμένο, ακόμη και αν το decoy service είναι σκόπιμα απλό.
- Στείλτε logs εκτός decoy host όταν είναι εφικτό και προστατεύστε τα με access controls και retention policies.
- Αντιμετωπίζετε IP addresses ως ευαίσθητα operational data και ακολουθείτε privacy, retention και οργανωτικές πολιτικές.
- Κατηγοριοποιήστε logged connections προσεκτικά· μπορεί να είναι automated scanning, monitoring, misconfiguration ή authorized test.
- Ποτέ μην κάνετε counterattack ή active retaliation με βάση γεγονότα που παρατηρήθηκαν σε honeypot.

## Βασικά συμπεράσματα

Ένα ασφαλές low-interaction honeypot είναι κυρίως ελεγχόμενος sensor: ακούει σε σκόπιμη θύρα, καταγράφει επαφές και προσφέρει στους defenders χώρο για εξάσκηση σε detection και log analysis. Η ποιότητα της ανάπτυξης εξαρτάται περισσότερο από isolation, egress control, evidence handling και προσεκτική ερμηνεία παρά από το πόσο πειστικό εμφανίζεται το decoy. Ξεκινήστε με υψηλή αχρησιμοποίητη θύρα σε private lab, επαληθεύστε μία authorized connection και τεκμηριώστε τόσο το event όσο και τα firewall boundaries που το περιόρισαν.
