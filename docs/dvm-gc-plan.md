# DVM garbage collection plan

## Why

`husk-tl-dvm.c` has no GC: every object is pinned (`o->refs = 1u << 30`) and nothing ever frees it.

Measured on a large app (9 dex, 74k classes):

| frames | objects | free before jetsam |
|---|---|---|
| 0 | 189 846 | 3268 MiB |
| boot | 889 634 | 2997 MiB |
| 126 | 7 320 372 | 495 MiB |
| 194 | 8 551 986 | 68 MiB (jetsam) |

## Design

1. **Register windows** — `dvm_call_inner` holds Dalvik registers in a C local
   `uint64_t *regs`; register `(regs, m, pc)` in a per-thread list so a collector can scan it.
2. **Safe point** — a global `g_gc_request`; checked at every backward branch.
   Threads park at the check until the collection ends.
3. **Roots** — thread register windows, class statics (`dvm_class::sf`), JNI
   global/local refs, the pending exception, monitors.
4. **Mark** — walk `jobj` fields by `dvm_field::type`, skip `native`/detached slots,
   BFS with a mark bit.
5. **Sweep** — free unreachable plain objects first (strings, arrays, objects whose
   fields are only references/primitives); native-backed objects stay pinned until stable.
6. **Trigger** — every N frames, when the live set grows by 50%, and from
   `System.gc()` / `Runtime.gc()`.

## Milestones

- [x] object counter + `objects` in `unity: alive`
- [ ] register windows + safe point (no collection yet)
- [ ] mark pass, statistics only (`dvm-gc: total= reachable= garbage=`)
- [ ] sweep plain objects
- [ ] sweep native-backed objects, finalizers, weak refs
