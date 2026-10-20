# anti-analysis — when the target fights back

> Merged from: yaklang/hack-skills (anti-debug matrices with reliability ratings, obfuscation identification table, deobfuscation routing), ljagiello/ctf-skills (anti-analysis taxonomy, unpacking flow), wshobson/agents (anti-reversing troubleshooting, authorized-use gate). Credit in ATTRIBUTION.md.

> **Authorized use only.** Bypassing anti-analysis is legitimate inside authorized analysis: CTF/lab targets, your own software, malware defense, licensed security research. The same techniques on software you have no rights to (DRM circumvention, license bypass for piracy) are out of scope and refuse politely.

## Identification first (what is fighting you?)

| Symptom in IDA/Ghidra/r2 | Likely defense | Start with |
| --- | --- | --- |
| Process exits/detects when debugged | Anti-debug (matrix below) | Identify the specific check, then choose a bypass |
| Runs in VM but not on metal (or vice versa) | Anti-VM/sandbox | CPUID/MAC/timing/artifact checks |
| Crashes when Frida attached | Anti-DBI | Detects instrumentation libraries |
| Flat CFG, one giant switch | Control-flow flattening | Symbolic execution to recover the CFG |
| Only `mov` instructions | movfuscator | demovfuscator / trace-based lifting / instruction counting |
| `pushad/pushfd` then weird dispatcher | VM protector entry | `references/custom-vms.md` |
| XOR/decode loop before real code runs | Self-modifying code / string encryption | Breakpoint after decode, dump |
| Impossible always-true/false conditions | Opaque predicates + junk code | Pattern-based removal |
| All strings unreadable | String encryption | Hook the decrypt routine, or emulate it |
| No imports in IAT | Import hiding (hash resolution) | Trace `GetProcAddress` / resolve the hash function |
| High entropy sections, few imports | Packing | Unpack flow below |

Packing vs obfuscation are different problems: packing hides bytes until runtime (dynamic unwrap); obfuscation mangles logic you must untangle statically/symbolically. Confusing them wastes the first hour.

## Linux anti-debug matrix

| # | Technique | Detection method | Reliability | Bypass |
| --- | --- | --- | --- | --- |
| L1 | `ptrace(PTRACE_TRACEME)` | self-attach fails if already traced | High | LD_PRELOAD shim, NOP patch, GDB `catch syscall` |
| L2 | `/proc/self/status` TracerPid | non-zero = traced | High | hook fopen/fread, patch string |
| L3 | `/proc/self/maps` scan | debugger/instrumentation .so names | Medium | hook/filter maps output, rename agent libs |
| L4 | `rdtsc` timing | cycle-count delta | Medium | fix registers at BP, hook the timing source |
| L5 | `clock_gettime` timing | syscall-clock delta | Medium | hook clock_gettime, return controlled values |
| L6 | SIGTRAP handler | debugger swallows the raise | High | GDB `handle SIGTRAP nostop pass` |
| L8 | fork+ptrace watchdog | child attaches to parent | High | kill child, patch fork, dual-attach |
| L9-L11 | LD_PRELOAD env / parent PID / exe path checks | getenv/getppid/readlink | Low | unset/hook the call |
| L12 | INT3 (0xCC) scanning | scan .text for breakpoints | Medium | hardware breakpoints only (DR0-DR3) |
| L13-L14 | `prctl(PR_SET_DUMPABLE,0)` / ASLR-off detection | prctl / personality | Medium | hook prctl; keep ASLR on while debugging |

## Windows anti-debug matrix (selection)

| # | Technique | Detection method | Bypass |
| --- | --- | --- | --- |
| W1/W3 | `IsDebuggerPresent` / direct PEB.BeingDebugged read | PEB byte | ScyllaHide, patch PEB+0x02 |
| W4/W5 | PEB.NtGlobalFlag / heap Flags+ForceFlags | debugger leaves heap traces | ScyllaHide zeroes them |
| W6-W8 | `NtQueryInformationProcess` (DebugPort 0x07 / DebugObjectHandle 0x1E / DebugFlags 0x1F - inverted!) | NtQIP info classes | hook NtQIP; return clean values |
| W13 | TLS callbacks | code runs *before* entry point | break on TLS callback explicitly (x64dbg option) |
| W14 | `NtSetInformationThread(HideFromDebugger)` | thread invisible to debugger | hook NtSIT / NOP the call |
| W15 | DR-register check | GetThreadContext reads DR0-DR3 | hook GTC, zero DRx |
| W10-W12, W18-W19 | INT 2D / INT 3 / UD2 / invalid-CloseHandle | exception-behavior differences | handle in VEH / ScyllaHide |

Anti-VM class: CPUID hypervisor bit, MAC OUI prefixes, timing under virtualization, registry/file artifacts, scarce resources; anti-DBI class: scanning for frida-agent/gadget libraries (rename them), thread-name checks, inline-hook detection (CRC of own code). The counter is always: identify the *specific* check (it is code - find it), then neutralize that one check - blanket "hide everything" tools (ScyllaHide) exist but know what they hide.

## Bypass strategy selection

Ordered by invasiveness - prefer the least:

1. **Debugger-side hiding** (ScyllaHide, `handle SIGTRAP nostop pass`, hook-the-check via Frida) - no bytes changed
2. **Conditional-jump patching** - NOP or invert the detection branch (static patch on a *copy*, or GDB `set $pc`)
3. **Emulation** - run the guarded fragment under Unicorn where the check's environment is whatever you say it is (`references/emulation.md`) - anti-debug often does not even apply
4. **The crash-instead-of-continue lesson**: before NOPing a "detected" branch, trace where that branch goes - if it initializes state used later (heap setup, key derivation), patching the jump corrupts state and you get a crash you will misdiagnose. Patch the *comparison operand* to the clean value instead, or set the breakpoint condition to always-false.

**False positives exist** (per yaklang matrix): timing checks fire under high CPU load; TracerPid shows container artifacts; maps-scans hit legitimate security tools. Calibrate: measure the guarded path's baseline 3x, threshold = mean + 3σ, and verify the tracer name via `/proc/<pid>/comm` before concluding "debugger". A check that fires on clean systems is not evidence of analysis detection.

## Packers and unpacking flow

1. **Identify**: `file`, Detect It Easy, section entropy (`rabin2 -S`), section names (`UPX0/UPX1`), tiny IAT
2. **Known packer**: try the standard tool first - UPX: `upx -d` (works when not modified); rest usually need the generic flow
3. **Generic dynamic unpack** (the OEP run): break on the calls a packer must make to start real code - `VirtualAlloc`/`VirtualProtect` (Windows), `mmap`/`mprotect` (Linux) - run, break at the last protect-then-exec transition, dump memory from the entry point region, rebuild imports (Scylla on Windows; manual or `unipacker` flows)
4. **Dump-and-fix**: the dumped image needs import-table reconstruction (IAT repair) - auto/semi-auto first; on repeated tool failures, stop static IAT repair (record it), switch to dynamic API breakpoints. The iron rule: no infinite static IAT grinding.
5. **Verify**: the unpacked copy should show real strings, real imports, decompilable functions - entropy drop is the measurable signal

## Obfuscation: untangle order

1. **Classify** with the table above (flattening / opaque predicates / junk / SMC / MBA / VM)
2. **Control-flow flattening**: symbolic execution (angr) to recover the real CFG; or trace-driven reordering (run it, log the dispatcher states, rebuild the order)
3. **Opaque predicates + junk**: pattern-based dead-code removal (D-810 for IDA decompiler, GOOMBA, Miasm-based passes); MBA expressions → identify (arity-of-boolean idioms) then simplify with msynth-style synthesis *with a random-input equivalence check* - never trust a simplifier without verifying equivalence on random inputs
4. **String encryption**: hook the decrypt routine at runtime (Frida), or emulate the routine under Unicorn and dump outputs in bulk
5. **SMC**: breakpoint *after* the decode loop, dump, work on the decoded image; re-dump if more layers appear (multi-stage: repeat until entropy is flat and imports look real)
6. When obfuscation wins on every static front: go dynamic/emulation and treat the obfuscated region as a black box with observable I/O
