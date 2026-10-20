# emulation — Unicorn, Qiling, angr, Z3: make the code run on your terms

> Merged from: p4nda0s/reverse-skills rev-unicorn-debug (raw-load-first principle, environment simulation), ljagiello/ctf-skills (unicorn-emulation, tools-emulation, angr workflows), yaklang/hack-skills (VM solving via emulation). Credit in ATTRIBUTION.md.

## Why emulate

Emulation is the third leg after static and dynamic: run *fragments* without the whole program, run foreign architectures natively, run hostile binaries without anti-analysis touching your real debugger, and - uniquely - rewind/replay state (snapshot brute-force oracles are impossible in a real process). Sometimes it is simply easier to rev with just Unicorn + the binary + the assembly than a full GDB/angr session.

## Choosing the engine

| Engine | Sweet spot | Decision notes |
| --- | --- | --- |
| **Unicorn** | A function, a shellcode blob, a VM handler; firmware MMIO; snapshot oracles | Raw CPU emulation only - you supply all context. Fastest to start, most manual |
| **Qiling** | Whole binaries with OS needs (syscalls, dynamic loader) on a foreign arch | Unicorn + OS layer; use when the fragment approach cannot supply libc/loader state |
| **angr** | Path exploration, constraint solving ("find the input that reaches success") | Symbolic execution; slow on huge state spaces, unbeatable on check logic |
| **QEMU** | Full system / user-mode emulation of foreign arch | Heavier; use when you need a whole environment, not an analysis harness |
| **Triton** | Dynamic symbolic execution with a concrete trace | When you want symbolic reasoning over an observed path |

Rule of thumb: **Unicorn for fragments, Qiling for programs, angr for questions.**

## Unicorn: the raw-load-first principle (per p4nda0s)

1. **Load the file raw** - do NOT parse ELF/PE/Mach-O headers first. Map bytes directly into memory at the right base. You emulate *specific functions*, not whole binaries. If raw loading faults (code references unmapped segments), parse minimally - map only the segments the target touches.
2. **Identify context dependencies**: what does the fragment call - libc (`malloc`, `memcpy`, `printf`), JNI (`GetStringUTFChars`), raw syscalls? Hook them and answer from Python.
3. **Callback everything**: `UC_HOOK_CODE`/`BLOCK` for tracing, `UC_HOOK_MEM_UNMAPPED` for fault recovery, `UC_HOOK_INTR` for syscall dispatch.
4. **Iterate on crashes**: each crash tells you what to map/hook next. This is the method, not a failure.
5. **Minimal trace output**: block-level beats instruction-level; counters and summaries beat per-step logs. Turn instruction tracing on only for small targeted ranges.

### Environment simulation table

| Dependency class | Examples | Simulation strategy |
| --- | --- | --- |
| libc | `malloc`, `free`, `memcpy`, `strlen`, `printf` | Hook address; implement in Python (bump allocator for malloc) |
| JNI | `GetStringUTFChars`, `FindClass`, `GetMethodID` | Build a fake JNIEnv function table in emulated memory; write `RET` stubs at each entry; hook the stubs |
| Syscalls | `read`, `write`, `mmap`, `ioctl` | `UC_HOOK_INTR`; dispatch by syscall number |
| Opaque globals | vtables, config blobs | Map a scratch page; initialize from the values static analysis found |

### Snapshot brute-force oracle

Unicorn's superpower: `uc.context_save()` / `uc.context_restore()` (or full memory snapshots). Per-guess cost collapses from "restart the process" to "restore a context":

```
save state → guess byte → run check fragment → read result → restore → next guess
```

Wrap it in an outer loop over positions and an inner loop over byte values: a 16-byte key at 256 tries/byte = 4096 fragment runs, seconds each. The same pattern solves VM key-chains (sequential per-position checks - see `references/custom-vms.md`).

## Qiling

```python
from qiling import Qiling
ql = Qiling(["./target", "arg"], "/root/fs-x8664_linux")   # rootfs supplies libs
ql.run()
```

Use when: foreign-arch ELF must run with its loader; you want syscall-level instrumentation (`ql.hook_syscall`); firmware with OS expectations. Heavier than Unicorn - reach for it when raw fragments cannot reconstruct enough context.

## angr: ask the binary questions

```python
import angr
proj = angr.Project("./target", auto_load_libs=False)
state = proj.factory.entry_state(stdin=angr.SimFileStream)
sm = proj.factory.simulation_manager(state)
sm.explore(find=lambda s: b"Correct" in s.posix.dumps(1),
           avoid=lambda s: b"Wrong"   in s.posix.dumps(1))
if sm.found:
    print(sm.found[0].posix.dumps(0))   # the input that reaches success
```

- Constrain the input space *first* (length, charset) or the solver drowns: `state.solver.add(len(inp) == 16)`, byte ranges via BVS constraints
- `explore` with find/avoid on stdout strings is the CTF-shaped workflow; custom avoid sets cut the state explosion
- Slow paths: heavy loops, checksums over the whole input (checksums defeat naive symbolic runs - emulate the checksum, solve the rest)
- angr's CFG (`proj.analyses.CFGFast()`) doubles as a static mapper when r2/Ghidra are awkward

## Z3: when the check is an equation

Decompiled check logic translates to Z3 constraints almost mechanically:

```python
from z3 import *
inp = [BitVec(f"c{i}", 8) for i in range(16)]
s = Solver()
for c in inp: s.add(c >= 0x20, c <= 0x7e)
# transcribe each decompiled transform as constraints:
s.add(inp[3] ^ inp[7] == 0x41)
s.add(inp[0] + inp[5] == 0xAA)
# ...
print(s.model() if s.check() == sat else "unsat")
```

Discipline: transcribe *exactly* what the decompiler shows - an off-by-one in your transcription produces a confidently wrong answer. Verify the model against the real binary before claiming victory (the-method.md, evidence ladder E1).

## Instruction-count side channels

For movfuscated-style targets where every observable is an instruction-count delta: Intel Pin (or a Unicorn block counter) measures executed instructions between two points; a genetic algorithm or simple hill-climb mutates the input and keeps variants that move the count toward the target. This is the emulation-native version of the timing oracle in `references/dynamic.md`.
