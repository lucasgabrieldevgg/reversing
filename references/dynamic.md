# dynamic — run it, watch it, dump it

> Merged from: ljagiello/ctf-skills (memory-dumping strategy, comparison direction, PIE debugging, decoy detection, oracle techniques), zhaoxuya520/reverse-skill (dynamic workflow), p4nda0s/reverse-skills rev-frida (modern Frida discipline), wshobson/agents (debugging expertise). Credit in ATTRIBUTION.md.

## When dynamic is the right path

Dynamic beats static whenever *observation* can answer the question: the check compares something (hook it), the answer is computed at runtime (dump it), the control flow depends on hidden state (trace it), or static reading is blocked by packing/obfuscation that execution unwraps naturally. Anti-analysis measures may fight you - `references/anti-analysis.md` - but many checks are cheaper to watch than to read.

## Tracing (the cheapest dynamic rung)

```bash
strace -f -s 500 ./target      # syscalls, follow forks, full strings
ltrace ./target                 # library calls - strcmp/memcmp args visible live
ltrace -e 'strcmp*|memcmp*|strncmp*' ./target   # only the comparisons
strace -c ./target              # syscall profile (which syscalls dominate)
```

Read a trace like a story: open/read = input, brk/mmap = allocation, write = output, a comparison right after a read = the check. `ltrace` showing `strcmp("AAAA", "S3cr3t!")` just handed you the expected value.

## GDB discipline

```bash
gdb ./target
start                    # run to main - resolves PIE base first
b *main+0xca             # breakpoint relative to main (PIE-safe)
b strcmp                 # break on the comparison itself
info registers rsi rdi   # arguments at the breakpoint
x/s $rsi                 # dump a string argument
x/32bx $rdi              # dump raw bytes
watch *(int*)0x5555-...  # hardware watchpoint on a variable
catch syscall ptrace     # syscall catchpoints
```

- **PIE rule**: never `b *0x401234` on a PIE binary before `start` - the base is random. `start` first, then relative breakpoints (`*main+offset`), or read the resolved base via `info proc mappings`.
- **Conditional breakpoints** slow execution 100x (they re-enter the debugger); prefer a breakpoint + ignore count (`ignore 1 500`) or a watchpoint.
- Python inside GDB (`python ... end`) turns the debugger into a scriptable oracle - dump memory, search patterns, even brute-force per-byte (see oracles below).
- pwndbg/GEF are drop-in upgrades (context on every stop, heap/ROP helpers); `brew install pwndbg/tap/pwndbg-gdb` on macOS, source build on Linux.
- Windows: x64dbg is the workhorse (ScyllaHide plugin hides the debugger from anti-debug checks). macOS/LLVM: lldb.

## Frida hooking (capture without reading)

Modern Frida CLI notes (per p4nda0s rev-frida): spawned processes resume *after* the script loads - there is no `--no-pause` anymore:

```bash
frida -U -f com.example.app -l hook.js      # spawn Android app with script
frida -U com.example.app -l hook.js         # attach to running app
frida ./target -l hook.js                   # local binary
```

```javascript
// The classic: capture what the program compares against
Interceptor.attach(Module.getExportByName(null, "strcmp"), {
  onEnter(args) { this.a = args[0].readUtf8String(); this.b = args[1].readUtf8String(); },
  onLeave(retval) { console.log(`strcmp("${this.a}", "${this.b}") = ${retval}`); }
});
// memcmp variant: read n bytes from both sides, hex-dump them
// Java (Android): Java.use("java.lang.String").equals.overload(...)
// ObjC (iOS): ObjC.classes.ClassName["- method:"] via Interceptor
```

- **Loader-aware hooking**: native `.so` code may not be loaded when the script starts. Hook `dlopen`/`android_dlopen_ext` return, or use `Module.load` callbacks, then attach to the module's exports (`Module.getExportByName("libfoo.so", "check")`).
- **Memory scanning**: `Process.enumerateRanges`, `Memory.scan` for patterns (find the flag buffer by magic prefix).
- **Bypassing anti-debug with Frida**: hook the check itself (`ptrace`, `getpid`+`/proc` reads, `IsDebuggerPresent` equivalents) and return the clean value - see `references/anti-analysis.md` for the full matrix.

## The memory-dumping strategy (the killer move)

**Key insight: let the program compute the answer, then dump it.** You do not need to reverse the transform - the program performs it for you:

1. Break at the *final* comparison (decoys live earlier - see below)
2. Enter any input of the correct length
3. Dump the computed expected value: `x/s $rsi` (or the register holding the computed side)

This turns "reverse a 40-round custom cipher" into "run it and read the output". It generalizes: break after any decryption/computation and read memory - unpacking works the same way (`references/anti-analysis.md`).

## Comparison direction (critical, read twice)

Two shapes, and misreading them costs hours:

1. `transform(input) == stored_target` → you must **invert** the transform (or Z3 it, or dump the target and work forward - see the-method quick-judgments)
2. `transform(stored_target) == input_expected` → the answer **is** the transformed data. Apply the transform to the stored blob; do not invert anything.

Distinguish by data flow: which side of the comparison is derived from user input at runtime, and which is a static blob in `.rodata`?

## Decoy detection

Pattern: several comparison targets in sequence, each with a different "success" message, only the last one real (or none of them - the real check is elsewhere entirely). Counter: locate *all* comparisons first (xrefs to `strcmp`/`memcmp`/custom compare), break at the **final** one, and treat earlier wins as decoys until proven otherwise.

## Oracles (when you cannot read the check at all)

When the check is a black box but responds measurably to input, build an oracle and brute-force through it:

- **ltrace/strace counting oracle**: number of `read()` calls or instructions varies per correct prefix → byte-at-a-time extraction (the classic SIGFPE/signal-handler counter variant works the same)
- **LD_PRELOAD memcmp oracle**: shim `memcmp` to report *how many* bytes matched; brute-force position by position
- **Timing oracle**: rdtsc-delta or runtime difference between guesses → per-character extraction (works on remote services too)
- **GDB/Frida bit-scraping**: watch the comparison result register and binary-search the input space

Each oracle needs calibration: run known-bad inputs, measure the baseline, then measure candidate bytes until one deviates. See `references/emulation.md` for snapshot-based oracles (fork per guess, no process restart cost).

## Discipline

- Work on a **copy**; never patch the original file in place (patch flows are in `references/anti-analysis.md`)
- Unknown/suspicious samples run in the isolated lab only - snapshot taken first, networking controlled or simulated
- Record every probe you ran (they become the evidence chain in `references/the-report.md`)
- One probe answers one question; batch questions, not probes - a 50-command gdb session you cannot explain is a session you cannot report
