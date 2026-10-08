# Δημιουργία Εξουσιοδοτημένου Εργαστηρίου WordPress

> **Μόνο για εκπαιδευτική και εξουσιοδοτημένη χρήση.** Δημιουργήστε το περιβάλλον σε τοπική εικονική μηχανή, σε απομονωμένο host-only δίκτυο ή σε προσωρινό project containers. Μην εκθέτετε σκόπιμα ευάλωτο λογισμικό στο δημόσιο Internet και μην επαναχρησιμοποιείτε τους εργαστηριακούς κωδικούς σε παραγωγικά συστήματα.

## Στόχοι μάθησης

Ο οδηγός παρουσιάζει τρεις ελεγχόμενες μεθόδους δημιουργίας WordPress εργαστηρίου: κλασική εγκατάσταση Linux με Apache, MariaDB και PHP, εγκατάσταση με Docker Compose και τοπική εγκατάσταση Windows με web stack. Κάθε εντολή συνοδεύεται από αντιπροσωπευτική έξοδο και εξήγηση της σημασίας της.

Οι τιμές `wordpress`, `wp_user` και οι κωδικοί είναι παραδείγματα για disposable lab. Δεν πρέπει να χρησιμοποιηθούν σε παραγωγή. Τα διαγράμματα είναι ASCII ώστε να παραμένουν φορητά και ανεξάρτητα από εξωτερικές εικόνες.

## Αρχιτεκτονική

```text
+----------------------+       HTTP        +------------------------+
| Browser              | ----------------> | Apache / WordPress     |
| localhost            |                   | PHP εφαρμογή           |
+----------------------+                   +-----------+------------+
                                                       |
                                                       | SQL
                                                       v
                                           +------------------------+
                                           | MariaDB / MySQL        |
                                           | wordpress database     |
                                           +------------------------+
```

Ο browser ζητά σελίδες από τον web server. Η PHP εκτελεί την εφαρμογή, ενώ η MariaDB ή MySQL αποθηκεύει άρθρα, λογαριασμούς, ρυθμίσεις και metadata. Ο διαχωρισμός βοηθά στην αντιμετώπιση προβλημάτων: HTTP σφάλματα συνδέονται συνήθως με web server ή PHP, ενώ database errors με credentials, hostname, port ή permissions.

## Ασφάλεια και προαπαιτούμενα

Χρησιμοποιήστε disposable VM ή ιδιωτικό training network και πάρτε snapshot πριν από αλλαγές. Μην εγκαθιστάτε ευάλωτα plugins σε production server και μην κάνετε port forwarding από router ή public cloud firewall προς το εργαστήριο.

```bash
cat /etc/os-release
id
```

Ενδεικτική έξοδος:

```text
PRETTY_NAME="Ubuntu 22.04.5 LTS"
uid=1000(student) gid=1000(student) groups=1000(student),27(sudo)
```

Η πρώτη εντολή εμφανίζει διανομή και έκδοση λειτουργικού. Η `id` εμφανίζει χρήστη, αριθμητικά IDs και groups. Η συμμετοχή στο group `sudo` σημαίνει ότι ο χρήστης μπορεί να ζητήσει administrative privileges σύμφωνα με την πολιτική του συστήματος.

## Μέθοδος Α: Εγκατάσταση Linux

### Εγκατάσταση Apache

```bash
sudo apt update
sudo apt install apache2
```

Ενδεικτική έξοδος:

```text
Reading package lists... Done
Building dependency tree... Done
The following NEW packages will be installed:
  apache2 apache2-bin apache2-data
Setting up apache2 ...
```

Η `apt update` ανανεώνει το τοπικό ευρετήριο πακέτων. Η `apt install apache2` εγκαθιστά τον web server και τα βοηθητικά πακέτα. Η ακριβής έξοδος εξαρτάται από τη διανομή και την έκδοση.

```bash
sudo systemctl enable --now apache2
sudo systemctl status apache2 --no-pager
```

Παράδειγμα εξόδου:

```text
● apache2.service - The Apache HTTP Server
     Loaded: loaded (...; enabled)
     Active: active (running)
```

Η `enable --now` ρυθμίζει εκκίνηση στο boot και ξεκινά άμεσα την υπηρεσία. Το `active (running)` δείχνει ότι ο Apache λειτουργεί. Αν αποτύχει, εξετάστε logs αντί να κάνετε επανειλημμένα restart.

```bash
curl -I http://127.0.0.1/
```

Ενδεικτική έξοδος:

```text
HTTP/1.1 200 OK
Server: Apache/2.4.x
Content-Type: text/html; charset=UTF-8
```

Η `200 OK` επιβεβαιώνει ότι ο Apache επέστρεψε απάντηση. Η κεφαλίδα `Server` παρέχει πληροφορία λογισμικού, αλλά δεν αποτελεί πλήρη αξιολόγηση ασφάλειας.

### Εγκατάσταση MariaDB

```bash
sudo apt install mariadb-server mariadb-client
sudo systemctl enable --now mariadb
```

Ενδεικτική έξοδος:

```text
The following NEW packages will be installed:
  mariadb-client mariadb-server
Created symlink ... mariadb.service ...
```

Το server package παρέχει τον database daemon και το client package εργαλεία όπως `mariadb` ή `mysql`. Η ενεργοποίηση και εκκίνηση καθιστά τη βάση διαθέσιμη τοπικά.

```bash
sudo mariadb-secure-installation
```

Σε ορισμένες εκδόσεις η εντολή ονομάζεται `mysql_secure_installation`. Τυπική ακολουθία:

```text
Enter current password for root (enter for none):  [press Enter]
Set root password? [Y/n]: Y
Remove anonymous users? [Y/n]: Y
Disallow root login remotely? [Y/n]: Y
Remove test database and access to it? [Y/n]: Y
Reload privilege tables now? [Y/n]: Y
```

Οι επιλογές αφαιρούν anonymous users, αποτρέπουν απομακρυσμένο database-root login, αφαιρούν τη test database και επαναφορτώνουν τα privilege tables. Χρησιμοποιήστε ισχυρό administrator password. Σε νεότερες MariaDB εγκαταστάσεις, το root μπορεί να χρησιμοποιεί socket authentication, οπότε τα prompts διαφέρουν.

### Εγκατάσταση PHP

```bash
sudo apt install php php-mysql libapache2-mod-php
sudo systemctl restart apache2
```

Ενδεικτική έξοδος:

```text
Setting up php ...
Setting up libapache2-mod-php ...
Processing triggers for libapache2-mod-php ...
```

Η PHP είναι ο interpreter, το `php-mysql` παρέχει database connectivity και το Apache module επιτρέπει στον Apache να εκτελεί PHP αρχεία. Το restart φορτώνει το νέο module configuration.

```bash
php -v
php -m | grep -E 'mysqli|PDO|mysql'
```

Παράδειγμα εξόδου:

```text
PHP 8.1.x (cli) ...
mysqli
PDO
pdo_mysql
```

Η `php -v` εμφανίζει έκδοση interpreter. Η δεύτερη εντολή επιβεβαιώνει modules για σύνδεση με MySQL-compatible database. Η έκδοση είναι πληροφοριακή· αν το εργαστήριο χρησιμοποιεί παλιό ή ευάλωτο λογισμικό, πρέπει να παραμένει απομονωμένο.

## Δημιουργία WordPress database

```bash
sudo mariadb
```

Εκτελέστε SQL για database και dedicated application user:

```sql
CREATE DATABASE wordpress CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'wp_user'@'localhost' IDENTIFIED BY 'Use-a-unique-lab-password';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wp_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Ενδεικτική έξοδος:

```text
Query OK, 1 row affected
Query OK, 0 rows affected
Query OK, 0 rows affected
Query OK, 0 rows affected
Bye
```

Η database αποθηκεύει τα WordPress δεδομένα. Ο application user περιορίζεται στο `localhost` και λαμβάνει δικαιώματα μόνο στη συγκεκριμένη database, όχι παγκόσμια administrator privileges. Το `FLUSH PRIVILEGES` ενημερώνει άμεσα τις privilege tables.

```bash
sudo mariadb -e "SHOW DATABASES; SELECT User,Host FROM mysql.user WHERE User='wp_user';"
```

Παράδειγμα εξόδου:

```text
Database
information_schema
mysql
performance_schema
wordpress

User    Host
wp_user localhost
```

Η έξοδος επιβεβαιώνει ότι υπάρχει η database και ότι ο χρήστης συνδέεται από `localhost`. Ο κωδικός δεν εμφανίζεται, κάτι ασφαλέστερο για terminal history και screenshots.

## Λήψη και εγκατάσταση WordPress

```bash
cd /tmp
curl -LO https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
sudo mv wordpress /var/www/html/wordpress
```

Ενδεικτική έξοδος:

```text
% Total    % Received    % Xferd  Average Speed
100  ...   100  ...      ...
```

Το `curl -LO` κατεβάζει το archive και διατηρεί το remote filename. Το `tar -xzf` αποσυμπιέζει gzip-compressed tar archive. Η `mv` μεταφέρει τον φάκελο στο συνηθισμένο Apache document root. Σε production, επαληθεύετε checksum ή signature πριν από εγκατάσταση.

Ρυθμίστε owner και permissions:

```bash
sudo chown -R www-data:www-data /var/www/html/wordpress
sudo find /var/www/html/wordpress -type d -exec chmod 755 {} \\\;
sudo find /var/www/html/wordpress -type f -exec chmod 644 {} \\\;
sudo mkdir -p /var/www/html/wordpress/wp-content/uploads
sudo chown -R www-data:www-data /var/www/html/wordpress/wp-content/uploads
```

Ενδεικτική έξοδος:

```text
# Καμία έξοδος σημαίνει συνήθως επιτυχή ολοκλήρωση.
```

Το `www-data:www-data` κάνει owner τον service account του Apache. Το `755` στους φακέλους επιτρέπει traversal και ανάγνωση, ενώ περιορίζει την εγγραφή στον owner. Το `644` στα αρχεία επιτρέπει ανάγνωση αλλά όχι γενική εγγραφή. Ο φάκελος uploads απαιτεί ιδιαίτερη προσοχή, επειδή είναι web-writable περιοχή.

## Ολοκλήρωση μέσω browser

Ανοίξτε:

```text
http://127.0.0.1/wordpress/
```

Ο installer ζητά συνήθως:

- Database name: `wordpress`.
- Database user: `wp_user`.
- Database password: ο εργαστηριακός κωδικός.
- Database host: `localhost`.
- Table prefix: για παράδειγμα `wp_`.

Τα database credentials συνδέουν το WordPress με τη MariaDB. Είναι διαφορετικά από τα WordPress administrator credentials που δημιουργούνται στο browser. Μετά από επιτυχή σύνδεση εμφανίζεται η σελίδα site information για site title και administrator account.

```bash
curl -I http://127.0.0.1/wordpress/
```

Παράδειγμα εξόδου:

```text
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8
```

Η `200` σημαίνει ότι Apache και WordPress επέστρεψαν σελίδα. Η `500` συχνά δείχνει PHP error, module που λείπει, permissions ή configuration πρόβλημα. Database connection error συνήθως αφορά όνομα database, user, password, host ή κατάσταση υπηρεσίας.

## Ρύθμιση site URL

Σε τοπικό lab μπορείτε να ορίσετε ρητά τις διευθύνσεις στο `wp-config.php`:

```php
define('WP_HOME', 'http://127.0.0.1/wordpress');
define('WP_SITEURL', 'http://127.0.0.1/wordpress');
```

Οι τιμές λένε στο WordPress πού φιλοξενείται το site και πού βρίσκονται τα core files. Η loopback διεύθυνση το περιορίζει στο τοπικό μηχάνημα. Αν χρησιμοποιήσετε private hostname, χρησιμοποιήστε το με συνέπεια.

## Μέθοδος Β: Docker Compose

Η containerized εγκατάσταση διαχωρίζει application και database σε reproducible services:

```bash
sudo apt update
sudo apt install docker.io docker-compose-plugin
sudo systemctl enable --now docker
mkdir -p ~/wordpress-lab
cd ~/wordpress-lab
```

Ελέγξτε τις εκδόσεις:

```bash
docker --version
docker compose version
```

Ενδεικτική έξοδος:

```text
Docker version 26.x, build ...
Docker Compose version v2.x
```

Οι εντολές επιβεβαιώνουν ότι υπάρχουν Docker Engine και Compose plugin. Τα package names διαφέρουν ανά διανομή.

Δημιουργήστε `compose.yaml`:

```yaml
services:
  db:
    image: mariadb:10.11
    restart: unless-stopped
    environment:
      MARIADB_DATABASE: wordpress
      MARIADB_USER: wordpress
      MARIADB_PASSWORD: change-this-lab-password
      MARIADB_ROOT_PASSWORD: change-this-root-password
    volumes:
      - db_data:/var/lib/mysql

  wordpress:
    image: wordpress:latest
    depends_on:
      - db
    ports:
      - "127.0.0.1:8000:80"
    restart: unless-stopped
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: change-this-lab-password
      WORDPRESS_DB_NAME: wordpress

volumes:
  db_data:
```

Η database αποθηκεύεται σε named volume. Το WordPress συνδέεται στη database μέσω του service name `db` και της εσωτερικής θύρας 3306. Το mapping `127.0.0.1:8000:80` εκθέτει την εφαρμογή μόνο στο loopback του host. Αλλάξτε τους example passwords πριν ξεκινήσετε.

```bash
docker compose up -d
```

Ενδεικτική έξοδος:

```text
[+] Running 3/3
 ✔ Network wordpress-lab_default  Created
 ✔ Container wordpress-lab-db-1  Started
 ✔ Container wordpress-lab-wordpress-1  Started
```

Η `-d` εκτελεί τα containers detached. Το network επιτρέπει επικοινωνία μεταξύ services και το volume διατηρεί database data όταν τα containers ξαναδημιουργούνται.

```bash
docker compose ps
docker compose logs --tail=30 wordpress
```

Παράδειγμα εξόδου:

```text
NAME                         SERVICE     STATUS          PORTS
wordpress-lab-db-1           db          running
wordpress-lab-wordpress-1    wordpress   running         127.0.0.1:8000->80/tcp
```

Η λίστα δείχνει αν κάθε service τρέχει και ποιο host port αντιστοιχεί στο container port. Τα logs βοηθούν στον εντοπισμό startup failure, PHP error και database connectivity προβλημάτων. Μην εμφανίζετε πραγματικούς κωδικούς σε logs ή screenshots.

Ανοίξτε:

```text
http://127.0.0.1:8000/
```

Το database host πρέπει να είναι `db:3306`, όχι `127.0.0.1`. Μέσα στο WordPress container, το `127.0.0.1` δείχνει στο ίδιο container, ενώ το `db` επιλύεται στο database service μέσω του Compose network.

Για τερματισμό:

```bash
docker compose down
```

Τυπική έξοδος:

```text
[+] Running 2/2
 ✔ Container wordpress-lab-wordpress-1  Removed
 ✔ Container wordpress-lab-db-1         Removed
```

Η εντολή αφαιρεί containers και network, αλλά συνήθως κρατά το named volume. Το `docker compose down -v` αφαιρεί και τα volumes, άρα διαγράφει σκόπιμα τα εργαστηριακά δεδομένα.

## Μέθοδος Γ: Windows local stack

Σε Windows, ένα local stack μπορεί να παρέχει Apache, PHP και MariaDB/MySQL μέσω desktop control panel. Κρατήστε το control panel στο localhost ή σε private training network. Τοποθετήστε τον WordPress φάκελο στο document root, για παράδειγμα `C:\xampp\htdocs\wordpress`.

Ελέγξτε τον web server:

```text
http://127.0.0.1/
```

Το control panel πρέπει να εμφανίζει Apache και database service ως running. Η απόκριση browser επιβεβαιώνει ότι ο web server είναι προσβάσιμος, αλλά όχι ότι η PHP και η database λειτουργούν σωστά.

Η τοπική database administration σελίδα μπορεί να είναι:

```text
http://127.0.0.1/phpmyadmin/
```

Δημιουργήστε database `wordpress` και ανοίξτε:

```text
http://127.0.0.1/wordpress/
```

Χρησιμοποιήστε τα credentials που ρυθμίστηκαν στο local stack. Default accounts με κενό password είναι αποδεκτά μόνο σε απομονωμένο disposable lab και ποτέ σε production.

## Προαιρετικά ευάλωτα components

Ένα penetration-testing lab μπορεί να περιλαμβάνει παλιό ή σκόπιμα ευάλωτο plugin, αλλά μόνο σε isolated snapshot. Μην χρησιμοποιείτε πραγματικούς κωδικούς, πραγματικό περιεχόμενο, public DNS, port forwarding ή shared networks. Καταγράψτε versions και αφαιρέστε τα components μετά την άσκηση.

Ασφαλέστερη εναλλακτική είναι dedicated vulnerable training image ή εφαρμογή αντί να αποδυναμώνετε κανονικό website. Κρατήστε ενεργό το host firewall και κάντε bind τοπικές υπηρεσίες στο loopback όταν δεν απαιτείται απομακρυσμένη πρόσβαση.

## Αντιμετώπιση προβλημάτων

### Ο Apache επιστρέφει 403

```bash
namei -l /var/www/html/wordpress
sudo apachectl configtest
```

Παράδειγμα εξόδου:

```text
Syntax OK
```

Η `namei -l` δείχνει permissions για κάθε directory component της διαδρομής. Η `apachectl configtest` ελέγχει τη σύνταξη Apache configuration. Το `Syntax OK` σημαίνει ότι η σύνταξη είναι σωστή, όχι ότι τα permissions ή οι runtime dependencies είναι σωστά.

### Η PHP κατεβαίνει αντί να εκτελείται

```bash
php -v
sudo systemctl restart apache2
sudo journalctl -u apache2 --since "10 minutes ago" --no-pager
```

Τα logs μπορεί να αναφέρουν module που λείπει, syntax error ή configuration issue. Μην εμφανίζετε production secrets κατά τη συλλογή diagnostic evidence.

### Το WordPress δεν συνδέεται στη database

```bash
sudo systemctl status mariadb --no-pager
mariadb -u wp_user -p -h 127.0.0.1 wordpress -e 'SELECT 1;'
```

Αναμενόμενη έξοδος:

```text
1
1
```

Η πρώτη εντολή ελέγχει αν τρέχει η MariaDB. Η δεύτερη κάνει authentication και εκτελεί harmless query. Αποτυχία μπορεί να σημαίνει λάθος password, host restriction, ανύπαρκτη database, stopped service ή λάθος port.

Για Docker Compose:

```bash
docker compose exec wordpress getent hosts db
docker compose ps
```

Παράδειγμα εξόδου:

```text
172.20.0.2      db
NAME                         SERVICE     STATUS
wordpress-lab-db-1           db          running
wordpress-lab-wordpress-1    wordpress   running
```

Το `getent` επιβεβαιώνει ότι το Compose network επιλύει το όνομα `db`. Η δεύτερη έξοδος επιβεβαιώνει ότι τρέχουν και τα δύο containers.

## Τερματισμός και καθαρισμός

Για package-based lab:

```bash
sudo systemctl disable --now apache2 mariadb
```

Ενδεικτική έξοδος:

```text
Removed ... apache2.service
Stopped apache2.service
Stopped mariadb.service
```

Η εντολή σταματά τις υπηρεσίες και εμποδίζει την αυτόματη εκκίνηση. Όταν ολοκληρωθεί η άσκηση, αφαιρέστε το snapshot ή την εικονική μηχανή. Για Docker χρησιμοποιήστε `docker compose down` και αφαιρέστε το volume μόνο αν δεν χρειάζεστε τα database data.

## Βασικά μαθήματα

Ένα WordPress lab έχει τρία βασικά επίπεδα: web server, PHP application και database. Η επιτυχής εγκατάσταση απαιτεί σωστά database credentials, ενεργές υπηρεσίες, σωστά filesystem permissions και προσβάσιμο local URL. Η Docker εγκατάσταση μειώνει τις χειροκίνητες εξαρτήσεις και βελτιώνει την επαναληψιμότητα, ενώ η package-based εγκατάσταση κάνει κάθε επίπεδο ορατό για εκμάθηση και troubleshooting.
