# Lab 02: OU Structure · Users · Groups · RBAC

![Platform](https://img.shields.io/badge/Platform-Windows%20Server%202025-blue?style=flat-square)
![Duration](https://img.shields.io/badge/Duration-45--90%20min-orange?style=flat-square)
![Cost](https://img.shields.io/badge/Estimated%20Cost-%240-brightgreen?style=flat-square)

---

Video Lab Link: https://www.loom.com/share/f8f4198ae1e24b6eadbc4980f2697c4d

Fixing an Issue in the lab video link: https://www.loom.com/share/b2716139ca1d467e9c1701647b7b2fa6

---
 
## Overview

These steps are where Active Directory starts to look and behave like a real enterprise environment. You will build the organisational structure that determines how policies are applied (OUs), then populate it with the identity objects that control who gets access to what (users and security groups).
The design principle here is the same one used in every well-run enterprise environment: access is always assigned to groups, never to individual users. A user gets access to something by being a member of the right group. When they leave, you remove them from groups — every door closes at once.

## Architecture
 
```
                      lab.local domain
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   ┌─────────────┐  ┌─────────────┐  ┌───────────────┐  │
│   │     IT OU   │  │    HR OU    │  │  Finance OU   │  │
│   │─────────────│  │─────────────│  │───────────────│  │
│   │ IT-Admins   │  │  HR-Staff   │  │ Finance-Staff │  │
│   │ (group)     │  │  (group)    │  │ (group)       │  │
│   │─────────────│  │─────────────│  │───────────────│  │
│   │ ajohnson    │  │ jsmith      │  │ mwilliams     │  │
│   │ (user)      │  │ (user)      │  │ (user)        │  │
│   └─────────────┘  └─────────────┘  └───────────────┘  │
│                                                         │
│   ┌──────────────────┐   ┌──────────────────────────┐   │
│   │  Workstations OU │   │  Service Accounts OU     │   │
│   │  Computer objects│   │  Non-interactive accounts│   │
│   └──────────────────┘   └──────────────────────────┘   │
└─────────────────────────────────────────────────────────┘


User → Added to group → Group granted access to resource
        (this is RBAC — every access decision follows this chain)
```
 
> **Why this matters for security:** When a user leaves the organisation, disabling their AD account removes their ability to authenticate anywhere on the domain. When you remove them from their security groups, every resource that group had access to is also revoked — in a single operation. This is why group-based access management is not just a best practice, it's the only approach that scales.
 
---
 
## Step 4 — Create the OU Structure
 
OUs are the organisational containers of Active Directory. They serve two purposes: they keep objects organised, and they are the scope of Group Policy application. A GPO linked to the `Finance` OU applies to every user and computer object inside that OU, and nothing outside it.
 
> **Design principle:** Model your OUs after how you manage people, not just how the org chart looks. If IT admins need to apply different policies to the Finance department than to HR, those departments need to be in separate OUs so you can link different GPOs to each.
 
### Create OUs via Active Directory Users and Computers
 
1. Open **Server Manager → Tools → Active Directory Users and Computers**
2. Expand `lab.local` in the left pane
3. Right-click `lab.local` → **New → Organizational Unit**
4. Create each OU in the table below
| OU Name | Purpose |
|---|---|
| `IT` | IT staff with elevated permissions and stricter auditing policy |
| `HR` | HR staff with access to sensitive user data |
| `Finance` | Finance team with restricted internet and locked-down workstations |
| `Workstations` | All domain-joined computer objects |
| `Service Accounts` | Non-interactive accounts used by applications and services |
 
### Create OUs via PowerShell
 
```powershell
$domain = "DC=lab,DC=local"
 
New-ADOrganizationalUnit -Name "IT"               -Path $domain
New-ADOrganizationalUnit -Name "HR"               -Path $domain
New-ADOrganizationalUnit -Name "Finance"          -Path $domain
New-ADOrganizationalUnit -Name "Workstations"     -Path $domain
New-ADOrganizationalUnit -Name "Service Accounts" -Path $domain
```
 
### Verify OUs were created
 
```powershell
Get-ADOrganizationalUnit -Filter * | Select Name, DistinguishedName
```
 
You should see all five OUs listed with their full distinguished names, for example: `OU=IT,DC=lab,DC=local`.
 
---
 
## Step 5 — Create Users, Groups, and RBAC Memberships
 
### 5a — Create Security Groups
 
Security groups are the mechanism that makes RBAC work. You create one group per role or department, grant permissions to the group, then add the right users to that group.
 
```powershell
New-ADGroup `
  -Name "IT-Admins" `
  -GroupScope Global `
  -GroupCategory Security `
  -Path "OU=IT,DC=lab,DC=local"
 
New-ADGroup `
  -Name "HR-Staff" `
  -GroupScope Global `
  -GroupCategory Security `
  -Path "OU=HR,DC=lab,DC=local"
 
New-ADGroup `
  -Name "Finance-Staff" `
  -GroupScope Global `
  -GroupCategory Security `
  -Path "OU=Finance,DC=lab,DC=local"
```
 
> **GroupScope explained:** `Global` scope means the group can contain users from the same domain and be used to grant permissions to resources in any domain in the forest. This is the standard scope for user-population groups in enterprise environments.
 
---
 
### 5b — Create User Accounts
 
Create one user per OU to simulate a populated environment.
 
```powershell
# IT OU user
New-ADUser `
  -Name "Alex Johnson" `
  -GivenName "Alex" `
  -Surname "Johnson" `
  -SamAccountName "ajohnson" `
  -UserPrincipalName "ajohnson@lab.local" `
  -Path "OU=IT,DC=lab,DC=local" `
  -AccountPassword (ConvertTo-SecureString "P@ssw0rd123!" -AsPlainText -Force) `
  -Enabled $true
 
# HR OU user
New-ADUser `
  -Name "Jane Smith" `
  -GivenName "Jane" `
  -Surname "Smith" `
  -SamAccountName "jsmith" `
  -UserPrincipalName "jsmith@lab.local" `
  -Path "OU=HR,DC=lab,DC=local" `
  -AccountPassword (ConvertTo-SecureString "P@ssw0rd123!" -AsPlainText -Force) `
  -Enabled $true
 
# Finance OU user
New-ADUser `
  -Name "Mark Williams" `
  -GivenName "Mark" `
  -Surname "Williams" `
  -SamAccountName "mwilliams" `
  -UserPrincipalName "mwilliams@lab.local" `
  -Path "OU=Finance,DC=lab,DC=local" `
  -AccountPassword (ConvertTo-SecureString "P@ssw0rd123!" -AsPlainText -Force) `
  -Enabled $true
```
 
---
 
### 5c — Add Users to Their Security Groups
 
```powershell
Add-ADGroupMember -Identity "IT-Admins"     -Members "ajohnson"
Add-ADGroupMember -Identity "HR-Staff"      -Members "jsmith"
Add-ADGroupMember -Identity "Finance-Staff" -Members "mwilliams"
```
 
### Verify group memberships
 
```powershell
Get-ADGroupMember -Identity "IT-Admins"     | Select Name, SamAccountName
Get-ADGroupMember -Identity "HR-Staff"      | Select Name, SamAccountName
Get-ADGroupMember -Identity "Finance-Staff" | Select Name, SamAccountName
```
 
---
 
## Account Lifecycle Management
 
These are the most common support tasks in any enterprise environment. Learn to do them correctly from the start.
 
### Reset a password
 
```powershell
Set-ADAccountPassword `
  -Identity "ajohnson" `
  -Reset `
  -NewPassword (ConvertTo-SecureString "NewP@ssw0rd!" -AsPlainText -Force)
 
Set-ADUser -Identity "ajohnson" -ChangePasswordAtLogon $true
```
 
### Unlock a locked account
 
```powershell
Unlock-ADAccount -Identity "ajohnson"
```
 
### Disable an account (offboarding)
 
```powershell
Disable-ADAccount -Identity "mwilliams"
 
Move-ADObject `
  -Identity (Get-ADUser "mwilliams").DistinguishedName `
  -TargetPath "OU=Disabled,DC=lab,DC=local"
```
 
> **Why disable instead of delete?** Deleted AD accounts cannot be restored without a backup or tombstone reactivation. Disabling preserves the account and its group membership history for audit, legal hold, and access review purposes. Most organisations retain disabled accounts for 30–90 days before permanent deletion.
 
---
 
## Verification Checklist
 
```powershell
# Check all users exist and are enabled
Get-ADUser -Filter * -SearchBase "DC=lab,DC=local" | Select Name, Enabled, DistinguishedName
 
# Confirm group memberships
Get-ADUser "ajohnson" -Properties MemberOf | Select -ExpandProperty MemberOf
 
# Count objects per OU
Get-ADObject -Filter * -SearchBase "OU=IT,DC=lab,DC=local" | Measure-Object
```
 
---
 
## Troubleshooting
 
| Issue | Cause | Fix |
|---|---|---|
| `New-ADUser` fails with path error | OU not created yet | Run Step 4 OU creation first, verify with `Get-ADOrganizationalUnit` |
| User created but cannot log in | Account not enabled | Add `-Enabled $true` to the `New-ADUser` command or run `Enable-ADAccount` |
| Password rejected during user creation | Does not meet complexity requirements | Password must include uppercase, lowercase, number, and special character |
| Cannot find user in ADUC | Created in wrong OU | Check `DistinguishedName` with `Get-ADUser`, use `Move-ADObject` to correct |
| Group membership not reflecting | Replication or cache delay | Run `gpupdate /force` on the affected machine and re-test |
 
---
 
## Key Concepts
 
| Term | Definition |
|---|---|
| Organisational Unit (OU) | Container inside a domain used to organise objects and scope Group Policies |
| Security Group | AD object used to grant permissions and apply policies to a set of users |
| GroupScope Global | Group can contain users from the same domain, usable across the forest |
| SamAccountName | Legacy logon name (`ajohnson`) — used for domain logon as `LAB\ajohnson` |
| UserPrincipalName (UPN) | Modern email-format logon (`ajohnson@lab.local`) — used for Entra ID sync |
| Distinguished Name (DN) | Full path to an AD object (e.g. `CN=Alex Johnson,OU=IT,DC=lab,DC=local`) |
| RBAC | Role-Based Access Control — permissions tied to roles (groups), not individuals |
