# targets — route table for what you are actually looking at

> Merged from: zhaoxuya520/reverse-skill (per-target subskill map), ljagiello/ctf-skills (languages/platforms corpus), wshobson/agents (malware/firmware analyst workflows). Credit in ATTRIBUTION.md.

Triage named the target; this table routes it. Entry commands only - depth lives in the other references.

## Native binaries

| Target | Tell | Entry moves |
| --- | --- | --- |
| ELF (Linux) | `file` | r2/readelf/objdump; `checksec`; Ghidra if decompilation needed |
| PE (Windows) | `file`, DOS stub | x64dbg + ScyllaHide; dumpbin /imports; TLS callbacks before entry |
| Mach-O (macOS) | `file`, fat/universal headers | lldb; note: linked binaries carry no DWARF - debug info lives in .o/.dSYM bundles, point dwarfdump there |
| Kernel module (.ko/.sys) | module metadata | IDA/Ghidra with driver-loader awareness; ioctl interface reconstruction |
| Static Go binary | `Go buildinfo`, fat symbols | GoReSym (recovers buildinfo, funcs, types); panic strings; channel/goroutine idioms |
| Rust binary | mangled symbols, panic strings | `rustfilt`/demangle; Option/Result/Vec idioms; serde hints at schema recovery |
| Swift | `_Tt` mangling | `swift demangle`; protocol witness tables for behavior |
| C++ | vtables, RTTI, STL shapes | vtable reconstruction; `c++filt`; the calling-convention/struct rules in `references/static.md` |
| Kotlin/JVM | .class/.jar | jadx (Java-level first), only drop to .so when JNI appears |

## Managed & bytecode targets

| Target | Tell | Entry moves |
| --- | --- | --- |
| Android APK | zip + classes.dex | jadx/apktool for Java+resources; smali edits via apktool rebuild; native side: `.so` in lib/ → r2/Ghidra + Frida (hook JNI `RegisterNatives` when names are dynamic) |
| .NET (C#) | CLR metadata | dnSpyEx/dnlib, de4dot for deobfuscation; ConfuserEx/SmartAssembly/.NET Reactor have dedicated unpackers; IL patching via dnlib |
| Python bytecode | `.pyc`, magic bytes | uncompyle6/decompyle3 (≤3.8); pycdc build-from-source for 3.9+; Pyarmor → static unpack (find the decryption stub) or runtime dump; opcode remapping = custom disassembler work |
| WASM | `\0asm` magic | wasm2wat (wat2wasm back), wasm-decompile; treat as a stack VM (`references/custom-vms.md`) |
| Unity game | `global-metadata.dat`, IL2CPP | IL2CPP: Il2CppDumper recovers types/methods → Ghidra script; Mono: dnSpy on Assembly-CSharp.dll |
| Electron app | app.asar | `npx asar extract`; JS is readable; native node modules → normal binaries |
| Java JAR | MANIFEST, .class files | jadx; obfuscated (ProGuard)? → deobfuscate by call-graph shape |

## Firmware & embedded

binwalk signature scan → extract filesystem (squashfs/cramfs) → find the interesting binary (ARM/MIPS/RTOS) → emulation: `qemu-aarch64-static`/`qemu-mips-static` user-mode, or Qiling for syscall needs, or Unicorn raw-load for bare-metal/MMIO fragments. Hardware extraction when the image is not on a filesystem you can read: UART/JTAG/SPI dumps. Automotive CAN bus captures: treat as a protocol (`references/protocols.md`).

## Malware (defensive, isolated lab only)

Phase discipline (per wshobson malware-analyst): **safe handling first** - disposable VM, snapshot, INetSim/fake network, hashes recorded, chain of custody. Then: static triage (strings, imports, packer ID, YARA matching) → behavioral run (API monitoring, file/registry/process/network watch) → unpacking (`references/anti-analysis.md` flow) → deep analysis with the standard references → IOC extraction and a defensive writeup. Malware analysis is the same reversing loop with higher stakes for isolation: nothing from the sample touches a real network or a production system.

## Games / anti-cheat

Game clients: Cheat Engine for memory scouting; Unity/IL2CPP route above; game protocol RE via packet capture (`references/protocols.md`). Anti-cheat internals are kernel-mode territory - the defensive/authorized framing applies; game-hacking on live third-party services is out of scope.

## The routing habit

When a target mixes layers (APK with a native packer inside; Electron wrapper around a C++ core; firmware that boots a Java card), name every layer out loud and pick the layer where the question lives. Most wasted RE hours are spent analyzing the wrong layer of a stacked target.
