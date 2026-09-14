# User Lifecycle Management — Onboarding via GPO

## Scenario

Extending the `rrl.local` environment to handle the employee onboarding lifecycle: a dedicated OU structure to track users through staging, active department assignment, and eventual offboarding, plus an onboarding GPO that automatically maps a shared network drive at logon. This models how a real organization structures user provisioning rather than dropping new hires straight into a flat user list.

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
- Created a test user, **Wanele Xulu**, inside `Staging` to move through the lifecycle
- Verified all OUs nested correctly in Active Directory Users and Computers

## 2. Shared Resource Setup

Before building the onboarding policy itself, set up the shared resource it would provision access to:

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

## What This Demonstrates

- Lifecycle-oriented OU design (staging → active → disabled) as distinct from department/functional OU design
- Linking a GPO to a specific OU to scope policy application narrowly (staging-only, not domain-wide)
- Logon script deployment via Group Policy, including why scripts belong in SYSVOL
- Basic Windows file sharing and NTFS/share-level permission configuration (read-only, Authenticated Users)
- Understanding of `net use` for drive mapping and UNC path syntax

## Next Steps

- Move Wanele Xulu from `Staging` into the correct `Active Users` sub-OU and confirm the onboarding GPO no longer applies once out of scope
- Add a corresponding offboarding GPO/process for moving users into `Disabled Users`
- Layer folder redirection or additional logon-script provisioning (e.g. mapping department-specific shares based on OU)
- Verify GPO application with `gpresult /r` on the client, tying into the GPO troubleshooting lab
