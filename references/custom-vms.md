# custom-vms — bytecode interpreters and VM protectors

> Merged from: yaklang/hack-skills vm-and-bytecode-reverse (the playbook: dispatcher ID, opcode mapping, custom disassembler, maze, protectors, decision tree), ljagiello/ctf-skills (VM patterns corpus), zhaoxuya520/reverse-skill (VM routing). Credit in ATTRIBUTION.md.

Base models often fail here in two ways: not recognizing the fetch-decode-execute pattern at all, or trying to analyze VM bytecode as native code. This reference exists to make both failures impossible.

## Quick identification

| Binary pattern | Likely VM type | Start with |
| --- | --- | --- |
| `while(1) { switch(bytecode[pc]) }` | switch dispatcher | map each case to an operation |
| indirect jump via table `jmp [table + opcode*8]` | table dispatcher | dump the jump table, analyze handlers |
| nested if-else chain on a byte value | if-chain dispatcher | same as switch, different syntax |
| stack push/pop dominant ops | stack VM | identify push/pop/arithmetic |
| `reg[X] = ...` array ops | register VM | map register indices to operations |
| 2D grid + direction input | maze challenge | extract grid, BFS/DFS |

Structural anatomy: bytecode program (data section) + program counter + register file or stack + memory area + the dispatcher loop (fetch opcode → decode/lookup handler → execute). If you can name all five in the decompilation, you have found the VM.

## The methodology (five steps)

**Step 1 - find the dispatcher.** Large switch in a loop; array of function pointers indexed by a data-buffer byte; one function with extreme cyclomatic complexity; xrefs to a buffer read byte-by-byte. This is the highest-value single function in the binary - name it, bookmark it.

**Step 2 - map opcodes to operations.** For each case/handler record:

| Property | How to read it |
| --- | --- |
| Opcode value | case number / table index |
| Operation type | what registers/stack change |
| Operand count | bytes consumed after the opcode |
| Operand type | immediate / register index / address |
| Side effects | output, memory write, flag modification |

**Step 3 - extract the bytecode program.** From the data section at the offset/length static analysis identified (`f.seek(off); f.read(n)`), or from the debugger after the loader decrypts it (self-decrypting VMs: dump post-decode).

**Step 4 - write the custom disassembler.**

```python
OPCODES = { 0x00: ("nop",0), 0x01: ("push",1), 0x02: ("pop",0),
            0x03: ("add",0), 0x07: ("jmp",2), 0x08: ("je",2),
            0x0A: ("mov",2), 0x0B: ("load",1), 0x0C: ("store",1),
            0xFF: ("halt",0) }   # (mnemonic, operand_bytes)

def disassemble(bc):
    pc = 0
    while pc < len(bc):
        op = bc[pc]
        mn, n = OPCODES.get(op, (f"UNK_{op:#04x}", 0))
        print(f"  {pc:04x}: {mn:8s} {' '.join(f'{b:#04x}' for b in bc[pc+1:pc+1+n])}")
        pc += 1 + n
```

**Step 5 - analyze the disassembled program** like ordinary assembly: find the input-reading opcodes, trace transformations, find the comparison, invert or satisfy it.

## Common VM shapes

- **Stack VM** (JVM/Python-like): operations on a value stack - PUSH imm, POP, ADD/SUB/XOR, CMP, JMP/JZ, PRINT, READ, HALT
- **Register VM** (x86-like): `MOV r,imm` `0x01 RR IIII`, `ADD r1,r2` `0x03 R1 R2`, flags register for conditional jumps, LOAD/STORE with addresses
- **Brainfuck/esolang**: eight commands; the trick is recognizing pointer-arithmetic loops as higher-level operations (character-by-character static analysis beats running it)
- **Nested VMs (VM in VM)**: analyze the outer VM first, extract inner bytecode, repeat - or try symbolic execution across both layers directly

## Maze challenges (the CTF staple)

Identification: directional input (WASD/UDLR/arrows/opcodes), a 2D array in data, x/y tracking, a win coordinate. Method: extract the grid (width/height from the access pattern, wall/path/start/end cell values from comparisons), print it, BFS the shortest path, convert the path to the expected input encoding. Direction encodings vary (WASD, UDLR, arrow scancodes, numbers, opcodes) - read the comparison to learn which.

```python
from collections import deque
def solve(maze, start, end):
    R, C = len(maze), len(maze[0]); q = deque([(start, "")]); seen = {start}
    while q:
        (r, c), path = q.popleft()
        if (r, c) == end: return path
        for nm, (dr, dc) in {'U':(-1,0),'D':(1,0),'L':(0,-1),'R':(0,1)}.items():
            nr, nc = r+dr, c+dc
            if 0 <= nr < R and 0 <= nc < C and maze[nr][nc] != '#' and (nr,nc) not in seen:
                seen.add((nr,nc)); q.append(((nr,nc), path+nm))
```

## VM protectors (commercial)

| Protector | Dispatcher style | Difficulty |
| --- | --- | --- |
| VMProtect | table + opaque predicates | High |
| Themida / Code Virtualizer | CISC-like, large handler set | High |
| Tigress (academic) | configurable layers | Medium-High |
| movfuscator | all-mov computation | Medium |
| custom CTF VM | simple switch | Low-Medium |

VMProtect approach: find the VM entry (pushad/pushfd sequence) → identify the VM context structure (saved registers, flags, bytecode pointer) → locate the handler table (obfuscated with opaque predicates) → per handler: strip junk, identify the core operation, document semantics → trace bytecode execution instruction-level → reconstruct the original logic from the trace. Tigress specifics: handlers may be split across functions, nested, polymorphic per build, or the bytecode may be encrypted per-fetch (hook the fetch or emulate).

Recurring architectures graduate to a Ghidra SLEIGH processor module (define registers/tokens/opcodes once, decompile the bytecode forever after) - worth it from the second challenge in the same ISA.

## Decision tree

```
Custom bytecode interpreter present?
├─ Dispatcher identifiable (switch/table/if-chain)?
│  ├─ few opcodes (<20) → simple CTF VM
│  │   ├─ stack-based → map push/pop/arith
│  │   ├─ register-based → map mov/add/cmp
│  │   └─ write disassembler → analyze → solve
│  └─ many opcodes (50+) → commercial protector
│      ├─ known protector → specific approach (above)
│      └─ custom → trace execution, pattern-match handlers
├─ no clear dispatcher?
│  ├─ all-mov → movfuscator (instruction-count side channel)
│  ├─ encrypted bytecode → find/trigger decryption, dump post-decode
│  └─ split/distributed handlers → trace to locate them
├─ maze? → extract grid, identify encoding, BFS, emit input
├─ input validation inside VM?
│  ├─ small input space → brute-force via Unicorn snapshot oracle
│  ├─ known input format → constrained angr solve
│  └─ complex check → write the disassembler, analyze the check
└─ nested VMs? → outer first, extract inner bytecode, repeat
   (or: symbolic execution may collapse both layers)
```

Tools that earn their keep here: Unicorn (fast handler emulation + snapshot oracles for sequential key-chain checks), angr (treat the whole VM as one constraint system), Pin/DynamoRIO (handler-execution traces), REVEN (full-system trace replay), Miasm (lift handlers to IR).
