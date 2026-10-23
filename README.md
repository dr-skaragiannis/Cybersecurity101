# CSEC 101 — Εργαστηριακό Υλικό Κυβερνοασφάλειας

Συλλογή εργαστηριακών ασκήσεων που καλύπτουν σταδιακά τη διαδρομή από τις βασικές δεξιότητες γραμμής εντολών έως την εκτέλεση ολοκληρωμένων δοκιμών παρείσδυσης (penetration testing) σε ευάλωτες εικονικές μηχανές.

---

## Περιεχόμενα

| # | Εργαστήριο | Θέμα | Επίπεδο | Τεκμηρίωση |
| :--- | :--- | :--- | :--- | :--- |
| 00 | **Virtualization** | Hypervisors, εισαγωγή OVA, snapshots, λειτουργίες δικτύου VM | Αρχάριος | [README.md](00.%20Virtualization/README.md) |
| 01 | **Εισαγωγή στο Linux CLI** | Πλοήγηση συστήματος αρχείων, εντολές, δίκτυο, δικαιώματα | Αρχάριος | [README.md](01.%20Εισαγωγή%20στο%20Linux%20CLI/README.md) |
| 02 | **Raven-CTF** | Write-up: αναγνώριση, WordPress, SSH, MySQL, privilege escalation | Μεσαίο | [README.md](02.%20Raven-CTF/README.md) |
| 03 | **DC-CTF** | Walkthrough: Drupalgeddon (CVE-2014-3704) → SUID `find` → root | Προχωρημένο | [README.md](03.%20DC-CTF/README.md) |

---

## Δομή Φακέλου

```
new/
├── README.md                        ← το παρόν αρχείο (ευρετήριο)
├── 00. Virtualization/
│   └── README.md                    Hypervisor, OVA/OVF, snapshots, δικτύωση VM
├── 01. Εισαγωγή στο Linux CLI/
│   └── README.md                    Θεωρία + 4 μέρη ασκήσεων με ερωτήματα
├── 02. Raven-CTF/
│   ├── README.md                    Write-up σε 8 βήματα
│   └── images/                      13 στιγμιότυπα οθόνης (image*.png)
└── 03. DC-CTF/
    ├── README.md                    Εγχειρίδιο 4 φάσεων + μέτρα αποκατάστασης
    └── images/                      13 στιγμιότυπα οθόνης (image*.png)
```

---

## 00. Virtualization

Εισαγωγή στην εικονικοποίηση και πρακτική ρύθμιση του **Oracle VM VirtualBox**: πώς λειτουργεί ο hypervisor κάτω από τις εικονικές μηχανές, πώς εισάγεται ένα έτοιμο image και πώς στήνεται σωστά το δίκτυο του εργαστηρίου. Είναι το προαπαιτούμενο βήμα για όλα τα υπόλοιπα εργαστήρια, αφού τόσο η μηχανή επίθεσης (Kali) όσο και οι στόχοι (Raven 1, DC-1) εκτελούνται ως VMs.

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

---

## 01. Εισαγωγή στο Linux CLI

Εργαστηριακή άσκηση με στόχο την εξοικείωση με το περιβάλλον γραμμής εντολών και τις βασικές εντολές δικτύου.

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

**Ροή επίλυσης:**

1. **Αναγνώριση δικτύου** — `netdiscover` / `nmap -sn` για εντοπισμό της IP του στόχου.
2. **Σάρωση θυρών** — `nmap -sC -sV` (SSH, HTTP και μία ακόμη υπηρεσία).
3. **Ανάλυση ιστοσελίδας** — Developer Tools (F12), σχόλια HTML → **Flag 1**.
4. **Directory fuzzing** — DirBuster / Gobuster → ανακάλυψη καταλόγου `wordpress`.
5. **WPScan** — έκδοση WordPress, plugins, απαρίθμηση χρηστών (`-e u`).
6. **SSH access** — αδύναμος κωδικός (`michael:michael`) → **Flag 2** στο `/var/www/html/`.
7. **Βάση δεδομένων** — `wp-config.php` → MySQL → `wp_posts` → **Flags 3 & 4**.
8. **Root** — hashes από `wp_users` → John the Ripper (`rockyou.txt`) → `steven` → `root`.

**Εργαλεία:** `netdiscover`, `nmap`, `DirBuster`/`gobuster`, `WPScan`, `ssh`, `mysql`, `john`.
**Πηγή:** [VulnHub — Raven: 1](https://www.vulnhub.com/entry/raven-1,256/)

---

## 03. DC-CTF

Πλήρες εγχειρίδιο δοκιμής παρείσδυσης στη μηχανή **DC-1**, δομημένο σε τέσσερις μεθοδολογικές φάσεις.

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

---

## Απαιτήσεις Περιβάλλοντος

- **Λογισμικό υπερεπόπτη:** VirtualBox ή VMware Workstation (βλ. εργαστήριο [00](00.%20Virtualization/README.md)).
- **Εικονικοποίηση υλικού:** ενεργή στο BIOS/UEFI (Intel VT-x / AMD-V).
- **Επιτιθέμενη μηχανή:** Kali Linux (ή οποιαδήποτε διανομή με τα εργαλεία των ασκήσεων).
- **Στόχοι:** εικόνες VM από [VulnHub](https://www.vulnhub.com/) (Raven 1, DC-1).
- **Δίκτυο:** απομονωμένο NAT Network ή Host-Only.
- **Γνώσεις:** άνεση με terminal, βασικές έννοιες TCP/IP.

---

## Συνιστώμενη Σειρά Εκτέλεσης

```
00. Virtualization   →   01. Εισαγωγή στο Linux CLI   →   02. Raven-CTF   →   03. DC-CTF
   (στήσιμο VMs,             (θεμέλια: CLI, δίκτυο)          (πρώτη CTF        (μεθοδολογία
    δίκτυο, snapshots)                                       εμπειρία)         pentest)
```

Το εργαστήριο 00 είναι προαπαιτούμενο για όλα: χωρίς εικονική μηχανή και σωστά ρυθμισμένο δίκτυο δεν εκτελείται κανένα από τα επόμενα. Το 01 με τη σειρά του παρέχει τις εντολές (`ls`, `grep`, `chmod`, `ss`, `ping`) που χρησιμοποιούνται άμεσα στα 02 και 03.

---

## Νομική Σημείωση & Δεοντολογία

Όλες οι τεχνικές και τα εργαλεία που περιγράφονται εδώ προορίζονται **αποκλειστικά για εκπαιδευτική χρήση** σε ελεγχόμενα, απομονωμένα περιβάλλοντα και σε συστήματα για τα οποία υπάρχει ρητή άδεια.

Η μη εξουσιοδοτημένη πρόσβαση σε συστήματα τρίτων αποτελεί ποινικό αδίκημα. Εκτελείτε τις ασκήσεις μόνο σε τοπικές εικονικές μηχανές ή σε πλατφόρμες που έχουν σχεδιαστεί γι' αυτόν τον σκοπό.

---

## Πηγές & Αναφορές

- [VulnHub — Raven: 1](https://www.vulnhub.com/entry/raven-1,256/)
- [Raven 1 CTF Write-up — Hariharan (corruptedprotocol.medium.com)](https://corruptedprotocol.medium.com/raven-1-ctf-write-up-1ddaf2cff9e2)
- [CVE-2014-3704 — Drupalgeddon](https://nvd.nist.gov/vuln/detail/CVE-2014-3704)
- [OWASP Core Rule Set](https://owasp.org/www-project-modsecurity-core-rule-set/)
- [Metasploit Framework](https://www.metasploit.com/)
- [Oracle VM VirtualBox — Documentation](https://www.virtualbox.org/manual/)
- [DMTF — Open Virtualization Format (DSP0243)](https://www.dmtf.org/standards/ovf)
