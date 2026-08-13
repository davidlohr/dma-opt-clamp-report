# kvio: `--engine uring_cmd` hangs on large KV objects (all threads in futex wait, zero I/O in flight)

Reporting a reproducible hang in the `raw_block` `uring_cmd` engine on the
`kvio` branch. The process stops making progress after the first object's
command batch is submitted; every thread ends up in `futex_do_wait` with **no
NVMe commands in flight**, so this looks like a lost wakeup / worker-pool
deadlock in userspace rather than a stuck device or kernel path.

## Reproducer

```
python examples/kv_cache_offload_io/run_kv_offload_io.py \
    --model meta-llama/Llama-3.1-405B --dtype bfloat16 --chunk-tokens 256 \
    --num-chunks 1 --iters 1 --device /dev/ng2n1 --engine uring_cmd
```

One 256-token chunk of Llama-3.1-405B geometry = 132,120,576 B (126 MiB) per
object, split by the default `--mdts-bytes 131072` into 1008 load / 1009 store
commands. `--num-chunks 1 --iters 1` is enough: the hang is not
duration- or load-dependent.

Last line of output, then silence indefinitely (observed >58 min in one run,
killed at 120 s in the instrumented run below):

```
LMCache INFO: RawBlockCore: skipping on-device metadata checkpoint load
              (core.py:330:lmcache.v1.storage_backend.raw_block.core)
```

## Environment

- kvio branch of `mcgrof/LMCache`, LMCache 0.5.3 / vLLM 0.27.1, Python 3.12,
  `maturin develop --release` for `rust/raw_block`
- Linux 7.2.0-rc1 (tip), x86_64, AMD EPYC 9124, AMD-Vi translated (DMA-FQ)
- Samsung PM9A3 1.9TB, MDTS 9 (2 MiB), blank/unmounted,
  `max_hw_sectors_kb=128`, `max_sectors_kb=128`
- Also reproduced on a kernel with `max_hw_sectors_kb=2048` /
  `max_sectors_kb=2048`, so it is independent of the transfer limits.

## Observed state (samples at 20 s, 60 s, 120 s)

```
=== t=20s ===   PID 8279 STAT Sl WCHAN futex_do_wait %CPU 28.3
=== t=60s ===   PID 8279 STAT Sl WCHAN futex_do_wait %CPU 17.2
=== t=120s ===  PID 8279 STAT Rl WCHAN futex_do_wait %CPU 11.1

threads (all seven, at every sample):
  TID 8279 8281 8282 8283 8284 8285 8286   STAT Sl   WCHAN futex_do_wait

/proc/<pid>/stack (identical at every sample):
  [<0>] futex_do_wait+0x3a/0x80
  [<0>] __futex_wait+0x99/0x110
  [<0>] futex_wait+0x7b/0x140
  [<0>] do_futex+0xf1/0x2d0
  [<0>] __x64_sys_futex+0x129/0x200
  [<0>] x64_sys_call+0x2158/0x26e0

/sys/block/nvme2n1/inflight:   0   0     (at every sample)
loadavg:                       0.93 -> 0.52 -> 0.19
dmesg:                         clean (only normal nvme probe messages)
```

The decaying %CPU is the initial burst averaged over a lengthening idle
window; the process performs no work after the first moments.

## Why this looks like userspace, not the kernel or device

- `inflight` is `0 0` at every sample: the block layer has no outstanding
  requests, so no NVMe command is pending completion.
- No timeouts, resets, or errors in `dmesg` across the whole run.
- The same 126 MiB object completes in ~100-200 ms through the *block-device*
  engines on the same device and kernel:
  `--engine io_uring --odirect --device /dev/nvme2n1` reports
  `load p50 86 ms / 1580 MB/s`, `--engine posix` reports `load p50 31 ms`.
- Every thread, including the main thread, is parked in `futex_do_wait`.

## Scaling / smaller objects

Not yet bisected against object size. Given the split arithmetic, a smaller
model would be the natural next data point — e.g. Llama-3.2-1B (8 MiB per
256-token chunk) is 64 commands at the default split, and 4 commands with
`--mdts-bytes 2097152`. If the hang is a queue-depth or completion-batching
issue in the worker pool, there should be a threshold in command count where
it appears.

## Side note on `--mdts-bytes`

Unrelated to the hang, but worth a documentation line: `--mdts-bytes`
defaults to 131072, so an A/B that intends to compare kernel transfer limits
issues 128 KiB commands in both arms unless the flag is raised. Also, a
2 MiB passthrough command from a page-scattered buffer fails with
`[Errno 22] io_uring I/O error` even when `max_hw_sectors` allows it, because
`NVME_MAX_SEGS` is 256 (256 x 4 KiB = 1 MiB); reaching 2 MiB commands needs
physically contiguous buffers.

## Context

Found while using kvio as an independent check of an NVMe transfer-limit
patch series; the full write-up of that comparison (including the block-path
results kvio produced successfully) is at
https://github.com/davidlohr/dma-opt-clamp-report/blob/main/kvio-analysis.md
