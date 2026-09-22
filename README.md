# Enterprise Active Directory

An enterprise-style virtual environment built in EVE-NG to practice standing up a Windows domain, structuring Active Directory, enforcing endpoint restrictions with Group Policy, and serving IP addresses from a DHCP scope. This repo documents the build, the configuration, and screenshot proof that the policies actually took effect on a client machine.

## Overview

- **Domain:** office.local
- **Domain Controller:** Windows Server 2019, running AD DS, DHCP, and DNS
- **Client:** Windows 10 VM (Win2), joined to the domain
- **Hypervisor / Network Emulator:** EVE-NG
- **Core goals:**
  - Stand up a Domain Controller and join a client to the domain
  - Organize Active Directory with a custom OU structure for policy targeting
  - Build and link a GPO for user-level endpoint hardening
  - Configure and validate a DHCP scope
  - Prove policy enforcement with before/after screenshots on the client

## Lab Topology

The lab runs on a switch connecting the Domain Controller, three Windows client VMs, and a router running EIGRP providing upstream connectivity out to the internet cloud.

![Lab Topology](/01-topology.png)

| Device | Role | Interface |
|---|---|---|
| Winserver | Domain Controller, DHCP, DNS | e0 to switch Gi1/0 |
| Win2 | Domain-joined client (primary test machine) | e0 to switch Gi0/1 |
| Win3 | Domain-joined client | e0 to switch Gi0/2 |
| Win4 | Domain-joined client | e0 to switch Gi0/3 |
| Switch | Layer 2 connectivity between all VMs and the router | Gi0/0 to vIOS |
| vIOS | Routes traffic out to the internet cloud | Gi0/0 to Net |

## Active Directory and OU Design

The Domain Controller was configured for the office.local forest and domain. Rather than leaving computer and user objects in the default containers, a custom Organizational Unit called **Workstations** was created to properly segregate client objects. This was a deliberate fix: the default Computers container in AD does not support Group Policy linking, so any GPO targeting client machines would silently fail to apply. 

This structure keeps policy scope clean and gives room to expand with additional OUs (for example, separating servers, staff workstations, and service accounts) as the lab grows.

## DHCP Configuration

DHCP is running on the Domain Controller, serving a scope for client workstations.

![DHCP console showing active scope](/02-dhcp-console.png)

**Scope details:**
- Scope name: `OFFICE_PC_IP`
- Network: `172.168.10.0`
- Address pool: `172.168.10.1` to `172.168.10.254`
- Excluded addresses: `172.168.10.1` and `172.168.10.2` (reserved for Domain Controller and Default Gateway)

![DHCP address pool and exclusions](/03-dhcp-scope.png)

Active leases confirm clients are pulling addresses correctly from the pool:

![DHCP address leases](/04-dhcp-leases.png)

## Group Policy Object: user_base_policy

A custom GPO named **user_base_policy** was created and linked to the Workstations OU. It applies under User Configuration > Policies > Administrative Templates, targeting Control Panel and System restrictions for any user logging into a machine in scope. 

![Server Manager navigation](/05-server-manager-menu.png)

### Control Panel restrictions

![Group Policy Management Editor, Control Panel settings](/06-gpo-control-panel.png)

### System restrictions

The following user-level restrictions were configured and enabled under System settings:

| Policy | State | Purpose |
|---|---|---|
| Prevent access to the command prompt | Enabled | Blocks cmd.exe and .bat/.cmd script execution to stop manual CLI use or script-based workarounds |
| Prevent access to registry editing tools | Enabled | Blocks regedit.exe to prevent unauthorized changes to the registry |
| Remove Task Manager | Enabled | Blocks taskmgr.exe to prevent users from killing processes or inspecting running services |
| Remove Change Password | Enabled | Removes the Change Password button from the Ctrl+Alt+Del security screen |

![System policy list showing enabled restrictions](/07-gpo-system-settings.png)



![Remove Task Manager policy dialog](/08-gpo-task-manager.png)

![Remove Change Password policy dialog](/09-gpo-change-password.png)

## Applying and Validating the Policy

After linking the GPO, policy was force-pushed from the Domain Controller and refreshed on the clients to avoid waiting on the default background refresh interval:

```
gpupdate /force
```

![gpupdate force on the Domain Controller](/10-gpupdate-force.png)

During testing, GPO changes did not always apply immediately in the user context. This was resolved by running `gpupdate /force` on the server side, followed by a full user logoff/logon (rather than just a lock/unlock) to force the user-context policy to reprocess.

## Proof of Enforcement

With the policy applied and refreshed on the client, the following restrictions were verified directly on Win2.

**Command Prompt is blocked:**

![Command prompt disabled by administrator](/11-cmd-disabled.png)

**Other applications, like the browser, continue to function normally**, confirming the restriction is scoped to the intended tools rather than breaking the whole session:

![Browser still functioning normally](/12-browser-still-works.png)

**Task Manager is blocked:**

![Task Manager disabled by administrator](/13-taskmgr-disabled.png)

**Restricted actions return the standard Windows administrative block dialog:**

![Restrictions dialog](/14-restrictions-dialog.png)


## Design Philosophy

This lab intentionally keeps a separation of concerns between layers of control:

- **Group Policy (this project)** is used strictly for endpoint hardening and desktop-level restrictions, locking down what a logged-in user can do on their own machine.
- **Perimeter security** (URL filtering, traffic inspection, firewall rules) is treated as a separate concern, handled at the network edge rather than bolted onto endpoint GPOs; ie Fortigate would be used for content filtration.

Keeping these responsibilities distinct mirrors how enterprise environments are actually structured, where endpoint policy and network security are managed independently but work together.

## Environment Summary

- **Platform:** EVE-NG
- **Domain Controller OS:** Windows Server
- **Client OS:** Windows 10
- **Domain:** office.local
- **DHCP scope:** 172.168.10.0/24, pool 172.168.10.1 to 172.168.10.254
- **Key GPO:** user_base_policy, linked to the Workstations OU
