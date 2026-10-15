<div align="center">

# Cybersecurity 101

**Introduction to Cybersecurity · Εισαγωγή στην Κυβερνοασφάλεια**

Hands-on, open-source lab guides for learning offensive and defensive security
in isolated environments.
Πρακτικοί, ανοικτοί εργαστηριακοί οδηγοί για εκμάθηση κυβερνοασφάλειας σε απομονωμένο περιβάλλον.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)
![Languages](https://img.shields.io/badge/languages-English%20%7C%20%CE%95%CE%BB%CE%BB%CE%B7%CE%BD%CE%B9%CE%BA%CE%AC-blue)

**Languages / Γλώσσες:** [English](#english) · [Ελληνικά](#ελληνικά)

</div>

---

> ⚠️ **Educational use only / Μόνο για εκπαιδευτική χρήση.**
> Every offensive technique in this repository must be practised **only** against
> machines you own or are explicitly authorised to test. See the
> [legal disclaimer](#legal-disclaimer--αποποίηση-ευθύνης) below.

---

## English

### Overview

**Cybersecurity 101** is a practical, open-source course of step-by-step lab
guides, penetration-testing walkthroughs and defensive-security tutorials. The
material is written as **console transcripts** — commands, output and explanation
side by side — rather than screenshots, so every step is reproducible and easy to
copy into your own terminal.

The guides form a single numbered learning path, from foundational Linux
command-line skills, through service enumeration and exploitation, to network
packet analysis and defensive deception (honeypots).

### Repository structure

```
Cybersecurity101/
├── README.md                        # this file — bilingual overview
├── CONTRIBUTING.md
├── LICENSE
├── guides/
│   ├── en/                          # English learning path (00 → 10)
│   │   ├── README.md                # English index
│   │   └── 00-*.md … 10-*.md
│   └── el/                          # Ελληνική διαδρομή (00 → 10)
│       ├── README.md                # Ελληνικό ευρετήριο
│       └── 00-*.md … 10-*.md
└── notebooks/
    └── 00-linux-fundamentals.ipynb  # extra hands-on Linux warm-up
```

### Where to start

1. **Set up your lab first.** Guide **00** walks you through importing a virtual
   machine and configuring an isolated network.
2. **Then follow the numbers.** Each guide builds on the previous one, 00 → 10.

📚 **Full indexes**

- **English:** [`guides/en/README.md`](guides/en/README.md)
- **Ελληνικά:** [`guides/el/README.md`](guides/el/README.md)

### Recommended lab setup

- **Attacker platform:** we recommend **Parrot OS Security** (import the ready-made VirtualBox OVA — no installation). Kali Linux works just as well if you already prefer it.
- **Hypervisor:** Oracle VM VirtualBox (guide 00 covers installation, OVA import and networking).
- **Traffic analysis:** `tcpdump`, `wireshark` / `tshark`.
- **Targets:** intentionally vulnerable VMs (for example the Raven 1 appliance used in guide 06).

> Start with guide **00** to build a safe, isolated lab network before running any
> of the offensive modules.

### What you will learn

- **Linux fundamentals** — shell, files, text processing, packages, permissions, processes, networking and Bash scripting.
- **Offensive security** — SSH auditing, anonymous/weak file-share access, and a full boot-to-root CTF walkthrough.
- **Web application security** — building and testing an authorised WordPress training lab.
- **Defensive security** — host firewalling with UFW, honeypots and threat telemetry.
- **Traffic analysis** — packet capture and Berkeley Packet Filter (BPF) filtering with `tcpdump`.

### Legal disclaimer / Αποποίηση ευθύνης

This repository is published **strictly for education and authorised security
testing**. Unauthorised access to systems you do not own, or for which you do not
have explicit written permission, is illegal in most jurisdictions. The authors
accept no liability for misuse of the information presented here.

### Contributing

Corrections, translations and new lab guides are welcome. Please read
[`CONTRIBUTING.md`](CONTRIBUTING.md) before opening a pull request.

### License

Released under the [MIT License](LICENSE).

---

## Ελληνικά

### Επισκόπηση

Το **Cybersecurity 101** είναι ένα πρακτικό, ανοικτό μάθημα με εργαστηριακούς
οδηγούς βήμα-προς-βήμα, τεχνικές penetration testing και αμυντική ασφάλεια. Το
υλικό είναι γραμμένο ως **καταγραφές τερματικού** (console transcripts) — εντολές,
έξοδος και επεξήγηση μαζί — αντί για screenshots, ώστε κάθε βήμα να είναι
αναπαραγώγιμο.

Οι οδηγοί σχηματίζουν μία αριθμημένη διαδρομή εκμάθησης: από τα βασικά του Linux,
στην αναγνώριση και εκμετάλλευση υπηρεσιών, έως την ανάλυση πακέτων και την
αμυντική εξαπάτηση (honeypots).

### Δομή αποθετηρίου

```
Cybersecurity101/
├── README.md                        # αυτό το αρχείο — δίγλωσση επισκόπηση
├── CONTRIBUTING.md
├── LICENSE
├── guides/
│   ├── en/                          # Αγγλική διαδρομή (00 → 10)
│   │   ├── README.md                # Αγγλικό ευρετήριο
│   │   └── 00-*.md … 10-*.md
│   └── el/                          # Ελληνική διαδρομή (00 → 10)
│       ├── README.md                # Ελληνικό ευρετήριο
│       └── 00-*.md … 10-*.md
└── notebooks/
    └── 00-linux-fundamentals.ipynb  # επιπλέον hands-on εισαγωγή στο Linux
```

### Από πού να ξεκινήσετε

1. **Στήστε πρώτα το εργαστήριό σας.** Ο οδηγός **00** σας καθοδηγεί στην εισαγωγή
   μιας εικονικής μηχανής και στη ρύθμιση απομονωμένου δικτύου.
2. **Συνεχίστε με τη σειρά.** Κάθε οδηγός βασίζεται στον προηγούμενο, 00 → 10.

📚 **Ευρετήρια**

- **Αγγλικά:** [`guides/en/README.md`](guides/en/README.md)
- **Ελληνικά:** [`guides/el/README.md`](guides/el/README.md)

### Συνιστώμενο εργαστηριακό περιβάλλον

- **Πλατφόρμα επίθεσης:** συνιστάται το **Parrot OS Security** (εισάγετε το έτοιμο OVA για VirtualBox — χωρίς εγκατάσταση). Το Kali Linux λειτουργεί εξίσου καλά, αν το προτιμάτε.
- **Hypervisor:** Oracle VM VirtualBox (ο οδηγός 00 καλύπτει εγκατάσταση, εισαγωγή OVA και δικτύωση).
- **Ανάλυση δικτύου:** `tcpdump`, `wireshark` / `tshark`.
- **Στόχοι:** σκόπιμα ευπαθείς εικονικές μηχανές (π.χ. το Raven 1 του οδηγού 06).

> Ξεκινήστε από τον οδηγό **00** για να στήσετε ένα ασφαλές, απομονωμένο δίκτυο
> πριν εκτελέσετε οποιοδήποτε offensive module.

### Τι θα μάθετε

- **Βασικά Linux** — κέλυφος, αρχεία, επεξεργασία κειμένου, πακέτα, δικαιώματα, διεργασίες, δίκτυα και Bash scripting.
- **Επιθετική ασφάλεια** — έλεγχος SSH, ανώνυμη/αδύναμη πρόσβαση σε κοινόχρηστα αρχεία και πλήρης οδηγός CTF (boot-to-root).
- **Ασφάλεια web εφαρμογών** — δημιουργία και δοκιμή εγκεκριμένου εργαστηρίου WordPress.
- **Αμυντική ασφάλεια** — firewall με UFW, honeypots και συλλογή τηλεμετρίας.
- **Ανάλυση κίνησης** — καταγραφή πακέτων και φίλτρα BPF με `tcpdump`.

### Αποποίηση ευθύνης

Το υλικό δημοσιεύεται **αυστηρά για εκπαιδευτικούς σκοπούς και εγκεκριμένες
δοκιμές ασφαλείας**. Η μη εξουσιοδοτημένη πρόσβαση σε συστήματα που δεν σας
ανήκουν — ή για τα οποία δεν έχετε ρητή γραπτή άδεια — είναι παράνομη στις
περισσότερες χώρες. Οι συγγραφείς δεν φέρουν καμία ευθύνη για κακή χρήση των
πληροφοριών.

### Συνεισφορά

Διορθώσεις, μεταφράσεις και νέοι οδηγοί είναι ευπρόσδεκτα. Διαβάστε το
[`CONTRIBUTING.md`](CONTRIBUTING.md) πριν υποβάλετε pull request.

### Άδεια Χρήσης

Διανέμεται υπό την [Άδεια MIT](LICENSE).
