# static — reading what does not run

> Merged from: wshobson/agents binary-analysis-patterns (control flow, data structures, decompilation patterns, calling conventions), ljagiello/ctf-skills tools corpus, p4nda0s/reverse-skills rev-struct (structure recovery). DWARF concepts from Trail of Bits dwarf-expert (concept only). Credit in ATTRIBUTION.md.

## When static is the right path

Static analysis is the default when: the target cannot run here (foreign arch, hostile environment), the question is structural ("what does this parse?", "what does this write?"), or you need the *whole* logic rather than one execution path. Dynamic beats it for anything observation can answer directly - see `references/dynamic.md` before committing to a long static read.

## Toolchain map

| Need | First choice | Notes |
| --- | --- | --- |
| Fast disassembly + scripting | radare2 / rizin | `aaa; afl; pdf @ main`; r2pipe for batch; `pdg` (r2ghidra) for decompilation - note the prebuilt r2ghidra zip is ABI-locked to one r2 version, `r2pm -ci r2ghidra` builds matching from source |
| Full decompiler + scripting | Ghidra | Free; headless: `analyzeHeadless proj/ tmp -import target -postScript script.py`; custom processor modules (SLEIGH) for VM ISAs |
| Industrial decompiler | IDA Pro / Binary Ninja | IDAPython for batch work; IDALib (IDA 9+) for headless runs |
| Online second opinion | dogbolt.org | Side-by-side decompiler comparison - validates assumptions |
| ELF structure | readelf / objdump / rabin2 | Sections, symbols, dynamic info |
| PE structure | dumpbin / CFF Explorer | Imports/exports first |
| File ID | file, TrID, Detect It Easy | Packer/compiler detection |

## Control-flow patterns (compiled idiom → source shape)

```asm
; if (a == b) → cmp + jcc; signed vs unsigned matters:
cmp eax, ebx        ; jl/jge = signed compare; jb/jae = unsigned compare

; for-loop → init + check at top + inc + jmp back
; while-loop → jmp to check first; do-while → body then check

; switch: jump table when dense (jmp [table + reg*8] after range check),
; sequential cmp/je chain when sparse or small
```

Compiler artifacts that are *not* source structure: inlined functions, tail calls (`jmp` instead of `call`+`ret`), dead code removal, optimizer-reordered blocks, RIP-relative addressing under PIE. Do not "fix" these in your mental model - annotate them.

## Data-structure recovery

Memory-access patterns reveal types:

| Pattern in code | Inference |
| --- | --- |
| `mov eax, [rbx + rcx*4]` | 4-byte-element array, rbx=base, rcx=index |
| `[rdi]`, `[rdi+8]`, `[rdi+0x10]` used as different fields | structure pointer; offsets define the layout |
| `mov rdi, [rdi+8]` in a loop with a null test | linked-list traversal (next at +8) |
| `movzx/movsx byte/word` | char/short field or variable |
| `movss/movsd` (xmm) | float/double |
| Access through `[reg + small_const]` after a virtual call | vtable offset - C++ class layout |

Reconstruction workflow (per p4nda0s rev-struct): collect all accesses to the candidate struct pointer across the function *and its callees* → table of (offset, width, access type) → merge into a layout → name fields by usage → apply the structure in the decompiler so every access reads as `s->field`. Cross-validate: every offset in the recovered layout must be explained by some access; an unexplained gap is either padding (alignment) or a missed field - check alignment first.

## Function-signature recovery

Calling conventions anchor parameters to registers:

| Platform | Integer/pointer args | Extra args | Callee-saved (typical) |
| --- | --- | --- | --- |
| x86-64 System V (Linux/macOS) | RDI, RSI, RDX, RCX, R8, R9 | stack | RBX, RBP, R12-R15 |
| x64 Windows | RCX, RDX, R8, R9 | stack | RBX, RBP, RDI, RSI, R12-R15 |
| x86 cdecl | stack (caller cleans) | - | - |
| x86 stdcall | stack (callee cleans) | - | - |
| ARM/ARM64 | R0-R3 / X0-X7 | stack | R4-R11, plus LR |

Read a function's first instructions: which registers are saved to locals = which parameters exist; what RAX/x0 holds at `ret` = the return. Floats travel in XMM0-7 (SysV) / XMM0-3 + XMM4-5 (Windows) - a function whose result lands in XMM0 returns a float/double.

## Symbols and debug info (the free shortcuts)

Before decompiling blind, mine what the binary already says about itself:

- `nm -C target` / `rabin2 -s` - symbols, demangled (C++ mangling reveals class + method names)
- GoReSym for Go (recovers buildinfo, function table, types), Rust panic strings and `demangle` for Rust, `swift demangle` for Swift
- `.debug_*` sections present? `readelf -S` → real DWARF: types, function signatures, locals - use them. Prefer `dwarfdump`/`llvm-dwarfdump` over readelf for DWARF-specific work (check `--version` first - libdwarf and LLVM ships differ)
- DWARF discipline (Trail of Bits concept): `llvm-dwarfdump --verify` for integrity; attributes may live on a referenced DIE (`DW_AT_abstract_origin`/`DW_AT_specification`) - resolve the chain before concluding an attribute is absent; when precision matters, look the standard detail up (dwarfstd.org) instead of answering from memory
- Stripped? Symbol-less function names (`sub_`, `fcn.`, `FUN_`) get renamed by you as you identify them - the renaming *is* the analysis log

## Decompiler-assisted reading order

1. Imports/exports first (what the program asks the OS for - `WS2_32` = networking, `CryptGenRandom` = crypto, `RegisterNatives` = JNI)
2. `main`/entry → TLS callbacks (run *before* entry on Windows PE - a classic hiding place) → thread creation
3. Follow the input: who reads stdin/argv/recv, and where that buffer flows
4. Identify the check: comparison calls, crypto loops, table lookups
5. Rename as you go; write the recovered pseudocode as comments

Scripting quick refs:

```python
# Ghidra (Jython) - find dangerous calls
for func in currentProgram.getFunctionManager().getFunctions(True):
    if func.getName() in ["strcpy", "sprintf", "gets", "system"]:
        for ref in getReferencesTo(func.getEntryPoint()):
            print("dangerous call at %s" % ref.getFromAddress())

# IDAPython - calls to a named function
for func_ea in idautils.Functions():
    if idc.get_func_name(func_ea) == "strcmp":  # then walk xrefs
        for xref in idautils.XrefsTo(func_ea):
            print("strcmp call at %x" % xref.frm)
```

## r2/r2pipe batch patterns

Batch mode without an interactive shell (per ljagiello radare2-r2pipe): `r2 -q -c "aaa; afl; pdf @ main" target`; `cmdj` returns JSON - beware command-echo traps when scripting sessions; ESIL emulation (`aei`, `aeim`, `aesu <addr>`) runs a single function without the OS. `wx` patches bytes in batch mode - for patching workflows see `references/anti-analysis.md` (bypass strategies).

## Common pitfalls

- **Optimizer artifacts**: decompiled loops may look nothing like source - trust behavior, not shape
- **PIE**: all code addresses are relative until a run resolves the base (`references/dynamic.md` - PIE-safe breakpoints)
- **Packed/obfuscated reads**: high-entropy sections, opaque predicates, junk bytes → route to `references/anti-analysis.md` before wasting hours
- **Single-decompiler trust**: when a decompiler output looks insane (VM bytecode lifted as native), cross-check with dogbolt.org or a second engine - the insanity is data, not code
