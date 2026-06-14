[README.md](https://github.com/user-attachments/files/28931524/README.md)
# Lab 01 — Active Directory & Identity Management

WATCH ME BUILD IT! https://www.loom.com/share/32130a203d8243b1be794fc8bf463514

**Platform:** Windows Server 2025 · Azure Free Account  
**Domain:** `lab.local`  
**Cert Alignment:** CompTIA Network+ · Security+ · AZ-104 (Azure Administrator)  
**Estimated Time:** 3–5 hours across multiple sessions  
**Estimated Cost:** $0 — fully covered by free tiers and evaluation licences

---

## Overview

This lab builds a functional Active Directory environment from the ground up — from a blank Windows Server VM to a fully managed domain with users, groups, Organisational Units, Group Policy enforcement, and common help desk operations.

Active Directory is the identity backbone of virtually every enterprise Windows environment. It answers one fundamental question: **who is allowed to do what?** Understanding how to build and manage it is foundational knowledge for IT Support, Sysadmin, Cloud Engineering, and Security Analysis roles.

> **Why this matters for cloud roles:** Microsoft Entra ID (formerly Azure AD) uses the same core concepts — users, groups, roles, conditional access. On-premises AD knowledge transfers directly to the cloud.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  ☁  Azure Free Tier                                          │
│                                                             │
│        ┌──────────────────────────────────────┐            │
│        │   Domain Controller (DC)             │            │
│        │   Windows Server 2025 · lab.local    │            │
│        │   AD DS · DNS · GPMC                 │            │
│        └──────────────────┬───────────────────┘            │
│                           │                                 │
│        ┌──────────────────▼───────────────────┐            │
│        │  Organisational Units — lab.local     │            │
│        │  ┌────────┐ ┌─────────┐ ┌────┐ ┌────┐│            │
│        │  │  IT    │ │ Finance │ │ HR │ │Sale││            │
│        │  │IT_Admin│ │Fin_Users│ │HR_U│ │S_U ││            │
│        │  │a.chen  │ │b.patel  │ │c.jo│ │d.sm││            │
│        │  │GPO ✓   │ │         │ │    │ │    ││            │
│        │  └────────┘ └─────────┘ └────┘ └────┘│            │
│        └──────────────────┬───────────────────┘            │
│                           │                                 │
│        ┌──────────────────▼───────────────────┐            │
│        │  Group Policy Objects (GPOs)          │            │
│        │  Password · Screen lock · USB block   │            │
│        └──────────────────┬───────────────────┘            │
│                           │                                 │
│        ┌──────────────────▼───────────────────┐            │
│        │  Domain-joined Workstation            │            │
│        │  Receives GPO · Authenticates via DC  │            │
│        └──────────────────────────────────────┘            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                               ▲
                   RDP (TCP 3389) from local machine
```

**Trust boundary:** All identity decisions flow through the Domain Controller. Every domain-joined machine trusts the DC to authenticate users and enforce policy. The only external access point is RDP on port 3389, used to manage the server from a local client.

---

## Career Relevance

| Role | How this lab applies |
|---|---|
| IT Support / Help Desk | Password resets, account unlocks, group membership changes — the top three ticket types in any enterprise |
| Sysadmin | OU design, GPO deployment, managing domain-joined machines at scale |
| Cloud Engineer | Entra ID uses the same concepts: users, groups, roles, conditional access. On-prem knowledge transfers directly |
| Security Analyst | AD is the most targeted system in ransomware attacks. Understanding how it works is the foundation of defending it |

---

## What This Lab Covers

| Skill | Real-world application |
|---|---|
| Promote a server to Domain Controller | The first step in every enterprise Windows environment |
| Create Organisational Units (OUs) | Organise users and computers by department; apply different policies to each |
| Create users, groups, and group memberships | Every access decision in an enterprise is group-based |
| Configure Group Policy Objects (GPOs) | Enforce password policy, screen lock, and USB restrictions centrally |
| Join a machine to the domain | Connect a workstation so it becomes a managed, policy-enforced resource |
| Configure role-based access with security groups | Apply the principle of least privilege in practice |
| Reset passwords and manage account lifecycle | The most frequent real-world task for IT support |

---

## Prerequisites

- An Azure Free Account (azure.microsoft.com/free) **or** a local machine with 8 GB RAM and VirtualBox installed
- A Remote Desktop (RDP) client on your local machine
- No prior Active Directory experience required

---

## Step 1 — Provision the VM

### Option A — Azure (Recommended)

Azure removes local hardware requirements. The VM runs in Microsoft's datacentre; you connect via RDP.

1. Go to [azure.microsoft.com/free](https://azure.microsoft.com/free) and create a free account
2. Sign in to [portal.azure.com](https://portal.azure.com)
3. Search for **Virtual machines** → **Create**
4. Use the settings below, then click **Review + Create → Create**

| Setting | Value | Reason |
|---|---|---|
| Region | West Europe | Selected for geographic proximity; pricing varies by region |
| Availability Zone | Zone 3 | Provides single-zone redundancy within the West Europe region |
| Image | Windows Server 2025 Datacenter — Gen2 | Latest server OS, includes 180-day evaluation licence |
| Size | Standard_D2s_v3 (2 vCPU, 8 GiB RAM) | More headroom than the minimum — AD DS ran comfortably with no resource pressure |
| Authentication | Password | Set a strong password — used to RDP in |
| Public inbound ports | Allow RDP (3389) | Required to connect from your local machine |
| OS disk | Standard SSD | Good performance, included in free-tier storage |

> **Cost note:** The Standard_D2s_v3 in West Europe costs approximately **$0.21/hour** while running. Stop the VM (do not delete it) at the end of every session — stopping pauses compute billing and preserves your configuration. Deleting the VM removes everything and you would need to rebuild. With $200 in free credits, budget roughly 950 hours of total runtime before credits are exhausted at this rate.

**Fix clipboard before connecting:**
By default, RDP does not share your clipboard — you will not be able to paste commands. Fix this before your first connection:

1. Open the Remote Desktop application on your local machine
2. Enter the VM's public IP address
3. Click **Show Options → Local Resources tab**
4. Check **Clipboard** under Local devices and resources
5. Click **Connect**

> If you are using the Azure portal browser console, download the RDP file instead (**Connect → Download RDP File**) and open it with the native Remote Desktop app. This is the recommended approach for all lab work.

### Option B — VirtualBox (Local)

| Requirement | Minimum |
|---|---|
| Host RAM | 8 GB (4 GB for VM, 4 GB for host OS) |
| Disk space | 60 GB free |
| CPU | Quad-core with virtualisation enabled in BIOS |

1. Download [VirtualBox](https://www.virtualbox.org) — free, no account required
2. Download the [Windows Server 2025 Evaluation ISO](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2025) from Microsoft Evaluation Center
3. Create a new VM: 4 GB RAM, 60 GB disk, type = Windows Server 2022
4. Mount the ISO and boot — follow the installation wizard
5. Select **Windows Server 2025 Datacenter with Desktop Experience** during setup

---

## Step 2 — Install Active Directory Domain Services

RDP into your Windows Server VM. Open **Server Manager** — it opens automatically on login.

> **What is a Domain Controller?**  
> A Domain Controller (DC) is the brain of the entire identity system. When a user logs in anywhere on the domain, credentials are checked against the DC. When you create a user account, you create it on the DC. Every machine that joins your network trusts this server to make authentication decisions.

**Via Server Manager:**
1. Click **Manage → Add Roles and Features**
2. Click **Next** until you reach **Server Roles**
3. Check **Active Directory Domain Services**
4. When prompted, click **Add Features** to include the management tools
5. Click **Next** through remaining pages → **Install**
6. Wait 2–3 minutes for installation → click **Close** (do not restart yet)

**Via PowerShell (alternative):**
```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
```

**Also install Group Policy Management Console (GPMC) now:**
```powershell
Install-WindowsFeature -Name GPMC
```

> **Why now?** Step 5 requires GPMC. Without it, the Group Policy Management option will not appear in Server Manager and you will hit a wall mid-lab. Once installed, close and reopen Server Manager — **Group Policy Management** will appear under the Tools menu as a separate window from Active Directory Users and Computers.

---

## Step 3 — Promote the Server to a Domain Controller

Installing the AD DS role does not create a domain. Promotion creates your forest, your domain, and makes this server the authoritative DNS and identity server.

> **What is a Forest and Domain?**  
> A **Forest** is the top-level container of your entire Active Directory structure — think of it as the organisation itself. A **Domain** is a boundary inside the forest with a name (ours is `lab.local`). Most small-to-medium organisations have one domain inside one forest.

**Via Server Manager:**
1. Click the **yellow warning flag** at the top right
2. Click **Promote this server to a domain controller**
3. Select **Add a new forest**
4. Set Root domain name to: `lab.local`
5. Set a **DSRM password** — write this down (needed for disaster recovery only)
6. Accept defaults on DNS Options and NetBIOS pages
7. Click **Install** — the server restarts automatically when complete

**Via PowerShell (alternative):**
```powershell
Import-Module ADDSDeployment
Install-ADDSForest `
  -DomainName 'lab.local' `
  -DomainNetBiosName 'LAB' `
  -InstallDns:$true `
  -SafeModeAdministratorPassword (ConvertTo-SecureString 'YourDSRMPassword!' -AsPlainText -Force) `
  -Force:$true
```

After the restart, you have created a new Active Directory forest called `lab.local`. This server is now the root Domain Controller — it runs DNS for the domain and is the authoritative source for all identity decisions.

---

## Step 4 — Build the Organisational Structure

Open **Active Directory Users and Computers (ADUC)** from the Tools menu in Server Manager.

### Organisational Units

> **What is an Organisational Unit (OU)?**  
> An OU is a folder inside Active Directory. You use OUs to organise users, computers, and groups by department or function. The real power of an OU is that you can link a Group Policy to it — every user or computer inside that OU automatically gets the policy applied.

```powershell
New-ADOrganizationalUnit -Name "IT"        -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Finance"   -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "HR"        -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Sales"     -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Computers" -Path "DC=lab,DC=local"
```

### Security Groups

> **What is a Security Group?**  
> Instead of granting access to resources for individual users one by one, you grant access to a group, then add users to that group. This is role-based access control (RBAC). When someone joins a department, add them to the group and they instantly inherit all access. When they leave, remove them and all access is revoked simultaneously.

```powershell
New-ADGroup -Name "IT_Admins"     -GroupScope Global -GroupCategory Security -Path "OU=IT,DC=lab,DC=local"
New-ADGroup -Name "Finance_Users" -GroupScope Global -GroupCategory Security -Path "OU=Finance,DC=lab,DC=local"
New-ADGroup -Name "HR_Users"      -GroupScope Global -GroupCategory Security -Path "OU=HR,DC=lab,DC=local"
New-ADGroup -Name "Sales_Users"   -GroupScope Global -GroupCategory Security -Path "OU=Sales,DC=lab,DC=local"
```

### User Accounts

> **Important:** Run the entire block below as one unit — do not run it line by line. The `$password` variable must be defined before the `New-ADUser` commands or PowerShell will fail. Select all, then paste the full block into PowerShell and press Enter.

```powershell
# Step 1 — define the password variable first
$password = ConvertTo-SecureString "Welcome@2026!" -AsPlainText -Force

# Step 2 — create all 4 users
New-ADUser -Name "alice.chen" -GivenName "Alice" -Surname "Chen" `
  -SamAccountName "alice.chen" -UserPrincipalName "alice.chen@lab.local" `
  -Path "OU=IT,DC=lab,DC=local" -AccountPassword $password -Enabled $true

New-ADUser -Name "bob.patel" -GivenName "Bob" -Surname "Patel" `
  -SamAccountName "bob.patel" -UserPrincipalName "bob.patel@lab.local" `
  -Path "OU=Finance,DC=lab,DC=local" -AccountPassword $password -Enabled $true

New-ADUser -Name "carol.jones" -GivenName "Carol" -Surname "Jones" `
  -SamAccountName "carol.jones" -UserPrincipalName "carol.jones@lab.local" `
  -Path "OU=HR,DC=lab,DC=local" -AccountPassword $password -Enabled $true

New-ADUser -Name "david.smith" -GivenName "David" -Surname "Smith" `
  -SamAccountName "david.smith" -UserPrincipalName "david.smith@lab.local" `
  -Path "OU=Sales,DC=lab,DC=local" -AccountPassword $password -Enabled $true

# Step 3 — add each user to their department group
Add-ADGroupMember -Identity "IT_Admins"     -Members "alice.chen"
Add-ADGroupMember -Identity "Finance_Users" -Members "bob.patel"
Add-ADGroupMember -Identity "HR_Users"      -Members "carol.jones"
Add-ADGroupMember -Identity "Sales_Users"   -Members "david.smith"
```

---

## Step 5 — Configure Group Policy

Open **Group Policy Management** from the Tools menu in Server Manager (separate from ADUC).

> **What is a Group Policy Object (GPO)?**  
> A GPO is a collection of settings applied automatically to every user or computer inside an OU. You create one GPO, link it to an OU, and every machine and user in that OU immediately gets those rules applied the next time they log in or run `gpupdate`. This is how organisations manage thousands of machines consistently without touching each one.

**Create and link the GPO:**
1. Expand **Forest: lab.local → Domains → lab.local**
2. Right-click the **IT OU** → **Create a GPO in this domain and link it here**
3. Name it: `IT Security Policy`
4. Right-click the new GPO → **Edit**
5. Configure the settings below

| Policy path | Setting | Value | Why |
|---|---|---|---|
| Computer Config → Windows Settings → Security → Account Policies → Password Policy | Minimum password length | 12 | Enforces strong passwords across all IT accounts |
| Computer Config → Windows Settings → Security → Account Policies → Password Policy | Password must meet complexity requirements | Enabled | Requires upper, lower, number, and symbol |
| Computer Config → Windows Settings → Security → Local Policies → Security Options | Interactive logon: Machine inactivity limit | 900 seconds | Auto-locks screen after 15 minutes |
| Computer Config → Administrative Templates → System → Removable Storage Access | All removable storage classes: Deny all access | Enabled | Prevents data exfiltration via USB drives |

**Test the GPO:**
Join a second VM to `lab.local`, move its computer account into the IT OU, run `gpupdate /force`, log in as `alice.chen`, and verify the screen lock policy takes effect.

---

## Step 6 — Common Help Desk Operations

These are the most frequent real-world tasks expected from day one in any IT support role.

### Reset a password

```powershell
Set-ADAccountPassword -Identity "bob.patel" -Reset `
  -NewPassword (ConvertTo-SecureString "NewPass@2026!" -AsPlainText -Force)
Set-ADUser -Identity "bob.patel" -ChangePasswordAtLogon $true
```

### Unlock a locked account

```powershell
Unlock-ADAccount -Identity "carol.jones"
```

### Disable an account (offboarding)

Disable rather than delete — disabling preserves account history and group memberships for audit purposes.

```powershell
# Disable the account
Disable-ADAccount -Identity "david.smith"

# Find all currently disabled accounts
Search-ADAccount -AccountDisabled | Select-Object Name, SamAccountName
```

### Audit and reporting

```powershell
# Find accounts that have not logged in for 90 days
$cutoff = (Get-Date).AddDays(-90)
Get-ADUser -Filter {LastLogonDate -lt $cutoff -and Enabled -eq $true} `
  -Properties LastLogonDate | Select-Object Name, LastLogonDate

# Check group membership for a specific user
Get-ADPrincipalGroupMembership -Identity "alice.chen" | Select-Object Name
```

---

## Verification Checks

Run these commands to confirm the lab is working correctly.

| Check | Command | Expected result |
|---|---|---|
| Domain Controller is running | `Get-ADDomainController` | Returns DC info including forest `lab.local` |
| OUs exist | `Get-ADOrganizationalUnit -Filter *` | Lists all 5 OUs created |
| Users exist and are enabled | `Get-ADUser -Filter {Enabled -eq $true}` | Lists the 4 test accounts |
| Group memberships correct | `Get-ADGroupMember -Identity IT_Admins` | Returns `alice.chen` |
| GPO is linked | `Get-GPInheritance -Target 'OU=IT,DC=lab,DC=local'` | Shows `IT Security Policy` as linked |

---

## Troubleshooting

| Problem | Fix |
|---|---|
| PowerShell prompts for `Name:` when creating users | You ran `New-ADUser` before defining `$password`. Run the entire block from Step 4 at once — the `$password` line must come first. |
| Cannot copy and paste into the VM | Open the RDP client → **Show Options → Local Resources** → check **Clipboard**. Reconnect. Or download the RDP file from Azure portal and use the native Remote Desktop app. |
| Promotion fails: DNS conflict | Set the NIC's preferred DNS to `127.0.0.1` before promoting, or use the VM's static IP |
| Cannot RDP after domain join | Log in as `LAB\Administrator` (domain admin), not just `Administrator` |
| GPO not applying | Run `gpupdate /force` on the target machine, then `gpresult /r` to see applied policies |
| User cannot log in after creation | Confirm the account is **Enabled** and `ChangePasswordAtLogon` is set |
| AD Users and Computers not showing | Run `dsa.msc` from the Run dialog, or run `Add-WindowsFeature RSAT-ADDS` |

---

## Key Concepts Reference

| Term | Definition |
|---|---|
| Domain Controller (DC) | Server running Active Directory; the authoritative source for all authentication in the domain |
| Forest | Top-level container of the entire AD structure — represents the organisation |
| Domain | A boundary inside the forest with its own name (e.g. `lab.local`) |
| Organisational Unit (OU) | A folder inside AD used to organise objects and apply Group Policy |
| Security Group | A container for user accounts used to grant access to resources at scale |
| Group Policy Object (GPO) | A collection of settings enforced automatically across all machines and users in an OU |
| DSRM | Directory Services Restore Mode — emergency recovery password for the DC |
| RBAC | Role-Based Access Control — granting access via group membership rather than individual permissions |

---

## Resources

- [Microsoft Learn — Active Directory Domain Services](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/virtual-dc/active-directory-domain-services-overview)
- [Microsoft Learn — Group Policy Overview](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/hh831791(v=ws.11))
- [Azure Free Account](https://azure.microsoft.com/free)
- [Windows Server 2025 Evaluation](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2025)

---

*Lab 01 of an ongoing series covering enterprise IT infrastructure, cloud security, and identity management.*
