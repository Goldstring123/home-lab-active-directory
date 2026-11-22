# Home Lab: Active Directory, Group Policy, and Help Desk Ticketing

A virtualized small-business IT environment I built to practice what an IT help desk does every day: managing users and permissions in Active Directory, applying Group Policy, supporting a domain-joined workstation, and tracking problems in a ticketing system. I resolved and documented 10 simulated help desk tickets.

<img src="network-diagram.png" alt="Network diagram of the lab" width="760">

## What I built
| Machine | Operating system | Role | Address |
|---|---|---|---|
| **DC01** | Windows Server 2022 | Domain controller for `lab.local`: Active Directory, DNS, Group Policy, file share | 192.168.10.10 |
| **CLIENT01** | Windows 11 Enterprise (evaluation) | Domain-joined workstation used to test user problems | 192.168.10.20 |
| **TICKET01** | Ubuntu Server 22.04 | osTicket 1.18 help desk (Apache, PHP, MySQL) | 192.168.10.30 |

Everything runs in Oracle VirtualBox 7.2 on one Windows 11 laptop. The three VMs share an isolated internal network, and NAT gives the client and the ticket server internet access for installs.

## Skills demonstrated
- **Active Directory:** a domain, 5 organizational units, 3 security groups, and 12 users; password resets, lockouts, onboarding, offboarding, and account changes.
- **Group Policy:** a domain password and lockout policy (10-character minimum, lockout after 5 attempts), a policy that blocks Control Panel for Sales users, and one that disables USB storage on workstations.
- **Permissions:** a Sales file share controlled by security group membership.
- **Troubleshooting:** DNS misconfiguration, a broken computer trust relationship, a computer in the wrong OU, and policy that was not applying.
- **Help desk process:** 10 tickets in osTicket, each with the issue, troubleshooting, root cause, resolution, and follow-up.
- **Tools:** PowerShell, Group Policy Management, Active Directory Users and Computers, VirtualBox, Ubuntu command line and SSH.

## Build screenshots

### Domain controller and Active Directory
<img src="screenshots/01-dc01-hostname-ip.png" width="620" alt="DC01 hostname and IP address">

*DC01 is named and has a fixed address on the lab network.*

<img src="screenshots/02-domain-lab-local.png" width="620" alt="Active Directory domain lab.local">

*The `lab.local` Active Directory domain is created.*

<img src="screenshots/03-ad-sales-ou.png" width="420" alt="Sales OU"> <img src="screenshots/04-ad-hr-ou.png" width="420" alt="HR OU"> <img src="screenshots/05-ad-it-ou.png" width="420" alt="IT OU">

*Organizational units for Sales (5 users), HR (4), and IT (3), each with its own security group.*

### Group Policy
<img src="screenshots/06-gpo-block-control-panel.png" width="620" alt="Block Control Panel policy">

*The "Sales - Block Control Panel" policy is linked to the Sales OU.*

<img src="screenshots/07-gpo-disable-usb.png" width="620" alt="Disable USB storage policy">

*The "Workstations - Disable USB Storage" policy is linked to the Workstations OU.*

### Domain-joined client
<img src="screenshots/08-client-domain-login-whoami.png" width="620" alt="whoami shows lab\mlopez">

*CLIENT01 is joined to the domain. A domain user (`lab\mlopez`) is logged in.*

<img src="screenshots/09-control-panel-restricted.png" width="620" alt="Control Panel restricted message">

*The Group Policy works: a Sales user is blocked from Control Panel.*

<img src="screenshots/10-sales-share-access.png" width="620" alt="Sales share opened from CLIENT01">

*A Sales user opens the `\\DC01\Sales` file share.*

### Ticketing server
<img src="screenshots/osticket-prerequisites.png" width="520" alt="osTicket prerequisites check">

*Ubuntu 22.04 passes osTicket's requirements check.*

<img src="screenshots/osticket-install-complete.png" width="520" alt="osTicket installation complete">

*osTicket is installed.*

<img src="screenshots/osticket-config-locked.png" width="620" alt="Config file permissions and setup folder removed">

*After setup I locked the installation down: the config file is read-only, and the installer folder is deleted.*

<img src="screenshots/osticket-dashboard.png" width="620" alt="osTicket staff panel">

*The staff panel, ready for tickets.*

## Help desk tickets

| # | Problem | Root cause | Fix |
|---|---|---|---|
| 1 | User forgot password | Forgotten password | Reset the password, forced a change at next logon |
| 2 | New employee needs an account | Onboarding | Created the account and added it to the right group |
| 3 | Can't open the Sales shared drive | User was missing from the group that grants access | Added her to the group; she signed out and back in |
| 4 | Account locked out | Too many failed logins | Unlocked the account and reset the password |
| 5 | Computer not applying policy | Computer was in the wrong container | Moved it into the Workstations OU |
| 6 | Employee left the company | Offboarding | Disabled the account, removed groups, moved it to a Disabled OU |
| 7 | "Trust relationship failed" on login | Computer account password out of sync | Repaired the secure channel with PowerShell |
| 8 | Sales user can't open Control Panel | Working as designed (Group Policy) | No change; documented how an exception would be requested |
| 9 | Can't reach the domain | DNS pointed to a public server | Set DNS back to the domain controller |
| 10 | User name change | Name change | Updated the account name, logon name, and UPN |

The full notes for every ticket are in [`tickets.md`](tickets.md).

<img src="screenshots/osticket-closed-tickets.png" width="760" alt="All 10 tickets closed in osTicket">

*All 10 tickets resolved and closed.*

<img src="screenshots/osticket-ticket-example.png" width="760" alt="Example ticket with internal note">

*An example ticket (#5) with its internal note.*

### Ticket evidence
Each ticket has before-and-after proof.

**Ticket 1: forgot password**

<img src="screenshots/ticket-01-password-reset-commands.png" width="520"> <img src="screenshots/ticket-01-forced-password-change.png" width="400">

**Ticket 2: new employee account**

<img src="screenshots/ticket-02-new-user-groups.png" width="620">

**Ticket 3: can't access the shared drive** (error, fix, result)

<img src="screenshots/ticket-03-before-access-denied.png" width="420"> <img src="screenshots/ticket-03-fix-add-to-group.png" width="420"> <img src="screenshots/ticket-03-after-share-opens.png" width="420">

**Ticket 4: account locked out** (locked screen, then lockout cleared)

<img src="screenshots/ticket-04-account-locked.png" width="420"> <img src="screenshots/ticket-04-06-verification.png" width="420">

**Ticket 5: computer not applying policy** (wrong container, policy missing, fix, policy applied)

<img src="screenshots/ticket-05-setup-wrong-container.png" width="420"> <img src="screenshots/ticket-05-before-policy-missing.png" width="420">
<img src="screenshots/ticket-05-fix-move-to-ou.png" width="420"> <img src="screenshots/ticket-05-after-policy-applied.png" width="420">

**Ticket 6: employee left** (account disabled, removed from group, login refused)

<img src="screenshots/ticket-04-06-verification.png" width="420"> <img src="screenshots/ticket-06-login-disabled.png" width="420">

**Ticket 7: trust relationship failed** (error, then repaired)

<img src="screenshots/ticket-07-trust-error.png" width="420"> <img src="screenshots/ticket-07-after-repaired.png" width="420">

**Ticket 8: Control Panel blocked by policy** (working as designed)

<img src="screenshots/ticket-08-policy-and-ou-check.png" width="520"> <img src="screenshots/09-control-panel-restricted.png" width="420">

**Ticket 9: can't reach the domain** (wrong DNS, policy update fails, fixed)

<img src="screenshots/ticket-09-before-dns-wrong.png" width="420"> <img src="screenshots/ticket-09-before-gpupdate-fails.png" width="420"> <img src="screenshots/ticket-09-after-fixed.png" width="420">

**Ticket 10: user name change**

<img src="screenshots/ticket-10-name-change.png" width="520"> <img src="screenshots/ticket-10-login-whoami.png" width="400">

## What I learned
- A user's access comes from the groups they belong to, and a change only reaches them after a new sign-in.
- Group Policy depends on where an object sits in Active Directory. A computer in the wrong container never receives the policy.
- Domain-joined computers need the domain controller as their DNS server, or they cannot find the domain.
- Writing the root cause and follow-up for each ticket turns a quick fix into something the next person can use.

## Limitations
- This is a **simulated environment**: the users, tickets, and reporter emails are made up. Tickets were created by me to practice, not reported by real users.
- The Windows installs are **evaluation versions**, and the domain has a single domain controller with no backup.
- I reach the osTicket staff panel from the host laptop's browser (through a VirtualBox port forward). I did not test opening it from CLIENT01.
- Passwords are not stored in this repository.

## Files
| File | What it is |
|---|---|
| [`tickets.md`](tickets.md) | The 10 ticket notes |
| [`commands-used.md`](commands-used.md) | The main commands used to build each machine |
| `network-diagram.png` / `.svg` | Lab network diagram |
| `screenshots/` | Evidence for each stage and ticket |
