# the-method — the workflow, the evidence ladder, the honesty rules

> Merged from: zhaoxuya520/reverse-skill (stage gates, evidence discipline, completion checklist), ljagiello/ctf-skills (problem-solving workflow, quick-wins ladder, honesty rules), wshobson/agents (4-phase methodology). Credit in ATTRIBUTION.md.

## Stage gates

The workflow is **Triage → Static → Dynamic → Synthesis**. Each stage has an exit condition; do not drift between stages without one:

| Stage | Goal | Exit when |
| --- | --- | --- |
| **Triage** | Know what the target *is* | Format, arch, compiler, packer, language named; route chosen |
| **Static** | Understand structure without running | Entry points mapped, interesting functions located, hypothesis formed |
| **Dynamic** | Confirm/refute the hypothesis by running it | Observed behavior matches or replaces the static read |
| **Synthesis** | Answer + evidence | The question is answered with a reproducible command or a labeled open question |

Stuck in a stage? That is an exit condition too: the same method failing twice means **switch approaches** (static ↔ dynamic ↔ emulation), not a third identical attempt.

## Triage protocol (minutes 0-5)

```bash
file target                 # type, arch, statically/dynamically linked
checksec --file=target      # mitigations (PIE, NX, canaries, RELRO)
strings -n 8 target | grep -inE "flag|ctf\{|key|secret|password|error|usage"
rabin2 -z target 2>/dev/null | grep -i flag    # ELF strings via r2
xxd target | grep -i flag                       # hex-level search
./target AAAA; echo test | ./target              # plain run with test input
```

Identify: **format** (PE/ELF/Mach-O/DEX/WASM/bytecode), **arch** (x86-64/ARM64/MIPS/RISC-V), **compiler** (GCC/Clang/MSVC/Go/Rust - symbol mangling tells), **packer** (UPX section names, entropy, `Detect It Easy`), **language** (Go buildinfo, Rust panic strings, .NET metadata, Python magic bytes). Route per `references/targets.md`.

## The quick-wins ladder (escalate one rung at a time)

1. **Strings** - many targets carry their answer in plaintext (or one XOR away).
2. **strace/ltrace** - the syscall/library-call trace often reveals the whole check flow (`ltrace ./target` shows `strcmp` arguments live).
3. **Hook the comparison** - Frida on `strcmp`/`memcmp`/`strncmp` captures the *expected* value without reading any assembly.
4. **Symbolic execution** - angr solves a large class of flag-checkers outright (constrain input format, explore paths to the success branch).
5. **Emulate** - Qiling/Unicorn when the target will not run here (foreign arch, hostile anti-debug, firmware).
6. **Manual decompiler reading** - Ghidra/IDA/r2 for everything else. You arrive here with a map, not blind.

Escalation rule: the cheap rung failed *for a reason you can name* (no strings, trace shows nothing, hook never fires) - not just "it did not finish in 10 seconds".

## Honesty rules for triage output

**Never pipe triage output through `head` on the first pass.** A truncated view is never a valid basis for a "no flag here" conclusion - the real match may live at line 22 of the filtered list. Grep for the signal instead of capping the volume:

```bash
strings target | grep -inE "flag|ctf\{|congrat|correct|wrong"   # filter, don't truncate
objdump -d --disassemble=main target                            # whole function, always
```

If output is genuinely too large, page it deliberately (`sed -n '200,400p'`) and *say which slice you read*. An unread tail must never become a negative conclusion. This holds even when the user urges "just show me the first lines".

## The evidence ladder

Every claim lands on one rung, and the rung is stated in the output:

| Rung | Claim type | Form |
| --- | --- | --- |
| E1 | **Executed evidence** | A command was run; here is its output (with address/offset) |
| E2 | **Derived evidence** | Read from code/decompilation; here is the function and the line of reasoning |
| E3 | **Labeled hypothesis** | Pattern matches X; confidence: low/medium/high; here is what would confirm it |
| E4 | **Open question** | Cannot determine without Y; here is the next probe |

"Fixed" and "solved" require E1. Anything else is honestly labeled - the report standard in `references/the-report.md` enforces the same ladder.

## Loop prevention (self-supervision)

- Same tool + same arguments called twice with no progress → the third call is forbidden; change approach.
- Same static read circling → go dynamic; same dynamic run circling → go static or emulate.
- One command repeated 3+ times → stop, re-read the last error, understand it before acting.
- A tool "missing" → check the real install path (`which`, `command -v`), never guess paths; after 2 failed auto-install attempts, emit manual install steps instead of retrying.
- Tool-call budget for a subtask ~30 → surface progress to the user, ask whether to continue.
- Approaching a decision you cannot make → present a numbered menu of the options that *materially change the next action*, not a status recap.

## Completion checklist (before claiming done)

- [ ] Did I *execute* each workflow stage I claim (not just read about it)?
- [ ] Is every finding backed by E1/E2 evidence with addresses or offsets - not "some function"?
- [ ] Are all commands reproducible (a copy-paste script or transcript, not descriptions)?
- [ ] Are uncertain conclusions labeled with confidence?
- [ ] Did I answer the question that was asked (not an adjacent, easier one)?
- [ ] Did I keep the work on a copy, inside the sandbox, on authorized targets only?

## Known triage quick-judgments

- **Comparison direction (critical).** Two shapes exist: `transform(input) == stored_target` (reverse the transform) and `transform(stored_target) == input` (the answer IS the transformed data - apply the transform to the stored blob, do not invert anything). Misreading this inverts hours of work.
- **Decoy flags.** Multiple comparison targets in sequence with different success messages = decoys before the real check. Break at the *final* comparison.
- **Common crypto shapes.** XOR single-byte (try all 256), XOR with known plaintext (`flag{`, `CTF{`), RC4 with a hardcoded key, permutation+XOR layers, XOR with position index (`^ i`, `^ (i & 0xff)`) over a repeating key. Constants identify algorithms: `0x5A827999` SHA-1, `0x67452301` MD5/SHA, S-boxes mean AES/DES kin.
- **Entropy tells packing.** A high-entropy `.text` with tiny imports = packed; unpack first (`references/anti-analysis.md`) or go dynamic immediately. Reading packed disassembly is wasted effort by definition.
