# the-report — findings that survive scrutiny

> Merged from: zhaoxuya520/reverse-skill (Evidence→Finding discipline, completion checklist, output quality rules), wshobson/agents (documentation phase). Credit in ATTRIBUTION.md.

## The standard

A reversing report is not a story about what you did; it is a chain of evidence a stranger can replay. Every finding traces to a command, every command runs copy-paste, and every gap is labeled as a gap.

## The evidence chain

```text
Evidence (command + output, with address/offset)
   → Finding (what it proves, one sentence)
      → Path (how to reproduce: script/transcript)
```

Rules that make the chain real:

- **Reproducible commands, not descriptions.** "I checked the imports" is noise; `rabin2 -i target | grep -i crypt` with its output is evidence.
- **Addresses, offsets, function names - never "some function".** `check_license+0x42`, `fcn.004013a0`, `sub_140001234` with your rename noted.
- **Confidence labels on every non-proven claim.** High (pattern + constants match), Medium (structure fits), Low (one weak signal). A finding without a label reads as proven; that is how reports lie.
- **Distinguish "read" from "executed".** Static reading (E2) and runtime observation (E1) are different rungs on the-method ladder - say which one supports each claim.
- **Negative results are results.** "strings contains no flag-shaped data (full output grepped, not truncated)" steers the next analyst away from a dead end - record it.

## What a finding contains

| Field | Example |
| --- | --- |
| Target + hash | `crackme.elf` (sha256:...) - identifies the exact binary |
| Location | `main+0xca`, file offset `0x2040` |
| Behavior | compares `transform(input)` against `0x5555...` blob |
| Evidence | the gdb/Frida transcript showing the comparison |
| Confidence | E1 (executed) / high |
| Reproduction | the exact command sequence |

## Documenting recovered knowledge

- **Functions**: signature (recovered convention), purpose, side effects, the rename you assigned
- **Structures**: the recovered layout table (offset/width/inferred type/field name) and the access evidence behind each field - the reconstruction method from `references/static.md`
- **Algorithms**: pseudocode next to the decompiled source; named constants ("SHA-1 by `0x5A827999`") rather than anonymous hex
- **Protocols/formats**: the written spec + parser/generator round-trip status (`references/protocols.md`)
- **Custom VMs**: the opcode table you recovered *is* the deliverable - it outlives the challenge

## Housekeeping

- **Anonymization**: reports about real engagements carry no un-anonymized target data (IPs, hosts, credentials) - zhaoxuya520's hard rule, kept
- **Copies and originals**: state where the working copy lives and that the original was untouched
- **Scope statement**: one line on what was authorized and what boundary was respected - it protects the user and the report's credibility
- **What you did NOT determine**: the open-questions list with the next probe for each (E4 rung) - the most useful section of any honest report and the one amateurs omit

## Before you say "done"

The completion checklist from `references/the-method.md`, in report form: every stage executed (not read), every finding evidenced, every command reproducible, every uncertainty labeled, the asked question answered, the boundary respected. If any box is unchecked, the task is not complete - and the report says so.
