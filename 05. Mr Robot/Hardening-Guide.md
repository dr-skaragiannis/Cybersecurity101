# Mr. Robot — Εγχειρίδιο Θωράκισης (Hardening & Remediation Guide)

Το [README.md](README.md) (§10) κατέγραψε έξι αδυναμίες με ταξινόμηση CWE και μια σύντομη λίστα μέτρων. Αυτό το εγχειρίδιο είναι η **πρακτική εφαρμογή** εκείνης της λίστας: για κάθε αδυναμία εξηγεί **γιατί** υπάρχει, **πώς** τη βρίσκουμε από μέσα, **ποια ρύθμιση** τη διορθώνει και **πώς αποδεικνύουμε** ότι η διόρθωση λειτουργεί.

Η αλυσίδα επίθεσης του Mr. Robot είναι χαρακτηριστική για WordPress:

```
robots.txt (Key 1 + λεξικό) → διαφορετικά μηνύματα σφάλματος → hydra χωρίς όριο
   → elliot:ER28-0652 → Theme Editor → PHP reverse shell (daemon)
   → password.raw-md5 αναγνώσιμο → su robot → SUID nmap 3.81 → root
```

**Κανένα** βήμα δεν χρειάστηκε ευπάθεια στον κώδικα του WordPress. Όλα ήταν **ρυθμίσεις** και **πρακτικές διαχείρισης**. Αυτό σημαίνει ότι η αναβάθμιση μόνη της **δεν** θα σταματούσε την επίθεση. Χρειάζονται οι ρυθμίσεις αυτού του εγχειριδίου.

> ⚠️ **Πριν ξεκινήσετε:** αν δουλεύετε σε τοπική VM, πάρτε **snapshot**. Αν δουλεύετε στο TryHackMe, οι αλλαγές χάνονται όταν τερματίσει η μηχανή, οπότε κρατήστε τις εντολές σε ένα αρχείο για να τις εφαρμόσετε ξανά.

> ℹ️ Οι εντολές εκτελούνται **στη μηχανή Mr. Robot ως `root`** (μετά το βήμα 8 του README). Οι IP είναι ενδεικτικές.

---

## Περιεχόμενα

1. [Χάρτης: Επίθεση → Αιτία → Διόρθωση](#1-χάρτης-επίθεση--αιτία--διόρθωση)
2. [Απογραφή: πού βρίσκονται τα πράγματα (Bitnami stack)](#2-απογραφή-πού-βρίσκονται-τα-πράγματα)
3. [Εξυγίανση του `robots.txt` και του web root](#3-εξυγίανση-του-robotstxt-και-του-web-root)
4. [Αποτροπή απαρίθμησης χρηστών (CWE-204)](#4-αποτροπή-απαρίθμησης-χρηστών)
5. [Προστασία της σελίδας σύνδεσης από brute-force](#5-προστασία-της-σελίδας-σύνδεσης-από-brute-force)
6. [Κωδικοί και έλεγχος ταυτότητας δύο παραγόντων (CWE-521)](#6-κωδικοί-και-έλεγχος-ταυτότητας-δύο-παραγόντων)
7. [Απενεργοποίηση του Theme Editor και κλείδωμα αρχείων (CWE-94)](#7-απενεργοποίηση-του-theme-editor-και-κλείδωμα-αρχείων)
8. [PHP και εξερχόμενες συνδέσεις (αποτροπή reverse shell)](#8-php-και-εξερχόμενες-συνδέσεις)
9. [Ο λογαριασμός υπηρεσίας `daemon`](#9-ο-λογαριασμός-υπηρεσίας-daemon)
10. [Κατάλογοι χρηστών και αποθήκευση κωδικών (CWE-327)](#10-κατάλογοι-χρηστών-και-αποθήκευση-κωδικών)
11. [SUID στο nmap (CWE-250)](#11-suid-στο-nmap)
12. [Αναβάθμιση λειτουργικού, WordPress, HTTPS](#12-αναβάθμιση-λειτουργικού-wordpress-https)
13. [Επαλήθευση](#13-επαλήθευση)
14. [Λίστα ελέγχου & ερωτήσεις αυτοελέγχου](#14-λίστα-ελέγχου--ερωτήσεις-αυτοελέγχου)

---

## 1. Χάρτης: Επίθεση → Αιτία → Διόρθωση

| Φάση (README) | Τι εκμεταλλεύτηκε ο επιτιθέμενος | CWE | Διόρθωση |
| :--- | :--- | :--- | :--- |
| 2. `robots.txt` | Διαδρομές προς `key-1-of-3.txt` και `fsocity.dic` | CWE-200 | §3 |
| 3. `hydra -L fs-list` | «Invalid username» vs «The password you entered for the username…» | CWE-204 | §4 |
| 3. `hydra -l elliot -P fs-list` | Χιλιάδες προσπάθειες χωρίς περιορισμό | CWE-307 | §5 |
| 3. `elliot:ER28-0652` | Κωδικός που υπήρχε σε λεξικό του ίδιου του site | CWE-521 | §6 |
| 4. Theme Editor → `archive.php` | Τροποποίηση κώδικα PHP από το dashboard | CWE-94 | §7 |
| 4. `nc -lnvp 1234` | Ο server συνδέθηκε προς τον επιτιθέμενο | — | §8 |
| 4. shell ως `daemon` | Ο Apache τρέχει ως γενικός λογαριασμός συστήματος | CWE-250 | §9 |
| 5. `password.raw-md5` | Hash αναγνώσιμο από όλους, MD5 χωρίς salt | CWE-327, CWE-732 | §10 |
| 6. `/usr/local/bin/nmap` SUID | `nmap --interactive` → `!sh` | CWE-250 | §11 |

---

## 2. Απογραφή: πού βρίσκονται τα πράγματα

Η μηχανή Mr. Robot **δεν** χρησιμοποιεί τα πακέτα Apache/PHP/MySQL της διανομής. Χρησιμοποιεί το **Bitnami WordPress Stack**, ένα αυτόνομο πακέτο εγκατεστημένο στο `/opt/bitnami`. Αυτό έχει σημασία:

* Το `apt upgrade` **δεν** αναβαθμίζει τον Apache, την PHP ή τη MySQL του Bitnami.
* Οι ρυθμίσεις **δεν** βρίσκονται στο `/etc/apache2` αλλά στο `/opt/bitnami/...`.

Πάντα επιβεβαιώνουμε τις διαδρομές πριν αλλάξουμε κάτι:

```bash
lsb_release -a                                   # Ubuntu 14.04 (Trusty)
ps -eo user,pid,cmd | grep -E '[h]ttpd|[a]pache|[m]ysqld'

# Πού είναι το WordPress και οι ρυθμίσεις
find / -name wp-config.php 2>/dev/null
find / -name httpd.conf 2>/dev/null
find / -name php.ini 2>/dev/null
ls /opt/bitnami/ 2>/dev/null
```

Οι τυπικές διαδρομές Bitnami (επιβεβαιώστε τις με τις παραπάνω εντολές):

| Στοιχείο | Διαδρομή |
| :--- | :--- |
| Web root WordPress | `/opt/bitnami/apps/wordpress/htdocs/` |
| `wp-config.php` | `/opt/bitnami/apps/wordpress/htdocs/wp-config.php` |
| Ρυθμίσεις Apache | `/opt/bitnami/apache2/conf/httpd.conf` |
| Ρυθμίσεις vhost WordPress | `/opt/bitnami/apps/wordpress/conf/httpd-app.conf` |
| `php.ini` | `/opt/bitnami/php/etc/php.ini` |
| Έλεγχος υπηρεσιών | `/opt/bitnami/ctlscript.sh {start\|stop\|restart} [apache\|mysql]` |

Στη συνέχεια του κειμένου χρησιμοποιούμε τη μεταβλητή:

```bash
export WP=/opt/bitnami/apps/wordpress/htdocs
```

---

## 3. Εξυγίανση του `robots.txt` και του web root

### 3.1 Τι είναι το `robots.txt` (και τι δεν είναι)

Το `robots.txt` είναι μια **ευγενική παράκληση** προς τις μηχανές αναζήτησης: «μην ευρετηριάζετε αυτές τις διαδρομές». **Δεν** είναι μηχανισμός ελέγχου πρόσβασης. Οποιοσδήποτε μπορεί να το διαβάσει, και οι επιτιθέμενοι το διαβάζουν **πρώτο**, γιατί δείχνει ακριβώς τι θέλει να κρύψει ο διαχειριστής.

```bash
cat $WP/robots.txt
# User-agent: *
# fsocity.dic
# key-1-of-3.txt
```

### 3.2 Διόρθωση

**α) Ό,τι δεν πρέπει να είναι δημόσιο, δεν βρίσκεται στο web root.** Μετακινούμε τα αρχεία εκτός:

```bash
mkdir -p /root/private && chmod 700 /root/private
mv $WP/fsocity.dic $WP/key-1-of-3.txt /root/private/
```

**β) Το `robots.txt` περιέχει μόνο ό,τι αφορά μηχανές αναζήτησης:**

```bash
cat > $WP/robots.txt <<'EOF'
User-agent: *
Disallow: /wp-admin/
Allow: /wp-admin/admin-ajax.php
EOF
```

> 💡 Ακόμα και το `Disallow: /wp-admin/` αποκαλύπτει ότι το site είναι WordPress. Αυτό είναι αποδεκτό, γιατί το WordPress αναγνωρίζεται ούτως ή άλλως από δεκάδες άλλα σημεία. Το σημαντικό είναι να μη δείχνει **ευαίσθητα** αρχεία.

**γ) Αναζήτηση για άλλα ευαίσθητα αρχεία στο web root:**

```bash
find $WP -maxdepth 2 -type f \( -name '*.txt' -o -name '*.dic' -o -name '*.bak' \
     -o -name '*.sql' -o -name '*.zip' -o -name '*.tar*' -o -name '*.log' \) -ls
```

Κάθε αποτέλεσμα που δεν είναι μέρος του WordPress (π.χ. `license.txt`, `readme.html` είναι μέρος του, αλλά αποκαλύπτουν έκδοση) αξιολογείται και αφαιρείται.

```bash
rm -f $WP/readme.html $WP/license.txt     # Αποκαλύπτουν την έκδοση του WordPress
```

**δ) Απαγόρευση directory listing** — στο `httpd-app.conf` του WordPress:

```apache
<Directory "/opt/bitnami/apps/wordpress/htdocs">
    Options -Indexes +FollowSymLinks
    AllowOverride All
    Require all granted
</Directory>
```

---

## 4. Αποτροπή απαρίθμησης χρηστών

### 4.1 Το πρόβλημα (CWE-204: Observable Response Discrepancy)

Το WordPress (σε εκείνη την έκδοση) απαντούσε **διαφορετικά** ανάλογα με το λάθος:

| Είσοδος | Μήνυμα |
| :--- | :--- |
| Ανύπαρκτος χρήστης | `ERROR: Invalid username.` |
| Υπαρκτός χρήστης, λάθος κωδικός | `ERROR: The password you entered for the username elliot is incorrect.` |

Αυτή η διαφορά επέτρεψε στο `hydra` να δοκιμάσει πρώτα **μόνο ονόματα** (11.451 δοκιμές) και μετά **μόνο κωδικούς** για τον `elliot` (11.451 δοκιμές). Σύνολο: ~23.000 δοκιμές. Χωρίς τη διαφορά θα χρειαζόταν να δοκιμαστούν όλοι οι **συνδυασμοί**: 11.451 × 11.451 ≈ **131 εκατομμύρια**.

> 📌 Η ενοποίηση μηνυμάτων δεν «κλείνει» μια ευπάθεια, αλλά πολλαπλασιάζει το κόστος της επίθεσης κατά **5.700 φορές**.

### 4.2 Διόρθωση με must-use plugin

Τα **must-use plugins** (`mu-plugins`) φορτώνονται αυτόματα από το WordPress και **δεν** μπορούν να απενεργοποιηθούν από το dashboard. Είναι ο σωστός τόπος για ρυθμίσεις ασφαλείας, γιατί ένας διαχειριστής (ή ένας επιτιθέμενος με λογαριασμό διαχειριστή) δεν τα σβήνει κατά λάθος.

```bash
mkdir -p $WP/wp-content/mu-plugins
cat > $WP/wp-content/mu-plugins/security-hardening.php <<'EOF'
<?php
/*
 * Plugin Name: Security Hardening
 * Description: Ενιαίο μήνυμα σφάλματος σύνδεσης και αποτροπή απαρίθμησης χρηστών.
 */

// 1. Ένα και μοναδικό μήνυμα για κάθε αποτυχημένη σύνδεση
add_filter('login_errors', function () {
    return 'Λανθασμένα στοιχεία σύνδεσης.';
});

// 2. Ενιαίο μήνυμα και στη φόρμα «Ξέχασα τον κωδικό»
add_filter('lostpassword_errors', function ($errors) {
    return new WP_Error('generic', 'Αν ο λογαριασμός υπάρχει, θα σταλεί email.');
});

// 3. Μπλοκάρισμα του /?author=N για επισκέπτες
add_action('template_redirect', function () {
    if (is_author() && !is_user_logged_in()) {
        wp_safe_redirect(home_url(), 301);
        exit;
    }
});

// 4. Απόκρυψη της λίστας χρηστών από το REST API για επισκέπτες
add_filter('rest_endpoints', function ($endpoints) {
    if (!is_user_logged_in()) {
        unset($endpoints['/wp/v2/users'], $endpoints['/wp/v2/users/(?P<id>[\d]+)']);
    }
    return $endpoints;
});
EOF

chown root:daemon $WP/wp-content/mu-plugins/security-hardening.php
chmod 640 $WP/wp-content/mu-plugins/security-hardening.php
```

* `add_filter('login_errors', ...)`: το WordPress περνά το μήνυμα σφάλματος μέσα από αυτό το «φίλτρο» πριν το εμφανίσει. Εμείς το αντικαθιστούμε πάντα με το ίδιο κείμενο.
* Το αρχείο ανήκει στον `root` και ο `daemon` μπορεί μόνο να το **διαβάσει**, άρα δεν μπορεί να το αλλάξει ή να το διαγράψει.

> ℹ️ Το REST API (`/wp-json/`) προστέθηκε στον πυρήνα στο WordPress 4.7. Στην έκδοση 4.3 του Mr. Robot δεν υπάρχει, αλλά ο κανόνας θα χρειαστεί μετά την αναβάθμιση (§12).

### 4.3 Χρόνος απόκρισης

Μια πιο λεπτή μορφή του ίδιου προβλήματος: αν ο έλεγχος κωδικού γίνεται **μόνο** για υπαρκτούς χρήστες, η απάντηση για υπαρκτό χρήστη αργεί λίγα χιλιοστά του δευτερολέπτου περισσότερο (timing attack). Το WordPress το αντιμετωπίζει εσωτερικά στις νεότερες εκδόσεις. Είναι ένας ακόμη λόγος για αναβάθμιση.

---

## 5. Προστασία της σελίδας σύνδεσης από brute-force

Ο `hydra` έστειλε χιλιάδες αιτήματα με `-t 30` (30 παράλληλα). Η εφαρμογή δεν αντέδρασε καθόλου. Εφαρμόζουμε **τρία επίπεδα** περιορισμού.

### 5.1 Επίπεδο 1: Περιορισμός πρόσβασης ανά IP (το ισχυρότερο)

Αν το διαχειριστικό περιβάλλον χρησιμοποιείται μόνο από συγκεκριμένο δίκτυο, **κανείς άλλος** δεν πρέπει να βλέπει καν τη σελίδα σύνδεσης. Στο `httpd-app.conf`:

```apache
<Files "wp-login.php">
    Require ip 192.168.56.0/24
</Files>

<Directory "/opt/bitnami/apps/wordpress/htdocs/wp-admin">
    <Files "admin-ajax.php">
        Require all granted
    </Files>
    <FilesMatch "^(?!admin-ajax\.php$).*$">
        Require ip 192.168.56.0/24
    </FilesMatch>
</Directory>
```

* Οι επισκέπτες εκτός δικτύου παίρνουν `403 Forbidden` πριν φτάσουν στην PHP.
* Το `admin-ajax.php` εξαιρείται, γιατί το χρησιμοποιούν θέματα και plugins και για δημόσιες λειτουργίες.

### 5.2 Επίπεδο 2: Δεύτερος κωδικός HTTP (Basic Auth)

Αν η πρόσβαση πρέπει να είναι από παντού, προσθέτουμε έναν **επιπλέον** κωδικό σε επίπεδο Apache, πριν από τη φόρμα του WordPress:

```bash
/opt/bitnami/apache2/bin/htpasswd -c /opt/bitnami/apache2/conf/.wp-admin-htpasswd adminuser
chmod 640 /opt/bitnami/apache2/conf/.wp-admin-htpasswd
```

```apache
<Files "wp-login.php">
    AuthType Basic
    AuthName "Restricted"
    AuthUserFile "/opt/bitnami/apache2/conf/.wp-admin-htpasswd"
    Require valid-user
</Files>
```

Τώρα το `hydra http-post-form` λαμβάνει `401 Unauthorized` σε **κάθε** αίτημα, ανεξάρτητα από τον κωδικό WordPress που δοκιμάζει.

> ⚠️ Το Basic Auth στέλνει τον κωδικό κωδικοποιημένο σε Base64, όχι κρυπτογραφημένο. **Πρέπει** να συνδυαστεί με HTTPS (§12.3).

### 5.3 Επίπεδο 3: Όριο αποτυχημένων προσπαθειών στην εφαρμογή

Με το WP-CLI εγκαθιστούμε ένα plugin περιορισμού προσπαθειών:

```bash
cd $WP
wp plugin install limit-login-attempts-reloaded --activate --allow-root
```

Ρυθμίσεις (Settings → Limit Login Attempts): **3** επιτρεπόμενες προσπάθειες, κλείδωμα **20 λεπτά**, μετά από **4** κλειδώματα αύξηση σε **24 ώρες**.

Με 3 προσπάθειες ανά 20 λεπτά, οι 11.451 κωδικοί του `fs-list` θα χρειάζονταν πάνω από **50 ημέρες** για έναν μόνο χρήστη.

### 5.4 Απενεργοποίηση του XML-RPC

Το README (§9.2) αναφέρει ότι το `wpscan` μπορεί να χρησιμοποιήσει το **`xmlrpc.php`** με τη μέθοδο `system.multicall`, που δοκιμάζει **εκατοντάδες κωδικούς σε ένα αίτημα**. Αυτό παρακάμπτει τα όρια ανά αίτημα. Αν δεν χρησιμοποιείται (π.χ. από την εφαρμογή κινητού του WordPress ή το Jetpack), το απενεργοποιούμε:

```apache
# httpd-app.conf (σύνταξη Apache 2.4)
<Files "xmlrpc.php">
    Require all denied
</Files>
```

και στο mu-plugin της §4.2:

```php
add_filter('xmlrpc_enabled', '__return_false');
```

> 💡 Το README (§10.2) δίνει τη σύνταξη `Order Allow,Deny / Deny from all`. Αυτή είναι η σύνταξη του **Apache 2.2**. Ο Apache 2.4 τη δέχεται μόνο αν είναι ενεργό το `mod_access_compat`. Η σύνταξη `Require all denied` είναι η σωστή για 2.4.

### 5.5 Εφαρμογή αλλαγών

```bash
/opt/bitnami/apache2/bin/apachectl -t           # Έλεγχος σύνταξης → Syntax OK
/opt/bitnami/ctlscript.sh restart apache
```

---

## 6. Κωδικοί και έλεγχος ταυτότητας δύο παραγόντων

### 6.1 Γιατί έπεσε ο κωδικός

Ο κωδικός `ER28-0652` υπήρχε **μέσα** στο `fsocity.dic`, ένα λεξικό που φιλοξενούσε το ίδιο το site. Γενικότερα: κάθε κωδικός που μπορεί να βρεθεί σε λεξικό, σε διαρροή ή να συναχθεί από πληροφορίες του οργανισμού (ονόματα, ημερομηνίες, κωδικοί υπαλλήλων) θεωρείται αδύναμος.

### 6.2 Αλλαγή κωδικών και τερματισμός συνεδριών

```bash
cd $WP
wp user list --role=administrator --fields=ID,user_login,user_email --allow-root

# Νέος τυχαίος κωδικός (32 χαρακτήρες)
NEWPASS=$(openssl rand -base64 24)
wp user update elliot --user_pass="$NEWPASS" --allow-root
echo "$NEWPASS"     # Αποθηκεύστε τον σε password manager

# Αποσύνδεση όλων των υπαρχουσών συνεδριών (και του επιτιθέμενου)
wp user session destroy --all --allow-root 2>/dev/null || wp config shuffle-salts --allow-root
```

Η εντολή `wp user session destroy` υπάρχει σε νεότερες εκδόσεις του WP-CLI. Η αλλαγή των **salts** στο `wp-config.php` πετυχαίνει το ίδιο: όλα τα cookies σύνδεσης ακυρώνονται.

### 6.3 Ελάχιστα δικαιώματα για τους λογαριασμούς WordPress

```bash
wp user list --fields=user_login,roles --allow-root
```

Ένας λογαριασμός που γράφει μόνο άρθρα χρειάζεται ρόλο **Editor** ή **Author**, όχι **Administrator**. Μόνο ο Administrator έχει πρόσβαση στον Theme Editor (§7). Λιγότεροι διαχειριστές = λιγότεροι λογαριασμοί που, αν παραβιαστούν, δίνουν RCE.

### 6.4 Έλεγχος ταυτότητας δύο παραγόντων (2FA)

Ακόμη κι αν ο επιτιθέμενος βρει τον κωδικό, χωρίς τον **δεύτερο παράγοντα** (κωδικός μιας χρήσης από εφαρμογή στο κινητό, TOTP) δεν μπορεί να συνδεθεί.

```bash
wp plugin install two-factor --activate --allow-root
```

Μετά, κάθε διαχειριστής ενεργοποιεί το 2FA από το προφίλ του (Users → Profile → Two-Factor Options → Authenticator App) σκανάροντας τον κωδικό QR.

---

## 7. Απενεργοποίηση του Theme Editor και κλείδωμα αρχείων

### 7.1 Το πρόβλημα (CWE-94: Code Injection)

Ο **Theme Editor** (`Appearance → Editor`) επιτρέπει σε κάθε Administrator να **γράψει κώδικα PHP** που εκτελείται στον server. Από την οπτική της ασφάλειας: **«κωδικός διαχειριστή WordPress» = «εκτέλεση κώδικα στον server»**.

Για να δουλέψει ο Editor χρειάζονται δύο πράγματα:

1. Η λειτουργία να είναι ενεργή στο WordPress.
2. Ο χρήστης του web server (`daemon`) να μπορεί να **γράψει** στα αρχεία του θέματος.

Διορθώνουμε και τα δύο.

### 7.2 Απενεργοποίηση στο `wp-config.php`

Πριν από τη γραμμή `/* That's all, stop editing! */`:

```php
define('DISALLOW_FILE_EDIT', true);   // Αφαιρεί τον Theme/Plugin Editor από το μενού
define('DISALLOW_FILE_MODS', true);   // Απαγορεύει ΚΑΙ εγκατάσταση/ενημέρωση plugins/themes από το dashboard
```

| Ρύθμιση | Τι αποτρέπει |
| :--- | :--- |
| `DISALLOW_FILE_EDIT` | Επεξεργασία PHP αρχείων από το dashboard (η επίθεση του README) |
| `DISALLOW_FILE_MODS` | Ανέβασμα ενός «plugin» που είναι στην πραγματικότητα web shell (εναλλακτική διαδρομή RCE) |

> ⚠️ Με το `DISALLOW_FILE_MODS`, οι ενημερώσεις plugins/themes **δεν** γίνονται από το dashboard. Γίνονται από τον διαχειριστή μέσω **WP-CLI** (`wp plugin update --all`). Αυτό είναι επιθυμητό: οι αλλαγές κώδικα γίνονται από το τερματικό, όχι από τον browser.

### 7.3 Κλείδωμα αρχείων: ο web server διαβάζει, δεν γράφει

```bash
cd $WP
chown -R root:daemon .
find . -type d -exec chmod 750 {} \;
find . -type f -exec chmod 640 {} \;

# Μόνο τα uploads είναι εγγράψιμα από τον web server
chown -R daemon:daemon wp-content/uploads

# Το wp-config.php: μόνο ανάγνωση από τον web server
chmod 440 wp-config.php
```

Τώρα, ακόμη κι αν το `DISALLOW_FILE_EDIT` αφαιρεθεί (π.χ. από επιτιθέμενο με πρόσβαση στη βάση που αλλάζει ρυθμίσεις), η εγγραφή στο `archive.php` αποτυγχάνει: ο `daemon` δεν έχει δικαίωμα εγγραφής.

### 7.4 Καμία εκτέλεση PHP στα uploads

```apache
# httpd-app.conf
<Directory "/opt/bitnami/apps/wordpress/htdocs/wp-content/uploads">
    <FilesMatch "\.(php|phtml|php\d|phar)$">
        Require all denied
    </FilesMatch>
</Directory>
```

### 7.5 Έλεγχος ακεραιότητας αρχείων WordPress

Μετά από παραβίαση, ελέγχουμε αν κάποιο αρχείο του πυρήνα έχει τροποποιηθεί:

```bash
wp core verify-checksums --allow-root
```

Για τα θέματα, που **δεν** καλύπτονται από τα checksums του πυρήνα, τα επανεγκαθιστούμε από καθαρή πηγή:

```bash
grep -l 'fsockopen\|proc_open' $WP/wp-content/themes/*/*.php     # Ίχνη reverse shell
wp theme install twentyfifteen --force --allow-root               # Επανεγκατάσταση (πριν το DISALLOW_FILE_MODS)
```

---

## 8. PHP και εξερχόμενες συνδέσεις

### 8.1 Πώς λειτουργεί το `php-reverse-shell`

Το script του pentestmonkey κάνει δύο πράγματα:

1. `fsockopen($ip, $port)`: ανοίγει σύνδεση TCP **προς** τον επιτιθέμενο.
2. `proc_open('/bin/sh -i', ...)`: εκκινεί ένα κέλυφος και συνδέει την είσοδο/έξοδό του με τη σύνδεση.

Μπορούμε να εμποδίσουμε **και τα δύο**.

### 8.2 Απενεργοποίηση συναρτήσεων εκτέλεσης στην PHP

Στο `/opt/bitnami/php/etc/php.ini`:

```ini
disable_functions = exec,passthru,shell_exec,system,proc_open,popen,pcntl_exec,show_source
expose_php        = Off
display_errors    = Off
allow_url_include = Off
open_basedir      = /opt/bitnami/apps/wordpress/htdocs:/opt/bitnami/apps/wordpress/tmp:/tmp
```

* Χωρίς `proc_open`, το `php-reverse-shell` αποτυγχάνει στο βήμα 2.
* Το WordPress core **δεν** χρειάζεται καμία από αυτές τις συναρτήσεις. Ορισμένα plugins μπορεί να χρειάζονται. Δοκιμάστε το site μετά την αλλαγή.
* Η `fsockopen` **δεν** απενεργοποιείται, γιατί το WordPress τη χρησιμοποιεί ως εναλλακτικό τρόπο για αιτήματα HTTP. Το βήμα 1 το αντιμετωπίζουμε στο επίπεδο δικτύου (§8.3).

```bash
/opt/bitnami/ctlscript.sh restart apache
```

### 8.3 Φιλτράρισμα εξερχόμενων συνδέσεων

Ο web server έχει λόγο να ξεκινά εξερχόμενες συνδέσεις μόνο για συγκεκριμένους σκοπούς (DNS, HTTPS για ενημερώσεις). **Ποτέ** προς τη θύρα 1234 ενός τυχαίου υπολογιστή.

```bash
iptables -A OUTPUT -o lo -j ACCEPT
iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
# Ο web server επιτρέπεται να κάνει DNS και HTTPS (π.χ. ενημερώσεις WordPress)
iptables -A OUTPUT -m owner --uid-owner daemon -p udp --dport 53  -j ACCEPT
iptables -A OUTPUT -m owner --uid-owner daemon -p tcp --dport 443 -j ACCEPT
# Οτιδήποτε άλλο νέο από τον web server: καταγραφή και απόρριψη
iptables -A OUTPUT -m owner --uid-owner daemon -m state --state NEW -j LOG --log-prefix "WEB-EGRESS: "
iptables -A OUTPUT -m owner --uid-owner daemon -m state --state NEW -j REJECT

apt-get install iptables-persistent
iptables-save > /etc/iptables/rules.v4
```

Η αναλυτική επεξήγηση κάθε παραμέτρου βρίσκεται στο [DC-1 §8](../03.%20DC-CTF/Hardening-Guide.md#8-περιορισμός-εξερχόμενων-συνδέσεων).

> 💡 Ένας πιο αυστηρός περιορισμός: να επιτρέπεται το HTTPS **μόνο** προς τις IP του `api.wordpress.org` και `downloads.wordpress.org`. Επειδή οι IP αλλάζουν, συνήθως χρησιμοποιείται ένας ενδιάμεσος proxy με λίστα επιτρεπόμενων ονομάτων.

---

## 9. Ο λογαριασμός υπηρεσίας `daemon`

### 9.1 Το πρόβλημα

Ο `daemon` είναι ένας **γενικός** λογαριασμός συστήματος, που υπάρχει σε κάθε Linux για ιστορικούς λόγους. Αν χρησιμοποιείται από πολλές υπηρεσίες, η παραβίαση της μίας δίνει πρόσβαση στα αρχεία όλων. Η σωστή πρακτική είναι: **κάθε υπηρεσία, δικός της χρήστης**.

### 9.2 Διόρθωση

```bash
# Ποια αρχεία/διεργασίες ανήκουν σήμερα στον daemon;
ps -u daemon -o pid,cmd
find / -xdev -user daemon -not -path '/proc/*' 2>/dev/null | head -50

# Ειδικός χρήστης για τον web server, χωρίς shell και χωρίς home
useradd --system --no-create-home --shell /usr/sbin/nologin wpweb
```

Στο `/opt/bitnami/apache2/conf/httpd.conf`:

```apache
User  wpweb
Group wpweb
```

Και ενημερώνουμε τα δικαιώματα της §7.3 με `wpweb` στη θέση του `daemon`:

```bash
chown -R root:wpweb $WP
chown -R wpweb:wpweb $WP/wp-content/uploads
/opt/bitnami/ctlscript.sh restart apache
ps -eo user,cmd | grep [h]ttpd
```

* `--system`: λογαριασμός συστήματος (UID < 1000), χωρίς λήξη κωδικού.
* `--shell /usr/sbin/nologin`: αν κάποιος προσπαθήσει να συνδεθεί ως `wpweb` (π.χ. με `su`), λαμβάνει «This account is currently not available».

Επιπλέον αφαιρούμε το shell από τον `daemon`, αν έχει:

```bash
grep '^daemon' /etc/passwd
usermod -s /usr/sbin/nologin daemon
```

---

## 10. Κατάλογοι χρηστών και αποθήκευση κωδικών

### 10.1 Το πρόβλημα (CWE-732 και CWE-327)

```text
drwxr-xr-x  robot robot  /home/robot
-rw-r--r--  robot robot  /home/robot/password.raw-md5
```

Δύο λάθη ταυτόχρονα:

1. **Δικαιώματα:** ο φάκελος (`755`) και το αρχείο (`644`) ήταν αναγνώσιμα από **κάθε** χρήστη, ακόμη και από τον `daemon`.
2. **Αποθήκευση:** ο κωδικός ήταν σε **MD5 χωρίς salt**. Το MD5 υπολογίζεται δισεκατομμύρια φορές το δευτερόλεπτο σε μια κάρτα γραφικών, και τα hashes απλών κωδικών υπάρχουν ήδη σε δημόσιες βάσεις (CrackStation). Ο κωδικός `abcdefghijklmnopqrstuvwxyz` βρέθηκε αμέσως.

### 10.2 Διόρθωση δικαιωμάτων

```bash
chmod 700 /home/robot
ls -ld /home/*                 # Έλεγχος για όλους τους χρήστες
```

Για να έχουν οι **νέοι** χρήστες ιδιωτικό φάκελο αυτόματα, στο `/etc/adduser.conf`:

```text
DIR_MODE=0700
```

και στο `/etc/login.defs`:

```text
UMASK 077
```

Το **umask** καθορίζει ποια δικαιώματα **αφαιρούνται** από νέα αρχεία. Με `077`, τα νέα αρχεία δημιουργούνται ως `600` και οι φάκελοι ως `700`, δηλαδή προσβάσιμα μόνο από τον ιδιοκτήτη.

### 10.3 Ποτέ κωδικοί ή hashes σε αρχεία κειμένου

Το `password.raw-md5` **δεν πρέπει να υπάρχει**. Οι κωδικοί των χρηστών Linux αποθηκεύονται **μόνο** στο `/etc/shadow`, που είναι αναγνώσιμο μόνο από τον root:

```bash
shred -u /home/robot/password.raw-md5     # Ασφαλής διαγραφή (αντικατάσταση περιεχομένου πριν τη διαγραφή)
ls -l /etc/shadow                          # → -rw-r----- root shadow
```

### 10.4 Νέος κωδικός με σύγχρονο αλγόριθμο

```bash
grep ENCRYPT_METHOD /etc/login.defs        # Πρέπει: SHA512 (ή YESCRYPT σε νεότερα συστήματα)
passwd robot
awk -F: '$1=="robot"{print substr($2,1,3)}' /etc/shadow    # → $6$ (SHA-512 με salt)
```

| Αλγόριθμος | Salt | Ταχύτητα σε GPU (ενδεικτικά) | Κατάλληλος; |
| :--- | :---: | :--- | :---: |
| MD5 (raw) | ✗ | δεκάδες δισεκατομμύρια/δευτ. | ✗ |
| SHA-512-crypt (`$6$`) | ✓ | εκατοντάδες χιλιάδες/δευτ. | ✓ |
| yescrypt (`$y$`) / bcrypt / Argon2id | ✓ | χιλιάδες/δευτ. (σκόπιμα αργός) | ✓✓ |

Η διαφορά μεταξύ MD5 και yescrypt είναι **εκατομμύρια φορές** στον χρόνο που χρειάζεται ο επιτιθέμενος.

---

## 11. SUID στο nmap

### 11.1 Γιατί υπήρχε και γιατί είναι λάθος

Το nmap χρειάζεται **raw sockets** για ορισμένους τύπους σάρωσης (π.χ. SYN scan, `-sS`), και τα raw sockets απαιτούν προνόμια root. Ένας διαχειριστής που ήθελε οι χρήστες να τρέχουν SYN scans έβαλε SUID root στο nmap. Η έκδοση 3.81 όμως είχε **interactive mode** (`--interactive`) με εντολή `!sh`, που ανοίγει shell **με τα δικαιώματα της διεργασίας**, δηλαδή root.

Η τεχνική της κατάχρησης SUID αναλύεται στο [DC-1 §9](../03.%20DC-CTF/Hardening-Guide.md#9-suid-κατανόηση-έλεγχος-διόρθωση).

### 11.2 Διόρθωση

```bash
ls -l /usr/local/bin/nmap         # -rwsr-xr-x root root
/usr/local/bin/nmap --version     # 3.81

# 1. Αφαίρεση SUID
chmod u-s /usr/local/bin/nmap

# 2. Το εργαλείο είναι εγκατεστημένο χειροκίνητα (/usr/local) και πολύ παλιό: αφαίρεση
rm -f /usr/local/bin/nmap
rm -rf /usr/local/share/nmap
```

Το nmap στο `/usr/local/bin` **δεν** διαχειρίζεται από το `apt`, άρα δεν ενημερωνόταν ποτέ. Λογισμικό εγκατεστημένο εκτός διαχειριστή πακέτων είναι συχνή πηγή ξεχασμένων, ευάλωτων εκδόσεων.

### 11.3 Αν ένας χρήστης χρειάζεται πράγματι nmap

**Επιλογή Α — capabilities αντί για SUID:**

```bash
apt-get install nmap                         # Έκδοση της διανομής, χωρίς interactive mode
setcap cap_net_raw,cap_net_admin,cap_net_bind_service+eip /usr/bin/nmap
getcap /usr/bin/nmap

# Περιορισμός: μόνο τα μέλη της ομάδας netscan μπορούν να το εκτελέσουν
groupadd netscan
chgrp netscan /usr/bin/nmap
chmod 750 /usr/bin/nmap
usermod -aG netscan robot
```

Ο χρήστης εκτελεί `nmap --privileged -sS <target>`. Η διεργασία έχει **μόνο** τις δυνατότητες δικτύου, όχι πλήρες root. Ένα `!sh` (αν υπήρχε) θα έδινε shell **ως `robot`**.

> ⚠️ Ακόμα και με capabilities, το nmap μπορεί να εκτελέσει **σενάρια Lua** (NSE, `--script`). Με `cap_net_admin` ένας χρήστης μπορεί να αλλάξει ρυθμίσεις δικτύου. Δίνετε το εργαλείο **μόνο** σε χρήστες που το χρειάζονται πραγματικά.

**Επιλογή Β — περιορισμένος κανόνας sudo:**

```text
# visudo -f /etc/sudoers.d/netscan
robot ALL=(root) /usr/bin/nmap -sS -p 1-1024 192.168.56.0/24
```

Ο χρήστης μπορεί να τρέξει **μόνο** τη συγκεκριμένη εντολή με τις συγκεκριμένες παραμέτρους.

### 11.4 Πλήρης έλεγχος SUID

```bash
find / -xdev \( -perm -4000 -o -perm -2000 \) -type f -exec ls -ld {} \; 2>/dev/null
```

Συγκρίνετε με τη λίστα αναμενόμενων SUID του [DC-1 §9.2](../03.%20DC-CTF/Hardening-Guide.md#92-έλεγχος-απογραφή-όλων-των-suidsgid) και δημιουργήστε γραμμή βάσης με καθημερινό έλεγχο όπως στο [DC-1 §9.5](../03.%20DC-CTF/Hardening-Guide.md#95-περιοδικός-έλεγχος-με-γραμμή-βάσης-baseline).

---

## 12. Αναβάθμιση λειτουργικού, WordPress, HTTPS

### 12.1 Λειτουργικό σύστημα

Η μηχανή τρέχει **Ubuntu 14.04 (Trusty)**, που έληξε στην κανονική υποστήριξη το 2019. Οι επιλογές:

| Επιλογή | Περιγραφή |
| :--- | :--- |
| **Ubuntu Pro / ESM** | Επεκτεταμένες ενημερώσεις ασφαλείας για παλιές εκδόσεις (`pro attach <token>`), δωρεάν για προσωπική χρήση έως 5 μηχανές |
| **Σταδιακή αναβάθμιση** | `do-release-upgrade` 14.04 → 16.04 → 18.04 → 20.04 → 22.04 → 24.04, ένα LTS κάθε φορά |
| **Νέα εγκατάσταση** (συνιστάται) | Ubuntu 24.04 LTS, μεταφορά αρχείων WordPress και dump βάσης |

```bash
# Ενημέρωση πακέτων της διανομής (ΔΕΝ αγγίζει το Bitnami)
apt-get update && apt-get upgrade
```

### 12.2 Το Bitnami stack και το WordPress

Επειδή ο Apache, η PHP και η MySQL είναι μέσα στο `/opt/bitnami`, **δεν** αναβαθμίζονται με το `apt`. Το Bitnami stack αναβαθμίζεται μόνο με **νέα εγκατάσταση** νεότερου stack και μεταφορά δεδομένων. Σε νέο σύστημα, προτιμήστε τα πακέτα της διανομής (`apt install apache2 php mariadb-server`) ώστε να λαμβάνουν αυτόματες ενημερώσεις ασφαλείας.

Για το WordPress (4.3.1 στο Mr. Robot):

```bash
cd $WP
wp db export /root/wp-backup.sql --allow-root
wp core update --allow-root
wp core update-db --allow-root
wp plugin update --all --allow-root
wp theme update --all --allow-root
```

> ⚠️ Το WordPress 6.x απαιτεί PHP ≥ 7.2.24. Η PHP του παλιού Bitnami stack μπορεί να μην επαρκεί, οπότε ο πυρήνας του WordPress θα αναβαθμιστεί μέχρι την τελευταία συμβατή έκδοση. Άλλος ένας λόγος για νέα εγκατάσταση.

### 12.3 HTTPS με έγκυρο πιστοποιητικό

Το site απαντούσε στη θύρα 443, αλλά με αυτο-υπογεγραμμένο πιστοποιητικό. Σε δημόσιο site χρησιμοποιούμε **Let's Encrypt**:

```bash
apt-get install certbot
certbot certonly --webroot -w $WP -d example.com
```

Και ανακατεύθυνση από HTTP σε HTTPS, ώστε οι κωδικοί (και του Basic Auth της §5.2) να μη μεταφέρονται ποτέ καθαρό κείμενο:

```apache
<VirtualHost *:80>
    ServerName example.com
    Redirect permanent / https://example.com/
</VirtualHost>
```

```php
// wp-config.php
define('FORCE_SSL_ADMIN', true);
```

### 12.4 Firewall

Το Mr. Robot εξέθετε μόνο 80/443 και είχε το 22 κλειστό. Αυτό είναι **σωστό**. Το επιβεβαιώνουμε με ρητό κανόνα, ώστε μια μελλοντική εγκατάσταση SSH να μην ανοίξει αυτόματα τη θύρα:

```bash
ufw default deny incoming
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow from 192.168.56.0/24 to any port 22 proto tcp     # Μόνο αν χρειάζεται SSH
ufw enable
```

> ⚠️ Το `ufw` και οι κανόνες `iptables` της §8.3 πρέπει να συνδυαστούν σωστά. Σε συστήματα με `ufw`, οι κανόνες εξόδου ανά χρήστη μπαίνουν στο `/etc/ufw/before.rules`.

---

## 13. Επαλήθευση

Από το **Kali**, επαναλαμβάνουμε την επίθεση βήμα προς βήμα:

```bash
# 1. robots.txt καθαρό, κανένα ευαίσθητο αρχείο
curl -s http://<MR_ROBOT_IP>/robots.txt
curl -I http://<MR_ROBOT_IP>/fsocity.dic                     # → 404
curl -I http://<MR_ROBOT_IP>/key-1-of-3.txt                  # → 404

# 2. Ίδιο μήνυμα για ανύπαρκτο και υπαρκτό χρήστη
curl -s -d 'log=noone&pwd=x'  http://<MR_ROBOT_IP>/wp-login.php | grep -o 'Λανθασμένα στοιχεία σύνδεσης'
curl -s -d 'log=elliot&pwd=x' http://<MR_ROBOT_IP>/wp-login.php | grep -o 'Λανθασμένα στοιχεία σύνδεσης'

# 3. Το hydra δεν φτάνει καν στη φόρμα (403/401) ή κλειδώνεται μετά από 3 δοκιμές
hydra -l elliot -P fs-list <MR_ROBOT_IP> http-post-form \
  "/wp-login.php:log=^USER^&pwd=^PASS^:F=Λανθασμένα" -t 30

# 4. XML-RPC κλειστό
curl -I http://<MR_ROBOT_IP>/xmlrpc.php                      # → 403
```

Στη **μηχανή Mr. Robot**:

```bash
# 5. Ο Theme Editor δεν υπάρχει και ο web server δεν γράφει στα θέματα
grep DISALLOW_FILE $WP/wp-config.php
su -s /bin/sh wpweb -c "touch $WP/wp-content/themes/twentyfifteen/archive.php"   # → Permission denied

# 6. Η PHP δεν εκτελεί εντολές
echo '<?php proc_open("id", [], $p);' > $WP/wp-content/uploads/t.php
curl -s http://localhost/wp-content/uploads/t.php            # → 403 (καμία εκτέλεση PHP στα uploads)
rm $WP/wp-content/uploads/t.php

# 7. Ο web server δεν ανοίγει εξερχόμενες συνδέσεις
su -s /bin/sh wpweb -c 'nc -w 3 <KALI_IP> 1234' ; dmesg | grep WEB-EGRESS | tail -n 2

# 8. Ο φάκελος του robot δεν είναι αναγνώσιμος
su -s /bin/sh wpweb -c 'ls /home/robot'                      # → Permission denied

# 9. Κανένα SUID nmap
ls -l /usr/local/bin/nmap 2>&1                              # → No such file
find / -xdev -perm -4000 -type f 2>/dev/null
```

---

## 14. Λίστα ελέγχου & ερωτήσεις αυτοελέγχου

### 14.1 Λίστα ελέγχου

- [ ] Snapshot ή αρχείο με τις εντολές θωράκισης
- [ ] Επιβεβαίωση διαδρομών Bitnami (`find`)
- [ ] `fsocity.dic`, `key-1-of-3.txt`, `readme.html`, `license.txt` εκτός web root
- [ ] `robots.txt` χωρίς ευαίσθητες διαδρομές, `Options -Indexes`
- [ ] mu-plugin: ενιαίο μήνυμα σφάλματος, μπλοκάρισμα `?author=` και REST users
- [ ] `wp-login.php`/`wp-admin` περιορισμένα σε IP ή με Basic Auth
- [ ] Plugin περιορισμού προσπαθειών σύνδεσης, XML-RPC απενεργοποιημένο
- [ ] Νέος τυχαίος κωδικός για τον `elliot`, νέα salts, 2FA για διαχειριστές
- [ ] `DISALLOW_FILE_EDIT` και `DISALLOW_FILE_MODS`
- [ ] Αρχεία WordPress σε `root:<web user>`, `750/640`, εγγράψιμα μόνο τα uploads
- [ ] Καμία εκτέλεση PHP στα uploads, `wp core verify-checksums` καθαρό, θέμα επανεγκατεστημένο
- [ ] `php.ini`: `disable_functions` (με `proc_open`), `open_basedir`, `expose_php Off`
- [ ] Egress filtering για τον χρήστη του web server
- [ ] Ειδικός χρήστης `wpweb` αντί για `daemon`
- [ ] `/home/robot` σε `700`, `password.raw-md5` διαγραμμένο, νέος κωδικός (`$6$` ή `$y$`), `UMASK 077`
- [ ] nmap 3.81 αφαιρέθηκε, κανένα μη αναμενόμενο SUID, γραμμή βάσης SUID
- [ ] Σχέδιο αναβάθμισης (Ubuntu Pro ή νέα εγκατάσταση), WordPress ενημερωμένο, HTTPS
- [ ] Επαλήθευση όλων των βημάτων της §13

### 14.2 Ερωτήσεις αυτοελέγχου

1. Γιατί το `robots.txt` **δεν** είναι μηχανισμός ασφαλείας; Ποιος είναι ο σωστός τρόπος να προστατεύσουμε ένα αρχείο που δεν πρέπει να είναι δημόσιο;
2. Υπολογίστε πόσες δοκιμές θα χρειαζόταν το `hydra` **με** και **χωρίς** τη διαφορά στα μηνύματα σφάλματος, για λεξικό 11.451 λέξεων. Τι δείχνει αυτό για την αξία της §4;
3. Το `DISALLOW_FILE_EDIT` απενεργοποιεί τον Theme Editor. Γιατί χρειαζόμαστε **επιπλέον** τα δικαιώματα αρχείων της §7.3;
4. Το `php-reverse-shell` χρησιμοποιεί `fsockopen` και `proc_open`. Γιατί απενεργοποιούμε μόνο τη δεύτερη στο `php.ini` και πώς αντιμετωπίζουμε την πρώτη;
5. Ποια είναι η διαφορά ανάμεσα στο MD5 χωρίς salt και στο SHA-512-crypt; Γιατί το salt κάνει άχρηστους τους προϋπολογισμένους πίνακες (rainbow tables);
6. Εξηγήστε γιατί το nmap με `setcap cap_net_raw+eip` είναι ασφαλέστερο από το nmap με SUID root. Υπάρχει παρ' όλα αυτά κάποιος κίνδυνος;
7. Γιατί το λογισμικό που εγκαθίσταται στο `/usr/local` ή στο `/opt` χρειάζεται ιδιαίτερη προσοχή όσον αφορά τις ενημερώσεις;

---

## Πηγές

- [WordPress — Hardening WordPress](https://developer.wordpress.org/advanced-administration/security/hardening/)
- [WordPress — Editing wp-config.php: DISALLOW_FILE_EDIT / DISALLOW_FILE_MODS](https://developer.wordpress.org/advanced-administration/wordpress/wp-config/#disable-the-plugin-and-theme-file-editor)
- [WordPress — Must Use Plugins](https://developer.wordpress.org/advanced-administration/plugins/mu-plugins/)
- [WP-CLI — Commands](https://developer.wordpress.org/cli/commands/)
- [MITRE CWE-204 — Observable Response Discrepancy](https://cwe.mitre.org/data/definitions/204.html)
- [MITRE CWE-307 — Improper Restriction of Excessive Authentication Attempts](https://cwe.mitre.org/data/definitions/307.html)
- [OWASP — Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [OWASP — Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- [GTFOBins — nmap](https://gtfobins.github.io/gtfobins/nmap/)
- [Nmap — Running as non-root with capabilities](https://secwiki.org/w/Running_nmap_as_an_unprivileged_user)
- [Ubuntu — Ubuntu Pro / ESM](https://ubuntu.com/pro)
