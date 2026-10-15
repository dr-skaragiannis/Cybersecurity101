# Ανώνυμες συνδέσεις για Pentesters: Αναλυτική παρουσίαση κοινόχρηστων αρχείων FTP, SMB και NFS

## Εισαγωγή

Τα πρωτόκολλα διαμοιρασμού αρχείων τροφοδοτούν την εσωτερική λειτουργία σχεδόν κάθε οργανισμού: FTP για τα παραδοσιακά σημεία υποβολής αρχείων, SMB για file servers συμβατούς με Windows και NFS για εξαγωγές Unix προς Unix. Κάθε πρωτόκολλο διαθέτει επίσης λειτουργία ανώνυμης ή guest πρόσβασης, και οι διαχειριστές συχνά ενεργοποιούν μία από αυτές για λόγους ευκολίας και ξεχνούν να την περιορίσουν.

Γι' αυτό τα λανθασμένα ρυθμισμένα κοινόχρηστα αρχεία εμφανίζονται ξανά και ξανά ως ευρήματα υψηλού αντίκτυπου σε εσωτερικούς ελέγχους: η πρόσβαση χωρίς αυθεντικοποίηση μπορεί να εκθέσει πηγαίο κώδικα, αρχεία αντιγράφων ασφαλείας, αρχεία ρυθμίσεων, διαπιστευτήρια και προσωπικά δεδομένα — πράγματα που κανένας περιμετρικός έλεγχος δεν μπορεί να ανακτήσει εκ των υστέρων.

Αυτός ο οδηγός δημιουργεί σκόπιμα ένα ευάλωτο εργαστηριακό περιβάλλον σε Ubuntu, εκθέτει και τις τρεις υπηρεσίες σε πρόσβαση χωρίς αυθεντικοποίηση και στη συνέχεια κάνει enumeration από ένα μηχάνημα επιτιθέμενου με Nmap, τον τυπικό FTP client, το `smbclient` και το NetExec.

<details>
<summary><strong>Περισσότερα</strong> — η ροή εργασίας που επαναλαμβάνεται σε κάθε ενότητα πρωτοκόλλου</summary>

Η ροή εργασίας είναι σκόπιμα επαναληπτική: επιβεβαίωση της έκθεσης, καταγραφή όσων είναι διαθέσιμα, ανάκτηση ενός marker file και στη συνέχεια κατανόηση της ρύθμισης στον server που ευθύνεται για την έκθεση. Κάθε ενότητα πρωτοκόλλου συνδυάζει επομένως τα αποτελέσματα της επίθεσης με το αμυντικό μέτρο που θα την είχε αποτρέψει, ώστε να συναντάτε πάντα την επίθεση και τη διόρθωσή της δίπλα-δίπλα.

Η επανάληψη είναι σκόπιμη ως παιδαγωγική μέθοδος: μέχρι το τρίτο πρωτόκολλο, ο ρυθμός του enumeration — επιβεβαίωση, καταγραφή, ανάκτηση, έλεγχος της ρύθμισης — γίνεται μυϊκή μνήμη, και αυτό που αλλάζει είναι μόνο τα εργαλεία και το υποκείμενο μοντέλο εμπιστοσύνης.

</details>

**Τοπολογία εργαστηρίου.** Οι ασκήσεις προϋποθέτουν δύο εικονικές μηχανές σε ένα απομονωμένο δίκτυο host-only. Ο στόχος εκτελεί Ubuntu 22.04 LTS με εγκατεστημένα τα `vsftpd`, Samba και NFS kernel server. Ο επιτιθέμενος είναι μια security distribution της οικογένειας Debian με εγκατεστημένα Nmap, FTP και SMB clients, NetExec και εργαλεία NFS client. Οι διευθύνσεις IP που χρησιμοποιούνται σε όλο τον οδηγό είναι το `192.168.1.9` για τον στόχο και το `192.168.1.17` για τον επιτιθέμενο.

| Ρόλος | Λειτουργικό σύστημα | Διεύθυνση | Σημειώσεις |
| --- | --- | --- | --- |
| Στόχος | Διακομιστής Ubuntu 22.04 LTS | `192.168.1.9` | `vsftpd`, Samba, NFS kernel server, hostname `ubuntu-lab` |
| Επιτιθέμενος | Διανομή ασφαλείας με βάση τη Debian | `192.168.1.17` | nmap, `ftp`, `smbclient`, NetExec, εργαλεία NFS |
| Δίκτυο | Τμήμα εργαστηρίου host-only / NAT | `192.168.1.0/24` | Χωρίς διαδρομή προς το Internet από το εργαστήριο |

> **Εξουσιοδότηση και εύρος.** Όλα όσα περιγράφονται εδώ αποτελούν τεχνικές επίθεσης. Εκτελέστε τα μόνο σε μηχανήματα που σας ανήκουν ή για τα οποία διαθέτετε γραπτή εξουσιοδότηση — ένα ειδικά σχεδιασμένο εικονικό εργαστήριο είναι ο κατάλληλος χώρος για εξάσκηση και ο μόνος χώρος όπου αυτές οι εντολές είναι ασφαλείς. Το enumeration χωρίς αυθεντικοποίηση, η πρόσβαση σε guest shares, η ανάκτηση αρχείων, το NFS mounting και οι δοκιμές privilege escalation είναι σαφώς ανιχνεύσιμες και επιθετικές ενέργειες σε δίκτυο παραγωγής, ενώ η εκτέλεσή τους χωρίς άδεια αποτελεί ποινικό αδίκημα στις περισσότερες δικαιοδοσίες. Τα αμυντικά μέτρα παρατίθενται δίπλα σε κάθε τεχνική, ώστε το υλικό να μπορεί να μελετηθεί και από την πλευρά της άμυνας.

### Πώς να διαβάζετε τα transcripts

Οι συνεδρίες τερματικού στον οδηγό ακολουθούν τις συμβάσεις του ίδιου του shell και ορισμένα σημεία αξίζει να είναι γνωστά πριν από την πρώτη εντολή:

* Ένα prompt που τελειώνει σε `#` σημαίνει ότι το shell εκτελείται ως λογαριασμός root (`root@kali:~#`), ενώ ένα prompt που τελειώνει σε `$` σημαίνει ότι πρόκειται για απλό χρήστη. Τόσο η ρύθμιση του στόχου όσο και το enumeration από την πλευρά του επιτιθέμενου παρουσιάζονται ως root, επειδή η εγκατάσταση πακέτων, η διαχείριση υπηρεσιών, το NFS mounting και το raw-socket scanning απαιτούν προνόμια.
* Το transcript `root@ubuntu-lab:~#` είναι το μηχάνημα-στόχος κατά τη δημιουργία του εργαστηρίου. Το transcript `root@kali:~#` είναι το μηχάνημα του επιτιθέμενου κατά το enumeration.
* Η έξοδος που παρουσιάζεται εδώ καταγράφηκε σε συστήματα της οικογένειας Debian και είναι ενδεικτική και όχι byte-identical: οι εκδόσεις πακέτων, οι αριθμοί θυρών, οι χρονικές σφραγίδες, οι διευθύνσεις MAC, οι ημερομηνίες καταλόγων και τα banners των εργαλείων θα διαφέρουν στο δικό σας εργαστήριο. Αυτό που έχει σημασία είναι η δομή της εξόδου.
* Τα περιεχόμενα αρχείων που εμφανίζονται χωρίς prompt είναι τα κυριολεκτικά bytes του αρχείου, όπως θα τα δείτε σε έναν editor.
* Όταν μια εντολή είναι αρκετά μεγάλη ώστε να σπάει σε δύο γραμμές στην τεκμηρίωση εργαλείων τρίτων, εδώ εμφανίζεται πάντα ως μία ενιαία γραμμή που μπορείτε να κάνετε paste.

### Πίνακας περιεχομένων

1. [Γιατί έχουν σημασία τα file shares χωρίς αυθεντικοποίηση](#1-γιατί-έχουν-σημασία-τα-file-shares-χωρίς-αυθεντικοποίηση)
2. [Ρύθμιση εργαστηρίου — στόχος, επιτιθέμενος και δίκτυο](#2-ρύθμιση-εργαστηρίου--στόχος-επιτιθέμενος-και-δίκτυο)
3. [Anonymous FTP — ρύθμιση του vsftpd](#3-anonymous-ftp--ρύθμιση-του-vsftpd)
4. [Anonymous FTP — enumeration και ανάκτηση](#4-anonymous-ftp--enumeration-και-ανάκτηση)
5. [Guest SMB — ρύθμιση του Samba](#5-guest-smb--ρύθμιση-του-samba)
6. [Guest SMB — enumeration και ανάκτηση](#6-guest-smb--enumeration-και-ανάκτηση)
7. [Insecure NFS — ρύθμιση των exports](#7-insecure-nfs--ρύθμιση-των-exports)
8. [Insecure NFS — enumeration, λήψη και mounting](#8-insecure-nfs--enumeration-λήψη-και-mounting)
9. [Σύνοψη hardening](#9-σύνοψη-hardening)
10. [Cheat sheet γρήγορης αναφοράς](#10-cheat-sheet-γρήγορης-αναφοράς)
11. [Ασκήσεις εξάσκησης](#11-ασκήσεις-εξάσκησης)

---

## 1. Γιατί έχουν σημασία τα file shares χωρίς αυθεντικοποίηση

Τα τρία πρωτόκολλα αυτού του οδηγού εκθέτουν δεδομένα με τρεις διαφορετικούς τρόπους, αλλά το βασικό σφάλμα είναι το ίδιο: ένας server εμπιστεύεται έναν network client χωρίς να επαληθεύει την ταυτότητά του.

<details>
<summary><strong>Περισσότερα</strong> — πώς εκθέτει δεδομένα κάθε πρωτόκολλο</summary>

Το FTP εκθέτει έναν ανώνυμο ριζικό κατάλογο μέσω του μη προνομιούχου λογαριασμού FTP. Το SMB εκθέτει shares σημειωμένα ως προσβάσιμα από guest. Το NFS εκθέτει τις exported διαδρομές εμπιστευόμενο το UID που δηλώνει ο client. Κάθε μία από αυτές τις συμπεριφορές μπορεί να είναι χρήσιμη σε ένα στενά ελεγχόμενο περιβάλλον, και καθεμία γίνεται εύρημα μόλις είναι προσβάσιμη από ένα μη έμπιστο τμήμα δικτύου.

</details>

Η επιχειρησιακή συνέπεια είναι ότι το file-share enumeration πρέπει να πραγματοποιείται νωρίς σε έναν εσωτερικό έλεγχο: πριν από το password guessing, πριν από το exploitation και πριν από το post-exploitation, ο επιχειρών πρέπει να ρωτά αν κάποιο μηχάνημα απλώς χαρίζει τα αρχεία του.

<details>
<summary><strong>Περισσότερα</strong> — γιατί αποδίδει γρηγορότερα από ένα brute-force</summary>

Οι εντολές είναι γρήγορες, τα στοιχεία είναι σαφή, και τα αρχεία που ανακτώνται συχνά περιέχουν τα διαπιστευτήρια ή τις λεπτομέρειες ρυθμίσεων που απαιτούνται για την επόμενη φάση. Ένα μόνο αντίγραφο ασφαλείας ή ένα deployment script προσβάσιμο από όλους μπορεί λοιπόν να περικόψει περισσότερο ένα engagement από ό,τι ένα μεγάλο brute-force.

</details>

Είναι επίσης χρήσιμο να διακρίνουμε τη διαθεσιμότητα από την εξουσιοδότηση. Ένα share μπορεί σκόπιμα να είναι διαθέσιμο και ταυτόχρονα να απαιτεί αυθεντικοποίηση· αυτό είναι φυσιολογικό.

<details>
<summary><strong>Περισσότερα</strong> — το ερώτημα που αποφασίζει αν πρόκειται για εύρημα</summary>

Η ευπάθεια εδώ δεν είναι η ύπαρξη FTP, SMB ή NFS, αλλά το ότι μια anonymous ή guest ταυτότητα μπορεί να περιηγηθεί ή να ανακτήσει δεδομένα που προορίζονται για αυθεντικοποιημένους χρήστες. Σε όλο αυτό το walkthrough, το κρίσιμο ερώτημα δεν είναι λοιπόν «είναι ανοιχτή η θύρα;» αλλά «τι μπορεί να κάνει ένας client χωρίς αυθεντικοποίηση μετά τη σύνδεση;» Μια ανοιχτή θύρα είναι μια παρατήρηση· ένα listing ή μια λήψη χωρίς αυθεντικοποίηση είναι εύρημα.

</details>

---

## 2. Ρύθμιση εργαστηρίου — στόχος, επιτιθέμενος και δίκτυο

Πριν ξεκινήσει ο έλεγχος, και οι δύο εικονικές μηχανές πρέπει να μπορούν να επικοινωνούν μεταξύ τους και με τίποτα άλλο. Τοποθετήστε τις στο ίδιο απομονωμένο τμήμα εργαστηρίου host-only ή NAT, ορίστε στον στόχο `192.168.1.9` και στον επιτιθέμενο `192.168.1.17` και δημιουργήστε snapshot πριν εγκαταστήσετε τις ευάλωτες υπηρεσίες.

<details>
<summary><strong>Περισσότερα</strong> — γιατί μετράει το snapshot</summary>

Αυτός ο οδηγός αποδυναμώνει σκόπιμα τις προεπιλογές, οπότε θέλετε ένα καθαρό σημείο επαναφοράς πριν και μετά τη φάση της επίθεσης. Το snapshot είναι επίσης η ταχύτερη επαναφορά όταν ένα βήμα hardening της ενότητας 9 τυχόν σας κλειδώσει έξω από το εργαστήριο — επαναφορά, επανεφαρμογή μόνο της ρύθμισης που αλλάξατε και συνέχεια.

</details>

Ο στόχος χρειάζεται τρία πακέτα server. Ενημερώστε πρώτα τον κατάλογο πακέτων:

```bash
apt update
```

**Έξοδος:**

```text
Hit:1 http://archive.ubuntu.com/ubuntu jammy InRelease
Hit:2 http://security.ubuntu.com/ubuntu jammy-security InRelease
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
All packages are up to date.
```

Εγκαταστήστε τους FTP, SMB και NFS servers στις αντίστοιχες ενότητες πρωτοκόλλου παρακάτω και όχι όλους μαζί. Αυτή η σειρά διατηρεί κάθε λανθασμένη ρύθμιση συνδεδεμένη με την υπηρεσία που την εισήγαγε και διευκολύνει την ερμηνεία των μεταγενέστερων ελέγχων hardening.

Ο επιτιθέμενος χρειάζεται τους αντίστοιχους clients και τα εργαλεία enumeration. Σε μια security distribution της οικογένειας Debian αυτά είναι συνήθως ήδη εγκατεστημένα· σε ένα ελάχιστο σύστημα, εγκαταστήστε Nmap, έναν FTP client γραμμής εντολών, εργαλεία Samba client, NetExec και εργαλεία NFS client.

<details>
<summary><strong>Περισσότερα</strong> — τι χρησιμοποιείται κάθε εργαλείο από την πλευρά του επιτιθέμενου</summary>

* Το Nmap επιβεβαιώνει ποια υπηρεσία ανήκει σε κάθε θύρα και, με τα scripts του, αν γίνεται δεκτή η anonymous πρόσβαση.
* Ένας FTP client γραμμής εντολών (`ftp`) διαχειρίζεται τη διαδραστική anonymous session και ανακτά αρχεία.
* Το `smbclient` και το NetExec (`nxc`) κάνουν enumeration και περιήγηση σε SMB shares· το NetExec μιλάει επίσης NFS.
* Τα εργαλεία NFS client (`showmount`, `rpcinfo`, `mount`) εντοπίζουν και κάνουν mount exported καταλόγους.

</details>

Επαληθέψτε πρώτα τη βασική συνδεσιμότητα, επειδή κάθε μεταγενέστερη αποτυχία διαγιγνώσκεται ευκολότερα όταν είναι γνωστό ότι η διαδρομή δικτύου λειτουργεί:

```bash
ping -c 2 192.168.1.9
```

**Έξοδος:**

```text
PING 192.168.1.9 (192.168.1.9) 56(84) bytes of data.
64 bytes from 192.168.1.9: icmp_seq=1 ttl=64 time=0.52 ms
64 bytes from 192.168.1.9: icmp_seq=2 ttl=64 time=0.48 ms

--- 192.168.1.9 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.480/0.500/0.520/0.020 ms
```

Χρόνος round-trip κάτω από ένα millisecond είναι συνήθης ένδειξη δύο εικονικών μηχανών στον ίδιο host ή στο ίδιο τμήμα εργαστηρίου.

<details>
<summary><strong>Περισσότερα</strong> — διορθώστε το δίκτυο πριν από οτιδήποτε άλλο</summary>

Αν το ping αποτύχει, σταματήστε και διορθώστε το virtual networking πριν συνεχίσετε· ένας μη προσβάσιμος στόχος θα κάνει κάθε σφάλμα συγκεκριμένης υπηρεσίας παραπλανητικό. Μια απόρριψη σύνδεσης στη θύρα 21 αργότερα μπορεί να σημαίνει «ο vsftpd δεν εκτελείται» — αλλά μόνο αν το ίδιο το δίκτυο είναι γνωστά υγιές, και αυτό ακριβώς επιβεβαιώνει αυτός ο έλεγχος.

</details>

---

## 3. Anonymous FTP — ρύθμιση του vsftpd

Το `vsftpd`, το Very Secure FTP Daemon, είναι ο προεπιλεγμένος FTP server σε πολλά συστήματα που προέρχονται από τη Debian. Από προεπιλογή ακούει στη TCP θύρα 21, υποστηρίζει συνδέσεις τοπικών χρηστών και απενεργοποιεί την anonymous πρόσβαση. Το εργαστήριο αντιστρέφει αυτή την τελευταία προεπιλογή και ρυθμίζει ένα passive anonymous share που μπορεί να διαβαστεί από οποιονδήποτε γείτονα του δικτύου.

### 3.1 Εγκατάσταση του vsftpd

Εγκαταστήστε το πακέτο του daemon:

```bash
apt install vsftpd
```

**Έξοδος:**

```text
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

Ένα πακέτο, ένας daemon, και η εγκατάσταση τελείωσε — το `vsftpd` είναι σκόπιμα μικρό.

<details>
<summary><strong>Περισσότερα</strong> — γιατί το μικρό footprint κάνει τον έλεγχο σημαντικό</summary>

Το footprint ενός μόνο πακέτου είναι ένας λόγος για τον οποίο οι ομάδες επιλέγουν το `vsftpd`: ο daemon έχει λίγες εξαρτήσεις και συντηρητικές προεπιλογές. Αυτή η απλότητα είναι επίσης ο λόγος που ο έλεγχος των ρυθμίσεων έχει σημασία. Δεν υπάρχουν πολλά προς έλεγχο, οπότε ο auditor πρέπει πραγματικά να ελέγξει τα πάντα — ένα σύντομο αρχείο ρυθμίσεων δεν αφήνει δικαιολογία για δειγματοληψία αντί για ανάγνωση.

</details>

### 3.2 Έλεγχος της προεπιλεγμένης ρύθμισης

Η ρύθμιση του server βρίσκεται στο `/etc/vsftpd.conf`. Πριν αλλάξετε οτιδήποτε, ελέγξτε τις directives που ελέγχουν τους listeners και την anonymous πρόσβαση:

```bash
grep -nE '^(#)?(listen|listen_ipv6|anonymous_enable|local_enable)=' /etc/vsftpd.conf
```

**Έξοδος:**

```text
12:listen=NO
18:listen_ipv6=YES
27:#anonymous_enable=NO
29:local_enable=YES
```

Η σημαντική γραμμή είναι η προεπιλογή της anonymous πρόσβασης: με απενεργοποιημένη την anonymous πρόσβαση, κανένα από τα βήματα FTP enumeration της ενότητας 4 δεν θα λειτουργούσε. Σημειώστε ότι το `#anonymous_enable=NO` είναι σχολιασμένο — η ισχύουσα τιμή προέρχεται από την ενσωματωμένη προεπιλογή, που είναι επίσης «όχι».

<details>
<summary><strong>Περισσότερα</strong> — η ανάγνωση των ρυθμίσεων listener και η καταγραφή της κατάστασης «πριν»</summary>

Οι ρυθμίσεις listener καθορίζουν αν ο daemon ακούει σε IPv4, IPv6 ή και στα δύο· η πακεταρισμένη προεπιλογή του Ubuntu χρησιμοποιεί τον IPv6 listener ενώ εξακολουθεί να δέχεται IPv4 συνδέσεις μέσω του dual-stack socket. Καταγράψτε τώρα την κατάσταση «πριν», επειδή η ενότητα 9 αναιρεί την αλλαγή του εργαστηρίου επαναφέροντας μια περιοριστική τιμή — η σύγκριση με μια καταγεγραμμένη βάση αναφοράς είναι ο τρόπος με τον οποίο αποδεικνύετε ότι το σύστημα επέστρεψε στη ρύθμιση με την οποία κυκλοφόρησε.

</details>

### 3.3 Ενεργοποίηση της anonymous πρόσβασης

Ανοίξτε το αρχείο ρυθμίσεων:

```bash
nano /etc/vsftpd.conf
```

Αλλάξτε την εντολή anonymous-access σε ενεργή τιμή `YES`:

```text
anonymous_enable=YES
```

Αφήστε το `local_enable=YES` αμετάβλητο για αυτό το εργαστήριο. Ο τελικός συνδυασμός δέχεται τόσο τους τοπικούς Linux χρήστες με τους κανονικούς τους κωδικούς όσο και τον anonymous χρήστη χωρίς κωδικό.

<details>
<summary><strong>Περισσότερα</strong> — γιατί ο συνδυασμός αυτός είναι επικίνδυνος σε production</summary>

Σε production, ο συνδυασμός αυτός αποτελεί κρίσιμη έκθεση όταν η θύρα 21 είναι προσβάσιμη: οποιοσδήποτε φτάνει στη θύρα FTP μπορεί να κάνει listing και download ό,τι μπορεί να διαβάσει ο λογαριασμός FTP. Οι τοπικοί χρήστες εξακολουθούν να έχουν τους κανονικούς λογαριασμούς τους, ενώ οι ανώνυμοι επισκέπτες παίρνουν μια δεύτερη πόρτα χωρίς κωδικό στον ίδιο daemon — μια πόρτα που η περισσότερη παρακολούθηση δικτύου δεν θα σημάνει, επειδή το «anonymous FTP» είναι τεκμηριωμένη λειτουργία και όχι υπογραφή exploit.

</details>

### 3.4 Δημιουργία του anonymous καταλόγου και του marker file

Ο ανώνυμος χρήστης FTP αντιστοιχίζεται στον τοπικό λογαριασμό `ftp`. Το εργαστήριο δημιουργεί έναν συμβατικό public υποκατάλογο και τον αρχικοποιεί με ένα μικρό marker ανάκτησης:

```bash
mkdir -p /var/ftp/pub
```

```bash
chown nobody:nogroup /var/ftp/pub
```

```bash
cd /var/ftp/pub
```

```bash
echo "Lab FTP retrieval marker" > note.txt
```

```bash
ls -l
```

**Έξοδος:**

```text
total 4
-rw-r--r-- 1 root root 25 Feb 10 12:01 note.txt
```

```bash
cat note.txt
```

**Έξοδος:**

```text
Lab FTP retrieval marker
```

Το share περιέχει τώρα ένα marker 25 byte που ανήκει στο root και είναι αναγνώσιμο από όλους — ακριβώς ό,τι χρειάζεται μια anonymous session για να αποδείξει ότι η ανάκτηση λειτουργεί.

<details>
<summary><strong>Περισσότερα</strong> — γιατί το pub, το nobody:nogroup και ένα απλό marker</summary>

Το όνομα καταλόγου `pub` είναι συμβατικό για έναν anonymous FTP drop folder, με ρίζες στα πρώτα Internet file archives, και είναι ακριβώς αυτό που αναμένει να βρει ένας επιτιθέμενος. Η ιδιοκτησία `nobody:nogroup` ακολουθεί τη συμβατική μη προνομιούχα ταυτότητα για δημόσια δεδομένα.

Το marker file παρέχει στο enumeration workflow ένα συγκεκριμένο artefact προς ανάκτηση· τα περιεχόμενά του είναι σκόπιμα απλά, επειδή το ζητούμενο είναι η ανάκτηση και όχι τα ίδια τα δεδομένα. Σε ένα πραγματικό engagement, το ισοδύναμο artefact θα μπορούσε να είναι ένα αρχείο αντιγράφων ασφαλείας ή ένα deployment script — το εργαστήριο χρησιμοποιεί ένα αβλαβές marker ώστε η άσκηση να εστιάζει στην πρόσβαση και όχι στο περιεχόμενο.

</details>

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

Κάθε εντολή έχει διαφορετική επίδραση: το `anon_root` τοποθετεί τις anonymous sessions στη ρίζα `/var/ftp/`, το `no_anon_password` αφαιρεί το prompt κωδικού, το `hide_ids` αποκρύπτει την ιδιοκτησία στα listings, και το ζεύγος passive-port περιορίζει τις συνδέσεις δεδομένων στις `40000–50000`.

<details>
<summary><strong>Περισσότερα</strong> — τι προσφέρει και τι όχι το hide_ids</summary>

Το `hide_ids` αξίζει ιδιαίτερη προσοχή από αμυντική σκοπιά. Υπάρχει ώστε οι διαχειριστές να μπορούν να προσφέρουν ένα anonymous share χωρίς να αποκαλύπτουν ποιος λογαριασμός Linux κατέχει κάθε αρχείο. Αυτό αποτελεί βελτίωση ιδιωτικότητας και όχι όριο ασφαλείας: δεν αποτρέπει το listing, το download ή την εξαγωγή συμπερασμάτων για την ύπαρξη ευαίσθητων αρχείων.

Η περιοχή passive-port είναι εξίσου μια βοηθητική ρύθμιση διαχείρισης: κάνει τους κανόνες firewall προβλέψιμους ώστε οι συνδέσεις δεδομένων να παραμένουν μέσα σε ένα γνωστό παράθυρο. Η προβλεψιμότητα βοηθά τους defenders μόνο αν αυτό το παράθυρο πραγματικά φιλτράρεται — μια μη φιλτραρισμένη περιοχή 10.000 θυρών είναι απλώς μια μεγάλη πόρτα.

</details>

### 3.6 Επανεκκίνηση και επαλήθευση της υπηρεσίας

Κάντε restart στον daemon για να εφαρμοστεί η ρύθμιση:

```bash
service vsftpd restart
```

```bash
cd ~
```

```bash
ss -tlnp | grep ':21'
```

**Έξοδος:**

```text
LISTEN 0      32                 *:21              *:*    users:(("vsftpd",pid=1842,fd=3))
```

```bash
systemctl status vsftpd --no-pager
```

**Έξοδος:**

```text
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

Η εντολή `service` δεν εμφανίζει έξοδο σε περίπτωση επιτυχίας — η σιωπή είναι επιτυχία. Η γραμμή του `ss` αποδεικνύει ότι ο daemon είναι συνδεδεμένος στη θύρα 21, και το `systemctl status` επιβεβαιώνει ότι η εκτελούμενη διεργασία χρησιμοποιεί το τροποποιημένο αρχείο ρυθμίσεων.

<details>
<summary><strong>Περισσότερα</strong> — τι να ελέγξετε όταν λείπει η θύρα 21</summary>

Σε ένα distribution που διαχειρίζεται το systemd, ένα κενό αποτέλεσμα μετά το `restart` είναι η αναμενόμενη ένδειξη επιτυχίας. Αν η θύρα 21 απουσιάζει, ελέγξτε ξανά τις εντολές listener πριν υποθέσετε ότι η anonymous πρόσβαση είναι σπασμένη: ένας daemon που δεν ακούει δεν μπορεί καν να υποβληθεί σε enumeration.

Η έξοδος του `systemctl status` είναι επίσης εκεί όπου θα εμφανιζόταν μια αποτυχημένη εκκίνηση — η γραμμή `Active:` θα έγραφε `failed` αντί για `active (running)`, και το journal (`journalctl -u vsftpd -n 20`) θα ονόμαζε την εντολή που έσπασε τη ρύθμιση. Το έθιμο να επιβεβαιώνετε «η διεργασία σηκώθηκε, η υποδοχή δέθηκε, η ρύθμιση φορτώθηκε» πριν περάσετε στο μηχάνημα του επιτιθέμενου σάς γλιτώνει σύγχυση αργότερα, όταν κάθε σφάλμα θα μπορούσε να αποδοθεί σε οποιαδήποτε πλευρά του δικτύου.

</details>

---

## 4. Anonymous FTP — enumeration και ανάκτηση

Με τον lab server ενεργό, το workflow του επιτιθέμενου έχει δύο κλασικά βήματα: επιβεβαίωση της anonymous πρόσβασης με Nmap και στη συνέχεια αλληλεπίδραση με το share μέσω μιας FTP session για ανάκτηση του marker.

### 4.1 Ανίχνευση υπηρεσίας με Nmap

Η επιλογή `-A` του Nmap ενεργοποιεί version detection, script scanning, OS detection και traceroute σε μία εκτέλεση. Πάνω σε μία μόνο θύρα εργαστηρίου, είναι ένας συμπαγής τρόπος να επιβεβαιώσετε τόσο τον daemon όσο και τη λανθασμένη ρύθμιση:

```bash
nmap -A -p 21 192.168.1.9
```

**Έξοδος:**

```text
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

Τρία στοιχεία καθορίζουν το επόμενο βήμα. Η υπηρεσία είναι `vsftpd 3.0.5` στην αναμενόμενη θύρα· το script `ftp-anon` αναφέρει `Anonymous FTP login allowed (FTP code 230)`, που είναι άμεση επιβεβαίωση ότι δεν απαιτείται κωδικός· και το script απαριθμεί ήδη τον κατάλογο `pub` μέσα στην anonymous root.

<details>
<summary><strong>Περισσότερα</strong> — η επιβεβαίωση του στόχου και πότε να μην χρησιμοποιείτε το -A</summary>

Ο επιχειρών γνωρίζει πού να κοιτάξει πριν ανοίξει μια διαδραστική session. Σε ένα τοπικό δίκτυο εργαστηρίου, το traceroute ενός άλματος και το πρόθεμα MAC της VMware επιβεβαιώνουν επίσης ότι πρόκειται για έναν κοντινό εικονικό στόχο και όχι για έναν δρομολογημένο host παραγωγής.

Η επιλογή `-A` είναι θορυβώδης και αργή σε σύγκριση με την `-sV`, οπότε χρησιμοποιείται καλύτερα αφού ένα στενότερο scan έχει βρει κάτι που αξίζει διερεύνηση. Εδώ η αξία της είναι η πληρότητα: μία εντολή καθιερώνει την έκδοση, το εύρημα της anonymous πρόσβασης και την αρχική απογραφή του καταλόγου.

</details>

### 4.2 FTP session και ανάκτηση αρχείου

Συνδεθείτε ως `anonymous`, ελέγξτε το share, μπείτε στο `pub` και κατεβάστε το `note.txt`:

```bash
ftp 192.168.1.9
```

**Έξοδος:**

```text
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
```

```bash
cat note.txt
```

**Έξοδος:**

```text
Lab FTP retrieval marker
```

Η session αυθεντικοποιείται ως `anonymous`, λαμβάνει κωδικό `230` για επιτυχή σύνδεση, κάνει listing της anonymous root, μεταβαίνει στο `pub` και κατεβάζει το marker. Το τοπικό `cat` επιβεβαιώνει ότι τα bytes που ανακτήθηκαν ταιριάζουν με αυτά που έσπειρε ο server.

<details>
<summary><strong>Περισσότερα</strong> — η passive λειτουργία, το cleartext και το αποκρυμμένο ownership</summary>

Οι γραμμές `229` δείχνουν συνδέσεις δεδομένων σε passive λειτουργία να ανοίγουν μέσα στην περιοχή `40000–50000` που έχει ρυθμιστεί. Η μεταφορά εδώ είναι ελάχιστη, αλλά η ίδια ροή εργασίας ανακτά αρχεία πολλών gigabyte χωρίς να απαιτείται πρόσθετη πρόσβαση.

Δύο λεπτομέρειες αξίζει να διατηρηθούν για το reporting. Πρώτον, το FTP μεταδίδει το username, τις εντολές, το file listing και τα περιεχόμενα των αρχείων χωρίς κρυπτογράφηση, εκτός αν έχει ρυθμιστεί TLS — οποιοσδήποτε βρίσκεται στη διαδρομή βλέπει το ίδιο transcript με εσάς. Δεύτερον, το `hide_ids=YES` είναι ορατό στο listing: όλα φαίνονται να ανήκουν στο `ftp:ftp`, παρότι το marker file δημιουργήθηκε από τον root. Η απόκρυψη της ιδιοκτησίας επομένως δεν απέτρεψε την αποκάλυψη· απλώς έκανε το listing λιγότερο πληροφοριακό.

</details>

---

## 5. Guest SMB — ρύθμιση του Samba

Το Samba υλοποιεί το πρωτόκολλο SMB/CIFS σε Linux και γεφυρώνει Linux servers με Windows-compatible clients. Όπως και το `vsftpd`, διαθέτει συντηρητικές προεπιλογές τις οποίες το εργαστήριο αποδυναμώνει σκόπιμα. Το προκύπτον export `[shares]` είναι browsable, writable και προσβάσιμο σε SMB clients χωρίς αυθεντικοποίηση.

### 5.1 Εγκατάσταση του Samba

Εγκαταστήστε το πακέτο του server:

```bash
apt install samba
```

**Έξοδος:**

```text
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

Ο installer εγκαθιστά το Samba μαζί με τις κοινές βιβλιοθήκες και τα υποστηρικτικά εργαλεία — έξι πακέτα συνολικά, περισσότερα από όσα χρειάστηκε το `vsftpd`.

<details>
<summary><strong>Περισσότερα</strong> — εγκατεστημένο δεν σημαίνει εκτεθειμένο</summary>

Ο daemon είναι έτοιμος για ρύθμιση μετά την εγκατάσταση, αλλά δεν εκθέτει ακόμη το lab share επειδή η προεπιλεγμένη ρύθμιση δηλώνει μόνο global συμπεριφορά και sections σχετικά με εκτυπωτές. Αυτή η διάκριση έχει σημασία και κατά τη διάρκεια ενός engagement: μια ανοιχτή θύρα 445 σε ένα φρεσκοεγκατεστημένο σύστημα διαφημίζει το Samba, αλλά η λίστα shares που βλέπει ένας anonymous client καθορίζεται εξ ολοκλήρου από το `smb.conf`.

</details>

### 5.2 Έλεγχος του καταλόγου ρυθμίσεων του Samba

Μεταβείτε στον κατάλογο ρυθμίσεων και ελέγξτε τα περιεχόμενά του:

```bash
cd /etc/samba/
```

```bash
ls -al
```

**Έξοδος:**

```text
total 32
drwxr-xr-x  3 root root  4096 Feb 10 12:10 .
drwxr-xr-x 96 root root  4096 Feb 10 11:58 ..
-rw-r--r--  1 root root    17 Feb 10 11:59 gdbcommands
-rw-r--r--  1 root root  9831 Feb 10 11:59 smb.conf
drwxr-xr-x  2 root root  4096 Feb 10 11:59 tls
```

```bash
nano smb.conf
```

Το βασικό αρχείο είναι το `smb.conf`· το `gdbcommands` υποστηρίζει το debugging και το `tls` περιέχει υλικό πιστοποιητικών.

<details>
<summary><strong>Περισσότερα</strong> — επικύρωση με testparm πριν από αλλαγές σε production</summary>

Η απευθείας επεξεργασία του `smb.conf` είναι σαφής για ένα εργαστήριο, αλλά οι αλλαγές σε production θα πρέπει πάντα να επικυρώνονται με `testparm` πριν εφαρμοστεί η νέα ρύθμιση. Το `testparm` αναλύει το αρχείο ακριβώς όπως θα το έκανε ο `smbd` και αναφέρει συντακτικά λάθη ή άγνωστες παραμέτρους, ώστε ένα τυπογραφικό λάθος να μην μπορεί να απενεργοποιήσει σιωπηλά ένα share — ή να ενεργοποιήσει σιωπηλά κάποιο άλλο.

</details>

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

Κάθε γραμμή συμβάλλει στην έκθεση. Το `guest ok = yes` αντιστοιχίζει clients χωρίς αυθεντικοποίηση σε guest πρόσβαση, το `public = yes` είναι legacy σύνταξη για την ίδια πρόθεση, και το `browsable = yes` κάνει το share ορατό κατά το enumeration.

Τα `read only = no` και `writable = yes` εκφράζουν δύο φορές την ίδια writable ρύθμιση: οι δύο εντολές είναι αντίστροφες γραφές της ίδιας επιλογής, επομένως οποιαδήποτε από τις δύο μόνη της θα επέτρεπε εγγραφές.

<details>
<summary><strong>Περισσότερα</strong> — γιατί η δημοσίευση του /var/www επιδεινώνει τον κίνδυνο</summary>

Η δημοσίευση του `/var/www/` επιδεινώνει τον κίνδυνο επειδή αυτή η διαδρομή συνήθως στηρίζει web content· ένα writable SMB share πάνω σε web-served files μπορεί να γίνει ένα primitive εγγραφής οπουδήποτε πίσω από ένα HTTP endpoint. Ένας επιτιθέμενος που δεν μπορεί να εκμεταλλευτεί απευθείας την web εφαρμογή μπορεί συχνά να ανεβάσει ένα webshell μέσω του share και μετά να το ζητήσει μέσω HTTP — δύο άσχετες επιφάνειες, ένας κοινός κατάλογος.

Μια ασφαλέστερη διάταξη δημοσιεύει μια απομονωμένη ρίζα όπως το `/srv/samba/lab` από την οποία δεν διαβάζει καμία άλλη υπηρεσία· η ενότητα 9 επιστρέφει σε αυτό το σημείο.

</details>

Επικυρώστε την τελική ρύθμιση πριν από το restart:

```bash
testparm -s
```

**Έξοδος:**

```text
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

Το `testparm` αναλύει την ισχύουσα ρύθμιση και αναφέρει συντακτικά ή σημασιολογικά προβλήματα. Η κανονικοποιημένη έξοδός του λειτουργεί διπλά και ως χρήσιμο στοιχείο: δείχνει τη διαδρομή του share και την writable κατάσταση χωρίς να απαιτεί από τον αναγνώστη να ερμηνεύσει κάθε περιττή εντολή του αρχείου προέλευσης.

Κάντε restart στον SMB daemon και επιβεβαιώστε τους αναμενόμενους listeners:

```bash
systemctl restart smbd
```

```bash
ss -tlnp | grep -E ':(139|445)'
```

**Έξοδος:**

```text
LISTEN 0      50                 *:139             *:*    users:(("smbd",pid=2261,fd=34))
LISTEN 0      50                 *:445             *:*    users:(("smbd",pid=2261,fd=33))
```

```bash
cd ~
```

Οι θύρες 139 και 445 είναι οι παραδοσιακοί listeners του SMB, και και οι δύο είναι ενεργοί.

<details>
<summary><strong>Περισσότερα</strong> — διάγνωση listeners που λείπουν</summary>

Αν υπάρχει μόνο μία, ελέγξτε αν κάποια άλλη υπηρεσία κατέχει την υποδοχή που λείπει και αν παρεμβαίνει το firewall του host. Αν δεν υπάρχει καμία, η ρύθμιση του Samba δεν φορτώθηκε όπως αναμενόταν.

Η θύρα 445 είναι αυτή που μετράει για τους σύγχρονους clients — μεταφέρει SMB2/SMB3 απευθείας πάνω από TCP — ενώ η 139 είναι η παλαιότερη υπηρεσία NetBIOS session. Το να βλέπετε και τις δύο είναι φυσιολογικό σε μια προεπιλεγμένη εγκατάσταση Samba· το να βλέπετε μόνο την 445 είναι επίσης εντάξει. Το να μην βλέπετε καμία σημαίνει ότι το restart δεν έπιασε, και το `systemctl status smbd` είναι η επόμενη στάση.

</details>

### 5.4 Δημιουργία και έλεγχος των κοινόχρηστων αρχείων

Δημιουργήστε ένα marker στον κοινόχρηστο web κατάλογο και επιβεβαιώστε τοπικά τα περιεχόμενά του:

```bash
echo "Lab SMB retrieval marker" > /var/www/file.txt
```

```bash
cd /var/www/
```

```bash
ls -l
```

**Έξοδος:**

```text
total 12
-rw-r--r-- 1 root root   25 Feb 10 12:14 file.txt
drwxr-xr-x 2 root root 4096 Feb 10 11:47 html
```

```bash
cat file.txt
```

**Έξοδος:**

```text
Lab SMB retrieval marker
```

```bash
cd ~
```

Η κοινόχρηστη ρίζα περιέχει τώρα τον επαληθεύσιμο στόχο ανάκτησης του εργαστηρίου δίπλα στον κανονικό κατάλογο web content `html`.

<details>
<summary><strong>Περισσότερα</strong> — γιατί έχει σημασία η δημιουργία γνωστού αρχείου</summary>

Η δημιουργία ενός γνωστού αρχείου έχει σημασία επειδή η επιτυχής anonymous πρόσβαση πρέπει να αποδεικνύεται με την ανάγνωση των ακριβών bytes και όχι απλώς με την εμφάνιση ενός ονόματος αρχείου. Ένα listing μπορεί να πετύχει ενώ τα downloads αποτυγχάνουν — δικαιώματα, φιλτράρισμα διαδρομών ή ένας κανόνας άρνησης μπορούν να μπλοκάρουν τη μεταφορά — οπότε η διαδρομή «γράψε εδώ, διάβασε εκεί» είναι αυτό που πραγματικά αποδεικνύει την έκθεση.

</details>

---

## 6. Guest SMB — enumeration και ανάκτηση

Δύο συμπληρωματικά εργαλεία χειρίζονται το Samba enumeration. Το NetExec ανακαλύπτει shares γρήγορα σε έναν ή πολλούς hosts, ενώ το `smbclient` παρέχει διαδραστική εργασία session πάνω σε ένα συγκεκριμένο export. Χρησιμοποιήστε NetExec για discovery και `smbclient` για retrieval.

### 6.1 Ανακάλυψη shares με NetExec

Αυθεντικοποιηθείτε ως `guest` με κενό password και κάντε enumeration των shares:

```bash
nxc smb 192.168.1.9 --shares -u 'guest' -p ''
```

**Έξοδος:**

```text
[*] SMB         192.168.1.9    445    UBUNTU-LAB     [*] Unix - Samba 4.17.7-Ubuntu
[+] SMB         192.168.1.9    445    UBUNTU-LAB     UBUNTU-LAB\guest: (Guest)
[*] SMB         192.168.1.9    445    UBUNTU-LAB     Enumerated shares
[*] SMB         192.168.1.9    445    UBUNTU-LAB     Share           Permissions     Remark
[*] SMB         192.168.1.9    445    UBUNTU-LAB     -----           -----------     ------
[*] SMB         192.168.1.9    445    UBUNTU-LAB     print$                          Printer Drivers
[*] SMB         192.168.1.9    445    UBUNTU-LAB     shares          READ            Lab file share
[*] SMB         192.168.1.9    445    UBUNTU-LAB     IPC$                            IPC Service
```

Η έξοδος επιβεβαιώνει guest authentication με το σημείο `(Guest)` και απαριθμεί τρία shares. Το δικαίωμα `READ` στο `shares` είναι η εξουσιοδότηση για να συνεχίσετε: ένας client χωρίς αυθεντικοποίηση μπορεί να περιηγηθεί και να κατεβάσει από αυτό το export.

<details>
<summary><strong>Περισσότερα</strong> — η ανάγνωση της λίστας shares και τα markers του NetExec</summary>

Το `print$` είναι το προεπιλεγμένο share προγραμμάτων οδήγησης εκτυπωτών, το `IPC$` είναι το share επικοινωνίας μεταξύ διεργασιών, και το `shares` είναι το lab export. Τα δύο πρώτα υπάρχουν σχεδόν σε κάθε Samba server και συνήθως δεν αποτελούν το εύρημα· ένα προσαρμοσμένο writable share με δικό του όνομα αποτελεί.

Η σύμβαση markers του NetExec είναι σταθερή σε όλα τα πρωτόκολλα. Οι ενημερωτικές γραμμές ξεκινούν με `[*]`, οι επιτυχίες με `[+]` και οι αποτυχίες με `[-]`. Αυτή η σύμβαση κάνει την έξοδο ενός host ευανάγνωστη και την έξοδο πολλών hosts σαρώσιμη — μετά από ένα sweep δεκάδων μηχανημάτων, οι γραμμές `[+]` είναι αυτές που διαβάζετε πρώτες.

</details>

### 6.2 Καταχώριση shares με `smbclient`

Η ίδια απογραφή μπορεί να ληφθεί απευθείας με το `smbclient`. Η επιλογή `-N` καταστέλλει το prompt κωδικού για guest access, ενώ η `-L` κάνει listing των shares:

```bash
smbclient -N -L //192.168.1.9
```

**Έξοδος:**

```text
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

Ο πίνακας shares συμφωνεί με το αποτέλεσμα του NetExec — δύο ανεξάρτητα εργαλεία, ίδια απογραφή.

<details>
<summary><strong>Περισσότερα</strong> — γιατί η αποτυχία SMB1 στο τέλος είναι καλό νέο</summary>

Τα μηνύματα SMB1 στο τέλος είναι ενημερωτικά: ο client επιχειρεί μια παρωχημένη διάλεκτο μόνο για το workgroup listing, η διαπραγμάτευση αποτυγχάνει, και ο server δεν υποβιβάζεται σε SMB1. Αυτή η αποτυχία είναι ευπρόσδεκτη προεπιλογή. Οι σύγχρονοι clients θα πρέπει να διαπραγματεύονται SMB2 ή SMB3, και η υποχώρηση σε παρωχημένη διάλεκτο θα πρέπει να παραμένει απενεργοποιημένη — ένας server που δέχεται ακόμη SMB1 παραδίδει στον επιτιθέμενο αδυναμίες σε επίπεδο πρωτοκόλλου που διορθώθηκαν στο SMB2 πριν από μια δεκαετία.

</details>

### 6.3 Διαδραστική ανάκτηση με `smbclient`

Ανοίξτε το export `shares` χωρίς κωδικό, κάντε listing και ανακτήστε το marker:

```bash
smbclient //192.168.1.9/shares -N
```

**Έξοδος:**

```text
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
```

```bash
cat file.txt
```

**Έξοδος:**

```text
Lab SMB retrieval marker
```

Το attribute `A` δηλώνει ένα κανονικό αρχείο και το `D` δηλώνει έναν κατάλογο, και το τοπικό `cat` αποδεικνύει ότι τα bytes που ανακτήθηκαν ταιριάζουν με το marker που δημιουργήθηκε.

<details>
<summary><strong>Περισσότερα</strong> — η ίδια session θα μπορούσε επίσης να γράφει</summary>

Επειδή το export είναι writable, η ίδια session θα μπορούσε επίσης να ανεβάσει ή να αντικαταστήσει αρχεία· σε εξουσιοδοτημένο έλεγχο, αυτή η δυνατότητα πρέπει να επιδεικνύεται προσεκτικά και με τον ελάχιστο δυνατό αντίκτυπο, κατά προτίμηση με ένα μοναδικά ονομασμένο marker που αφαιρείται αμέσως μετά.

Η δυνατότητα εγγραφής μέσω καναλιού χωρίς αυθεντικοποίηση είναι εύρημα ουσιαστικά χειρότερο από την ανάγνωση: μια read-only έκθεση διαρρέει δεδομένα, ενώ μια writable μπορεί να φυτέψει ένα webshell, να αλλοιώσει μια ρύθμιση ή να καταστρέψει ένα αντίγραφο ασφαλείας. Αναφέρετε τη διάκριση ρητά.

</details>

---

## 7. Insecure NFS — ρύθμιση των exports

Το NFS, το Network File System, είναι το κανονικό πρωτόκολλο διαμοιρασμού αρχείων Unix προς Unix. Σε αντίθεση με τα FTP και SMB, το NFS από προεπιλογή δεν αυθεντικοποιεί χρήστες σε επίπεδο πρωτοκόλλου· εμπιστεύεται το UID που δηλώνει ο client. Οι επιλογές export παρακάτω οπλίζουν σκόπιμα αυτό το μοντέλο εμπιστοσύνης για το εργαστήριο.

### 7.1 Εγκατάσταση του NFS kernel server

Εγκαταστήστε το πακέτο του server:

```bash
apt install nfs-kernel-server -y
```

**Έξοδος:**

```text
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

Εγκαθίστανται πέντε πακέτα, και τα υποστηρικτικά δεν είναι προαιρετικά — ο server τα χρειάζεται για να αντιστοιχίζει ταυτότητες, να διαχειρίζεται κλειδιά και να καταχωρίζει τις υπηρεσίες RPC του.

<details>
<summary><strong>Περισσότερα</strong> — τι κάνουν τα υποστηρικτικά πακέτα και γιατί μετράει η θύρα 111</summary>

Το `nfs-common` παρέχει βοηθητικά εργαλεία από την πλευρά του client, το `libnfsidmap1` χειρίζεται την αντιστοίχιση ταυτοτήτων, το `keyutils` υποστηρίζει τη διαχείριση κλειδιών, και το `rpcbind` παρέχει την υπηρεσία portmapper στην TCP/UDP θύρα 111. Οι clients ερωτούν τον portmapper για να εντοπίσουν τους daemons NFS και mount, επομένως μια ανοιχτή θύρα 111 αποτελεί ισχυρή ένδειξη ότι η υποδομή NFS είναι προσβάσιμη στον host — ακόμη και πριν μάθετε αν πραγματικά μοιράζεται κάποιο export.

</details>

### 7.2 Δημιουργία του καταλόγου export

Δημιουργήστε έναν public κατάλογο export, κάντε τον world-writable και δημιουργήστε ένα marker file:

```bash
mkdir -p /srv/nfs/public
```

```bash
chmod 777 /srv/nfs/public
```

```bash
cd /srv/nfs/public
```

```bash
echo "Lab NFS retrieval marker" > data.txt
```

```bash
ls -l
```

**Έξοδος:**

```text
total 4
-rw-r--r-- 1 root root 25 Feb 10 12:20 data.txt
```

```bash
cat data.txt
```

**Έξοδος:**

```text
Lab NFS retrieval marker
```

```bash
cd ~
```

Ο κατάλογος είναι πλέον εγγράψιμος από οποιονδήποτε λογαριασμό, σε οποιοδήποτε μηχάνημα, που μπορεί να φτάσει το export.

<details>
<summary><strong>Περισσότερα</strong> — γιατί το 777 μαζί με no_root_squash είναι κλασικός συνδυασμός</summary>

Τα world-writable δικαιώματα `777` είναι επικίνδυνα από μόνα τους και γίνονται ιδιαίτερα επικίνδυνα όταν συνδυάζονται με την επιλογή export `no_root_squash` παρακάτω. Ο συνδυασμός αυτός επιτρέπει σε έναν απομακρυσμένο root χρήστη να γράφει αρχεία που διατηρούν root ownership στον server, κάτι που αποτελεί κλασική επιφάνεια persistence και privilege escalation: ένα αρχείο που φυτεύεται μέσω του mount εμφανίζεται στον server σαν να το είχε δημιουργήσει τοπικά ο root.

</details>

### 7.3 Ορισμός του insecure export

Ο έλεγχος πρόσβασης του NFS βρίσκεται στο `/etc/exports`. Ανοίξτε το:

```bash
nano /etc/exports
```

Προσθέστε την ακόλουθη export line μόνο για το εργαστήριο:

```text
/srv/nfs/public *(rw,sync,no_subtree_check,no_root_squash,insecure)
```

Οι επιλογές συνδυάζουν τέσσερις επικίνδυνες αποφάσεις. Το wildcard `*` εξάγει τον κατάλογο σε κάθε διεύθυνση client, το `rw` παρέχει πρόσβαση ανάγνωσης και εγγραφής, το `no_root_squash` απενεργοποιεί την προεπιλεγμένη προστασία που αντιστοιχίζει τα απομακρυσμένα root αιτήματα στο `nobody`, και το `insecure` επιτρέπει συνδέσεις από μη προνομιούχες θύρες πηγής πάνω από το 1024.

<details>
<summary><strong>Περισσότερα</strong> — η ακριβής επίδραση κάθε επιλογής και η παγίδα του κόμματος</summary>

Το `no_root_squash` είναι η αιχμηρή κόψη: η προεπιλεγμένη συμπεριφορά αντιστοιχίζει τα απομακρυσμένα root αιτήματα στον μη προνομιούχο λογαριασμό `nobody`, και αυτό το flag αφαιρεί την αντιστοίχιση — ένας client που ενεργεί ως root μπορεί να ενεργεί ως root στα εξαγόμενα αρχεία. Το `insecure` αφαιρεί την ιστορική απαίτηση οι clients NFS να χρησιμοποιούν προνομιούχα θύρα, γεγονός που διευρύνει το φάσμα των διεργασιών του client που μπορούν καν να συνδεθούν.

Χρησιμοποιήστε κόμματα μεταξύ των επιλογών. Ένα κόμμα που λείπει ή ένα κατά λάθος κενό μπορεί να αλλάξει το parsing ή να εμποδίσει τη φόρτωση του export, και η προκύπτουσα αποτυχία μπορεί να μοιάζει με πρόβλημα δικτύου ενώ στην πραγματικότητα είναι σφάλμα ρύθμισης ενός χαρακτήρα. Αν το `showmount -e` δεν δείχνει τίποτα μετά από μια αλλαγή, υποψιαστείτε τη σύνταξη πριν το firewall.

</details>

### 7.4 Εφαρμογή και επαλήθευση του export

Επαναφορτώστε τον πίνακα export, ελέγξτε το ισχύον export, κάντε restart στον NFS server και επαληθεύστε τους listeners:

```bash
exportfs -a
```

```bash
exportfs -v
```

**Έξοδος:**

```text
/srv/nfs/public
		*(rw,wdelay,no_root_squash,no_subtree_check,sec=sys,rw,insecure,no_root_squash,no_all_squash)
```

```bash
systemctl restart nfs-kernel-server
```

```bash
ss -tlnp | grep -E ':(111|2049)'
```

**Έξοδος:**

```text
LISTEN 0      4096               *:111             *:*    users:(("rpcbind",pid=2418,fd=4))
LISTEN 0      64                 *:2049            *:*    users:(("nfsd",pid=2431,fd=9))
```

Το `exportfs -a` εξάγει όλους τους καταχωρισμένους καταλόγους χωρίς να απαιτεί restart του daemon, και το `exportfs -v` εμφανίζει τις ισχύουσες επιλογές — συμπεριλαμβανομένου του `no_root_squash`, που επιβεβαιώνει ότι η αλλαγή του εργαστηρίου είναι ενεργή. Τόσο ο portmapper στη θύρα 111 όσο και το NFS στη θύρα 2049 ακούν.

<details>
<summary><strong>Περισσότερα</strong> — τι εγγυώνται το restart και ο έλεγχος των listeners</summary>

Το κενό αποτέλεσμα από το `exportfs -a` είναι φυσιολογικό· σημαίνει ότι δεν υπήρξαν σφάλματα, όχι ότι «δεν συνέβη τίποτα». Το `systemctl restart` ανακυκλώνει τα user-space τμήματα του kernel-server και παραλαμβάνει ευρύτερες αλλαγές της υπηρεσίας.

Ο τελικός έλεγχος επιβεβαιώνει τόσο τον RPC portmapper στη θύρα 111 όσο και το NFS στη θύρα 2049· αν λείπει οποιοδήποτε από τα δύο, το enumeration θα αποτύχει πριν αποκτήσουν σημασία η αυθεντικοποίηση ή η συμπεριφορά του UID. Δύο ενεργοί listeners είναι η βάση που αποδεικνύετε μία φορά εδώ, ώστε κάθε μεταγενέστερη αποτυχία στην πλευρά του επιτιθέμενου να αποδίδεται στο βήμα enumeration και όχι στον server.

</details>

---

## 8. Insecure NFS — enumeration, λήψη και mounting

Το NFS enumeration έχει τρία χρήσιμα επίπεδα: ανακάλυψη των exports, απομακρυσμένη επιθεώρηση ή λήψη μεμονωμένων αρχείων, και mount του export για πλήρη πρόσβαση στο filesystem. Το NetExec καλύπτει τα δύο πρώτα χωρίς να κάνει mount τίποτα. Ένα παραδοσιακό mount παρέχει το τρίτο και εκθέτει τον πλήρη αντίκτυπο του `no_root_squash`.

### 8.1 Κλασική ανακάλυψη exports με `showmount` και `rpcinfo`

Πριν χρησιμοποιήσετε το NetExec, επιβεβαιώστε την κλασική διαδρομή discovery από την πλευρά του επιτιθέμενου. Το `showmount -e` ρωτά τον στόχο ποιους καταλόγους εξάγει, και το `rpcinfo -p` απαριθμεί τις καταχωρισμένες υπηρεσίες RPC:

```bash
showmount -e 192.168.1.9
```

**Έξοδος:**

```text
Export list for 192.168.1.9:
/srv/nfs/public *
```

```bash
rpcinfo -p 192.168.1.9 | head -12
```

**Έξοδος:**

```text
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

Η λίστα export ονομάζει την ευάλωτη διαδρομή και την προδιαγραφή wildcard client της, και ο πίνακας RPC επιβεβαιώνει τον portmapper, τον daemon mount και τις υπηρεσίες NFS πίσω από αυτόν.

<details>
<summary><strong>Περισσότερα</strong> — γιατί οι δυναμικές θύρες mountd περιπλέκουν το firewall</summary>

Οι δυναμικές θύρες του `mountd` είναι φυσιολογικές εκτός αν ο διαχειριστής τις καρφώνει· η παρουσία τους εξηγεί γιατί το firewall του NFS είναι πιο περίπλοκο από το άνοιγμα της θύρας 2049 μόνο. Ο daemon mount ακούει σε μια εφήμερη θύρα που αλλάζει μεταξύ των επανεκκινήσεων, οπότε ένας κανόνας firewall γραμμένος για τη χθεσινή θύρα σταματά σιωπηλά να ταιριάζει — οι defenders είτε καρφώνουν τη θύρα στη ρύθμιση του server είτε επιτρέπουν ολόκληρη την περιοχή RPC, καθένα με τα συμβιβαστικά του.

</details>

### 8.2 Enumeration και listing των exports με NetExec

Η ενότητα NFS του NetExec παρέχει την ίδια ροή discovery στη διεπαφή που χρησιμοποιείται ήδη για SMB:

```bash
nxc nfs 192.168.1.9 --enum-shares
```

**Έξοδος:**

```text
[*] NFS         192.168.1.9    2049   UBUNTU-LAB     [*] Enumerating NFS exports
[+] NFS         192.168.1.9    2049   UBUNTU-LAB     /srv/nfs/public (rw, no_root_squash, root escape: True)
```

```bash
nxc nfs 192.168.1.9 --share '/srv/nfs/public' --ls '/'
```

**Έξοδος:**

```text
[*] NFS         192.168.1.9    2049   UBUNTU-LAB     [*] Listing / on /srv/nfs/public
[*] NFS         192.168.1.9    2049   UBUNTU-LAB     drwxrwxrwx root root 4096 .
[*] NFS         192.168.1.9    2049   UBUNTU-LAB     -rw-r--r-- root root   25 data.txt
```

Η πρώτη εντολή ανακαλύπτει το export και επισημαίνει τον επικίνδυνο συνδυασμό του: writable access και `no_root_squash`, συνοψισμένο ως `root escape: True`. Η δεύτερη εντολή κάνει listing της ρίζας του export χωρίς τοπικό mount και αποκαλύπτει το `data.txt`.

<details>
<summary><strong>Περισσότερα</strong> — τι σημαίνει root escape: True</summary>

Το `root escape: True` είναι η συντομογραφία του NetExec για «ένας απομακρυσμένος root client θα διατηρούσε root ownership σε αυτό το export» — ακριβώς η συνθήκη που διαμόρφωσε η ενότητα 7.3. Το listing δεν απαιτεί διαπιστευτήρια επειδή το ίδιο το export δεν επιβάλλει έλεγχο αυθεντικοποίησης· η μόνη πύλη είναι η δικτυακή προσβασιμότητα στη θύρα 2049.

Γι' αυτό το ίδιο listing από ένα μη έμπιστο τμήμα είναι ο έλεγχος που μετράει: μέσα από το εργαστήριο αποδεικνύει ότι το export λειτουργεί, αλλά από ένα γειτονικό VLAN θα αποδείκνυε την έκθεση.

</details>

### 8.3 One-shot λήψη αρχείου με NetExec

Το NetExec μπορεί να κατεβάσει ένα αρχείο απευθείας από το export. Η εντολή δέχεται την απομακρυσμένη διαδρομή σχετικά με τη ρίζα του share και έναν τοπικό προορισμό:

```bash
nxc nfs 192.168.1.9 --share /srv/nfs/public/ --get-file data.txt data.txt
```

**Έξοδος:**

```text
[*] NFS         192.168.1.9    2049   UBUNTU-LAB     [*] Downloading data.txt from /srv/nfs/public/
[+] NFS         192.168.1.9    2049   UBUNTU-LAB     File successfully downloaded to data.txt
```

```bash
cat data.txt
```

**Έξοδος:**

```text
Lab NFS retrieval marker
```

Αυτή είναι η ταχύτερη διαδρομή από το «το αρχείο υπάρχει» στο «το αρχείο βρίσκεται στον τοπικό δίσκο» — χωρίς mount point, χωρίς πρόσβαση shell στον στόχο, ένα απομακρυσμένο αρχείο που γίνεται ένα τοπικό αρχείο.

<details>
<summary><strong>Περισσότερα</strong> — γιατί αυτός είναι και ο καθαρότερος τρόπος συλλογής στοιχείων</summary>

Δεν δημιουργείται mount point, δεν αλλάζει μόνιμη κατάσταση του filesystem στην πλευρά του επιτιθέμενου, και δεν απαιτείται πρόσβαση shell στον στόχο. Για τη διαχείριση στοιχείων, είναι επίσης η καθαρότερη μέθοδος: ένα απομακρυσμένο αρχείο γίνεται ένα τοπικό αρχείο χωρίς ευρύτερες παρενέργειες στο filesystem.

Ένα αντίγραφο που κατεβάστηκε μπορεί να γίνει hash και να αρχειοθετηθεί ως έχει· ένα mounted filesystem, αντίθετα, αφήνει ξεπερασμένα file handles και ένα αλλοιωμένο τοπικό δέντρο καταλόγων πίσω του αν το μηχάνημα επανεκκινήσει στη μέση της ανάλυσης. Όταν ο στόχος είναι ένα μόνο αρχείο, το download κερδίζει το mount κάθε φορά.

</details>

### 8.4 Mounting του export με `mount -t nfs`

Ένα mount παρέχει πλήρη σημασιολογία filesystem: `ls`, `find`, `grep`, `cp`, επιθεώρηση δικαιωμάτων και εκτέλεση πάνω στο απομακρυσμένο export σαν να ήταν τοπικός κατάλογος. Δημιουργήστε ένα mount point και κάντε mount το export:

```bash
mkdir -p /tmp/nfs
```

```bash
mount -t nfs 192.168.1.9:/srv/nfs/public /tmp/nfs
```

```bash
ls -la /tmp/nfs
```

**Έξοδος:**

```text
total 12
drwxrwxrwx  2 root root 4096 Feb 10 12:20 .
drwxrwxrwt 14 root root 4096 Feb 10 12:24 ..
-rw-r--r--  1 root root   25 Feb 10 12:20 data.txt
```

```bash
cat /tmp/nfs/data.txt
```

**Έξοδος:**

```text
Lab NFS retrieval marker
```

Ο mounted κατάλογος διατηρεί την ιδιοκτησία και τα δικαιώματα από την πλευρά του server, συμπεριλαμβανομένης της world-writable πρόσβασης και της root ιδιοκτησίας — το export συμπεριφέρεται ακριβώς όπως ένας τοπικός φάκελος.

<details>
<summary><strong>Περισσότερα</strong> — πώς το no_root_squash μετατρέπει ένα share σε εργαλείο privilege escalation</summary>

Εξαιτίας του `no_root_squash`, οι λειτουργίες που εκτελούνται ως root μέσω αυτού του mount διατηρούν την ταυτότητα root στα αρχεία της πλευράς του server. Τα αρχεία που γράφονται μέσω του mount μπορούν να φέρουν bits ιδιοκτησίας και δικαιωμάτων που κανονικά θα απαιτούσαν τοπική πρόσβαση root για να δημιουργηθούν — για παράδειγμα ένα setuid root binary που ρίχνεται σε έναν world-readable κατάλογο, το οποίο μπορεί στη συνέχεια να εκτελέσει οποιοσδήποτε τοπικός χρήστης στον στόχο.

Αυτή είναι η διαφορά ανάμεσα σε «share που διαρρέει» και «απομακρυσμένο privilege escalation»: το share το ίδιο δεν χρειάζεται καμία ευπάθεια, μόνο το λάθος ζεύγος επιλογών export.

</details>

Αφαιρέστε το export όταν τελειώσετε:

```bash
umount /tmp/nfs
```

```bash
ls /tmp/nfs
```

<details>
<summary><strong>Περισσότερα</strong> — γιατί το mount πρέπει να αφαιρεθεί</summary>

Ο καθαρισμός έχει σημασία σε ένα κοινόχρηστο εργαστήριο. Ένα ξεχασμένο mount μπορεί να μπερδέψει μεταγενέστερους ελέγχους, να διατηρήσει ξεπερασμένα file handles μετά την αλλαγή του export και να αφήσει παραπλανητικά περιεχόμενα καταλόγου στο `/tmp`. Επαληθεύστε ότι το mount point είναι κενό μετά το unmount — ένα κενό `ls` είναι η επιβεβαίωση ότι δεν έμεινε τίποτα πίσω.

Σε ένα πραγματικό engagement ισχύει η ίδια πειθαρχία: τα mounts που ξεχνάτε είναι mounts που ο επόμενος tester, ή ένας auditor που εξετάζει το μηχάνημα, θα βρει αποδιδόμενα σε εσάς.

</details>

---

## 9. Σύνοψη hardening

Το εργαστήριο έχει πλέον δεχθεί επίθεση από άκρη σε άκρη, και κάθε έκθεση πρωτοκόλλου προήλθε από έναν μικρό αριθμό επιλογών ρύθμισης. Η ενότητα αυτή συγκεντρώνει τα αντίστοιχα μέτρα σε ένα σημείο.

### 9.1 FTP hardening

**Απενεργοποιήστε την anonymous πρόσβαση.** Ορίστε `anonymous_enable=NO` στο `/etc/vsftpd.conf` εκτός αν μια τεκμηριωμένη επιχειρηματική ανάγκη δικαιολογεί την anonymous ανάκτηση. Ακόμη και τότε, προτιμήστε το SFTP πάνω από SSH για μεταφορές αρχείων σε μη έμπιστα δίκτυα, επειδή το SFTP παρέχει αυθεντικοποίηση και κρυπτογράφηση από προεπιλογή.

**Επιβάλετε TLS όπου το FTP πρέπει να παραμείνει.** Ορίστε `ssl_enable=YES` με `force_local_logins_ssl=YES` και `force_local_data_ssl=YES`, ώστε τα διαπιστευτήρια και το μεταφερόμενο περιεχόμενο να μην διασχίζουν το δίκτυο σε cleartext.

**Περιορίστε την έκθεση στο δίκτυο.** Δέστε το `vsftpd` σε εσωτερικές διεπαφές όπου είναι δυνατόν, περιορίστε τη θύρα 21 σε γνωστά υποδίκτυα διαχείρισης με firewall του host, και ανοίξτε μόνο τη ρυθμισμένη περιοχή passive-port. Ένα share που δεν είναι προσβάσιμο από ένα μη έμπιστο τμήμα δεν μπορεί να υποβληθεί σε enumeration από αυτό το τμήμα.

<details>
<summary><strong>Περισσότερα</strong> — ανάπτυξη FTPS χωρίς να σπάσετε τις μεταφορές αρχείων που χρειάζεστε ακόμη</summary>

Τα πιστοποιητικά, η συμβατότητα των clients και οι κανόνες firewall της περιοχής passive-port πρέπει να διαχειρίζονται ως μέρος της ίδιας ανάπτυξης TLS. Παλαιότερα scripts που μιλούν απλό FTP θα αποτύχουν θορυβωδώς μόλις ενεργοποιηθεί το `force_local_logins_ssl`, οπότε καταγράψτε τους clients πριν γυρίσετε τον διακόπτη — και αντιμετωπίστε κάθε client που δεν μπορεί να κάνει TLS ως λόγο να αντικαταστήσετε τη μέθοδο μεταφοράς, όχι να αποδυναμώσετε την πολιτική.

Στην πλευρά του firewall, θυμηθείτε ότι το FTP χρειάζεται τόσο το κανάλι ελέγχου (θύρα 21) όσο και την περιοχή δεδομένων passive· το άνοιγμα του ενός χωρίς το άλλο παράγει timeouts που μοιάζουν με σφάλματα του server.

</details>

### 9.2 Samba hardening

**Απενεργοποιήστε την guest πρόσβαση.** Αφαιρέστε τα `guest ok = yes` και `public = yes` από κάθε share, ορίστε `map to guest = Never` στο `[global]`, και απαιτήστε έγκυρους λογαριασμούς Linux για κάθε σύνδεση. Η anonymous πρόσβαση SMB θα πρέπει να είναι εξαίρεση, τεκμηριωμένη και τακτικά ελεγχόμενη.

**Απενεργοποιήστε το SMB1.** Ορίστε `min protocol = SMB2` στο `[global]` για να αποτρέψετε την υποχώρηση στην παρωχημένη διάλεκτο SMB1.

**Περιορίστε τις διαδρομές των shares.** Μην δημοσιεύετε ποτέ τα `/var/www`, `/etc`, τους οικιακούς καταλόγους, τις ρίζες αντιγράφων ασφαλείας ή οποιαδήποτε διαδρομή στηρίζει άλλη υπηρεσία. Χρησιμοποιήστε απομονωμένες ρίζες shares όπως `/srv/samba/<sharename>`, ώστε ένα λάθος ελέγχου πρόσβασης να έχει μικρότερο εύρος ζημιάς. Συνδυάστε αυτή τη διάταξη με δικαιώματα filesystem που αρνούνται τις εγγραφές εκτός αν οι εγγραφές απαιτούνται ρητά.

<details>
<summary><strong>Περισσότερα</strong> — τι αλλάζει πραγματικά το map to guest = Never</summary>

Το `map to guest = Never` κλείνει το διακριτικό μισό της έκθεσης: με αυτό σε ισχύ, ένα άγνωστο όνομα χρήστη απορρίπτεται ευθέως αντί να αντιστοιχίζεται σιωπηλά στον λογαριασμό guest. Το να αφαιρέσετε το `guest ok` από ένα share ενώ παραμένει το `map to guest = Bad User` στο `[global]` μπορεί ακόμη να διερευνηθεί από clients που μαντεύουν πώς συμπεριφέρεται η αντιστοίχιση, οπότε και οι δύο αλλαγές ανήκουν στην ίδια αλλαγή.

Το SMB1 αξίζει την ίδια αντιμετώπιση με ζώνη και τιράντες: οι σύγχρονοι clients διαπραγματεύονται SMB2 ή SMB3 από προεπιλογή, οπότε το `min protocol = SMB2` σπάνια σπάει τη νόμιμη πρόσβαση ενώ αφαιρεί μια διαδρομή πρωτοκόλλου με γνωστές, οπλισμένες αδυναμίες.

</details>

### 9.3 NFS hardening

**Ενεργοποιήστε το `root_squash`.** Αφαιρέστε το `no_root_squash` από κάθε export line. Η προεπιλεγμένη συμπεριφορά `root_squash` αντιστοιχίζει τα απομακρυσμένα root αιτήματα στον μη προνομιούχο λογαριασμό `nobody` και αφαιρεί το διάνυσμα μονιμότητας της ιδιοκτησίας αρχείων που παρουσιάστηκε στην ενότητα 8.

**Περιορίστε τα wildcards των exports.** Αντικαταστήστε το `*` με ρητές διευθύνσεις client ή περιοχές CIDR, ώστε μόνο γνωστοί hosts να μπορούν να κάνουν mount το export. Ένας backup server, ένας application host ή ένας σταθμός διαχείρισης θα πρέπει να ονομάζονται μεμονωμένα· ένα ολόκληρο μη έμπιστο υποδίκτυο δεν θα πρέπει να εμπιστεύεται από προεπιλογή.

**Αυθεντικοποιήστε το NFS όπου το δίκτυο δεν είναι έμπιστο.** Χρησιμοποιήστε NFSv4 με ασφάλεια Kerberos, όπως `sec=krb5p`, όπου απαιτούνται αυθεντικοποίηση και κρυπτογράφηση. Χωρίς Kerberos, το προεπιλεγμένο μοντέλο εμπιστοσύνης στο UID του NFS είναι ακατάλληλο για εχθρικά ή zero-trust δίκτυα.

**Εφαρμόστε firewall στις θύρες RPC και NFS.** Αποκλείστε την TCP/UDP θύρα 111 για το `rpcbind`, την TCP/UDP θύρα 2049 για το NFS και τη θύρα του daemon mount για κάθε μη έμπιστη πηγή. Καρφώστε τις δυναμικές υπηρεσίες RPC σε σταθερές θύρες όπου η πολιτική firewall το απαιτεί, και επαληθεύστε την έκθεση από ένα μη έμπιστο τμήμα δικτύου και όχι μόνο από το localhost.

<details>
<summary><strong>Περισσότερα</strong> — η επιλογή επιπέδου ασφάλειας NFS</summary>

Τα επίπεδα `sec=` του Kerberos ανταλλάσσουν προστασία με επιβάρυνση: το `krb5` αυθεντικοποιεί το mount, το `krb5i` ελέγχει επιπλέον την ακεραιότητα κάθε block, και το `krb5p` κρυπτογραφεί και την κίνηση. Επιλέξτε `krb5p` όταν η διαδρομή διασχίζει κάτι που δεν ελέγχετε· το `krb5i` συχνά αρκεί σε ένα παρακολουθούμενο εσωτερικό τμήμα.

Όποιο επίπεδο κι αν επιλέξετε, θυμηθείτε ότι το Kerberos προσθέτει εξάρτηση από τον συγχρονισμό χρόνου και από έναν προσβάσιμο KDC — δοκιμάστε το mount από έναν καθαρό client πριν δηλώσετε ολοκληρωμένη τη μετάβαση.

</details>

### 9.4 Ανίχνευση και auditing

**Ελέγχετε απευθείας τα αρχεία ρυθμίσεων.** Κάθε σημαντική λανθασμένη ρύθμιση αυτού του walkthrough είναι ορατή με μία αναζήτηση ανά αρχείο. Χρήσιμοι αρχικοί έλεγχοι περιλαμβάνουν τα `anonymous_enable`, `guest ok`, `public`, `map to guest`, `min protocol`, `no_root_squash`, τα wildcards των exports και τους world-writable καταλόγους export.

**Κάντε scan από την οπτική του επιτιθέμενου.** Ερευνήστε περιοδικά για τις θύρες 21, 111, 139, 445 και 2049 από μη έμπιστα τμήματα. Μια ανοιχτή θύρα είναι μόνο η αρχή· ακολουθήστε την με τους ίδιους anonymous και guest ελέγχους που χρησιμοποιούνται στις ενότητες 4, 6 και 8.

**Ελέγχετε τα inventories των shares και τα logs.** Διατηρείτε τα logs των FTP, Samba και NFS, δημιουργήστε alerts για anonymous ή guest authentication, και ελέγχετε τα inventories των exports και των shares μετά από κάθε αλλαγή ρύθμισης.

<details>
<summary><strong>Περισσότερα</strong> — η τριμηνιαία ρουτίνα που κλείνει κάθε primitive εδώ</summary>

Ένας τριμηνιαίος έλεγχος των `/etc/vsftpd.conf`, `/etc/samba/smb.conf` και `/etc/exports`, σε συνδυασμό με scans σε επίπεδο δικτύου, κλείνει κάθε primitive που παρουσιάζεται σε αυτόν τον οδηγό. Τα greps των ρυθμίσεων διαρκούν δευτερόλεπτα· η αξία προέρχεται από το να τα κάνετε προγραμματισμένα, επειδή τα shares προστίθενται ταχύτερα τις εβδομάδες μετά την απόφαση μιας ομάδας ότι «χρειάζεται απλώς ένας γρήγορος φάκελος υποβολής».

Ο έλεγχος των logs ολοκληρώνει τον κύκλο: μια anonymous σύνδεση που εμφανίζεται σε log FTP ή Samba σε έναν host που υποτίθεται ότι έχει απενεργοποιημένη την anonymous πρόσβαση είναι ένα άμεσο σήμα παρέκκλισης.

</details>

### 9.5 Τελική ανάλυση

Τα FTP, SMB και NFS στηρίζουν συλλογικά μεγάλο μέρος του enterprise file sharing. Κάθε πρωτόκολλο διαθέτει συντηρητικές προεπιλογές, και καθένα αποδυναμώνεται συστηματικά για λόγους ευκολίας. Αυτό το εργαστήριο αναπαρήγαγε τις προκύπτουσες λανθασμένες ρυθμίσεις: anonymous FTP με αποκρυμμένη ιδιοκτησία, guest-accessible Samba shares πάνω σε ευαίσθητες διαδρομές, και NFS exports με `no_root_squash` και wildcard clients.

Οι αντίστοιχες ροές enumeration συμπτύσσουν την ανακάλυψη και την εκμετάλλευση σε έναν μικρό αριθμό εντολών — μια χούφτα κλήσεις Nmap, `ftp`, `smbclient`, `nxc` και `showmount` πήγαν κάθε έκθεση από το «η θύρα είναι ανοιχτή» στο «το αρχείο βρίσκεται στον δίσκο του επιτιθέμενου».

<details>
<summary><strong>Περισσότερα</strong> — τι παίρνουν μαζί τους οι defenders από αυτό το εργαστήριο</summary>

Οι defenders κερδίζουν αντιμετωπίζοντας τη ρύθμιση των file shares ως security-critical infrastructure και όχι ως βοηθητική υποδομή. Κάθε διόρθωση είναι σύντομη, κάθε έκθεση είναι ανιχνεύσιμη, και το ίδιο εργαστήριο υποστηρίζει και τις δύο πλευρές της διαδικασίας: τον operator που εξασκεί το attack chain και τον defender που επαληθεύει ότι τα μέτρα πράγματι το πιάνουν.

Ο κοινός παρονομαστής και στα τρία πρωτόκολλα είναι η εμπιστοσύνη: το FTP εμπιστεύεται μια anonymous ταυτότητα, το SMB εμπιστεύεται μια guest αντιστοίχιση, το NFS εμπιστεύεται ένα UID που δηλώνει ο client. Το hardening κάθε υπηρεσίας σημαίνει να κάνετε την εμπιστοσύνη ρητή — ονομασμένοι λογαριασμοί, ονομασμένες διευθύνσεις client, αυθεντικοποιημένα mounts — και έπειτα να επαληθεύετε από έξω ότι η πολιτική ισχύει.

</details>

---

## 10. Cheat sheet γρήγορης αναφοράς

| Φάση | Εντολή | Τι κάνει |
| --- | --- | --- |
| Ρύθμιση | `apt update` | ενημερώνει τον κατάλογο πακέτων πριν από την εγκατάσταση |
| Ρύθμιση | `apt install vsftpd` | εγκαθιστά τον FTP daemon |
| Ρύθμιση | `apt install samba` | εγκαθιστά το Samba και τις υποστηρικτικές του βιβλιοθήκες |
| Ρύθμιση | `apt install nfs-kernel-server -y` | εγκαθιστά τον NFS server, τις βιβλιοθήκες client και το `rpcbind` |
| Ρύθμιση | `ss -tlnp \| grep ':21'` | επιβεβαιώνει ότι το FTP ακούει στη θύρα 21 |
| Ρύθμιση | `ss -tlnp \| grep -E ':(139\|445)'` | επιβεβαιώνει ότι το SMB ακούει στις θύρες 139 και 445 |
| Ρύθμιση | `ss -tlnp \| grep -E ':(111\|2049)'` | επιβεβαιώνει ότι υπάρχουν listeners RPC και NFS |
| FTP ρυθμίσεις | `nano /etc/vsftpd.conf` | επεξεργάζεται τη ρύθμιση του FTP server |
| FTP ρυθμίσεις | `anonymous_enable=YES` | ενεργοποιεί ανώνυμες συνδέσεις FTP χωρίς κωδικό |
| FTP ρυθμίσεις | `anon_root=/var/ftp/` | τοποθετεί τις anonymous sessions στη ρίζα `/var/ftp/` |
| FTP ρυθμίσεις | `no_anon_password=YES` | αφαιρεί το prompt κωδικού για anonymous |
| FTP ρυθμίσεις | `hide_ids=YES` | αποκρύπτει το UID/GID στα listings FTP |
| FTP ρυθμίσεις | `pasv_min_port=40000` and `pasv_max_port=50000` | περιορίζει τις θύρες δεδομένων σε passive λειτουργία |
| FTP ρυθμίσεις | `service vsftpd restart` | εφαρμόζει τις αλλαγές ρύθμισης FTP |
| FTP αρχεία | `mkdir -p /var/ftp/pub` | δημιουργεί τον συμβατικό anonymous υποκατάλογο |
| FTP αρχεία | `chown nobody:nogroup /var/ftp/pub` | αναθέτει τη συμβατική μη προνομιούχα ιδιοκτησία |
| FTP αρχεία | `echo "Lab FTP retrieval marker" > /var/ftp/pub/note.txt` | δημιουργεί ένα επαληθεύσιμο FTP marker |
| FTP αναγνώριση | `nmap -A -p 21 <target>` | επιβεβαιώνει την έκδοση FTP και την anonymous πρόσβαση |
| FTP πρόσβαση | `ftp <target>` then `anonymous` | ανοίγει μια FTP session χωρίς κωδικό |
| FTP πρόσβαση | `ls`, `cd pub`, `ls`, `get note.txt`, `bye` | περιηγείται στο anonymous share και ανακτά ένα αρχείο |
| SMB ρυθμίσεις | `nano /etc/samba/smb.conf` | επεξεργάζεται τη ρύθμιση του Samba |
| SMB ρυθμίσεις | `testparm -s` | επικυρώνει την ισχύουσα ρύθμιση Samba |
| SMB ρυθμίσεις | `guest ok = yes` | αντιστοιχίζει clients χωρίς αυθεντικοποίηση σε guest πρόσβαση |
| SMB ρυθμίσεις | `public = yes` | legacy συνώνυμο για πρόθεση guest πρόσβασης |
| SMB ρυθμίσεις | `browsable = yes` | κάνει το share ορατό κατά το enumeration |
| SMB ρυθμίσεις | `read only = no` and `writable = yes` | κάνει το share εγγράψιμο |
| SMB ρυθμίσεις | `systemctl restart smbd` | εφαρμόζει τις αλλαγές ρύθμισης Samba |
| SMB αρχεία | `echo "Lab SMB retrieval marker" > /var/www/file.txt` | δημιουργεί ένα επαληθεύσιμο SMB marker |
| SMB αναγνώριση | `nxc smb <target> --shares -u 'guest' -p ''` | κάνει enumeration των shares ως guest |
| SMB πρόσβαση | `smbclient -N -L //<target>` | εμφανίζει τα shares χωρίς κωδικό |
| SMB πρόσβαση | `smbclient //<target>/shares -N` | ανοίγει το share με guest πρόσβαση |
| SMB πρόσβαση | `ls`, `get file.txt`, `exit` | κάνει listing και ανακτά ένα αρχείο μέσω SMB |
| NFS ρυθμίσεις | `nano /etc/exports` | επεξεργάζεται τον πίνακα exports του NFS |
| NFS ρυθμίσεις | `/srv/nfs/public *(rw,sync,no_subtree_check,no_root_squash,insecure)` | δημοσιεύει έναν ευάλωτο export μόνο για το εργαστήριο |
| NFS ρυθμίσεις | `exportfs -a` | επαναφορτώνει κάθε εγγραφή του πίνακα exports |
| NFS ρυθμίσεις | `exportfs -v` | εμφανίζει τις ισχύουσες επιλογές export |
| NFS ρυθμίσεις | `systemctl restart nfs-kernel-server` | επανεκκινεί τις υπηρεσίες NFS του user space |
| NFS αρχεία | `mkdir -p /srv/nfs/public && chmod 777 /srv/nfs/public` | δημιουργεί έναν world-writable κατάλογο export |
| NFS αρχεία | `echo "Lab NFS retrieval marker" > /srv/nfs/public/data.txt` | δημιουργεί ένα επαληθεύσιμο NFS marker |
| NFS αναγνώριση | `showmount -e <target>` | εμφανίζει τους καταλόγους που εξάγονται στο δίκτυο |
| NFS αναγνώριση | `rpcinfo -p <target>` | εμφανίζει τις καταχωρισμένες υπηρεσίες RPC |
| NFS αναγνώριση | `nxc nfs <target> --enum-shares` | ανακαλύπτει NFS exports με το NetExec |
| NFS αναγνώριση | `nxc nfs <target> --share '/srv/nfs/public' --ls '/'` | κάνει listing της ρίζας ενός export χωρίς mount |
| NFS πρόσβαση | `nxc nfs <target> --share /srv/nfs/public/ --get-file data.txt data.txt` | κατεβάζει ένα αρχείο NFS χωρίς mount |
| NFS πρόσβαση | `mount -t nfs <target>:/srv/nfs/public /tmp/nfs` | κάνει mount το export τοπικά |
| NFS πρόσβαση | `ls -la /tmp/nfs` and `cat /tmp/nfs/data.txt` | ελέγχει και διαβάζει το mounted export |
| NFS πρόσβαση | `umount /tmp/nfs` | αφαιρεί το τοπικό mount NFS |
| Άμυνα | `grep -nE 'anonymous_enable\|guest ok\|public =\|map to guest\|min protocol' /etc/vsftpd.conf /etc/samba/smb.conf` | εντοπίζει επικίνδυνες directives FTP/SMB |
| Άμυνα | `grep -nE 'no_root_squash\|\*\(\|insecure' /etc/exports` | εντοπίζει επικίνδυνες επιλογές export NFS |
| Άμυνα | `nmap -sV -p 21,111,139,445,2049 <target>` | ελέγχει την έκθεση διαμοιρασμού αρχείων από το δίκτυο |

---

## 11. Ασκήσεις εξάσκησης

Η ακόλουθη σειρά έχει σχεδιαστεί για εκτέλεση σε απομονωμένο εργαστήριο με δύο εικονικές μηχανές και χωρίς διαδρομή προς το Internet. Κάθε άσκηση βασίζεται στην προηγούμενη, και αρκετές έχουν ως στόχο να δείξουν πώς μία αμυντική αλλαγή μεταβάλλει την έξοδο από την πλευρά του επιτιθέμενου.

1. **Στήστε το πεδίο.** Εγκαταστήστε τα `vsftpd`, Samba και `nfs-kernel-server` στον στόχο Ubuntu. Χρησιμοποιήστε το `ss -tlnp` για να επιβεβαιώσετε listeners στις θύρες 21, 139, 445, 111 και 2049. Εξηγήστε ποια διεργασία κατέχει κάθε listener και γιατί ο RPC portmapper πρέπει να συμπεριλαμβάνεται στον έλεγχο.
2. **Αποδείξτε το anonymous FTP.** Εκτελέστε `nmap -A -p 21` κατά του στόχου και εντοπίστε την έξοδο του script που αποδεικνύει την anonymous πρόσβαση. Στη συνέχεια ανοίξτε μια FTP session ως `anonymous`, ανακτήστε το `note.txt` και συγκρίνετε τα bytes του με το marker που δημιουργήθηκε στον server.
3. **Απενεργοποιήστε το anonymous FTP.** Ορίστε `anonymous_enable=NO`, κάντε restart στο `vsftpd` και επαναλάβετε τόσο τον έλεγχο Nmap όσο και τη διαδραστική σύνδεση. Καταγράψτε την ακριβώς αλλαγμένη έξοδο και εξηγήστε ποια ρύθμιση στον server την προκάλεσε.
4. **Χαρτογραφήστε το guest SMB.** Κάντε enumeration των shares ως `guest` με το NetExec και έπειτα με `smbclient -N -L`. Συγκρίνετε τα δύο inventories και εντοπίστε κάθε share που δεν θα έπρεπε κανονικά να είναι διαθέσιμο σε έναν client χωρίς αυθεντικοποίηση.
5. **Ανάκτηση μέσω SMB.** Συνδεθείτε στο export `shares` με το `smbclient`, κάντε listing των περιεχομένων του, ανακτήστε το `file.txt` και επαληθεύστε τοπικά τα περιεχόμενά του. Περιγράψτε τι αποκαλύπτουν τα SMB attributes για τα αρχεία σε σχέση με τους καταλόγους.
6. **Αφαιρέστε την guest πρόσβαση.** Αφαιρέστε τα `guest ok` και `public` από το lab share, ορίστε `map to guest = Never`, επικυρώστε με `testparm`, κάντε restart στο Samba και εκτελέστε ξανά και τις δύο εντολές enumeration. Τεκμηριώστε ποια συμπεριφορά αλλάζει πρώτη από την οπτική του επιτιθέμενου.
7. **Ανακαλύψτε το NFS κλασικά.** Χρησιμοποιήστε τα `showmount -e` και `rpcinfo -p` για να εντοπίσετε την εξαγόμενη διαδρομή και τις υπηρεσίες RPC που την υποστηρίζουν. Εξηγήστε γιατί ένας κανόνας firewall μόνο για τη θύρα 2049 μπορεί να μην αρκεί.
8. **Κάντε enumeration του NFS με το NetExec.** Εκτελέστε `--enum-shares` και έπειτα κάντε listing της ρίζας του export. Εντοπίστε τον επικίνδυνο συνδυασμό επιλογών στην έξοδο και εξηγήστε τι σημαίνει το `root escape: True` για την ιδιοκτησία των αρχείων.
9. **Λήψη χωρίς mount.** Ανακτήστε το `data.txt` με το `--get-file` του NetExec, έπειτα κάντε mount το export και διαβάστε το ίδιο αρχείο μέσω `/tmp/nfs`. Συγκρίνετε τις δύο μεθόδους ανάκτησης ως προς την ταχύτητα, το forensic αποτύπωμα και τις διαθέσιμες λειτουργίες filesystem.
10. **Επαναφέρετε ασφαλείς προεπιλογές NFS.** Αντικαταστήστε το wildcard client με μια ρητή διεύθυνση του εργαστηρίου, αφαιρέστε τα `no_root_squash` και `insecure`, επαναφορτώστε με `exportfs -a` και εκτελέστε ξανά το enumeration με NetExec και τη δοκιμή mount. Καταγράψτε ποιες λειτουργίες εξακολουθούν να δουλεύουν και ποιες πλέον αποτυγχάνουν.
11. **Ελέγξτε και τις τρεις υπηρεσίες.** Γράψτε έναν έλεγχο `grep` για κάθε αρχείο ρυθμίσεων που θα εντοπίζει την αντίστοιχη λανθασμένη ρύθμιση του εργαστηρίου. Στη συνέχεια εκτελέστε μία μόνο εντολή Nmap που καλύπτει όλες τις σχετικές θύρες διαμοιρασμού αρχείων και εξηγήστε τι μπορεί και τι δεν μπορεί να αποδείξει το scan σχετικά με την εξουσιοδότηση.
12. **Γράψτε την αναφορά.** Συνοψίστε όλη την αλυσίδα σε ένα εύρημα πεντακοσίων λέξεων: τις αρχικές αδυναμίες, τη διαδρομή εκμετάλλευσης ανά πρωτόκολλο, τον επιχειρηματικό αντίκτυπο και τρία μέτρα που θα είχαν αποτρέψει ανεξάρτητα το καθένα την ανάκτηση χωρίς αυθεντικοποίηση.

Η ολοκλήρωση αυτών των δώδεκα ασκήσεων από άκρη σε άκρη θα σας έχει περάσει από κάθε εντολή αυτού του οδηγού τουλάχιστον μία φορά και, πιο χρήσιμα, από τον συλλογισμό πίσω από καθεμία από αυτές — που είναι το κομμάτι που μεταφέρεται σε έναν διαφορετικό στόχο, μια διαφορετική υπηρεσία και μια διαφορετική αποστολή.
