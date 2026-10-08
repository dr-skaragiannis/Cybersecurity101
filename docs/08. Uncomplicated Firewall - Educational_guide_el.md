# UFW: Εκπαιδευτικός Οδηγός Ρύθμισης Firewall

> **Χρήση μόνο σε συστήματα που σας ανήκουν ή διαχειρίζεστε με εξουσιοδότηση.** Μια αλλαγή firewall μπορεί να διακόψει SSH, web υπηρεσίες, monitoring και άλλες λειτουργίες. Πριν αλλάξετε απομακρυσμένο server, διατηρήστε ενεργή συνεδρία SSH και διαθέστε console ή out-of-band πρόσβαση για ανάκτηση.

## Τι είναι το UFW

Το UFW, δηλαδή Uncomplicated Firewall, είναι εργαλείο γραμμής εντολών για τη διαχείριση κανόνων firewall σε Linux. Προσφέρει πιο κατανοητές εντολές από την απευθείας επεξεργασία κανόνων χαμηλού επιπέδου. Μπορεί να ελέγχει πρόσβαση με βάση source IP, subnet, service profile, θύρα και πρωτόκολλο.

Το UFW λειτουργεί μαζί με τις υπηρεσίες του host, τα cloud security groups, το routing και τυχόν upstream firewall. Ένας κανόνας που επιτρέπει SSH στο UFW δεν αρκεί αν το cloud firewall μπλοκάρει τη σύνδεση. Επίσης, το firewall δεν αντικαθιστά authentication, ενημερώσεις λογισμικού ή ασφαλή ρύθμιση εφαρμογών.

## Προετοιμασία και ασφάλεια

Χρειάζεστε administrative privileges. Εντοπίστε πρώτα τη θύρα SSH και τις υπηρεσίες που ακούν στο δίκτυο:

```bash
id
sudo ss -tulpn
```

Ενδεικτική έξοδος:

```text
uid=1000(admin) gid=1000(admin) groups=1000(admin)
Netid State  Local Address:Port  Process
LISTEN 0      128    0.0.0.0:22   users:(('sshd',pid=742,fd=3))
LISTEN 0      128    0.0.0.0:80   users:(('apache2',pid=911,fd=4))
```

Η `id` επιβεβαιώνει τον τρέχοντα χρήστη και τις ομάδες του. Η `ss` εμφανίζει listening TCP και UDP sockets. Στο παράδειγμα, το SSH ακούει στη θύρα 22 και το HTTP στη θύρα 80. Το inventory αυτό πρέπει να καθοδηγεί τους κανόνες· δεν πρέπει να επιτρέπετε μηχανικά κάθε υπηρεσία που εντοπίζετε.

## Εγκατάσταση UFW

Σε Debian-based διανομή:

```bash
sudo apt update
sudo apt install ufw
```

Ενδεικτική έξοδος:

```text
Reading package lists... Done
Building dependency tree... Done
The following NEW packages will be installed:
  ufw
Setting up ufw ...
```

Η `apt update` ανανεώνει τα metadata των πακέτων. Η `apt install ufw` εγκαθιστά το εργαλείο και τα υποστηρικτικά αρχεία. Η ακριβής έξοδος διαφέρει ανάλογα με τη διανομή και την έκδοση πακέτου. Εξετάζετε πάντα τυχόν prompts πριν επιβεβαιώσετε αλλαγές.

Ελέγξτε την αρχική κατάσταση:

```bash
sudo ufw status verbose
```

Τυπική έξοδος:

```text
Status: inactive
```

Το `inactive` σημαίνει ότι το UFW είναι εγκατεστημένο αλλά δεν εφαρμόζει ακόμη κανόνες. Αυτό επιτρέπει να δημιουργήσετε πολιτική πρόσβασης πριν ενεργοποιήσετε την επιβολή της.

## Προεπιλεγμένη πολιτική

Μια συνηθισμένη πολιτική server απορρίπτει νέες εισερχόμενες συνδέσεις και επιτρέπει εξερχόμενες:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

Ενδεικτική έξοδος:

```text
Default incoming policy changed to 'deny'
Default outgoing policy changed to 'allow'
```

Η incoming policy ελέγχει νέες συνδέσεις προς τον server. Η outgoing policy ελέγχει συνδέσεις που ξεκινούν από τον server. Οι προεπιλογές δεν αντικαθιστούν τους ειδικούς κανόνες υπηρεσιών και δεν πρέπει να θεωρούνται πλήρης πολιτική ασφάλειας.

## Επιτρέψτε το SSH πριν την ενεργοποίηση

Αν το SSH ακούει στη θύρα 22:

```bash
sudo ufw allow 22/tcp
```

Ενδεικτική έξοδος:

```text
Rule added
Rule added (v6)
```

Ο κανόνας επιτρέπει TCP συνδέσεις στη θύρα 22 για IPv4 και IPv6, εφόσον και τα δύο είναι ενεργά. Αν το SSH χρησιμοποιεί διαφορετική θύρα, χρησιμοποιήστε τη σωστή:

```bash
sudo ufw allow 7822/tcp
```

Μην επιτρέπετε τη θύρα 22 απλώς επειδή είναι η συνηθισμένη. Επιβεβαιώστε τη θύρα με `ss`, τη ρύθμιση του SSH daemon ή την κονσόλα του παρόχου. Κρατήστε ανοιχτή την τρέχουσα συνεδρία SSH έως ότου δοκιμαστεί νέα σύνδεση.

## Profiles εφαρμογών

Δείτε τα διαθέσιμα application profiles:

```bash
sudo ufw app list
```

Παράδειγμα εξόδου:

```text
Available applications:
  Apache
  Apache Full
  OpenSSH
```

Τα profiles προέρχονται από εγκατεστημένα αρχεία εφαρμογών. Ένα profile αντιστοιχίζει φιλικό όνομα σε μία ή περισσότερες θύρες και πρωτόκολλα. Εξετάστε το πριν το χρησιμοποιήσετε:

```bash
sudo ufw app info "Apache Full"
```

Ενδεικτική έξοδος:

```text
Profile: Apache Full
Title: Web Server (HTTP,HTTPS)
Ports: 80/tcp,443/tcp
```

Η έξοδος δείχνει ότι το profile ανοίγει HTTP και HTTPS. Αν απαιτείται μόνο κρυπτογραφημένη web πρόσβαση, προτιμήστε κανόνα που επιτρέπει μόνο την 443.

```bash
sudo ufw allow "Apache Full"
```

Ενδεικτική έξοδος:

```text
Rule added
Rule added (v6)
```

Τα εισαγωγικά χρειάζονται επειδή το όνομα περιέχει κενό. Μην ανοίγετε υπηρεσία που δεν είναι αναγκαία ή σωστά ασφαλισμένη.

## Επιτρέψτε συγκεκριμένη IP

Για γενική πρόσβαση από συγκεκριμένη trusted IP:

```bash
sudo ufw allow from 192.168.1.1
```

Ενδεικτική έξοδος:

```text
Rule added
```

Ο κανόνας είναι ευρύς και μπορεί να επιτρέπει πρόσβαση σε πολλές υπηρεσίες. Ασφαλέστερη, πιο περιορισμένη μορφή:

```bash
sudo ufw allow from 192.168.1.1 to any port 22 proto tcp
```

Αυτό επιτρέπει SSH μόνο από την IP `192.168.1.1`. Το `to any port 22` ορίζει τη θύρα προορισμού και το `proto tcp` περιορίζει το πρωτόκολλο.

Αφαίρεση του ευρέος κανόνα:

```bash
sudo ufw delete allow from 192.168.1.1
```

Τυπική έξοδος:

```text
Rule deleted
```

Η εντολή διαγραφής πρέπει να αντιστοιχεί στο νόημα του αρχικού κανόνα. Αν ο κανόνας είχε θύρα ή πρωτόκολλο, συμπεριλάβετε τα ή χρησιμοποιήστε αριθμημένους κανόνες.

## Επιτρέψτε subnet

```bash
sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp
```

Ενδεικτική έξοδος:

```text
Rule added
```

Το `/24` αντιπροσωπεύει το σχετικό δίκτυο διευθύνσεων `192.168.1.x`. Ο κανόνας επιτρέπει SSH από αυτό το δίκτυο. Όσο μεγαλύτερο είναι το trusted subnet, τόσο περισσότερα συστήματα μπορούν να προσπαθήσουν να συνδεθούν, επομένως χρειάζεται προσεκτικός σχεδιασμός.

## Προβολή και διαγραφή αριθμημένων κανόνων

```bash
sudo ufw status numbered
```

Παράδειγμα εξόδου:

```text
Status: active

     To                         Action      From
[ 1] 22/tcp                     ALLOW IN    192.168.1.1
[ 2] 80/tcp                     ALLOW IN    Anywhere
[ 3] 443/tcp                    ALLOW IN    Anywhere
```

Η αριθμημένη λίστα δείχνει σειρά, προορισμό, ενέργεια, κατεύθυνση και πηγή. Για διαγραφή του δεύτερου κανόνα:

```bash
sudo ufw delete 2
```

Ενδεικτική έξοδος:

```text
Deleting:
 allow 80/tcp
Proceed with operation (y|n)? y
Rule deleted
```

Οι αριθμοί αλλάζουν μετά από διαγραφές. Εκτελείτε ξανά `ufw status numbered` πριν διαγράψετε επόμενο κανόνα.

## Απόρριψη IP ή subnet

```bash
sudo ufw deny from 192.168.1.1
sudo ufw deny from 192.168.1.0/24
```

Κάθε εντολή προσθέτει κανόνα απόρριψης για την αντίστοιχη πηγή. Οι ευρείες αρνήσεις μπορεί να μπλοκάρουν administrators, monitoring ή health checks. Προτιμήστε ακριβείς κανόνες και καταγράψτε τον λόγο ύπαρξης κάθε deny.

Για αφαίρεση:

```bash
sudo ufw delete deny from 192.168.1.1
```

Ενδεικτική έξοδος:

```text
Rule deleted
```

## Ενεργοποίηση και επαλήθευση

Όταν έχουν προστεθεί οι απαιτούμενοι κανόνες:

```bash
sudo ufw enable
```

Τυπική έξοδος:

```text
Command may disrupt existing ssh connections. Proceed with operation (y|n)? y
Firewall is active and enabled on system startup
```

Η προειδοποίηση υπάρχει επειδή λανθασμένος κανόνας μπορεί να εμποδίσει το SSH. Επιβεβαιώστε πρώτα ότι υπάρχει κανόνας για τη σωστή θύρα. Η δεύτερη γραμμή σημαίνει ότι το firewall ενεργοποιήθηκε τώρα και θα ξεκινά κατά το boot.

Ελέγξτε τη ρύθμιση:

```bash
sudo ufw status verbose
sudo ufw show added
```

Ενδεικτική έξοδος:

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

Η verbose κατάσταση εμφανίζει ενεργοποίηση, logging, default policies και κανόνες. Το `show added` δείχνει κανόνες που προστέθηκαν μέσω UFW. Οι IPv6 εγγραφές είναι σημαντικές όταν ο server έχει IPv6, διαφορετικά η αξιολόγηση μόνο IPv4 μπορεί να δώσει ελλιπή εικόνα.

## Logging και αντιμετώπιση προβλημάτων

Ενεργοποιήστε μέτριο logging όταν χρειάζεστε ορατότητα:

```bash
sudo ufw logging medium
sudo ufw status verbose
```

Ενδεικτική έξοδος:

```text
Logging enabled
Status: active
Logging: on (medium)
```

Το επίπεδο logging καθορίζει πόσες πληροφορίες firewall καταγράφονται. Υψηλότερο logging βοηθά στη διάγνωση αλλά αυξάνει τον όγκο logs.

```bash
sudo journalctl -k --since "10 minutes ago" | grep -i ufw
```

Παράδειγμα εξόδου:

```text
kernel: [UFW BLOCK] IN=eth0 OUT= SRC=203.0.113.50 DST=198.51.100.10 PROTO=TCP DPT=23
```

Το `UFW BLOCK` δηλώνει απορριφθέν πακέτο. Το `SRC` είναι η πηγή, το `DST` ο προορισμός, το `PROTO` το πρωτόκολλο και το `DPT` η θύρα προορισμού. Τα logs βοηθούν να ξεχωρίσετε κακόβουλες σαρώσεις από νόμιμη κίνηση που αποκλείστηκε κατά λάθος.

## Reload, disable και reset

```bash
sudo ufw reload
```

Αναμενόμενη έξοδος:

```text
Firewall reloaded
```

Το reload εφαρμόζει τις αλλαγές χωρίς επανεκκίνηση συστήματος.

```bash
sudo ufw disable
```

Ενδεικτική έξοδος:

```text
Firewall stopped and disabled on system startup
```

Το disable σταματά την ενεργή επιβολή και εμποδίζει την αυτόματη εκκίνηση στο boot. Χρησιμοποιήστε το μόνο όταν γνωρίζετε την έκθεση που δημιουργείται.

```bash
sudo ufw reset
```

Τυπική έξοδος:

```text
Resetting all rules to installed defaults. Proceed with operation (y|n)? y
Backing up 'user.rules' to '/etc/ufw/user.rules.20261008_190000'
```

Το reset διαγράφει τους user-defined κανόνες και επαναφέρει τις αρχικές προεπιλογές. Μπορεί να αφαιρέσει και τον κανόνα SSH, επομένως προτιμήστε console ή βεβαιωθείτε ότι μπορείτε να επαναφέρετε άμεσα την πρόσβαση.

## ICMP και ping

Το ping χρησιμοποιεί ICMP, όχι TCP ή UDP. Η πλήρης απαγόρευση ping δεν κάνει έναν server αόρατο και μπορεί να επηρεάσει monitoring, διάγνωση, Path MTU Discovery και IPv6. Συνήθως είναι καλύτερο να επιτρέπονται τα απαραίτητα ICMP types και να εφαρμόζονται authentication, ενημερώσεις, least privilege και monitoring.

Αν απαιτείται αλλαγή ειδικής ICMP συμπεριφοράς, κρατήστε πρώτα backup:

```bash
sudo cp -a /etc/ufw/before.rules /etc/ufw/before.rules.backup
sudoedit /etc/ufw/before.rules
sudo ufw reload
```

Αναμενόμενη έξοδος:

```text
Firewall reloaded
```

Το backup επιτρέπει ανάκτηση από συντακτικό ή λογικό λάθος. Το `sudoedit` επεξεργάζεται το αρχείο με ασφαλέστερο τρόπο. Μετά την αλλαγή δοκιμάστε τόσο την επιτρεπόμενη όσο και την ανεπιθύμητη κίνηση. Οι IPv4 και IPv6 κανόνες πρέπει να εξεταστούν ξεχωριστά.

## Τελική λίστα επαλήθευσης

```bash
sudo ufw status numbered
sudo ss -tulpn
```

Επιβεβαιώστε ότι:

- Η θύρα SSH επιτρέπεται μόνο από τις απαιτούμενες administrative πηγές.
- Οι web θύρες είναι ανοικτές μόνο όταν η υπηρεσία πρέπει να είναι δημόσια.
- Δεν εκτίθενται μη χρησιμοποιούμενες υπηρεσίες.
- Οι IPv4 και IPv6 κανόνες συμφωνούν με τη σχεδίαση.
- Οι default policies είναι καταγεγραμμένες.
- Το logging έχει κατάλληλο επίπεδο.
- Υπάρχει rollback plan ή console access.

## Βασικά μαθήματα

Μια σωστή πολιτική firewall ξεκινά από inventory υπηρεσιών και trusted πηγών, όχι από τυφλή αντιγραφή εντολών. Ορίστε περιοριστική προεπιλογή, επιτρέψτε μόνο την απαραίτητη κίνηση, ελέγξτε τους κανόνες και δοκιμάστε τη συνδεσιμότητα πριν κλείσετε τη διαχειριστική συνεδρία. Η εξήγηση κάθε output στο change record βοηθά τον επόμενο administrator να καταλάβει τι ενεργοποιήθηκε, τι αποκλείστηκε και τι αφαιρέθηκε.
