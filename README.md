# Hybrid Identity Lab — On-Prem AD to Microsoft Entra ID

Extends an existing on-premises Active Directory home lab into a hybrid identity environment by syncing on-prem users to Microsoft Entra ID (formerly Azure AD) using Microsoft Entra Connect.

**Companion repo:** this builds directly on [https://github.com/sibghatrehman/Active-Directory-Home-Lab]()`]— same domain controller, same OU structure, same users. This repo covers everything added on top: the sync server, the Entra ID tenant, and identity sync/verification.

## Overview

Most organizations don't move to the cloud all at once — they run hybrid, with an on-prem AD as the source of truth and Entra ID handling cloud sign-in. This lab reproduces that pattern on a single subnet: a dedicated sync server, no changes to the existing DC's role, and only a specific OU synced to the cloud rather than the whole directory.

| | |
|---|---|
| **On-prem domain** | `mydomain.com` (from the AD DS lab) |
| **Cloud tenant** | `sibghatsonugmail.onmicrosoft.com` — Entra ID Free tier |
| **Sync tool** | Microsoft Entra Connect (Password Hash Sync) |
| **Sync scope** | `OU=_USERS` only (IT / HR / Management) — `_ADMINS` intentionally excluded |
| **Sync server** | Domain-joined member server, internal network only |

## Architecture

- **SyncServer01** is a separate domain-joined member server — Entra Connect is deliberately *not* installed on the DC
- It sits on the same internal subnet as the DC and client (`172.16.0.0/24`), with a static IP so its address never changes
- Internet-bound traffic to Entra ID's endpoints (port 443) routes out through the DC's existing NAT — no second NIC needed
- Only the `_USERS` OU is in scope for sync; `_ADMINS` stays on-prem only, by design

## Build Steps

### 1. Sync Server Prep
- New VM, domain-joined to `mydomain.com`, Internal Network only (same NIC setup as the client)
- Static IP outside the DHCP scope (or a DHCP reservation) so the address is predictable
- Confirmed outbound HTTPS reaches Microsoft's endpoints through the DC's NAT before installing anything:
  ```powershell
  Test-NetConnection login.microsoftonline.com -Port 443
  ```

### 2. Entra ID Tenant
- Created a free Entra ID tenant (`mydomain.onmicrosoft.com`)
- Created a dedicated Hybrid Identity Administrator account for setup — not a personal/global admin used elsewhere

### 3. Install Microsoft Entra Connect
- Downloaded from the Entra admin center (Entra ID → Entra Connect → Connect Sync) — the standalone installer link has been retired; it's admin-center-only now
- Pre-registered MFA for the install account at **aka.ms/mfasetup** *before* running the wizard (see Troubleshooting — the wizard's embedded browser can't handle first-time MFA registration)
- Temporarily added the on-prem install account to **Enterprise Admins** (required for forest-level setup), removed it again once configuration completed
- Configured:
  - Sign-in method: **Password Hash Synchronization**
  - Scope: OU-based filtering → `_USERS` only
  - Accidental-delete protection: left at default threshold
- Ran initial sync

### 4. Verification
```powershell
Get-ADSyncScheduler                          # confirms scheduled delta sync is active
Start-ADSyncSyncCycle -PolicyType Delta      # manual trigger, for testing
```
- Confirmed synced users appear in the Entra admin center as "Synced from on-premises"
- Signed in as a synced user at **myaccount.microsoft.com** using their on-prem password — proves password hash sync actually authenticates, not just that the object exists
- Checked **zero sync errors** in the admin center (a sync run can report "success" while individual objects silently fail)
- Confirmed **Licenses → All products** shows only the free Entra ID plan — no paid features enabled

## Troubleshooting

**Issue 1 — "Update your browser" error during MFA setup in the wizard**
Misleading message. The wizard uses an embedded browser control that can't render the modern MFA *registration* page — it has nothing to do with the system's actual browser. Fix: register MFA for the account separately at `aka.ms/mfasetup` in a normal browser first, then the wizard only needs to handle an auth *challenge* (which it can do fine), not registration.

**Issue 2 — "User is not a member of Enterprise Admins"**
The wizard requires **Enterprise Admins**, a forest-wide group distinct from Domain Admins, because initial setup touches forest-level configuration. Fixed by temporarily adding the install account:
```powershell
Add-ADGroupMember -Identity "Enterprise Admins" -Members a-srehman
# ... run the wizard ...
Remove-ADGroupMember -Identity "Enterprise Admins" -Members a-srehman
```
Removed the membership again after setup completed — Enterprise Admins is only needed during install, and leaving standing membership violates least-privilege.

## Screenshots
  Screenshots are added in the REPO for the references and results. 

## Skills Demonstrated

`Microsoft Entra ID` · `Entra Connect / Hybrid Identity` · `Password Hash Synchronization` · `Active Directory (Enterprise Admins, forest-level administration)` · `PowerShell (ADSync module)` · `Least-Privilege Access Management` · `MFA Configuration` · `Troubleshooting`

## Resources Used

- Server Academy — *Syncing your On-Premise Users to Azure Active Directory* — primary walkthrough for the sync setup
- Microsoft Learn — Entra Connect documentation, used for prerequisites and the current (admin-center-only) download path
- Microsoft Q&A — community threads on the "update your browser" MFA error, used to confirm root cause during troubleshooting

