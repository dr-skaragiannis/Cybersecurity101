
# Detailed Notes — Lab 00: Virtualization

This file is the **expanded edition** of the lab's `README.md`.

# VirtualBox Architecture, OVA Import, and Advanced Network Configuration

> ### ⚡ Recommended Quick Start — Parrot OS + OVA Import
>
> **Use Kali (or a comparable distribution).** Throughout this
> course, **Kali Linux** is the recommended attack platform; however, you are free to use any distribution or operating system you prefer.
>
> **The fastest way to get started is to import the prebuilt OVA — no installation required:**
>
> 1. Download the **prebuilt Kali Linux image for VirtualBox** from
>    <https://www.kali.org/get-kali/#kali-platforms> — select the **Virtual Box** option. On macOS, UTM is preferable to VirtualBox.
> 2. Before booting, ensure that virtualization is enabled in the
>    BIOS/UEFI (Intel VT-x / AMD-V), then start the machine. Log in with the
>    default credentials listed on the download page.
>
> The remainder of this guide explains how these virtual machines work, how
> to import them manually, and how to configure their networking correctly.

---

## 1. Introduction to Virtualization and Oracle VM VirtualBox


### Explanation §1 — What a "Virtual Machine" Means in Practice

A **virtual machine (VM)** is a *file* (or set of files) that contains an entire "computer": a virtual disk, a virtual network adapter, virtual RAM, and virtual CPU cores. The hypervisor acts as emulated ("virtual") hardware: when the guest operating system writes to its "disk," the hypervisor translates that operation into a write within a `.vdi` file on the physical system (the host).

**Why this matters in this course:** every attack lab requires a *victim* that we can freely break. A VM provides:

1. **Isolation** — malware or an exploit remains confined to the guest.
2. **Repeatability** — with a snapshot, we can return to a clean state within seconds.
3. **Multiple roles** — the same physical machine can simultaneously host Kali (the attacker) and DC-1 (the target).

**Analysis of the architecture described in the original text:**

* **Ring -1 / root mode:** the CPU has a privilege level *below* the operating system. This is where the hypervisor runs when VT-x/AMD-V is available. Without it, the hypervisor must "translate" every privileged instruction (binary translation), which is slower.
* **EPT / NPT:** *Extended Page Tables* (Intel) and *Nested Page Tables* (AMD) allow the CPU to translate guest addresses to host hardware addresses without the hypervisor intervening on every memory access.
* **Guest Additions:** optional drivers inside the guest that improve graphics, shared folders, and clock synchronization. They are not required for security, but they make life easier.

**Common misconception:** "a VM is more secure than the host." This holds **only** if the hypervisor is kept up to date. *VM escape* attacks exist that break out of the guest into the host, which is why we do not open untrusted images on a machine that holds sensitive data.

**Quick check:** Why is a VM with 4 vCPUs not "four times faster"? *(Answer: vCPUs share the physical cores; if the host has 4 cores and two VMs with 4 vCPUs each are running, the hypervisor time-shares the cores and all of them slow down.)*

Virtualization places a **hypervisor** between the hardware and the software that runs on it. The hypervisor emulates an entire hardware platform — CPU, memory, disk, and network — and each such platform constitutes a **Virtual Machine (VM)**.

Hypervisors fall into two categories:

* **Type 1 (Bare-Metal):** These run directly on the physical hardware, with no operating system beneath them (VMware ESXi, KVM, Xen, Proxmox VE). They are fast and low-latency, which is why they are the norm in data center infrastructure.
* **Type 2 (Hosted):** These are installed as an ordinary application within an operating system (Oracle VM VirtualBox, VMware Workstation). The host manages the physical devices, while the hypervisor mediates on behalf of the virtual machine.

Modern x86 processors add a third dimension to this picture: **hardware-assisted virtualization**. Intel **VT-x** (with **EPT** for nested paging) and AMD **AMD-V** (with **NPT**) allow the virtual machine to execute most instructions directly at a reduced privilege level, while sensitive instructions are trapped by the hypervisor — this is how a VM runs at near-native speed.

```
+--------------------------------------------------------------+
|             Physical Host Hardware (CPU, RAM, Network Card)  |
+--------------------------------------------------------------+
|            Host Operating System (Linux / Windows / macOS)   |
+--------------------------------------------------------------+
|       Type 2 Hypervisor (VirtualBox Engine & Kernel Drivers) |
+------------------------------+-------------------------------+
| Virtual Machine A (Guest VM) | Virtual Machine B (Guest VM)  |
|   +----------------------+   |   +----------------------+    |
|   | Guest OS             |   |   | Guest OS             |    |
|   | Virtual Devices      |   |   | Virtual Devices      |    |
+---+----------------------+---+---+----------------------+----+
```

<details>
<summary><strong>More</strong> — how a Type 2 hypervisor sits on your machine</summary>

**What is Oracle VM VirtualBox?** An open-source, cross-platform Type 2 hypervisor for Windows, Linux, and macOS. It provides each virtual machine with virtual CPU cores, configurable memory, emulated storage controllers (SATA, NVMe, SCSI, IDE), and virtual network adapters. This isolation makes it an essential tool for security testing, automated deployment pipelines, protocol analysis, and software testing.

</details>

---

## 2. Understanding and Downloading OVA Files

A virtual machine is not a single file: it is a collection of configuration files and disk images. To make virtual machines portable across hypervisors, the DMTF established the **Open Virtualization Format (OVF)** standard.

### Difference Between OVF and OVA


### Explanation §2 — Why a Standard Exists and What We Gain

Without a standard, every hypervisor would have its own format and would have to "guess" the settings of the others. **OVF** solves this problem by separating *what the machine is* (its hardware, described in XML) from its *contents* (the disks).

**The three components of an OVF package, with an example:**

| File | Role | What you will see if you open it |
| :--- | :--- | :--- |
| `machine.ovf` | Hardware description in XML | `<VirtualSystem>`, `<Item>` with `ovf:capacity` (disk size), `<VirtualHardwareSection>` with vCPU/RAM |
| `machine-disk1.vmdk` | The disk | Binary file, typically 2–20 GB |
| `machine.mf` | Checksum manifest | Lines of the form `SHA256(machine.ovf)= ...` |

**Why OVA is safer to transfer:** because it is a single file, no component can go "missing" (e.g., downloading the `.ovf` but not the disk). Moreover, because `tar` preserves file order, the manifest can be verified in its entirety.

**Practical verification (Linux/macOS):**

```bash
tar -tf kali.ova                  # lists the contents without extracting
tar -xf kali.ova kali.mf && cat kali.mf   # view the hashes
sha256sum kali-disk1.vmdk          # compare against the .mf
```

**Common mistake:** downloading an "OVF" from a website and receiving only the `.ovf` file. The import then fails with "file not found" — the disk is missing.

**OVF** is a folder of separate files; **OVA** is the same package bundled into a single `tar` file.

<details>
<summary><strong>More</strong> — what an OVF package contains</summary>

* **OVF (Open Virtualization Format):** a set of uncompressed files — an XML descriptor (`.ovf`) containing the hardware specification (vCPUs, RAM, disk controllers, network adapters), one or more disk images (`.vmdk` or `.vhd`), and an optional integrity manifest (`.mf`) containing SHA hash values. The format is defined by the DMTF in specification **DSP0243**, and it is the manifest that makes the package verifiable end to end.
* **OVA (Open Virtual Appliance):** the entire OVF package bundled into a single POSIX `tar` archive. Because it is a single file, it avoids loss and incompatibility problems during download or transfer.

</details>

---

## 3. Importing and Configuring the OVA in VirtualBox


### Explanation §3 — A Field-by-Field Analysis of the Import

Importing is a **format conversion**: the DMTF template becomes a VirtualBox `.vbox` record. In practice, three things happen in sequence:

1. **Validation** — the manifest is checked; if the SHA hash does not match, VirtualBox issues a warning (possibly a partial download or corruption).
2. **Hardware mapping** — the OVF `<Item>` elements become settings: `VirtualSystemType` → guest OS type, `ovf:capacity` → virtual disk size.
3. **Disk conversion/copy** — if the disk is a `.vmdk`, it may be copied as is or converted to `.vdi`.

**The three decisions you must make in the import dialog:**

* **RAM:** Kali requires **at least 2 GB** (4 GB is comfortable). Rule: *the total RAM of all VMs running simultaneously ≤ 60–70% of physical RAM*. If you exceed this, the host begins *swapping* and everything slows down — often more than a VM with less memory would.
* **vCPU:** allocate **1–2**. Assigning more cores than physically exist causes contention, not acceleration.
* **MAC address policy:** every virtual adapter has a MAC address. Two VMs with the **same MAC** on the same network cause *ARP conflicts*: the switch cannot forward frames correctly, and packets end up at the wrong machine. For this reason, when cloning, always select **Generate new MAC addresses**.

**Verifying a successful import:** the machine appears in the list without a yellow exclamation mark. If you see a warning, it usually concerns a *missing shared folder* or an *audio device* — harmless for lab purposes.

**Useful CLI commands (for verification without the GUI):**

```bash
VBoxManage import kali.ova --vsys 0 --cpus 2 --memory 4096
VBoxManage list vms          # confirm registration
```

The import process unpacks the tar archive, translates the hardware specification into VirtualBox's native XML schema (`.vbox`), and copies the virtual disks to local storage.

1. Open **Oracle VM VirtualBox Manager**.
2. Navigate to **File** $\rightarrow$ **Import Appliance...** (or `Ctrl + I` / `Cmd + I`).
3. In the **Source** field, select **Local File System**, click the folder icon, and choose the `.ova` file. Click **Next**.
4. On the **Appliance Settings** tab, review the specifications:
   * **Name & Base Folder:** adjust the destination folder if you keep VM disks on a separate NVMe drive or secondary storage.
   * **CPU & RAM:** allocate resources according to the host's capacity, and avoid exceeding the number of physical cores or 50–60% of total memory, so as to prevent swapping and performance degradation.
   * **MAC Address Policy:** select **Generate new MAC addresses for all network adapters** if you intend to clone the machine or run multiple copies; identical MAC addresses on the same Layer 2 segment cause ARP conflicts and packet loss.
5. Click **Finish**. VirtualBox unpacks and converts the disks, and the new machine appears in the list, ready for network configuration.

<details>
<summary><strong>More</strong> — where the import wizard does its work</summary>

Importing is more than copying. It unpacks the OVA archive, translates the DMTF hardware schema into VirtualBox's native XML layout (`.vbox`), and converts or copies each virtual disk to local storage.

Because the disks are converted at this stage, this is the moment to decide **where** they will be stored. A fast SSD/NVMe drive makes an enormous difference to boot times and to the responsiveness of the virtual machines.

</details>

### Snapshots, Disk Formats, and Restoration


### Explanation §3.1 — How Copy-on-Write Works in Practice

Think of the base disk as **frozen**: as soon as you take a snapshot, the disk becomes read-only and a *second*, empty file (the differencing image) is created. Every new write goes **only** to the second file. When a block is read, VirtualBox first checks the differencing image and, if the block is not there, falls back to the base disk.

**What this means in practice:**

* A snapshot is **instantaneous** (it does not copy gigabytes of data).
* After many changes, the differencing image grows — it may even exceed the size of the original disk.
* A **chain** of snapshots (snapshot upon snapshot) adds latency to every I/O operation and makes the machine's state difficult to reason about.

**Critical distinction — snapshot ≠ backup:** if the host's disk fails, **both** the base disk **and** the snapshots are lost, because they reside on the same physical disk. A backup means *a copy on a different medium*.

**Recommended workflow for every lab:**

```bash
VBoxManage snapshot "DC-1" take "clean-install"     # before you start
VBoxManage snapshot "DC-1" list                     # view the list
VBoxManage snapshot "DC-1" restore "clean-install"  # revert afterward
VBoxManage snapshot "DC-1" delete "clean-install"   # clean up when finished
```

**Common mistake:** taking a snapshot while the VM is in a *saved state* rather than *powered off*. The result may be an inconsistent file system in the guest. Always shut down cleanly (`sudo shutdown -h now`).

VirtualBox stores each disk either in its own native format (`.vdi`) or in the interoperable `.vmdk`/`.vhd` formats. A **snapshot** does not copy the entire disk: it freezes the current state and opens a **differencing image** that records every subsequent write using copy-on-write — which is why it is fast and inexpensive.

In a security lab, this is the single most useful safety habit: take a snapshot *before* every destructive exercise (**Machine → Take Snapshots**, or `VBoxManage snapshot "<vm>" take "clean-before-lab"`) and revert to it afterward. Two consequences follow from the copy-on-write design: the differencing image grows with every write (snapshots are **not** backups — copy the `.vdi` file instead), and a long snapshot chain slows the machine down, so merge them once the exercise is complete.

---

## 4. Networking Modes in VirtualBox: An Architectural Analysis


### Explanation §4 — The Network as an Isolation Tool

In a security lab, the choice of network **is** part of the exercise: it determines who can see whom. Consider three questions:

1. *Does the VM need to reach the internet?* (updates, downloading tools)
2. *Does the host need to communicate with the VM?* (SSH, browser, file transfer)
3. *Do the VMs need to communicate with one another?* (Kali ↔ target)

The answers lead to different modes. For Labs 02–04, the answer is usually **1 = not always, 2 = yes, 3 = yes** → **NAT Network** or **Host-Only**.

**The technical detail that explains all of this behavior:** VirtualBox implements a *virtual switch* and, depending on the mode, a *virtual router*:

* **NAT** = a separate router **per VM**, each with its own `10.0.2.0/24`. This is why two VMs on NAT cannot see each other.
* **NAT Network** = **a single shared** router/switch for all VMs attached to that named network.
* **Host-Only** = a switch only, **without** an outward-facing router.
* **Bridged** = the virtual adapter "piggybacks" on the physical one and obtains an IP address from the home/office router.

**Lab caution:** if you use **Bridged**, the vulnerable VM obtains an address on your real LAN. This is the easiest way to expose a target machine to other computers on the network. Use it only if the exercise requires it, and always on an isolated network.

VirtualBox offers several networking modes, and the correct choice depends on the degree of isolation and access you require between the host, the VMs, and the external network. The diagram contrasts the two modes you will use most often:

```
       +-------------------------------------------------------+
       |                     Physical LAN                      |
       +-------------------------------------------------------+
                                  ^
                                  | (NAT Masquerade / Bridged)
       +--------------------------v----------------------------+
       |                Host Machine (Physical NIC)            |
       +-------------------------------------------------------+
           |                                       |
    [NAT Engine / Virtual Router]           [Host-Only Virtual Adapter]
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
  (VM-to-VM Communication: YES,           (Host <-> VM Communication: YES
   Internet Access: YES)                   Internet Access: NO)
```

The table below is the quickest way to choose:

| Network Mode | VM $\rightarrow$ Internet | Host $\rightarrow$ VM Directly | VM $\rightarrow$ Host Directly | VM-to-VM Communication |
| :--- | :---: | :---: | :---: | :---: |
| **Default NAT** | Yes | No (requires Port Forwarding) | Limited (via Gateway) | No |
| **Host-Only** | No | Yes (direct IP) | Yes (direct IP) | Yes |
| **NAT Network** | Yes | No (requires Port Forwarding) | Limited (via Gateway) | Yes |
| **Bridged**| Yes | Yes (as a peer device on the LAN)| Yes | Yes |

### Host-Only Adapter vs. NAT Network — The Key Differences


### Explanation §4.2 — How to Decide in 10 Seconds

A practical decision algorithm:

```
Does the VM need internet access?
├── NO   -> Host-Only  (complete isolation, ideal for malware/forensics)
└── YES
     ├── Must it also see other VMs? -> NAT Network
     └── Must it be a "device" on the physical LAN? -> Bridged (with caution)
```

**Examples from the course:**

| Scenario | Choice | Reason |
| :--- | :--- | :--- |
| Kali attacks DC-1 | **NAT Network** | Both machines must communicate, and Kali runs `apt update` |
| Analysis of a suspicious file | **Host-Only** | The sample must not be able to contact an external command-and-control server (phone-home) |
| Lab with a single VM and no others | **NAT** | Simple; sufficient for browsing/updates |
| Attack from a second physical laptop | **Bridged** | The two hosts must be on the same L2 segment |

**What "Port Forwarding" means:** under NAT, the router does not permit inbound connections. To reach port 80 on the VM from the host, you define a rule such as `host:8080 → guest:80`. In a NAT Network, the rule is placed on the **shared** router, not per machine.

**Exam trap:** "NAT Network gives each VM its own subnet." **False** — that is what plain NAT does. NAT Network gives **a shared** subnet to all of them.

These two modes are the ones most frequently confused, so here is the short answer:

| | **Host-Only Adapter** | **NAT Network** |
| :--- | :--- | :--- |
| **Purpose** | Host ↔ VM and VM ↔ VM communication on a private, offline network | VM ↔ VM communication **and** shared internet access |
| **Internet access** | ❌ None — fully isolated | ✅ Yes — outbound via the host's connection |
| **Host → VM** | ✅ Direct (ping / browse / SSH to the VM's IP) | ❌ Only with port forwarding |
| **VM → VM** | ✅ Yes | ✅ Yes |
| **Typical use** | Malware analysis, traffic capture, closed-network exercises | Multi-node labs that require updates and downloads |

**Rule of thumb:** if the VMs need internet access, use **NAT Network**; if they must not reach the internet, use **Host-Only Adapter**. To have both, give the VM two adapters (dual-homed).

> ⚠️ **NAT Network is not the same as NAT.** The **NAT** option is the *per-VM default*: it hides each machine in its own private subnet, so two VMs on plain NAT cannot see each other. **NAT Network** is a *shared, named* network that you create once and to which you attach multiple VMs. When configuring the machine, you must select **NAT Network** in the **Attached to:** field — **not** **NAT** — and then choose the name of the network you created.

### How to Enable a NAT Network (Not NAT)


### Explanation §4.3 — Why "Central" Creation Is Required

A NAT Network is not created from the machine's settings window because it **does not belong to any single machine**: it is part of VirtualBox's infrastructure. That is why it is located under `Tools → Network`. If you look in the VM's settings, you will only see its name available for selection.

**Verification from the terminal (useful when the GUI "hangs"):**

```bash
VBoxManage natnetwork list
VBoxManage natnetwork add --netname LabNet --network "10.0.2.0/24" --enable --dhcp on
VBoxManage natnetwork start --netname LabNet
```

**Criteria for choosing the IPv4 prefix:**

* It must be an **RFC 1918** range (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`).
* It must **not** overlap with your physical network — otherwise the host misroutes traffic and the VM becomes unreachable.
* `10.0.2.0/24` is the customary choice and does not conflict with home networks (`192.168.x.x`).

**Why DHCP should be on:** guests obtain an IP address automatically, without static configuration. If you disable it, you must manually assign an IP address, netmask, and gateway to every guest.

1. Open **Tools → Network → NAT Networks** (this is the only place where a NAT Network can be created).
2. Click **Create (＋)**. A shared network named `NatNetwork` appears.
3. Select it, set an **IPv4 Prefix** that does not conflict with your physical network (e.g., `10.0.2.0/24` or `192.168.15.0/24`), and enable **Enable DHCP**.
4. Click **Apply**. The network now exists centrally and can be attached to as many VMs as you wish.
5. On each VM (**Settings → Network → Adapter 1**), select **Attached to: NAT Network** and choose the network by name. Do not select plain **NAT**.

<details>
<summary><strong>More</strong> — what each networking mode actually does</summary>

**Mode 1: Default NAT.** The default for every new machine. VirtualBox builds an isolated virtual router *per machine* on its own subnet (typically `10.0.2.0/24`), assigning the guest `10.0.2.15` with gateway `10.0.2.2`, and rewrites outbound packets so that they appear to originate from the host's physical IP address. The isolation is bidirectional: neither the internet nor the host can reach the guest without manual port forwarding, and two VMs on plain NAT cannot communicate.

**Mode 2: Host-Only Adapter.** VirtualBox creates an emulated software adapter on the host (`vboxnet0` on Linux/macOS, `VirtualBox Host-Only Ethernet Adapter` on Windows) and a private subnet (e.g., `192.168.56.0/24`) shared only by the host and the attached machines. The host can ping, SSH into, or browse to their IP addresses, and VMs on the same adapter can communicate with one another — but nothing is forwarded to the physical network card, so the internet is unreachable. This makes it ideal for malware analysis, fuzzing, and closed sandboxes.

**Mode 3: NAT Network.** This combines NAT's outbound access with the shared broadcast domain of an internal network. Instead of one router per machine, VirtualBox runs a **shared virtual router and switch**: all attached machines share a single subnet (e.g., `10.0.2.0/24`), communicate over TCP/UDP and broadcasts, and reach the internet through the router's stateful SNAT. Inbound connections remain closed unless you define port-forwarding rules on the router itself.

</details>

---

## 5. Optimal NAT Network Configuration (Step by Step)


### Explanation §5 — What to Verify After Configuration

The configuration has "succeeded" only when **all three** of the following hold:

1. **IP address on the same subnet:** inside the guest, `ip a` (Linux) shows an address of the form `10.0.2.x`.
2. **Ping between VMs:** from Kali, `ping <target-IP>` receives replies.
3. **Internet egress:** `ping -c 3 8.8.8.8` receives replies (if not, check DNS: `cat /etc/resolv.conf`).

**Troubleshooting table:**

| Symptom | Likely cause | Solution |
| :--- | :--- | :--- |
| The VM does not obtain an IP address | DHCP disabled or wrong network | Enable DHCP / verify the network name |
| Ping to another VM fails | The VMs are attached to **different** networks or to plain NAT | Place them on the same NAT Network |
| Has an IP address but no internet | Incorrect gateway/DNS | `ip route`, `cat /etc/resolv.conf` |
| Reaches the internet but not the host | Expected behavior in a NAT Network | Use port forwarding or a second Host-Only adapter |
| Two VMs with the same MAC address | Cloned without generating a new MAC | Settings → Network → Advanced → new MAC |

**Dual-homed VM (both modes):** set `Adapter 2` to `Host-Only` and `Adapter 1` to `NAT Network`. The machine then has internet access **and** direct reachability from the host — the most flexible layout for a lab.

To allow multiple virtual machines to communicate with one another while retaining internet access, create and assign a dedicated NAT Network.

### Step 1: Creating the Central NAT Network

1. In the main VirtualBox window, open **Tools** (the icon with three lines at the top left) and select **Network**.
2. Select the **NAT Networks** tab.
3. Click **Create**. An entry named `NatNetwork` will be created.
4. Select `NatNetwork` and configure its properties:
   * **IPv4 Prefix:** a non-conflicting RFC 1918 range, such as `10.0.2.0/24` or `192.168.15.0/24`, that does not overlap with the host's physical network.
   * **Enable DHCP:** enable this so that the built-in DHCP server automatically assigns addresses to the machines at boot.
   * **Enable IPv6:** leave this disabled unless your scenario tests dual-stack operation or IPv6 routing.
5. Click **Apply**.

### Step 2: Attaching the Virtual Machines to the NAT Network


### Explanation §5.2 — Why the Adapter Type Matters

* **`Intel PRO/1000 MT Desktop (82540EM)`** — the default; it works everywhere, even with old guests. Use it unless you have a reason to choose something else.
* **`Paravirtualized Network (virtio-net)`** — faster on Linux guests, but requires drivers inside the guest. In old images (e.g., the Debian Wheezy image of DC-1), the driver may be absent and no network adapter will appear.

**Promiscuous mode:** when `tcpdump`/Wireshark runs **inside** a guest and needs to see the traffic of other VMs, `Allow All` or `Allow VMs` is required. In normal use, leave it at `Deny` — this is also a security consideration (preventing eavesdropping).

**Verification from the host (Windows/Linux):**

```bash
VBoxManage showvminfo "DC-1" | grep -i "NIC 1"
# expected: NIC 1: MAC: ..., Attachment: NAT Network, Name: LabNet
```

If `Attachment` shows `NAT` rather than `NAT Network`, the machine is on an isolated network and will not see the others — the most common mistake in this lab.

For each machine you want to join this shared environment:

1. Ensure that the machine is powered off.
2. Select the machine and click **Settings** (or `Ctrl + S`).
3. Navigate to the **Network** category.
4. On the **Adapter 1** tab:
   * Check **Enable Network Adapter**.
   * In **Attached to:**, select **NAT Network** — *not* the plain **NAT** option, which places the machine on a separate private network.
   * In **Name:**, select the network from Step 1 (e.g., `NatNetwork`).
5. Expand **Advanced**:
   * **Adapter Type:** keep `Intel PRO/1000 MT Desktop (82540EM)` for broad compatibility, or choose `Paravirtualized Network (virtio-net)` for Linux guests with paravirtualized I/O.
   * **Promiscuous Mode:** `Deny` for normal use; `Allow VMs` or `Allow All` only when running a packet analyzer (Wireshark, Suricata) inside a guest.
6. Click **OK** and repeat the procedure for each machine.

---

---


## Appendix A — Glossary

| Term | Meaning |
| :--- | :--- |
| **Hypervisor** | Software that creates and runs VMs |
| **Type 1 / Type 2** | Runs on hardware / runs on an operating system |
| **Guest / Host** | The virtual system / the physical machine |
| **OVA / OVF** | A single `tar` file / a folder of files describing a machine |
| **vCPU** | A virtual processor as seen by the guest |
| **Snapshot** | A frozen state of disk and memory at a given point in time |
| **Differencing image** | A file that records only the changes made after the snapshot |
| **NAT** | Network address translation; many private addresses behind a single public one |
| **Port forwarding** | A rule that exposes a guest port through a host port |
| **Bridged** | The VM appears as a separate device on the physical LAN |
| **Promiscuous mode** | The adapter also accepts packets not addressed to it |

## Appendix B — Self-Assessment Questions

1. What is the difference between a Type 1 and a Type 2 hypervisor? Give one example and one advantage of each.
2. Why is OVA preferable to OVF for distribution to students?
3. How much RAM would you allocate to two VMs on a host with 8 GB? Justify your answer.
4. Why does a chain of five snapshots slow the machine down?
5. Which networking mode would you use to set up a lab with Kali + DC-1 + Raven 1 so that they also have internet access? Why **not** plain NAT?
6. Describe what changes when you select "Generate new MAC addresses" and what problem it prevents.
7. A guest has no network adapter at all. What is the most likely cause, and how do you fix it?

### Suggested Answers

1. A **Type 1 (bare-metal)** hypervisor runs directly on the hardware, with no intermediate operating system (e.g., VMware ESXi, Microsoft Hyper-V, Xen); advantage: maximum performance and isolation, ideal for servers/data centers. A **Type 2 (hosted)** hypervisor runs as an application within a host OS (e.g., VirtualBox, VMware Workstation/Player); advantage: easy installation and use on a personal computer, ideal for labs. Type 1 has lower overhead because no host OS sits in between.
2. **OVF** is a set of separate files (the `.ovf` descriptor, `.vmdk` disks, and the `.mf` manifest), whereas **OVA** is a single package file (tar) that contains them all. For distribution to students, OVA is preferable because it is **one** file: no component can be forgotten or lost, it is easier to transfer, and the checksum is verified against a single file.
3. The host needs approximately 2–3 GB to operate smoothly, leaving approximately 5 GB for the VMs — a reasonable allocation is **~2 GB per VM** (4 GB in total), leaving headroom for the host. We do not assign all of the RAM to the VMs: the host would then swap to disk, and both host and guests would slow down. Rule: *Total VM RAM ≤ Total RAM − RAM required by the host*.
4. Each snapshot is a differencing disk that holds only the changes since the previous one. With a chain of five snapshots, every read/write may need to traverse several layers to locate the most recent version of a block, increasing I/O and latency. For this reason, we keep few snapshots and merge/delete those that are no longer needed.
5. We use **NAT Network** (not plain NAT). Under plain NAT, each VM is on a separate, isolated network and the VMs **cannot see one another** — Kali would be unable to attack DC-1/Raven. NAT Network places all VMs on the **same** virtual subnet (with DHCP) so that they can communicate, while also providing internet access through the host (e.g., for `apt`/downloads). Host-Only would allow communication between VMs, but **without** internet access.
6. It assigns **new, unique MAC addresses** to the virtual adapters instead of retaining those of the original (on clone/import). It prevents **MAC conflicts** (two machines with the same MAC on the same network), which cause confused ARP resolution, packet loss, and incorrect or duplicate IP assignment by the DHCP server.
7. Most likely cause: the VM's network adapter is **disabled/disconnected** (or was never added) in the settings. Fix: enable the adapter, select the correct mode (e.g., NAT Network), and tick "Cable connected"; if the driver is missing inside the guest, select a compatible adapter type (e.g., Intel PRO/1000) or install the Guest Additions.

## Appendix C — Pre-Lab Checklist

- [ ] Virtualization (Intel VT-x / AMD-V) enabled in the BIOS/UEFI
- [ ] At least 2 GB of RAM and 20 GB of disk space free for the VM
- [ ] A snapshot named `clean-install` taken before every exercise
- [ ] A NAT Network (not NAT) with DHCP enabled, shared by all VMs
- [ ] A `ping` check between the VMs before the attack begins
- [ ] A copy of the `.vdi` file on a different medium (if you want a genuine backup)
