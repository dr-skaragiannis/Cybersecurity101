# VirtualBox Architecture, OVA Deployment, and Advanced Network Configuration

> ### ⚡ Recommended quick start — Parrot OS + OVA import
>
> **Use Parrot OS instead of Kali (and similar distributions).** Throughout this
> course the recommended attacker platform is **Parrot OS Security**. It ships
> with every tool used in the later guides (`nmap`, `hydra`, `wpscan`, `tcpdump`
> and more), and it is lighter and more beginner-friendly day to day. Kali Linux
> works just as well if you already prefer it.
>
> **The fastest way to begin is to import the ready-made OVA — no installation:**
>
> 1. Download the **Parrot OS VirtualBox image (`.ova`)** from
>    <https://parrotsec.org/download/> — choose the **Virtual Box** option.
> 2. In VirtualBox choose **File → Import Appliance…**, select the downloaded
>    `.ova` file and follow the wizard.
> 3. Before booting, make sure virtualization is enabled in your BIOS/UEFI
>    (Intel VT-x / AMD-V), then start the VM and log in with the default
>    credentials listed on the download page.
>
> The rest of this document explains how these appliances work, how to import
> them manually, and how to configure their networking correctly.

---

## 1. Introduction to Virtualization and Oracle VM VirtualBox

Virtualization puts a **hypervisor** between the hardware and the software it runs. The hypervisor emulates a whole hardware platform — CPU, memory, disk and network — and each such emulated platform is a **Virtual Machine (VM)**.

Hypervisors come in two tiers:

*   **Type-1 (bare-metal):** run directly on the hardware with no host operating system beneath them (VMware ESXi, KVM, Xen, Proxmox VE). They are fast and low-latency, which makes them the standard for data-centre infrastructure.
*   **Type-2 (hosted):** install as an ordinary program inside a conventional operating system (Oracle VM VirtualBox, VMware Workstation). The host manages the physical devices; the hypervisor mediates the guest.

```
+--------------------------------------------------------------+
|                     Host Hardware (CPU, RAM, NIC)            |
+--------------------------------------------------------------+
|                     Host Operating System (Linux / Windows)  |
+--------------------------------------------------------------+
|       Type-2 Hypervisor (VirtualBox Engine & Kernel Drivers)  |
+------------------------------+-------------------------------+
|   Virtual Machine A (Guest)  |   Virtual Machine B (Guest)   |
|   +----------------------+   |   +----------------------+    |
|   | Guest OS             |   |   | Guest OS             |    |
|   | Virtual Devices/NIC  |   |   | Virtual Devices/NIC  |    |
+---+----------------------+---+---+----------------------+----+
```

<details>
<summary><strong>Read more</strong> — how a Type-2 hypervisor sits on your machine</summary>

**What is Oracle VM VirtualBox?** An open-source, cross-platform Type-2 hypervisor for Windows, Linux and macOS hosts. It gives each guest virtualized CPU cores, configurable memory, emulated storage controllers (SATA, NVMe, SCSI, IDE) and software-defined network interfaces. That isolation makes it a standard tool for security evaluation, automated build pipelines, protocol analysis and cross-platform testing.

</details>

---

## 2. Understanding and Acquiring OVA Appliances

A VM is not a single file: it is a set of configuration files and disk images. To make them portable between hypervisors, the DMTF defined the **Open Virtualization Format (OVF)**.

### OVA vs. OVF

**OVF** is a directory package of separate files; **OVA** is that same package in a single `tar` archive.

<details>
<summary><strong>Read more</strong> — what is inside an OVF package</summary>

*   **OVF (Open Virtualization Format):** a set of uncompressed files — an XML descriptor (`.ovf`) describing the virtual hardware (vCPUs, RAM, disk controllers, network cards), one or more disk images (`.vmdk` or `.vhd`), and an optional manifest (`.mf`) of SHA checksums for integrity.
*   **OVA (Open Virtual Appliance):** the whole OVF package packed into one POSIX `tar` archive. Being a single file, it avoids fragmentation and configuration drift during download or transfer.

</details>

### Downloading OVA files safely

Pre-built appliances come from official maintainers (Kali Linux, Ubuntu Cloud Images, Debian, or dedicated security-lab distributions). Two habits keep you safe.

1.  **Download source:** take images only from official developer portals or verified mirrors.
2.  **Cryptographic verification:** large images can be corrupted in transit or tampered with, so always fetch the vendor's published SHA-256 checksum over HTTPS.

Before opening VirtualBox, verify the archive against that hash in your host terminal:

*   **Linux / macOS:**

        ```bash
        sha256sum appliance_image.ova
        ```

*   **Windows (PowerShell):**

        ```powershell
        Get-FileHash -Algorithm SHA256 .\appliance_image.ova
        ```

Compare the computed digest character for character against the vendor's official value.

---

## 3. Importing and Configuring an OVA in VirtualBox

Importing unpacks the tarball, translates the hardware description into VirtualBox's own `.vbox` XML, and copies the virtual disks into local storage.

1.  Launch **Oracle VM VirtualBox Manager**.
2.  Go to **File** $\rightarrow$ **Import Appliance...** (or `Ctrl + I` / `Cmd + I`).
3.  Under **Source**, choose **Local File System**, click the folder icon, and select your verified `.ova` file. Click **Next**.
4.  On the **Appliance Settings** screen, review the target specifications:
    *   **Name & Base Folder:** adjust the target directory if you keep VM disks on a dedicated NVMe or secondary storage pool.
    *   **CPU & RAM:** size these in proportion to the host, and avoid exceeding the physical core count or about $50\text{--}60\%$ of host memory, which would cause host paging and thrashing.
    *   **MAC Address Policy:** choose **Generate new MAC addresses for all network adapters** if you will clone the machine or run several instances at once; identical MACs on one layer-2 segment cause ARP collisions and packet loss.
5.  Click **Finish**. VirtualBox unpacks and converts the disks, and the new VM appears in your inventory, ready for network provisioning.

<details>
<summary><strong>Read more</strong> — where the import wizard does its work</summary>

The import is more than a copy. It unpacks the OVA tarball, translates the DMTF hardware schema into VirtualBox's native `.vbox` XML layout, and converts or copies each virtual disk into the local storage repository.

Because the disks are converted here, this is also the moment to decide **where** they live. Keeping VMs on a fast SSD or NVMe makes an enormous difference to boot times and to the responsiveness of the guests you will run throughout the course.

</details>

---

## 4. VirtualBox Network Modes: Architectural Analysis

VirtualBox offers several networking modes, and the right choice depends on how much isolation and reach you need between host, guest and the outside world. The diagram contrasts the two you will use most:

```
       +-------------------------------------------------------+
       |                     Physical LAN                      |
       +-------------------------------------------------------+
                                  ^
                                  | (NAT Masquerade / Bridged)
       +--------------------------v----------------------------+
       |                   Host Machine (Physical NIC)         |
       +-------------------------------------------------------+
           |                                       |
    [NAT Engine / Virtual Router]          [Host-Only Virtual Adapter]
           |                                       |
    +------v------+                         +------v------+
    | NAT Network |                         |  Host-Only  |
    | 10.0.2.0/24 |                         | 192.168.56.x|
    +------+------+                         +------+------+
           |                                       |
     +-----+-----+                           +-----+-----+
     |           |                           |           |
  +--v---+    +--v---+                    +--v---+    +--v---+
  | VM 1 |    | VM 2 |                    | VM 1 |    | VM 2 |
  +------+    +------+                    +------+    +------+
  (Inter-VM Comms OK,                     (Host <-> Guest Comms OK,
   Outbound Internet OK)                   No Outbound Internet)
```

The table below is the fastest way to choose:

| Network Mode | VM $\rightarrow$ Internet | Host $\rightarrow$ VM Direct | VM $\rightarrow$ Host Direct | Inter-VM Communication |
| :--- | :---: | :---: | :---: | :---: |
| **Default NAT** | Yes | No (requires port forwarding) | Limited (via Gateway IP) | No |
| **Host-Only** | No | Yes (direct IP) | Yes (direct IP) | Yes |
| **NAT Network** | Yes | No (requires port forwarding) | Limited (via Gateway IP) | Yes |
| **Bridged** | Yes | Yes (peers on physical LAN) | Yes | Yes |

### Host-Only Adapter vs. NAT Network — the key differences

These are the two modes students most often mix up, so here is the short version:

| | **Host-Only Adapter** | **NAT Network** |
| :--- | :--- | :--- |
| **Purpose** | Host ↔ VM and VM ↔ VM traffic on a private, offline segment | VM ↔ VM traffic **plus** shared internet access |
| **Internet access** | ❌ None — fully isolated from the outside world | ✅ Yes — outbound through the host's connection |
| **Host → VM** | ✅ Direct (ping / browse / SSH the VM's IP) | ❌ Only through port forwarding |
| **VM → VM** | ✅ Yes | ✅ Yes |
| **Typical use** | Malware analysis, packet capture, exercises on a closed network | Multi-node labs that also need packages, updates and downloads |

**Rule of thumb:** if your VMs need the internet, use a **NAT Network**; if they must not touch the internet, use a **Host-Only Adapter**. To get both at once, dual-home the VM with one adapter of each type (see §6).

> ⚠️ **A NAT Network is not the same as NAT.** The **NAT** option is the *per-VM* default: it hides each machine on its own private subnet, so two NAT VMs cannot see each other. A **NAT Network** is a *shared, named* network that you create once and attach many VMs to. When you configure a VM you must pick **NAT Network** from the **Attached to:** dropdown — **not** **NAT** — and then select the network name you created.

### How to enable a NAT Network (not NAT)

1. Open **Tools → Network → NAT Networks** (this is the only place a NAT Network is created).
2. Click **Create** (＋). A shared network named `NatNetwork` appears.
3. Select it, set a non-conflicting **IPv4 Prefix** (e.g. `10.0.2.0/24` or `192.168.15.0/24`) and tick **Enable DHCP**.
4. Click **Apply**. The network now exists globally and can be attached to any number of VMs.
5. Per VM (**Settings → Network → Adapter 1**), set **Attached to: NAT Network** and choose the network by name. Do **not** choose the plain **NAT** entry.

<details>
<summary><strong>Read more</strong> — what each network mode actually does</summary>

**Mode 1: Default NAT.** The default for a new VM. VirtualBox builds an isolated virtual router *per VM* on its own subnet (typically `10.0.2.0/24`), giving the guest `10.0.2.15` with gateway `10.0.2.2`, and rewrites outbound packets to look as if they came from the host's physical IP. The isolation cuts both ways: neither the internet nor the host can reach the guest without manual port forwarding, and two NAT VMs cannot talk to each other.

**Mode 2: Host-Only Adapter.** VirtualBox creates an emulated software adapter on the host (`vboxnet0` on Linux/macOS, `VirtualBox Host-Only Ethernet Adapter` on Windows) and a private subnet (e.g. `192.168.56.0/24`) shared only by the host and the attached guests. The host can ping, SSH or browse the VMs at their IPs, and guests on the same adapter can reach each other — but nothing is forwarded to the physical NIC, so the internet is unreachable. That is what makes it ideal for malware analysis, protocol fuzzing and closed sandboxes.

**Mode 3: NAT Network.** This combines NAT's outbound routing with the shared broadcast domain of an internal network. Instead of one router per machine, VirtualBox runs a **shared software router and switch**: all attached VMs share a subnet (e.g. `10.0.2.0/24`), reach each other on standard TCP/UDP ports and broadcasts, and reach the internet through the router's stateful SNAT. Inbound connections stay blocked unless you declare port-forwarding rules on the router itself.

</details>

---

## 5. Setting Up a NAT Network for Best Usage

To let several VMs talk to one another while still reaching the internet, create and assign a dedicated NAT Network.

### Step 1: Provision the global NAT Network

1.  In VirtualBox, open **Tools** (the hamburger/grid icon, top left) and choose **Network**.
2.  Select the **NAT Networks** tab.
3.  Click **Create** (the plus icon). VirtualBox creates an interface named `NatNetwork` by default.
4.  Highlight `NatNetwork` and set its properties:
    *   **IPv4 Prefix:** a non-conflicting RFC 1918 CIDR block, such as `10.0.2.0/24` or `192.168.15.0/24`, that does not clash with your host's physical LAN.
    *   **Enable DHCP:** tick this so VirtualBox's built-in DHCP server hands guests their leases automatically at boot.
    *   **Enable IPv6:** leave it off unless your topology really evaluates dual-stack or IPv6 routing.
5.  Click **Apply**.

### Step 2: Attach virtual machines to the NAT Network

For every VM that should join this shared environment:

1.  Make sure the VM is powered down.
2.  Select the VM and click **Settings** (or `Ctrl + S`).
3.  Open the **Network** category.
4.  Select **Adapter 1**:
    *   Check **Enable Network Adapter**.
    *   In **Attached to:** pick **NAT Network** — *not* the plain **NAT** entry, which would give this VM a private network of its own.
    *   In **Name:** pick the network from Step 1 (e.g. `NatNetwork`).
5.  Expand **Advanced**:
    *   **Adapter Type:** keep `Intel PRO/1000 MT Desktop (82540EM)` for broad compatibility, or `Paravirtualized Network (virtio-net)` for a Linux guest configured for paravirtualized I/O.
    *   **Promiscuous Mode:** `Deny` for normal use; `Allow VMs` or `Allow All` only when you run a packet analyser (Wireshark, Suricata) inside a guest.
6.  Click **OK**, and repeat the mapping for each machine.

---

## 6. Accessing Services: Port Forwarding vs. Dual-Homing

Because a NAT Network hides guests behind an internal router, reaching their services from the host (SSH, a local web server) needs one of two setups.

### Option A: NAT Network port forwarding

Use this when you need only a few specific ports (SSH on 22, HTTP on 80).

1.  Go to **Tools** $\rightarrow$ **Network** $\rightarrow$ **NAT Networks** $\rightarrow$ select `NatNetwork`.
2.  Click **Port Forwarding**.
3.  Add a rule:
    *   **Name:** `SSH-VM1`
    *   **Protocol:** `TCP`
    *   **Host IP:** `127.0.0.1` (binding to localhost keeps the lab VM off your physical LAN)
    *   **Host Port:** `2222`
    *   **Guest IP:** `10.0.2.4` (the guest's internal IP)
    *   **Guest Port:** `22`
4.  From the host terminal, connect through the forwarded port:

    ```bash
    ssh -p 2222 user@127.0.0.1
    ```

### Option B: Dual-homed architecture (the recommended setup)

Forwarding dozens of ports across many VMs gets tedious. The cleanest layout for development, penetration testing and multi-node clusters is a **dual-homed** VM — one adapter for the internet and peers, one for direct host access:

*   **Adapter 1 $\rightarrow$ NAT Network:** outbound internet plus an internal LAN for VM-to-VM traffic.
*   **Adapter 2 $\rightarrow$ Host-Only Adapter:** direct, uninhibited layer-3 access from the host to every VM service, with no per-port rules to manage.

```
       +-----------------------------------------------------------+
       |                        Host Machine                       |
       |                   IP: 192.168.56.1 (vboxnet0)             |
       +-----------------------------------------------------------+
               |                                           |
    [Host-Only Subnet: 192.168.56.0/24]      [NAT Network: 10.0.2.0/24]
               |                                           |
       +-------v-------------------------------------------v-------+
       |               Virtual Machine (Dual-Homed)                |
       |                                                           |
       |  Adapter 2 (eth1): 192.168.56.101  <-- Direct Host Access |
       |  Adapter 1 (eth0): 10.0.2.15       <-- Internet & Peers   |
       +-----------------------------------------------------------+
```

To configure it:

1.  In the VM's **Settings** $\rightarrow$ **Network**, leave **Adapter 1** on **NAT Network**.
2.  Open the **Adapter 2** tab:
    *   Check **Enable Network Adapter**.
    *   In **Attached to:** select **Host-Only Adapter**.
    *   Select your host's interface (e.g. `VirtualBox Host-Only Ethernet Adapter` or `vboxnet0`).
3.  Boot the VM. The guest enumerates two interfaces (e.g. `eth0` and `eth1`).
    *   The default gateway resolves via Adapter 1 to reach external repositories.
    *   Adapter 2 receives an address on the Host-Only subnet (e.g. `192.168.56.101`), so you can target its services directly from the host without per-port maps.
