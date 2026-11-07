# Bandit — OverTheWire Wargame Walkthrough

Το **Bandit** είναι ένα εκπαιδευτικό παιχνίδι (wargame) **34 επιπέδων** της κοινότητας **OverTheWire**, το οποίο εκτελείται σε πραγματικό εξυπηρετητή Linux. Σε κάθε επίπεδο συνδέεστε μέσω **SSH** ως διαφορετικός χρήστης (`bandit0`, `bandit1`, …) και καλείστε να εντοπίσετε τον **κωδικό πρόσβασης του επόμενου χρήστη**. Κάθε επίπεδο εισάγει μία νέα εντολή ή έννοια του Linux, οπότε το Bandit αποτελεί τη φυσική συνέχεια του εργαστηρίου [01. Εισαγωγή στο Linux CLI](../01.%20Εισαγωγή%20στο%20Linux%20CLI/README.md): εκεί μαθαίνετε τις εντολές, εδώ τις εφαρμόζετε σε απομακρυσμένο σύστημα για να λύσετε συγκεκριμένα προβλήματα.

Ο οδηγός αυτός είναι ελληνική, εμπλουτισμένη απόδοση του write-up του Alexandre Cheron.

> - **Πλατφόρμα:** [OverTheWire — Bandit](https://overthewire.org/wargames/bandit/)
> - **Writeup στα Αγγλικά:** [[OTW] Write-up for the Bandit Wargame — A. Cheron](https://axcheron.github.io/writeups/otw/bandit/)
> - **Εγχειρίδιο (PDF):** [Lab 02 - Bandit.pdf](Lab%2002%20-%20Bandit.pdf)

#### **Προσοχή! Οι κωδικοί που εμφανίζονται στις εξόδους των εντολών προέρχονται από το 2018. Το OverTheWire τους αλλάζει περιοδικά, επομένως δεν είναι πλέον έγκυροι. Το εκπαιδευτικό ενδιαφέρον βρίσκεται στη μέθοδο επίλυσης και όχι στη συμβολοσειρά. Επιχειρήστε πρώτα να λύσετε κάθε επίπεδο μόνοι σας και ανατρέξτε στη λύση μόνο αν δεν μπορείτε να προχωρήσετε.**

---

## Στόχοι του Εργαστηρίου

Με την ολοκλήρωση του Bandit θα μπορείτε να:

1. Συνδέεστε σε απομακρυσμένα συστήματα μέσω `ssh` ή PuTTY και να μεταφέρετε αρχεία μέσω `scp`, με κωδικό ή ιδιωτικό κλειδί.
2. Εντοπίζετε αρχεία βάσει ονόματος, μεγέθους, ιδιοκτήτη και δικαιωμάτων (`find`, `file`, `ls -la`).
3. Φιλτράρετε και μετασχηματίζετε κείμενο μέσω διασωληνώσεων (`grep`, `sort`, `uniq`, `tr`, `base64`).
4. Επικοινωνείτε με δικτυακές υπηρεσίες μέσω `nc` και `openssl s_client` και να εκτελείτε σάρωση θυρών.
5. Κατανοείτε πώς τα εκτελέσιμα **SUID** και οι εργασίες **cron** μπορούν να οδηγήσουν σε κλιμάκωση δικαιωμάτων.
6. Αξιοποιείτε το ιστορικό του **git** για την ανάκτηση δεδομένων που έχουν αφαιρεθεί.
7. Διαφεύγετε από περιορισμένα κελύφη (`more` → `vi`, `$0`).

---

## Δομή του Παιχνιδιού

| Θεματική ενότητα | Επίπεδα και περιεχόμενο |
| :--- | :--- |
| **Αρχεία και ονόματα** | 0–6: `ls`, `cat`, κρυφά αρχεία, ειδικά ονόματα αρχείων, `file`, `find` |
| **Επεξεργασία κειμένου** | 7–12: `grep`, `sort \| uniq`, `strings`, Base64, ROT13, συμπίεση |
| **Δίκτυο και κλειδιά** | 13–20: κλειδιά SSH, `nc`, `openssl`, σάρωση θυρών, `diff`, SUID |
| **Αυτοματοποίηση** | 21–24: `cron`, σενάρια bash, εξαντλητική αναζήτηση |
| **Διαφυγή και Git** | 25–33: διαφυγή μέσω `more` / `vi`, ιστορικό, κλάδοι, ετικέτες, αποστολή αλλαγών, `$0` |

```
[ Σύνδεση SSH ]
       │
       ▼
[ 0–6   Αρχεία & ονόματα ]        ls, cat, file, find
       │
       ▼
[ 7–12  Επεξεργασία κειμένου ]    grep, sort|uniq, strings, base64, tr, xxd
       │
       ▼
[ 13–20 Δίκτυο & κλειδιά ]        ssh -i, nc, openssl, diff, SUID
       │
       ▼
[ 21–24 Αυτοματοποίηση ]          cron, σενάρια bash, brute force
       │
       ▼
[ 25–33 Διαφυγή & Git ]           more → vi, git log/branch/tag/push, $0
```

---

## Σύνδεση στον Εξυπηρετητή μέσω SSH

Η σύνδεση γίνεται μέσω του πρωτοκόλλου **SSH (Secure Shell)**, το οποίο παρέχει κρυπτογραφημένο κανάλι και πρόσβαση σε κέλυφος ενός απομακρυσμένου συστήματος. Ό,τι πληκτρολογείτε στο τοπικό τερματικό εκτελείται στον εξυπηρετητή.

| Παράμετρος | Τιμή |
| :--- | :--- |
| **Εξυπηρετητής (host)** | `bandit.labs.overthewire.org` |
| **Θύρα (port)** | `2220` (και όχι η προεπιλεγμένη θύρα 22 του SSH) |
| **Όνομα χρήστη** | `bandit0` για το πρώτο επίπεδο, `banditN` για το επίπεδο N |
| **Κωδικός πρόσβασης** | `bandit0` για το πρώτο επίπεδο, οι επόμενοι εντοπίζονται κατά την επίλυση |

```
ssh   bandit0  @  bandit.labs.overthewire.org   -p 2220
│     │           │                             │
│     │           │                             └─ θύρα (όχι η προεπιλεγμένη 22)
│     │           └─ ο απομακρυσμένος εξυπηρετητής (host)
│     └─ ο χρήστης, δηλαδή το επίπεδο στο οποίο συνδέεστε
└─ το πρόγραμμα-πελάτης SSH
```

### Α. Από τερματικό Linux, macOS ή Windows 10/11

Τα Linux, το macOS και τα σύγχρονα Windows (PowerShell / Command Prompt) διαθέτουν ενσωματωμένο πελάτη OpenSSH:

```bash
$ ssh bandit0@bandit.labs.overthewire.org -p 2220
The authenticity of host '[bandit.labs.overthewire.org]:2220' can't be established.
ED25519 key fingerprint is SHA256:C2ihUBV7ihnV1wUXRb4RrEcLfXC5CXlhmAAM/urerLY.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
bandit0@bandit.labs.overthewire.org's password:     ← πληκτρολογήστε bandit0
bandit0@bandit:~$
```

1. **Επαλήθευση εξυπηρετητή:** στην πρώτη σύνδεση απαντήστε `yes`. Το κλειδί αποθηκεύεται στο `~/.ssh/known_hosts`.
2. **Κωδικός:** κατά την πληκτρολόγηση δεν εμφανίζονται χαρακτήρες. Αυτό είναι αναμενόμενο και αποτελεί μέτρο ασφαλείας.
3. **Επιτυχία:** η προτροπή `bandit0@bandit:~$` δείχνει ότι βρίσκεστε στον εξυπηρετητή.
4. **Αποσύνδεση:** `exit` ή `Ctrl + D`.

### Β. Από Windows με το PuTTY

Κατεβάστε το PuTTY **μόνο** από την [επίσημη τοποθεσία](https://www.chiark.greenend.org.uk/~sgtatham/putty/), καθώς κυκλοφορούν τροποποιημένες εκδόσεις με κακόβουλο λογισμικό. Το πακέτο περιλαμβάνει και τα **PuTTYgen** και **PSCP**, που χρειάζονται σε επόμενα επίπεδα.

1. Στο **Session**, Host Name: `bandit.labs.overthewire.org`, Port: `2220`, Connection type: **SSH**.
2. (Προαιρετικά) Saved Sessions: `bandit` → **Save**.
3. **Open** → στο *PuTTY Security Alert* επιλέξτε **Accept**.
4. `login as: bandit0` και στη συνέχεια ο κωδικός.

> **Αντιγραφή/επικόλληση στο PuTTY:** η επιλογή κειμένου με το ποντίκι το αντιγράφει, το δεξί κλικ το επικολλά. Το `Ctrl + C` διακόπτει την εκτελούμενη εντολή.

**Σύνδεση με ιδιωτικό κλειδί στο PuTTY (επίπεδα 13 και 16):** το PuTTY απαιτεί κλειδιά μορφής `.ppk`. Στο PuTTYgen: *Conversions → Import key* → *Save private key* (π.χ. `bandit14.ppk`), και στο PuTTY: *Connection → SSH → Auth → Credentials → Private key file*. Για μεταφορά αρχείων χρησιμοποιήστε `pscp -P 2220 bandit13@bandit.labs.overthewire.org:sshkey.private .` ή το WinSCP.

### Συνήθη σφάλματα σύνδεσης

| Μήνυμα | Αιτία και αντιμετώπιση |
| :--- | :--- |
| `Connection timed out` / `Connection refused` | Λανθασμένη θύρα (συνήθως παράλειψη του `-p 2220`) ή αποκλεισμός από το τείχος προστασίας. |
| `Permission denied, please try again.` | Λάθος κωδικός ή όνομα χρήστη. Οι κωδικοί διακρίνουν πεζά από κεφαλαία. |
| `UNPROTECTED PRIVATE KEY FILE!` | Το κλειδί είναι αναγνώσιμο από άλλους. Εκτελέστε `chmod 400 <αρχείο>`. |
| `REMOTE HOST IDENTIFICATION HAS CHANGED!` | Το κλειδί του εξυπηρετητή άλλαξε. Αν η αλλαγή είναι αναμενόμενη: `ssh-keygen -R "[bandit.labs.overthewire.org]:2220"`. |

---

## Μέρος Α: Αρχεία και Ονόματα (Επίπεδα 0–6)

### Επίπεδο 0 → 1: Αρχική σύνδεση
**Εντολές:** `ssh` · `ls -la` · `cat`

**Στόχος:** ο κωδικός βρίσκεται στο αρχείο `readme` του προσωπικού καταλόγου.

```bash
bandit0@bandit:~$ ls -la
-rw-r-----  1 bandit1 bandit0   33 Oct 16 14:00 readme
bandit0@bandit:~$ cat readme
boJ9jbbUNNfktd78OOpsqOltutMc3MY1
```

**Επεξήγηση — ερμηνεία μιας γραμμής της `ls -l`:**

```
-rw-r-----   bandit1   bandit0   33   readme
│└┬┘└┬┘└┬┘   │         │         │
│ │  │  │    │         │         └─ μέγεθος σε bytes
│ │  │  │    │         └─ ομάδα (group)
│ │  │  │    └─ ιδιοκτήτης (owner)
│ │  │  └─ λοιποί χρήστες: ---  (κανένα δικαίωμα)
│ │  └─ ομάδα: r--  (ανάγνωση) ← ο χρήστης bandit0
│ └─ ιδιοκτήτης: rw- (ανάγνωση/εγγραφή)
└─ τύπος: - αρχείο, d κατάλογος, l σύνδεσμος
```

### Επίπεδο 1 → 2: Αρχείο με όνομα «-»
**Εντολές:** `cat` · stdin · σχετική διαδρομή

```bash
bandit1@bandit:~$ cat ./-
CV1DtqXWVFXTvM2F0k09SHz0YwRINYA9
```

Για πολλές εντολές το όρισμα `-` σημαίνει «ανάγνωση από την τυπική είσοδο (stdin)», οπότε η `cat -` περιμένει είσοδο από το πληκτρολόγιο. Με τη διαδρομή `./-` δηλώνετε ρητά ότι πρόκειται για αρχείο του τρέχοντος καταλόγου.

### Επίπεδο 2 → 3: Ονόματα με κενά διαστήματα
**Εντολές:** quoting · escaping · αυτόματη συμπλήρωση

```bash
bandit2@bandit:~$ cat ./--   όνομα αρχείου.
UmHadQclWmgdLOKQ3YNgjWxGoRMb5luK
```

Το κέλυφος χωρίζει τα ορίσματα στα κενά, οπότε χωρίς εισαγωγικά η `cat` θα αναζητούσε τέσσερα αρχεία. Ισοδύναμα: `cat spaces\ in\ this\ filename`. Πρακτικά: πληκτρολογήστε `cat sp` και πατήστε `Tab`.

### Επίπεδο 3 → 4: Κρυφά αρχεία
**Εντολές:** `cd` · `ls -a` · dotfiles

```bash
bandit3@bandit:~$ cd inhere/
bandit3@bandit:~/inhere$ ls -la
-rw-r----- 1 bandit4 bandit3   33 Dec 28 14:34 .hidden
bandit3@bandit:~/inhere$ cat .hidden
pIwrPrtPN36QITSp3EQaw936yaFoFgAB
```

Στο Linux το «κρυφό αρχείο» είναι απλώς σύμβαση: κάθε όνομα που αρχίζει με τελεία παραλείπεται από την απλή `ls`. Η παράμετρος `-a` (all) τα εμφανίζει.

> 🛡️ **Ασφάλεια:** η `ls -la` είναι από τα πρώτα βήματα στην ανάλυση ενός συστήματος. Κακόβουλα αρχεία συχνά κρύβονται με ονόματα όπως `.cache` ή `...`.

### Επίπεδο 4 → 5: Εντοπισμός αρχείου κειμένου βάσει τύπου
**Εντολές:** `file` · χαρακτήρες μπαλαντέρ

```bash
bandit4@bandit:~/inhere$ file ./-file0*
./-file00: data
./-file07: ASCII text
...
bandit4@bandit:~/inhere$ cat ./-file07
koReBOKuIDDepwhWk7jZC0RTdopnAYKh
```

Η `file` εξετάζει τα πρώτα bytes ενός αρχείου («magic bytes») και αναγνωρίζει τον τύπο ανεξάρτητα από το όνομα ή την κατάληξη.

### Επίπεδο 5 → 6: Αναζήτηση βάσει ιδιοτήτων
**Εντολές:** `find` · `-size` · `-readable` · `! -executable`

**Στόχος:** αρχείο αναγνώσιμο, 1033 bytes, μη εκτελέσιμο.

```bash
bandit5@bandit:~$ find ./inhere/ -type f -readable ! -executable -size 1033c
./inhere/maybehere07/.file2
bandit5@bandit:~$ cat ./inhere/maybehere07/.file2
DXjZPULLxYr17uwoI01bNLQbtFemEgo7
```

| Παράμετρος | Σημασία |
| :--- | :--- |
| `-type f` | μόνο κανονικά αρχεία (όχι κατάλογοι) |
| `-readable` | αναγνώσιμα από τον τρέχοντα χρήστη |
| `! -executable` | το `!` αντιστρέφει το φίλτρο: μη εκτελέσιμα |
| `-size 1033c` | ακριβώς 1033 bytes (`c` = bytes, `k` = KiB, `M` = MiB) |

### Επίπεδο 6 → 7: Αναζήτηση σε ολόκληρο το σύστημα αρχείων
**Εντολές:** `find /` · `-user` · `-group` · `2>/dev/null`

**Στόχος:** αρχείο με ιδιοκτήτη `bandit7`, ομάδα `bandit6`, μέγεθος 33 bytes.

```bash
bandit6@bandit:~$ find / -type f -size 33c -group bandit6 -user bandit7 2>/dev/null
/var/lib/dpkg/info/bandit7.password
bandit6@bandit:~$ cat /var/lib/dpkg/info/bandit7.password
HKBPTKQnIay4Fw76bEy8PVxKEDQRKTzs
```

Όταν η αναζήτηση ξεκινά από τη ρίζα `/`, η `find` παράγει εκατοντάδες μηνύματα `Permission denied`. Αυτά γράφονται στη **stderr** (ροή 2), όχι στη **stdout** (ροή 1). Το `2>/dev/null` τα απορρίπτει. Εναλλακτικά, το `2>&1 | grep -v "Permission denied"` τα συγχωνεύει με τη stdout και τα φιλτράρει.

---

## Μέρος Β: Επεξεργασία Κειμένου (Επίπεδα 7–12)

### Επίπεδο 7 → 8: Αναζήτηση συμβολοσειράς
**Εντολές:** `grep` · `find -exec`

```bash
bandit7@bandit:~$ grep millionth data.txt
millionth    cvX2JJa4CFALtqS87jk27qwqGhBM9plV
```

Το αρχικό write-up χρησιμοποιεί `find / -name "data.txt" -exec grep -H 'millionth' {} \; 2>/dev/null`. Η `-exec` εκτελεί μια εντολή για κάθε αποτέλεσμα: το `{}` αντικαθίσταται από το όνομα του αρχείου και το `\;` κλείνει την εντολή.

### Επίπεδο 8 → 9: Μοναδική γραμμή
**Εντολές:** `sort` · `uniq` · διασωλήνωση

```bash
bandit8@bandit:~$ sort data.txt | uniq -u
UsvVyFSfZZWbi6wgC7dAFyFuR6jQQUhR
```

```
data.txt ──> sort ──> uniq -c ──> grep "1 " ──> κωδικός
             │        │           │
             │        │           └─ διατήρηση των γραμμών με μετρητή 1
             │        └─ καταμέτρηση εμφανίσεων κάθε γραμμής
             └─ ομαδοποίηση των ίδιων γραμμών σε διαδοχικές θέσεις
```

Η `uniq` εντοπίζει μόνο **διαδοχικές** όμοιες γραμμές, γι' αυτό προηγείται η `sort`.

### Επίπεδο 9 → 10: Αναγνώσιμες συμβολοσειρές από δυαδικό αρχείο
**Εντολές:** `strings` · `grep ^`

```bash
bandit9@bandit:~$ strings data.txt | grep "^=="
========== password
========== isa
========== truKLdjsbJ5g7yyJ2X2R0o3a5HQJFuLk
```

Η `strings` εξάγει μόνο τις ακολουθίες εκτυπώσιμων χαρακτήρων. Στις κανονικές εκφράσεις το `^` είναι άγκυρα για την αρχή της γραμμής και το `$` για το τέλος.

> 🛡️ **Ασφάλεια:** η `strings` είναι πρώτο βήμα στην ανάλυση άγνωστων εκτελέσιμων και κακόβουλου λογισμικού, επειδή URLs, IPs και κωδικοί είναι συχνά άμεσα ορατοί.

### Επίπεδο 10 → 11: Αποκωδικοποίηση Base64
**Εντολές:** `base64 -d`

```bash
bandit10@bandit:~$ base64 -d data.txt
The password is IFukwKGsFW8MOq3IRFqrxE1hxTNEbUPR
```

| Έννοια | Λειτουργία |
| :--- | :--- |
| **Κωδικοποίηση** (Base64, hex) | Αλλάζει τη μορφή για μεταφορά. Χωρίς κλειδί, αντιστρέφεται από οποιονδήποτε. |
| **Κρυπτογράφηση** (AES, RSA) | Αποκρύπτει το περιεχόμενο. Χωρίς κλειδί δεν αντιστρέφεται. |
| **Κατακερματισμός** (MD5, SHA-256) | Μονόδρομο αποτύπωμα σταθερού μήκους. Δεν αντιστρέφεται. |

**Αναγνώριση Base64:** μόνο οι χαρακτήρες `A–Z a–z 0–9 + /`, μήκος πολλαπλάσιο του 4, συχνά καταλήγει σε `=` ή `==` (padding).

### Επίπεδο 11 → 12: ROT13
**Εντολές:** `tr` · κρυπτοσύστημα του Καίσαρα

```bash
bandit11@bandit:~$ cat data.txt | tr 'A-Za-z' 'N-ZA-Mn-za-m'
The password is 5Te8Y4drgCRfCx8ugdwuEX8KFC6k2EUu
```

```
πριν:  A B C D E F G H I J K L M N O P Q R S T U V W X Y Z
       │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │
μετά:  N O P Q R S T U V W X Y Z A B C D E F G H I J K L M
```

Επειδή το αλφάβητο έχει 26 γράμματα, η διπλή εφαρμογή του ROT13 επαναφέρει το αρχικό κείμενο. Η ίδια εντολή κωδικοποιεί και αποκωδικοποιεί.

### Επίπεδο 12 → 13: Διαδοχική αποσυμπίεση
**Εντολές:** `xxd -r` · `gzip` · `bzip2` · `tar` · `/tmp`

**Στόχος:** το `data.txt` είναι hexdump ενός αρχείου που έχει συμπιεστεί επανειλημμένα.

```bash
# Ο προσωπικός κατάλογος δεν είναι εγγράψιμος, δουλεύουμε στο /tmp
bandit12@bandit:~$ mkdir /tmp/ax && cp data.txt /tmp/ax && cd /tmp/ax
bandit12@bandit:/tmp/ax$ xxd -r data.txt data.out
bandit12@bandit:/tmp/ax$ file data.out
data.out: gzip compressed data, was "data2.bin", ...
bandit12@bandit:/tmp/ax$ mv data.out data.gz && gzip -d data.gz
bandit12@bandit:/tmp/ax$ file data
data: bzip2 compressed data, block size = 900k
bandit12@bandit:/tmp/ax$ bzip2 -d data
...                                    # επανάληψη: file → αποσυμπίεση
bandit12@bandit:/tmp/ax$ cat data8
The password is 8ZjyCRiBWFYkneahHwxCv3wb2a1ORpYL
```

| Αποτέλεσμα της `file` | Εντολή αποσυμπίεσης |
| :--- | :--- |
| `gzip compressed data` | `mv x x.gz && gzip -d x.gz` (η gzip απαιτεί κατάληξη `.gz`) |
| `bzip2 compressed data` | `bzip2 -d x` (παράγει `x.out` αν δεν αναγνωρίσει κατάληξη) |
| `POSIX tar archive` | `tar -xf x` |
| `ASCII text` | `cat x`: τέλος της διαδικασίας |

```
hexdump ─xxd -r─> gzip ─> bzip2 ─> gzip ─> tar ─> tar ─> bzip2 ─> tar ─> gzip ─> ASCII
```

---

## Μέρος Γ: Δίκτυο και Κλειδιά (Επίπεδα 13–20)

### Επίπεδο 13 → 14: Σύνδεση με ιδιωτικό κλειδί
**Εντολές:** `ssh -i` · `scp` · `chmod 400`

```bash
# Στον τοπικό υπολογιστή (Linux, macOS ή PowerShell)
$ scp -P 2220 bandit13@bandit.labs.overthewire.org:sshkey.private .
$ chmod 400 sshkey.private
$ ssh -i sshkey.private bandit14@bandit.labs.overthewire.org -p 2220
bandit14@bandit:~$
```

Ο εξυπηρετητής έχει το δημόσιο κλειδί του `bandit14` στο `~/.ssh/authorized_keys`. Όποιος κατέχει το αντίστοιχο ιδιωτικό κλειδί συνδέεται χωρίς κωδικό.

> ⚠️ **Συνήθη λάθη:** (α) η `scp` θέλει **κεφαλαίο** `-P` για τη θύρα, ενώ η `ssh` **πεζό** `-p`. (β) Χωρίς `chmod 400` ο πελάτης απορρίπτει το κλειδί με `UNPROTECTED PRIVATE KEY FILE`.

### Επίπεδο 14 → 15: Αποστολή δεδομένων σε θύρα TCP
**Εντολές:** `nc` · TCP · localhost

```bash
bandit14@bandit:~$ cat /etc/bandit_pass/bandit14 | nc localhost 30000
Correct!
BfMYroe26WYalil77FoDi9qh59eK5xNr
```

Το `nc` (netcat) ανοίγει μια απλή σύνδεση TCP: ό,τι μπει στη stdin του στέλνεται στον εξυπηρετητή και η απάντηση εμφανίζεται στη stdout.

### Επίπεδο 15 → 16: Επικοινωνία μέσω SSL/TLS
**Εντολές:** `openssl s_client`

```bash
bandit15@bandit:~$ cat /etc/bandit_pass/bandit15 | openssl s_client -connect localhost:30001 -quiet
verify error:num=18:self signed certificate
Correct!
cluFn7wTiGryunymYOu4RcffSxQluehd
```

Μια θύρα TLS απαιτεί πρώτα χειραψία (handshake), οπότε το απλό `nc` αποτυγχάνει. Το `openssl s_client` ολοκληρώνει τη χειραψία και μετά λειτουργεί όπως το `nc`. Το `self signed certificate` είναι αναμενόμενο σε εργαστηριακό περιβάλλον.

### Επίπεδο 16 → 17: Σάρωση θυρών
**Εντολές:** `/dev/tcp` · `openssl` · `nmap`

**Στόχος:** μία θύρα στο εύρος 31000–32000 μιλά SSL και επιστρέφει τα διαπιστευτήρια.

```bash
bandit16@bandit:~$ for i in {31000..32000}; do
>   (echo > /dev/tcp/localhost/$i) >& /dev/null && echo "Port $i open"
> done
Port 31518 open
Port 31790 open
bandit16@bandit:~$ cat /etc/bandit_pass/bandit16 | openssl s_client -connect localhost:31790 -quiet
Correct!
-----BEGIN RSA PRIVATE KEY-----
...
-----END RSA PRIVATE KEY-----
```

```
31000 ───────────────────── 31518 ─────────── 31790 ──────── 32000
                              │                 │
                              echo              SSL ok ──> ιδιωτικό κλειδί RSA
```

Το bash υποστηρίζει την ειδική διαδρομή `/dev/tcp/HOST/PORT`: η εγγραφή σε αυτήν επιχειρεί σύνδεση TCP. Αποθηκεύστε το κλειδί (π.χ. `sshkey`), εφαρμόστε `chmod 400` και συνδεθείτε ως `bandit17`. Με εξειδικευμένο εργαλείο: `nmap -sV -p 31000-32000 localhost`.

### Επίπεδο 17 → 18: Σύγκριση αρχείων
**Εντολές:** `diff`

```bash
bandit17@bandit:~$ diff passwords.old passwords.new
42c42
< 6vcSC74ROI95NqkKaeEC2ABVMDX9TyUr
---
> kfBf3eYk5BPBRzwjqutbbfE887SVc5Yd
```

Το `42c42` σημαίνει ότι η γραμμή 42 άλλαξε (change). Το `<` αφορά το πρώτο (παλαιό) αρχείο και το `>` το δεύτερο (νέο).

### Επίπεδο 18 → 19: Παράκαμψη τροποποιημένου `.bashrc`
**Εντολές:** `ssh <εντολή>`

```bash
$ ssh bandit18@bandit.labs.overthewire.org -p 2220
Byebye !
Connection to bandit.labs.overthewire.org closed.
$ ssh bandit18@bandit.labs.overthewire.org -p 2220 "cat readme"
IueksS7Ubh8G3DCwVzrTd8rAVOwq3M5x
```

Το `.bashrc` φορτώνεται μόνο σε **διαδραστικό** κέλυφος. Όταν η εντολή δίνεται ως όρισμα της `ssh`, εκτελείται χωρίς διαδραστικό κέλυφος και η τροποποίηση δεν ενεργοποιείται. Στο PuTTY: *Connection → SSH → Remote command*.

### Επίπεδο 19 → 20: Εκτελέσιμο SUID
**Εντολές:** setuid · κλιμάκωση δικαιωμάτων

```bash
bandit19@bandit:~$ ls -l bandit20-do
-rwsr-x--- 1 bandit20 bandit19 7296 Oct 16 14:00 bandit20-do
bandit19@bandit:~$ ./bandit20-do cat /etc/bandit_pass/bandit20
GbKksEFF4yrVs6il55v6gwY5aVje5f0j
```

Το `s` στη θέση του `x` του ιδιοκτήτη (`-rwsr-x---`) είναι το bit **SUID**: το αρχείο εκτελείται με τα δικαιώματα του **ιδιοκτήτη του** (`bandit20`), όχι του χρήστη που το εκκινεί.

> 🛡️ **Ασφάλεια:** μετά την απόκτηση πρόσβασης, το `find / -perm -4000 2>/dev/null` είναι από τα πρώτα βήματα αναζήτησης κλιμάκωσης δικαιωμάτων. Θα το συναντήσετε ξανά στα [DC-CTF](../04.%20DC-CTF/README.md), [Raven2-CTF](../05.%20Raven2-CTF/README.md) και [Mr Robot](../06.%20Mr%20Robot/README.md).

### Επίπεδο 20 → 21: Ακρόαση θύρας με netcat
**Εντολές:** `nc -l` · ακροατής · setuid

**Στόχος:** το SUID `suconnect` συνδέεται σε θύρα του localhost, διαβάζει μία γραμμή και, αν ταιριάζει με τον κωδικό του `bandit20`, στέλνει τον κωδικό του `bandit21`.

```bash
# Τερματικό 1: ο ακροατής (listener)
bandit20@bandit:~$ nc -lp 31337 < /etc/bandit_pass/bandit20
gE269g2h3mw3pwgrj0Ha9Uoqen1c9DGr

# Τερματικό 2: ο πελάτης
bandit20@bandit:~$ ./suconnect 31337
Read: GbKksEFF4yrVs6il55v6gwY5aVje5f0j
Password matches, sending next password
```

```
  Τερματικό 1                                   Τερματικό 2
┌──────────────────┐   ① σύνδεση στη 31337   ┌────────────────────┐
│  nc -lp 31337    │ <────────────────────── │  ./suconnect 31337 │
│  (ακροατής)      │ ② κωδικός bandit20 ───> │  (εκτελείται ως    │
│                  │ <─── ③ κωδικός bandit21 │   bandit21)        │
└──────────────────┘                         └────────────────────┘
```

Με ένα μόνο τερματικό, τρέξτε τον ακροατή στο παρασκήνιο: `nc -lp 31337 < /etc/bandit_pass/bandit20 &` και μετά `./suconnect 31337`.

---

## Μέρος Δ: Αυτοματοποίηση (Επίπεδα 21–24)

### Επίπεδο 21 → 22: Εργασίες cron (Ι): ανάγνωση ρυθμίσεων
**Εντολές:** `cron` · `/etc/cron.d`

```bash
bandit21@bandit:~$ cat /etc/cron.d/cronjob_bandit22
* * * * * bandit22 /usr/bin/cronjob_bandit22.sh &> /dev/null
bandit21@bandit:~$ cat /usr/bin/cronjob_bandit22.sh
#!/bin/bash
chmod 644 /tmp/t7O6lds9S0RqQh9aMcz6ShpAoZKF7fgv
cat /etc/bandit_pass/bandit22 > /tmp/t7O6lds9S0RqQh9aMcz6ShpAoZKF7fgv
bandit21@bandit:~$ cat /tmp/t7O6lds9S0RqQh9aMcz6ShpAoZKF7fgv
Yk7owGAcWjwMVRwrTesJEwB7WVOiILLI
```

```
*    *    *    *    *    bandit22   /usr/bin/cronjob_bandit22.sh
│    │    │    │    │    │          │
│    │    │    │    │    │          └─ η εντολή που εκτελείται
│    │    │    │    │    └─ ο χρήστης υπό τον οποίο εκτελείται
│    │    │    │    └─ ημέρα εβδομάδας (0–7)
│    │    │    └─ μήνας (1–12)
│    │    └─ ημέρα μήνα (1–31)
│    └─ ώρα (0–23)
└─ λεπτό (0–59)          * = «κάθε»  →  εδώ: κάθε λεπτό
```

### Επίπεδο 22 → 23: Εργασίες cron (ΙΙ): ανάλυση σεναρίου
**Εντολές:** σενάριο bash · `md5sum` · `whoami`

```bash
bandit22@bandit:~$ cat /usr/bin/cronjob_bandit23.sh
#!/bin/bash
myname=$(whoami)
mytarget=$(echo I am user $myname | md5sum | cut -d ' ' -f 1)
cat /etc/bandit_pass/$myname > /tmp/$mytarget
bandit22@bandit:~$ echo "I am user bandit23" | md5sum
8ca319486bfbbc3663ea0fbe81326349  -
bandit22@bandit:~$ cat /tmp/8ca319486bfbbc3663ea0fbe81326349
jc1udXuA1tiHqjIsL8yaapX5XIAI6i0n
```

Το σενάριο εκτελείται από το cron ως `bandit23`, άρα `$myname = bandit23`. Αρκεί να υπολογίσετε το ίδιο MD5.

> ⚠️ **Σύνηθες λάθος:** η εκτέλεση του σεναρίου ως `bandit22`. Τότε η `whoami` επιστρέφει `bandit22` και το αποτύπωμα είναι λάθος.

### Επίπεδο 23 → 24: Εργασίες cron (ΙΙΙ): συγγραφή σεναρίου
**Εντολές:** σενάριο κελύφους · `/var/spool` · δικαιώματα καταλόγου

**Στόχος:** η εργασία cron του `bandit24` εκτελεί και διαγράφει κάθε σενάριο στο `/var/spool/bandit24`.

```bash
bandit23@bandit:~$ mkdir /tmp/alex1234 && cd /tmp/alex1234
bandit23@bandit:/tmp/alex1234$ cat > script.sh <<'EOF'
#!/bin/sh
cat /etc/bandit_pass/bandit24 >> /tmp/alex1234/bandit24pass
EOF
bandit23@bandit:/tmp/alex1234$ chmod 777 script.sh /tmp/alex1234
bandit23@bandit:/tmp/alex1234$ cp script.sh /var/spool/bandit24/
# Αναμονή έως ένα λεπτό
bandit23@bandit:/tmp/alex1234$ cat bandit24pass
UoMYTrfrBFHyQXmg6gzctqAwOmw1IohZ
```

```
cron ──(κάθε λεπτό)──> cronjob_bandit24.sh ──(ως bandit24)──> /var/spool/bandit24/script.sh
                                                                   │
                                                                   ▼
                                                   /tmp/alex1234/bandit24pass
```

Το `chmod 777` στον κατάλογο χρειάζεται επειδή το σενάριο εκτελείται ως `bandit24`: χωρίς δικαίωμα εγγραφής, το αρχείο εξόδου δεν δημιουργείται.

> 🛡️ **Ασφάλεια:** εγγράψιμοι κατάλογοι ή σενάρια που εκτελεί ο `root` μέσω cron αποτελούν συνήθη οδό κλιμάκωσης δικαιωμάτων σε πραγματικά συστήματα.

### Επίπεδο 24 → 25: Εξαντλητική αναζήτηση τετραψήφιου PIN
**Εντολές:** brute force · `seq` · `nc`

```bash
bandit24@bandit:~$ PASS=$(cat /etc/bandit_pass/bandit24)
bandit24@bandit:~$ for pin in $(seq -w 0000 9999); do echo "$PASS $pin"; done \
>     | nc localhost 30002 | grep -v Wrong
Correct!
The password of user bandit25 is uNG9O58gUE7snukf3bvZ0rxhtnjzSGzG
```

Η `seq -w 0000 9999` παράγει όλα τα PIN με προπορευόμενα μηδενικά. Όλες οι δοκιμές στέλνονται μέσω **μίας** σύνδεσης, που είναι πολύ ταχύτερο από 10.000 ξεχωριστές. Αν η υπηρεσία κλείσει τη σύνδεση πρόωρα, χωρίστε το εύρος (π.χ. `0000–4999` και `5000–9999`).

> 🛡️ **Ασφάλεια:** 10.000 συνδυασμοί εξαντλούνται σε δευτερόλεπτα. Γι' αυτό τα πραγματικά συστήματα εφαρμόζουν **rate limiting** ή κλείδωμα λογαριασμού.

---

## Μέρος Ε: Διαφυγή από Κελύφη και Git (Επίπεδα 25–33)

### Επίπεδο 25 → 26: Διαφυγή μέσω `more`
**Εντολές:** `/etc/passwd` · `more` · `vi`

```bash
bandit25@bandit:~$ grep bandit26 /etc/passwd
bandit26:x:11026:11026:bandit level 26:/home/bandit26:/usr/bin/showtext
bandit25@bandit:~$ cat /usr/bin/showtext
#!/bin/sh
export TERM=linux
more ~/text.txt
exit 0
bandit25@bandit:~$ ssh -i bandit26.sshkey bandit26@localhost
# Σμικρύνετε το παράθυρο (περίπου 6 γραμμές) και συνδεθείτε ξανά:
--More--(83%)
# Πατήστε v για τον vi. Εντός του vi:
:e /etc/bandit_pass/bandit26
5czgV9L3Xx8JPOyRbXh6lQbmIOWvPT6Z
```

```
ssh bandit26 ──> showtext ──> more ──[ v ]──> vi ──[ :set shell / :! ]──> bash
                              │
                              └─ σταματά στο --More-- μόνο αν το κείμενο δεν χωρά
```

Κάθε πρόγραμμα που μπορεί να εκκινήσει άλλο πρόγραμμα είναι πιθανή οδός διαφυγής. Με μικρό παράθυρο η `more` σταματά στο `--More--`, όπου το `v` ανοίγει τον `vi`.

### Επίπεδο 26 → 27: Εντολές κελύφους από τον `vi`
**Εντολές:** `:set shell` · `:!` · setuid

```
:set shell=/bin/bash
:!ls -la
-rwsr-x---  1 bandit27 bandit26 7296 Oct 16 14:00 bandit27-do
:!./bandit27-do cat /etc/bandit_pass/bandit27
3ba3118a22e93127a4ed485be72ef5ea
```

Ο `vi` εκτελεί τις εντολές `:!` μέσω του κελύφους του χρήστη (εδώ το `showtext`). Με `:set shell=/bin/bash` ορίζεται το bash. Στη συνέχεια, όπως στο επίπεδο 19, το SUID `bandit27-do` διαβάζει τον κωδικό. Το `:shell` ανοίγει πλήρες διαδραστικό κέλυφος.

> 🛡️ **Ασφάλεια:** τέτοιες τεχνικές διαφυγής καταγράφονται συστηματικά στο [GTFOBins](https://gtfobins.github.io/), βασική πηγή για ελέγχους διείσδυσης.

### Επίπεδο 27 → 28: Git (Ι): κλωνοποίηση
**Εντολές:** `git clone`

```bash
bandit27@bandit:~$ mkdir /tmp/repo123 && cd /tmp/repo123
bandit27@bandit:/tmp/repo123$ git clone ssh://bandit27-git@localhost/home/bandit27-git/repo
bandit27@bandit:/tmp/repo123$ cat repo/README
The password to the next level is: 0ef186ac70e04ea33b4c1853d2526fa2
```

Ο κωδικός του `bandit27-git` είναι ίδιος με του `bandit27`. Τα επόμενα τέσσερα επίπεδα ακολουθούν την ίδια διαδικασία (`bandit28-git`, `bandit29-git`, …).

### Επίπεδο 28 → 29: Git (ΙΙ): ανάκτηση από το ιστορικό
**Εντολές:** `git log` · `git checkout`

```bash
bandit28@bandit:/tmp/repo1337/repo$ cat README.md
- password: xxxxxxxxxx
bandit28@bandit:/tmp/repo1337/repo$ git log --oneline
073c27c fix info leak
186a103 add missing data
b67405d initial commit of README.md
bandit28@bandit:/tmp/repo1337/repo$ git checkout 186a103
bandit28@bandit:/tmp/repo1337/repo$ cat README.md
- password: bbc96594b4e001778eee9975372716b2
```

```
b67405d ─────────> 186a103 ─────────────> 073c27c  (HEAD)
initial commit     add missing data       fix info leak
                   ▲ περιέχει τον κωδικό  ▲ τον αφαίρεσε από το README
```

Εναλλακτικά: `git log -p` ή `git show 073c27c`.

> 🛡️ **Ασφάλεια:** κωδικοί και κλειδιά API που μπήκαν σε commit παραμένουν στο ιστορικό ακόμη κι αν αφαιρεθούν αργότερα. Εργαλεία όπως τα **trufflehog** και **gitleaks** εντοπίζουν τέτοιες διαρροές.

### Επίπεδο 29 → 30: Git (ΙΙΙ): κλάδοι
**Εντολές:** `git branch -r` · `git checkout`

```bash
bandit29@bandit:/tmp/plop123/repo$ cat README.md
- password: <no passwords in production!>
bandit29@bandit:/tmp/plop123/repo$ git branch -r
  origin/dev
  origin/master
  origin/sploits-dev
bandit29@bandit:/tmp/plop123/repo$ git checkout dev && cat README.md
- password: 5b90576bedb2cc04c86a9e924ce42faf
```

Η φράση «no passwords in production» είναι ένδειξη: ο `master` είναι η παραγωγή, οπότε ψάχνουμε σε άλλους κλάδους.

### Επίπεδο 30 → 31: Git (IV): ετικέτες
**Εντολές:** `git tag` · `git show`

```bash
bandit30@bandit:/tmp/plop1234/repo$ git tag
secret
bandit30@bandit:/tmp/plop1234/repo$ git show secret
47e603bb428404d265f59c42920d81e5
```

Ούτε το ιστορικό ούτε οι κλάδοι περιέχουν τον κωδικό. Οι **ετικέτες (tags)** είναι ονομαστικές αναφορές σε αντικείμενα του αποθετηρίου, εδώ σε ένα blob.

### Επίπεδο 31 → 32: Git (V): αποστολή αλλαγών
**Εντολές:** `git add -f` · `.gitignore` · `git push` · hooks

```bash
bandit31@bandit:/tmp/plop12345/repo$ echo "May I come in?" > key.txt
bandit31@bandit:/tmp/plop12345/repo$ git add -f key.txt
bandit31@bandit:/tmp/plop12345/repo$ git commit -m "key.txt"
bandit31@bandit:/tmp/plop12345/repo$ git push origin master
remote: Well done! Here is the password for the next level:
remote: 56a9bf19c63d650ce78e6ec0354ee45e
 ! [remote rejected] master -> master (pre-receive hook declined)
```

Το `.gitignore` περιέχει τον κανόνα `*.txt`, οπότε χρειάζεται `add -f`. Στον εξυπηρετητή, ένα **pre-receive hook** εμφανίζει τον κωδικό και μετά απορρίπτει την αποστολή. Η απόρριψη είναι αναμενόμενη.

### Επίπεδο 32 → 33: Διαφυγή από κέλυφος κεφαλαίων
**Εντολές:** `$0`

```
WELCOME TO THE UPPERCASE SHELL
>> ls
sh: 1: LS: not found
>> $0
$ whoami
bandit33
$ cat /etc/bandit_pass/bandit33
c9c3199ddf4121b10cf581a98d51caee
```

Το κέλυφος μετατρέπει κάθε είσοδο σε κεφαλαία, και το Linux διακρίνει πεζά από κεφαλαία (`LS` ≠ `ls`). Η μεταβλητή `$0` όμως δεν έχει γράμματα: περιέχει το όνομα του τρέχοντος κελύφους (`sh`), οπότε η εκτέλεσή της ανοίγει κανονικό κέλυφος. Επειδή το πρόγραμμα είναι SUID, το νέο κέλυφος τρέχει ως `bandit33`.

### Επίπεδο 33: Ολοκλήρωση

```bash
bandit33@bandit:~$ cat README.txt
Congratulations on solving the last level of this game!
```

Ως επόμενο βήμα προτείνονται τα wargames **Leviathan** (σύντομο, με έμφαση στα εκτελέσιμα) και **Natas** (ασφάλεια διαδικτυακών εφαρμογών).

---

## Παράρτημα Α: Σύνοψη Εντολών ανά Επίπεδο

| Επίπεδο | Κύρια εντολή | Έννοια |
| :--- | :--- | :--- |
| 0–3 | `ssh`, `ls -la`, `cat ./-` | Σύνδεση, ειδικά ονόματα, κρυφά αρχεία |
| 4–6 | `file`, `find` | Αναζήτηση βάσει τύπου, μεγέθους, ιδιοκτήτη |
| 7–9 | `grep`, `sort \| uniq`, `strings` | Φιλτράρισμα κειμένου με διασωληνώσεις |
| 10–12 | `base64 -d`, `tr`, `xxd -r` | Κωδικοποιήσεις και συμπίεση |
| 13 | `scp`, `ssh -i` | Αυθεντικοποίηση με κλειδί |
| 14–16 | `nc`, `openssl s_client`, `/dev/tcp` | TCP, TLS, σάρωση θυρών |
| 17–18 | `diff`, `ssh host "cmd"` | Σύγκριση αρχείων, μη διαδραστικό κέλυφος |
| 19–20 | `./bandit20-do`, `nc -l` | SUID, ακρόαση θύρας |
| 21–23 | `cat /etc/cron.d/*`, `md5sum` | Ανάλυση και εκμετάλλευση εργασιών cron |
| 24 | `seq -w` + `nc` | Εξαντλητική αναζήτηση |
| 25–26 | `more` → `vi` → `:!` | Διαφυγή από περιορισμένο κέλυφος |
| 27–31 | `git log / branch / tag / push` | Ευαίσθητα δεδομένα σε αποθετήρια git |
| 32 | `$0` | Παράκαμψη φίλτρου εισόδου |

## Παράρτημα Β: Γλωσσάρι

| Όρος | Σημασία |
| :--- | :--- |
| **SSH** | Πρωτόκολλο κρυπτογραφημένης απομακρυσμένης πρόσβασης σε κέλυφος (θύρα 22, εδώ 2220) |
| **PuTTY / PuTTYgen** | Γραφικός πελάτης SSH για Windows / εργαλείο διαχείρισης και μετατροπής κλειδιών |
| **stdin / stdout / stderr** | Τυπική είσοδος / έξοδος / έξοδος σφαλμάτων (ροές 0 / 1 / 2) |
| **Διασωλήνωση (`\|`)** | Σύνδεση της stdout μιας εντολής με τη stdin της επόμενης |
| **Dotfile** | Αρχείο που αρχίζει με `.`, δεν εμφανίζεται με την απλή `ls` |
| **Base64** | Κωδικοποίηση δυαδικών δεδομένων σε κείμενο (δεν είναι κρυπτογράφηση) |
| **ROT13** | Μετατόπιση γραμμάτων κατά 13 θέσεις, παραλλαγή του κρυπτοσυστήματος του Καίσαρα |
| **Hexdump** | Αναπαράσταση bytes σε δεκαεξαδική μορφή (`xxd`) |
| **TLS / SSL** | Πρωτόκολλα κρυπτογράφησης μιας σύνδεσης TCP |
| **Σάρωση θυρών** | Έλεγχος των θυρών ενός συστήματος που δέχονται συνδέσεις |
| **SUID** | Εκτέλεση αρχείου με τα δικαιώματα του ιδιοκτήτη του |
| **Cron** | Χρονοπρογραμματιστής περιοδικών εργασιών |
| **Εξαντλητική αναζήτηση** | Δοκιμή όλων των πιθανών τιμών (brute force) |
| **Διαφυγή από κέλυφος** | Απόκτηση πλήρους κελύφους μέσα από περιορισμένο πρόγραμμα (shell escape) |
| **Git hook** | Σενάριο που εκτελείται αυτόματα σε συμβάντα του git (π.χ. push) |

## Παράρτημα Γ: Λίστα Ελέγχου Δεξιοτήτων

- [ ] Συνδέομαι με `ssh` ή PuTTY σε μη προεπιλεγμένη θύρα, με κωδικό ή ιδιωτικό κλειδί
- [ ] Μετατρέπω κλειδί OpenSSH σε μορφή `.ppk` με το PuTTYgen
- [ ] Διαχειρίζομαι αρχεία με ιδιαίτερα ονόματα (`-`, κενά, αρχική τελεία)
- [ ] Συντάσσω εντολή `find` με φίλτρα μεγέθους, ιδιοκτήτη και δικαιωμάτων
- [ ] Εξηγώ τη διαφορά μεταξύ `2>/dev/null` και `2>&1`
- [ ] Συνθέτω διασωληνώσεις με `sort`, `uniq`, `grep`, `tr`
- [ ] Διακρίνω την κωδικοποίηση, την κρυπτογράφηση και τον κατακερματισμό
- [ ] Επικοινωνώ με δικτυακή υπηρεσία μέσω `nc` και `openssl s_client`
- [ ] Εξηγώ τη σημασία του `s` στο `-rwsr-x---` και τους κινδύνους του
- [ ] Ερμηνεύω μια γραμμή crontab και προσδιορίζω τον χρήστη υπό τον οποίο εκτελείται
- [ ] Αναζητώ ευαίσθητα δεδομένα στο ιστορικό, τους κλάδους και τις ετικέτες ενός αποθετηρίου git
- [ ] Διαφεύγω από περιορισμένο κέλυφος μέσω `vi` ή `$0`

### Σύνοψη
Το Bandit δεν απαιτεί εργαλεία επίθεσης: κάθε επίπεδο λύνεται με τις βασικές εντολές του Linux. Οι ίδιες τεχνικές (κρυφά αρχεία, `find`, SUID, cron, διαφυγή από κέλυφος, διαρροές στο git) εμφανίζονται ξανά στα CTF μηχανήματα που ακολουθούν, ξεκινώντας από το [03. Raven-CTF](../03.%20Raven-CTF/README.md).

**Πηγή:** A. Cheron, «[OTW] Write-up for the Bandit Wargame», [axcheron.github.io](https://axcheron.github.io/writeups/otw/bandit/). Ελληνική απόδοση και επεξηγήσεις για το μάθημα CSEC 101.
