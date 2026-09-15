# User Lifecycle Management — Onboarding, Activation & Offboarding via GPO

## Scenario

Modeling the full employee lifecycle in `rrl.local`: a lifecycle-based OU structure, an onboarding GPO that provisions access automatically, PowerShell-driven activation into a department, an offboarding GPO that locks a disabled account out entirely, and verification that policies actually applied on the client — not just that they were configured.

> **Lab note:** `CompanyShare` is hosted directly on the DC (`RRL-DC01`) for simplicity in this lab. In a production environment, user file shares would live on a dedicated file server rather than a domain controller, to keep DC resources isolated and reduce attack surface on a security-critical role.

## 1. OU Structure

Created a lifecycle-based OU structure directly under the domain root, separate from department membership, so a user's *position in the lifecycle* is visible independently of *who they work for*.

```
rrl.local
├── Staging
├── Active Users
│     ├── IT
│     ├── Finance
│     └── HR
├── Disabled Users
└── Service Accounts
```

- `Staging` — holding area for new hires before department assignment
- `Active Users` — subdivided by department (`IT`, `Finance`, `HR`) for currently active employees
- `Disabled Users` — destination for offboarded accounts
- `Service Accounts` — kept separate from human user accounts entirely
- Created a test user, **Wanele Xulu** (`wxulu`), inside `Staging` to move through the full lifecycle
- Verified all OUs nested correctly in Active Directory Users and Computers

## 2. Shared Resource Setup

Set up the resource the onboarding policy would provision access to:

- Created a folder `C:\CompanyShare` on the DC
- Added an Acceptable Use Policy (AUP) document inside it
- Shared the folder via **Properties → Sharing → Advanced Sharing**, setting share permissions to **Read-only for Authenticated Users**

## 3. Onboarding GPO

**Creating and linking the GPO**

- Opened Group Policy Management → **Forest → Domains → rrl.local**
- Right-clicked the `Staging` OU → **Create a GPO in this domain, and link it here...**
- Named it **Onboarding Staging Policy** — scoped to `Staging` only, so it applies exclusively to users still in the onboarding phase

**Building the logon script**

- Edited the GPO: **User Configuration → Policies → Windows Settings → Scripts (Logon/Logoff) → Logon**
- Clicked **Add... → Browse**, which opens the GPO's SYSVOL scripts folder directly (scripts must live in SYSVOL so they replicate to all DCs and are reachable by clients)
- Created a new script file, `logon.bat`, inside that folder
- Edited the script to map a network drive to the shared folder:

```bat
net use Z: \\RRL-DC01\CompanyShare
```

- Saved the script, then selected `logon.bat` in the **Add a Script** dialog and confirmed

## 4. Activation — Moving the User Out of Staging

Rather than dragging-and-dropping in the GUI, activation was scripted:

```powershell
Get-ADUser -Identity wxulu | Move-ADObject -TargetPath "OU=IT,OU=Active Users,DC=rrl,DC=local"
```

- Pipes the user object from `Get-ADUser` directly into `Move-ADObject`, moving `wxulu` from `Staging` into `IT` under `Active Users`
- Verified in Active Directory Users and Computers: `wxulu` no longer appeared in `Staging` and was now listed under `Active Users → IT`
- Leaving `Staging` also means the onboarding GPO (scoped only to that OU) no longer applies to this account

## 5. Offboarding GPO

Built to strip a disabled account of effectively all access, not just disable login:

- In Group Policy Management, created a GPO on the `Disabled Users` OU named **Offboarding Policy**
- Edited: **Computer Configuration → Policies → Windows Settings → Security Settings → Local Policies → User Rights Assignment**
- Configured three **Deny** rights, each scoped to `Domain Users`:
  - **Deny log on locally**
  - **Deny access to this computer from the network**
  - **Deny log on through Remote Desktop Services**

Stacking all three closes the main paths a disabled-but-not-yet-deleted account could still be used through — local console, network share access, and RDP — rather than relying on `Disable-ADAccount` alone.

## 6. Offboarding the Test User

```powershell
Disable-ADAccount -Identity wxulu
Move-ADObject -Identity (Get-ADUser wxulu).DistinguishedName -TargetPath "OU=Disabled Users,DC=rrl,DC=local"
```

- Disables the account first, then moves it into `Disabled Users`, where the offboarding GPO's deny rights take effect
- Verified in Active Directory Users and Computers: `wxulu` moved out of `Active Users → IT` and into `Disabled Users`

## 7. Proving the Policies Actually Applied

Configuring a GPO isn't the same as confirming it took effect — verified directly on the domain-joined client:

- Forced a policy refresh:

```cmd
gpupdate /force
```

- Waited for confirmation that the computer policy update completed successfully
- Generated a full application report:

```cmd
gpresult /h C:\report.html
```

- Opened the HTML report and confirmed the expected GPOs (onboarding and offboarding policies) appeared as applied, rather than assuming success from the GPO editor alone

