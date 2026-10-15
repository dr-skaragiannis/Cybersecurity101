# Ανώνυμες συνδέσεις για Pentesters: Αναλυτική παρουσίαση κοινόχρηστων αρχείων FTP, SMB και NFS

## Εισαγωγή

Τα πρωτόκολλα διαμοιρασμού αρχείων αποτελούν βασικό μέρος της υποδομής σχεδόν κάθε οργανισμού: FTP for legacy data drops, SMB for Windows-compatible file servers, and NFS for Unix-to-Unix exports. Κάθε πρωτόκολλο διαθέτει επίσης λειτουργία ανώνυμης ή guest πρόσβασης και οι διαχειριστές συχνά ενεργοποιούν μία από αυτές για λόγους ευκολίας και ξεχνούν να την περιορίσουν. Γι' αυτό οι λανθασμένα ρυθμισμένα κοινόχρηστοι πόροι εμφανίζονται ξανά και ξανά ως ευρήματα υψηλού αντίκτυπου σε εσωτερικούς ελέγχους. Η πρόσβαση χωρίς αυθεντικοποίηση μπορεί να εκθέσει πηγαίο κώδικα, αντίγραφα ασφαλείας, αρχεία ρυθμίσεων, διαπιστευτήρια και προσωπικά δεδομένα, τα οποία κανένας περιμετρικός έλεγχος δεν μπορεί να προστατεύσει εκ των υστέρων.

Ο οδηγός αυτός δημιουργεί ένα σκόπιμα ευάλωτο εργαστηριακό περιβάλλον σε Ubuntu, εκθέτει και τις τρεις υπηρεσίες σε πρόσβαση χωρίς αυθεντικοποίηση και στη συνέχεια κάνει enumeration από ένα μηχάνημα επιτιθέμενου με Nmap, τον τυπικό FTP client, το `smbclient` και το NetExec. Η ροή εργασίας είναι σκόπιμα επαναληπτική: επιβεβαίωση της έκθεσης, καταγραφή όσων είναι διαθέσιμα, ανάκτηση ενός marker file και στη συνέχεια κατανόηση της ρύθμισης στον server που ευθύνεται για την έκθεση. Κάθε ενότητα πρωτοκόλλου συνδυάζει επομένως τα αποτελέσματα της επίθεσης με το αμυντικό μέτρο που θα την είχε αποτρέψει.

****Τοπολογία εργαστηρίου.**** Οι ασκήσεις προϋποθέτουν δύο εικονικές μηχανές σε απομονωμένο δίκτυο host-only. Ο στόχος εκτελεί Ubuntu 22.04 LTS με εγκατεστημένα τα `vsftpd`, Samba και NFS kernel server. Ο επιτιθέμενος είναι μια security distribution της οικογένειας Debian με εγκατεστημένα Nmap, FTP και SMB clients, NetExec και εργαλεία NFS client. Οι διευθύνσεις IP που χρησιμοποιούνται σε όλο τον οδηγό είναι `192.168.1.9` for the target and `192.168.1.17` for the attacker.

| Role | Λειτουργικό σύστημα | Διεύθυνση | Σημειώσεις |
| --- | --- | --- | --- |
| Στόχος | Ubuntu 22.04 LTS server | `192.168.1.9` | `vsftpd`, Samba, NFS kernel server, hostname `ubuntu-lab` |
| Επιτιθέμενος | Debian-based security distro | `192.168.1.17` | nmap, `ftp`, `smbclient`, NetExec, NFS utilities |
| Δίκτυο | Τμήμα εργαστηριακού δικτύου Host-only / NAT | `192.168.1.0/24` | Χωρίς διαδρομή προς το Internet από το εργαστήριο |

> ****Εξουσιοδότηση και εύρος.**** Όλα όσα περιγράφονται εδώ αποτελούν τεχνικές επίθεσης. Εκτελέστε τα μόνο σε μηχανήματα που σας ανήκουν ή για τα οποία διαθέτετε γραπτή εξουσιοδότηση — ένα ειδικά σχεδιασμένο εικονικό εργαστήριο είναι ο κατάλληλος χώρος για εξάσκηση και ο μόνος χώρος όπου αυτές οι εντολές είναι ασφαλείς. Το enumeration χωρίς αυθεντικοποίηση, η πρόσβαση σε guest shares, η ανάκτηση αρχείων, το NFS mounting και οι δοκιμές privilege escalation είναι σαφώς ανιχνεύσιμες και επιθετικές ενέργειες σε δίκτυο παραγωγής, ενώ η εκτέλεσή τους χωρίς άδεια αποτελεί ποινικό αδίκημα στις περισσότερες δικαιοδοσίες. Τα αμυντικά μέτρα παρατίθενται δίπλα σε κάθε τεχνική, ώστε το υλικό να μπορεί να μελετηθεί και από την πλευρά της άμυνας.

### Πώς να διαβάζετε τα transcripts

Οι συνεδρίες τερματικού στον οδηγό ακολουθούν τις συμβάσεις του ίδιου του shell και ορισμένα σημεία αξίζει να είναι γνωστά πριν από την πρώτη εντολή:

* Ένα prompt που τελειώνει σε `#` σημαίνει ότι το shell εκτελείται ως λογαριασμός root (`root@kali:~#`), ενώ ένα prompt που τελειώνει σε `$` σημαίνει ότι πρόκειται για απλό χρήστη. Τόσο η ρύθμιση του στόχου όσο και το enumeration από την πλευρά του επιτιθέμενου παρουσιάζονται ως root, επειδή η εγκατάσταση πακέτων, η διαχείριση υπηρεσιών, το NFS mounting και το raw-socket scanning απαιτούν προνόμια.
* Το transcript `root@ubuntu-lab:~#` είναι το μηχάνημα-στόχος κατά τη δημιουργία του εργαστηρίου. Το transcript `root@kali:~#` είναι το μηχάνημα του επιτιθέμενου κατά το enumeration.
* Η έξοδος που παρουσιάζεται εδώ καταγράφηκε σε συστήματα της οικογένειας Debian και είναι ενδεικτική και όχι byte-identical: package versions, port numbers, timestamps, MAC addresses, directory dates, and tool banners will differ on your own lab. Αυτό που έχει σημασία είναι η δομή της εξόδου.
* Τα περιεχόμενα αρχείων που εμφανίζονται χωρίς prompt είναι τα κυριολεκτικά bytes του αρχείου, όπως θα εμφανίζονταν σε έναν editor.
* Όταν μια εντολή είναι αρκετά μεγάλη ώστε να σπάει σε δύο γραμμές στην τεκμηρίωση εργαλείων τρίτων, εδώ εμφανίζεται πάντα ως μία ενιαία γραμμή που μπορείτε να κάνετε paste.

### Πίνακας περιεχομένων

1. [Γιατί έχουν σημασία τα file shares χωρίς αυθεντικοποίηση](#1-why-unauthenticated-file-shares-matter)
2. [Ρύθμιση εργαστηρίου — στόχος, επιτιθέμενος και δίκτυο](#2-lab-setup--target-attacker-and-network)
3. [Anonymous FTP — ρύθμιση του vsftpd](#3-anonymous-ftp--configuring-vsftpd)
4. [Anonymous FTP — enumeration και ανάκτηση](#4-anonymous-ftp--enumeration-and-retrieval)
5. [Guest SMB — ρύθμιση του Samba](#5-guest-smb--configuring-samba)
6. [Guest SMB — enumeration και ανάκτηση](#6-guest-smb--enumeration-and-retrieval)
7. [Insecure NFS — ρύθμιση των exports](#7-insecure-nfs--configuring-exports)
8. [Insecure NFS — enumeration, λήψη και mounting](#8-insecure-nfs--enumeration-download-and-mounting)
9. [Σύνοψη hardening](#9-hardening-summary)
10. [Cheat sheet γρήγορης αναφοράς](#10-quick-reference-cheat-sheet)
11. [Ασκήσεις εξάσκησης](#11-practice-exercises)

---

## 1. Γιατί έχουν σημασία τα file shares χωρίς αυθεντικοποίηση

Τα τρία πρωτόκολλα αυτού του οδηγού εκθέτουν δεδομένα με τρεις διαφορετικούς τρόπους, αλλά το βασικό σφάλμα είναι το ίδιο: ένας server εμπιστεύεται έναν network client χωρίς να επαληθεύει την ταυτότητά του. FTP exposes an anonymous root directory through the unprivileged FTP account. SMB exposes shares marked as guest-accessible. NFS exposes exported paths while trusting the UID claimed by the client. Κάθε μία από αυτές τις συμπεριφορές μπορεί να είναι χρήσιμη σε αυστηρά ελεγχόμενο περιβάλλον και καθεμία αποτελεί εύρημα μόλις είναι προσβάσιμη από μη έμπιστο τμήμα δικτύου.

Η επιχειρησιακή συνέπεια είναι ότι το file-share enumeration πρέπει να πραγματοποιείται νωρίς σε έναν εσωτερικό έλεγχο. Πριν από password guessing, exploitation και post-exploitation, ο ελεγκτής πρέπει να εξετάζει αν κάποιο μηχάνημα απλώς παρέχει αρχεία χωρίς έλεγχο. Οι εντολές είναι γρήγορες, τα στοιχεία είναι σαφή και τα αρχεία που ανακτώνται συχνά περιέχουν τα διαπιστευτήρια ή τις λεπτομέρειες ρυθμίσεων που απαιτούνται για την επόμενη φάση. A single world-readable backup or deployment script can therefore shorten an engagement more than a long brute-force run.

Είναι επίσης χρήσιμο να διακρίνουμε τη διαθεσιμότητα από την εξουσιοδότηση. Ένα share μπορεί σκόπιμα να είναι διαθέσιμο και ταυτόχρονα να απαιτεί αυθεντικοποίηση· αυτό είναι φυσιολογικό. Η ευπάθεια εδώ δεν είναι η ύπαρξη FTP, SMB ή NFS, αλλά το ότι μια anonymous ή guest ταυτότητα μπορεί να περιηγηθεί ή να ανακτήσει δεδομένα που προορίζονται για αυθεντικοποιημένους χρήστες. Σε όλο αυτό το walkthrough, επομένως, το κρίσιμο ερώτημα δεν είναι «είναι ανοιχτή η θύρα;» αλλά «τι μπορεί να κάνει ένας client χωρίς αυθεντικοποίηση μετά τη σύνδεση;»

---

## 2. Ρύθμιση εργαστηρίου — στόχος, επιτιθέμενος και δίκτυο

Πριν ξεκινήσει ο έλεγχος, και οι δύο εικονικές μηχανές πρέπει να μπορούν να επικοινωνούν μεταξύ τους και με τίποτα άλλο. Τοποθετήστε τις στο ίδιο απομονωμένο τμήμα εργαστηρίου host-only ή NAT, ορίστε στον στόχο `192.168.1.9` και στον επιτιθέμενο `192.168.1.17` και δημιουργήστε snapshot πριν εγκαταστήσετε τις ευάλωτες υπηρεσίες. Το snapshot είναι σημαντικό επειδή ο οδηγός αποδυναμώνει σκόπιμα τις προεπιλεγμένες ρυθμίσεις· χρειάζεστε ένα καθαρό σημείο επαναφοράς πριν και μετά τη φάση επίθεσης.

Ο στόχος χρειάζεται τρία πακέτα server:

```bash
root@ubuntu-lab:~# apt update
Hit:1 http://archive.ubuntu.com/ubuntu jammy InRelease
Hit:2 http://security.ubuntu.com/ubuntu jammy-security InRelease
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
All packages are up to date.
```

Εγκαταστήστε τους FTP, SMB και NFS servers στις αντίστοιχες ενότητες πρωτοκόλλου παρακάτω και όχι όλους μαζί. Αυτή η σειρά διατηρεί κάθε λανθασμένη ρύθμιση συνδεδεμένη με την υπηρεσία που την εισήγαγε και διευκολύνει την ερμηνεία των μεταγενέστερων hardening checks.

Ο επιτιθέμενος χρειάζεται τους αντίστοιχους clients και τα εργαλεία enumeration. On a Debian-family security distribution these are commonly installed already; on a minimal system, install Nmap, a command-line FTP client, Samba client utilities, NetExec, and NFS client tools. Επαληθεύστε πρώτα τη βασική συνδεσιμότητα, επειδή κάθε μεταγενέστερη αποτυχία διαγιγνώσκεται ευκολότερα όταν είναι γνωστό ότι η διαδρομή δικτύου λειτουργεί:

```bash
root@kali:~# ping -c 2 192.168.1.9
PING 192.168.1.9 (192.168.1.9) 56(84) bytes of data.
64 bytes from 192.168.1.9: icmp_seq=1 ttl=64 time=0.52 ms
64 bytes from 192.168.1.9: icmp_seq=2 ttl=64 time=0.48 ms

--- 192.168.1.9 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.480/0.500/0.520/0.020 ms
```

Χρόνος round-trip κάτω από ένα millisecond είναι συνήθης ένδειξη δύο εικονικών μηχανών στον ίδιο host ή στο ίδιο τμήμα εργαστηρίου. Αν το ping αποτύχει, σταματήστε και διορθώστε το virtual networking πριν συνεχίσετε· ένας μη προσβάσιμος στόχος θα κάνει κάθε σφάλμα συγκεκριμένης υπηρεσίας παραπλανητικό.

---

## 3. Anonymous FTP — ρύθμιση του vsftpd

Το `vsftpd` (Very Secure FTP Daemon) είναι ο προεπιλεγμένος FTP server σε πολλά Debian-derived συστήματα. Από προεπιλογή ακούει στη TCP θύρα 21, υποστηρίζει συνδέσεις τοπικών χρηστών και απενεργοποιεί την anonymous πρόσβαση. Το εργαστήριο αντιστρέφει αυτή την τελευταία προεπιλογή και ρυθμίζει ένα passive anonymous share που μπορεί να διαβαστεί από οποιονδήποτε γείτονα του δικτύου.

### 3.1 Εγκατάσταση του vsftpd

Εγκαταστήστε το πακέτο του daemon:

```bash
root@ubuntu-lab:~# apt install vsftpd
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following NEW packages will be installed:
  vsftpd
0 upgraded, 1 newly installed, 0 to remove and 0 not upgraded.
Need to get 142 kB of archives.
After this operation, 385 kB of additional disk space will be used.
Do you want to continue? [Y/n] y
Get:1 http://archive.ubuntu.com/ubuntu jammy/main amd64 vsftpd amd64 3.0.5-0ubuntu1 [142 kB]
Fetched 142 kB in 1s (198 kB/s)
Preconfiguring packages ...
Selecting previously unselected package vsftpd.
Preparing to unpack .../vsftpd_3.0.5-0ubuntu1_amd64.deb ...
Unpacking vsftpd (3.0.5-0ubuntu1) ...
Setting up vsftpd (3.0.5-0ubuntu1) ...
Processing triggers for man-db (2.10.2-1) ...
```

Το μικρό footprint ενός μόνο πακέτου είναι ένας λόγος για τον οποίο οι ομάδες επιλέγουν το `vsftpd`: ο daemon έχει λίγες εξαρτήσεις και συντηρητικές προεπιλογές. Αυτή η απλότητα είναι επίσης ο λόγος που ο έλεγχος των ρυθμίσεων έχει σημασία. Δεν υπάρχουν πολλά προς έλεγχο, επομένως ο auditor πρέπει στην πράξη να ελέγξει τα πάντα.

### 3.2 Έλεγχος της προεπιλεγμένης ρύθμισης

Η ρύθμιση του server βρίσκεται στο `/etc/vsftpd.conf`. Πριν αλλάξετε οτιδήποτε, ελέγξτε τις directives που ελέγχουν τους listeners και την anonymous πρόσβαση:

```bash
root@ubuntu-lab:~# grep -nE '^(#)?(listen|listen_ipv6|anonymous_enable|local_enable)=' /etc/vsftpd.conf
12:listen=NO
18:listen_ipv6=YES
27:#anonymous_enable=NO
29:local_enable=YES
```

Η σημαντική γραμμή είναι η προεπιλογή της anonymous πρόσβασης. Με απενεργοποιημένη την anonymous πρόσβαση, κανένα από τα FTP enumeration βήματα της ενότητας 4 δεν θα λειτουργούσε. Οι ρυθμίσεις listener καθορίζουν αν ο daemon ακούει σε IPv4, IPv6 ή και στα δύο· η πακεταρισμένη προεπιλογή του Ubuntu χρησιμοποιεί τον IPv6 listener ενώ εξακολουθεί να δέχεται IPv4 συνδέσεις μέσω dual-stack socket. Καταγράψτε τώρα την κατάσταση «πριν», επειδή η ενότητα 9 αναιρεί την αλλαγή του εργαστηρίου επαναφέροντας μια περιοριστική τιμή.

### 3.3 Ενεργοποίηση anonymous πρόσβασης

Ανοίξτε το αρχείο ρυθμίσεων:

```bash
root@ubuntu-lab:~# nano /etc/vsftpd.conf
```

Αλλάξτε τη directive anonymous-access σε ενεργή τιμή `YES`:

```text
anonymous_enable=YES
```

Αφήστε το `local_enable=YES` αμετάβλητο για αυτό το εργαστήριο. Ο συνδυασμός αυτός δέχεται τόσο τοπικούς Linux χρήστες με τους κανονικούς κωδικούς τους όσο και τον anonymous χρήστη χωρίς κωδικό. Σε production, ο συνδυασμός αυτός αποτελεί κρίσιμη έκθεση όταν η θύρα 21 είναι προσβάσιμη: οποιοσδήποτε φτάσει στη θύρα FTP μπορεί να κάνει listing και download οτιδήποτε μπορεί να διαβάσει ο FTP account.

### 3.4 Δημιουργία anonymous directory και marker file

Ο anonymous FTP χρήστης αντιστοιχίζεται στον τοπικό λογαριασμό `ftp`. Το εργαστήριο δημιουργεί έναν συμβατικό public υποκατάλογο και τον αρχικοποιεί με ένα μικρό retrieval marker:

```bash
root@ubuntu-lab:~# mkdir -p /var/ftp/pub
root@ubuntu-lab:~# chown nobody:nogroup /var/ftp/pub
root@ubuntu-lab:~# cd /var/ftp/pub
root@ubuntu-lab:/var/ftp/pub# echo "Lab FTP retrieval marker" > note.txt
root@ubuntu-lab:/var/ftp/pub# ls -l
total 4
-rw-r--r-- 1 root root 25 Feb 10 12:01 note.txt
root@ubuntu-lab:/var/ftp/pub# cat note.txt
Lab FTP retrieval marker
```

Το όνομα `pub` είναι συμβατικό για anonymous FTP drop folder, με ρίζες στα πρώτα Internet file archives, και είναι ακριβώς αυτό που αναμένει να βρει ένας επιτιθέμενος. Η ιδιοκτησία `nobody:nogroup` ακολουθεί τη συμβατική μη προνομιούχα ταυτότητα για public data. Το marker file παρέχει στο enumeration workflow ένα συγκεκριμένο artefact προς ανάκτηση· τα περιεχόμενά του είναι σκόπιμα απλά, επειδή το ζητούμενο είναι η ανάκτηση και όχι τα ίδια τα δεδομένα.

### 3.5 Ρύθμιση της anonymous λειτουργίας

Προσθέστε το block anonymous mode στο `/etc/vsftpd.conf`:

```text
# Point anonymous users at the directory created earlier.
anon_root=/var/ftp/

# Do not prompt for a password on the command line.
no_anon_password=YES

# Show the user and group as ftp:ftp, regardless of the owner.
hide_ids=YES

# Constrain the passive-mode port range for predictable firewalling.
pasv_min_port=40000
pasv_max_port=50000
```

Κάθε directive έχει διαφορετική επίδραση. Το `anon_root` τοποθετεί τις anonymous sessions στη ρίζα `/var/ftp/`, έτσι το share ξεκινά από τον κατάλογο που προετοιμάστηκε παραπάνω. Το `no_anon_password` αφαιρεί το password prompt για anonymous logins. Το `hide_ids` αποκρύπτει το υποκείμενο UID και GID στα directory listings, αποτρέποντας το ownership enumeration αλλά αποκρύπτοντας επίσης χρήσιμες forensic πληροφορίες. Το ζεύγος passive-port περιορίζει τις θύρες data connection στις `40000–50000`, καθιστώντας τους firewall κανόνες προβλέψιμους.

Από αμυντική σκοπιά, το `hide_ids` απαιτεί ιδιαίτερη προσοχή. Υπάρχει ώστε οι διαχειριστές να μπορούν να προσφέρουν anonymous share χωρίς να αποκαλύπτουν ποιος Linux account κατέχει κάθε αρχείο. Αυτό αποτελεί βελτίωση ιδιωτικότητας και όχι security boundary: δεν αποτρέπει το listing, το downloading ή την εξαγωγή συμπερασμάτων για την ύπαρξη ευαίσθητων αρχείων.

### 3.6 Επανεκκίνηση και επαλήθευση της υπηρεσίας

Κάντε restart στον daemon για να εφαρμοστεί η ρύθμιση:

```bash
root@ubuntu-lab:/var/ftp/pub# service vsftpd restart
root@ubuntu-lab:/var/ftp/pub# cd ~
root@ubuntu-lab:~# ss -tlnp | grep ':21'
LISTEN 0      32                 *:21              *:*    users:(("vsftpd",pid=1842,fd=3))
root@ubuntu-lab:~# systemctl status vsftpd --no-pager
● vsftpd.service - vsftpd FTP server
     Loaded: loaded (/lib/systemd/system/vsftpd.service; enabled; vendor preset: enabled)
     Active: active (running) since Sat 2026-10-10 12:04:11 UTC; 18s ago
   Main PID: 1842 (vsftpd)
      Tasks: 1 (limit: 2261)
     Memory: 1.1M
        CPU: 9ms
     CGroup: /system.slice/vsftpd.service
             └─1842 /usr/sbin/vsftpd /etc/vsftpd.conf
```

Η εντολή `service` δεν εμφανίζει output σε περίπτωση επιτυχίας· σε distribution που διαχειρίζεται μέσω systemd, κενό αποτέλεσμα μετά το `restart` αποτελεί αναμενόμενη ένδειξη επιτυχίας. Η έξοδος του `ss` αποδεικνύει ότι ο daemon είναι συνδεδεμένος στη θύρα 21 και το `systemctl status` επιβεβαιώνει ότι η εκτελούμενη διεργασία χρησιμοποιεί το τροποποιημένο αρχείο ρυθμίσεων. If port 21 is absent, recheck the listener directives before assuming that anonymous access is broken: a daemon that is not listening cannot be enumerated at all.

---

## 4. Anonymous FTP — enumeration και ανάκτηση

Με τον lab server ενεργό, το workflow του επιτιθέμενου έχει δύο βασικά βήματα: επιβεβαίωση της anonymous πρόσβασης με Nmap και στη συνέχεια αλληλεπίδραση με το share μέσω FTP session για ανάκτηση του marker.

### 4.1 Ανίχνευση υπηρεσίας με Nmap

Η επιλογή `-A` του Nmap ενεργοποιεί version detection, script scanning, OS detection και traceroute σε μία εκτέλεση. Against a single lab port, it is a compact way to confirm both the daemon and the misconfiguration:

```bash
root@kali:~# nmap -A -p 21 192.168.1.9
Starting Nmap 7.94 ( https://nmap.org ) at 2026-10-10 12:06 UTC
Nmap scan report for 192.168.1.9
Host is up (0.00052s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| drwxr-xr-x    2 ftp      ftp          4096 Feb 10 12:01 pub
|_End of status.
MAC Address: 00:0C:29:1B:2C:3D (VMware)
Service Info: OS: Unix

TRACEROUTE
HOP RTT     ADDRESS
1   0.52 ms 192.168.1.9

Nmap done: 1 IP address (1 host up) scanned in 8.42 seconds
```

Τρία στοιχεία καθορίζουν το επόμενο βήμα. First, the service is `vsftpd 3.0.5` on the expected port. Second, the `ftp-anon` script reports `Anonymous FTP login allowed (FTP code 230)`, which is direct confirmation that no password is required. Third, the script lists the `pub` directory inside the anonymous root, so the operator already knows where to look before opening an interactive session. On a local lab network, the one-hop traceroute and VMware MAC prefix also corroborate that this is a nearby virtual target rather than a routed production host.

Η επιλογή `-A` είναι πιο θορυβώδης και αργή σε σχέση με την `-sV`, επομένως είναι προτιμότερο να χρησιμοποιείται αφού ένα πιο περιορισμένο scan έχει εντοπίσει κάτι που αξίζει διερεύνηση. Here its value is completeness: one command establishes the version, the anonymous-access finding, and the initial directory inventory.

### 4.2 FTP session και ανάκτηση αρχείου

Συνδεθείτε ως `anonymous`, ελέγξτε το share, μπείτε στο `pub` και κατεβάστε το `note.txt`:

```bash
root@kali:~# ftp 192.168.1.9
Connected to 192.168.1.9.
220 (vsFTPd 3.0.5)
Name (192.168.1.9:root): anonymous
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||40217|).
150 Here comes the directory listing.
drwxr-xr-x    2 ftp      ftp          4096 Feb 10 12:01 pub
226 Directory send OK.
ftp> cd pub
250 Directory successfully changed.
ftp> ls
229 Entering Extended Passive Mode (|||40245|).
150 Here comes the directory listing.
-rw-r--r--    1 ftp      ftp            25 Feb 10 12:01 note.txt
226 Directory send OK.
ftp> get note.txt
local: note.txt remote: note.txt
229 Entering Extended Passive Mode (|||40271|).
150 Opening BINARY mode data connection for note.txt (25 bytes).
100% |********************************|    25        0.01 KiB/s    00:00 ETA
226 Transfer complete.
25 bytes received in 0.00 secs (12.4 kB/s)
ftp> bye
221 Goodbye.
root@kali:~# cat note.txt
Lab FTP retrieval marker
```

Η session αυθεντικοποιείται ως `anonymous`, λαμβάνει κωδικό `230` για επιτυχή σύνδεση, κάνει listing της anonymous root, μεταβαίνει στο `pub` και κατεβάζει το marker. The `229` lines show passive-mode data connections being opened inside the configured `40000–50000` range. The transfer is tiny here, but the same workflow retrieves multi-gigabyte archives with no additional access required.

Δύο λεπτομέρειες αξίζει να διατηρηθούν για το reporting. Πρώτον, το FTP μεταδίδει το username, τις εντολές, το file listing και τα περιεχόμενα των αρχείων χωρίς κρυπτογράφηση, εκτός αν έχει ρυθμιστεί TLS. Δεύτερον, το `hide_ids=YES` είναι ορατό στο listing: όλα φαίνονται να ανήκουν στο `ftp:ftp`, παρότι το marker file δημιουργήθηκε από root. Επομένως, το ownership masking δεν απέτρεψε την αποκάλυψη· απλώς έκανε το listing λιγότερο πληροφοριακό.

---

## 5. Guest SMB — ρύθμιση του Samba

Το Samba υλοποιεί το πρωτόκολλο SMB/CIFS σε Linux και γεφυρώνει Linux servers με Windows-compatible clients. Όπως και το `vsftpd`, διαθέτει συντηρητικές προεπιλογές τις οποίες το εργαστήριο αποδυναμώνει σκόπιμα. Το resulting `[shares]` export είναι browsable, writable και προσβάσιμο σε SMB clients χωρίς αυθεντικοποίηση.

### 5.1 Εγκατάσταση του Samba

Εγκαταστήστε το server package:

```bash
root@ubuntu-lab:~# apt install samba
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following additional packages will be installed:
  python3-samba samba-common samba-common-bin samba-dsdb-modules tdb-tools
Suggested packages:
  bind9 bind9utils ctdb ldb-tools ntp smbldap-tools winbind heimdal-clients
The following NEW packages will be installed:
  python3-samba samba samba-common samba-common-bin samba-dsdb-modules tdb-tools
0 upgraded, 6 newly installed, 0 to remove and 0 not upgraded.
Need to get 2,981 kB of archives.
After this operation, 27.4 MB of additional disk space will be used.
Do you want to continue? [Y/n] y
```

Ο installer εγκαθιστά το Samba μαζί με τις κοινές βιβλιοθήκες και τα υποστηρικτικά εργαλεία. Ο daemon είναι έτοιμος για ρύθμιση μετά την εγκατάσταση, αλλά δεν εκθέτει ακόμη το lab share επειδή η προεπιλεγμένη ρύθμιση δηλώνει μόνο global συμπεριφορά και sections σχετικά με printers.

### 5.2 Έλεγχος του καταλόγου ρυθμίσεων του Samba

Μεταβείτε στον κατάλογο ρυθμίσεων και ελέγξτε τα περιεχόμενά του:

```bash
root@ubuntu-lab:~# cd /etc/samba/
root@ubuntu-lab:/etc/samba# ls -al
total 32
drwxr-xr-x  3 root root  4096 Feb 10 12:10 .
drwxr-xr-x 96 root root  4096 Feb 10 11:58 ..
-rw-r--r--  1 root root    17 Feb 10 11:59 gdbcommands
-rw-r--r--  1 root root  9831 Feb 10 11:59 smb.conf
drwxr-xr-x  2 root root  4096 Feb 10 11:59 tls
root@ubuntu-lab:/etc/samba# nano smb.conf
```

Το βασικό αρχείο είναι το `smb.conf`· το `gdbcommands` υποστηρίζει debugging και το `tls` περιέχει υλικό πιστοποιητικών. Η απευθείας επεξεργασία του `smb.conf` είναι σαφής για εργαστήριο, αλλά οι αλλαγές σε production πρέπει πάντα να επικυρώνονται με `testparm` πριν εφαρμοστεί η νέα ρύθμιση.

### 5.3 Ορισμός του `[shares]` section με guest πρόσβαση

Προσθέστε τον ορισμό του lab share:

```text
[shares]
path = /var/www/
available = yes
read only = no
browsable = yes
public = yes
writable = yes
guest ok = yes
```

Κάθε γραμμή συμβάλλει στην έκθεση. Το `guest ok = yes` αντιστοιχίζει clients χωρίς αυθεντικοποίηση σε guest πρόσβαση. Το `public = yes` είναι legacy σύνταξη για την ίδια πρόθεση. Το `browsable = yes` κάνει το share ορατό κατά το enumeration. Τα `read only = no` και `writable = yes` εκφράζουν δύο φορές την ίδια writable ρύθμιση: οι δύο directives είναι αντίστροφες μορφές της ίδιας επιλογής, επομένως οποιαδήποτε από τις δύο μόνη της θα επέτρεπε writes. Η δημοσίευση του `/var/www/` αυξάνει τον κίνδυνο επειδή αυτή η διαδρομή συνήθως περιέχει web content· ένα writable SMB share πάνω σε web-served files μπορεί να μετατραπεί σε write-anywhere primitive πίσω από ένα HTTP endpoint.

Επικυρώστε την τελική ρύθμιση πριν από το restart:

```bash
root@ubuntu-lab:/etc/samba# testparm -s
Loaded services file OK.
Weak crypto is allowed by default.

Server role: ROLE_STANDALONE

# Global parameters
[global]
	map to guest = Bad User
	obey pam restrictions = Yes
	pam password change = Yes
	unix password sync = Yes

[shares]
	path = /var/www/
	read only = No
```

Το `testparm` αναλύει την effective configuration και αναφέρει συντακτικά ή σημασιολογικά προβλήματα. Το normalized output του είναι επίσης χρήσιμο evidence: δείχνει το share path και την writable κατάσταση χωρίς να απαιτεί από τον αναγνώστη να ερμηνεύσει κάθε redundant directive του source file.

Κάντε restart στον SMB daemon και επιβεβαιώστε τους αναμενόμενους listeners:

```bash
root@ubuntu-lab:/etc/samba# systemctl restart smbd
root@ubuntu-lab:/etc/samba# ss -tlnp | grep -E ':(139|445)'
LISTEN 0      50                 *:139             *:*    users:(("smbd",pid=2261,fd=34))
LISTEN 0      50                 *:445             *:*    users:(("smbd",pid=2261,fd=33))
root@ubuntu-lab:/etc/samba# cd ~
```

Οι θύρες 139 και 445 είναι οι παραδοσιακοί SMB listeners. If only one is present, check whether another service owns the missing socket and whether the host firewall is interfering. If neither is present, the Samba configuration did not load as expected.

### 5.4 Δημιουργία και έλεγχος των shared files

Δημιουργήστε ένα marker στον shared web directory και επιβεβαιώστε τοπικά τα περιεχόμενά του:

```bash
root@ubuntu-lab:~# echo "Lab SMB retrieval marker" > /var/www/file.txt
root@ubuntu-lab:~# cd /var/www/
root@ubuntu-lab:/var/www# ls -l
total 12
-rw-r--r-- 1 root root   25 Feb 10 12:14 file.txt
drwxr-xr-x 2 root root 4096 Feb 10 11:47 html
root@ubuntu-lab:/var/www# cat file.txt
Lab SMB retrieval marker
root@ubuntu-lab:/var/www# cd ~
```

Ο υποκατάλογος `html` είναι ο κανονικός κατάλογος web content, ενώ το `file.txt` είναι ο επαληθεύσιμος στόχος ανάκτησης του εργαστηρίου. Seeding a known file matters because successful anonymous access should be proved by reading back exact bytes, not merely by listing a filename.

---

## 6. Guest SMB — enumeration και ανάκτηση

Δύο συμπληρωματικά εργαλεία χειρίζονται το Samba enumeration. Το NetExec ανακαλύπτει shares γρήγορα σε έναν ή πολλούς hosts, ενώ το `smbclient` παρέχει διαδραστική εργασία session πάνω σε ένα συγκεκριμένο export. Χρησιμοποιήστε NetExec για discovery και `smbclient` για retrieval.

### 6.1 Ανακάλυψη shares με NetExec

Αυθεντικοποιηθείτε ως `guest` με κενό password και κάντε enumeration των shares:

```bash
root@kali:~# nxc smb 192.168.1.9 --shares -u 'guest' -p ''
[*] SMB         192.168.1.9    445    UBUNTU-LAB     [*] Unix - Samba 4.17.7-Ubuntu
[+] SMB         192.168.1.9    445    UBUNTU-LAB     UBUNTU-LAB\guest: (Guest)
[*] SMB         192.168.1.9    445    UBUNTU-LAB     Enumerated shares
[*] SMB         192.168.1.9    445    UBUNTU-LAB     Share           Permissions     Remark
[*] SMB         192.168.1.9    445    UBUNTU-LAB     -----           -----------     ------
[*] SMB         192.168.1.9    445    UBUNTU-LAB     print$                          Printer Drivers
[*] SMB         192.168.1.9    445    UBUNTU-LAB     shares          READ            Lab file share
[*] SMB         192.168.1.9    445    UBUNTU-LAB     IPC$                            IPC Service
```

Η έξοδος επιβεβαιώνει guest authentication με το marker `(Guest)` και εμφανίζει τρία shares. `print$` is the default printer-driver share, `IPC$` is the interprocess-communication share, and `shares` is the lab export. The `READ` permission against `shares` is the authorisation to continue: an unauthenticated client can browse and download from that export.

NetExec’s marker convention is consistent across protocols. Informational lines begin with `[*]`, successes begin with `[+]`, and failures begin with `[-]`. That convention makes one-host output readable and multi-host output scannable.

### 6.2 Καταχώριση shares με `smbclient`

The same inventory can be obtained directly with `smbclient`. The `-N` option suppresses the password prompt for guest access, while `-L` lists shares:

```bash
root@kali:~# smbclient -N -L //192.168.1.9
Anonymous login successful

	Sharename       Type      Comment
	---------       ----      -------
	print$          Disk      Printer Drivers
	shares          Disk      Lab file share
	IPC$            IPC       IPC Service (Samba 4.17.7-Ubuntu)
Reconnecting with SMB1 for workgroup listing.
smbXcli_negprot_smb1_done: No compatible protocol selected by client, err=NT_STATUS_INVALID_PARAMETER
Unable to connect with SMB1 -- no workgroup available
```

Ο πίνακας shares συμφωνεί με το αποτέλεσμα του NetExec. The trailing SMB1 messages are also informative: the client attempts an obsolete dialect only for workgroup listing, the negotiation fails, and the server does not fall back to SMB1. That failure is a welcome default. Modern clients should negotiate SMB2 or SMB3, and obsolete-dialect fallback should remain disabled.

### 6.3 Διαδραστική ανάκτηση με `smbclient`

Ανοίξτε το `shares` export χωρίς password, κάντε listing και ανακτήστε το marker:

```bash
root@kali:~# smbclient //192.168.1.9/shares -N
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Sat Oct 10 12:14:22 2026
  ..                                  D        0  Sat Oct 10 11:47:09 2026
  file.txt                            A       25  Sat Oct 10 12:14:22 2026
  html                                D        0  Sat Oct 10 11:47:09 2026

		26632192 blocks of size 1024. 18874368 blocks available
smb: \> get file.txt
getting file \file.txt of size 25 as file.txt (1.2 KiloBytes/sec) (average 1.2 KiloBytes/sec)
smb: \> exit
root@kali:~# cat file.txt
Lab SMB retrieval marker
```

Το attribute `A` δηλώνει κανονικό αρχείο και το `D` δηλώνει directory. The local `cat` proves that the retrieved bytes match the seeded marker. Επειδή το export είναι writable, η ίδια session θα μπορούσε επίσης να ανεβάσει ή να αντικαταστήσει αρχεία· σε εξουσιοδοτημένο έλεγχο, αυτή η δυνατότητα πρέπει να επιδεικνύεται προσεκτικά και με τον ελάχιστο δυνατό αντίκτυπο, κατά προτίμηση με ένα μοναδικά ονομασμένο marker που αφαιρείται αμέσως μετά.

---

## 7. Insecure NFS — ρύθμιση των exports

NFS, the Δίκτυο File System, is the canonical Unix-to-Unix file-sharing protocol. Σε αντίθεση με FTP και SMB, το NFS από προεπιλογή δεν αυθεντικοποιεί χρήστες σε επίπεδο πρωτοκόλλου· εμπιστεύεται το UID που δηλώνει ο client. Οι επιλογές export παρακάτω εκμεταλλεύονται σκόπιμα αυτό το μοντέλο εμπιστοσύνης στο εργαστήριο.

### 7.1 Εγκατάσταση του NFS kernel server

Εγκαταστήστε το server package:

```bash
root@ubuntu-lab:~# apt install nfs-kernel-server -y
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following additional packages will be installed:
  keyutils libnfsidmap1 nfs-common rpcbind
Suggested packages:
  watchdog
The following NEW packages will be installed:
  keyutils libnfsidmap1 nfs-common nfs-kernel-server rpcbind
0 upgraded, 5 newly installed, 0 to remove and 0 not upgraded.
Need to get 1,048 kB of archives.
After this operation, 3,862 kB of additional disk space will be used.
```

Τα υποστηρικτικά πακέτα είναι σημαντικά. `nfs-common` provides client-side utilities, `libnfsidmap1` handles identity mapping, `keyutils` supports key management, and `rpcbind` provides the portmapper service on TCP/UDP port 111. Οι clients ερωτούν το portmapper για να εντοπίσουν τους NFS και mount daemons, επομένως μια ανοικτή θύρα 111 αποτελεί ισχυρή ένδειξη ότι η NFS υποδομή είναι προσβάσιμη στον host.

### 7.2 Δημιουργία του export directory

Δημιουργήστε έναν public export directory, κάντε τον world-writable και δημιουργήστε ένα marker file:

```bash
root@ubuntu-lab:~# mkdir -p /srv/nfs/public
root@ubuntu-lab:~# chmod 777 /srv/nfs/public
root@ubuntu-lab:~# cd /srv/nfs/public
root@ubuntu-lab:/srv/nfs/public# echo "Lab NFS retrieval marker" > data.txt
root@ubuntu-lab:/srv/nfs/public# ls -l
total 4
-rw-r--r-- 1 root root 25 Feb 10 12:20 data.txt
root@ubuntu-lab:/srv/nfs/public# cat data.txt
Lab NFS retrieval marker
root@ubuntu-lab:/srv/nfs/public# cd ~
```

Τα world-writable δικαιώματα `777` είναι επικίνδυνα από μόνα τους και γίνονται ιδιαίτερα επικίνδυνα όταν συνδυάζονται με την επιλογή export `no_root_squash` παρακάτω. Ο συνδυασμός αυτός επιτρέπει σε απομακρυσμένο root χρήστη να γράφει αρχεία που διατηρούν root ownership στον server, κάτι που αποτελεί κλασική επιφάνεια persistence και privilege escalation.

### 7.3 Ορισμός του insecure export

Ο έλεγχος πρόσβασης NFS βρίσκεται στο `/etc/exports`. Open it:

```bash
root@ubuntu-lab:~# nano /etc/exports
```

Προσθέστε την ακόλουθη export line μόνο για το εργαστήριο:

```text
/srv/nfs/public *(rw,sync,no_subtree_check,no_root_squash,insecure)
```

Οι επιλογές συνδυάζουν τέσσερις επικίνδυνες ρυθμίσεις. Το wildcard `*` εξάγει τον κατάλογο σε κάθε client address. Το `rw` παρέχει read και write access. Το `no_root_squash` απενεργοποιεί την προεπιλεγμένη προστασία που αντιστοιχίζει remote root requests στον μη προνομιούχο λογαριασμό `nobody`· με αυτό το flag, ένας client που λειτουργεί ως root μπορεί να ενεργεί ως root στα exported files. Το `insecure` επιτρέπει σε clients να συνδέονται από μη προνομιούχες source ports πάνω από 1024, αφαιρώντας την ιστορική απαίτηση οι NFS clients να χρησιμοποιούν privileged port.

Χρησιμοποιήστε κόμματα μεταξύ των επιλογών. Ένα κόμμα που λείπει ή ένα κατά λάθος κενό μπορεί να αλλάξει το parsing ή να εμποδίσει τη φόρτωση του export και η resulting αποτυχία μπορεί να μοιάζει με network problem ενώ στην πραγματικότητα είναι configuration error ενός χαρακτήρα.

### 7.4 Εφαρμογή και επαλήθευση του export

Επαναφορτώστε τον export table, ελέγξτε το effective export, κάντε restart στον NFS server και επαληθεύστε τους listeners:

```bash
root@ubuntu-lab:~# exportfs -a
root@ubuntu-lab:~# exportfs -v
/srv/nfs/public
		*(rw,wdelay,no_root_squash,no_subtree_check,sec=sys,rw,insecure,no_root_squash,no_all_squash)
root@ubuntu-lab:~# systemctl restart nfs-kernel-server
root@ubuntu-lab:~# ss -tlnp | grep -E ':(111|2049)'
LISTEN 0      4096               *:111             *:*    users:(("rpcbind",pid=2418,fd=4))
LISTEN 0      64                 *:2049            *:*    users:(("nfsd",pid=2431,fd=9))
```

Το `exportfs -a` εξάγει όλους τους καταχωρισμένους καταλόγους χωρίς να απαιτεί restart του daemon, ενώ το `exportfs -v` εμφανίζει τις effective options. Κενό output από το `exportfs -a` είναι φυσιολογικό. The `systemctl restart` cycles the kernel-server user-space components and picks up broader service changes. Ο τελικός έλεγχος επιβεβαιώνει τόσο το RPC portmapper στη θύρα 111 όσο και το NFS στη θύρα 2049· αν λείπει οποιοδήποτε από τα δύο, το enumeration θα αποτύχει πριν αποκτήσουν σημασία η αυθεντικοποίηση ή η συμπεριφορά UID.

---

## 8. Insecure NFS — enumeration, λήψη και mounting

Το NFS enumeration έχει τρία χρήσιμα επίπεδα: ανακάλυψη exports, απομακρυσμένο inspection ή download μεμονωμένων αρχείων και mount του export για πλήρη filesystem access. NetExec covers the first two without mounting anything. A traditional mount provides the third and exposes the full impact of `no_root_squash`.

### 8.1 Κλασική ανακάλυψη exports με `showmount` και `rpcinfo`

Πριν χρησιμοποιήσετε NetExec, επιβεβαιώστε την κλασική διαδρομή discovery από την πλευρά του επιτιθέμενου. `showmount -e` asks the target which directories it exports, and `rpcinfo -p` lists registered RPC services:

```bash
root@kali:~# showmount -e 192.168.1.9
Export list for 192.168.1.9:
/srv/nfs/public *
root@kali:~# rpcinfo -p 192.168.1.9 | head -12
   program vers proto   port  service
    100000    4   tcp    111  portmapper
    100000    3   tcp    111  portmapper
    100000    2   tcp    111  portmapper
    100000    4   udp    111  portmapper
    100005    3   tcp  45231  mountd
    100005    3   udp  42768  mountd
    100003    4   tcp   2049  nfs
    100003    3   tcp   2049  nfs
    100003    4   udp   2049  nfs
    100003    3   udp   2049  nfs
```

Η export list ονομάζει το ευάλωτο path και το wildcard client specification. The RPC table confirms the portmapper, mount daemon, and NFS services. Dynamic `mountd` ports are normal unless the administrator pins them; their presence explains why NFS firewalling is more involved than opening port 2049 alone.

### 8.2 Enumeration και listing NFS exports με NetExec

Το NFS module του NetExec παρέχει το ίδιο discovery workflow στο interface που χρησιμοποιήθηκε ήδη για SMB:

```bash
root@kali:~# nxc nfs 192.168.1.9 --enum-shares
[*] NFS         192.168.1.9    2049   UBUNTU-LAB     [*] Enumerating NFS exports
[+] NFS         192.168.1.9    2049   UBUNTU-LAB     /srv/nfs/public (rw, no_root_squash, root escape: True)
root@kali:~# nxc nfs 192.168.1.9 --share '/srv/nfs/public' --ls '/'
[*] NFS         192.168.1.9    2049   UBUNTU-LAB     [*] Listing / on /srv/nfs/public
[*] NFS         192.168.1.9    2049   UBUNTU-LAB     drwxrwxrwx root root 4096 .
[*] NFS         192.168.1.9    2049   UBUNTU-LAB     -rw-r--r-- root root   25 data.txt
```

Η πρώτη εντολή ανακαλύπτει το export και επισημαίνει τον επικίνδυνο συνδυασμό του: writable access και `no_root_squash`, συνοψισμένο ως `root escape: True`. Η δεύτερη εντολή κάνει listing της export root χωρίς local mount και αποκαλύπτει το `data.txt`. Το listing δεν απαιτεί credentials επειδή το ίδιο το export δεν επιβάλλει authentication check.

### 8.3 One-shot λήψη αρχείου με NetExec

Το NetExec μπορεί να κατεβάσει ένα αρχείο απευθείας από το export. The command takes the remote path relative to the share root and a local destination:

```bash
root@kali:~# nxc nfs 192.168.1.9 --share /srv/nfs/public/ --get-file data.txt data.txt
[*] NFS         192.168.1.9    2049   UBUNTU-LAB     [*] Downloading data.txt from /srv/nfs/public/
[+] NFS         192.168.1.9    2049   UBUNTU-LAB     File successfully downloaded to data.txt
root@kali:~# cat data.txt
Lab NFS retrieval marker
```

Αυτός είναι ο ταχύτερος τρόπος από το «το αρχείο υπάρχει» στο «το αρχείο βρίσκεται στον τοπικό δίσκο». No mount point is created, no persistent filesystem state is changed on the attacker, and no shell access to the target is required. For evidence handling, it is also the cleanest method: one remote file becomes one local file with no broader filesystem side effects.

### 8.4 Mounting του export με `mount -t nfs`

Ένα mount παρέχει πλήρη filesystem semantics: `ls`, `find`, `grep`, `cp`, permission inspection, and execution against the remote export as if it were a local directory. Create a mount point and mount the export:

```bash
root@kali:~# mkdir -p /tmp/nfs
root@kali:~# mount -t nfs 192.168.1.9:/srv/nfs/public /tmp/nfs
root@kali:~# ls -la /tmp/nfs
total 12
drwxrwxrwx  2 root root 4096 Feb 10 12:20 .
drwxrwxrwt 14 root root 4096 Feb 10 12:24 ..
-rw-r--r--  1 root root   25 Feb 10 12:20 data.txt
root@kali:~# cat /tmp/nfs/data.txt
Lab NFS retrieval marker
```

Ο mounted directory διατηρεί το server-side ownership και permissions, συμπεριλαμβανομένων του world-writable access και του root ownership. Λόγω του `no_root_squash`, οι λειτουργίες που εκτελούνται ως root μέσω αυτού του mount διατηρούν την root identity στα server-side files. Αυτή η συμπεριφορά μετατρέπει ένα απλό file share σε privilege-escalation primitive: τα αρχεία που γράφονται μέσω του mount μπορούν να έχουν ownership και permission bits που κανονικά απαιτούν local root access για να δημιουργηθούν.

Unmount the export when finished:

```bash
root@kali:~# umount /tmp/nfs
root@kali:~# ls /tmp/nfs
root@kali:~#
```

Ο καθαρισμός έχει σημασία σε ένα κοινόχρηστο εργαστήριο. A forgotten mount can confuse later tests, retain stale file handles after the export changes, and leave misleading directory contents in `/tmp`. Verify that the mount point is empty after unmounting.

---

## 9. Σύνοψη hardening

Το εργαστήριο έχει πλέον ελεγχθεί end to end και κάθε έκθεση πρωτοκόλλου προήλθε από έναν μικρό αριθμό επιλογών ρύθμισης. Η ενότητα αυτή συγκεντρώνει τα αντίστοιχα controls σε ένα σημείο.

### 9.1 FTP hardening

**Απενεργοποιήστε την anonymous πρόσβαση.** Set `anonymous_enable=NO` in `/etc/vsftpd.conf` unless a documented business case justifies anonymous retrieval. Even then, prefer SFTP over SSH for file transfers across untrusted networks, because SFTP provides authentication and encryption by default.

**Επιβάλετε TLS όπου το FTP πρέπει να παραμείνει.** Set `ssl_enable=YES` with `force_local_logins_ssl=YES` and `force_local_data_ssl=YES` so credentials and transferred content do not traverse the network in cleartext. Certificates, client compatibility, and passive-port firewall rules must be managed as part of the same deployment.

**Περιορίστε την έκθεση στο δίκτυο.** Bind `vsftpd` to internal interfaces where possible, limit port 21 to known management subnets with a host firewall, and open only the configured passive-port range. A share that cannot be reached from an untrusted segment cannot be enumerated from that segment.

### 9.2 Samba hardening

**Απενεργοποιήστε την guest πρόσβαση.** Remove `guest ok = yes` and `public = yes` from every share, set `map to guest = Never` in `[global]`, and require valid Linux accounts for every connection. Anonymous SMB access should be exceptional, documented, and regularly reviewed.

**Απενεργοποιήστε το SMB1.** Set `min protocol = SMB2` in `[global]` to prevent fallback to the obsolete SMB1 dialect. Modern clients negotiate SMB2 or SMB3 by default, so this control rarely breaks legitimate access while removing a historically fragile protocol path.

**Περιορίστε τα share paths.** Never publish `/var/www`, `/etc`, home directories, backup roots, or any path that backs onto another service. Use isolated share roots such as `/srv/samba/<sharename>` so that an access-control mistake has a smaller blast radius. Pair that layout with filesystem permissions that deny writes unless writes are explicitly required.

### 9.3 NFS hardening

**Ενεργοποιήστε το `root_squash`.** Αφαιρέστε το `no_root_squash` από κάθε export line. The default `root_squash` behaviour maps remote root requests to the unprivileged `nobody` account and removes the file-ownership persistence vector demonstrated in section 8.

**Περιορίστε τα wildcards των exports.** Αντικαταστήστε το `*` με explicit client addresses ή CIDR ranges, ώστε μόνο γνωστοί hosts να μπορούν να κάνουν mount το export. A backup server, application host, or management workstation should be named individually; an entire untrusted subnet should not be trusted by default.

**Χρησιμοποιήστε αυθεντικοποίηση NFS όταν το δίκτυο δεν είναι έμπιστο.** Χρησιμοποιήστε NFSv4 με Kerberos security, όπως `sec=krb5p`, όταν απαιτούνται αυθεντικοποίηση και κρυπτογράφηση. Χωρίς Kerberos, το default UID-trust model του NFS είναι ακατάλληλο για hostile ή zero-trust networks.

**Εφαρμόστε firewall στις RPC και NFS θύρες.** Αποκλείστε TCP/UDP port 111 για `rpcbind`, TCP/UDP port 2049 για NFS και τη mount-daemon port για κάθε μη έμπιστη πηγή. Pin dynamic RPC services to stable ports where firewall policy requires it, and verify exposure from an untrusted network segment rather than only from localhost.

### 9.4 Ανίχνευση και auditing

**Ελέγχετε απευθείας τα αρχεία ρυθμίσεων.** Κάθε βασική λανθασμένη ρύθμιση αυτού του walkthrough είναι ορατή με μία αναζήτηση ανά αρχείο. Useful starting checks include `anonymous_enable`, `guest ok`, `public`, `map to guest`, `min protocol`, `no_root_squash`, export wildcards, and world-writable export directories.

**Κάντε scan από την οπτική του επιτιθέμενου.** Εκτελείτε περιοδικά probes για τις θύρες 21, 111, 139, 445 και 2049 από μη έμπιστα segments. Μια ανοικτή θύρα είναι μόνο η αρχή· ακολουθήστε την με τους ίδιους anonymous και guest ελέγχους που χρησιμοποιούνται στις ενότητες 4, 6 και 8.

**Ελέγχετε τα inventories των shares και τα logs.** Διατηρείτε τα FTP, Samba και NFS logs, δημιουργήστε alerts για anonymous ή guest authentication και ελέγχετε τα inventories των exports και shares μετά από κάθε αλλαγή ρύθμισης. A quarterly audit of `/etc/vsftpd.conf`, `/etc/samba/smb.conf`, and `/etc/exports`, paired with network-level scans, closes every primitive demonstrated in this guide.

### 9.5 Τελική ανάλυση

Τα FTP, SMB και NFS αποτελούν συλλογικά βασική υποδομή για μεγάλο μέρος του enterprise file sharing. Each protocol ships with conservative defaults, and each is routinely weakened for convenience. This lab reproduced the resulting misconfigurations: anonymous FTP with masked ownership, guest-accessible Samba shares over sensitive paths, and NFS exports with `no_root_squash` and wildcard clients. The matching enumeration workflows collapse discovery and exploitation into a small number of commands.

Οι defenders κερδίζουν αντιμετωπίζοντας τη ρύθμιση των file shares ως security-critical infrastructure και όχι ως βοηθητική υποδομή. Κάθε διόρθωση είναι σύντομη, κάθε έκθεση είναι ανιχνεύσιμη και το ίδιο εργαστήριο υποστηρίζει και τις δύο πλευρές της διαδικασίας: τον operator που εξασκεί το attack chain και τον defender που επαληθεύει ότι τα controls πράγματι το εντοπίζουν.

---

## 10. Cheat sheet γρήγορης αναφοράς

| Φάση | Εντολή | Τι κάνει |
| --- | --- | --- |
| Ρύθμιση | `apt update` | refreshes the package catalogue before installation |
| Ρύθμιση | `apt install vsftpd` | installs the FTP daemon |
| Ρύθμιση | `apt install samba` | installs Samba and its supporting libraries |
| Ρύθμιση | `apt install nfs-kernel-server -y` | installs the NFS server, client libraries, and `rpcbind` |
| Ρύθμιση | `ss -tlnp \| grep ':21'` | confirms that FTP is listening on port 21 |
| Ρύθμιση | `ss -tlnp \| grep -E ':(139\|445)'` | confirms that SMB is listening on ports 139 and 445 |
| Ρύθμιση | `ss -tlnp \| grep -E ':(111\|2049)'` | confirms that RPC and NFS listeners are present |
| FTP ρυθμίσεις | `nano /etc/vsftpd.conf` | edits the FTP server configuration |
| FTP ρυθμίσεις | `anonymous_enable=YES` | enables passwordless anonymous FTP logins |
| FTP ρυθμίσεις | `anon_root=/var/ftp/` | roots anonymous sessions in `/var/ftp/` |
| FTP ρυθμίσεις | `no_anon_password=YES` | removes the anonymous password prompt |
| FTP ρυθμίσεις | `hide_ids=YES` | masks UID/GID ownership in FTP listings |
| FTP ρυθμίσεις | `pasv_min_port=40000` and `pasv_max_port=50000` | constrains passive-mode data ports |
| FTP ρυθμίσεις | `service vsftpd restart` | applies FTP configuration changes |
| FTP αρχεία | `mkdir -p /var/ftp/pub` | creates the conventional anonymous subdirectory |
| FTP αρχεία | `chown nobody:nogroup /var/ftp/pub` | assigns the conventional unprivileged owner |
| FTP αρχεία | `echo "Lab FTP retrieval marker" > /var/ftp/pub/note.txt` | seeds a verifiable FTP marker |
| FTP αναγνώριση | `nmap -A -p 21 <target>` | confirms FTP version and anonymous access |
| FTP πρόσβαση | `ftp <target>` then `anonymous` | opens a passwordless FTP session |
| FTP πρόσβαση | `ls`, `cd pub`, `ls`, `get note.txt`, `bye` | browses the anonymous share and retrieves a file |
| SMB ρυθμίσεις | `nano /etc/samba/smb.conf` | edits the Samba configuration |
| SMB ρυθμίσεις | `testparm -s` | validates the effective Samba configuration |
| SMB ρυθμίσεις | `guest ok = yes` | maps unauthenticated clients to guest access |
| SMB ρυθμίσεις | `public = yes` | legacy synonym for guest-accessible intent |
| SMB ρυθμίσεις | `browsable = yes` | makes the share visible during enumeration |
| SMB ρυθμίσεις | `read only = no` and `writable = yes` | makes the share writable |
| SMB ρυθμίσεις | `systemctl restart smbd` | applies Samba configuration changes |
| SMB αρχεία | `echo "Lab SMB retrieval marker" > /var/www/file.txt` | seeds a verifiable SMB marker |
| SMB αναγνώριση | `nxc smb <target> --shares -u 'guest' -p ''` | enumerates shares as guest |
| SMB πρόσβαση | `smbclient -N -L //<target>` | lists shares without a password |
| SMB πρόσβαση | `smbclient //<target>/shares -N` | opens the guest-accessible share |
| SMB πρόσβαση | `ls`, `get file.txt`, `exit` | lists and retrieves a file over SMB |
| NFS ρυθμίσεις | `nano /etc/exports` | edits the NFS export table |
| NFS ρυθμίσεις | `/srv/nfs/public *(rw,sync,no_subtree_check,no_root_squash,insecure)` | publishes an insecure lab-only export |
| NFS ρυθμίσεις | `exportfs -a` | reloads every export-table entry |
| NFS ρυθμίσεις | `exportfs -v` | displays effective export options |
| NFS ρυθμίσεις | `systemctl restart nfs-kernel-server` | restarts NFS user-space services |
| NFS αρχεία | `mkdir -p /srv/nfs/public && chmod 777 /srv/nfs/public` | creates a world-writable export directory |
| NFS αρχεία | `echo "Lab NFS retrieval marker" > /srv/nfs/public/data.txt` | seeds a verifiable NFS marker |
| NFS αναγνώριση | `showmount -e <target>` | lists directories exported to the network |
| NFS αναγνώριση | `rpcinfo -p <target>` | lists registered RPC services |
| NFS αναγνώριση | `nxc nfs <target> --enum-shares` | discovers NFS exports with NetExec |
| NFS αναγνώριση | `nxc nfs <target> --share '/srv/nfs/public' --ls '/'` | lists an export root without mounting |
| NFS πρόσβαση | `nxc nfs <target> --share /srv/nfs/public/ --get-file data.txt data.txt` | downloads one NFS file without mounting |
| NFS πρόσβαση | `mount -t nfs <target>:/srv/nfs/public /tmp/nfs` | mounts the export locally |
| NFS πρόσβαση | `ls -la /tmp/nfs` and `cat /tmp/nfs/data.txt` | inspects and reads the mounted export |
| NFS πρόσβαση | `umount /tmp/nfs` | removes the local NFS mount |
| Άμυνα | `grep -nE 'anonymous_enable\|guest ok\|public =\|map to guest\|min protocol' /etc/vsftpd.conf /etc/samba/smb.conf` | finds risky FTP/SMB directives |
| Άμυνα | `grep -nE 'no_root_squash\|\*\(\|insecure' /etc/exports` | finds risky NFS export options |
| Άμυνα | `nmap -sV -p 21,111,139,445,2049 <target>` | checks file-sharing exposure from the network |

---

## 11. Ασκήσεις εξάσκησης

Η ακόλουθη σειρά έχει σχεδιαστεί για εκτέλεση σε απομονωμένο εργαστήριο με δύο εικονικές μηχανές και χωρίς διαδρομή προς το Internet. Κάθε άσκηση βασίζεται στην προηγούμενη και αρκετές έχουν ως στόχο να δείξουν πώς μία αμυντική αλλαγή μεταβάλλει την έξοδο από την πλευρά του επιτιθέμενου.

1. **Build the range.** Install `vsftpd`, Samba, and `nfs-kernel-server` on the Ubuntu target. Use `ss -tlnp` to confirm listeners on ports 21, 139, 445, 111, and 2049. Explain which process owns each listener and why the RPC portmapper must be included in the check.
2. **Prove anonymous FTP.** Run `nmap -A -p 21` against the target and identify the script output that proves anonymous access. Then open an FTP session as `anonymous`, retrieve `note.txt`, and compare its bytes with the marker created on the server.
3. **Disable anonymous FTP.** Set `anonymous_enable=NO`, restart `vsftpd`, and repeat both the Nmap check and the interactive login. Record the exact changed output and explain which server-side setting produced it.
4. **Map guest SMB.** Enumerate shares as `guest` with NetExec and then with `smbclient -N -L`. Compare the two inventories and identify every share that should not normally be available to an unauthenticated client.
5. **Retrieve over SMB.** Connect to the `shares` export with `smbclient`, list its contents, retrieve `file.txt`, and verify its contents locally. Describe what the SMB file attributes reveal about files versus directories.
6. **Remove guest access.** Remove `guest ok` and `public` from the lab share, set `map to guest = Never`, validate with `testparm`, restart Samba, and rerun both enumeration commands. Document which behaviour changes first from the attacker’s perspective.
7. **Discover NFS classically.** Use `showmount -e` and `rpcinfo -p` to identify the exported path and the RPC services supporting it. Explain why a firewall rule for port 2049 alone may be insufficient.
8. **Enumerate NFS with NetExec.** Run `--enum-shares` and then list the export root. Identify the dangerous option combination in the output and explain what `root escape: True` means for file ownership.
9. **Download without mounting.** Retrieve `data.txt` with NetExec `--get-file`, then mount the export and read the same file through `/tmp/nfs`. Compare the two retrieval methods for speed, forensic footprint, and available filesystem operations.
10. **Restore safe NFS defaults.** Replace the wildcard client with an explicit lab address, remove `no_root_squash` and `insecure`, reload with `exportfs -a`, and rerun NetExec enumeration and the mount test. Record which operations still work and which now fail.
11. **Audit all three services.** Write one `grep` check for each configuration file that would detect the corresponding lab misconfiguration. Then run a single Nmap command covering all relevant file-sharing ports and explain what the scan can and cannot prove about authorisation.
12. **Write the report.** Summarise the whole chain as a five-hundred-word finding: the initial weaknesses, the protocol-by-protocol exploitation path, the business impact, and three controls that would each have independently prevented unauthenticated retrieval.

Working through those twelve exercises end to end will have taken you through every command in this guide at least once and, more usefully, through the reasoning behind each of them — which is the part that transfers to a different target, a different service and a different engagement.
