# Help Desk Ticket Log

The 10 simulated tickets I resolved in the lab, as recorded in osTicket. Each entry lists the issue, what I checked, the root cause, the fix, and a follow-up. No passwords appear here.

---
## Ticket 1: User forgot password (mlopez)
**Priority:** Medium | **Help topic:** Account access

**Issue:** Maria Lopez (Sales) reported she could not log in because she forgot her password.

**Notes:**
- **Troubleshooting:** Confirmed the account exists and is enabled in Active Directory (Sales OU). Verified the user's identity before changing credentials (simulated). Confirmed the account was not locked out.
- **Root cause:** Forgotten password. No account fault.
- **Resolution:** Reset the password with `Set-ADAccountPassword mlopez -Reset` and issued a temporary password that meets the domain policy (10+ characters, complexity). Set "change password at next logon." At the next sign-in the user was prompted to choose a new password. 
- **Follow-up:** Reminded the user the new password must be 10+ characters with upper and lower case, a number, and a symbol.

---

## Ticket 2: New employee needs an account (rortiz)
**Priority:** Medium | **Help topic:** Onboarding

**Issue:** HR requested a new account for Ryan Ortiz, a new Sales hire.

**Notes:**
- **Troubleshooting:** Confirmed the username `rortiz` was not already in use and that the department is Sales.
- **Root cause:** New hire onboarding.
- **Resolution:** Created the account in the Sales OU with `New-ADUser` (temporary password, change at next logon). Added the user to GG-Sales with `Add-ADGroupMember`. Verified membership with `Get-ADPrincipalGroupMembership rortiz`, which listed GG-Sales.
- **Follow-up:** Temporary password to be given to the user through a secure channel (simulated).

---

## Ticket 3: Can't access Sales shared drive (gkim)
**Priority:** Medium | **Help topic:** Permissions

**Issue:** Grace Kim reported "Access is denied" when opening \\DC01\Sales.

**Notes:**
- **Troubleshooting:** Reproduced the error by opening \\DC01\Sales as the user on CLIENT01. Checked group membership with `Get-ADPrincipalGroupMembership gkim`: she was in Domain Users and GG-HR only. The share grants access to GG-Sales.
- **Root cause:** User was missing from the security group that grants access to the share.
- **Resolution:** Added the user to the group with `Add-ADGroupMember GG-Sales gkim`, had her sign out and back in so the new membership applied, and confirmed the share opens.
- **Follow-up:** Access is controlled by group, not by user. Access requests should come through a ticket so they are approved and logged.

---

## Ticket 4: Account locked out (jcarter)
**Priority:** High | **Help topic:** Account access

**Issue:** James Carter could not log in to CLIENT01. The screen said the referenced account is currently locked out.

**Notes:**
- **Troubleshooting:** Reproduced the lockout with repeated failed logins. Checked the account with `Get-ADUser jcarter -Properties LockedOut`, which returned True. The domain policy locks accounts after 5 failed attempts for 30 minutes.
- **Root cause:** Too many incorrect password attempts triggered the lockout policy.
- **Resolution:** Unlocked the account with `Unlock-ADAccount jcarter`. The user did not remember the password, so it was reset after identity verification (simulated). Confirmed `LockedOut` returns False and the user can log in.
- **Follow-up:** Explained the lockout policy and advised calling the help desk after two failed attempts instead of repeating guesses.

---

## Ticket 5: Computer not applying policy (CLIENT01)
**Priority:** Medium | **Help topic:** Group Policy

**Issue:** CLIENT01 was not receiving the "Disable USB Storage" policy.

**Notes:**
- **Troubleshooting:** Ran `gpupdate /force` and `gpresult /r /scope computer` on CLIENT01. Only the Default Domain Policy was listed. Checked the computer account's location in Active Directory: it was in the default Computers container, not the Workstations OU where the policy is linked.
- **Root cause:** The computer account was in the wrong location, so the OU-linked policy never reached it.
- **Resolution:** Moved the account to the Workstations OU with `Move-ADObject`, restarted CLIENT01, and re-ran `gpresult`. The USB policy now applies.
- **Follow-up:** New computers land in the default Computers container, so they should be moved into the correct OU when they are joined.

---

## Ticket 6: Employee left the company (nbrooks)
**Priority:** High | **Help topic:** Offboarding

**Issue:** HR reported that Nina Brooks has left the company and her access must be removed.

**Notes:**
- **Troubleshooting:** Located the account in the HR OU and confirmed her group membership (GG-HR).
- **Root cause:** Planned offboarding.
- **Resolution:** Disabled the account with `Disable-ADAccount nbrooks`, removed her from GG-HR, and moved the account to the Disabled OU. Verified Enabled is False, the account is in the Disabled OU, and she no longer appears in the GG-HR member list. A test login returned "Your account has been disabled."
- **Follow-up:** The account is kept, not deleted, for audit and retention. In a real environment, also recover equipment and review mailbox and file ownership.

---

## Ticket 7: "Trust relationship failed" on login (CLIENT01)
**Priority:** High | **Help topic:** Domain / Computer account

**Issue:** A user could not log in to CLIENT01. The screen said "The trust relationship between this workstation and the primary domain failed."

**Notes:**
- **Troubleshooting:** Domain logins were refused, so I used a cached administrator login with the network disconnected, then reconnected and checked the secure channel with `Test-ComputerSecureChannel`.
- **Root cause:** The computer account password in Active Directory no longer matched the one stored on the workstation, so the domain rejected the computer's login.
- **Resolution:** Repaired the secure channel with `Test-ComputerSecureChannel -Repair` using domain administrator credentials and confirmed it returned True.
- **Follow-up:** Avoid restoring old snapshots of domain-joined computers without checking the trust afterward. If the repair fails, remove the computer from the domain and rejoin it.

---

## Ticket 8: Sales user can't open Control Panel (mlopez)
**Priority:** Low | **Help topic:** Group Policy

**Issue:** Maria Lopez (Sales) reported she could not open Control Panel and got a "Restrictions" error.

**Notes:**
- **Troubleshooting:** Confirmed her account is in the Sales OU and that the "Sales - Block Control Panel" Group Policy is enabled and linked to that OU.
- **Root cause:** Working as designed. The policy deliberately blocks Control Panel for the Sales OU.
- **Resolution:** No change made. Explained the policy to the user. An exception would require manager approval and a security-filtering change.
- **Follow-up:** Pointed the user to the help desk for any setting changes she needs.

---

## Ticket 9: Can't reach the domain (CLIENT01)
**Priority:** High | **Help topic:** Network / DNS

**Issue:** CLIENT01 could not apply Group Policy, and the user reported problems reaching domain resources.

**Notes:**
- **Troubleshooting:** `gpupdate /force` failed with a name resolution error. `Resolve-DnsName lab.local` returned "DNS name does not exist." `Get-DnsClientServerAddress` showed the lab adapter pointing to a public DNS server (8.8.8.8) instead of the domain controller.
- **Root cause:** Incorrect DNS server. Domain-joined computers must use the domain controller for DNS to find the domain.
- **Resolution:** Set the DNS server back to 192.168.10.10 with `Set-DnsClientServerAddress`, flushed the cache with `ipconfig /flushdns`, and confirmed `Resolve-DnsName lab.local` returns the domain controller.
- **Follow-up:** DNS on domain computers should come from the DC or DHCP, not be set by hand.

---

## Ticket 10: User name change (Aisha Khan to Aisha Malik)
**Priority:** Low | **Help topic:** Account changes

**Issue:** Aisha Khan changed her name to Aisha Malik, and her account details needed updating.

**Notes:**
- **Troubleshooting:** Confirmed the existing account `akhan` in the Sales OU.
- **Root cause:** Name change.
- **Resolution:** Updated the surname, display name, logon name and UPN with `Set-ADUser`, renamed the directory object with `Rename-ADObject`, and reset the password after identity verification (simulated). Verified the new name in Active Directory and logged in on CLIENT01 as the renamed account.
- **Follow-up:** She should use the new username from now on. In a real environment, email and other systems would also need the new name.
