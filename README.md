# Ruy-Lopez

<p align="center">
<img src="https://github.com/S3cur3Th1sSh1t/Ruy-Lopez/blob/main/images/Ruy_Lopez_Opening.jpg?raw=true" alt="Ruy Lopez Opening" width="400" height="400">
</p>

Endpoint Detection and Response systems (EDRs) are like the white player in a Chess game:

- They do the first move with hooks loaded directly via the kernel
- The EDR DLL is typically loaded directly after `ntdll.dll`

But what if we can prevent their DLL from being loaded at all? Do we get the white player and can do the first moves (for the new process at least)? 



This repository contains the Proof-of-Concept(PoC) for a new approach to completely prevent DLLs from being loaded into a newly spawned process.
The initial use-case idea was to block AV/EDR vendor DLLs from being loaded, so that userland hooking based detections are bypassed.

</br></br>
<ins>The simplified workflow of the PoC looks as follows:</ins>

<p align="center">
<img src="https://github.com/S3cur3Th1sSh1t/Ruy-Lopez/blob/main/images/Idea.png" alt="Workflow" width="700" height="450">
</p>

The SubFolder `HookForward` contains the actual PIC-Code which can be used as EntryPoint for a hooked `NtCreateSection` function. `Blockdll.nim` on the other hand side spawns a new Powershell process in suspended mode, injects the shellcode into that process and remotely hooks `NtCreateSecion` to `JMP` to our shellcode. As this is a PoC, *only* `amsi.dll` is being blocked in a new Powershell process, which effectively leads to an AMSI bypass. But the PoC was also tested against multiple EDR vendors and their DLLs without throwing an alert or without being blocked **before** releasing it. I expect detections to come up after releasing it here.

## Challenges / Limitations

- When customizing this PoC, you can only use `ntdll.dll` functions in the PIC-Code, as the process is not fully initialized yet when the hook occurs and therefore only `ntdll.dll` is loaded. Other DLLs also cannot be loaded by the shellcode, because process initialization has to take place first.
- This PoC can only prevent DLLs from being loaded which are not injected but instead loaded normally. Some vendors inject specific or single DLLs.

## Setup

On linux, the PIC-Code was found to be compiled correctly with `mingw-w64` version `version 10-win32 20220324 (GCC)`. With that version installed, the shellcode can be compiled with a simple `make` and extracted from the `.text` section via `bash extract.sh`. Newer `mingw-w64` versions, such as 12 did lead to crashes for me, which I'm currently not planning to troubleshoot/fix.

If you'd like to compile from Windows, you can use the following commands:

```powershell
as -o directjump.o directjump_as.asm
gcc ApiResolve.c -Wall -m64 -ffunction-sections -fno-asynchronous-unwind-tables -nostdlib -fno-ident -O2 -c -o ApiResolve.o -Wl,--no-seh
gcc HookShellcode.c -Wall -m64 -masm=intel -ffunction-sections -fno-asynchronous-unwind-tables -nostdlib -fno-ident -O2 -c -o HookShellcode.o -Wl,--no-seh
ld -s directjump.o ApiResolve.o HookShellcode.o -o HookShellcode.exe
gcc extract.c -o extract.exe
extract.exe
```

You also need to have [Nim](https://nim-lang.org/) installed for this PoC.

<ins>After installation, the dependencies can be installed via the following oneliner:</ins>

```nim
nimble install winim
```

<ins>The PoC can than be compiled with:</ins>

```nim
nim c -d:release -d=mingw -d:noRes BlockDll.nim # Cross compile
nim c -d:release BlockDll.nim # Windows
```

<p align="center">
<img src="https://github.com/S3cur3Th1sSh1t/Ruy-Lopez/blob/main/images/PoC.png" alt="PoC" width="750" height="375">
</p>


## OPSec improvement ideas

- Userland-hook evasion for injection from the host process
- RX Shellcode (needs some PIC-code changes)
- RX permissions for the hooked function memory (see comments)
- Use hashing instead of plain APIs to block
- Use hardware breakpoints instead of hooking

## CREDITS

- [Ceri Coburn](https://twitter.com/_EthicalChaos_) - Help all over the PoC
- [Sven Rath](https://twitter.com/eversinc33) - General idea, review and initial PoC inspiration
- [Alejandro Pinna](https://twitter.com/frodosobon) - Initial idea came after reading [his blogpost](https://waawaa.github.io/es/amsi_bypass-hooking-NtCreateSection/) 
- [Charles Hamilton](https://twitter.com/MrUn1k0d3r) - QA help when writing PIC code
- [Chetan Nayak](https://twitter.com/NinjaParanoid) - QA help when writing PIC code + the Chess idea [^1]

[^1]: https://bruteratel.com/release/2022/08/18/Release-Scandinavian-Defense/
