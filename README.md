# reversing

A big skill for **reverse engineering** — the discipline AI agents need when there is no source: CTF binaries, crackmes, unknown APKs, packed executables, custom VMs, firmware, protocols, and defensive malware analysis. Born from merging the best RE skills found in the open ecosystem (the stage-gated router packs, the CTF technique corpora, the agent methodology frameworks, the VM playbooks, and the emulation-first toolkits) into one discipline: identify before interpreting, spend cheap probes first, let the program compute the answer, and prove every claim with an address.

```
Most reversing work is not "read all the assembly". It is a loop: triage what
the target IS, probe the cheapest observable, form one hypothesis, test it,
and escalate only when the cheap route stalls. This skill makes both failure
modes impossible: skipping triage, and grinding a stalled approach.
```

## O que é (PT-BR)

Uma big skill de **engenharia reversa** para agentes de IA — para quando não existe código-fonte: binários de CTF, crackmes, APKs desconhecidos, executables empacotados, VMs customizadas, firmware, protocolos e análise defensiva de malware. Síntese profissional das melhores skills de REA do ecossistema aberto (zhaoxuya520/reverse-skill, ljagiello/ctf-skills, wshobson/agents, yaklang/hack-skills, p4nda0s/reverse-skills + conceito do Trail of Bits), com licenças verificadas 1:1 e crédito em todas as superfícies — o mesmo processo das skills anteriores da coleção.

## What's inside

| Piece | What it does |
| --- | --- |
| **The reversing contract** (`SKILL.md`) | Six rules: identify before you interpret, cheap before expensive, let the program compute, answer the question asked, prove it or label it, respect the boundary |
| **The method** (`references/the-method.md`) | The stage-gated workflow (triage → static → dynamic → synthesis), the quick-wins ladder, the evidence ladder (E1-E4), honesty rules ("never pipe triage through head"), loop prevention, completion checklist |
| **Static** (`references/static.md`) | Disassembly and decompilation patterns, data-structure recovery from access patterns, function signatures via calling conventions, symbols and DWARF shortcuts, Ghidra/IDAPython/r2pipe scripting |
| **Dynamic** (`references/dynamic.md`) | GDB discipline (PIE-safe breakpoints), strace/ltrace tracing, Frida hooking (capture the comparison, loader-aware), the memory-dumping killer move, comparison direction, decoys, oracles |
| **Emulation** (`references/emulation.md`) | The engine decision tree, Unicorn raw-load-first + environment simulation, snapshot brute-force oracles, Qiling, angr explore, Z3 constraint transcription, instruction-count side channels |
| **Anti-analysis** (`references/anti-analysis.md`) | Linux/Windows anti-debug matrices with reliability ratings and false-positive notes, anti-VM/anti-DBI, packers and the unpacking flow, obfuscation identification table, MBA, bypass strategy selection |
| **Custom VMs** (`references/custom-vms.md`) | The VM playbook: dispatcher identification, opcode mapping, custom disassembler, maze challenges, VMProtect/Tigress, the full decision tree |
| **Targets** (`references/targets.md`) | Route table for PE/ELF/Mach-O, Go/Rust/Swift/C++, APK/Android, .NET, Python bytecode, WASM, Unity, Electron, firmware, malware (defensive lab discipline) |
| **Protocols** (`references/protocols.md`) | Wire and file-format RE: capture, dissection, field identification, spec writing, parser/generator validation, boofuzz |
| **The report** (`references/the-report.md`) | Evidence-chain reporting: reproducible commands, addresses not vibes, confidence labels, open questions |

Plus a Cursor rules export at `exports/reversing.mdc` and 12 eval cases in `evals/evals.json`.

## Install

**Claude Code / GLM / any SKILL.md host:**
```bash
git clone https://github.com/lucasgabrieldevgg/reversing.git ~/.claude/skills/reversing
```

**Cursor:** copy `exports/reversing.mdc` to `.cursor/rules/reversing.mdc`.

**ClawHub / skills.sh:** the folder is standard skill layout — publish as-is.

## The three modes (the center of mass)

**Triage** runs on first contact: name the format/arch/packer/language out loud, run the quick-wins ladder (strings → strace/ltrace → hook the comparison → symbolic execution), and route to the right deep technique. **Deep-dive** answers "how does this work?" — static reading + dynamic runs + emulation in a loop, with loop prevention as a hard rule (the same method failing twice means switch approaches), fighting anti-analysis only when it actually blocks. **Solve** extracts the answer itself: memory-dump the computed flag at the final comparison, feed the check to Z3, brute-force through an emulation snapshot oracle, or write the custom disassembler for the VM bytecode.

The references are a shelf, not a script: a typical task loads 1–3 of them.

## Scope & safety

This skill analyzes targets you are authorized to analyze: your own binaries, CTF/lab sandboxes, licensed interoperability and security research, and malware handled defensively in isolated disposable environments. It does not bypass commercial licensing or DRM, crack protections on software you have no rights to, target third-party systems, or weaponize findings. That boundary is stated in the skill itself and enforced by its evals.

## Collection

Part of the skill collection: **theskill** (craftsmanship: process + code + prose), **design**, **research**, **motion**, **video**, **security**, **threed**, **projects**, and this one — each a big skill built the same way: survey the ecosystem, merge the best, credit everything.

## Attribution

`reversing` is an original synthesis (MIT) merging upstream material from `zhaoxuya520/reverse-skill` (MIT), `ljagiello/ctf-skills` (MIT), `wshobson/agents` by Seth Hobson (MIT), `yaklang/hack-skills` by VillanCh (MIT), and `p4nda0s/reverse-skills` (MIT, declared in README — caveated), with concept-only inspiration from Trail of Bits' `dwarf-expert` (CC BY-SA 4.0). 6 GitHub repos were cloned and studied in full plus 9 ClawHub-native skills surveyed; 5 were merged, the rest are documented and credited in [`ATTRIBUTION.md`](ATTRIBUTION.md).

## License

MIT for this repo's original content. Upstream material retains its own licenses — see ATTRIBUTION.md.
