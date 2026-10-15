# Cybersecurity 101 — English Guides

The English learning path, in order. Start at **00** and work down the list: each
guide builds on the previous one. Commands are shown as console transcripts rather
than screenshots, so every step is easy to reproduce.

> 🔧 **Recommended platform:** [Parrot OS](https://parrotsec.org/download/) — import
> the ready-made VirtualBox OVA (no installation). Kali Linux works too. See
> guide **00** for the full lab setup.

> ⚠️ **Authorised labs only.** Practise these techniques only against machines you
> own or are explicitly authorised to test.

## Learning path

| # | Guide | Topics |
| :---: | :--- | :--- |
| 00 | [Virtualization & VirtualBox: OVA Deployment and Networking](00-virtualization-and-virtualbox.md) | Isolated lab build, OVA import, NAT vs. Host-Only vs. NAT Network |
| 01 | [Linux Fundamentals — Part 1](01-linux-fundamentals-part-1.md) | Shell, files, text tools, APT packages, permissions |
| 02 | [Linux Fundamentals — Part 2](02-linux-fundamentals-part-2.md) | Networking, process management, environment variables |
| 03 | [Linux Fundamentals — Part 3](03-linux-fundamentals-part-3.md) | Bash scripting, cron automation, Apache, SSH, FTP |
| 04 | [SSH Penetration Testing (Port 22)](04-ssh-penetration-testing.md) | Recon, brute force, keys, tunnelling, persistence, hardening |
| 05 | [Anonymous File Shares: FTP, SMB & NFS](05-anonymous-file-shares.md) | Unauthenticated file-share enumeration and exploitation |
| 06 | [Raven 1 — Web & Privilege-Escalation Lab](06-raven-1-web-privilege-escalation.md) | Boot-to-root CTF walkthrough |
| 07 | [UFW Firewall Administration](07-ufw-firewall.md) | Host-based defence, UFW rule configuration |
| 08 | [Building an Authorised WordPress Training Lab](08-wordpress-pentest-lab.md) | CMS setup, plugin auditing, WPScan, exploitation |
| 09 | [Operating a Low-Interaction Honeypot](09-honeypot-lab.md) | Deception, attacker monitoring, telemetry |
| 10 | [tcpdump: Practical Packet Capture & Analysis](10-tcpdump-packet-capture.md) | Packet capture, BPF filters, traffic inspection |

## Extra material

- [`notebooks/00-linux-fundamentals.ipynb`](../../notebooks/00-linux-fundamentals.ipynb) — a hands-on Jupyter warm-up for Linux.

## Other languages

Greek versions of every guide: [`guides/el/`](../el/README.md).

## License

Released under the [MIT License](../../LICENSE).
