# CSEC 101 — Εργαστηριακό Υλικό Κυβερνοασφάλειας

Συλλογή εργαστηριακών ασκήσεων που καλύπτουν σταδιακά τις βασικές δεξιότητες σχετικά με τη χρήσης της γραμμής εντολών έως την εκτέλεση ολοκληρωμένων δοκιμών παρείσδυσης (penetration testing) σε ευάλωτες εικονικές μηχανές.

---

## Περιεχόμενα

| # | Εργαστήριο | Θέμα | Επίπεδο | Τεκμηρίωση |
| :--- | :--- | :--- | :--- | :--- |
| 00 | **Virtualization** | Hypervisors, εισαγωγή OVA, snapshots, λειτουργίες δικτύου VM | Αρχάριος | [README.md](00.%20Virtualization/README.md) |
| 01 | **Εισαγωγή στο Linux CLI** | Πλοήγηση συστήματος αρχείων, εντολές, δίκτυο, δικαιώματα | Αρχάριος | [README.md](01.%20Εισαγωγή%20στο%20Linux%20CLI/README.md) |
| 02 | **Raven-CTF** | Write-up: αναγνώριση, WordPress, SSH, MySQL, privilege escalation | Αρχάριος | [README.md](02.%20Raven-CTF/README.md) |
| 03 | **DC-CTF** | Walkthrough: Drupalgeddon (CVE-2014-3704) → SUID `find` → root | Αρχάριος | [README.md](03.%20DC-CTF/README.md) |
| 04 | **Raven2-CTF** | Write-up: PHPMailer RCE (CVE-2016-10033) → MySQL UDF → SUID `find` | Μεσαίο | [README.md](04.%20Raven2-CTF/README.md) |
| 05 | **Mr Robot** | Write-up: WordPress enumeration → reverse shell → SUID nmap | Μεσαίο | [README.md](05.%20Mr%20Robot/README.md) |

---

## Δομή Φακέλου

```
new/
├── README.md                        ← το παρόν αρχείο (ευρετήριο)
├── 00. Virtualization/
│   ├── README.md                    Hypervisor, OVA/OVF, snapshots, δικτύωση VM
│   └── Detailed-Notes.md            Αναλυτικές σημειώσεις με θεωρία
├── 01. Εισαγωγή στο Linux CLI/
│   ├── README.md                    Θεωρία + 4 μέρη ασκήσεων με ερωτήματα
│   └── Detailed-Notes.md            Αναλυτικές σημειώσεις + απαντήσεις
├── 02. Raven-CTF/
│   ├── README.md                    Write-up σε 8 βήματα
│   ├── Detailed-Notes.md            Αναλυτικές σημειώσεις ανά βήμα
│   └── images/                      15 στιγμιότυπα οθόνης (image*.png)
├── 03. DC-CTF/
│   ├── README.md                    Εγχειρίδιο 4 φάσεων + μέτρα αποκατάστασης
│   ├── Detailed-Notes.md            Αναλυτικές σημειώσεις ανά φάση
│   └── images/                      13 στιγμιότυπα οθόνης (image*.png)
├── 04. Raven2-CTF/
│   ├── README.md                    Write-up σε 6 ενότητες
│   ├── Detailed-Notes.md            Αναλυτικές σημειώσεις ανά ενότητα
│   └── images/                      13 στιγμιότυπα οθόνης (image*.png)
└── 05. Mr Robot/
    ├── README.md                    Write-up: 6 φάσεις, 32 εικόνες
    └── images/                      32 στιγμιότυπα (από το writeup)
```

---

## 00. Virtualization

Εισαγωγή στην εικονικοποίηση και πρακτική ρύθμιση του **Oracle VM VirtualBox**: πώς λειτουργεί ο hypervisor κάτω από τις εικονικές μηχανές, πώς εισάγεται ένα έτοιμο image και πώς στήνεται σωστά το δίκτυο του εργαστηρίου. Είναι το προαπαιτούμενο βήμα για όλα τα υπόλοιπα εργαστήρια, αφού τόσο η μηχανή επίθεσης (Kali) όσο και οι στόχοι (Raven 1, DC-1, Raven 2) εκτελούνται ως VMs.

**Θεματικές ενότητες:**

- **Hypervisors:** διάκριση **Τύπου 1 (bare-metal** — ESXi, KVM, Xen, Proxmox) και **Τύπου 2 (hosted** — VirtualBox, VMware Workstation), με υποστήριξη υλικού **Intel VT-x/EPT** και **AMD-V/NPT**. Περιλαμβάνεται διάγραμμα της στοίβας host → hypervisor → guest VMs.
- **Πακέτα OVF/OVA:** η μορφή **Open Virtualization Format** (DMTF DSP0243) και γιατί το **OVA** είναι το ίδιο πακέτο σε ένα μόνο αρχείο `tar` — με manifest ακεραιότητας (`.mf`), περιγραφή υλικού (`.ovf`) και εικόνες δίσκου (`.vmdk`/`.vhd`).
- **Εισαγωγή OVA στο VirtualBox:** `File → Import Appliance…` (`Ctrl+I`), έλεγχος CPU/RAM (έως ~50–60% της μνήμης του host), επιλογή φακέλου αποθήκευσης και **Generate new MAC addresses** όταν πρόκειται για κλώνο.
- **Snapshots & δίσκοι:** copy-on-write **differencing images**, γιατί το snapshot δεν είναι αντίγραφο ασφαλείας και γιατί οι μεγάλες αλυσίδες snapshots επιβραδύνουν τη μηχανή.
- **Λειτουργίες δικτύωσης:** αναλυτική σύγκριση **NAT**, **NAT Network**, **Host-Only** και **Bridged**, με βήμα-προς-βήμα δημιουργία κοινού NAT Network (`Tools → Network → NAT Networks`).

**Πίνακας επιλογής δικτύου (σύνοψη):**

| Λειτουργία | VM → Internet | Host → VM άμεσα | VM ↔ VM | Τυπική χρήση |
| :--- | :---: | :---: | :---: | :--- |
| **NAT** (προεπιλογή) | Ναι | Όχι (port forwarding) | Όχι | Μεμονωμένη μηχανή με πρόσβαση στο internet |
| **NAT Network** | Ναι | Όχι (port forwarding) | Ναι | Εργαστήρια πολλών κόμβων με updates |
| **Host-Only** | Όχι | Ναι | Ναι | Απομονωμένο lab, ανάλυση malware |
| **Bridged** | Ναι | Ναι | Ναι | Η VM ως ισότιμη συσκευή στο φυσικό LAN |

> ⚠️ **NAT ≠ NAT Network.** Η απλή επιλογή **NAT** δίνει σε κάθε μηχανή το δικό της ιδιωτικό υποδίκτυο, οπότε δύο VMs δεν βλέπονται μεταξύ τους. Για εργαστήριο με πολλές μηχανές επιλέγετε **Attached to: NAT Network** και το όνομα του δικτύου που δημιουργήσατε.

**Γρήγορη εκκίνηση:** κατεβάστε το έτοιμο image του **Kali Linux** για VirtualBox από το [kali.org/get-kali](https://www.kali.org/get-kali/#kali-platforms) (σε macOS προτιμήστε το UTM), βεβαιωθείτε ότι είναι ενεργή η εικονικοποίηση στο BIOS/UEFI (Intel VT-x / AMD-V) και εισάγετε το OVA — χωρίς εγκατάσταση.

📘 **Αναλυτικές σημειώσεις:** [Detailed-Notes.md](00.%20Virtualization/Detailed-Notes.md) — θεωρία, ανάλυση κάθε βήματος, πίνακες διάγνωσης, γλωσσάρι και ερωτήσεις αυτοελέγχου.

---

## 01. Εισαγωγή στο Linux CLI

Εργαστηριακή άσκηση με στόχο την εξοικείωση με το περιβάλλον γραμμής εντολών και τις βασικές εντολές δικτύου.

📘 **Αναλυτικές σημειώσεις:** [Detailed-Notes.md](01.%20Εισαγωγή%20στο%20Linux%20CLI/Detailed-Notes.md) — επεξήγηση κάθε εντολής, θεωρία δικτύου, **απαντήσεις στα ερωτήματα** και γλωσσάρι.

**Θεματικές ενότητες:**

- **Μέρος Α — Πλοήγηση:** `pwd`, `mkdir -p`, `cd` με απόλυτες/σχετικές διαδρομές, `cd ..`, `cd ../..`, `cd -`, καθώς και `ls` με παραμέτρους `-l`, `-a`, `-h`, `-t`, `-r`, `-R` και wildcards.
- **Μέρος Β — Φιλτράρισμα & Pipes:** `grep`, ανακατεύθυνση εξόδου (`>`), διασωλήνωση (`|`), `wc`.
- **Μέρος Γ — Έλεγχος δικτύου:** `ifconfig` / `ip addr show`, `ping`, `ss -tuln`, `nslookup`, `traceroute`.
- **Μέρος Δ — Δικαιώματα αρχείων:** `chmod +x`, αριθμητική μορφή `755`, ανάλυση owner/group/others.

**Παραδοτέο:** απαντήσεις στα ερωτήματα 1.1, 1.2, 2.1, 3.1, 3.2 και 4.1 που εμφανίζονται μέσα στο κείμενο.

**Απαιτήσεις:** Οποιαδήποτε διανομή Linux (ή VM), βασικές εντολές δικτύου, δικαιώματα `sudo` για ορισμένες ενέργειες.

---

## 02. Raven-CTF

Μεταφρασμένο και εμπλουτισμένο write-up για τη μηχανή **Raven 1** της VulnHub.

📘 **Αναλυτικές σημειώσεις:** [Detailed-Notes.md](02.%20Raven-CTF/Detailed-Notes.md) — μεθοδολογία, ανάλυση κάθε βήματος και εικόνας, παγίδες και ερωτήσεις αυτοελέγχου.

**Ροή επίλυσης:**

1. **Αναγνώριση δικτύου** — `netdiscover` / `nmap -sn` για εντοπισμό της IP του στόχου.
2. **Σάρωση θυρών** — `nmap -sC -sV` (SSH, HTTP και μία ακόμη υπηρεσία).
3. **Ανάλυση ιστοσελίδας** — Developer Tools (F12), σχόλια HTML → **Flag 1**.
4. **Directory fuzzing** — DirBuster / Gobuster → ανακάλυψη καταλόγου `wordpress`.
5. **WPScan** — έκδοση WordPress, plugins, απαρίθμηση χρηστών (`-e u`).
6. **SSH access** — brute-force με `hydra` σε λεξικό `rockyou.txt` → αδύναμος κωδικός (`michael:michael`) → **Flag 2** στο `/var/www/html/`.
7. **Βάση δεδομένων** — `wp-config.php` → MySQL → `wp_posts` → **Flags 3 & 4**.
8. **Root** — hashes από `wp_users` → John the Ripper (`rockyou.txt`) → `steven` → `root`.

**Εργαλεία:** `netdiscover`, `nmap`, `DirBuster`/`gobuster`, `WPScan`, `hydra`, `ssh`, `mysql`, `john`.
**Πηγή:** [VulnHub — Raven: 1](https://www.vulnhub.com/entry/raven-1,256/) · [Writeup: Hariharan](https://corruptedprotocol.medium.com/raven-1-ctf-write-up-1ddaf2cff9e2)

---

## 03. DC-CTF

Πλήρες εγχειρίδιο δοκιμής παρείσδυσης στη μηχανή **DC-1**, δομημένο σε τέσσερις μεθοδολογικές φάσεις.

📘 **Αναλυτικές σημειώσεις:** [Detailed-Notes.md](03.%20DC-CTF/Detailed-Notes.md) — θεωρία Drupalgeddon, Metasploit, PTY, SUID και CVSS, με γλωσσάρι και ερωτήσεις.

**Φάσεις:**

1. **Αναγνώριση & Ανακάλυψη Υπηρεσιών**
   - Απομονωμένο εικονικό δίκτυο (Host-Only / NAT).
   - `netdiscover -r 192.168.35.0/24` — εντοπισμός κόμβων μέσω ARP.
   - `nmap -sV -p- 192.168.35.131` — πλήρης σάρωση 65.535 θυρών.
   - Ταυτοποίηση Apache / Debian Wheezy / **Drupal 7.x**.
2. **Ανάλυση Ευπαθειών & Εκμετάλλευση**
   - `msfconsole` → `search drupal` → `exploit/multi/http/drupal_drupageddon`.
   - **CVE-2014-3704** (SQL Injection → RCE), CVSS **9.8 Critical**.
   - Άνοιγμα συνεδρίας Meterpreter.
3. **Μετα-Εκμετάλλευση**
   - `shell` → αναβάθμιση σε PTY: `python -c 'import pty; pty.spawn("/bin/bash")'`.
   - Εντοπισμός `flag1.txt` στο `/var/www` και οδηγιών για το `settings.php`.
4. **Κλιμάκωση Προνομίων**
   - `find / -perm -u=s -type f 2>/dev/null` — εντοπισμός SUID binaries.
   - Κατάχρηση SUID στο `/usr/bin/find`: `/usr/bin/find . -exec /bin/sh \; -quit`.
   - Επαλήθευση με `whoami` και ανάκτηση του `thefinalflag.txt` από το `/root`.

**Τεχνική σύνοψη & αποκατάσταση:** πίνακας επικινδυνότητας (CVSS v3) και μέτρα hardening — patch management, WAF/ModSecurity με OWASP CRS, αφαίρεση SUID (`chmod u-s /usr/bin/find`), έλεγχοι ακεραιότητας (AIDE/Tripwire), AppArmor/SELinux.

**Εργαλεία:** `netdiscover`, `nmap`, Metasploit Framework, Meterpreter, `find`, `cat`, `whoami`.
**Πηγή:** [VulnHub — DC: 1](https://www.vulnhub.com/entry/dc-1,292/) · [Writeup: Hacking Articles](https://www.hackingarticles.in/dc-1-vulnhub-walkthrough/)

---

## 04. Raven2-CTF

Write-up δοκιμής διείσδυσης στη μηχανή **Raven 2** (VulnHub), επιπέδου *Beginner/Intermediate*, με στόχο την ανάκτηση 4 σημαιών και την απόκτηση δικαιωμάτων root.

📘 **Αναλυτικές σημειώσεις:** [Detailed-Notes.md](04.%20Raven2-CTF/Detailed-Notes.md) — PHPMailer RCE, MySQL UDF, SUID, με αναλυτική εξήγηση κάθε εντολής.

**Ροή επίλυσης:**

1. **Αναγνώριση δικτύου** — `netdiscover` για εντοπισμό του στόχου (`192.168.1.101`), `nmap -A` για OS/version detection (22 SSH, 80 HTTP, 111 RPC).
2. **Απαρίθμηση ιστού** — `dirb` → κατάλογος `/vendor/`, όπου το αρχείο `PATH` περιέχει το **Flag 1** και το `VERSION` αποκαλύπτει **PHPMailer 5.2.6**.
3. **Αρχική πρόσβαση** — Exploit-DB **40974** / **CVE-2016-10033** (PHPMailer < 5.2.18 RCE) σε Python: `nc -lvp 4444` για αντίστροφο κέλυφος, αναβάθμιση σε TTY με `pty.spawn`, ανάκτηση **Flag 2** από το `/var/www`.
4. **Απαρίθμηση συστήματος** — `wp-config.php` → διαπιστευτήρια MySQL (`R@v3nSecurity`), `LinEnum.sh` για εντοπισμό ευπαθειών.
5. **MySQL UDF (CVE-2016-6662)** — μεταγλώττιση του `1518.c` (`gcc -shared`), φόρτωση ως plugin μέσω `dumpfile` και κλήση `do_system('chmod u+s /usr/bin/find')`.
6. **Root** — `find raj -exec "/bin/sh" \;` → κέλυφος root, **Flag 4** στο `/root` και **Flag 3** στο `wp-content/uploads/2018/11`.

**Εργαλεία:** `netdiscover`, `nmap`, `dirb`, Exploit-DB / `searchsploit`, `nc`, MySQL client, `gcc`, `find`.
**Πηγή:** [VulnHub — Raven: 2](https://www.vulnhub.com/entry/raven-2,269/) · [Writeup: Hacking Articles](https://www.hackingarticles.in/raven-2-vulnhub-walkthrough/)


---

## 05. Mr Robot

Πλήρες write-up για τη μηχανή **Mr. Robot** (TryHackMe): από την απαρίθμηση του WordPress μέχρι το `root`, με 32 στιγμιότυπα οθόνης, ανάλυση κάθε εντολής και επεξήγηση κάθε εικόνας.

📘 **Αναλυτικό κείμενο:** [README.md](05.%20Mr%20Robot/README.md)

**Ροή επίλυσης:**

1. **Αναγνώριση** — `nmap -sV -T4` → μόνο 80/443, Apache httpd.
2. **Απαρίθμηση** — `gobuster` → `/wp-login`, `/robots.txt`· το `robots.txt` δίνει **Key 1** και το λεξικό `fsocity.dic`.
3. **Καθαρισμός λεξικού** — `sort | uniq` από 858.160 σε **11.451** λέξεις (-98,6%).
4. **Διαπιστευτήρια** — διαφορετικά μηνύματα σφάλματος (user enumeration) + Burp Proxy + `hydra` (δύο περάσματα) → `elliot:ER28-0652`.
5. **Αρχική πρόσβαση** — Theme Editor → PHP reverse shell (`archive.php`) → `nc -lnvp 1234` → shell ως `daemon`.
6. **Pivoting** — `password.raw-md5` → CrackStation/`john` → `su robot` (με `pty.spawn`) → **Key 2**.
7. **Κλιμάκωση** — SUID `/usr/local/bin/nmap` → GTFOBins `--interactive` + `!sh` → root → **Key 3**.
8. **Εναλλακτικές** — Burp Intruder, `wpscan`, John the Ripper, Hashcat.

**Εργαλεία:** `nmap`, `gobuster`, Burp Suite, `hydra`, `wpscan`, `nc`, `john`, `hashcat`, GTFOBins.
**Πηγή:** [Mr Robot CTF Walkthrough (2023) — Charalampos Spanias](https://medium.com/@cspanias/thms-mr-robot-ctf-walkthrough-2023-55ca5c19fbaf) · [TryHackMe room](https://tryhackme.com/room/mrrobot)

---

## Απαιτήσεις Περιβάλλοντος

- **Λογισμικό:** VirtualBox ή VMware Workstation (βλ. εργαστήριο [00](00.%20Virtualization/README.md)).
- **Εικονικοποίηση υλικού:** ενεργή στο BIOS/UEFI (Intel VT-x / AMD-V).
- **Επιτιθέμενη μηχανή:** Kali Linux (ή οποιαδήποτε διανομή με τα εργαλεία των ασκήσεων).
- **Στόχοι:** εικόνες VM από [VulnHub](https://www.vulnhub.com/) (Raven 1, DC-1, Raven 2) και το room [Mr Robot](https://tryhackme.com/room/mrrobot) στο TryHackMe.
- **Δίκτυο:** απομονωμένο NAT Network ή Host-Only.
- **Γνώσεις:** άνεση με terminal, βασικές έννοιες TCP/IP.

---

## Συνιστώμενη Σειρά Εκτέλεσης

```
00. Virtualization   →   01. Εισαγωγή στο Linux CLI   →   02. Raven-CTF   →   03. DC-CTF   →   04. Raven2-CTF   →   05. Mr Robot
   (στήσιμο VMs,             (θεμέλια: CLI, δίκτυο)          (πρώτη CTF        (μεθοδολογία       (web RCE +          (WordPress →
    δίκτυο, snapshots)                                       εμπειρία)         pentest)           privesc UDF)         root)
```

Το εργαστήριο 00 είναι προαπαιτούμενο για όλα: χωρίς εικονική μηχανή και σωστά ρυθμισμένο δίκτυο δεν εκτελείται κανένα από τα επόμενα. Το 01 με τη σειρά του παρέχει τις εντολές (`ls`, `grep`, `chmod`, `ss`, `ping`) που χρησιμοποιούνται άμεσα στα 02, 03, 04 και 05. Το 04 είναι η φυσική συνέχεια του 02 (ίδια σειρά Raven, δυσκολότερη διαδρομή root)· το 05 συνδυάζει enumerations ιστού, brute-force και SUID σε ένα ενιαίο σενάριο.

---

## Νομική Σημείωση & Δεοντολογία

Όλες οι τεχνικές και τα εργαλεία που περιγράφονται εδώ προορίζονται **αποκλειστικά για εκπαιδευτική χρήση** σε ελεγχόμενα, απομονωμένα περιβάλλοντα και σε συστήματα για τα οποία υπάρχει ρητή άδεια.

Η μη εξουσιοδοτημένη πρόσβαση σε συστήματα τρίτων αποτελεί ποινικό αδίκημα. Εκτελείτε τις ασκήσεις μόνο σε τοπικές εικονικές μηχανές ή σε πλατφόρμες που έχουν σχεδιαστεί γι' αυτόν τον σκοπό.

---

## Πηγές & Αναφορές

### Εργαστήριο 00 — Virtualization
- [Oracle VM VirtualBox — Documentation](https://www.virtualbox.org/manual/)
- [DMTF — Open Virtualization Format (DSP0243)](https://www.dmtf.org/standards/ovf)
- [Kali Linux — Downloads (VirtualBox images)](https://www.kali.org/get-kali/#kali-platforms)

### Εργαστήριο 02 — Raven: 1
- **Λήψη μηχανήματος:** [VulnHub — Raven: 1](https://www.vulnhub.com/entry/raven-1,256/)
- **Writeup αναφοράς:** [Raven 1 CTF Write-up — Hariharan (corruptedprotocol.medium.com)](https://corruptedprotocol.medium.com/raven-1-ctf-write-up-1ddaf2cff9e2)
- **Εργαλεία:** [WPScan](https://wpscan.com/), [OWASP DirBuster/Gobuster](https://www.kali.org/tools/gobuster/), [John the Ripper](https://www.openwall.com/john/)

### Εργαστήριο 03 — DC: 1
- **Λήψη μηχανήματος:** [VulnHub — DC: 1](https://www.vulnhub.com/entry/dc-1,292/)
- **Writeup αναφοράς:** [DC-1: Vulnhub Walkthrough — Hacking Articles](https://www.hackingarticles.in/dc-1-vulnhub-walkthrough/)
- **Ευπάθεια:** [CVE-2014-3704 — Drupalgeddon (NVD)](https://nvd.nist.gov/vuln/detail/CVE-2014-3704)
- **Module:** [Metasploit — `exploit/multi/http/drupal_drupageddon`](https://www.rapid7.com/db/modules/exploit/multi/http/drupal_drupageddon/)
- **Αποκατάσταση:** [OWASP Core Rule Set (ModSecurity)](https://owasp.org/www-project-modsecurity-core-rule-set/)

### Εργαστήριο 04 — Raven: 2
- **Λήψη μηχανήματος:** [VulnHub — Raven: 2](https://www.vulnhub.com/entry/raven-2,269/)
- **Writeup αναφοράς:** [Raven 2: Vulnhub Walkthrough — Hacking Articles](https://www.hackingarticles.in/raven-2-vulnhub-walkthrough/)
- **Ευπάθειες:** [CVE-2016-10033 — PHPMailer RCE (NVD)](https://nvd.nist.gov/vuln/detail/CVE-2016-10033) · [CVE-2016-6662 — MySQL/MariaDB privilege escalation (NVD)](https://nvd.nist.gov/vuln/detail/CVE-2016-6662)
- **Exploit:** [Exploit-DB 40974 — PHPMailer < 5.2.18 RCE (Python)](https://www.exploit-db.com/exploits/40974)

### Εργαστήριο 05 — Mr Robot
- **Writeup αναφοράς:** [Mr Robot CTF Walkthrough (2023) — Charalampos Spanias](https://medium.com/@cspanias/thms-mr-robot-ctf-walkthrough-2023-55ca5c19fbaf)
- **Πλατφόρμα:** [TryHackMe — Mr Robot CTF](https://tryhackme.com/room/mrrobot)
- **GTFOBins:** [nmap](https://gtfobins.github.io/gtfobins/nmap/)
- **Reverse shell:** [pentestmonkey/php-reverse-shell](https://github.com/pentestmonkey/php-reverse-shell)

### Γενικά
- [Metasploit Framework](https://www.metasploit.com/)
- [Nmap — Reference Guide](https://nmap.org/book/man.html)
- [VulnHub — Vulnerable Machines](https://www.vulnhub.com/)
