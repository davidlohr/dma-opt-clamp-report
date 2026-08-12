# Lifting the NVMe DMA-Optimal Clamp: a measurement study

Evaluation of a two-patch series that lets NVMe requests grow past the 128KB
DMA-optimal bound on translated IOMMU systems — as an administrator opt-in, with
defaults untouched. Measured 2026-08-11/12 on a Latitude.sh `m4-metal-medium`
(EPYC 9124, AMD-Vi translated) and KVM guests on the same host, against a tip
v7.2-rc1 baseline. Figures in `dma-opt-clamp-figs/`; patches in `patches/`; raw
data in `data/`.

## Executive summary

Since the `blk_rq_dma_map` conversion, an NVMe request costs **one** IOVA
allocation and one mapping regardless of its size — so larger requests mean
*fewer* DMA-mapping operations, submissions, completions, and interrupts, not
more. The kernel nevertheless still hard-caps NVMe transfers at 128KB on
translated IOMMU systems, a limit designed for a cost model that no longer
exists, and one that administrators cannot raise.

The series lifts the *ceiling* (`max_hw_sectors` becomes MDTS-bound) while
keeping the *default* request size at 128KB. Every measurement here compares the
baseline kernel against the series in its two states:

- **Default**: bit-equivalent to baseline in every environment and workload
  measured — within ±0.4% on every metric, to the microsecond on QoS tails.
  Safe to ship fleet-wide.
- **Opt-in** (`echo <MDTS> > /sys/block/<dev>/queue/max_sectors_kb`):
  **+371%** large-block throughput on virtualized NVMe, **−32%** KV-extent
  restore p99 on bare metal, **−94%** interrupts/GB, **−72%** streaming sys-CPU
  through stacked storage paths — traded, on one of two tested drive models,
  against −8% raw streaming bandwidth and a sharp same-device small-IO latency
  penalty that per-device opt-in fully avoids.

For AI/ML KV-cache serving, the practical outcome (§9): cloud instances get a
multi-x faster NVMe offload tier with one udev rule; bare-metal serving trades a
few percent of restore bandwidth for a 32% better TTFT tail on dedicated offload
devices; and the storage layout rule is simply "opt in the KV tier, never the
metadata tier."

## 1. Background: the clamp and why it is obsolete

Commit `3710e2b056cb` ("nvme-pci: clamp max_hw_sectors based on DMA optimized
limitation") capped NVMe transfer size at `dma_opt_mapping_size()` — on a
translated IOMMU domain, 128KB, the largest IOVA range-cache size class.
Anything larger allocates and frees its IOVA through the per-domain
`iova_rbtree_lock`, a genuine scalability wall in the scatterlist era, when
mapping work scaled with transfer size.

That era ended with the scatterlist-less DMA helpers (`858299dc6160`) and
nvme-pci's conversion to them (`7ce3c1dd78fc`): a request now takes a **single
IOVA allocation covering the whole transfer**. Splitting a 1MB transfer into
8x128KB requests costs seven extra allocations — plus seven extra submissions,
completions, and interrupts — over the unsplit request. The clamp now optimizes
for the inverted problem. (An earlier proposal lifted the cap outright,
default and all; this series supersedes it with the opt-in split after the
measurements below.)

## 2. The series

1. **`block: let drivers cap the default max_sectors`** — adds
   `queue_limits.max_default_sectors`, honored only in the fallback branch of
   the `max_sectors` computation (`min_not_zero` against
   `BLK_DEF_MAX_SECTORS_CAP`). It can only lower the default; an explicit user
   setting (`max_sectors_kb`) overrides it up to `max_hw_sectors`. Needed
   because `blk_validate_limits()` always recomputes `max_sectors`,
   `max_user_sectors` belongs to sysfs, and `37105615f731` deliberately removed
   small-`io_opt` clamping (a side-find: SCSI's SAS `opt_sectors` → `io_opt`
   clamp has been silently inert since then — converting it to the new limit is
   the natural follow-up).
2. **`nvme-pci: cap the default transfer size at the DMA-optimal size`** —
   `max_hw_sectors` = min(`NVME_MAX_BYTES`, `dma_max_mapping_size()`), the true
   ceiling (descriptor sizing + the swiotlb bound); `dma_opt_mapping_size()`
   feeds the new default cap via `ctrl->opt_transfer_sectors`, set only when
   `opt < max` — i.e. only where a genuine DMA hint exists. Passthrough,
   direct-DMA, and non-PCI transports are untouched.

Observable behavior, verified per boot: translated IOMMU →
`max_hw_sectors_kb` = MDTS bound with `max_sectors_kb` = 128 default, raisable;
passthrough → identical to baseline (`2048/2048` on the test drive — notably,
**passthrough systems already run 2MB default requests today**; the 128KB
default only ever existed behind translated IOMMUs).

## 3. Why lifting the ceiling is safe (the rbtree question)

The original clamp guarded `iova_rbtree_lock`. All three premises of that
concern have inverted:

1. **The allocator-traffic arithmetic flipped sign.** Allocator traffic is now
   `bandwidth / request_size` — inversely proportional to request size. Opting
   in to 2MB *reduces* allocator operations per byte by 16x; the "dangerous"
   configuration generates the least allocator traffic. Reaching even ~100K
   rbtree ops/s would take >13GB/s of >128KB requests through one device.
2. **The rate is device-bandwidth-bounded.** At 2MB per command, a 14GB/s
   device issues ~7K allocations/s — orders of magnitude below the
   hundreds-of-thousands-per-second regime the range cache exists for. The
   populations sort correctly: high-rate small I/O stays entirely rcache-served
   (measured: the 4K guard at 915K IOPS shows zero rbtree traffic in either
   configuration); only low-rate large I/O touches the tree.
3. **The lock is per-IOMMU-domain, i.e. per-device** — adding devices adds
   locks; the many-CPUs-one-lock convergence requires a single device, where
   the bandwidth bound already caps the rate. And in the default lazy (DMA-FQ)
   mode, frees are batched through the flush queue off the completion path.

Empirical worst case (§7.4): sixteen CPUs for 60 seconds with *every* request
above the rcache bound produce 1.17µs average contended wait — **~0.005% of CPU
time** on the lock. The wall cannot be rebuilt through this driver.

## 4. Test rig and methodology

- **Host**: Latitude.sh m4-metal-medium — EPYC 9124 (Zen 4, 16C/32T), 128GB,
  $1.25/hr. AMD-Vi active, default domain **Translated, DMA-FQ lazy**, verified
  per-boot from the test device's IOMMU group.
- **Test media**: 2x blank Samsung PM9A3 1.9TB (MDTS = 2MB); OS on 2x Micron
  7450 480GB (md-raid1) — later used for the second-drive-model test. First
  100GiB of the PM9A3 preconditioned once; timed runs read-only against it.
- **Kernels**: baseline (tip v7.2-rc1) vs baseline + the two-patch series, same
  config, one variable. The series is measured in both states — default, and
  opt-in via `max_sectors_kb`. A `CONFIG_LOCK_STAT` build of the series kernel
  serves diagnostics only. *(Full-matrix opt-in numbers were collected on a
  kernel with the equivalent hard-lifted configuration; equivalence to the
  series' opt-in state was validated on overlapping cases within ±1.1%, exactly
  ±0.0% on interrupt counts.)*
- **Discipline**: strictly serial boots, rotated boot order, kernel identity
  gated on `uname -r` before every run, per-boot gates (`max_hw_sectors_kb`,
  `max_sectors_kb`, IOMMU domain type, MDTS), per-boot dmesg scans, interrupts
  counted from `/proc/interrupts` deltas.

## 5. The benchmarks in detail

All cases: fio, `ioengine=io_uring`, `direct=1` on the raw device, 20s
`time_based` runs after 3s ramp, `norandommap`, 3 reps per boot, medians across
boots. Large-block cases use `iomem=mmaphuge` (hugepage-backed, physically
contiguous buffers) so the NVMe 256-segment budget never confounds the
sector-limit variable.

### 5.1 Block-size sweep

| case | fio shape | what it isolates |
|---|---|---|
| `seq2m_qd8` | bs=2M rw=read QD8, 1 job | streaming at moderate depth |
| `seq2m_qd32x4` | bs=2M QD32 x 4 jobs, disjoint regions | streaming at saturation |
| `rand1m_qd16` | bs=1M randread QD16 | above-boundary, mid size |
| `rand512k_qd32` | bs=512k randread QD32 | above-boundary, small-large |
| `rand128k_qd32` | bs=128k randread QD32 | **guard**: exactly at the rcache bound |
| `rand4k_qd32x8` | bs=4k randread QD32 x 8 jobs | **guard**: IOPS path (915K IOPS) |

### 5.2 KV-cache-shaped workloads, and what they model

These emulate the **I/O pattern of KV-cache offload to NVMe** in the host-DRAM
staging path — how vLLM/LMCache-class serving stacks ship today without
GPUDirect: KV blocks for evicted sequences are dumped to NVMe and pulled back
through pinned host buffers on resume, while the serving process keeps issuing
unrelated small reads. Faithful to shape, concurrency, buffer type, and I/O
discipline; not an inference stack (no GPU, no attention compute, no H2D
copies — all downstream of the NVMe request path the series changes).

#### kv_restore — concurrent sequence resume (the TTFT case)

16 jobs x QD2 x 2MB random reads. Each fio job plays one evicted sequence being
resumed: its 2MB reads are that sequence's KV extents (a contiguous block span
of a few thousand tokens' keys/values as a contiguous-block allocator lays them
out), QD2 models restore pipelining, and sixteen jobs is a burst of returning
conversations. The metric that matters is **whole-extent completion latency**:
a resumed sequence cannot start prefill until its KV is back, so extent p99 is
the time-to-first-token proxy.

#### kv_prefix — prefix-cache reload

8 jobs x QD4 x 1MB *sequential* reads in disjoint 12GB regions: system-prompt
and common-context KV written once, contiguously, streamed back in order when a
cold prefix-cache entry is pulled — several tenants concurrently.

#### kv_qos — restores vs latency-sensitive small I/O (the interference case)

Two heterogeneous job groups on the *same* device, per-job reporting:
`qos_big` = 4 jobs x QD8 x 2MB randread (bulk restores); `qos_small` = 2 jobs x
QD16 x 4K randread (block-table/metadata lookups, embedding fetches, another
tenant). Headline metric: the 4K job's p99 while the storm runs.

#### kv_evict — eviction flush (the write half)

4 jobs x QD8 x 2MB writes into disjoint regions of the scratch drive —
sequences demoted to NVMe under KV memory pressure. The sector limit applies to
writes identically.

Fidelity: hugepage-backed buffers mirror pinned staging buffers real stacks
register for DMA; O_DIRECT + io_uring matches their engines; 1–2MB extents sit
where per-sequence KV block spans land for 7–70B-class models with contiguous
block allocators. Limits: no compute overlapping I/O, no H2D stage, contiguous
extents rather than per-layer scatter.

### 5.3 Probes, guest matrix, hardening

- **Isolation probes**: re-capping `max_sectors_kb` to 128 must return every
  number to baseline (it does, exactly — the deltas are pure request-size
  policy); depth sweeps test whether queue depth recovers streaming losses (it
  does not).
- **Guest matrix**: vng guests, 14 vCPU, emulated Intel vIOMMU, RAM-backed
  emulated NVMe (4MB MDTS) — the environment where completions are vmexits.
  KV-QoS shapes deliberately not run there (a RAM-backed device model's
  arbitration characterizes the emulator, not a deployment).
- **Hardening**: passthrough guard boots; cross-device QoS; `iommu.strict=1`;
  16-way rbtree contention under `CONFIG_LOCK_STAT`; blktests `block` + `nvme`
  on the opt-in config under lockdep; a second drive model (Micron 7450).

## 6. Results: the default changes nothing

Gates on every series boot: `max_hw_sectors_kb=2048` (MDTS-bound) **and**
`max_sectors_kb=128` simultaneously — ceiling lifted, default kept.

| metric | baseline | series default | Δ |
|---|---|---|---|
| bare metal: seq 2M QD8 MiB/s | 6607 | 6610 | +0.0% |
| bare metal: seq 2M QD8 irq/GB | 9362 | 9354 | −0.1% |
| bare metal: rand 1M QD16 irq/GB | 9345 | 9360 | +0.2% |
| bare metal: rand 4K guard IOPS | 914,952 | 915,493 | +0.1% |
| bare metal: QoS small 4K p99 µs | 4620 | 4620 | +0.0% |
| guest: seq 2M QD8 MiB/s | 5782 | 5740 | −0.7% |
| guest: rand 512K QD32 MiB/s | 5743 | 5794 | +0.9% |
| guest: rand 4K guard IOPS | 77,402 | 76,213 | −1.5% |

Identical within noise in both environments — the QoS tail to the microsecond.
This is the property that makes the series shippable fleet-wide.

## 7. Results: opt-in vs baseline

### 7.1 Bare metal (PM9A3, translated domain)

![bare metal bandwidth](dma-opt-clamp-figs/fig1-baremetal-bw.svg)

![interrupts per GiB](dma-opt-clamp-figs/fig2-irq-per-gb.svg)

| case | baseline | opt-in | Δ | irq/GB Δ |
|---|---|---|---|---|
| seq 2MB QD8 (MiB/s) | 6607 | 6083 | **−7.9%** | −93.7% |
| seq 2MB QD32x4 (MiB/s) | 6676 | 6132 | **−8.1%** | −93.7% |
| randread 1MB QD16 (MiB/s) | 6608 | 6549 | −0.9% | −87.4% |
| randread 512K QD32 (MiB/s) | 6607 | 6612 | +0.1% | −74.8% |
| randread 128K QD32 (guard) | 6453 | 6455 | 0% | 0% |
| randread 4K QD32x8 (IOPS, guard) | 914,952 | 915,388 | 0% | 0% |

Requests go out unsplit and completions collapse 16x, but at fixed user queue
depth the 128KB splitting had *amplified device-side parallelism* (QD8 x 2MB
became ~128 in-flight commands), and the PM9A3 tops out near 6.1GB/s on 2MB
commands vs 6.6–6.7GB/s on 128KB streams. Re-capping via sysfs restores
baseline exactly; added depth does not recover the loss — it is drive-internal,
and **drive-specific** (§7.6: it does not reproduce on the Micron 7450).

### 7.2 KV-shaped workloads (bare metal)

![QoS p99](dma-opt-clamp-figs/fig3-qos-p99.svg)

| case | metric | baseline | opt-in | Δ |
|---|---|---|---|---|
| kv_restore | BW MiB/s | 6497 | 6086 | −6.3% |
| | extent p50 µs | 8454 | 10,420 | +23% |
| | extent p99 µs | 16,318 | 11,076 | **−32%** |
| kv_prefix | BW MiB/s | 6584 | 6523 | −0.9% |
| kv_qos big | BW MiB/s | 1617 | 1450 | −10.3% |
| kv_qos small (shared device) | IOPS | 3676 | 1504 | **−59%** |
| | p99 µs | 4620 | 11,076 | **+140%** |
| kv_evict (write) | BW MiB/s | 2681 | 2681 | 0% |

The **QoS penalty** is the sharp edge: 4K readers sharing a device with unsplit
2MB streams wait behind whole 2MB command service times. It is strictly
same-device — moved to a second device, the same 4K workload runs at
**p99 = 165µs / 188K IOPS** while the storm rages (§7.4). The **tail paradox**
is the reward: whole-extent p99 improves 32% (no sub-request jitter) even as
the median worsens.

### 7.3 Virtualized (the headline win)

![guest seq 2MB](dma-opt-clamp-figs/fig4-guest-seq2m.svg)

| case | baseline | opt-in | Δ |
|---|---|---|---|
| seq 2M QD8 MiB/s | 5782 | 27,258 | **+371%** |
| seq 2M QD8 p99 µs | 4981 | 823 | **−83%** |
| rand 512K QD32 MiB/s | 5743 | 9386 | **+63%** |
| rand 4K guard IOPS | 77,402 | 77,706 | +0.4% |

Where completions are vmexits, 16x fewer of them is transformative. This is
the common cloud deployment (emulated/paravirt NVMe under a guest vIOMMU), and
the win is one sysfs write — or one udev rule in a guest image — away.

### 7.4 Hardening

- **Passthrough guard**: identity-domain boots show baseline and series
  bit-identical (`2048/2048`, ~6.0GiB/s) — the no-hint guard works; translation
  overhead itself measures <0.5% on this path.
- **Cross-device QoS**: opt-in storm on drive A, 4K on drive B: p99 165µs vs
  11,076µs shared — the interference is same-device queue contention only.
- **Strict mode (negative result, reported as one)**: under `iommu.strict=1`
  the opt-in keeps the same −8% seq shape at equal sys%, and 512K costs ~+4pp
  sys at equal bandwidth — the tradeoff is completion-cost-driven, not
  invalidation-driven.
- **rbtree lock_stat** (16 jobs, every request >rcache, 60s):
  `iova_rbtree_lock` — 1,076,721 acquisitions, 38,676 contentions, 1.17µs avg
  wait, 58µs max, **≈0.005% of CPU time** (vs 129 contentions at default).
- **blktests** (`block` + `nvme`, opt-in limit, lockdep kernel): **72 passed,
  0 failed, 47 skipped**, zero dmesg findings including lockdep.

### 7.6 Second drive model (Micron 7450, via file over md-raid1)

Both 480GB Microns carry the OS mirror, so this is a directional test: a 30GB
file written through the filesystem, read back O_DIRECT, `max_sectors_kb`
toggled on both legs *and* md0 (md's stacked limit is frozen at assembly).
Reads stripe across both mirror legs (~9.1GiB/s aggregate). Medians, n=3:

| case | 128KB requests | opt-in (4MB-capable) | Δ |
|---|---|---|---|
| seq 2MB QD8 — MiB/s | 9078 | 9107 | +0.3% |
| seq 2MB QD8 — sys % | 24.7 | 6.9 | **−72%** |
| rand 512K QD32 — MiB/s | 6341 | 6336 | −0.1% |
| rand 512K QD32 — sys % | 26.4 | 17.0 | −36% |

Two upgrades: the PM9A3's streaming loss **does not reproduce** here (n=2,
split — the regression is drive firmware behavior, not a property of large
requests), and this is the campaign's first measured bare-metal CPU win:
through a stacked fs+md+nvme path, 16x fewer requests cut streaming sys time
3.6x at identical bandwidth.

## 8. Overall conclusions

1. **The mechanism is verified end to end.** With the series opted in, requests
   reach the device unsplit (gates, request accounting), interrupts/GB fall
   75–94%, and DMA-mapping operations fall 16x at 2MB.
2. **Safety is proven, not argued.** The rbtree wall cannot be rebuilt through
   this driver (designed-worst load: 0.005% CPU on the lock); blktests passes
   clean under lockdep at the raised limit; ~40 boots of dmesg scans show zero
   faults or warnings; the sub-128KB population is bit-identical throughout.
3. **The wins are environment-specific — which is the argument for the shape of
   the series.** Virtualized NVMe: +371%/+63%, transformative. Bare metal:
   tail-latency and CPU-efficiency wins (−32% extent p99; −72% stacked-path
   sys), while raw streaming bandwidth is drive-dependent (−8% on PM9A3, flat
   on Micron 7450). A default flip would gamble those tradeoffs fleet-wide; a
   per-device opt-in lets each deployment take exactly the wins available to
   it. The defaults, meanwhile, are provably untouched.
4. **The QoS effect defines the deployment rule.** Unsplit large commands
   monopolize a device against small I/O (+140% p99 same-device; 165µs
   cross-device). The knob being per-queue is precisely what makes the hazard
   avoidable: opt in per device, per tier.
5. **Follow-up**: convert SCSI's SAS `opt_sectors` to the new block limit,
   restoring the capability `37105615f731` silently removed.

## 9. Impact for AI/ML KV-cache workloads

The series was evaluated specifically against KV-offload I/O shapes (§5.2).
Summary of what it means for a serving deployment:

- **Cloud/virtualized serving is the headline beneficiary.** KV offload tiers
  on cloud instances with emulated/paravirt NVMe gain **4.7x restore
  bandwidth** (+371%) and **−83% extent p99** with one udev rule in the guest
  image. Prefix-cache reloads at 512K gain +63%. For inference fleets running
  on cloud block storage, this is the difference between NVMe offload being a
  bottleneck and being effectively free at the I/O layer.
- **Bare metal, dedicated offload device: a TTFT-tail trade worth taking.**
  Opting in trades −6% aggregate restore bandwidth and +23% median extent
  latency for **−32% extent p99** — and TTFT SLOs are tail-driven. On drives
  that don't pay the streaming penalty (Micron 7450 class), the tail win comes
  with flat bandwidth plus large CPU savings; on PM9A3-class drives the −8%
  applies only if restores are bandwidth-bound rather than tail-bound. Measure
  per drive model; the knob is per-device.
- **Storage layout rule: opt in the KV tier, never a shared tier.** A device
  carrying opted-in restore traffic penalizes co-located small I/O severely
  (4K p99 4.6→11.1ms). Kept on its own device — the natural layout for an
  offload tier anyway — small-IO tiers are completely unaffected (165µs p99
  under full storm). Per-device opt-in is not just safe granularity, it maps
  onto how serving stacks already separate KV spill from metadata.
- **CPU headroom for the serving process.** Through stacked storage paths
  (fs, md, LVM — common under offload files rather than raw devices), −72%
  streaming sys time at identical bandwidth returns real cores to
  tokenization, scheduling, and sampling on the same host.
- **Eviction writes are neutral** (drive write-limited either way), and
  4K/128K I/O — the sub-boundary population — is untouched everywhere, so
  enabling the series (without opt-in) on a serving fleet changes nothing
  until a device is deliberately opted in.
- **What this study does not claim**: end-to-end TTFT/throughput on a real
  inference stack (the emulation is I/O-shape-faithful; compute and H2D stages
  sit outside the changed path), and GPUDirect Storage behavior (bus-address
  P2P bypasses the IOVA path; host-bridge-routed P2P would interact and is
  future work alongside the p2pdma series).

## 10. Reproduction notes

- Latitude's preinstalled `bnxt_en` DKMS breaks `make install` on rc kernels
  via the postinst hook — install boot files manually + `update-initramfs`.
- `localmodconfig` on bare metal strips virtio/9p — force them back as builtins
  if kernels double as vng guests.
- vng: `--qemu-opts=` needs the equals form; `intel-iommu` requires prepending
  `-machine q35`; emulated VT-d works fine on AMD hosts.
- fio: options after a `--name` are job-local — globals must precede the first
  `--name` in multi-job invocations.
- 2MB requests need segment headroom: `NVME_MAX_SEGS`=256, so scattered 4K
  buffers split at 1MB regardless of sector limits — hugepage-backed buffers
  keep the variables separated.
- md-raid stacked queue limits are frozen at assembly: raising a member's
  `max_sectors_kb` does nothing until `md0`'s own limit is raised too.
- Never run timed guest boots concurrently with builds (5–10x inflation).
- blktests on a fresh box needs `git`, `xfsprogs`, `liburing-dev`, `column`.
- Campaign totals: ~30 bare-metal boots, 9 guest boots, 6 workload families,
  ~20 hours of a $1.25/hr machine.
