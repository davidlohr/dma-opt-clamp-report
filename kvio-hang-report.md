# kvio raw_block `uring_cmd`: store hangs — CQ overflow backlog is never flushed (root cause + fixes)

The `raw_block` `uring_cmd` store path can strand completed I/O forever. All
NVMe commands succeed at the device; their completions sit in the kernel's
CQ **overflow backlog**, which the worker never flushes because it only
enters the kernel when it has new SQEs to submit. Root cause, falsifying
experiments, and two independent fixes below.

## Reproducer

```
python examples/kv_cache_offload_io/run_kv_offload_io.py \
    --model meta-llama/Llama-3.1-405B --dtype bfloat16 --chunk-tokens 256 \
    --num-chunks 1 --iters 1 --device /dev/ng3n1 --engine uring_cmd
```

Hangs in `wait_iouring()` forever (py-spy: MainThread at
`_write_uring_cmd_buffers`, core.py:1387). `--num-chunks 1 --iters 1`
suffices. Environment: kvio branch of mcgrof/LMCache, io-uring crate 0.7.14,
Linux 7.2.0-rc1, Samsung PM9A3; reproduced on three kernel configs.

## Root cause

**Constants.** The runner hardcodes `iouring_queue_depth=8`
(run_kv_offload_io.py:198; the library default of 256 is not used). The ring
is built without `setup_cqsize()` (lib.rs:1061), so the kernel gives
`cq_entries = 2 * sq_entries` = **16 CQEs total** (io_uring.c).

**Fan-out.** `_write_uring_cmd_buffers` splits the object at
`--mdts-bytes` (default 131072) and pushes *every* chunk into one
`batched_write()` call, then blocks in one `wait_iouring()`
(core.py:1369-1387). A 126 MiB object = 1008 payload chunks + 1 header =
**1009 concurrent submissions against a 16-entry completion queue**.

**Broken flow control.** The worker throttles on
`available = ring_size - submission_len()` (lib.rs:1568), but
`submission_len()` counts *unconsumed SQ slots*, which io-uring resets to 0
after every `submit()`. It measures nothing about outstanding requests, so
the worker pumps 8 SQEs per iteration with zero backpressure (the true
outstanding count sits unused in its own `in_flight` HashMap, lib.rs:1360).

**The strand.** Once >16 completions accumulate, the kernel sets the sticky
CQ-overflow bit and force-overflows *every* subsequent CQE
(io_cqe_cache_refill). Overflowed CQEs are only delivered by an
`io_uring_enter(GETEVENTS)` flush. The io-uring crate's `submit()` does set
GETEVENTS when the overflow flag is up — which is why the submission phase
limps through, delivering ~16 per submit — but the worker's reap path is
pure shared-memory (`ring.completion().collect()`) and its idle path is
`epoll_wait` (lib.rs:1543-1547): **after the final `submit()` it never
enters the kernel again.** The completion eventfd still fires; the worker
wakes, finds the (empty) CQ ring empty, and sleeps forever. Everything that
overflows after the last submit is stranded.

## The smoking gun

`/proc/<pid>/fdinfo` of the io_uring fd while hung (1009-command run):

```
SqMask: 0x7    SqHead: 1009  SqTail: 1009     <- all submitted and consumed
CqMask: 0xf    CqHead: 232   CqTail: 232      <- app reaped 232, ring empty
CqOverflowList:
  user_data=233, res=0, flags=0
  user_data=234, res=0, flags=0
  ...                                          <- everything else, res=0 = SUCCESS
```

plus `/sys/block/nvme3n1/inflight: 0 0` and clean dmesg: the device
completed every command; the results were never delivered. The 232 also
checks out arithmetically: ~13 submit-triggered overflow flushes x 16 CQEs
+ the initial un-overflowed fills.

## It is a race, not a count threshold (falsification data)

Bisecting showed neither command count nor object size is the variable:

| config | store cmds | bytes | result (N=3) |
|---|---|---|---|
| 126 MiB @ 2M mdts | 64 | 126 MiB | COMPLETE |
| 126 MiB @ 512K | 253 | 126 MiB | COMPLETE |
| 126 MiB @ 128K | 1009 | 126 MiB | HANG |
| 12 MiB @ 128K | 97 | 12 MiB | HANG |
| 6 MiB @ 64K | 97 | 6 MiB | COMPLETE |
| 3 MiB @ 32K | 97 | 3 MiB | COMPLETE |

At a fixed 97 commands, sweeping per-command size shows a **danger band**
(N=2 each): 32K completes, 64K marginal, **96K–160K hang**, 192K+ complete —
the default `--mdts-bytes 131072` sits in the middle of it. The race is
completion timing vs. submission pacing: commands that complete very fast
are reaped before the 16-CQE ring ever overflows; very slow commands
complete gradually after submission ends and are reaped via eventfd wakeups
as they trickle in; the mid-band bursts 8-at-a-time into a 16-slot ring
during submission, trips the sticky overflow bit, and strands. Onset is
probabilistic near the band edges (e.g. 66 cmds hung 1/5 at 8.1 MiB).

Consistent with the mechanism, raising the queue depth makes it vanish:
qd=8 hangs 5/5 across configs; **qd=16/32/64/128/256 completed 23/23**.

Only the store path is exposed: loads issue one command at a time
(core.py:1436), and the block-device engines don't have the mdts fan-out at
all (`_write_buffers` issues the object as one write) — matching the
original observation that `--engine io_uring` completes in ~100 ms.

**Introducing commit**: 7021790b ("Missing io_uring changes and introducing
nvme io_uring_cmd (passthrough) changes (#3274)") added both the unbounded
fan-out and the uring_cmd worker path. The earlier eventfd/epoll rework
(4fb03710) is *not* the regression — the pre-image polled at 10µs with the
same submit-only kernel entry, so it would strand identically, just spinning
instead of sleeping.

## Fixes (independent; both worth doing)

1. **Throttle on outstanding, not SQ occupancy** (lib.rs:1568):
   `available = min(ring_size, cq_entries).saturating_sub(in_flight.len())`
   — the `in_flight` map already tracks exactly the right quantity. This
   keeps completions inside CQ capacity so overflow never starts.
2. **Flush overflow on idle wakeups**: when the eventfd fires and
   `IORING_SQ_CQ_OVERFLOW` is set in the ring's sq_flags, call
   `submitter.submit()` (or `io_uring_enter(0, GETEVENTS)`) even with an
   empty queue, then re-reap. This makes the worker robust to overflow from
   any cause, not just this one.

Defense in depth: also consider `setup_cqsize(ring_size * 4)` and honoring
the library's own `DEFAULT_IOURING_QUEUE_DEPTH = 256` in the example runner
instead of the hardcoded 8.

**Workaround for users today**: pass a queue depth ≥16 (or use
`--mdts-bytes` outside the 64K–160K band), though both merely make the race
unlikely rather than impossible.

## Context

Found while using kvio as an independent check of an NVMe transfer-limit
patch series:
https://github.com/davidlohr/dma-opt-clamp-report/blob/main/kvio-analysis.md
