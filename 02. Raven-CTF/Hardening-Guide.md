# Raven 1 — Εγχειρίδιο Θωράκισης (Hardening & Remediation Guide)

Το [README.md](README.md) έδειξε **πώς** παραβιάστηκε το Raven 1. Αυτό το εγχειρίδιο εξηγεί **γιατί** ήταν δυνατή κάθε παραβίαση και **πώς** τη διορθώνουμε, μόνο με ρυθμίσεις, αναβαθμίσεις και αλλαγές δικαιωμάτων στο ίδιο το σύστημα.

Η λογική του εγχειριδίου είναι η εξής: κάθε βήμα της επίθεσης άνοιξε επειδή υπήρχε **ένα λάθος στις ρυθμίσεις**. Αν διορθώσουμε το λάθος, το αντίστοιχο βήμα δεν γίνεται. Αν διορθώσουμε **περισσότερα από ένα** λάθη στην ίδια αλυσίδα, η επίθεση σταματά ακόμη κι αν μία από τις διορθώσεις αποτύχει. Αυτό λέγεται **Άμυνα σε Βάθος (Defense in Depth)**.

> ⚠️ **Πριν ξεκινήσετε:** πάρτε **snapshot** της VM (βλ. εργαστήριο [00](../00.%20Virtualization/README.md)). Έτσι μπορείτε να επιστρέψετε στην ευάλωτη κατάσταση, να ξανατρέξετε την επίθεση και να δείτε ότι πλέον αποτυγχάνει. Αυτή η σύγκριση «πριν/μετά» είναι το πιο χρήσιμο κομμάτι της άσκησης.

> ℹ️ Όλες οι εντολές εκτελούνται **στη μηχανή Raven** ως `root` (π.χ. μετά από `su -`). Οι IP διευθύνσεις είναι ενδεικτικές.

---

## Περιεχόμενα

1. [Χάρτης: Επίθεση → Αιτία → Διόρθωση](#1-χάρτης-επίθεση--αιτία--διόρθωση)
2. [Απογραφή του συστήματος (τι έχουμε στα χέρια μας)](#2-απογραφή-του-συστήματος)
3. [Αναβάθμιση λειτουργικού και πακέτων](#3-αναβάθμιση-λειτουργικού-και-πακέτων)
4. [Ελαχιστοποίηση επιφάνειας επίθεσης: υπηρεσίες & firewall](#4-ελαχιστοποίηση-επιφάνειας-επίθεσης)
5. [Θωράκιση Apache (σχόλια HTML, listing, banners)](#5-θωράκιση-apache)
6. [Θωράκιση WordPress (έκδοση, απαρίθμηση χρηστών, wp-config)](#6-θωράκιση-wordpress)
7. [Θωράκιση SSH & πολιτική κωδικών (hydra, michael:michael)](#7-θωράκιση-ssh--πολιτική-κωδικών)
8. [Θωράκιση MySQL (root χωρίς περιορισμούς)](#8-θωράκιση-mysql)
9. [Διόρθωση sudo (NOPASSWD python)](#9-διόρθωση-sudo)
10. [Hashes κωδικών & επαναχρησιμοποίηση κωδικών (John the Ripper)](#10-hashes-κωδικών--επαναχρησιμοποίηση)
11. [Επαλήθευση: ξανατρέχουμε την επίθεση](#11-επαλήθευση)
12. [Λίστα ελέγχου & ερωτήσεις αυτοελέγχου](#12-λίστα-ελέγχου--ερωτήσεις-αυτοελέγχου)

---

## 1. Χάρτης: Επίθεση → Αιτία → Διόρθωση

| Βήμα επίθεσης (README) | Τι εκμεταλλεύτηκε ο επιτιθέμενος | Ρίζα του προβλήματος | Διόρθωση (ενότητα) |
| :--- | :--- | :--- | :--- |
| 2. `nmap` | Θύρες 22, 80, 111 ανοιχτές προς όλους | Περιττές υπηρεσίες, κανένα firewall | §4 |
| 3. Flag 1 στο HTML | Σχόλιο `<!-- flag -->` στον κώδικα σελίδας | Ευαίσθητες πληροφορίες στον client | §5 |
| 4. DirBuster | Ανακάλυψη `/wordpress/`, `/vendor/` | Directory listing, αρχεία ανάπτυξης στο web root | §5 |
| 5. WPScan | Έκδοση WP, χρήστες `michael`, `steven` | Παλιό WordPress, ανοιχτή απαρίθμηση χρηστών | §3, §6 |
| 6. `hydra` → SSH | `michael:michael` | Κανένα όριο προσπαθειών, password login, αδύναμος κωδικός | §7 |
| 7. `wp-config.php` → MySQL `root` | Κωδικός root της βάσης σε αναγνώσιμο αρχείο | Η εφαρμογή χρησιμοποιεί τον `root` της MySQL | §6, §8 |
| 8. `sudo python` | `(ALL) NOPASSWD: /usr/bin/python` | Λάθος κανόνας sudoers | §9 |
| 8b. `john` | Hash του `steven` σπάει με `rockyou.txt` | Αδύναμος κωδικός, επαναχρησιμοποίηση κωδικών | §10 |

> 💡 Παρατηρήστε ότι **κανένα** βήμα δεν χρειάστηκε κάποιο «προχωρημένο» exploit. Όλα ήταν **λάθη ρυθμίσεων** (misconfigurations). Γι' αυτό η διόρθωσή τους είναι κυρίως θέμα πειθαρχίας και όχι κόστους.

---

## 2. Απογραφή του συστήματος

Δεν μπορούμε να προστατέψουμε κάτι που δεν ξέρουμε ότι υπάρχει. Ο διαχειριστής ξεκινά πάντα με την ίδια ερώτηση που έκανε ο επιτιθέμενος: **τι τρέχει σε αυτό το μηχάνημα;**

```bash
# Ποια διανομή και έκδοση έχουμε;
cat /etc/os-release
uname -r

# Ποιες υπηρεσίες ακούνε σε θύρες δικτύου;
ss -tulpn
```

* `cat /etc/os-release`: δείχνει τη διανομή. Στο Raven 1 είναι **Debian 8 (Jessie)**, έκδοση που δεν λαμβάνει πλέον ενημερώσεις ασφαλείας.
* `uname -r`: η έκδοση του πυρήνα (kernel). Παλιός πυρήνας σημαίνει γνωστές ευπάθειες τοπικής κλιμάκωσης προνομίων.
* `ss -tulpn`: το ίδιο που βλέπει το `nmap`, αλλά **από μέσα**. Οι σημαίες σημαίνουν: `-t` TCP, `-u` UDP, `-l` μόνο θύρες σε αναμονή (listening), `-p` ποια διεργασία την κατέχει, `-n` αριθμοί αντί για ονόματα.

Κοιτάξτε τη στήλη `Local Address`:

| Τιμή | Σημασία |
| :--- | :--- |
| `0.0.0.0:22` ή `*:22` | Ακούει σε **όλες** τις διεπαφές, άρα είναι προσβάσιμο από το δίκτυο |
| `127.0.0.1:3306` | Ακούει **μόνο τοπικά**, δεν φαίνεται από έξω (αυτό θέλουμε για τη MySQL) |

```bash
# Ποιοι χρήστες έχουν shell σύνδεσης;
grep -vE 'nologin|false' /etc/passwd

# Ποιοι κανόνες sudo υπάρχουν;
cat /etc/sudoers; ls -la /etc/sudoers.d/
```

Καταγράψτε τα αποτελέσματα. Θα τα συγκρίνετε με την κατάσταση μετά τη θωράκιση.

---

## 3. Αναβάθμιση λειτουργικού και πακέτων

### 3.1 Γιατί έχει σημασία

Το WPScan αναγνώρισε την ακριβή έκδοση του WordPress και τη συσχέτισε με γνωστές ευπάθειες (CVE). Το ίδιο ισχύει για Apache, PHP, MySQL και τον πυρήνα. **Μια γνωστή ευπάθεια σε παλιό λογισμικό είναι ο πιο φθηνός τρόπος παραβίασης**, γιατί το exploit υπάρχει ήδη δημόσια.

Ο κύκλος ζωής μιας διανομής έχει σημασία:

| Διανομή | Κατάσταση |
| :--- | :--- |
| Debian 8 Jessie (Raven 1) | **Τέλος υποστήριξης**. Τα αποθετήρια έχουν μεταφερθεί στο `archive.debian.org` |
| Debian 12 Bookworm / 13 Trixie | Υποστηρίζονται, λαμβάνουν security updates |

### 3.2 Άμεση ενημέρωση εντός της ίδιας έκδοσης

Επειδή το Jessie είναι αρχειοθετημένο, το `apt update` αποτυγχάνει με τα αρχικά αποθετήρια. Τα στρέφουμε στο αρχείο:

```bash
cp /etc/apt/sources.list /etc/apt/sources.list.bak
cat > /etc/apt/sources.list <<'EOF'
deb http://archive.debian.org/debian jessie main
deb http://archive.debian.org/debian-security jessie/updates main
EOF

apt-get -o Acquire::Check-Valid-Until=false update
apt-get upgrade
```

* Κρατάμε αντίγραφο (`.bak`) πριν αλλάξουμε αρχείο ρυθμίσεων. Είναι καλή συνήθεια για κάθε αλλαγή.
* `Acquire::Check-Valid-Until=false`: τα αρχειοθετημένα αποθετήρια έχουν «ληγμένη» υπογραφή χρόνου. Η παράμετρος λέει στο `apt` να τη δεχτεί. **Αυτό είναι αποδεκτό μόνο σε εργαστήριο**.
* `apt-get upgrade`: εγκαθιστά τις τελευταίες εκδόσεις που υπήρξαν ποτέ για το Jessie.

### 3.3 Η πραγματική λύση: μετάβαση σε υποστηριζόμενη έκδοση

Η ενημέρωση εντός Jessie κλείνει **κάποια** κενά, όχι όλα, γιατί για ευπάθειες μετά το τέλος υποστήριξης δεν θα βγει ποτέ διόρθωση. Σε παραγωγικό περιβάλλον η σωστή απόφαση είναι **νέα εγκατάσταση** σε Debian 12/13 και μεταφορά της εφαρμογής (αρχεία + dump βάσης). Η αναβάθμιση «σκαλί-σκαλί» (Jessie → Stretch → Buster → Bullseye → Bookworm) είναι δυνατή αλλά επίπονη και επιρρεπής σε σφάλματα.

### 3.4 Αυτόματες ενημερώσεις ασφαλείας (σε υποστηριζόμενο σύστημα)

```bash
apt install unattended-upgrades
dpkg-reconfigure -plow unattended-upgrades
```

Το πακέτο εγκαθιστά καθημερινά **μόνο** τις ενημερώσεις ασφαλείας. Έτσι το κενό μεταξύ «δημοσίευσης διόρθωσης» και «εφαρμογής της» μειώνεται από μήνες σε ώρες.

---

## 4. Ελαχιστοποίηση επιφάνειας επίθεσης

### 4.1 Απενεργοποίηση του rpcbind (θύρα 111)

Το `rpcbind` χρειάζεται μόνο για υπηρεσίες RPC όπως το **NFS**. Ο web server δεν τις χρησιμοποιεί. Κάθε υπηρεσία που δεν χρειάζεται είναι επιπλέον κώδικας που μπορεί να έχει ευπάθεια και επιπλέον πληροφορία για τον επιτιθέμενο.

```bash
# Βεβαιωνόμαστε ότι δεν υπάρχουν NFS mounts που το χρειάζονται
mount | grep nfs

# Απενεργοποίηση (Debian 8 χρησιμοποιεί systemd)
systemctl stop rpcbind.socket rpcbind.service
systemctl disable rpcbind.socket rpcbind.service
systemctl mask rpcbind.service
```

* `stop`: σταματά την υπηρεσία τώρα.
* `disable`: δεν ξεκινά στην επόμενη εκκίνηση.
* `mask`: την «κλειδώνει», ώστε να μην μπορεί να ξεκινήσει ούτε ως εξάρτηση άλλης υπηρεσίας.
* Απενεργοποιούμε και το `.socket`, γιατί με το socket activation το systemd θα ξεκινούσε ξανά την υπηρεσία μόλις έρθει σύνδεση στη θύρα 111.

Αν η υπηρεσία δεν χρειάζεται καθόλου: `apt purge rpcbind nfs-common`.

### 4.2 Firewall με πολιτική «απαγόρευση εκτός αν επιτρέπεται»

Η σωστή φιλοσοφία είναι το **default deny**: κλείνουμε τα πάντα και ανοίγουμε μόνο ό,τι χρειάζεται.

```bash
apt install ufw

ufw default deny incoming       # Όλα τα εισερχόμενα απορρίπτονται
ufw default allow outgoing      # Τα εξερχόμενα επιτρέπονται (θα το σφίξουμε αργότερα αν χρειαστεί)

ufw allow 80/tcp                # HTTP για το site
ufw allow 443/tcp               # HTTPS (μόλις ρυθμιστεί πιστοποιητικό)

# SSH μόνο από το υποδίκτυο διαχείρισης, ΟΧΙ από παντού
ufw allow from 192.168.56.0/24 to any port 22 proto tcp

ufw enable
ufw status verbose
```

> ⚠️ **Προσοχή στη σειρά!** Αν συνδέεστε μέσω SSH, προσθέστε τον κανόνα για τη θύρα 22 **πριν** το `ufw enable`, αλλιώς θα κλειδωθείτε έξω από το μηχάνημα.

**Γιατί περιορίζουμε το SSH σε υποδίκτυο;** Ο επιτιθέμενος έτρεξε το `hydra` από το ίδιο δίκτυο με το site. Αν το SSH δεχόταν συνδέσεις μόνο από ένα δίκτυο διαχείρισης, η επίθεση brute-force δεν θα ξεκινούσε καν.

Επιπλέον, το `ufw` έχει ενσωματωμένο **περιορισμό ρυθμού** (rate limiting):

```bash
ufw limit 22/tcp
```

Το `limit` απορρίπτει μια IP που ανοίγει **6 ή περισσότερες συνδέσεις μέσα σε 30 δευτερόλεπτα**. Το `hydra` ανοίγει δεκάδες παράλληλες συνδέσεις, άρα θα κοβόταν αμέσως.

---

## 5. Θωράκιση Apache

### 5.1 Ευαίσθητα σχόλια στον κώδικα HTML (Flag 1)

**Το πρόβλημα:** ό,τι στέλνει ο server στον browser το βλέπει **ο καθένας** με `F12` ή `Ctrl+U`. Τα σχόλια HTML (`<!-- -->`) δεν είναι κρυφά, απλώς δεν εμφανίζονται στη σελίδα.

**Έλεγχος:** ψάχνουμε σχόλια σε όλα τα στατικά αρχεία:

```bash
grep -rn '<!--' /var/www/html --include=*.html --include=*.php | less
```

**Διόρθωση:** αφαιρούμε κάθε σχόλιο που περιέχει πληροφορίες (κωδικούς, διαδρομές, ονόματα χρηστών, σημειώσεις προγραμματιστών). Σε μια σωστή διαδικασία ανάπτυξης, τα σχόλια αφαιρούνται αυτόματα στο στάδιο «build» πριν το ανέβασμα στον server.

### 5.2 Directory Listing (DirBuster)

**Το πρόβλημα:** όταν ένας κατάλογος δεν έχει `index.html`/`index.php`, ο Apache με την επιλογή `Indexes` εμφανίζει **λίστα όλων των αρχείων** του. Ο επιτιθέμενος βλέπει όλη τη δομή χωρίς καν να μαντέψει ονόματα.

**Έλεγχος:**

```bash
grep -rn "Indexes" /etc/apache2/
curl -s http://localhost/vendor/ | head
```

Αν το `curl` επιστρέψει `Index of /vendor`, το listing είναι ενεργό.

**Διόρθωση:** στο `/etc/apache2/apache2.conf`, στο μπλοκ του `/var/www/`:

```apache
<Directory /var/www/>
    Options -Indexes +FollowSymLinks
    AllowOverride All
    Require all granted
</Directory>
```

* `-Indexes`: το πρόσημο `-` **αφαιρεί** την επιλογή. Ο Apache πλέον απαντά `403 Forbidden` αντί για λίστα αρχείων.
* `AllowOverride All`: χρειάζεται ώστε το WordPress να χρησιμοποιεί `.htaccess` για τα permalinks.

### 5.3 Αρχεία ανάπτυξης μέσα στο web root (`/vendor/`)

Ο κατάλογος `vendor/` (βιβλιοθήκες Composer, π.χ. PHPMailer) **δεν πρέπει ποτέ** να είναι προσβάσιμος από τον browser. Το ίδιο ισχύει για `.git/`, `composer.json`, `*.bak`, `*.sql`.

```apache
# /etc/apache2/conf-available/deny-dev-files.conf
<DirectoryMatch "/(vendor|\.git|node_modules)/">
    Require all denied
</DirectoryMatch>

<FilesMatch "\.(bak|sql|old|swp|log|md|json|lock)$">
    Require all denied
</FilesMatch>
```

```bash
a2enconf deny-dev-files
```

Ακόμα καλύτερη λύση είναι να **μετακινηθεί** το `vendor/` έξω από το `/var/www/html` και να φορτώνεται από τον κώδικα PHP με απόλυτη διαδρομή.

### 5.4 Απόκρυψη εκδόσεων (banners)

Το `nmap -sV` και οι σελίδες σφάλματος του Apache αποκαλύπτουν εκδόσεις (`Apache/2.4.10 (Debian)`). Η απόκρυψη **δεν διορθώνει** καμία ευπάθεια, αλλά κάνει την αναγνώριση πιο αργή και θορυβώδη.

```apache
# /etc/apache2/conf-available/security.conf
ServerTokens Prod        # Header: "Server: Apache" χωρίς έκδοση/OS
ServerSignature Off      # Χωρίς υπογραφή στο κάτω μέρος των σελίδων σφάλματος
TraceEnable Off          # Απενεργοποιεί τη μέθοδο HTTP TRACE

Header always set X-Content-Type-Options "nosniff"
Header always set X-Frame-Options "SAMEORIGIN"
```

```bash
# Το ίδιο και για την PHP: κρύβει το header "X-Powered-By: PHP/5.x"
sed -i 's/^expose_php = On/expose_php = Off/' /etc/php5/apache2/php.ini

a2enmod headers
apache2ctl configtest      # ΠΑΝΤΑ έλεγχος σύνταξης πριν την επανεκκίνηση
systemctl reload apache2
```

> 💡 Το `apache2ctl configtest` πρέπει να απαντά `Syntax OK`. Αν κάνετε `reload` με λάθος ρύθμιση, ο Apache μπορεί να μη σηκωθεί και το site να πέσει.

---

## 6. Θωράκιση WordPress

### 6.1 Αναβάθμιση πυρήνα, plugins, themes

Το WordPress του Raven 1 είναι έκδοση **4.8.x** (2017). Το WPScan έδειξε δεκάδες γνωστές ευπάθειες για αυτή την έκδοση.

Με το **WP-CLI** (το επίσημο εργαλείο γραμμής εντολών):

```bash
# Εγκατάσταση WP-CLI
curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
chmod +x wp-cli.phar && mv wp-cli.phar /usr/local/bin/wp

cd /var/www/html/wordpress

# Πάντα backup πριν από αναβάθμιση
sudo -u www-data wp db export /root/wp-backup.sql
tar czf /root/wp-files-backup.tgz /var/www/html/wordpress

sudo -u www-data wp core version
sudo -u www-data wp core update
sudo -u www-data wp plugin update --all
sudo -u www-data wp theme update --all
sudo -u www-data wp core verify-checksums
```

* Εκτελούμε ως `www-data` (`sudo -u`) για να μη δημιουργηθούν αρχεία που ανήκουν στον root.
* `wp core verify-checksums`: συγκρίνει κάθε αρχείο του πυρήνα με τα επίσημα checksums. Αν κάποιο αρχείο έχει πειραχτεί (π.χ. ο επιτιθέμενος έβαλε backdoor), θα φανεί εδώ.

> ⚠️ Οι πρόσφατες εκδόσεις του WordPress απαιτούν PHP 7.2.24+ (συνιστάται 8.x). Το Debian 8 έχει PHP 5.6. Άλλος ένας λόγος για τον οποίο η μετάβαση σε υποστηριζόμενη διανομή (§3.3) είναι αναγκαία.

### 6.2 Αποτροπή απαρίθμησης χρηστών (WPScan `-e u`)

Το WPScan βρήκε τους `michael` και `steven` με δύο βασικούς τρόπους:

1. **Author archives:** `/?author=1` κάνει ανακατεύθυνση στο `/author/michael/`.
2. **REST API:** `/wp-json/wp/v2/users` επιστρέφει λίστα χρηστών σε JSON.

Μπλοκάρουμε και τα δύο στο `.htaccess` του WordPress (πάνω από το μπλοκ `# BEGIN WordPress`):

```apache
<IfModule mod_rewrite.c>
RewriteEngine On
# 1. Μπλοκάρισμα ?author=N
RewriteCond %{QUERY_STRING} (^|&)author=\d+ [NC]
RewriteRule ^ - [F,L]
# 2. Μπλοκάρισμα της λίστας χρηστών στο REST API
RewriteRule ^wp-json/wp/v2/users - [F,L]
</IfModule>
```

* `RewriteCond`: η συνθήκη. Ελέγχει αν το query string περιέχει `author=` ακολουθούμενο από αριθμό.
* `RewriteRule ^ - [F,L]`: `-` σημαίνει «μην αλλάξεις το URL», `F` επιστρέφει `403 Forbidden`, `L` σταματά την επεξεργασία κανόνων.

```bash
a2enmod rewrite && systemctl reload apache2
```

**Επιπλέον:** τα ονόματα σύνδεσης (`user_login`) δεν πρέπει να είναι ίδια με τα δημόσια ονόματα εμφάνισης (`display_name`). Αν το δημόσιο όνομα είναι «Michael», το όνομα σύνδεσης ας είναι κάτι μη προβλέψιμο.

### 6.3 Προστασία του `wp-config.php`

**Το πρόβλημα:** ο `michael` (απλός χρήστης SSH) διάβασε το `wp-config.php`, γιατί είχε δικαιώματα ανάγνωσης για όλους (`644`).

**Διόρθωση δικαιωμάτων:**

```bash
cd /var/www/html/wordpress
ls -l wp-config.php                 # Πριν: -rw-r--r-- (644) → διαβάζεται από όλους

chown root:www-data wp-config.php
chmod 640 wp-config.php
ls -l wp-config.php                 # Μετά: -rw-r----- root www-data
```

| Ψηφίο | Ποιος | Δικαίωμα | Γιατί |
| :---: | :--- | :--- | :--- |
| `6` | owner (`root`) | read + write | Μόνο ο διαχειριστής το αλλάζει |
| `4` | group (`www-data`) | read | Ο Apache/PHP χρειάζεται να το διαβάσει |
| `0` | others | τίποτα | Ο `michael` και κάθε άλλος χρήστης δεν το βλέπει |

**Μην χρησιμοποιείτε τον `root` της MySQL στο WordPress.** Θα φτιάξουμε ξεχωριστό χρήστη με περιορισμένα δικαιώματα στην §8.3.

**Επιπλέον ρυθμίσεις στο `wp-config.php`:**

```php
define('DISALLOW_FILE_EDIT', true);   // Απενεργοποιεί τον επεξεργαστή θεμάτων/plugins στο dashboard
define('FORCE_SSL_ADMIN', true);      // Το wp-admin μόνο μέσω HTTPS (όταν υπάρχει πιστοποιητικό)
define('WP_DEBUG', false);            // Χωρίς μηνύματα σφάλματος που αποκαλύπτουν διαδρομές
```

Και ανανεώνουμε τα **salts** (κλειδιά που υπογράφουν τα cookies σύνδεσης), ώστε να αποσυνδεθούν όλες οι υπάρχουσες συνεδρίες, συμπεριλαμβανομένου πιθανώς του επιτιθέμενου:

```bash
sudo -u www-data wp config shuffle-salts
```

### 6.4 Δικαιώματα αρχείων όλου του WordPress

Ο κανόνας είναι: **ο web server διαβάζει, δεν γράφει** (εκτός από τον φάκελο uploads).

```bash
cd /var/www/html/wordpress
chown -R root:www-data .
find . -type d -exec chmod 750 {} \;
find . -type f -exec chmod 640 {} \;

# Μόνο ο φάκελος uploads χρειάζεται εγγραφή από τον web server
chown -R www-data:www-data wp-content/uploads
```

Και απαγορεύουμε την εκτέλεση PHP μέσα στα uploads, ώστε ένα ανεβασμένο αρχείο `shell.php` να μη μπορεί να τρέξει:

```apache
# /var/www/html/wordpress/wp-content/uploads/.htaccess
<FilesMatch "\.(php|phtml|php\d)$">
    Require all denied
</FilesMatch>
```

---

## 7. Θωράκιση SSH & πολιτική κωδικών

Αυτό ήταν το **κρίσιμο** σημείο εισόδου: το `hydra` βρήκε το `michael:michael`. Τρία ανεξάρτητα λάθη συνέβαλαν:

1. Το SSH δεχόταν **κωδικούς** αντί για κλειδιά.
2. Δεν υπήρχε **όριο αποτυχημένων προσπαθειών**.
3. Ο κωδικός ήταν **ίδιος με το όνομα χρήστη**.

### 7.1 Αυθεντικοποίηση με κλειδιά αντί για κωδικούς

Ένα κλειδί Ed25519 είναι πρακτικά αδύνατο να σπάσει με brute-force, σε αντίθεση με έναν κωδικό που μπορεί να βρίσκεται στο `rockyou.txt`.

**Στον υπολογιστή του διαχειριστή (όχι στο Raven):**

```bash
ssh-keygen -t ed25519 -C "admin@lab"     # Δημιουργεί ζεύγος ιδιωτικού/δημόσιου κλειδιού
ssh-copy-id michael@<RAVEN_IP>           # Αντιγράφει το δημόσιο κλειδί στο ~/.ssh/authorized_keys
ssh michael@<RAVEN_IP>                   # Δοκιμή: πρέπει να μπει ΧΩΡΙΣ κωδικό
```

> ⚠️ **Μην απενεργοποιήσετε τους κωδικούς πριν επιβεβαιώσετε ότι η σύνδεση με κλειδί λειτουργεί.** Αλλιώς κλειδώνεστε έξω.

### 7.2 Ρύθμιση του `sshd_config`

```bash
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
nano /etc/ssh/sshd_config
```

```text
PermitRootLogin no               # Ο root δεν συνδέεται απευθείας, μόνο μέσω su/sudo
PasswordAuthentication no        # Μόνο κλειδιά: το hydra δεν έχει τι να δοκιμάσει
PubkeyAuthentication yes
PermitEmptyPasswords no
ChallengeResponseAuthentication no
MaxAuthTries 3                   # Μέγιστο 3 προσπάθειες ανά σύνδεση
LoginGraceTime 30                # 30 δευτ. για να ολοκληρωθεί η σύνδεση
MaxStartups 10:30:60             # Περιορίζει ταυτόχρονες μη-αυθεντικοποιημένες συνδέσεις
AllowUsers michael steven        # Λευκή λίστα: μόνο αυτοί μπορούν να συνδεθούν
X11Forwarding no
ClientAliveInterval 300
ClientAliveCountMax 2            # Αποσύνδεση αδρανών συνεδριών μετά από ~10 λεπτά
```

Σημαντικές επεξηγήσεις:

* **`MaxStartups 10:30:60`**: μετά από 10 ανοιχτές μη-αυθεντικοποιημένες συνδέσεις, ο `sshd` αρχίζει να απορρίπτει νέες με πιθανότητα 30%, που αυξάνεται γραμμικά μέχρι 100% στις 60. Αυτό «φρενάρει» τα εργαλεία που ανοίγουν πολλές παράλληλες συνδέσεις, όπως το `hydra -t 16`.
* **`AllowUsers`**: ακόμη και αν δημιουργηθεί αργότερα κάποιος λογαριασμός με αδύναμο κωδικό (π.χ. λογαριασμός υπηρεσίας), δεν θα μπορεί να συνδεθεί μέσω SSH.

```bash
sshd -t                     # Έλεγχος σύνταξης. Καμία έξοδος = OK
systemctl reload ssh
```

> 💡 Κρατήστε **ανοιχτή** την τρέχουσα συνεδρία SSH και δοκιμάστε σύνδεση από **δεύτερο** τερματικό. Αν κάτι πάει στραβά, έχετε ακόμη πρόσβαση για να το διορθώσετε.

### 7.3 Πολιτική ισχυρών κωδικών (PAM)

Ακόμα και με κλειδιά στο SSH, οι κωδικοί χρησιμοποιούνται για `sudo`, `su` και την κονσόλα. Το **PAM** (Pluggable Authentication Modules) ελέγχει κάθε νέο κωδικό πριν τον δεχτεί.

```bash
apt install libpam-pwquality
```

Στο `/etc/security/pwquality.conf`:

```text
minlen = 14          # Ελάχιστο μήκος 14 χαρακτήρων
minclass = 3         # Τουλάχιστον 3 κατηγορίες (πεζά, κεφαλαία, ψηφία, σύμβολα)
maxrepeat = 3        # Όχι πάνω από 3 ίδιους χαρακτήρες στη σειρά
usercheck = 1        # Απορρίπτει κωδικό που περιέχει το όνομα χρήστη (michael:michael!)
dictcheck = 1        # Απορρίπτει λέξεις λεξικού
enforce_for_root
```

Η ρύθμιση `usercheck = 1` θα είχε αποτρέψει **ακριβώς** τον κωδικό `michael`.

### 7.4 Κλείδωμα λογαριασμού μετά από αποτυχίες

Το PAM μπορεί να κλειδώνει προσωρινά λογαριασμό μετά από διαδοχικές αποτυχίες. Στο Debian 8 το module είναι το `pam_tally2` (στα νεότερα συστήματα το `pam_faillock`).

**Debian 8 — `/etc/pam.d/common-auth`** (στην αρχή του αρχείου):

```text
auth required pam_tally2.so onerr=fail deny=5 unlock_time=900
```

**Debian 11+ / Ubuntu 22.04+ — `/etc/security/faillock.conf`:**

```text
deny = 5
unlock_time = 900
fail_interval = 900
```

Μετά από 5 αποτυχίες ο λογαριασμός κλειδώνει για 15 λεπτά. Με ρυθμό 5 δοκιμές ανά 15 λεπτά, η δοκιμή των 14 εκατομμυρίων κωδικών του `rockyou.txt` θα χρειαζόταν περίπου 80 χρόνια.

```bash
pam_tally2 --user michael            # Πόσες αποτυχίες έχει ο χρήστης (Debian 8)
pam_tally2 --user michael --reset    # Ξεκλείδωμα από τον διαχειριστή
```

### 7.5 Αλλαγή των εκτεθειμένων κωδικών

Οι κωδικοί `michael` και `pink84` είναι πλέον **γνωστοί**. Αλλάζουν υποχρεωτικά:

```bash
passwd michael
passwd steven
passwd root
chage -d 0 michael     # Αναγκάζει τον χρήστη να αλλάξει κωδικό στην επόμενη σύνδεση
```

---

## 8. Θωράκιση MySQL

### 8.1 Τι πήγε στραβά

Από το `wp-config.php` ο επιτιθέμενος πήρε τον κωδικό του **`root` της MySQL** (`R@v3nSecurity`). Με αυτόν διάβασε **όλες** τις βάσεις, εξήγαγε hashes και (στο Raven 2) έγραψε αρχεία στον δίσκο. Το WordPress δεν χρειάζεται καθόλου αυτά τα δικαιώματα.

### 8.2 Βασική θωράκιση με `mysql_secure_installation`

```bash
mysql_secure_installation
```

Το εργαλείο κάνει διαδοχικά:

1. Αλλαγή του κωδικού του `root` της βάσης (απαντήστε **Y** και ορίστε **νέο** κωδικό).
2. Διαγραφή ανώνυμων χρηστών.
3. Απαγόρευση σύνδεσης του `root` από απομακρυσμένους υπολογιστές.
4. Διαγραφή της δοκιμαστικής βάσης `test`.
5. Επαναφόρτωση των πινάκων δικαιωμάτων.

### 8.3 Ξεχωριστός χρήστης για την εφαρμογή (Least Privilege)

```sql
-- Συνδεόμαστε ως root της βάσης: mysql -u root -p
CREATE USER 'wp_app'@'localhost' IDENTIFIED BY 'Ένας-Μακρύς-Τυχαίος-Κωδικός-2026!';

GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, ALTER, INDEX, DROP
  ON wordpress.* TO 'wp_app'@'localhost';

FLUSH PRIVILEGES;
SHOW GRANTS FOR 'wp_app'@'localhost';
```

* `'wp_app'@'localhost'`: ο χρήστης μπορεί να συνδεθεί **μόνο** από τον ίδιο server.
* `ON wordpress.*`: δικαιώματα **μόνο** στη βάση `wordpress`, όχι στη `mysql` (εκεί που βρίσκονται οι κωδικοί των χρηστών της βάσης).
* **Δεν** δίνουμε `FILE` (ανάγνωση/εγγραφή αρχείων στον δίσκο), `SUPER`, `GRANT OPTION`, `PROCESS`.

Ενημερώνουμε το `wp-config.php`:

```php
define('DB_USER', 'wp_app');
define('DB_PASSWORD', 'Ένας-Μακρύς-Τυχαίος-Κωδικός-2026!');
```

Ακόμα κι αν ο επιτιθέμενος διαβάσει ξανά το `wp-config.php`, θα έχει πρόσβαση μόνο στη βάση του WordPress, όχι σε όλο τον server βάσεων.

### 8.4 Ρυθμίσεις του `mysqld`

Στο `/etc/mysql/my.cnf` (ή `/etc/mysql/mysql.conf.d/mysqld.cnf`), ενότητα `[mysqld]`:

```ini
[mysqld]
user             = mysql      # Η υπηρεσία τρέχει ως χρήστης mysql, ΟΧΙ ως root
bind-address     = 127.0.0.1  # Ακούει μόνο τοπικά, δεν είναι προσβάσιμη από το δίκτυο
local-infile     = 0          # Απενεργοποιεί το LOAD DATA LOCAL INFILE
secure-file-priv = /var/lib/mysql-files   # Περιορίζει LOAD_FILE / INTO OUTFILE σε έναν φάκελο
symbolic-links   = 0
```

```bash
mkdir -p /var/lib/mysql-files && chown mysql:mysql /var/lib/mysql-files && chmod 750 /var/lib/mysql-files
systemctl restart mysql

# Επαλήθευση: ποιος χρήστης τρέχει τη MySQL;
ps -o user,cmd -C mysqld
```

Το **`user = mysql`** είναι κρίσιμο: στο Raven η MySQL έτρεχε ως `root`, οπότε ό,τι έκανε η βάση (π.χ. εγγραφή αρχείου) γινόταν με δικαιώματα root. Η εκμετάλλευση αυτού του λάθους αναλύεται στο [Hardening-Guide του Raven 2](../04.%20Raven2-CTF/Hardening-Guide.md).

---

## 9. Διόρθωση sudo

### 9.1 Γιατί το `NOPASSWD: /usr/bin/python` ισοδυναμεί με root

```text
User steven may run the following commands on raven:
    (ALL) NOPASSWD: /usr/bin/python
```

Ο διαχειριστής ίσως σκέφτηκε «δίνω στον steven μόνο την Python». Όμως η Python είναι **πλήρης γλώσσα προγραμματισμού**: μπορεί να ανοίξει shell, να διαβάσει/γράψει οποιοδήποτε αρχείο, να αλλάξει κωδικούς. Το ίδιο ισχύει για `perl`, `ruby`, `vim`, `less`, `find`, `awk`, `nmap`, `tar` κ.ά. Ο ιστότοπος **GTFOBins** καταγράφει εκατοντάδες τέτοια προγράμματα.

> 📌 **Κανόνας:** δώσε sudo σε ένα πρόγραμμα **μόνο** αν είσαι βέβαιος ότι δεν μπορεί να εκτελέσει άλλες εντολές ή να γράψει αυθαίρετα αρχεία.

### 9.2 Έλεγχος

```bash
sudo -l -U steven               # Τι μπορεί να τρέξει ο steven
grep -rn NOPASSWD /etc/sudoers /etc/sudoers.d/
```

### 9.3 Διόρθωση

**Πάντα** με `visudo`, ποτέ με απευθείας επεξεργασία του αρχείου:

```bash
visudo                       # Για το /etc/sudoers
visudo -f /etc/sudoers.d/steven
```

Το `visudo` ελέγχει τη σύνταξη πριν αποθηκεύσει. Ένα συντακτικό λάθος στο `sudoers` μπορεί να απενεργοποιήσει το `sudo` εντελώς.

**Αφαιρούμε** τη γραμμή:

```text
steven ALL=(ALL) NOPASSWD: /usr/bin/python
```

Αν ο `steven` χρειάζεται πράγματι να κάνει κάτι συγκεκριμένο ως root (π.χ. να επανεκκινεί τον Apache), του δίνουμε **ακριβώς** αυτό:

```text
steven ALL=(root) /bin/systemctl reload apache2
```

* Χωρίς `NOPASSWD`: ο χρήστης πρέπει να δώσει τον κωδικό του. Έτσι, αν κάποιος κλέψει τη συνεδρία του, δεν παίρνει αυτόματα δικαιώματα.
* Συγκεκριμένη εντολή **με** παραμέτρους: δεν μπορεί να τρέξει `systemctl` με άλλες παραμέτρους.

Αν χρειάζεται να τρέξει **δικό του** script ως root, το script πρέπει να ανήκει στον root και να μην είναι εγγράψιμο από τον `steven`:

```bash
chown root:root /usr/local/sbin/backup.sh
chmod 755 /usr/local/sbin/backup.sh     # Ο steven εκτελεί, ΔΕΝ τροποποιεί
```

### 9.4 Ενισχυτικές ρυθμίσεις sudo

```text
Defaults    use_pty                         # Το sudo τρέχει σε δικό του pseudo-terminal
Defaults    logfile="/var/log/sudo.log"     # Καταγραφή κάθε χρήσης του sudo
Defaults    timestamp_timeout=5             # Ο κωδικός «θυμάται» μόνο για 5 λεπτά
Defaults    passwd_tries=3
```

---

## 10. Hashes κωδικών & επαναχρησιμοποίηση

### 10.1 Τι έδειξε το John the Ripper

Ο John έσπασε το hash του `steven` (`pink84`) επειδή ο κωδικός υπήρχε στο `rockyou.txt`. **Το πρόβλημα δεν ήταν ο αλγόριθμος, ήταν ο κωδικός.** Κανένας αλγόριθμος hashing δεν προστατεύει έναν κωδικό που βρίσκεται σε δημόσιο λεξικό: ο επιτιθέμενος απλώς υπολογίζει το hash κάθε λέξης και συγκρίνει.

Το δεύτερο πρόβλημα ήταν η **επαναχρησιμοποίηση**: ο ίδιος κωδικός ίσχυε στο WordPress **και** στο λειτουργικό σύστημα.

### 10.2 Διόρθωση

1. **Μοναδικοί κωδικοί ανά σύστημα.** Ο κωδικός WordPress ≠ κωδικός Linux ≠ κωδικός MySQL ≠ κωδικός root. Χρησιμοποιήστε password manager (π.χ. KeePassXC).
2. **Ισχυρότεροι αλγόριθμοι.** Το WordPress από την έκδοση 6.8 αποθηκεύει τους κωδικούς με **bcrypt** αντί για phpass, οπότε η αναβάθμιση (§6.1) βοηθά και εδώ. Οι υπάρχοντες κωδικοί μετατρέπονται αυτόματα στην επόμενη σύνδεση του χρήστη.
3. **Ισχυρό hashing στο Linux.** Ελέγξτε τον αλγόριθμο στο `/etc/shadow`:

   ```bash
   awk -F: '$2 ~ /^\$/ {print $1, substr($2,1,4)}' /etc/shadow
   ```

   | Πρόθεμα | Αλγόριθμος | Εκτίμηση |
   | :--- | :--- | :--- |
   | `$1$` | MD5-crypt | Ακατάλληλο |
   | `$6$` | SHA-512-crypt | Αποδεκτό |
   | `$y$` | yescrypt | Προεπιλογή σε Debian 11+, το καλύτερο |

   Στο `/etc/login.defs` ορίζουμε `ENCRYPT_METHOD SHA512` (ή `YESCRYPT` σε νεότερα συστήματα). Ισχύει για **νέους** κωδικούς, γι' αυτό η αλλαγή κωδικών της §7.5 γίνεται **μετά** από αυτή τη ρύθμιση.

4. **Ξεχωριστός κωδικός root, κλειδωμένος root.** Αφού οι διαχειριστές χρησιμοποιούν `sudo`, ο άμεσος λογαριασμός root μπορεί να κλειδωθεί:

   ```bash
   passwd -l root      # Κλειδώνει τον κωδικό του root. Το `su -` δεν δουλεύει πλέον
   ```

   > ⚠️ Κάντε το **μόνο** αφού επιβεβαιώσετε ότι τουλάχιστον ένας χρήστης έχει πλήρη δικαιώματα sudo, αλλιώς χάνετε πρόσβαση διαχειριστή.

---

## 11. Επαλήθευση

Μια διόρθωση δεν θεωρείται ολοκληρωμένη μέχρι να **αποδειχτεί** ότι η επίθεση αποτυγχάνει. Από το Kali, επαναλαμβάνουμε κάθε βήμα:

```bash
# 1. Σάρωση: πρέπει να φαίνεται μόνο το 80 (και το 22 μόνο από το δίκτυο διαχείρισης)
nmap -sV -p- <RAVEN_IP>

# 2. Directory listing: πρέπει 403
curl -I http://<RAVEN_IP>/vendor/

# 3. Απαρίθμηση χρηστών: πρέπει 403
curl -I "http://<RAVEN_IP>/wordpress/?author=1"
curl -I http://<RAVEN_IP>/wordpress/wp-json/wp/v2/users

# 4. SSH με κωδικό: πρέπει "Permission denied (publickey)"
ssh -o PreferredAuthentications=password michael@<RAVEN_IP>

# 5. Hydra: πρέπει να αποτύχει αμέσως
hydra -l michael -P /usr/share/wordlists/rockyou.txt ssh://<RAVEN_IP>
```

Στο **Raven**:

```bash
# 6. Ο michael δεν διαβάζει το wp-config
sudo -u michael cat /var/www/html/wordpress/wp-config.php     # → Permission denied

# 7. Ο steven δεν έχει πλέον sudo python
sudo -l -U steven

# 8. Η MySQL τρέχει ως mysql και ακούει μόνο τοπικά
ps -o user,cmd -C mysqld
ss -tlnp | grep 3306                                         # → 127.0.0.1:3306
```

Καταγράψτε τα αποτελέσματα σε πίνακα «Πριν / Μετά». Αυτός ο πίνακας είναι το βασικό παραδοτέο μιας αναφοράς αποκατάστασης.

---

## 12. Λίστα ελέγχου & ερωτήσεις αυτοελέγχου

### 12.1 Λίστα ελέγχου

- [ ] Snapshot πριν τις αλλαγές
- [ ] Ενημέρωση πακέτων / σχέδιο μετάβασης σε υποστηριζόμενο Debian
- [ ] `rpcbind` απενεργοποιημένο, firewall `default deny`, SSH μόνο από δίκτυο διαχείρισης
- [ ] Σχόλια HTML καθαρισμένα, `Options -Indexes`, `/vendor` μπλοκαρισμένο
- [ ] `ServerTokens Prod`, `ServerSignature Off`, `expose_php Off`
- [ ] WordPress ενημερωμένο, απαρίθμηση χρηστών μπλοκαρισμένη, `DISALLOW_FILE_EDIT`
- [ ] `wp-config.php` σε `640 root:www-data`, νέα salts
- [ ] SSH: κλειδιά, `PasswordAuthentication no`, `PermitRootLogin no`, `MaxAuthTries 3`, `AllowUsers`
- [ ] PAM: `pwquality` (`usercheck=1`) και κλείδωμα μετά από 5 αποτυχίες
- [ ] Όλοι οι εκτεθειμένοι κωδικοί άλλαξαν
- [ ] MySQL: νέος κωδικός root, χρήστης `wp_app` με ελάχιστα δικαιώματα, `bind-address 127.0.0.1`, `user = mysql`
- [ ] `sudoers`: αφαιρέθηκε το `NOPASSWD: /usr/bin/python`
- [ ] Επαλήθευση όλων των βημάτων της §11

### 12.2 Ερωτήσεις αυτοελέγχου

1. Γιατί η απόκρυψη της έκδοσης του Apache (`ServerTokens Prod`) **δεν** θεωρείται διόρθωση ευπάθειας; Τι προσφέρει τελικά;
2. Ο `michael` είχε κωδικό ίδιο με το όνομα χρήστη. Ποια **τρία** ανεξάρτητα μέτρα αυτού του εγχειριδίου θα είχαν αποτρέψει, το καθένα μόνο του, την επιτυχία του `hydra`;
3. Γιατί το `wp-config.php` πρέπει να έχει group `www-data` και όχι να ανήκει εξ ολοκλήρου στον `www-data`;
4. Εξηγήστε γιατί το `sudo` σε `/usr/bin/python` ισοδυναμεί με πλήρη πρόσβαση root. Αναφέρετε τρία ακόμη προγράμματα με την ίδια ιδιότητα.
5. Ποια είναι η διαφορά μεταξύ `systemctl disable` και `systemctl mask`;
6. Αν ο κωδικός του `steven` είχε αποθηκευτεί με bcrypt αντί για phpass, θα είχε αποτύχει το John the Ripper; Αιτιολογήστε.
7. Ποιο δικαίωμα MySQL επιτρέπει τις εντολές `LOAD_FILE()` και `INTO DUMPFILE`, και γιατί δεν πρέπει να το έχει ο χρήστης της εφαρμογής;

---

## Πηγές

- [Debian Security Manual — Securing Debian](https://www.debian.org/doc/manuals/securing-debian-manual/)
- [WordPress — Hardening WordPress](https://developer.wordpress.org/advanced-administration/security/hardening/)
- [OpenSSH — sshd_config(5)](https://man.openbsd.org/sshd_config)
- [MySQL — Security Guidelines](https://dev.mysql.com/doc/refman/8.0/en/security-guidelines.html)
- [GTFOBins — python](https://gtfobins.github.io/gtfobins/python/)
- [CIS Benchmarks — Debian Linux / Apache HTTP Server](https://www.cisecurity.org/cis-benchmarks)
