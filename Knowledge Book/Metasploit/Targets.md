# Metasploit Targets

## Overview

In Metasploit, **targets** are the specific operating system / application versions that an exploit module is designed to work against.

They matter because many exploits depend on:

- exact OS version
- service pack level
- application version
- architecture
- language pack
- memory layout / return addresses

A module may support:

- one generic target
- several very specific targets
- an `Automatic` mode that tries to detect the best fit

---

# 1. What `show targets` Does

The `show targets` command displays the list of supported targets for the **currently selected exploit module**.

If you run it without first selecting an exploit module:

```bash
show targets
```

You will see:

```text
[-] No exploit module selected.
```

---

# 2. Where Targets Appear

Targets are shown in two places most often:

## In `options`

```bash
options
```

Example section:

```text
Exploit target:

   Id  Name
   --  ----
   0   Automatic
```

---

## In `show targets`

```bash
show targets
```

Example:

```text
Exploit targets:

   Id  Name
   --  ----
   0   Automatic
   1   IE 7 on Windows XP SP3
   2   IE 8 on Windows XP SP3
   3   IE 7 on Windows Vista
   4   IE 8 on Windows Vista
   5   IE 8 on Windows 7
   6   IE 9 on Windows 7
```

---

# 3. Automatic Target Selection

Many modules default to:

```text
0 Automatic
```

This means Metasploit will try to determine the correct target automatically based on:

- service fingerprinting
- exploit logic
- returned responses
- known detection checks

This is convenient, but not always best.

---

## When Automatic is Fine

Use automatic when:

- you have limited version info
- the module is reliable
- target detection is supported well
- lab speed matters more than manual precision

---

## When Manual Target Selection is Better

Set the target manually when:

- you already know exact OS / application version
- exploit stability matters
- automatic detection is unreliable
- the module depends on exact ROP / memory layout

---

# 4. Viewing Target Details with `info`

Use:

```bash
info
```

This is one of the best first steps when using a new module.

It helps you understand:

- what the exploit does
- which systems are supported
- whether target-specific dependencies exist
- whether `check` is available
- if special prerequisites are needed

---

## Example

```bash
info
```

Might show:

```text
Available targets:
  Id  Name
  --  ----
  0   Automatic
  1   IE 7 on Windows XP SP3
  2   IE 8 on Windows XP SP3
  3   IE 7 on Windows Vista
  4   IE 8 on Windows Vista
  5   IE 8 on Windows 7
  6   IE 9 on Windows 7
```

---

# 5. Setting a Target

If you know the exact target, choose it manually.

## Example

```bash
set target 6
```

Output:

```text
target => 6
```

This tells Metasploit to use:

```text
IE 9 on Windows 7
```

rather than relying on auto-detection.

---

# 6. Example Workflow

## Select Module

```bash
use exploit/windows/browser/ie_execcommand_uaf
```

## View Options

```bash
options
```

## Show Supported Targets

```bash
show targets
```

## Pick Specific Target

```bash
set target 6
```

---

# 7. Why Targets Matter

Exploit success often depends on small differences such as:

- Windows XP vs Windows 7
- IE 8 vs IE 9
- x86 vs x64
- service pack levels
- language versions
- installed dependencies
- memory protection differences

A mismatch can cause:

- exploit failure
- crash without shell
- unstable session
- wrong ROP chain
- incorrect return address use

---

# 8. Target Types Can Vary By

Metasploit targets may differ based on:

| Variable | Why It Matters |
|----------|----------------|
| OS version | different memory layout / APIs |
| service pack | shifted offsets / patched code |
| application version | different vulnerable code path |
| architecture | x86 vs x64 shellcode / memory |
| language pack | different addresses in modules |
| installed libraries | affects ROP chain validity |
| hooks / security software | may shift addresses |

---

# 9. Example: Browser Exploit Targeting

For browser exploits, target options may be very specific:

```text
IE 7 on Windows XP SP3
IE 8 on Windows Vista
IE 9 on Windows 7
```

Why so specific?

Because client-side exploits often depend heavily on:

- exact browser version
- exact OS version
- exact ROP gadgets
- exact library layout

---

# 10. Reading the Module Description Matters

The `info` output often includes important target dependencies.

Example ideas you might see:

- requires JRE 1.6.x or below
- requires `msvcrt` present
- exploit valid only on a certain service pack
- target must expose named pipe
- language-specific offset dependency

These details are critical for reliable exploitation.

---

# 11. Practical OSCP Guidance

Before setting a target, try to answer:

- what OS is this?
- what version is this?
- what architecture is this?
- what service pack is this?
- what app version is this?
- are any dependencies mentioned in `info`?

Useful enumeration sources:

```bash
nmap -sV
nmap -O
banner grabbing
manual web fingerprinting
SMB enumeration
application version strings
```

---

# 12. Related Exploit Development Note

In more advanced exploit work, targets may differ because of:

- return addresses
- `jmp esp`
- `pop pop ret`
- SEH behaviour
- ROP chain requirements

To identify correct target manually, you may need to:

- obtain target binaries
- inspect loaded modules
- find reliable addresses
- use tools like `msfpescan`

This becomes more important in manual exploit development than in routine Metasploit use.

---

# 13. Common Commands

## Show targets

```bash
show targets
```

## View full module info

```bash
info
```

## Set target automatically

```bash
set target 0
```

## Set specific target

```bash
set target 6
```

## Re-check options

```bash
options
```

---

# 14. Quick Example

```bash
use exploit/windows/browser/ie_execcommand_uaf
info
show targets
set target 6
options
```

---

# 15. Key Takeaways

- targets define which OS / application versions the exploit supports
- `Automatic` is convenient but not always optimal
- `info` should be one of your first commands with a new module
- manual target selection is better when you know exact version details
- exploit failure may simply mean the wrong target was selected

---

# Cheatsheet

## Commands

```bash
show targets
info
set target 0
set target <id>
options
```

## Workflow

```bash
use <exploit>
info
show targets
set target <id>
options
run
```

---

# Related Notes

- Metasploit overview
- Metasploit modules
- payload selection
- exploit workflow
- Meterpreter basics
- exploit development

---

# Tags

#metasploit #targets #exploitation #msfconsole #oscp