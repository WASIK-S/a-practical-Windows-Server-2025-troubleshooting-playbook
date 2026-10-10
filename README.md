# Windows Server 2025 — Active Directory Troubleshooting Playbook

A practical, command-based reference for diagnosing and resolving domain authentication issues in a Windows Server 2025 Active Directory lab.

## Project Overview

This playbook documents a real troubleshooting scenario in my lab environment, where a domain user could not sign in to a Windows Server machine. I investigated domain controller health, DNS, Active Directory replication, account status, and local logon permissions before implementing and verifying the resolution.

**Primary objective:** Identify why domain authentication was failing, verify the Active Directory environment, apply the appropriate correction, and document repeatable troubleshooting steps.

### Environment

| Component | Configuration |
|---|---|
| Server OS | Windows Server 2025 |
| Active Directory domain | `abbu.com` |
| Original primary DC | `ADMIN-SYS` |
| Primary DC IP | `10.10.10.100` |
| Second server | `USER-SYS` |
| Second server IP | `10.2.2.100` |
| Domain user tested | `fin01` |
| Administrative tools | PowerShell, Active Directory tools, `dcdiag`, `repadmin`, `secedit` |

> Lab note: These are the recorded lab settings. Confirm that the IP addresses, network configuration, and host roles match your current environment before reusing commands.

---

## 1. Problem Statement

When attempting to sign in to the Windows server using a domain account, Windows displayed:

> The sign-in method you're trying to use isn't allowed. For more info, contact your network administrator.

This message indicated that the attempted sign-in was not permitted. It did not, by itself, prove whether the cause was a user-right assignment, local group membership, Group Policy, domain controller configuration, or another authentication-related issue.

The goal was to investigate systematically rather than make random policy changes.

## 2. Troubleshooting Workflow

The investigation followed this sequence:

1. Confirm the domain and server roles.
2. Verify Active Directory domain controller discovery.
3. Check replication and domain controller health.
4. Inspect DNS configuration.
5. Verify the domain user account.
6. Examine local group membership and interactive logon rights.
7. Correct the affected server's configuration.
8. Test domain-user sign-in and verify the result.

---

## 3. Step 1 — Confirm Domain Membership and Server Role

Run the following commands in PowerShell on the affected server.

```powershell
Get-CimInstance Win32_ComputerSystem |
    Select-Object Name, Domain, PartOfDomain, DomainRole
```

### What to check

- `Name`: the current computer name.
- `Domain`: the joined domain.
- `PartOfDomain`: whether the computer is domain-joined.
- `DomainRole`: the computer's domain role.

Common `DomainRole` values:

| Value | Meaning |
|---:|---|
| 0 | Standalone workstation |
| 1 | Member workstation |
| 2 | Standalone server |
| 3 | Member server |
| 4 | Additional domain controller (legacy backup-DC classification) |
| 5 | Primary domain controller (legacy classification) |

**Observed result:** `USER-SYS` initially had `DomainRole = 4`, confirming that it was operating as an additional domain controller rather than an ordinary member server.

After the demotion, its role was verified as `DomainRole = 3`, and it remained joined to `abbu.com`.

### Why this mattered

Domain controllers and member servers have different responsibilities and security configurations. Confirming the role helped establish which troubleshooting path applied to the affected machine.

---

## 4. Step 2 — Verify Domain Controller Discovery

Run these commands from a domain-connected machine:

```powershell
nltest /dsgetdc:abbu.com
```

```powershell
nslookup abbu.com
```

```powershell
nslookup ADMIN-SYS
```

Check the discovered domain controller, DNS server, domain name, and returned IP addresses.

You can also query Active Directory for domain controllers:

```powershell
Get-ADDomainController -Filter * |
    Select-Object HostName, IPv4Address, IsGlobalCatalog
```

If the Active Directory PowerShell module is unavailable, install the appropriate Remote Server Administration Tools feature or run the query from a system that has the module.

### Interpretation

- A successful `nltest` result indicates that a domain controller was discovered.
- DNS failures may prevent clients from locating domain controllers.
- A successful lookup alone does not prove that authentication and replication are healthy.

**Observed result:** Both `ADMIN-SYS` and `USER-SYS` were initially discoverable as domain controllers.

---

## 5. Step 3 — Check Active Directory Replication

On `ADMIN-SYS`, run:

```powershell
repadmin /replsummary
```

To inspect replication partners and the latest replication status:

```powershell
repadmin /showrepl
```

### Expected result

Look for:

- Zero replication failures.
- No persistent replication errors.
- Recent successful replication attempts.
- Correct replication partners.

**Observed result:** `repadmin /replsummary` reported zero replication failures in the investigation.

### Why this matters

Replication problems can cause inconsistent directory data across domain controllers. However, healthy replication does not rule out local security-policy or sign-in-permission problems.

---

## 6. Step 4 — Run Domain Controller Diagnostics

On `ADMIN-SYS`, run:

```powershell
dcdiag /s:ADMIN-SYS /test:Advertising /test:Services /test:DNS
```

These checks examine selected aspects of domain controller advertising, services, and DNS health.

You can also inspect the domain's DNS zones:

```powershell
Get-DnsServerZone |
    Select-Object ZoneName, IsDsIntegrated, ZoneType
```

**Observed result:**

- Advertising, Services, and DNS diagnostic tests passed.
- The `abbu.com` and `_msdcs.abbu.com` zones were present and Active Directory-integrated.

### Troubleshooting note

If a test fails, inspect the specific error before changing configuration. A diagnostic failure is evidence to investigate, not automatic proof of the root cause.

---

## 7. Step 5 — Verify the Domain User Account

A mistyped account name occurred during the investigation: `soc1` was queried instead of `soc01`. The lookup returned no matching user.

This reinforced the importance of confirming the exact account name before diagnosing authentication.

Run:

```powershell
Get-ADUser fin01 -Properties Enabled, LockedOut, MemberOf |
    Select-Object SamAccountName, Enabled, LockedOut, MemberOf
```

For the `soc01` account:

```powershell
Get-ADUser soc01 -Properties Enabled, LockedOut, MemberOf |
    Select-Object SamAccountName, Enabled, LockedOut, MemberOf
```

To check whether an account has a workstation restriction:

```powershell
Get-ADUser soc01 -Properties LogonWorkstations |
    Select-Object SamAccountName, LogonWorkstations
```

### What to verify

- The account exists under the correct name.
- `Enabled` is `True`.
- `LockedOut` is `False`.
- Group membership is appropriate.
- `LogonWorkstations` is not restricting the intended computer.

**Observed result:** `soc01` was enabled, was not locked out, and had no workstation restriction. Its recorded group membership was Domain Users.

These findings helped rule out several account-level causes but did not independently identify the original sign-in failure.

---

## 8. Step 6 — Inspect Local Users Group Membership

After `USER-SYS` was converted to a member server, inspect its local Users group.

Run on `USER-SYS` in an elevated PowerShell session:

```powershell
Get-LocalGroupMember -Group "Users"
```

The observed membership included:

- `NT AUTHORITY\Authenticated Users`
- `NT AUTHORITY\INTERACTIVE`

The domain group `ABBU\Domain Users` was not listed.

### Add Domain Users to the local Users group

The following command was used on `USER-SYS`:

```powershell
Add-LocalGroupMember -Group "Users" -Member "ABBU\Domain Users"
```

Verify the result:

```powershell
Get-LocalGroupMember -Group "Users"
```

**Observed result:** `ABBU\Domain Users` was added to the local Users group, and the membership was verified.

### Important distinction

Membership in the local Users group and permission to sign in interactively are related but separate controls. Local or domain Group Policy can still deny logon, so verify effective user-right assignments when the error persists.

---

## 9. Step 7 — Inspect Interactive Logon Rights

The sign-in error made the interactive logon user-right assignments relevant to the investigation.

Export the effective security configuration on the affected machine:

```powershell
secedit /export /cfg C:\Windows\Temp\effective-security.inf
```

Find the interactive logon setting:

```powershell
Select-String `
    -Path C:\Windows\Temp\effective-security.inf `
    -Pattern "SeInteractiveLogonRight|SeDenyInteractiveLogonRight"
```

The important settings are:

- `SeInteractiveLogonRight`: **Allow log on locally**
- `SeDenyInteractiveLogonRight`: **Deny log on locally**

A deny assignment can override an allow assignment. Review both settings and the effective policy rather than relying only on a graphical policy editor.

To review applied Group Policy, run:

```powershell
gpresult /scope computer /r
```

For a more detailed report:

```powershell
gpresult /h C:\Windows\Temp\gpresult.html
```

Open the generated HTML report and review the policies applied to the computer.

### Observed result

The exported security configuration on `USER-SYS` contained an entry for `SeInteractiveLogonRight`, including the Everyone SID, a domain-specific SID, the built-in Administrator SID, and the built-in Administrators SID.

This was evidence of the effective assignment at the time of export. It should not be interpreted as proof that one particular SID or policy was the sole original cause.

---

## 10. Step 8 — Demote USER-SYS to a Member Server

The investigation identified that `USER-SYS` was an additional domain controller. The chosen correction was to demote it through the normal Active Directory Domain Services demotion workflow.

**Do not force-remove a domain controller when a normal demotion is possible.** A normal demotion allows the server to remove its domain controller role through the supported process.

Before demotion, verify:

- Another healthy domain controller is available.
- DNS and other required roles are hosted appropriately.
- The server is not the only holder of a required directory role.
- Replication and directory health have been checked.
- Any required DNS, Global Catalog, or other dependencies are understood.

In this lab, `ADMIN-SYS` held all FSMO roles. The normal demotion workflow was completed on `USER-SYS`, and the DNS/Global Catalog warning was reviewed during the process.

After demotion, verify the server role:

```powershell
Get-CimInstance Win32_ComputerSystem |
    Select-Object Name, Domain, PartOfDomain, DomainRole
```

**Observed result:**

```text
Name        : USER-SYS
Domain      : abbu.com
PartOfDomain: True
DomainRole  : 3
```

This confirmed that `USER-SYS` remained domain-joined as a member server.

> Demoting a domain controller is an architectural change, not a universal fix for this sign-in error. It was the chosen change in this lab because the affected server was being transitioned to a member-server role.

---

## 11. Step 9 — Verify the Resolution

After the demotion and local Users group membership change, the domain user `fin01` successfully signed in to `USER-SYS`.

The following outcomes were verified:

- `USER-SYS` was joined to `abbu.com`.
- Its computer role was `DomainRole = 3`.
- `ABBU\Domain Users` appeared in the local Users group.
- The `fin01` sign-in test succeeded.

### Final outcome

**Confirmed:** The sign-in issue was resolved in the tested scenario following the server-role change and local group membership update.

**Not conclusively established:** The available evidence does not isolate one original root cause. The sequence of changes and successful test establishes the observed resolution, but it does not prove which individual change was necessary or sufficient.

---

## 12. Troubleshooting Quick Reference

| Symptom | Command or check | What to investigate |
|---|---|---|
| Domain user cannot sign in | Check exact error and event logs | Logon rights, deny policies, account restrictions |
| Domain controller cannot be found | `nltest /dsgetdc:abbu.com` | DNS configuration and connectivity |
| DNS lookup fails | `nslookup abbu.com` | DNS server and AD DNS records |
| Replication errors appear | `repadmin /replsummary` | Replication partners and directory health |
| Domain controller tests fail | `dcdiag` | Specific failing service or test |
| User lookup returns no object | `Get-ADUser <account>` | Account spelling and domain context |
| User is denied locally | `Get-LocalGroupMember -Group "Users"` | Local group membership and effective logon rights |
| Policy changes do not take effect | `gpresult /h C:\Windows\Temp\gpresult.html` | Winning GPO and effective user-right assignments |

---

## 13. Lessons Learned

1. **Start with evidence.** Record the exact error message and identify the affected computer and account before making changes.
2. **Confirm the machine's role.** A domain controller and a member server require different troubleshooting approaches.
3. **Check DNS before blaming authentication.** Active Directory depends on DNS to locate domain controllers and services.
4. **Use multiple diagnostic tools.** `dcdiag`, `repadmin`, `nltest`, `Get-ADUser`, and `secedit` each answer different questions.
5. **Check both allow and deny rights.** A user may be in an appropriate local group and still be denied interactive logon.
6. **Verify the exact account name.** A typo can produce a misleading account-not-found result.
7. **Make one change at a time where possible.** Re-test after each change so the cause can be isolated.
8. **Use normal demotion procedures.** Do not force-remove a domain controller merely to resolve an authentication symptom.
9. **Record the actual result.** A successful test is stronger evidence than assuming a command fixed the issue.
10. **Separate confirmed facts from hypotheses.** Document what was observed, what changed, and what remains unproven.

---

## 14. Suggested Next Checks for Future Incidents

If the same sign-in error occurs again, collect additional evidence before making structural changes.

### Review security events

On the affected server, inspect the Security log around the failed sign-in time. Relevant events can include failed logons and logon-right or policy-related errors. Event IDs and details vary by failure type, so review the event message and status/substatus fields rather than relying on the ID alone.

### Check effective policy

```powershell
gpresult /h C:\Windows\Temp\gpresult.html
```

Review the winning policy for **Allow log on locally** and **Deny log on locally**. If domain Group Policy controls the setting, correct the appropriate GPO rather than making an untracked local change.

### Re-test with a known domain account

Use an authorized test account and record:

- The exact computer name.
- The exact account used.
- Whether the test was console or Remote Desktop sign-in.
- The exact error message.
- The time of the test.
- Relevant event log entries.

This creates a repeatable troubleshooting record.

---

## Conclusion

This lab exercise demonstrated a structured approach to Windows Server 2025 Active Directory troubleshooting: verify server roles, validate domain controller and DNS health, inspect replication and user accounts, review local group membership and effective security policy, then test the result.

The most important takeaway is simple: **diagnose first, change deliberately, and verify the outcome.**

*Environment: Windows Server 2025, Active Directory Domain Services, PowerShell. This is a lab reference, and commands should be adapted to the target environment before production use.*
