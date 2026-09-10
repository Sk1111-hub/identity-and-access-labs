# Rhino Route Logistics — Active Directory Deployment

## Scenario

Rhino Route Logistics, a fictional logistics company with multiple branch offices, needed a centralized identity and access structure before onboarding additional sites. This build simulates standing up their first branch (Durban) on a new Active Directory forest: domain controller, organizational unit structure, security groups, and a domain-joined client — with delegated access managed through group membership rather than individual permissions.

## Environment

| Component | Detail |
|---|---|
| Domain Controller | Windows Server 2025 Datacenter, Azure Edition — `RRL-DC01` |
| Domain | `rrl.local` (new forest) |
| Client VM | Same OS edition, same resource group, same VNet/subnet as the DC |
| Networking | DC's private IP set to static (prevents DNS/AD breakage from IP drift) |

## 1. Domain Controller Promotion

- Promoted the server via Server Manager → *Promote this server to a domain controller*
- Created a new forest, root domain `rrl.local`
- Left forest/domain functional levels at default
- Enabled **DNS Server** and **Global Catalog** roles
  - DNS lets clients locate the DC; Global Catalog lets the DC answer forest-wide queries
- Left DNS delegation, NetBIOS name, and default paths (database, logs, SYSVOL) unchanged
- Ran prerequisites check, installed, and let the VM reboot into its new role
- Verified the promotion by logging in as Domain Admin
<img width="1920" height="1011" alt="RRL SCREENSHOT" src="https://github.com/user-attachments/assets/2b9aa952-f0f2-4e47-9a94-fbf3c08a7034" />


## 2. OU Structure — Branch-Based Design

OUs were structured by physical location so that Group Policy and delegated administration can be scoped per branch without affecting others.

```
rrl.local
└── _Branches            (protected from accidental deletion)
    └── Durban            (protected from accidental deletion)
        ├── Users
        ├── Workstations
        └── Laptops
```

- `_Branches` created as the top-level container for all branch OUs
- `Durban` created as the first branch, matching the physical location naming convention
- Sub-OUs (`Users`, `Workstations`, `Laptops`) created for targeted policy application and easier troubleshooting
- Every OU protected from accidental deletion
- Verified nesting under *Active Directory Users and Computers*

## 3. User Provisioning

- Created 3 users inside `_Branches > Durban > Users`
- Set initial passwords; unchecked *User must change password at next logon* (lab convenience — would be enabled in production)
- Left *User cannot change password* and *Password never expires* unchecked
- Verified users appeared under the correct nested OU

## 4. Security Groups — Centralized, Not Branch-Scoped

Groups were deliberately kept out of the branch OU hierarchy and centralized under their own OU. This follows the standard access model: **users are placed into groups, and groups are granted access to resources** — not individual users.

```
rrl.local
└── _Groups               (protected from accidental deletion)
    ├── Helpdesk           (Global, Security)
    ├── Accounting         (Global, Security)
    └── ITSupport          (Global, Security)
```

- Created the `_Groups` OU directly under the domain root
- Created `Helpdesk`, `Accounting`, and `ITSupport` as Global scope, Security type groups
- Verified group configuration after creation

## 5. Group Membership (GUI)

For each group:
1. Open the group → **Members** tab → **Add...**
2. Type the username → **Check Names** → **OK**

Repeated for all three groups against their respective users.

## 6. Client Domain Join

- Client VM built in the same resource group, same OS edition, and matched VNet/subnet to the DC
- **Before joining:** set the client's preferred DNS server to the DC's private IP (Network Connections → Ethernet → Properties → IPv4)
  - AD relies on DNS to locate domain controllers — incorrect DNS is the most common cause of domain-join failures
- Joined via **Settings → System → About → Domain or Workgroup → Change**, entered `rrl.local`
- Logged in with domain credentials, restarted to apply
- Verified by logging into the client with the domain admin account

## 7. Delegated RDP Access via PowerShell

Rather than adding access through the GUI, group membership for Remote Desktop was delegated via script on the client VM:

```powershell
Add-LocalGroupMember -Group "Remote Desktop Users" -Member "RRL\Helpdesk"
Add-LocalGroupMember -Group "Remote Desktop Users" -Member "RRL\Accounting"
Add-LocalGroupMember -Group "Remote Desktop Users" -Member "RRL\ITSupport"
```

Verified by logging out and back in as one of the domain users created earlier.

## What This Demonstrates

- Active Directory forest/domain design fundamentals (DNS, Global Catalog roles)
- OU design aligned to organizational structure for delegated administration and GPO targeting
- Security group scoping (Global/Security) and the groups-over-individuals access model
- DNS's role in domain controller location, and awareness of the most common domain-join failure mode
- Domain-joining and troubleshooting a client machine
- Basic AD-aware PowerShell (`Add-LocalGroupMember`) as an alternative to GUI administration
- Azure VM networking fundamentals (static IP addressing, VNet/subnet alignment)

## Next Steps

- Link actual GPOs to the `Workstations` and `Laptops` sub-OUs
- Add a second branch OU to prove the structure scales across locations
- Delegate branch-level administration rights instead of using Domain Admin for all changes
- Document a password/lockout policy per branch via fine-grained password policies
