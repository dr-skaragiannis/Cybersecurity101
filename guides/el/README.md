# Cybersecurity 101 — Οδηγοί στα Ελληνικά

Η ελληνική διαδρομή εκμάθησης, με σειρά. Ξεκινήστε από το **00** και προχωρήστε
παρακάτω: κάθε οδηγός βασίζεται στον προηγούμενο. Οι εντολές εμφανίζονται ως
καταγραφές τερματικού αντί για screenshots, ώστε κάθε βήμα να είναι εύκολα
αναπαραγώγιμο.

> 🔧 **Συνιστώμενη πλατφόρμα:** [Parrot OS](https://parrotsec.org/download/) —
> εισάγετε το έτοιμο OVA για VirtualBox (χωρίς εγκατάσταση). Το Kali Linux
> λειτουργεί επίσης. Δείτε τον οδηγό **00** για το πλήρες στήσιμο του εργαστηρίου.

> ⚠️ **Μόνο για εγκεκριμένα εργαστήρια.** Εφαρμόζετε αυτές τις τεχνικές μόνο σε
> μηχανήματα που σας ανήκουν ή για τα οποία έχετε ρητή άδεια.

## Διαδρομή εκμάθησης

| # | Οδηγός | Περιεχόμενο |
| :---: | :--- | :--- |
| 00 | [Virtualization & VirtualBox: Εισαγωγή OVA και Δίκτυα](00-virtualization-and-virtualbox.md) | Στήσιμο απομονωμένου lab, import OVA, NAT vs. Host-Only vs. NAT Network |
| 01 | [Βασικά Linux — Μέρος 1](01-linux-fundamentals-part-1.md) | Κέλυφος, αρχεία, κείμενο, πακέτα APT, δικαιώματα |
| 02 | [Βασικά Linux — Μέρος 2](02-linux-fundamentals-part-2.md) | Δίκτυα, διεργασίες, μεταβλητές περιβάλλοντος |
| 03 | [Βασικά Linux — Μέρος 3](03-linux-fundamentals-part-3.md) | Bash scripting, cron, υπηρεσίες Apache, SSH, FTP |
| 04 | [Penetration Testing SSH (Πόρτα 22)](04-ssh-penetration-testing.md) | Αναγνώριση, brute force, κλειδιά, tunnelling, hardening |
| 05 | [Ανώνυμες Κοινόχρηστες Υπηρεσίες: FTP, SMB & NFS](05-anonymous-file-shares.md) | Αναγνώριση και εκμετάλλευση ανώνυμης πρόσβασης |
| 06 | [Raven 1 — Web & Privilege Escalation](06-raven-1-web-privilege-escalation.md) | Πλήρης οδηγός Boot-to-Root CTF |
| 07 | [Διαχείριση Firewall με UFW](07-ufw-firewall.md) | Αμυντική προστασία, κανόνες UFW |
| 08 | [Εργαστήριο WordPress](08-wordpress-pentest-lab.md) | Στήσιμο CMS, έλεγχος plugins, WPScan, exploitation |
| 09 | [Low-Interaction Honeypot](09-honeypot-lab.md) | Υλοποίηση honeypot, καταγραφή επιθέσεων, τηλεμετρία |
| 10 | [tcpdump: Καταγραφή & Ανάλυση Πακέτων](10-tcpdump-packet-capture.md) | Καταγραφή κίνησης, φίλτρα BPF, ανάλυση πακέτων |

## Επιπλέον υλικό

- [`notebooks/00-linux-fundamentals.ipynb`](../../notebooks/00-linux-fundamentals.ipynb) — εισαγωγικό Jupyter notebook για το Linux.

## Άλλες γλώσσες

Αγγλικές εκδόσεις όλων των οδηγών: [`guides/en/`](../en/README.md).

## Άδεια Χρήσης

Διανέμεται υπό την [Άδεια MIT](../../LICENSE).
