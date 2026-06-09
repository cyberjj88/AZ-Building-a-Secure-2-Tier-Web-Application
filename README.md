# 🔐 Lab 02 — Building a Secure 2-Tier Web Application on Azure

**Difficulty:** 🟢 Beginner
**Estimated Time:** ⏱ 60 Minutes
**Cloud Provider:** Microsoft Azure
**Topic Area:** IaaS · Virtual Networking · NSGs · Network Segmentation

---

## 📋 Overview

This lab introduces the foundational **IaaS (Infrastructure as a Service)** pattern used in real-world cloud deployments: the **2-Tier Architecture**. You will manually provision two virtual machines — a web server and a database server — and place them in separate, isolated network subnets inside an Azure Virtual Network.

The core security lesson here is **network segmentation**: the database server has no public IP address and is only reachable from the web subnet. This mirrors how production environments are designed to protect sensitive backend systems from direct internet exposure.

---

## 🏗️ Architecture Diagram

```
                         PUBLIC INTERNET
                               │
                        HTTPS / SSH (22)
                               │
          ┌────────────────────▼────────────────────────┐
          │            AZURE — East US Region            │
          │                                              │
          │   ┌──────────────────────────────────────┐   │
          │   │   Resource Group: rg-lab02-[yourname] │   │
          │   │                                      │   │
          │   │  ┌───────────────────────────────┐   │   │
          │   │  │   VNet: vnet-lab02             │   │   │
          │   │  │   Address Space: 10.0.0.0/16   │   │   │
          │   │  │                               │   │   │
          │   │  │  ┌─────────────────────────┐  │   │   │
          │   │  │  │  🌐 PUBLIC SUBNET        │  │   │   │
          │   │  │  │  snet-web (10.0.1.0/24)  │  │   │   │
          │   │  │  │                         │  │   │   │
          │   │  │  │  ┌───────────────────┐  │  │   │   │
          │   │  │  │  │    vm-web-01       │  │  │   │   │
          │   │  │  │  │  Public IP ✅      │  │  │   │   │
          │   │  │  │  │  Ports: 80, 22     │  │  │   │   │
          │   │  │  │  │  NSG: Allow HTTP   │  │  │   │   │
          │   │  │  │  │       Allow SSH    │  │  │   │   │
          │   │  │  │  └─────────┬─────────┘  │  │   │   │
          │   │  │  └────────────┼────────────┘  │   │   │
          │   │  │               │ Internal       │   │   │
          │   │  │               │ VNet Traffic   │   │   │
          │   │  │               │ (10.0.1.0/24)  │   │   │
          │   │  │  ┌────────────▼────────────┐  │   │   │
          │   │  │  │  🔒 PRIVATE SUBNET       │  │   │   │
          │   │  │  │  snet-db  (10.0.2.0/24)  │  │   │   │
          │   │  │  │                         │  │   │   │
          │   │  │  │  ┌───────────────────┐  │  │   │   │
          │   │  │  │  │    vm-db-01        │  │  │   │   │
          │   │  │  │  │  Public IP ❌      │  │  │   │   │
          │   │  │  │  │  No direct internet│  │  │   │   │
          │   │  │  │  │  NSG: Allow ONLY   │  │  │   │   │
          │   │  │  │  │  from 10.0.1.0/24  │  │  │   │   │
          │   │  │  │  └───────────────────┘  │  │   │   │
          │   │  │  └─────────────────────────┘  │   │   │
          │   │  └───────────────────────────────┘   │   │
          │   └──────────────────────────────────────┘   │
          └──────────────────────────────────────────────┘

  TRAFFIC RULES:
  ✅ Internet  → vm-web-01   (Port 80, 22)         ALLOWED
  ✅ vm-web-01 → vm-db-01   (Internal VNet only)   ALLOWED
  ❌ Internet  → vm-db-01   (No Public IP)          BLOCKED
  ❌ vm-db-01  → Internet   (Outbound restricted)   BLOCKED
```

> **Key Security Principle:** The database server is not reachable from the internet — not because of a firewall rule alone, but because it has **no public IP address**. The NSG rule adds an explicit allowlist, permitting only the web subnet to initiate connections. This is **defense in depth**.

---

## 🎯 Objectives

By completing this lab, you will be able to:

- [x] Create an **Azure Virtual Network** with multiple subnets
- [x] Deploy **Linux Virtual Machines** into specific subnets
- [x] Understand the difference between a **public** and **private** subnet
- [x] Use a **jump host pattern** to access a private server via a public one
- [x] Configure **Network Security Group (NSG) inbound rules** to restrict access
- [x] Explain **network segmentation** as a core cloud security control

---

## ✅ Prerequisites

- [ ] Active Azure Subscription *(Free Tier is sufficient)*
- [ ] Completed Week 2 Video Modules
- [ ] Terminal installed *(macOS: built-in Terminal · Windows: Git Bash or Windows Terminal)*
- [ ] Basic familiarity with SSH

---

## 🏷️ Lab Variables & Naming Conventions

| Resource | Value |
|---|---|
| **Resource Group** | `rg-lab02-[yourname]` |
| **Location** | East US |
| **Virtual Network** | `vnet-lab02` |
| **Address Space** | `10.0.0.0/16` |
| **Subnet 1 (Public)** | `snet-web` · `10.0.1.0/24` |
| **Subnet 2 (Private)** | `snet-db` · `10.0.2.0/24` |
| **Web Server VM** | `vm-web-01` |
| **Database Server VM** | `vm-db-01` |
| **SSH Key Pair** | `key-lab02` |

---

## 🔬 Lab Instructions

### Phase 1 — Build the Network Foundation

1. In the Azure Portal, search for **Virtual Networks** → click **+ Create**.
2. Fill in the **Basics** tab:
   - **Resource Group:** Create New → `rg-lab02-[yourname]`
   - **Name:** `vnet-lab02`
   - **Region:** `East US`
3. Navigate to the **IP Addresses** tab:
   - **Address space:** `10.0.0.0/16`
   - Add **Subnet 1:** Name: `snet-web`, Range: `10.0.1.0/24`
   - Add **Subnet 2:** Name: `snet-db`, Range: `10.0.2.0/24`
4. Click **Review + create** → **Create**.

> 💡 The `/16` address space gives you 65,536 possible IP addresses. The two `/24` subnets each carve out 256 addresses for their tier — plenty of room to grow.

---

### Phase 2 — Deploy the Web Server (Public Tier)

1. Search for **Virtual Machines** → click **+ Create** → **Azure virtual machine**.
2. Fill in the **Basics** tab:
   - **Resource Group:** `rg-lab02-[yourname]`
   - **VM Name:** `vm-web-01`
   - **Region:** `East US`
   - **Image:** `Ubuntu Server 20.04 LTS`
   - **Size:** `Standard_B1s`
   - **Authentication:** SSH public key
   - **Key pair name:** `key-lab02`
   - **Public inbound ports:** Allow selected → check **HTTP (80)** and **SSH (22)**
3. Navigate to the **Networking** tab:
   - **Virtual Network:** `vnet-lab02`
   - **Subnet:** `snet-web`
   - **Public IP:** Create New (Standard)
4. Click **Review + create** → **Create**.
5. When prompted, **download the `.pem` private key file** and save it somewhere safe. You will need this to SSH in.

---

### Phase 3 — Deploy the Database Server (Private Tier)

1. Create another **Virtual Machine** with the following settings.
2. Fill in the **Basics** tab:
   - **Resource Group:** `rg-lab02-[yourname]`
   - **VM Name:** `vm-db-01`
   - **Region:** `East US`
   - **Image:** `Ubuntu Server 20.04 LTS`
   - **Size:** `Standard_B1s`
   - **Key pair name:** Use existing → `key-lab02`
   - **Public inbound ports:** Allow selected → **SSH (22)** *(temporary — we'll lock this down)*
3. Navigate to the **Networking** tab — **⚠️ Critical configuration below:**

   | Field | Value | Why |
   |---|---|---|
   | Virtual Network | `vnet-lab02` | Same VNet so both VMs can communicate internally |
   | Subnet | `snet-db` | Places the VM in the private tier |
   | Public IP | **None** | No public IP = no direct internet path to this VM |

4. Click **Review + create** → **Create**.

---

### Phase 4 — Validate Connectivity (The Jump Host Pattern)

Since `vm-db-01` has no public IP, you **cannot** SSH into it directly from your home machine. You must first land on the web server and then hop to the database server from there. This is called a **jump host** (or bastion host) pattern.

**Step 1 — Get the Private IP of the DB Server**

Navigate to `vm-db-01` in the portal. Under the **Overview** tab, locate **Private IP address**. It should be `10.0.2.4`. Copy it.

**Step 2 — SSH into the Web Server**

Set correct permissions on your key file, then connect:

```bash
# Set correct key permissions (required on macOS/Linux)
chmod 400 key-lab02.pem

# SSH into the Web Server using its Public IP
ssh -i key-lab02.pem azureuser@[PUBLIC-IP-OF-vm-web-01]
```

**Step 3 — Ping the Database Server from Inside the VNet**

Once connected to `vm-web-01`, run:

```bash
ping 10.0.2.4
```

**Expected Output:**

```
PING 10.0.2.4 (10.0.2.4) 56(84) bytes of data.
64 bytes from 10.0.2.4: icmp_seq=1 ttl=64 time=1.23 ms
64 bytes from 10.0.2.4: icmp_seq=2 ttl=64 time=0.98 ms
```

Press `Ctrl + C` to stop. ✅ Replies confirm both VMs are communicating inside the VNet.

---

### Phase 5 — Lock Down the Database with an NSG Rule

The ping works — but right now, the DB server's NSG is too permissive. We need to add an explicit rule that **only allows traffic from the web subnet** and blocks everything else.

1. Navigate to the **vm-db-01** resource in the portal.
2. In the left menu, click **Networking**.
3. Click on the **Network Security Group** name (e.g., `vm-db-01-nsg`).
4. Click **Inbound security rules** → **+ Add**.
5. Configure the rule as follows:

   | Field | Value |
   |---|---|
   | **Source** | IP Addresses |
   | **Source IP / CIDR** | `10.0.1.0/24` *(the web subnet)* |
   | **Source port ranges** | `*` |
   | **Destination** | Any |
   | **Service** | Custom |
   | **Destination port ranges** | `*` *(or `3306` for MySQL / `5432` for PostgreSQL)* |
   | **Action** | Allow |
   | **Priority** | `100` |
   | **Name** | `Allow-Web-Subnet` |

6. Click **Add**.

> 🛡️ **Why this matters:** Azure NSGs use a default-deny posture for rules not explicitly permitted. By setting source to `10.0.1.0/24`, you've created an allowlist. Only packets originating from the web subnet will be accepted — no other subnet, and certainly not the public internet, can reach this VM.

---

## 🔧 Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| `ping` from web VM returns no replies | `vm-db-01` deployed into the wrong subnet | Go to `vm-db-01` → Networking → confirm subnet shows `snet-db` |
| `ping` fails despite correct subnet | VMs in different VNets | Both VMs must be in `vnet-lab02`. Check the VNet field on each VM's Networking tab |
| Can't SSH into `vm-db-01` from local machine | No Public IP assigned | Expected — use the jump host pattern via `vm-web-01` |
| `Permission denied (publickey)` when SSH-ing | Key permissions too open | Run `chmod 400 key-lab02.pem` before connecting |
| `vm-web-01` SSH times out | Port 22 blocked by NSG | Confirm the web VM's NSG has an inbound rule allowing TCP port 22 |

---

## 🔒 Security Notes

This lab demonstrates several important security principles worth calling out explicitly:

**1. No Public IP = No Attack Surface**
Removing the public IP from `vm-db-01` eliminates the attack surface entirely. An attacker cannot target what they cannot reach. This is different from a firewall rule — there is simply no network path.

**2. Principle of Least Privilege (Network)**
The NSG rule on `vm-db-01` only permits the specific CIDR range of the web subnet (`10.0.1.0/24`). No other source — including other subnets you might add later — gets access unless explicitly permitted.

**3. The Jump Host Pattern**
`vm-web-01` acts as a controlled entry point to the private tier. In production, this role is filled by **Azure Bastion** (a managed, fully hardened jump host). Building it manually first gives you the intuition for why Bastion exists.

**4. Defense in Depth**
We have two overlapping controls protecting `vm-db-01`: no public IP (network-level) and a restrictive NSG (firewall-level). Either alone is good. Both together is better.

---

## 💡 Key Concepts Covered

| Concept | Definition |
|---|---|
| **IaaS** | Infrastructure as a Service — you manage the OS and above; Azure manages the physical hardware |
| **Virtual Network (VNet)** | An isolated, private network in Azure where your resources communicate |
| **Subnet** | A range of IP addresses within a VNet used to segment resources by tier or function |
| **Network Security Group (NSG)** | A stateful firewall attached to a subnet or NIC that filters traffic using allow/deny rules |
| **Jump Host** | A hardened server used as an intermediary to access servers in a private subnet |
| **Private IP** | An IP address reachable only within the VNet — not accessible from the internet |
| **Public IP** | An internet-routable IP address assigned to a resource for external access |
| **CIDR Notation** | A compact way to express IP address ranges (e.g., `10.0.1.0/24` = 256 addresses) |
| **Defense in Depth** | Layering multiple security controls so that no single failure exposes the system |

---

## 📁 Repository Structure

```
lab-02-azure-2tier-webapp/
│
├── README.md          # This file — full lab documentation
└── screenshots/       # Optional: add your validation screenshots here
    ├── vnet-config.png
    ├── vm-web-networking.png
    ├── vm-db-networking.png
    ├── ping-success.png
    └── nsg-rule.png
```

---

## 🧹 Clean Up Resources

> 💡 Deleting the Resource Group removes **all** resources inside it — both VMs, the VNet, NSGs, disks, and public IPs — in a single action.

1. Navigate to **Resource Groups** in the Azure Portal.
2. Click `rg-lab02-[yourname]`.
3. Click **Delete resource group**.
4. Type the resource group name to confirm.
5. Click **Delete**.

> ⚠️ VMs accrue compute charges while running. If you plan to return to the lab later, **Stop** (deallocate) the VMs rather than deleting them to pause billing.

---

## 🔗 References

- [Azure Virtual Network Documentation](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-overview)
- [Network Security Groups Overview](https://learn.microsoft.com/en-us/azure/virtual-network/network-security-groups-overview)
- [Azure Bastion (Production Jump Host)](https://learn.microsoft.com/en-us/azure/bastion/bastion-overview)
- [Linux VM SSH Key Authentication in Azure](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/create-ssh-keys-detailed)
- [Azure VM Sizes — B-Series Burstable](https://learn.microsoft.com/en-us/azure/virtual-machines/bv1-series)

---
