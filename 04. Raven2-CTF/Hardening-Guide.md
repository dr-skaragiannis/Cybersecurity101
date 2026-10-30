# Raven 2 — Εγχειρίδιο Θωράκισης (Hardening & Remediation Guide)

Το Raven 2 είναι η «διορθωμένη» έκδοση του Raven 1: ο δημιουργός έκλεισε τα προφανή κενά (αδύναμοι κωδικοί SSH, `sudo python`). Η επίθεση του [README.md](README.md) βρήκε όμως **άλλη** διαδρομή:

```
/vendor/ ανοιχτό → PHPMailer 5.2.x → RCE ως www-data → wp-config.php → MySQL root
       → MySQL τρέχει ως root → UDF → chmod u+s /usr/bin/find → root
```

**Το κεντρικό μάθημα του Raven 2:** η διόρθωση **μόνο** των σημείων που εκμεταλλεύτηκε η προηγούμενη επίθεση δεν αρκεί. Χρειάζεται συστηματική θωράκιση **όλων** των επιπέδων: βιβλιοθήκες κώδικα, web server, βάση δεδομένων, λειτουργικό σύστημα.

Αυτό το εγχειρίδιο εστιάζει σε όσα είναι **νέα** σε σχέση με το Raven 1. Για τα κοινά (firewall, SSH, WordPress, `wp-config.php`) παραπέμπει στο [Hardening-Guide του Raven 1](../02.%20Raven-CTF/Hardening-Guide.md).

> ⚠️ **Πριν ξεκινήσετε:** πάρτε **snapshot** της VM. Εκτελέστε πρώτα την επίθεση, ώστε να δείτε τα ίχνη που αφήνει (§3), και μετά εφαρμόστε τη θωράκιση.

> ℹ️ Οι εντολές εκτελούνται **στο Raven 2 ως `root`**. Οι IP διευθύνσεις είναι ενδεικτικές.

---

## Περιεχόμενα

1. [Χάρτης: Επίθεση → Αιτία → Διόρθωση](#1-χάρτης-επίθεση--αιτία--διόρθωση)
2. [Απογραφή του συστήματος](#2-απογραφή-του-συστήματος)
3. [Καθαρισμός μετά την παραβίαση (Incident Cleanup)](#3-καθαρισμός-μετά-την-παραβίαση)
4. [Κατανόηση του PHPMailer RCE (CVE-2016-10033)](#4-κατανόηση-του-phpmailer-rce)
5. [Αναβάθμιση του PHPMailer και ασφαλής κώδικας φόρμας](#5-αναβάθμιση-του-phpmailer-και-ασφαλής-κώδικας)
6. [Web root: δικαιώματα που ακυρώνουν το exploit](#6-web-root-δικαιώματα-που-ακυρώνουν-το-exploit)
7. [Apache: `/vendor/` και directory listing](#7-apache-vendor-και-directory-listing)
8. [MySQL: τρέχει ως root (η ρίζα της κλιμάκωσης)](#8-mysql-τρέχει-ως-root)
9. [MySQL: UDF, `FILE`, `secure_file_priv`, `plugin_dir`](#9-mysql-udf-file-secure_file_priv-plugin_dir)
10. [MySQL: αναβάθμιση και CVE-2016-6662](#10-mysql-αναβάθμιση-και-cve-2016-6662)
11. [Λειτουργικό: εργαλεία, `/tmp`, εξερχόμενες συνδέσεις, SUID](#11-λειτουργικό-εργαλεία-tmp-εξερχόμενες-συνδέσεις-suid)
12. [Επαλήθευση](#12-επαλήθευση)
13. [Λίστα ελέγχου & ερωτήσεις αυτοελέγχου](#13-λίστα-ελέγχου--ερωτήσεις-αυτοελέγχου)

---

## 1. Χάρτης: Επίθεση → Αιτία → Διόρθωση

| Βήμα επίθεσης (README) | Τι εκμεταλλεύτηκε ο επιτιθέμενος | Ρίζα του προβλήματος | Διόρθωση |
| :--- | :--- | :--- | :--- |
| 3. `dirb` → `/vendor/` | Λίστα αρχείων, `PATH` (Flag 1), `VERSION` | Directory listing, βιβλιοθήκες μέσα στο web root | §7 |
| 4. Exploit-DB 40974 | Εκτέλεση εντολών μέσω της φόρμας `contact.php` | PHPMailer < 5.2.18 | §4, §5 |
| 4. `backdoor.php` γράφτηκε στο web root | Μόνιμο web shell | Ο `www-data` μπορεί να γράψει στο `/var/www/html` | §6 |
| 4. `nc -lvp 4444` | Reverse shell προς τον επιτιθέμενο | Καμία πολιτική εξερχόμενων συνδέσεων | §11.3 |
| 5.1 `wp-config.php` | Κωδικός **root** της MySQL | Η εφαρμογή χρησιμοποιεί τον root της βάσης | §9.4, [Raven 1 §8](../02.%20Raven-CTF/Hardening-Guide.md#8-θωράκιση-mysql) |
| 5.1 `wget LinEnum.sh` | Λήψη εργαλείων από το Kali | Ελεύθερη έξοδος, εγγράψιμο/εκτελέσιμο `/tmp` | §11 |
| 5.2 `gcc ... 1518.so` | Μεταγλώττιση κακόβουλης βιβλιοθήκης | (στο Kali ή στον στόχο) μεταγλωττιστής διαθέσιμος | §11.1 |
| 5.2 `INTO DUMPFILE '/usr/lib/mysql/plugin/'` | Εγγραφή αρχείου στον φάκελο plugins | Δικαίωμα `FILE`, κενό `secure_file_priv`, εγγράψιμο `plugin_dir` | §9 |
| 5.2 `do_system('chmod u+s ...')` | Εντολή συστήματος **ως root** | Η υπηρεσία `mysqld` τρέχει ως `root` | §8 |
| 6. `find raj -exec /bin/sh \;` | Root shell | SUID στο `find` (το έβαλε ο επιτιθέμενος) | §3, §11.4 |

---

## 2. Απογραφή του συστήματος

```bash
cat /etc/os-release                         # Debian 8 Jessie
ss -tulpn                                   # Θύρες 22, 80, 111 και 3306 (τοπικά;)

# Ποιος χρήστης τρέχει κάθε υπηρεσία;
ps -eo user,pid,cmd | grep -E 'mysqld|apache2' | grep -v grep
```

Η τελευταία εντολή είναι η πιο σημαντική του Raven 2. Αν βλέπετε:

```text
root      612  /usr/sbin/mysqld --basedir=/usr --datadir=/var/lib/mysql --user=root ...
```

τότε **κάθε** ενέργεια της βάσης δεδομένων γίνεται με δικαιώματα root. Αυτό είναι το κρίσιμο εύρημα.

```bash
# Ποια έκδοση PHPMailer χρησιμοποιεί το site;
cat /var/www/html/vendor/VERSION
grep -rn "Version" /var/www/html/vendor/class.phpmailer.php | head -3

# Ποια έκδοση MySQL;
mysql --version
```

---

## 3. Καθαρισμός μετά την παραβίαση

Αν το σύστημα έχει ήδη δεχθεί την επίθεση (όπως στο εργαστήριο), η θωράκιση **δεν αρκεί**. Ο επιτιθέμενος έχει αφήσει **τρεις μηχανισμούς επιστροφής** (persistence) που θα λειτουργούν και μετά την αναβάθμιση:

| Ίχνος | Πού | Γιατί είναι επικίνδυνο |
| :--- | :--- | :--- |
| `backdoor.php` (ή `rev.php`) | `/var/www/html/` | Web shell: εκτέλεση εντολών μέσω browser |
| `1518.so` + συνάρτηση `do_system` | `/usr/lib/mysql/plugin/`, πίνακας `mysql.func` | Εκτέλεση εντολών μέσω SQL |
| SUID στο `/usr/bin/find` | Σύστημα αρχείων | Άμεσο root για κάθε τοπικό χρήστη |

### 3.1 Εντοπισμός

```bash
# Αρχεία PHP στο web root που άλλαξαν τις τελευταίες 30 μέρες
find /var/www/html -name '*.php' -mtime -30 -ls

# Αρχεία που ανήκουν στον www-data (αυτά που δημιούργησε ο web server)
find /var/www/html -user www-data -type f -ls

# Περιεχόμενο του backdoor
head -c 400 /var/www/html/backdoor.php
```

```sql
-- Μέσα στη MySQL (mysql -u root -p)
SELECT * FROM mysql.func;      -- Οι User Defined Functions που έχουν καταχωριστεί
SHOW TABLES FROM mysql LIKE 'foo';
```

```bash
ls -la /usr/lib/mysql/plugin/
ls -l /usr/bin/find
ls -la /tmp
```

### 3.2 Αφαίρεση

```bash
# Κρατάμε αντίγραφα για ανάλυση (forensics) πριν διαγράψουμε
mkdir -p /root/evidence && chmod 700 /root/evidence
cp -a /var/www/html/backdoor.php /usr/lib/mysql/plugin/1518.so /tmp/LinEnum.sh /root/evidence/ 2>/dev/null

rm -f /var/www/html/backdoor.php /var/www/html/rev.php
rm -f /tmp/1518.so /tmp/LinEnum.sh /tmp/raj
```

```sql
DROP FUNCTION IF EXISTS do_system;
DROP TABLE IF EXISTS mysql.foo;
```

```bash
rm -f /usr/lib/mysql/plugin/1518.so
chmod u-s /usr/bin/find
```

> 💡 **Γιατί κρατάμε αντίγραφα;** Σε πραγματικό περιστατικό, τα αρχεία του επιτιθέμενου δείχνουν τι έκανε, από ποια IP συνδέθηκε (μέσα στο reverse shell υπάρχει η IP του) και αν πήγε και σε άλλα συστήματα.

> ⚠️ Σε πραγματικό περιβάλλον, μετά από παραβίαση με πρόσβαση root, η μόνη **βέβαιη** επαναφορά είναι η **επανεγκατάσταση** από καθαρή πηγή. Ένας επιτιθέμενος με root μπορεί να έχει αφήσει κρυμμένους μηχανισμούς που δεν εντοπίζονται εύκολα (π.χ. τροποποιημένα εκτελέσιμα, cron jobs, κλειδιά SSH).

Επιπλέον έλεγχοι για persistence:

```bash
cat /root/.ssh/authorized_keys /home/*/.ssh/authorized_keys 2>/dev/null
ls -la /etc/cron.* /var/spool/cron/crontabs/
awk -F: '$3 == 0 {print $1}' /etc/passwd     # Χρήστες με UID 0 (πρέπει να είναι μόνο ο root)
```

---

## 4. Κατανόηση του PHPMailer RCE

### 4.1 Πώς στέλνει email η PHP

Η συνάρτηση `mail()` της PHP **δεν** μιλάει απευθείας SMTP. Καλεί το εξωτερικό πρόγραμμα **`sendmail`** (ή το συμβατό του Exim/Postfix) μέσω του λειτουργικού. Δέχεται μια **5η παράμετρο** με επιπλέον ορίσματα για το `sendmail`:

```php
mail($to, $subject, $body, $headers, "-f sender@example.com");
//                                     ↑ ορίσματα που περνούν στη γραμμή εντολών του sendmail
```

Το `-f` ορίζει τη διεύθυνση αποστολέα (envelope sender).

### 4.2 Το λάθος

Το PHPMailer < 5.2.18 έπαιρνε τη διεύθυνση αποστολέα **από τη φόρμα επικοινωνίας** (ό,τι έγραψε ο επισκέπτης στο πεδίο «Email») και την έβαζε στο `-f`. Ο έλεγχος της διεύθυνσης ήταν ελλιπής: μια διεύθυνση όπως η παρακάτω περνούσε ως «έγκυρη»:

```text
"attacker\" -oQ/tmp/ -X/var/www/html/backdoor.php  some"@email.com
```

Τα εισαγωγικά και το κενό «σπάνε» το όρισμα και το `sendmail` λαμβάνει **επιπλέον παραμέτρους**:

* **`-X /var/www/html/backdoor.php`**: «γράψε όλη την κίνηση SMTP σε αυτό το αρχείο καταγραφής».

Το σώμα του μηνύματος (που επίσης ελέγχει ο επιτιθέμενος) περιέχει κώδικα PHP, π.χ. `<?php system($_GET['cmd']); ?>`. Το `sendmail` τον γράφει στο «αρχείο καταγραφής», που βρίσκεται μέσα στο web root και έχει κατάληξη `.php`. Ο Apache το εκτελεί όταν ο επιτιθέμενος το ανοίξει στον browser.

### 4.3 Τρεις προϋποθέσεις, τρεις ευκαιρίες άμυνας

Για να πετύχει η επίθεση χρειάζονται **ταυτόχρονα**:

| Προϋπόθεση | Πώς την ακυρώνουμε |
| :--- | :--- |
| 1. Ευάλωτο PHPMailer που περνά είσοδο χρήστη στο `-f` | Αναβάθμιση + σταθερός αποστολέας (§5) |
| 2. Ο `www-data` μπορεί να **γράψει** στο web root | Δικαιώματα αρχείων (§6) |
| 3. Ο Apache **εκτελεί** οποιοδήποτε `.php` αρχείο εκεί | Περιορισμός εκτέλεσης PHP (§6.3) |

Αρκεί **μία** από αυτές να λείπει για να αποτύχει η επίθεση. Γι' αυτό τις διορθώνουμε **όλες**.

---

## 5. Αναβάθμιση του PHPMailer και ασφαλής κώδικας

### 5.1 Ποια έκδοση

| Έκδοση | Κατάσταση |
| :--- | :--- |
| < 5.2.18 | Ευάλωτη στο CVE-2016-10033 |
| 5.2.18 – 5.2.19 | Η διόρθωση παρακάμπτεται (CVE-2016-10045) |
| 5.2.20 – 5.2.27 | Διορθωμένη, αλλά ο κλάδος 5.2 δεν υποστηρίζεται πλέον |
| **6.x (τελευταία)** | **Συνιστώμενη**. Απαιτεί PHP ≥ 5.5 |

### 5.2 Αναβάθμιση με Composer (συνιστώμενος τρόπος)

Το **Composer** είναι ο διαχειριστής πακέτων της PHP. Γνωρίζει ποιες βιβλιοθήκες χρησιμοποιεί το έργο και σε ποια έκδοση, οπότε οι αναβαθμίσεις γίνονται με μία εντολή.

```bash
apt install composer        # ή εγκατάσταση από getcomposer.org

# Δομή: οι βιβλιοθήκες ΕΞΩ από το web root
mkdir -p /var/www/app && cd /var/www/app
composer require phpmailer/phpmailer:^6.9

# Έλεγχος γνωστών ευπαθειών σε όλες τις βιβλιοθήκες του έργου
composer audit
```

Το `composer audit` συγκρίνει τις εγκατεστημένες εκδόσεις με τη βάση ευπαθειών του PHP Security Advisories. Στο Raven 2 θα είχε εμφανίσει αμέσως το CVE-2016-10033.

Αφαιρούμε τον παλιό κατάλογο από το web root:

```bash
mv /var/www/html/vendor /root/evidence/vendor-old
```

### 5.3 Ασφαλής κώδικας στη φόρμα επικοινωνίας

Η αναβάθμιση διορθώνει τη βιβλιοθήκη, αλλά ο κώδικας του `contact.php` πρέπει επίσης να ακολουθεί σωστές πρακτικές. Ο **αποστολέας** ενός email από φόρμα επικοινωνίας είναι **πάντα** ο server. Η διεύθυνση του επισκέπτη μπαίνει μόνο στο **Reply-To**:

```php
<?php
require '/var/www/app/vendor/autoload.php';    // Απόλυτη διαδρομή, έξω από το web root

use PHPMailer\PHPMailer\PHPMailer;

$email = filter_input(INPUT_POST, 'email', FILTER_VALIDATE_EMAIL);
if ($email === false || $email === null) {
    http_response_code(400);
    exit('Μη έγκυρη διεύθυνση email.');
}

$mail = new PHPMailer(true);
$mail->setFrom('no-reply@raven.local', 'Raven Security');   // ΣΤΑΘΕΡΟΣ αποστολέας
$mail->addReplyTo($email);                                   // Η διεύθυνση του χρήστη ΜΟΝΟ εδώ
$mail->addAddress('contact@raven.local');
$mail->Subject = 'Νέο μήνυμα από τη φόρμα';
$mail->Body    = strip_tags($_POST['message'] ?? '');
$mail->send();
```

* `FILTER_VALIDATE_EMAIL`: απορρίπτει διευθύνσεις με εισαγωγικά, κενά και άλλους ειδικούς χαρακτήρες.
* `setFrom()` με **σταθερή** τιμή: τίποτα από τον χρήστη δεν καταλήγει στη γραμμή εντολών του `sendmail`.

### 5.4 Αποστολή μέσω SMTP αντί για `sendmail`

Ακόμα ασφαλέστερο είναι να μην καλείται **καθόλου** εξωτερικό πρόγραμμα. Το PHPMailer μπορεί να μιλήσει απευθείας SMTP:

```php
$mail->isSMTP();
$mail->Host = '127.0.0.1';      // Τοπικός mail server ή εξωτερικός relay
$mail->Port = 25;
```

Χωρίς κλήση του `sendmail`, η κατηγορία επιθέσεων «έγχυση παραμέτρων στη γραμμή εντολών» εξαφανίζεται.

---

## 6. Web root: δικαιώματα που ακυρώνουν το exploit

### 6.1 Το πρόβλημα

```bash
ls -ld /var/www/html
# drwxrwxrwx  ή  drwxr-xr-x www-data www-data
```

Αν ο `www-data` είναι **ιδιοκτήτης** του `/var/www/html` ή ο φάκελος είναι `777`, τότε το `sendmail` (που τρέχει ως `www-data` όταν το καλεί η PHP) μπορεί να δημιουργήσει το `backdoor.php` εκεί.

### 6.2 Διόρθωση

```bash
cd /var/www/html
chown -R root:www-data .
find . -type d -exec chmod 750 {} \;
find . -type f -exec chmod 640 {} \;

# Εξαίρεση μόνο για τα uploads του WordPress
chown -R www-data:www-data wordpress/wp-content/uploads
```

Επαλήθευση:

```bash
su -s /bin/sh www-data -c 'touch /var/www/html/test.php'
# → touch: cannot touch '/var/www/html/test.php': Permission denied
```

Με αυτή τη ρύθμιση, ακόμη και το **ευάλωτο** PHPMailer θα αποτύγχανε: το `sendmail -X /var/www/html/backdoor.php` δεν θα μπορούσε να δημιουργήσει το αρχείο.

### 6.3 Απαγόρευση εκτέλεσης PHP σε εγγράψιμους φακέλους

Ο επιτιθέμενος θα μπορούσε να στοχεύσει τον **μόνο** εγγράψιμο φάκελο, τα uploads. Εκεί απαγορεύουμε την εκτέλεση PHP:

```apache
# /etc/apache2/conf-available/no-php-uploads.conf
<Directory /var/www/html/wordpress/wp-content/uploads>
    <FilesMatch "\.(php|phtml|php\d|phar)$">
        Require all denied
    </FilesMatch>
    php_admin_flag engine off
</Directory>
```

```bash
a2enconf no-php-uploads
apache2ctl configtest && systemctl reload apache2
```

* `php_admin_flag engine off`: απενεργοποιεί τη μηχανή PHP για τον φάκελο. Το `php_admin_*` **δεν** μπορεί να παρακαμφθεί από αρχείο `.htaccess`, σε αντίθεση με το `php_flag`.
* Η ρύθμιση γίνεται στη ρύθμιση του Apache, όχι σε `.htaccess`, ώστε να μην μπορεί να τη σβήσει κάποιος που αποκτά εγγραφή στον φάκελο.

---

## 7. Apache: `/vendor/` και directory listing

Το Flag 1 βρέθηκε στο αρχείο `/vendor/PATH` και η έκδοση PHPMailer στο `/vendor/VERSION`. Δύο ξεχωριστά προβλήματα:

1. **Directory listing**: ο Apache έδειχνε λίστα αρχείων.
2. **Βιβλιοθήκες στο web root**: ο φάκελος `vendor/` ήταν προσβάσιμος.

Το πρώτο διορθώνεται όπως στο [Raven 1 §5.2](../02.%20Raven-CTF/Hardening-Guide.md#52-directory-listing-dirbuster) (`Options -Indexes`). Το δεύτερο το λύσαμε στην §5.2 μετακινώντας τις βιβλιοθήκες στο `/var/www/app/`. Ως **δεύτερη γραμμή άμυνας**, απαγορεύουμε την πρόσβαση σε τέτοιους φακέλους γενικά:

```apache
# /etc/apache2/conf-available/deny-dev-files.conf
<DirectoryMatch "/(vendor|\.git|\.svn|node_modules)(/|$)">
    Require all denied
</DirectoryMatch>
<FilesMatch "^(composer\.(json|lock)|PATH|VERSION|.*\.(bak|old|sql|log|md))$">
    Require all denied
</FilesMatch>
```

```bash
a2enconf deny-dev-files
apache2ctl configtest && systemctl reload apache2
```

Εφαρμόστε επίσης τις ρυθμίσεις `ServerTokens Prod`, `ServerSignature Off` και `expose_php = Off` του [Raven 1 §5.4](../02.%20Raven-CTF/Hardening-Guide.md#54-απόκρυψη-εκδόσεων-banners).

---

## 8. MySQL: τρέχει ως root

### 8.1 Γιατί αυτό είναι το πιο σοβαρό λάθος του Raven 2

Ένας web server κινδυνεύει πάντα να παραβιαστεί, γι' αυτό τρέχει ως **`www-data`**, έναν χρήστη χωρίς προνόμια. Αν τον παραβιάσεις, έχεις τα δικαιώματα του `www-data`, όχι του root. Αυτό είναι ο **διαχωρισμός προνομίων** (privilege separation).

Η MySQL στο Raven 2 έτρεχε ως **root**. Η συνάρτηση `do_system()` εκτελείται **μέσα** στη διεργασία `mysqld`, άρα με τα δικαιώματά της. Ο επιτιθέμενος, μέσω SQL, έγινε root.

Αν η MySQL έτρεχε ως `mysql` (κανονική ρύθμιση), το `do_system('chmod u+s /usr/bin/find')` θα αποτύγχανε με `Operation not permitted`, γιατί μόνο ο ιδιοκτήτης του αρχείου (root) μπορεί να αλλάξει τα δικαιώματά του.

### 8.2 Πού ορίζεται ο χρήστης

```bash
grep -rn "^user" /etc/mysql/
```

Αναζητούμε `user = root` σε κάποιο αρχείο. Η διόρθωση στο `/etc/mysql/my.cnf`:

```ini
[mysqld]
user = mysql
```

### 8.3 Διόρθωση ιδιοκτησίας αρχείων

Αφού η MySQL έτρεχε ως root, τα αρχεία δεδομένων της μπορεί να έχουν δημιουργηθεί ως root. Αν τα αφήσουμε έτσι, η υπηρεσία δεν θα ξεκινήσει με τον νέο χρήστη.

```bash
systemctl stop mysql

chown -R mysql:mysql /var/lib/mysql          # Δεδομένα βάσης
chown -R mysql:adm   /var/log/mysql 2>/dev/null
chown mysql:mysql    /var/run/mysqld

# Τα αρχεία ρυθμίσεων ανήκουν στον root και ΔΕΝ είναι εγγράψιμα από τον mysql (βλ. §10)
chown -R root:root /etc/mysql
find /etc/mysql -type f -exec chmod 644 {} \;

systemctl start mysql
ps -eo user,cmd | grep [m]ysqld
# → mysql  /usr/sbin/mysqld ... --user=mysql
```

> 💡 Το `grep [m]ysqld` είναι ένα κόλπο: η κανονική έκφραση `[m]ysqld` ταιριάζει με το «mysqld», αλλά όχι με τη γραμμή της ίδιας της εντολής `grep` (που περιέχει «[m]ysqld»). Έτσι δεν χρειάζεται `grep -v grep`.

---

## 9. MySQL: UDF, `FILE`, `secure_file_priv`, `plugin_dir`

### 9.1 Τι είναι οι User Defined Functions

Η MySQL επιτρέπει την επέκτασή της με **συναρτήσεις γραμμένες σε C**, που φορτώνονται από αρχεία `.so` (shared objects) του φακέλου `plugin_dir`. Αυτή είναι νόμιμη δυνατότητα. Η επίθεση συνδύασε **τέσσερα** ελλείμματα:

| # | Βήμα της επίθεσης | Τι χρειάστηκε | Πώς το ελέγχουμε |
| :--- | :--- | :--- | :--- |
| 1 | `load_file('/tmp/1518.so')` | Δικαίωμα `FILE`, `secure_file_priv` κενό | `SELECT @@secure_file_priv;` |
| 2 | `INTO DUMPFILE '/usr/lib/mysql/plugin/1518.so'` | Ο `mysqld` μπορεί να γράψει στο `plugin_dir` | `ls -ld $(mysql -NBe 'select @@plugin_dir')` |
| 3 | `CREATE FUNCTION do_system ...` | Δικαίωμα `INSERT` στη βάση `mysql` | `SHOW GRANTS;` |
| 4 | `do_system('chmod ...')` εκτελέστηκε ως root | Η υπηρεσία τρέχει ως root | §8 |

### 9.2 Έλεγχος της τρέχουσας κατάστασης

```sql
SELECT @@version, @@plugin_dir, @@secure_file_priv;
SELECT user, host, File_priv, Super_priv FROM mysql.user;
```

| Τιμή του `secure_file_priv` | Σημασία |
| :--- | :--- |
| κενό (`''`) | **Επικίνδυνο**: `LOAD_FILE`/`INTO OUTFILE` λειτουργούν σε **οποιονδήποτε** φάκελο |
| `/var/lib/mysql-files/` | Λειτουργούν **μόνο** μέσα σε αυτόν τον φάκελο |
| `NULL` | Απενεργοποιημένα εντελώς |

### 9.3 Διόρθωση

**α) Περιορισμός λειτουργιών αρχείων** — `/etc/mysql/my.cnf`, ενότητα `[mysqld]`:

```ini
[mysqld]
user             = mysql
bind-address     = 127.0.0.1
secure-file-priv = /var/lib/mysql-files
local-infile     = 0
skip-symbolic-links
```

```bash
mkdir -p /var/lib/mysql-files
chown mysql:mysql /var/lib/mysql-files
chmod 750 /var/lib/mysql-files
```

**β) Ο φάκελος plugins δεν είναι εγγράψιμος από τη MySQL:**

```bash
PLUGIN_DIR=$(mysql -u root -p -NBe 'SELECT @@plugin_dir')
chown -R root:root "$PLUGIN_DIR"
chmod 755 "$PLUGIN_DIR"
find "$PLUGIN_DIR" -type f -exec chmod 644 {} \;
```

Η MySQL χρειάζεται να **διαβάζει** τα plugins, όχι να γράφει νέα. Νέα plugins εγκαθιστά μόνο ο διαχειριστής του λειτουργικού, ως root.

```bash
systemctl restart mysql
```

### 9.4 Ξεχωριστός χρήστης για το WordPress, χωρίς `FILE`

Όπως στο [Raven 1 §8.3](../02.%20Raven-CTF/Hardening-Guide.md#83-ξεχωριστός-χρήστης-για-την-εφαρμογή-least-privilege):

```sql
CREATE USER 'wp_app'@'localhost' IDENTIFIED BY 'Ένας-Μακρύς-Τυχαίος-Κωδικός-2026!';
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, ALTER, INDEX, DROP
  ON wordpress.* TO 'wp_app'@'localhost';

-- Ο root της βάσης παίρνει ΝΕΟ κωδικό (ο R@v3nSecurity είναι εκτεθειμένος)
SET PASSWORD FOR 'root'@'localhost' = PASSWORD('Άλλος-Μοναδικός-Κωδικός-Root!');

-- Αφαίρεση του FILE από κάθε χρήστη εκτός του root
SELECT user, host FROM mysql.user WHERE File_priv = 'Y';
-- REVOKE FILE ON *.* FROM '<χρήστης>'@'<host>';   για όσους εμφανιστούν
FLUSH PRIVILEGES;
```

Ο χρήστης `wp_app`:

* **Δεν** έχει `FILE`, άρα δεν μπορεί να κάνει `LOAD_FILE` ή `INTO DUMPFILE` (βήματα 1-2).
* **Δεν** έχει δικαιώματα στη βάση `mysql`, άρα δεν μπορεί να κάνει `CREATE FUNCTION` (βήμα 3).

Ακόμη κι αν ο επιτιθέμενος διαβάσει το `wp-config.php`, **κανένα** από τα βήματα της UDF δεν είναι δυνατό.

---

## 10. MySQL: αναβάθμιση και CVE-2016-6662

### 10.1 Τι είναι το CVE-2016-6662

Το README αναφέρει το CVE-2016-6662. Αυτή η ευπάθεια σχετίζεται άμεσα με τις ρυθμίσεις που κάναμε, γι' αυτό αξίζει να εξηγηθεί.

Το script εκκίνησης **`mysqld_safe`** τρέχει ως **root** και διαβάζει το `my.cnf`. Το `my.cnf` μπορεί να περιέχει την επιλογή `malloc-lib=`, που ορίζει μια βιβλιοθήκη η οποία φορτώνεται κατά την εκκίνηση. Αν ένας επιτιθέμενος καταφέρει να **γράψει** στο `my.cnf` (π.χ. μέσω του `general_log_file` της MySQL, που επιτρέπει στη βάση να γράφει log σε όποιο αρχείο ορίσεις), μπορεί να προσθέσει `malloc-lib=/tmp/κακόβουλο.so`. Στην επόμενη επανεκκίνηση η βιβλιοθήκη φορτώνεται **ως root**, ακόμη κι αν ο `mysqld` τρέχει ως `mysql`.

### 10.2 Διόρθωση

1. **Αναβάθμιση** σε MySQL ≥ 5.5.52 / 5.6.33 / 5.7.15 (ή αντίστοιχη MariaDB). Οι εκδόσεις αυτές απορρίπτουν το `malloc-lib` από μη ασφαλείς διαδρομές.

   ```bash
   apt-get update && apt-get install --only-upgrade mysql-server
   mysql --version
   ```

2. **Τα αρχεία ρυθμίσεων ανήκουν στον root** και δεν είναι εγγράψιμα από τον `mysql` (το κάναμε στην §8.3). Επιβεβαίωση:

   ```bash
   ls -l /etc/mysql/my.cnf /etc/mysql/conf.d/
   ls -la /var/lib/mysql/my.cnf 2>/dev/null     # Δεν πρέπει να υπάρχει
   ```

   Η MySQL διαβάζει `my.cnf` και από τον φάκελο δεδομένων (`/var/lib/mysql/`), ο οποίος **ανήκει** στον `mysql`. Αν υπάρχει εκεί `my.cnf`, διαγράφεται και δημιουργούμε ένα κενό, κλειδωμένο αρχείο ώστε να μην μπορεί να δημιουργηθεί ξανά:

   ```bash
   touch /var/lib/mysql/my.cnf
   chown root:root /var/lib/mysql/my.cnf
   chmod 644 /var/lib/mysql/my.cnf
   ```

3. **Μόνο ο root της βάσης** μπορεί να αλλάξει το `general_log_file` (απαιτεί `SUPER`). Αυτό ισχύει ήδη αν ο χρήστης της εφαρμογής δεν έχει `SUPER` (§9.4).

### 10.3 Μακροπρόθεσμα

Το Debian 8 έχει MySQL 5.5, που δεν υποστηρίζεται πλέον. Σε υποστηριζόμενο Debian 12/13 η προεπιλογή είναι **MariaDB 10.11 / 11.x**, όπου το `secure_file_priv` και ο χρήστης `mysql` είναι ρυθμισμένα σωστά εξαρχής.

---

## 11. Λειτουργικό: εργαλεία, `/tmp`, εξερχόμενες συνδέσεις, SUID

### 11.1 Αφαίρεση εργαλείων ανάπτυξης από τον server παραγωγής

Ένας web server **δεν χρειάζεται** μεταγλωττιστή. Η παρουσία του `gcc` επιτρέπει στον επιτιθέμενο να μεταγλωττίσει exploits **πάνω στον στόχο**, για τη σωστή αρχιτεκτονική και τις σωστές βιβλιοθήκες.

```bash
dpkg -l | grep -E '^ii\s+(gcc|g\+\+|make|cc|build-essential)'
apt-get purge gcc g++ make build-essential
apt-get autoremove
```

Αν η αφαίρεση δεν είναι δυνατή (π.χ. τα χρειάζεται κάποιο πακέτο), περιορίζουμε την εκτέλεσή τους μόνο στον root:

```bash
chmod 700 /usr/bin/gcc* /usr/bin/cc /usr/bin/make 2>/dev/null
```

### 11.2 `/tmp` χωρίς εκτέλεση και SUID

Η επίθεση χρησιμοποίησε το `/tmp` για το `LinEnum.sh` και το `1518.so`. Εφαρμόζουμε τις επιλογές `nosuid,nodev,noexec` όπως εξηγούνται στο [DC-1 §10](../03.%20DC-CTF/Hardening-Guide.md#10-επιλογές-προσάρτησης):

```text
# /etc/fstab
tmpfs  /tmp      tmpfs  defaults,nosuid,nodev,noexec,size=512M  0 0
tmpfs  /dev/shm  tmpfs  defaults,nosuid,nodev,noexec            0 0
```

```bash
mount -o remount /tmp ; mount -o remount /dev/shm
```

Επιπλέον, με `secure-file-priv = /var/lib/mysql-files` (§9.3), η MySQL **δεν** μπορεί πλέον να διαβάσει το `/tmp/1518.so`.

### 11.3 Περιορισμός εξερχόμενων συνδέσεων

Ο web server ξεκίνησε **δύο** εξερχόμενες συνδέσεις: το reverse shell (`nc` στη θύρα 4444) και τη λήψη του `LinEnum.sh` (`wget`). Το φιλτράρισμα εξόδου ανά χρήστη εξηγείται αναλυτικά στο [DC-1 §8](../03.%20DC-CTF/Hardening-Guide.md#8-περιορισμός-εξερχόμενων-συνδέσεων). Στο Raven 2 προσθέτουμε έναν ειδικό κανόνα: η φόρμα επικοινωνίας χρειάζεται να στέλνει email, οπότε επιτρέπουμε **μόνο** SMTP προς τον mail server:

```bash
iptables -A OUTPUT -o lo -j ACCEPT
iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
# Μόνο SMTP προς τον εσωτερικό mail relay (ενδεικτική IP)
iptables -A OUTPUT -m owner --uid-owner www-data -p tcp -d 192.168.56.10 --dport 25 -j ACCEPT
iptables -A OUTPUT -m owner --uid-owner www-data -m state --state NEW -j LOG --log-prefix "WWW-EGRESS: "
iptables -A OUTPUT -m owner --uid-owner www-data -m state --state NEW -j REJECT

# Ο mysql δεν έχει ΚΑΝΕΝΑ λόγο να ανοίγει εξερχόμενες συνδέσεις
iptables -A OUTPUT -m owner --uid-owner mysql -m state --state NEW -j REJECT

apt-get install iptables-persistent && iptables-save > /etc/iptables/rules.v4
```

### 11.4 SUID: γραμμή βάσης και καθημερινός έλεγχος

Στο Raven 2 το SUID στο `find` **το πρόσθεσε ο επιτιθέμενος**. Ένας καθημερινός έλεγχος σε σχέση με γραμμή βάσης θα το εντόπιζε. Δείτε το script του [DC-1 §9.5](../03.%20DC-CTF/Hardening-Guide.md#95-περιοδικός-έλεγχος-με-γραμμή-βάσης-baseline).

### 11.5 Τα υπόλοιπα (κοινά με το Raven 1)

Εφαρμόστε επίσης από το [Hardening-Guide του Raven 1](../02.%20Raven-CTF/Hardening-Guide.md):

* §3 — ενημέρωση πακέτων / μετάβαση σε υποστηριζόμενο Debian
* §4 — απενεργοποίηση `rpcbind`, firewall `default deny`
* §6 — WordPress: αναβάθμιση, απαρίθμηση χρηστών, `wp-config.php` σε `640`, `DISALLOW_FILE_EDIT`
* §7 — SSH με κλειδιά, `MaxAuthTries`, PAM

---

## 12. Επαλήθευση

Επαναφέρετε το **ευάλωτο snapshot** σε μια δεύτερη VM για σύγκριση, ή επαναλάβετε την επίθεση στη θωρακισμένη μηχανή.

Από το **Kali**:

```bash
# 1. Το /vendor/ δεν είναι προσβάσιμο
curl -I http://<RAVEN2_IP>/vendor/              # → 403 ή 404
curl -I http://<RAVEN2_IP>/vendor/PATH          # → 403 ή 404

# 2. Το exploit 40974 αποτυγχάνει
python 40974.py
curl -I http://<RAVEN2_IP>/backdoor.php         # → 404 (το αρχείο δεν δημιουργήθηκε)
```

Στο **Raven 2**:

```bash
# 3. Ο www-data δεν γράφει στο web root
su -s /bin/sh www-data -c 'touch /var/www/html/x.php'          # → Permission denied

# 4. Η MySQL τρέχει ως mysql
ps -eo user,cmd | grep [m]ysqld

# 5. Οι λειτουργίες αρχείων είναι περιορισμένες
mysql -u root -p -e "SELECT @@secure_file_priv;"                 # → /var/lib/mysql-files/
mysql -u root -p -e "SELECT load_file('/etc/passwd');"           # → NULL

# 6. Ο φάκελος plugins δεν είναι εγγράψιμος
mysql -u root -p -e "SELECT 'x' INTO DUMPFILE '/usr/lib/mysql/plugin/test.so';"
# → ERROR 1290: The MySQL server is running with the --secure-file-priv option

# 7. Ο χρήστης της εφαρμογής δεν μπορεί να δημιουργήσει συναρτήσεις
mysql -u wp_app -p -e "CREATE FUNCTION x RETURNS INTEGER SONAME 'x.so';"
# → ERROR 1044: Access denied

# 8. Καμία UDF, κανένα SUID find
mysql -u root -p -e "SELECT * FROM mysql.func;"                  # → κενό
ls -l /usr/bin/find                                              # → -rwxr-xr-x
```

---

## 13. Λίστα ελέγχου & ερωτήσεις αυτοελέγχου

### 13.1 Λίστα ελέγχου

- [ ] Snapshot, αντίγραφα αποδεικτικών στοιχείων στο `/root/evidence`
- [ ] Αφαιρέθηκαν `backdoor.php`, `1518.so`, `do_system`, πίνακας `foo`, SUID στο `find`
- [ ] Έλεγχος για persistence (κλειδιά SSH, cron, χρήστες UID 0)
- [ ] PHPMailer 6.x μέσω Composer, εκτός web root, `composer audit` καθαρό
- [ ] `contact.php`: σταθερός `setFrom`, διεύθυνση χρήστη μόνο στο `Reply-To`, `FILTER_VALIDATE_EMAIL`
- [ ] Web root σε `root:www-data`, `750/640`, ο `www-data` δεν γράφει
- [ ] Καμία εκτέλεση PHP στα uploads (`php_admin_flag engine off`)
- [ ] `Options -Indexes`, μπλοκάρισμα `vendor/`, `.git/`, `PATH`, `VERSION`
- [ ] MySQL: `user = mysql`, `bind-address = 127.0.0.1`, `secure-file-priv`, `local-infile = 0`
- [ ] `plugin_dir` σε `root:root 755`, `/etc/mysql` μη εγγράψιμο από τον `mysql`
- [ ] Χρήστης `wp_app` χωρίς `FILE`/`SUPER`, νέος κωδικός root της βάσης
- [ ] MySQL ≥ 5.5.52 (ή σχέδιο μετάβασης σε MariaDB υποστηριζόμενης διανομής)
- [ ] `gcc`/`make` αφαιρέθηκαν, `/tmp` με `noexec,nosuid,nodev`
- [ ] Egress filtering για `www-data` και `mysql`
- [ ] Γραμμή βάσης SUID και καθημερινός έλεγχος
- [ ] Μέτρα του Raven 1 (firewall, SSH, WordPress)
- [ ] Επαλήθευση όλων των βημάτων της §12

### 13.2 Ερωτήσεις αυτοελέγχου

1. Η `mail()` της PHP καλεί ένα εξωτερικό πρόγραμμα. Εξηγήστε πώς αυτό επέτρεψε στο CVE-2016-10033 να γράψει αρχείο στον δίσκο.
2. Αναφέρετε τρία **ανεξάρτητα** μέτρα της §5 και §6, το καθένα από τα οποία αρκεί μόνο του για να αποτύχει το exploit 40974.
3. Γιατί η διεύθυνση email του επισκέπτη πρέπει να μπαίνει στο `Reply-To` και όχι στο `From`/`Sender`;
4. Η επίθεση UDF χρειάστηκε τέσσερις προϋποθέσεις (§9.1). Ποια από αυτές είναι η **πιο κρίσιμη**, δηλαδή αυτή που, αν λείπει, κάνει τις άλλες τρεις ακίνδυνες για το λειτουργικό σύστημα;
5. Τι ακριβώς ελέγχει η ρύθμιση `secure_file_priv` και ποια η διαφορά ανάμεσα στην κενή τιμή και στην τιμή `NULL`;
6. Γιατί η MySQL μπορεί να κινδυνεύει από το CVE-2016-6662 **ακόμη κι αν** τρέχει ως χρήστης `mysql`; Ποιο μέτρο της §10 το αντιμετωπίζει;
7. Γιατί η απλή αναβάθμιση του PHPMailer **δεν αρκεί** σε ένα σύστημα που έχει ήδη παραβιαστεί;

---

## Πηγές

- [NVD — CVE-2016-10033 (PHPMailer)](https://nvd.nist.gov/vuln/detail/CVE-2016-10033) · [CVE-2016-10045](https://nvd.nist.gov/vuln/detail/CVE-2016-10045)
- [PHPMailer — GitHub & Security notes](https://github.com/PHPMailer/PHPMailer/blob/master/SECURITY.md)
- [NVD — CVE-2016-6662 (MySQL)](https://nvd.nist.gov/vuln/detail/CVE-2016-6662)
- [MySQL — secure_file_priv](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_secure_file_priv)
- [MySQL — Security Guidelines](https://dev.mysql.com/doc/refman/8.0/en/security-guidelines.html)
- [Composer — audit command](https://getcomposer.org/doc/03-cli.md#audit)
- [Apache HTTP Server — Security Tips](https://httpd.apache.org/docs/2.4/misc/security_tips.html)
