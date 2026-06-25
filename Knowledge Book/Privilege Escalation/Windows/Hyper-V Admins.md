## Overview

The `Hyper-V Administrators` group has full access to all Hyper-V features.

This group can become extremely sensitive in Active Directory environments where Domain Controllers are virtualized.

If a Domain Controller is virtualized, virtualization administrators should effectively be treated as Domain Admins because they may be able to access the Domain Controller's virtual disks and extract credential material.

> [!warning]
> If Domain Controllers run as Hyper-V virtual machines, members of `Hyper-V Administrators` may be able to compromise the domain.

---

# Why Hyper-V Administrators Matters

A Hyper-V administrator may be able to:

- manage virtual machines
- access virtual hard disks
- clone live virtual machines
- mount virtual disks offline
- access Domain Controller files
- obtain `NTDS.dit`
- extract NTLM password hashes for all domain users

High-value Domain Controller file:

```text
NTDS.dit
```

If the virtualized Domain Controller disk can be mounted offline, `NTDS.dit` may be extracted and processed for domain credential hashes.

---

# Virtualized Domain Controller Risk

## Attack Concept

If a Domain Controller is running as a VM, a Hyper-V administrator may be able to:

1. Clone the live Domain Controller VM.
2. Mount the cloned virtual disk offline.
3. Locate the Active Directory database.
4. Extract `NTDS.dit`.
5. Extract NTLM hashes offline.

Relevant path:

```text
C:\Windows\NTDS\ntds.dit
```

> [!important]
> This is why virtualization admins for hosts running Domain Controllers should be considered equivalent to highly privileged AD admins.

---

# Hyper-V Admin to SYSTEM via Hard Link Abuse

## Vulnerable Behavior

The notes describe a documented behavior where, when a virtual machine is deleted, `vmms.exe` attempts to restore original file permissions on the corresponding `.vhdx` file.

This action is performed as:

```text
NT AUTHORITY\SYSTEM
```

The issue is that `vmms.exe` performs this permission restoration without impersonating the user.

This may allow a user to:

1. Delete the `.vhdx` file.
2. Create a native hard link pointing that file path to a protected SYSTEM file.
3. Trigger Hyper-V cleanup behavior.
4. Gain full permissions over the protected file.

---

# Vulnerability Context

The notes mention two relevant vulnerabilities:

```text
CVE-2018-0952
CVE-2019-0841
```

If the operating system is vulnerable to either issue, the hard link abuse path may be used to gain SYSTEM privileges.

If the operating system is not vulnerable, another route may be needed.

---

# Alternative Target – SYSTEM Service Binary

If the host is not vulnerable to the listed CVEs, the notes suggest targeting an application service that:

- runs as SYSTEM
- is installed on the server
- is startable by unprivileged users
- has a binary that can be replaced after permissions are modified

Example application:

```text
Firefox
```

Example service:

```text
Mozilla Maintenance Service
```

Example service binary:

```text
C:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe
```

---

# Method – Abuse Hyper-V Hard Link Behavior to Replace a SYSTEM Service Binary

## Attack Flow

1. Identify Hyper-V Administrators membership.
2. Identify a suitable SYSTEM service binary.
3. Use or modify the referenced Hyper-V hard link PoC.
4. Grant the current user full permissions over the target binary.
5. Take ownership of the target binary.
6. Replace the binary with a malicious executable.
7. Start the service.
8. Obtain command execution as SYSTEM.

> [!warning]
> Replacing service binaries is destructive and may break legitimate application functionality. Only perform this with explicit authorization.

---

# Step 1 – Identify Target File

The example target file is the Mozilla Maintenance Service binary.

```text
C:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe
```

This binary is useful in the notes because the service runs as SYSTEM and can be started.

---

# Step 2 – Modify / Run the Hyper-V EOP Script

The notes reference updating a PowerShell proof-of-concept for NT hard link abuse.

Target goal:

```text
Grant the current user full permissions over maintenanceservice.exe
```

Referenced target:

```text
C:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe
```

> [!note]
> The source notes do not include the full modified script content. Preserve the target path and behavior, but avoid assuming script changes not provided.

---

# Step 3 – Take Ownership of the Target File

After running the PowerShell script, the current user should have full control of the file.

Take ownership:

```cmd
takeown /F "C:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe"
```

Source command style:

```cmd
takeown /F C:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe
```

> [!tip]
> Use quotes around paths containing spaces to avoid parsing issues.

---

# Step 4 – Replace the Service Binary

Replace the original binary with a malicious executable named:

```text
maintenanceservice.exe
```

Target path:

```text
C:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe
```

> [!important]
> Keep a backup of the original binary if authorized and practical so the service can be restored afterward.

---

# Step 5 – Start the Mozilla Maintenance Service

Start the service to execute the replaced binary as SYSTEM.

```cmd
sc.exe start MozillaMaintenance
```

If successful, the malicious replacement binary executes in the service context.

Expected context:

```text
NT AUTHORITY\SYSTEM
```

---

# Practical Decision Tree

## Is the user in Hyper-V Administrators?

Check local group membership using a group enumeration method such as:

```cmd
whoami /groups
```

or:

```cmd
net localgroup "Hyper-V Administrators"
```

---

## Are Domain Controllers virtualized on this Hyper-V host?

If yes, treat Hyper-V admin access as domain-critical.

Potential impact:

```text
Offline extraction of NTDS.dit from a virtual Domain Controller disk
```

---

## Is the host vulnerable to the hard link EOP path?

Check whether the target operating system is vulnerable to:

```text
CVE-2018-0952
CVE-2019-0841
```

If vulnerable, the hard link permission restoration behavior may allow SYSTEM escalation.

---

## Is there a suitable SYSTEM service binary?

Look for a service that:

- runs as SYSTEM
- can be started by the current user
- has a binary path that can be targeted
- is safe to modify with authorization

Example from the notes:

```text
MozillaMaintenance
```

Binary:

```text
C:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe
```

---

# Troubleshooting

## User is not in Hyper-V Administrators

This specific Hyper-V group abuse path likely does not apply.

Check for other privilege escalation paths.

---

## Domain Controller is not virtualized

The Domain Controller disk extraction risk may not apply, but local SYSTEM escalation through Hyper-V behavior may still be relevant depending on host configuration and patch level.

---

## Hard link exploit does not work

Possible causes:

- host is patched
- CVE condition is not present
- PoC does not match OS version
- target file path is incorrect
- hard link creation failed
- permissions were not modified as expected
- endpoint protection blocked behavior

---

## Cannot replace service binary

Possible causes:

- ownership was not successfully changed
- current user lacks write permission
- file is locked
- service is running
- path was not quoted correctly
- endpoint protection blocked replacement

---

## Service does not start

Possible causes:

- malicious executable is invalid
- service expects specific arguments or behavior
- service control permissions are insufficient
- original service binary was corrupted
- endpoint protection blocked execution

---

# Cleanup

## Restore Original Service Binary

If the Mozilla Maintenance Service binary was replaced, restore the original file.

```text
C:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe
```

## Restore Permissions

Attempt to restore:

- original owner
- original ACL
- original service binary
- original service behavior

## Validate Service Health

Check service status:

```cmd
sc.exe query MozillaMaintenance
```

Optionally start the service after restoration:

```cmd
sc.exe start MozillaMaintenance
```

> [!warning]
> If cleanup cannot be completed, document the exact changes and notify the client.

---

# Command Reference

## Check current group memberships

```cmd
whoami /groups
```

## Check Hyper-V Administrators local group

```cmd
net localgroup "Hyper-V Administrators"
```

## Target Mozilla Maintenance Service binary

```text
C:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe
```

## Take ownership of target binary

```cmd
takeown /F "C:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe"
```

## Start Mozilla Maintenance Service

```cmd
sc.exe start MozillaMaintenance
```

## Query Mozilla Maintenance Service

```cmd
sc.exe query MozillaMaintenance
```

## Domain Controller NTDS.dit path

```text
C:\Windows\NTDS\ntds.dit
```

---

# Detection and Defensive Notes

## Administrative Risk

Treat Hyper-V administrators as highly privileged when they manage hosts running:

- Domain Controllers
- certificate authorities
- identity infrastructure
- sensitive application servers
- backup servers

---

## Defensive Controls

Consider:

- avoiding virtualization admin access overlap with AD admin access
- limiting `Hyper-V Administrators` membership
- monitoring VM cloning and export operations
- monitoring VHD/VHDX access
- monitoring service binary replacement
- monitoring hard link abuse indicators
- patching systems vulnerable to listed CVEs
- restricting who can start SYSTEM services
- validating service ACLs
- maintaining strong audit logs on Hyper-V hosts

---

# Key Takeaways

- `Hyper-V Administrators` have full access to Hyper-V features.
- If Domain Controllers are virtualized, Hyper-V admins should be treated as Domain Admin-equivalent.
- Hyper-V admins may clone a Domain Controller and mount the disk offline to obtain `NTDS.dit`.
- `vmms.exe` has been documented restoring `.vhdx` permissions as SYSTEM without impersonating the user.
- On vulnerable systems, native hard link abuse may lead to SYSTEM privileges.
- The notes mention `CVE-2018-0952` and `CVE-2019-0841` as relevant vulnerability contexts.
- If the direct CVE path is unavailable, a startable SYSTEM service binary may be targeted.
- The example target is Firefox’s `Mozilla Maintenance Service`.
- Replacing service binaries is destructive and requires explicit authorization.
- Cleanup should restore the original binary, owner, ACL, and service state.

---

# Related Notes

- [[Windows Privilege Escalation]]
- [[Windows Built-In Groups]]
- [[Hyper-V]]
- [[Hyper-V Administrators]]
- [[Domain Controller]]
- [[NTDS.dit]]
- [[Windows Services]]
- [[Hard Link Abuse]]
- [[CVE-2018-0952]]
- [[CVE-2019-0841]]
- [[SeTakeOwnershipPrivilege]]
- [[Access Control Lists]]

---

# Tags

#windows
#privilege-escalation
#hyper-v
#hyper-v-administrators
#system
#domain-controller
#ntds-dit
#vhdx
#hard-link
#cve-2018-0952
#cve-2019-0841
#windows-services
#mozilla-maintenance-service
#pentesting