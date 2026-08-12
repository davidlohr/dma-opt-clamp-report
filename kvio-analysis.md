# kvio: an independent check of the DMA-optimal limit

[kvio](https://github.com/SamsungDS/ebpf-syscall/blob/ebpf-fixes/docs/kvio.html)
projects the NVMe I/O a real LLM KV-cache offload would issue — from model
geometry alone, no GPU — and executes it against a real device through
LMCache's `raw_block` engine (POSIX, io_uring, or `io_uring_cmd` passthrough),
with an eBPF monitor to validate projection against wire commands.

That makes it a useful adversary for [this study](README.md): a second tool,
written by other people for a different purpose, exercising the same kernel
limit with the largest KV objects in circulation. This document reports what
it found — including two findings the fio-based campaign had not surfaced, and
two limits of the suite itself.

## Setup

- Host: EPYC 9124, AMD-Vi translated (DMA-FQ), blank Samsung PM9A3 (MDTS 2MB).
- Kernels: `kvbase` (tip v7.2-rc1) vs `kvopt` (+ the two-patch series, opted in).
- Model: **meta-llama/Llama-3.1-405B** — the largest KV object in
  [kvcache.io's projections](https://kvcache.io/io-projections.html): 126
  layers x 8 KV heads x 128 head_dim x bf16 x 256-token chunk =
  **132,120,576 B (126 MiB) per offload block**. kvio's catalog geometry, the
  kvcache.io projection, and an independent derivation agree byte-for-byte.
- Workload: 2 chunks x 2 iterations + warmup, block device, io_uring,
  O_DIRECT; engine command size (`--mdts-bytes`) swept 128K → 2M; 2 reps.

## Finding 1: the clamp also caps `max_segments` — by a factor of 8

The per-boot gates showed something the fio campaign never measured:

| kernel | `max_hw_sectors_kb` | `max_sectors_kb` | **`max_segments`** |
|---|---|---|---|
| kvbase | 128 | 128 | **33** |
| kvopt | 2048 | 2048 | **256** |

This is not a coincidence of configuration — nvme-pci *derives* the segment
limit from the transfer limit (`drivers/nvme/host/core.c`):

```c
static u32 nvme_max_drv_segments(struct nvme_ctrl *ctrl)
{
	return ctrl->max_hw_sectors / (NVME_CTRL_PAGE_SIZE >> SECTOR_SHIFT) + 1;
}
lim->max_segments = min_t(u32, USHRT_MAX,
	min_not_zero(nvme_max_drv_segments(ctrl), ctrl->max_segments));
```

128KB/4KB + 1 = **33**; 2MB/4KB + 1 = 513, capped by `NVME_MAX_SEGS` to
**256**. So the DMA-optimal clamp has a second-order effect nobody discusses:
it limits how many discontiguous pages a single command may gather. On the
clamped kernel a request built from ordinary scattered 4K pages tops out at
33 segments — about 132KB — *regardless* of what `max_sectors_kb` would
otherwise allow. Raising the ceiling raises both limits together.

This also explains a fio-side result: the study's superpage cliff needed
hugepage-backed buffers, and this is the other half of the same coin —
scattered buffers are segment-bound long before they are sector-bound.

## Finding 2: the block-path A/B — restores gain 13-20%, stores are flat

![kvio restore bandwidth](dma-opt-clamp-figs/fig9-kvio-restore.svg)

Medians of 2 reps, 126 MiB objects:

| engine cmd size | load (restore) MB/s |  | store MB/s |  |
|---|---|---|---|---|
| | kvbase | kvopt | kvbase | kvopt |
| 128 KiB | 1332 | **1537** (+15.3%) | 678 | 700 (+3.2%) |
| 512 KiB | 1434 | **1628** (+13.5%) | 709 | 713 (+0.5%) |
| 1 MiB | 1346 | **1610** (+19.6%) | 695 | 704 (+1.4%) |
| 2 MiB | 1429 | **1645** (+15.1%) | 687 | 698 (+1.6%) |

Restore bandwidth improves in every configuration tested (4/4, +13.5% to
+19.6%); store bandwidth is flat within noise, as the fio campaign also found
(the drive's write path is the bottleneck either way). Per-point variance is
high — individual reps differ by up to 20% on the clamped kernel — so the
magnitude is soft, but the direction is unanimous.

Note the shape: performance is roughly flat *across* engine command sizes
within each kernel. That is the point. The engine asks for 2MB commands; the
kernel decides what actually reaches the device. On `kvbase` everything
collapses to ≤128KB / 33 segments no matter what the application requests; on
`kvopt` the same requests reach the device as up to 1MB commands
(segment-bound at 256 x 4KB). **Application-level tuning cannot escape the
kernel limit — only the kernel knob can.**

## Finding 3: 2MB passthrough fails even with the ceiling lifted

With `--engine uring_cmd --mdts-bytes 2097152` on `kvopt` (max_hw = 2MB):

```
RawBlockCore write failed ...: [Errno 22] io_uring I/O error
```

`EINVAL`, not a sector-limit rejection: a 2MB transfer from a page-scattered
buffer needs 512 segments, and `NVME_MAX_SEGS` is 256. Passthrough
(`io_uring_cmd`) bypasses `max_sectors` entirely — `blk_rq_map_user_iov()`
bounds it by `max_hw_sectors` — so the series' ceiling lift *does* change
passthrough behavior by default, but the segment budget still caps a scattered
2MB command. Reaching 2MB commands needs physically contiguous (hugepage)
buffers, exactly as on the fio side.

## Finding 4 (suite limitation): the passthrough engine wedges at ~1000 commands

`--engine uring_cmd` with the default 128KB command size hangs indefinitely on
a 126 MiB object (1008 commands): the process blocks with **zero device
inflight** and load average 0.00, and never returns. It is not the kernel —
no I/O is outstanding, dmesg is clean, and the same object completes in
milliseconds through the block-device io_uring engine. Reproduced on both
kernels. The passthrough numbers in this document are therefore limited to
the single-command-size cases that complete; the wire-validation step
(`kvio_validate.py`) could not be exercised, since it needs a completed
passthrough capture.

A second, smaller gotcha: `--mdts-bytes` defaults to 131072, so a naive
`kvio` A/B feeds *both* kernels 128KB commands and measures nothing about the
kernel's limit. The engine's split must be raised above the boundary under
test.

## What this adds to the study

1. **Independent confirmation** that the clamp costs real bandwidth on
   KV-restore-shaped I/O at production object sizes — from a different tool,
   a different I/O engine, and the largest model geometry in circulation.
2. **A new mechanism**: the clamp silently caps `max_segments` at 33, which
   bounds scattered-buffer requests to ~132KB independently of the sector
   limit. This deserves a sentence in the series' changelog: lifting
   `max_hw_sectors` restores both the transfer *and* the gather capability.
3. **Corroboration of the contiguity requirement** from the opposite
   direction: fio showed the superpage cliff needs hugepages; kvio shows a
   scattered 2MB command is rejected outright by the segment budget.
4. A reminder that **application-level request sizing cannot compensate**:
   sweeping the engine's command size changed almost nothing within a kernel,
   while changing the kernel moved every point.

## Reproduction

```sh
git clone -b kvio https://github.com/mcgrof/LMCache && cd LMCache
# rust ext: (cd rust/raw_block && maturin develop --release)
python examples/kv_cache_offload_io/run_kv_offload_io.py \
    --model meta-llama/Llama-3.1-405B --dtype bfloat16 \
    --chunk-tokens 256 --num-chunks 2 --iters 2 --warmup 1 \
    --device /dev/nvme2n1 --engine io_uring --odirect --mdts-bytes 2097152
```

Deps beyond the documented list (LMCache 0.5.x pulls them in transitively):
`msgspec sortedcontainers requests prometheus_client py-cpuinfo psutil
aiohttp numba aiofile pyzmq transformers`. Raw logs for both kernels:
`data/kvio-final.tgz`.
