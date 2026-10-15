# Λειτουργία Low-Interaction Honeypot σε Απομονωμένο Εργαστήριο

> **Μόνο για εξουσιοδοτημένη αμυντική χρήση.** Ένα honeypot είναι υπηρεσία-δόλωμα που παρατηρεί μη αναμενόμενες προσπάθειες πρόσβασης. Χρησιμοποιήστε το μόνο σε ελεγχόμενο εργαστήριο ή σε οργανισμό με γραπτή άδεια, τεκμηριωμένο monitoring, κανόνες διατήρησης δεδομένων και αυστηρή απομόνωση δικτύου. Μην κάνετε αντίποινα σε διευθύνσεις πηγής και μην χρησιμοποιείτε honeypot ως πλατφόρμα σάρωσης ή επίθεσης.

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

Οι παράγραφοι μετά από κάθε έξοδο εξηγούν *τι κάνει το εργαλείο* και *γιατί έχει σημασία το βήμα*, ώστε να μάθετε τη μέθοδο, όχι μόνο τη σύνταξη.

## Σκοπός και περιορισμοί

Ένα low-interaction honeypot ακούει σε επιλεγμένη θύρα δικτύου, καταγράφει απόπειρες σύνδεσης και μπορεί να επιστρέφει ένα απλό banner ή προειδοποιητικό μήνυμα. Είναι χρήσιμο για εκμάθηση του τρόπου με τον οποίο εμφανίζονται στα logs οι σαρώσεις Internet και τα μη εξουσιοδοτημένα probes.

Δεν αποτελεί πλήρη προσομοίωση SSH, HTTP, FTP ή άλλου πρωτοκόλλου. Μην το θεωρείτε αντικαταστάτη IDS, patching, authentication ή firewall παραγωγικού περιβάλλοντος.

<details>
<summary><strong>Περισσότερα</strong> — η ασφαλής σχεδίαση και ο στόχος του containment</summary>

Μια ασφαλής σχεδίαση τοποθετεί το decoy σε dedicated VM ή isolated VLAN. Επιτρέψτε μόνο τις inbound θύρες που θέλετε να παρακολουθείτε, περιορίστε το outbound traffic σε απαραίτητους management προορισμούς και στείλτε logs σε ξεχωριστό collector όταν είναι εφικτό.

Ο κύριος στόχος ασφάλειας είναι το **containment**: ακόμη και αν το decoy παραβιαστεί, δεν πρέπει να παρέχει πρόσβαση σε παραγωγικά συστήματα ή στο ευρύτερο Internet.

</details>

## Αρχιτεκτονική αναφοράς

Το διάγραμμα παρακάτω δείχνει το επιθυμητό σχήμα του lab: ένα απομονωμένο τμήμα, ένα decoy και ένα ξεχωριστό σημείο για την ανάγνωση των logs.

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

Ο test client πρέπει να είναι μηχάνημα που ελέγχετε. Το firewall πρέπει να αποτρέπει το honeypot από το να φτάσει σε production δίκτυα και να αρνείται περιττές outbound συνδέσεις.

<details>
<summary><strong>Περισσότερα</strong> — γιατί τα στοιχεία φυλάσσονται σε τρίτο μηχάνημα</summary>

Ξεχωριστό log-analysis host μειώνει τον κίνδυνο αλλοίωσης στοιχείων αν παραβιαστεί το decoy. Μόνο περιορισμένα logs ταξιδεύουν εκεί, οπότε ένας επιτιθέμενος που ελέγχει το decoy δεν μπορεί να αγγίξει τα αρχεία με το τι έκανε.

</details>

## Προετοιμασία εργαστηριακού host

Πριν εγκαταστήσετε οτιδήποτε, βεβαιωθείτε για τρία πράγματα: ποιο λειτουργικό σύστημα τρέχει το host, ποιον λογαριασμό κρατάτε και ποιες υπηρεσίες ήδη ακούν.

```bash
cat /etc/os-release
id
ip -brief address
sudo ss -tulpn
```

**Έξοδος:**

```text
PRETTY_NAME="Kali GNU/Linux Rolling"
uid=1000(analyst) gid=1000(analyst) groups=1000(analyst),27(sudo)
eth0             UP             192.168.56.20/24
Netid State  Local Address:Port  Process
LISTEN 0      128    127.0.0.1:631  users:(('cupsd',pid=621,fd=6))
```

Οι τέσσερις εντολές περιγράφουν το host από τέσσερις οπτικές: τη διανομή, την ταυτότητά σας, τη δικτυακή διεύθυνση και τις ανοιχτές υποδοχές.

<details>
<summary><strong>Περισσότερα</strong> — τι λέει κάθε γραμμή και η σύγκρουση θύρας που αποφεύγουμε</summary>

* Η `cat /etc/os-release` εξασφαλίζει σωστές εντολές package management για τη διανομή.
* Η `id` επιβεβαιώνει τον local user και τα administrative privileges (εδώ `27(sudo)`).
* Η `ip -brief address` εμφανίζει το interface και τη lab IP (`192.168.56.20/24`).
* Η `ss -tulpn` κάνει inventory των TCP/UDP υπηρεσιών που ήδη ακούν.

Η τελευταία γραμμή μετράει περισσότερο για το επόμενο βήμα: μια πραγματική υπηρεσία είναι ήδη δεσμευμένη στο `127.0.0.1:631`. Μην ξεκινάτε decoy σε port που χρησιμοποιείται από πραγματική υπηρεσία.

</details>

Ενημερώστε το host πριν προσθέσετε εργαστηριακό λογισμικό:

```bash
sudo apt update
sudo apt upgrade -y
```

**Έξοδος:**

```text
Hit:1 http://http.kali.org/kali kali-rolling InRelease
Reading package lists... Done
Building dependency tree... Done
0 upgraded, 0 newly installed, 0 to remove
```

Η `apt update` ανακτά package metadata και η `apt upgrade -y` εφαρμόζει διαθέσιμες ενημερώσεις χωρίς prompt για κάθε πακέτο. Η τελευταία γραμμή δείχνει ότι δεν υπήρχαν εκκρεμείς αναβαθμίσεις.

<details>
<summary><strong>Περισσότερα</strong> — κρατήστε το host ενημερωμένο ακόμη και με απλό decoy</summary>

Διατηρείτε το λειτουργικό σύστημα ενημερωμένο ακόμη και όταν η decoy υπηρεσία είναι σκόπιμα απλή. Η *υπηρεσία* πρέπει να είναι τετριμμένη· το *host* δεν πρέπει να είναι σκόπιμα ευάλωτο.

</details>

## Εγκατάσταση και έλεγχος εργαλείου

Ένα Ruby-based low-interaction honeypot μπορεί να διανέμεται ως source code. Κλωνοποιήστε μόνο reviewed και authorized training copy σε dedicated directory.

```bash
sudo apt install -y git ruby
mkdir -p ~/honeypot-lab
cd ~/honeypot-lab
git clone https://example.invalid/training/pentbox.git pentbox
cd pentbox
ruby --version
```

**Έξοδος:**

```text
Cloning into 'pentbox'...
done.
ruby 3.1.xp0 (202x-xx-xx revision ...) [x86_64-linux-gnu]
```

Η `git` ανακτά version-controlled source και η `ruby` παρέχει τον interpreter. Η γραμμή `ruby --version` επιβεβαιώνει ότι υπάρχει runtime.

<details>
<summary><strong>Περισσότερα</strong> — το placeholder repository και ο κανόνας του review</summary>

Το URL είναι σκόπιμα μη δρομολογήσιμο placeholder (`example.invalid`). Αντικαταστήστε το μόνο με repository που έχει εγκριθεί και εξεταστεί από τον οργανισμό σας.

Διαβάστε τον κώδικα, επαληθεύστε την προέλευσή του και μην εκτελείτε άγνωστα scripts ως root.

</details>

Ελέγξτε αρχεία και βοήθεια πριν εκτελέσετε πρόγραμμα:

```bash
find . -maxdepth 2 -type f | sort | head -30
sed -n '1,80p' pentbox.rb
ruby pentbox.rb --help
```

**Έξοδος:**

```text
./LICENSE
./README.md
./pentbox.rb
./other/log_honeypot.txt
Usage: pentbox.rb [options]
```

Η `find` δίνει σύντομο inventory του repository. Η `sed -n '1,80p'` εμφανίζει μόνο το αρχικό τμήμα του script για review. Η βοήθεια μπορεί να διαφέρει ανά έκδοση.

<details>
<summary><strong>Περισσότερα</strong> — αν το εργαλείο δεν υποστηρίζει <code>--help</code></summary>

Αν ένα script δεν υποστηρίζει `--help`, σταματήστε και διαβάστε την τοπική τεκμηρίωση, αντί να μαντέψετε τη συμπεριφορά του. Η ανάγνωση των πρώτων ογδόντα γραμμών πριν από την εκτέλεση είναι όλη η πειθαρχία αυτού του βήματος.

</details>

## Ρύθμιση ελάχιστου decoy

Επιλέξτε αρχικά υψηλή, αχρησιμοποίητη TCP θύρα, όπως `8088`, αντί να μιμηθείτε privileged ή production service. Επιβεβαιώστε ότι είναι ελεύθερη:

```bash
sudo ss -ltnp '( sport = :8088 )'
```

**Έξοδος:**

```text
State  Recv-Q Send-Q Local Address:Port Peer Address:Port Process
```

Η κεφαλίδα χωρίς γραμμές δεδομένων σημαίνει ότι κανένα TCP process δεν ακούει στη θύρα 8088.

<details>
<summary><strong>Περισσότερα</strong> — τι κάνουμε αν εμφανιστεί γραμμή</summary>

Αν εμφανιστεί γραμμή, επιλέξτε άλλη θύρα — ή σταματήστε το συγκρουόμενο lab service, αλλά μόνο αν έχετε εξουσιοδότηση.

</details>

Μια τυπική ροή μενού για low-interaction εργαλείο μοιάζει με αυτό:

```text
1) Network tools
2) Honeypot
3) Manual configuration

Listening port: 8088
Banner: Training decoy — access is monitored.
Log path: ./other/log_honeypot.txt
```

Χρησιμοποιήστε σαφές banner και dedicated log path. Σε internal lab, το banner βοηθά να ξεχωρίζετε το decoy από production service.

<details>
<summary><strong>Περισσότερα</strong> — το μενού είναι ενδεικτικό και το banner δεν είναι μεταμφίεση</summary>

Τα ακριβή ονόματα και οι επιλογές του μενού εξαρτώνται από την εγκατεστημένη έκδοση.

Μην ισχυρίζεστε ότι η υπηρεσία είναι πραγματική τράπεζα, κυβερνητική υπηρεσία ή άλλος οργανισμός. Το banner υπάρχει ώστε ένας νόμιμος χειριστής να αναγνωρίζει το decoy γι' αυτό που είναι.

</details>

Μετά την εκκίνηση του listener, επιβεβαιώστε τοπικά τη θύρα:

```bash
sudo ss -ltnp '( sport = :8088 )'
```

**Έξοδος:**

```text
State  Recv-Q Send-Q Local Address:Port Peer Address:Port Process
LISTEN 0      128    0.0.0.0:8088  0.0.0.0:*     users:(('ruby',pid=1842,fd=7))
```

Το `LISTEN` σημαίνει ότι μια διεργασία περιμένει νέες TCP συνδέσεις και το `0.0.0.0:8088` ότι δέχεται IPv4 συνδέσεις σε όλα τα interfaces. Το PID και το process name βοηθούν σε monitoring και ελεγχόμενο shutdown.

<details>
<summary><strong>Περισσότερα</strong> — bind στο loopback αντί για κάθε interface</summary>

Για αυστηρά τοπικό test, προτιμήστε bind στο `127.0.0.1` αν το εργαλείο το υποστηρίζει. Η ακρόαση στο `0.0.0.0` εκθέτει το decoy σε κάθε interface — ακριβώς αυτό που περιορίζουν οι κανόνες firewall της επόμενης ενότητας.

</details>

## Firewall containment

Το host firewall πρέπει να επιτρέπει μόνο intended lab access και να περιορίζει περιττή outbound δραστηριότητα. Η παρακάτω policy είναι εννοιολογικό παράδειγμα UFW για isolated subnet.

```bash
sudo ufw default deny incoming
sudo ufw default deny outgoing
sudo ufw allow from 192.168.56.0/24 to any port 22 proto tcp
sudo ufw allow from 192.168.56.0/24 to any port 8088 proto tcp
sudo ufw allow out to 192.168.56.10 port 514 proto udp
sudo ufw enable
sudo ufw status numbered
```

**Έξοδος:**

```text
Status: active

     To                         Action      From
[ 1] 22/tcp                     ALLOW IN    192.168.56.0/24
[ 2] 8088/tcp                   ALLOW IN    192.168.56.0/24
[ 3] 514/udp                    ALLOW OUT   192.168.56.10
```

Οι default-deny πολιτικές απορρίπτουν μη αιτηθέν inbound και outbound traffic, εκτός αν μεταγενέστερος κανόνας το επιτρέπει.

<details>
<summary><strong>Περισσότερα</strong> — κανόνας προς κανόνα και οι προσωρινές εξαιρέσεις</summary>

* Ο κανόνας 1 επιτρέπει SSH management μόνο από το lab subnet.
* Ο κανόνας 2 επιτρέπει πρόσβαση στο decoy μόνο από το ίδιο subnet.
* Ο κανόνας 3 επιτρέπει UDP syslog forwarding μόνο προς τον log collector.

Προσαρμόστε τη policy στην πραγματική management path και δοκιμάστε από console πριν τερματίσετε το SSH. Αν χρειάζονται DNS, NTP, package updates ή άλλη management υπηρεσία, προσθέστε στενούς προσωρινούς κανόνες, ολοκληρώστε την εργασία και αφαιρέστε τους μετά.

</details>

## Δοκιμή από εξουσιοδοτημένο client

Από ξεχωριστό client μέσα στο isolated subnet, ελέγξτε μόνο το listener που δημιουργήσατε:

```bash
nc -vz 192.168.56.20 8088
```

**Έξοδος:**

```text
Connection to 192.168.56.20 8088 port [tcp/*] succeeded!
```

Το Netcat επιχειρεί TCP σύνδεση. Το success message επιβεβαιώνει network reachability, firewall allowance και listener στη target port.

<details>
<summary><strong>Περισσότερα</strong> — τι αποδεικνύει και τι δεν αποδεικνύει ένα επιτυχές <code>nc</code></summary>

Δεν αποδεικνύει ότι υλοποιείται πλήρες application protocol. Μια θύρα που απαντά δεν είναι υπηρεσία που συμπεριφέρεται σαν την πραγματική.

Μην στέλνετε exploit payloads· μια απλή connection test αρκεί για validation logging και containment.

</details>

Για παρατήρηση text banner, εφόσον το εργαλείο επιστρέφει banner:

```bash
nc 192.168.56.20 8088
```

**Έξοδος:**

```text
Training decoy — access is monitored.
```

Η εντολή δημιουργεί interactive σύνδεση και εμφανίζει το banner του decoy. Τερματίστε με `Ctrl+C`.

<details>
<summary><strong>Περισσότερα</strong> — γιατί το banner έχει σημασία για τον defender</summary>

Το banner είναι ο μόνος εξωτερικός ισχυρισμός του decoy, οπότε πρέπει να είναι ακριβές και μη απειλητικό. Δηλώνει σε έναν περιστασιακό επισκέπτη τι είναι η θύρα και στην ομάδα σας ότι βλέπει το lab και όχι πραγματικό σύστημα.

</details>

## Παρακολούθηση και ερμηνεία logs

### Παρακολούθηση του log σε πραγματικό χρόνο

Παρακολουθήστε το decoy log σε πραγματικό χρόνο:

```bash
tail -F ./other/log_honeypot.txt
```

**Έξοδος:**

```text
2026-10-08T20:25:14+03:00 connection source=192.168.56.30:49152 destination_port=8088
```

Η `tail -F` συνεχίζει να παρακολουθεί το αρχείο ακόμη και αν γίνει rotate ή αναδημιουργηθεί. Το timestamp βοηθά στην ανακατασκευή γεγονότων και η source IP με την ephemeral source port αναγνωρίζουν τον connecting test client.

Το decoy εκτυπώνει επίσης τη δική του γραμμή στην κονσόλα για κάθε επαφή:

```text
CONNECTION ATTEMPT DETECTED from 192.168.56.30:49152
```

Αυτό υποδηλώνει ότι ένα απομακρυσμένο endpoint έκανε ή επιχείρησε σύνδεση.

<details>
<summary><strong>Περισσότερα</strong> — ένα event δεν είναι απόδειξη πρόθεσης</summary>

Είναι event, όχι απόδειξη κακόβουλης πρόθεσης: health checks, scanners, misconfigurations και authorized tests μπορούν να δημιουργήσουν παρόμοια entries. Διατηρήστε το περιβάλλον γύρω από το γεγονός πριν το ταξινομήσετε.

</details>

### Έλεγχος πρόσφατων εγγραφών

Ελέγξτε πρόσφατες εγγραφές χωρίς να ακολουθείτε συνεχώς το αρχείο:

```bash
tail -n 50 ./other/log_honeypot.txt
```

**Έξοδος:**

```text
2026-10-08T20:25:14+03:00 connection source=192.168.56.30:49152 destination_port=8088
2026-10-08T20:26:03+03:00 connection source=192.168.56.31:49411 destination_port=8088
```

Οι δύο εγγραφές δείχνουν ξεχωριστές προσπάθειες από διαφορετικούς hosts και source ports.

<details>
<summary><strong>Περισσότερα</strong> — γιατί μια source port δεν είναι ταυτότητα</summary>

Η source port συνήθως αλλάζει μεταξύ outbound συνδέσεων και δεν είναι αξιόπιστη ταυτότητα χρήστη ή συσκευής. Συσχετίστε τη με firewall logs, DHCP leases, asset inventory και approved test records.

</details>

## Βασική περίληψη logs

Εξαγάγετε source addresses και μετρήστε τες:

```bash
awk -F'source=' '/source=/{split($2,a,":"); print a[1]}' ./other/log_honeypot.txt | sort | uniq -c | sort -nr
```

**Έξοδος:**

```text
      8 192.168.56.30
      2 192.168.56.31
```

Η `awk` επιλέγει entries με `source=` και απομονώνει τη source address πριν από το colon. Η `sort` ομαδοποιεί ίδιες διευθύνσεις, η `uniq -c` μετρά occurrences και η τελική `sort -nr` ταξινομεί αριθμητικά από τη μεγαλύτερη συχνότητα.

<details>
<summary><strong>Περισσότερα</strong> — τι λέει και τι δεν λέει η μέτρηση</summary>

Η μέτρηση αναδεικνύει συχνή επικοινωνία, αλλά δεν αποδεικνύει κακόβουλη πρόθεση ή attribution. Οκτώ συνδέσεις από μία διεύθυνση μπορεί να είναι scanner, monitoring probe ή συνάδελφος που εκτελεί εγκεκριμένο test.

</details>

## Ασφαλής διατήρηση στοιχείων

Κρατήστε αντίγραφο του log πριν την ανάλυση και προστατέψτε τα permissions του:

```bash
install -d -m 0750 ~/honeypot-evidence
cp --preserve=mode,timestamps ./other/log_honeypot.txt ~/honeypot-evidence/log_$(date +%F_%H%M%S).txt
chmod 0640 ~/honeypot-evidence/*.txt
sha256sum ~/honeypot-evidence/*.txt
```

**Έξοδος:**

```text
c9a4...e17b  /home/analyst/honeypot-evidence/log_2026-10-08_202700.txt
```

Το directory mode περιορίζει την πρόσβαση σε owner και group και η `cp --preserve` διατηρεί χρήσιμα metadata. Το SHA-256 digest είναι fingerprint του evidence file: αν το αρχείο αλλάξει, συνήθως αλλάζει και το digest.

<details>
<summary><strong>Περισσότερα</strong> — η ακεραιότητα δεν είναι πατρότητα</summary>

Ένα hash τεκμηριώνει ακεραιότητα σε σχέση με καταγεγραμμένη τιμή, αλλά δεν αποδεικνύει ποιος δημιούργησε το event. Αντιγράψτε το αρχείο πριν το αναλύσετε, ώστε ό,τι κάνετε μετά να μην μπορεί να αλλοιώσει το πρωτότυπο.

</details>

## Ελεγχόμενος τερματισμός

Αν το decoy τρέχει στο foreground, χρησιμοποιήστε `Ctrl+C` στο τερματικό του. Επιβεβαιώστε ότι η θύρα δεν ακούει πλέον:

```bash
sudo ss -ltnp '( sport = :8088 )'
```

**Έξοδος:**

```text
State  Recv-Q Send-Q Local Address:Port Peer Address:Port Process
```

Η κενή έξοδος σημαίνει ότι δεν υπάρχει πλέον TCP listener στη θύρα 8088.

Στη συνέχεια αφαιρέστε το προσωρινό firewall access αν το lab δεν το χρειάζεται:

```bash
sudo ufw status numbered
sudo ufw delete allow from 192.168.56.0/24 to any port 8088 proto tcp
```

Η πρώτη εντολή πρέπει να εκτελείται αμέσως πριν από τη διαγραφή, επειδή τα rule numbers και το rule set μπορεί να έχουν αλλάξει. Η δεύτερη αφαιρεί την ακριβή inbound εξαίρεση για τη decoy port.

<details>
<summary><strong>Περισσότερα</strong> — μην αφήνετε τίποτα πίσω</summary>

Η αφαίρεση της inbound εξαίρεσης επιστρέφει το host στη σκληρυμένη κατάστασή του. Αν αφήσετε έναν lab κανόνα στη θέση του, μια ξεχασμένη decoy θύρα παραμένει προσβάσιμη πολύ μετά το τέλος της άσκησης.

</details>

## Λειτουργικές οδηγίες

- Κρατήστε το honeypot σε isolated VLAN, private subnet ή dedicated virtual switch.
- Χρησιμοποιήστε restrictive egress rules και μην επιτρέπετε στο decoy πρόσβαση σε production ή αυθαίρετους Internet προορισμούς.
- Διατηρείτε το host OS ενημερωμένο, ακόμη και αν το decoy service είναι σκόπιμα απλό.
- Στείλτε logs εκτός decoy host όταν είναι εφικτό και προστατεύστε τα με access controls και retention policies.
- Αντιμετωπίζετε IP addresses ως ευαίσθητα operational data και ακολουθείτε privacy, retention και οργανωτικές πολιτικές.
- Κατηγοριοποιήστε logged connections προσεκτικά· μπορεί να είναι automated scanning, monitoring, misconfiguration ή authorized test.
- Ποτέ μην κάνετε counterattack ή active retaliation με βάση γεγονότα που παρατηρήθηκαν σε honeypot.

## Βασικά συμπεράσματα

Ένα ασφαλές low-interaction honeypot είναι κυρίως ελεγχόμενος sensor: ακούει σε σκόπιμη θύρα, καταγράφει επαφές και προσφέρει στους defenders χώρο για εξάσκηση σε detection και log analysis.

Η ποιότητα της ανάπτυξης εξαρτάται περισσότερο από isolation, egress control, evidence handling και προσεκτική ερμηνεία παρά από το πόσο πειστικό εμφανίζεται το decoy.

Ξεκινήστε με υψηλή αχρησιμοποίητη θύρα σε private lab, επαληθεύστε μία authorized connection και τεκμηριώστε τόσο το event όσο και τα firewall boundaries που το περιόρισαν.
