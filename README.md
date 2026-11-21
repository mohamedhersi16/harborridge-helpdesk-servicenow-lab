# Harbor Ridge Logistics – Help Desk & Identity Lab

A hands-on help desk lab built from scratch in Hyper-V. I set up a Windows Server domain controller and a Windows 11 workstation for a fictional trucking and warehouse company, connected a ServiceNow instance as the ticketing system, and worked six realistic tickets from intake to resolution.

The tickets cover the core of entry-level help desk and identity work: account lockouts, password resets, new hires, department transfers, access requests, and terminations. Every ticket follows the same flow a real service desk uses: log it, verify the request, do the work, test it, document it, resolve it.

All work was done in the GUI first to build a solid understanding of each tool. PowerShell equivalents are noted where useful.

---

## Environment

| Component | Details |
|---|---|
| Hypervisor | Hyper-V on Windows 11 Pro (Dell OptiPlex, 16 GB RAM) |
| Lab network | Internal virtual switch `HarborRidge-LAN` (no internet access) |
| Domain controller | `HRL-DC01` – Windows Server 2022, AD DS + DNS, `192.168.50.10` |
| Workstation | `HRL-WS01` – Windows 11 Enterprise, `192.168.50.20`, joined to the domain |
| Domain | `harborridge.internal` |
| Ticketing system | ServiceNow Personal Developer Instance |

### Directory layout

```
harborridge.internal
└── Harbor Ridge
    ├── Staff
    │   ├── Dispatch
    │   ├── Finance
    │   ├── HR
    │   ├── IT
    │   └── Warehouse
    ├── Groups
    ├── Workstations
    └── Disabled Users
```

### Employees

| Name | Username | Department | Title |
|---|---|---|---|
| Tanya Morales | tmorales | Dispatch | Dispatcher |
| Derek Shaw | dshaw | Dispatch | Dispatch Supervisor |
| Luis Ortega | lortega | Warehouse → Dispatch | Warehouse Associate → Dispatcher |
| Kevin Pratt | kpratt | Warehouse | Forklift Operator (terminated in Ticket 6) |
| Rachel Kim | rkim | Finance | Finance Manager |
| Brenda Lyons | blyons | HR | HR Manager |
| Jalen Brooks | jbrooks | IT | Help Desk Technician |
| Nadia Farah | nfarah | IT | Systems Administrator |
| Omar Bennett | obennett | Warehouse | Warehouse Associate (new hire in Ticket 3) |

### Security groups

| Group | Purpose |
|---|---|
| SG-HelpDesk | Help desk team, with delegated reset and unlock rights on staff OUs |
| SG-Dispatch, SG-Warehouse, SG-Finance, SG-HR, SG-IT | Department access |
| SG-PayrollReports-Read | Read-only access to the Finance Payroll Reports share |

---

## Part 1 – Building the environment

### Domain controller

Installed Windows Server 2022, renamed the server, added the AD DS role, and promoted it to a new forest, `harborridge.internal`. Verified the domain in Server Manager, Active Directory Users and Computers, and DNS Manager, then took a clean checkpoint.

![Server edition](HelpDesk/03-server-edition.png)
![Server Manager](HelpDesk/04-server-manager.png)
![Server renamed](HelpDesk/06-renamed-server.png)
![AD DS role selected](HelpDesk/07-adds-role-selected.png)
![AD DS installed](HelpDesk/08-adds-installed.png)
![DC promotion](HelpDesk/09-dc-promotion-installing.png)
![Domain confirmed](HelpDesk/10-domain-joined.png)
![ADUC](HelpDesk/11-aduc-domain.png)
![DNS zone](HelpDesk/12-dns-zone.png)

### Departments, users, and groups

Built the OU structure to match the org chart, created each employee in their department OU with title and department filled in, and created security groups. Members were added by searching names with Check Names rather than typing exact usernames, the same way you would on the job.

![OU structure](HelpDesk/13-ou-structure.png)
![New user wizard](HelpDesk/14-new-user-wizard.png)
![Organization tab](HelpDesk/15-user-organization-tab.png)
![Dispatch users](HelpDesk/16-dispatch-users.png)
![Groups created](HelpDesk/17-groups-created.png)
![Check Names](HelpDesk/18-check-names.png)
![SG-IT members](HelpDesk/19-sg-it-members.png)

### Help desk delegation

Used the Delegation of Control wizard with a custom task to give SG-HelpDesk only the rights needed for daily account support on user objects in staff OUs: reset password, read/write lockoutTime (unlock), and read/write pwdLastSet (force password change). The built-in "reset password" task was skipped because it does not include unlocking accounts.

![Delegation summary](HelpDesk/21-delegation-summary.png)

### Workstation

Created HRL-WS01 as a Generation 2 VM with TPM enabled, installed Windows 11 with a local setup account, gave it a static IP with the DC as its DNS server, confirmed DNS resolution with nslookup, renamed it, and joined it to the domain. Moved the computer object out of the default Computers container into the Workstations OU so future policies apply to it, then confirmed a domain sign-in.

![Client TPM](HelpDesk/22-client-tpm.png)
![Static IP and DNS](HelpDesk/24-ws01-static-ip-dns.png)
![nslookup](HelpDesk/25-ws01-nslookup-dc.png)
![Domain join](HelpDesk/26-ws01-domain-join.png)
![Computer object](HelpDesk/27-dc-ws01-computer-object.png)
![Domain sign-in](HelpDesk/28-ws01-domain-signin.png)
![Workstations OU](HelpDesk/30-ws01-in-workstations-ou.png)

### ServiceNow setup

Created an `HRL Service Desk` assignment group, added myself as a member, and gave the group the `itil` role so members can be assigned tickets. Created the Harbor Ridge employees as ServiceNow users with user IDs matching their AD usernames, and saved a filtered queue for the team.

![HRL Service Desk group](HelpDesk/23-servicenow-hrl-group.png)
![New ServiceNow user](HelpDesk/31-servicenow-new-user.png)
![Harbor Ridge users](HelpDesk/32-servicenow-hrl-users.png)
![itil role on group](HelpDesk/36-hrl-group-itil-role.png)

---

## Part 2 – The tickets

| # | Ticket | Caller | Channel | Priority | Skill shown |
|---|---|---|---|---|---|
| 1 | INC0010001 – Account locked out | Tanya Morales | Phone | 3 - Moderate | Lockout policy, unlock, user verification |
| 2 | INC0010003 – Forgotten password | Derek Shaw | Phone | 4 - Low | Identity verification, secure reset |
| 3 | INC0010004 – New hire account | Brenda Lyons (HR) | Email | 4 - Low | Joiner process, approval before access |
| 4 | INC0010005 – Department transfer | Derek Shaw | Email | 4 - Low | Mover process, preventing privilege creep |
| 5 | INC0010006 – Shared folder access | Brenda Lyons | Self-service portal | 4 - Low | Owner approval, group-based least privilege |
| 6 | INC0010007 – Employee termination | Brenda Lyons (HR) | Phone | 3 - Moderate | Leaver process, offboarding across systems |

### Ticket 1 – Account lockout

**Scenario:** Tanya in Dispatch is locked out of her workstation after too many wrong passwords.

**Prep:** Set an account lockout threshold of 5 attempts in the Default Domain Policy. Lockout and password policies for domain accounts only take effect at the domain level.

**Work:**
1. Reproduced the lockout on HRL-WS01 and confirmed it on the DC.
2. Logged the ticket, assigned it to the HRL Service Desk, and noted that the caller's identity was verified.
3. Found the user with Find in ADUC instead of browsing folders, and unlocked the account. Unlocked only, with no password reset, since she knew her password.
4. Confirmed with the user: she signed in on HRL-WS01 and `whoami` returned `harborridge\tmorales`.
5. Resolved with a root cause and a prevention tip in the resolution notes.

PowerShell equivalent: `Unlock-ADAccount -Identity tmorales` and `Search-ADAccount -LockedOut`

![Lockout policy](HelpDesk/33-lockout-policy.png)
![Locked out on WS01](HelpDesk/34a-tanya-locked-out.png)
![Locked out in AD](HelpDesk/34b-dc-tanya-locked.png)
![Ticket logged](HelpDesk/35-ticket1-logged.png)
![Ticket in progress](HelpDesk/37-ticket1-in-progress.png)
![Unlock](HelpDesk/38-tanya-unlock.png)
![Signed in](HelpDesk/39-tanya-signed-in.png)
![Resolution](HelpDesk/40-ticket1-resolution.png)

### Ticket 2 – Forgotten password

**Scenario:** Derek returns from leave and can't remember his password. His account is not locked.

**Work:**
1. Logged the ticket and documented identity verification by calling back the number on his HR record, not a number the caller provided. Password reset calls are a common social engineering target.
2. Confirmed the account was not locked, then reset the password with "User must change password at next logon" so only the user knows the final password.
3. Confirmed he could sign in and was prompted to set his own password.

PowerShell equivalent:
```powershell
Set-ADAccountPassword -Identity dshaw -Reset -NewPassword (Read-Host -AsSecureString "New password")
Set-ADUser -Identity dshaw -ChangePasswordAtLogon $true
```

![Ticket logged](HelpDesk/41-ticket2-logged.png)
![Password reset](HelpDesk/42-derek-reset-password.png)
![Signed in](HelpDesk/43-derek-signed-in.png)
![Resolution](HelpDesk/44-ticket2-resolution.png)

### Ticket 3 – New hire

**Scenario:** HR requests an account for Omar Bennett, a new Warehouse Associate starting Monday.

**Work:**
1. Logged the request with HR as the caller and their approval documented before any account was created.
2. Created the account in the Warehouse OU, set title and department, and added him to SG-Warehouse so he can reach department resources on day one.
3. Verified with `net user obennett /domain` instead of signing in as him, which would have used up his first-logon password prompt.
4. Created the matching ServiceNow user so he can submit tickets.
5. Noted that the temporary password goes to the hiring manager by phone, not email.

Note: In a production ServiceNow environment, onboarding would normally be a Service Request through the catalog (ServiceNow even suggested the New Hire Order Guide while I logged this ticket). I used incidents for every ticket to keep the lab simple.

![Ticket logged](HelpDesk/45-ticket3-logged.png)
![New user](HelpDesk/46-omar-new-user.png)
![Group membership](HelpDesk/47-omar-memberof.png)
![Verified](HelpDesk/48-omar-verified.png)
![Resolution](HelpDesk/49-ticket3-resolved.png)

### Ticket 4 – Department transfer

**Scenario:** Luis is promoted from Warehouse Associate to Dispatcher. His new manager requests the change.

**Work:**
1. Captured his group membership before the change as evidence.
2. Removed SG-Warehouse and added SG-Dispatch, so he gains new access and loses old access in the same change. Skipping the removal step is how privilege creep builds up over time.
3. Updated his title, department, and manager.
4. Moved his account to the Dispatch OU.
5. Verified with `net user lortega /domain`: Dispatch access present, Warehouse access gone.

PowerShell equivalent:
```powershell
Remove-ADGroupMember SG-Warehouse -Members lortega -Confirm:$false
Add-ADGroupMember SG-Dispatch -Members lortega
Set-ADUser lortega -Title "Dispatcher" -Department "Dispatch" -Manager dshaw
Get-ADUser lortega | Move-ADObject -TargetPath "OU=Dispatch,OU=Staff,OU=Harbor Ridge,DC=harborridge,DC=internal"
```

![Ticket logged](HelpDesk/50-ticket4-logged.png)
![Groups before](HelpDesk/51-luis-groups-before.png)
![Groups after](HelpDesk/52-luis-groups-after.png)
![Moved to Dispatch](HelpDesk/53-luis-in-dispatch.png)
![Verified](HelpDesk/54-luis-groups-verified.png)
![Resolution](HelpDesk/55-ticket4-resolution.png)

### Ticket 5 – Shared folder access

**Scenario:** Brenda in HR needs read-only access to the Finance Payroll Reports folder for an audit.

**Prep:** Created `C:\Shares\Payroll Reports`, disabled inheritance and removed the domain-wide Users entries, then granted SG-Finance Modify and a new resource group, SG-PayrollReports-Read, Read & execute. Shared the folder with Authenticated Users at the share level and kept the real access control in NTFS.

Note: In this lab the DC doubles as the file server. In production, shares would live on a separate file server.

**Work:**
1. Brenda submitted the request herself through the ServiceNow Service Portal (simulated with impersonation).
2. Triaged the unassigned ticket: set category, routed it to the HRL Service Desk, and cleaned up the short description.
3. Requested and documented approval from the folder owner, Rachel Kim, before granting anything.
4. Added Brenda to SG-PayrollReports-Read.
5. Verified with the folder's Effective Access tab: read allowed, write and delete denied.
6. Resolution notes state that the access is group-based so it can be removed when the audit ends.

![Read group](HelpDesk/56-payroll-read-group.png)
![NTFS permissions](HelpDesk/57-payroll-ntfs-permissions.png)
![Share permissions](HelpDesk/58-payroll-share-permissions.png)
![Portal as Brenda](HelpDesk/59a-portal-as-brenda.png)
![Portal request](HelpDesk/59b-ticket5-portal-request.png)
![Owner approval](HelpDesk/60-ticket5-approval.png)
![Added to group](HelpDesk/61-brenda-added-to-group.png)
![Effective access](HelpDesk/62-brenda-effective-access.png)
![Resolution](HelpDesk/63-ticket5-resolution.png)

### Ticket 6 – Employee termination

**Scenario:** HR reports that Kevin Pratt is terminated effective today and all access must be removed.

**Work:**
1. Logged with high urgency, since active access for a terminated employee is a real insider risk.
2. Disabled the account first to cut access immediately.
3. Reset the password to a random value in case the account is ever re-enabled by mistake.
4. Captured his groups, then removed department access. Domain Users stays because it is the required primary group.
5. Added a termination note with the ticket number to the account description.
6. Moved the account to the Disabled Users OU. The account is disabled, not deleted, so files and audit history are kept for the retention period.
7. Deactivated his ServiceNow user record, since offboarding covers every system, not just AD.
8. Verified with `net user kpratt /domain`: account inactive, only Domain Users remaining.

PowerShell equivalent:
```powershell
Disable-ADAccount kpratt
Remove-ADGroupMember SG-Warehouse -Members kpratt -Confirm:$false
Set-ADUser kpratt -Description "Terminated 11/20/2026 - INC0010007"
Get-ADUser kpratt | Move-ADObject -TargetPath "OU=Disabled Users,OU=Harbor Ridge,DC=harborridge,DC=internal"
```

![Ticket logged](HelpDesk/64-ticket6-logged.png)
![Groups before](HelpDesk/65-kevin-groups-before.png)
![Disabled Users OU](HelpDesk/66-kevin-disabled-users.png)
![Verified](HelpDesk/67-kevin-verified-disabled.png)
![Resolution](HelpDesk/68-ticket6-resolution.png)

### All tickets resolved

![All tickets resolved](HelpDesk/69-hrl-all-tickets-resolved.png)

---

## Troubleshooting along the way

Real problems I ran into and how I solved them:

- **VM would not boot from the install ISO.** The "press any key to boot from CD" prompt only lasts a few seconds. Fixed by opening the VM window first, then resetting and pressing Enter right away.
- **"Not enough memory" when starting the workstation.** The VM needed 4 GB free all at once next to the DC and browser. Fixed by capping Dynamic Memory and lowering startup RAM to 2 GB after Windows was installed.
- **Accidentally reverted the workstation to an automatic checkpoint taken before Windows was installed.** Rebuilt and rejoined it. The DC, users, groups, and policies were unaffected. Turned off automatic checkpoints and switched to named manual checkpoints.
- **Windows 11 setup required internet.** The lab network is internal only, so I used a local account during setup and joined the domain afterward.
- **Domain user could not sign in through the VM window.** Hyper-V Enhanced Session uses Remote Desktop, and standard users are not in Remote Desktop Users. Switched to Basic Session for a console sign-in.
- **Could not assign tickets to myself.** The Assigned to field only lists group members who have the itil role. Added the role to the HRL Service Desk group.
- **A user account ended up at the domain root instead of an OU during a move.** Found it with Advanced Features turned on and moved it to the correct OU. An account outside its OU misses that OU's delegation and policies.
- **Could not remove the Users entries from the payroll folder.** They were inherited from C:\. Disabled inheritance and converted the entries to explicit permissions first.
- **Self-service tickets did not show in the team queue.** Portal users cannot pick an assignment group, so their tickets arrive unassigned and need triage first.

---

## Skills demonstrated

- Active Directory: OUs, users, security groups, delegation of control, Group Policy (account lockout)
- Identity lifecycle: joiner, mover, and leaver processes
- Least privilege: group-based access, resource groups, owner approval, removing old access
- NTFS and share permissions, inheritance, Effective Access
- Windows networking: static IP, DNS, nslookup, domain join
- ServiceNow: incident intake by phone, email, and self-service; triage; assignment groups and roles; work notes vs resolution notes; resolution codes; filtered queues; impersonation
- Help desk practice: identity verification before resets, verifying fixes with the user, documenting as you work
- Hyper-V: Generation 2 VMs, virtual switches, Dynamic Memory, checkpoints
