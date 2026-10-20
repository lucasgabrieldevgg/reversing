---
name: reversing
description: "Reverse engineering for AI agents: understand how compiled, obfuscated, packed, or virtualized software works - from a CTF binary to an unknown APK, firmware image, custom VM, or network protocol. Three modes in one skill. MODE 1 - TRIAGE (default entry): identify the target before interpreting it (file type, architecture, compiler, packer, language), run the cheap-probes-first ladder (strings, then strace/ltrace, then a hook, then symbolic execution - escalate only when the cheap probe fails), and route to the right deep technique instead of guessing. MODE 2 - DEEP-DIVE: static + dynamic + emulation loops - disassembly and decompilation patterns, structure and type recovery, debugger discipline (PIE-safe breakpoints), Frida hooking (hook strcmp/memcmp to capture expected values), Unicorn raw-load emulation, Qiling, angr, Z3 constraint solving - with loop prevention (same method failing twice means switch approaches, static to dynamic to emulation) until the question is answered. MODE 3 - SOLVE: extract the answer itself - memory-dump the computed flag at the final comparison, feed the check logic to Z3, brute-force through an emulation oracle, or write a custom disassembler for the bytecode VM. Covers PE/ELF/Mach-O/DEX/WASM, Go/Rust/Swift/C++/Python-bytecode targets, Android APKs and JNI, custom VMs and bytecode interpreters (dispatcher identification, opcode mapping, VMProtect/Themida/Tigress), anti-debug/anti-VM/anti-DBI matrices with bypass strategies, packers and unpacking, obfuscation identification and MBA simplification, network protocol reverse engineering, embedded firmware, defensive malware analysis in isolated labs, and evidence-grade reporting with addresses, offsets, and reproducible commands. Use this skill whenever the user asks to 'reverse engineer', 'decompile', 'disassemble', 'crack this crackme', 'solve this reversing challenge', 'understand what this binary does', 'find the flag', 'unpack this', 'bypass anti-debug', 'analyze this APK/so', 'figure out this protocol', 'recover the algorithm', 'what does this malware do' - and proactively whenever a task stalls on understanding opaque code that has no source, an undocumented file format, or an unknown protocol."
version: 1.0.0
license: MIT for the synthesis; upstream content keeps its own licenses - see ATTRIBUTION.md
---

# reversing — understand software that has no source

`reversing` works the way a senior reverse engineer would: identify the target before theorizing about it, spend cheap probes before expensive ones, let the program compute the answer whenever possible, and never claim a behavior that cannot be shown with an address and a command output.

Most reversing work is not "read all the assembly". It is a loop: triage what the target *is*, probe the cheapest observable, form one hypothesis, test it, and escalate only when the cheap route stalls. AI agents that skip triage waste hours decompiling a UPX-packed binary byte-by-byte; agents that skip escalation grind the same failed static read in circles. This skill exists to make both failures impossible.

## The reversing contract

1. **Identify before you interpret.** `file`, arch, compiler, packer, and language are named before the first theory about behavior. A "weird custom crypto" that turns out to be a packed .NET assembly is a triage failure, not a analysis failure.
2. **Cheap before expensive.** strings → strace/ltrace → hook the comparison → symbolic execution → manual decompiler reading. Escalate one rung only when the current rung fails. Most CTF-grade targets die on rung one or two.
3. **Let the program compute.** The fastest route to an answer is usually: run to the final comparison, enter any input of the right length, dump the computed expected value from memory. Reversing the transform is the fallback, not the default.
4. **Answer the question asked.** Reverse the path between input and the question - the check function, the protocol handshake, the config parser - not the whole binary. Exhaustive understanding of irrelevant code is scope creep with extra steps.
5. **Prove it or label it.** Every behavior claim is a command output with an address/offset, or it is explicitly labeled a hypothesis with a confidence level. "Probably RC4" is a hypothesis until a key-schedule loop is pointed at.
6. **Respect the boundary.** Authorized targets only: your own binaries, CTF/lab sandboxes, samples you are licensed to analyze, interoperability research. Work on copies, stay offline by default, and never let the task drift to third-party systems without saying so.

## Three modes

| Mode | When it runs | What it does |
| --- | --- | --- |
| **Triage** (default entry) | First contact with any unknown target | Identify format/arch/packer/language, run the quick-wins ladder, route to the right deep technique |
| **Deep-dive** | "How does this work?", "recover the algorithm / format / protocol" | Static reading + dynamic runs + emulation in a loop, fighting anti-analysis when it blocks, until the question is answered |
| **Solve** | The goal is an answer: flag, key, serial, license check | Extract it directly - memory-dump at the comparison, Z3 the check, brute-force an emulation oracle, disassemble the VM bytecode |

## Routing table

Read only what the task needs - do not preload the whole skill:

| When | Read |
| --- | --- |
| First contact with any unknown binary/target; the overall workflow | `references/the-method.md` |
| Reading disassembly, decompiling, recovering structs/types/symbols | `references/static.md` |
| Debugging, tracing, hooking, dumping memory at breakpoints | `references/dynamic.md` |
| Emulating fragments, symbolic execution, snapshot brute-force oracles | `references/emulation.md` |
| The target fights back: anti-debug, anti-VM, anti-DBI, packers, obfuscation | `references/anti-analysis.md` |
| Custom VM / bytecode interpreter / VM protector (VMProtect, Themida) | `references/custom-vms.md` |
| Non-native targets: APK/Android, WASM, .NET, Go/Rust/Swift, firmware, games, malware | `references/targets.md` |
| An unknown network protocol or file format | `references/protocols.md` |
| Writing up findings, evidence chain, reproducible reports | `references/the-report.md` |

## The loop in one breath

**Identify** (file, arch, packer, language - out loud) → **probe cheap** (strings, traces, a plain run with test input) → **pick the path** (read it statically, run it dynamically, or emulate it) → **fight the defenses** (only when they actually block you) → **extract the answer** (dump it, solve it with Z3, or brute-force it) → **report with evidence** (addresses, commands, confidence labels).

## Scope & safety — read first, enforce always

This skill analyzes targets the user is authorized to analyze: their own binaries, CTF challenges and crackmes, lab samples, licensed software for interoperability and security research, and malware handled defensively in isolated, disposable environments (snapshot taken, no production access, controlled or simulated networking). It does not help bypass licensing or DRM on software the user has no rights to, crack commercial protections for piracy, evade detection on third-party systems, or weaponize anything found. Local sandbox/CTF contexts are the default assumption - do not interrogate the user for repeated confirmations - but any drift toward a third-party target without stated authorization stops and says so. Unknown samples execute only inside the sandbox, on copies, never on originals.

## Load-on-demand discipline

The references are a shelf, not a script: a typical task touches 1-3 of them (the-method plus whatever the target turned out to be). Loading everything for a strings-level challenge wastes context and dilutes attention. Route from the table above, and go deeper only when the target demands it.

## Sources & attribution

This skill is an original synthesis: the stage-gated workflow, evidence discipline, and completion checklist are merged from `zhaoxuya520/reverse-skill` (MIT); the CTF technique corpus (quick-wins ladder, memory-dumping strategy, comparison-direction doctrine, emulation references, honesty rules) from `ljagiello/ctf-skills` (MIT); the 4-phase methodology, binary-analysis pattern catalog, protocol workflow, and anti-reversing troubleshooting from `wshobson/agents` (MIT, Seth Hobson); the VM/bytecode playbook, anti-debug matrices with reliability ratings, and obfuscation identification from `yaklang/hack-skills` (MIT, VillanCh); the raw-load-first Unicorn principle and environment-simulation strategy from `p4nda0s/reverse-skills` (MIT, declared in README); the look-up-the-standard discipline and DWARF verification concepts from Trail of Bits' `dwarf-expert` (CC BY-SA 4.0, concept only). Full details, licenses, and the surveyed-but-excluded list: [ATTRIBUTION.md](ATTRIBUTION.md).
