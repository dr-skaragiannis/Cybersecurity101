# Cybersecurity 101 / Εισαγωγή στην Κυβερνοασφάλεια

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)
[![Status: Active](https://img.shields.io/badge/Status-Active-success.svg)]()

> **Languages / Γλώσσες:** [English](#english) | [Ελληνικά](#ελληνικά)

---

<a name="english"></a>
## English

### Overview
**Cybersecurity 101** is an introductory, hands-on educational repository and curriculum designed for students, developers, IT professionals, and security enthusiasts. It covers foundational cybersecurity principles, hands-on defensive and offensive techniques, standard security frameworks, and interactive lab environments.

---

### Table of Contents
1. [Curriculum Modules](#curriculum-modules)
2. [Lab Environments & Hands-On Practice](#lab-environments--hands-on-practice)
3. [Recommended Tooling](#recommended-tooling)
4. [Getting Started](#getting-started)
5. [Contributing](#contributing)
6. [License](#license)

---

### Curriculum Modules

| Module | Topic | Core Concepts |
| :--- | :--- | :--- |
| **01** | **Foundations of Information Security** | CIA Triad (Confidentiality, Integrity, Availability), AAA, Threat Modeling (STRIDE, DREAD), Attack Surfaces |
| **02** | **Network Security & Protocols** | OSI & TCP/IP stack security, DNS, DHCP, ARP Spoofing, Packet Analysis (Wireshark/tshark), Firewalls, IDS/IPS |
| **03** | **Cryptography Essentials** | Symmetric vs. Asymmetric encryption (AES, RSA, ECC), Hashing (SHA-256), Digital Signatures, PKI, Post-Quantum Cryptography basics |
| **04** | **System & Endpoint Security** | Linux & Windows Hardening, Access Control (RBAC, DAC, MAC), Privilege Escalation, Sysmon, eBPF telemetry |
| **05** | **Web Application Security** | OWASP Top 10 (SQLi, XSS, CSRF, SSRF, IDOR), HTTP Headers, Secure Cookies, API Security |
| **06** | **Identity & Access Management (IAM)** | Authentication factors (MFA, Passkeys), SSO, OAuth 2.0, OpenID Connect, Zero Trust Architecture |
| **07** | **SOC, SIEM & Incident Response** | Log aggregation, SIEM architecture (Wazuh, ELK), Detection engineering (Sigma, YARA, Suricata), NIST CSF, Incident Response Lifecycle |
| **08** | **Cyber Ranges & Gamified Learning** | Capture The Flag (CTF) challenges, Jeopardy vs. Attack-Defense formats, Hands-on scenario-based emulation |

---

### Lab Environments & Hands-On Practice
Hands-on labs are structured in individual directories under `/labs`:

- `labs/01-packet-analysis/`: PCAP analysis exercises using `tcpdump` and `Wireshark`.
- `labs/02-cryptography/`: Practical hashing, key generation, and message signing in Python.
- `labs/03-web-vulns/`: Vulnerability recreation scripts and containerized vulnerable apps.
- `labs/04-defensive-detection/`: Writing Suricata inspection rules and YARA signatures.

#### Prerequisites
- [Docker](https://docs.docker.com/get-docker/) & [Docker Compose](https://docs.docker.com/compose/)
- [Python 3.10+](https://www.python.org/)
- [Wireshark](https://www.wireshark.org/) / `tshark`

---

### Recommended Tooling

- **Analysis & Telemetry:** Wireshark, Zeek, Suricata, NetworkMiner
- **Security Testing & Auditing:** Nmap, Burp Suite, OWASP ZAP, Nikto
- **Defensive & Detection:** Wazuh, OpenSearch/Elasticsearch, YARA, Sigma
- **Sandboxes & Virtualization:** Docker, VirtualBox, UTM, Vagrant

---

### Getting Started

```bash
# Clone the repository
git clone [https://github.com/dr-skaragiannis/Cybersecurity101.git](https://github.com/dr-skaragiannis/Cybersecurity101.git)
cd Cybersecurity101

# Inspect lab resources
ls -la labs/

# Launch a lab environment (e.g., Web Security Lab)
cd labs/03-web-vulns
docker compose up -d
