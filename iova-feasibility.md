# Can the IOVA allocator make `max` safe? A feasibility study of retiring the DMA-optimal clamp

Everything else in this repository treats the 128KB clamp as a given and
routes around it (per-device opt-in). This document attacks the root: the
clamp exists because IOVA allocations larger than the rcache bound take a
globally-locked rbtree, and on big machines that lock has caused watchdog
lockups. If that pathology can be engineered away, `dma_opt_mapping_size()`
stops being smaller than `dma_max_mapping_size()` and the clamp — and our
opt-in knob — become unnecessary.

Method: allocator code analysis, upstream-history reconstruction, a
one-line prototype under adversarial load, and a scaled reproduction of the
original lockup's mechanism with a before/after A/B. Kernel: tip v7.2-rc1;
hardware: 16-core/32-thread EPYC 9124, AMD-Vi translated (DMA-FQ), Samsung
PM9A3 (MDTS 2MB).

## 1. The original evidence, precisely

The clamp (commit `3710e2b056cb`, "nvme-pci: clamp max_hw_sectors based on
DMA optimized limitation", v6.4) cites a 448-core AMD server with an
MDTS=4MB NVMe under fio:

- **Soft lockup**: `fq_flush_timeout` — the DMA-FQ per-CPU flush timer —
  observed spending **31 seconds**, with a single
  `free_iova → _raw_spin_lock_irqsave` acquisition waiting **85.6ms**.
- **Hard lockup**: `free_iova_fast` spinning on `iova_rbtree_lock` from
  `nvme_pci_complete_batch` — **inside the completion IRQ handler**.

Note what this is: a **free-side** collapse. With 4MB transfers every IOVA
is above the rcache bound, so every completion's deferred free goes
`fq_ring_free → free_iova → iova_rbtree_lock`, and hundreds of per-CPU
flush timers drain their queues against one spinlock, each drain taking the
lock once per entry, in timer/IRQ context. Allocation-side contention rides
along, but the watchdog traces are frees.

The commit's fix was to advertise `dma_opt_mapping_size()` (the rcache
bound, 128KB) as the transfer cap — treating the allocator's limitation as
a device property.

## 2. The allocator today (v7.2-rc1 mechanics)

All in `drivers/iommu/iova.c` unless noted:

- **One lock per `iova_domain`** covers: the alloc walk (`rb_prev` from a
  cached node, retry pass via `iova_find_limit()`), `find_iova`, every
  uncached free (`private_find_iova` + `rb_erase` + rebalance), the
  `max32_alloc_size` fail-fast *and its only reset point* (inside
  `__cached_rbnode_delete_update()`), and — the worst critical section —
  `iova_magazine_free_pfns()`: **127 lookups + erases under a single
  hold**, called from the depot reaper, CPU-hotplug drains, and the
  alloc-failure retry storm.
- **The rcache**: per-CPU loaded/prev magazines (127 PFNs each, 1KB), a
  per-class depot with its own lock (one hit per 127 ops in cross-CPU
  flows), and since the flexible-depot rework a 100ms reaper bounding
  steady-state depot depth at `num_online_cpus()` magazines. Cached
  classes: orders 0–5, i.e. ≤128KB at 4K pages (`IOVA_RANGE_CACHE_MAX_SIZE
  = 6`); `iova_rcache_range()` → `iommu_dma_opt_mapping_size()` hard-wires
  that constant into `dma_opt_mapping_size()`.
- **Cached PFNs stay resident in the rbtree** (the magazine free looks
  them up by PFN and would WARN otherwise), so cache capacity also sets
  tree depth for whoever does take the slow path.
- **Traffic shape**: one IOVA allocation per *request* — and this was true
  in the scatterlist era too (`iommu_dma_map_sg()` allocated one IOVA per
  sg list). The two-step DMA API did not reduce per-request allocations;
  what it universalized is the single-alloc pattern and lower per-request
  CPU cost. The clamp's real benefit was never alloc *count* — it was
  keeping every allocation *inside the cacheable size range*. Corollary:
  larger requests mean strictly fewer allocations per byte moved.
- **Sharding**: one domain per IOMMU group. A direct-attached NVMe with
  ACS gets its own domain, its own lock, its own caches. The
  counterexamples that matter: **VMD** (every drive behind the endpoint
  shares one domain), multi-disk SAS HBAs (the hardware of the original
  2021 contention reports: 128-core Kunpeng, one HBA, 12 SSDs), and
  non-ACS topologies.

## 3. The prototype: one line

```
-#define IOVA_RANGE_CACHE_MAX_SIZE 6	/* log of max cached IOVA range size (in pages) */
+#define IOVA_RANGE_CACHE_MAX_SIZE 10	/* log of max cached IOVA range size (in pages) */
```

Classes now cover orders 0–9 (≤2MB). `iova_rcache_range()` returns 2MB, so
`dma_opt_mapping_size()` returns 2MB, so **nvme-pci's existing clamp lands
at MDTS with no nvme change at all**: the test kernel boots with
`max_hw_sectors_kb=2048` and default `max_sectors_kb=2048`, no opt-in knob
involved. (With the PRP segment fix also applied, `max_segments=513`.)

### Adversarial storm (every request in the previously-uncached range)

16 jobs × 2MB × QD8 randread, 60s, 193K requests, `CONFIG_LOCK_STAT=y`:

| lock | acquisitions | contentions | max wait |
|---|---|---|---|
| `iova_rbtree_lock` | 4,238 | **3** | 1.32µs |
| per-CPU `cpu_rcache->lock` | 387,988 | **0** | — |
| depot `rcache->lock` | 4,930 | 1 | 0.52µs |

The rbtree is reduced to first-fill traffic; the three contentions are the
initial cache-population races. Mixed-size churn (256K/512K/1M/2M split,
four classes cycling, 824K cached ops) leaks only 1.7% of operations to the
rbtree, with 9 contentions. Neither storm ranks the iova locks anywhere
near the machine's background contenders (`jiffies_lock`, LRU, runqueues).

### The inversion nobody expected

The same fio spec confined to the *old* 128KB regime (same boot,
`max_sectors_kb=128`) produced **478 rbtree contentions — 160× more than
the 2MB storm** — because 17× the transaction rate churns the depot even
with every size cached, plus 3.4M extra `sq_lock` submissions for the same
bytes. And the application-visible latency of a 2MB read: p99 39.6ms
(deterministic, p50==p99) vs 112.7ms split into 128KB pieces — **2.85×
worse under the clamp**. Large requests are not the allocator's adversary;
at equal bandwidth they are its relief. The 2023 framing has it backwards
for saturating sequential-ish loads.

### Cost, measured

Dedicated `iommu_iova_magazine` slab: 31,900 magazines ≈ 31.2MB machine-wide
on 49 domains × 32 CPUs × 10 classes — **+12.75MB over the baseline's 6
classes (+256KB per domain)**. Storm-driven depot growth: tens of KB,
reaped within seconds. (The eager allocation pattern, not the total, is the
upstream-sensitive part — see §6.)

## 4. Reproducing the 3710e2b056cb mechanism, scaled

The storm above shows the *fixed* world. The reproduction kernel (`kvls`)
is the same build **without** the rcache extension: `max_sectors_kb=2048`
puts every 2MB alloc — and every FQ-drained free, the exact 448-core stack
— on the rbtree. Same fio storm (32 jobs × QD16 × 2MB, 120s), same
lock_stat, plus a bpftrace histogram over `fq_flush_timeout` itself.

32 jobs × QD16 × 2MB randread, 120s, ~387K requests, both kernels at
`max_hw_sectors_kb=2048` / `max_sectors_kb=2048`, DMA-FQ domain, lock_stat:

| metric | `kvls` (stock allocator) | `kvrc` (rcache→2MB) | ratio |
|---|---|---|---|
| `iova_rbtree_lock` acquisitions | **848,922** | 43,601 | 19× |
| `iova_rbtree_lock` contentions | **9,089** | **100** | **91×** |
| max wait | **196.9µs** | 9.8µs | 20× |
| total wait | 10.0ms | 0.23ms | 43× |
| avg hold time | 2.36µs | 0.39µs | 6× |
| per-CPU rcache (acq / cont) | 64K / 0 | 859K / **0** | — |
| bandwidth | 6464 MiB/s | 6412 MiB/s | saturated either way |
| app p99 / max | 599.8 / 712.1ms | 488.6 / 620.2ms | kvrc better |

On `kvls`, exactly as in 2023: ~849K rbtree acquisitions ≈ one alloc plus
one FQ-drained free per request, average hold 2.36µs (the >rcache walk +
`rb_erase`), 9,089 convoy events, and a 197µs worst single wait — **at only
32 threads**. On `kvrc` the same storm sends 859K operations through the
per-CPU caches with zero contentions; the rbtree sees only first-fill and
reaper traffic at 0.39µs holds. The bpftrace histograms over
`fq_flush_timeout` agree from the free side: the stock kernel's drains
cluster at 64–128µs with a tail to 256µs, the extended kernel's shift down
and lose the tail — the drain no longer serializes on a global lock.

### Scaling the measurement to the commit's numbers

The commit's 85.6ms single-acquisition wait needs no exotic explanation:
a queued spinlock hands the lock around in FIFO order, so one waiter's
delay ≈ (waiters ahead) × (their hold times). An FQ drain holds the lock
once per entry (up to 256 entries per CPU queue). With measured per-free
hold time *h* and N CPUs draining concurrently, the expected worst wait is
roughly `N × 256 × h`. Plugging in the measurement: our worst observed wait, 196.9µs at 32
threads, is ~83 lock handoffs of 2.36µs each — a convoy of a few
concurrent drains. The commit's 85.6ms wait is **36,300 handoffs** at the
same per-entry cost: ≈142 full 256-entry FQ drains queued ahead — roughly a
third of that machine's 448 per-CPU flush timers landing in one window.
The arithmetic connects our 32-thread measurement to their watchdog trace
with one multiplication; no exotic mechanism is required, and the convoy
length grows superlinearly with CPU count because each drain holds the
lock for a full ~600µs sweep (256 × 2.36µs).

That arithmetic is the feasibility core: on the extended-rcache kernel the
per-CPU term never forms — frees of cached sizes go
`free_iova_fast → iova_rcache_insert` and touch no global lock from any
context, so there is nothing for 448 CPUs to multiply. The pathology is
not attenuated; its mechanism is removed.

## 5. Upstream history: why the cache was never extended

- **John Garry tried, 2021–2022**, five versions: first
  [`dma_set_max_opt_size()`](https://lore.kernel.org/all/1616160348-29451-1-git-send-email-john.garry@huawei.com/)
  (v1, Mar 2021; +117% IOPS on hisi_sas), later pivoting to Robin Murphy's
  preferred shape — a per-group sysfs
  [`max_opt_dma_size`](https://lore.kernel.org/all/1649071634-188535-1-git-send-email-john.garry@huawei.com/)
  (v5, Apr 2022) that actually resizes the rcaches. Will Deacon reviewed
  [v4](https://lore.kernel.org/all/20210802150153.GC28735@willie-the-truck/)
  with only mechanical nits and
  [**Acked** the core v5 patches](https://lore.kernel.org/all/20220706121057.GF2403@willie-the-truck/)
  in July 2022. It was never NAKed: it died of reconfiguration UX
  (resizing required unbind + default-domain reallocation) and because the
  driver-side clamp captured the benchmark win with no IOMMU surgery —
  [`dma_opt_mapping_size()` for SCSI](https://lore.kernel.org/all/1657797329-98541-1-git-send-email-john.garry@huawei.com/)
  ([applied by hch](https://lore.kernel.org/all/20220719040635.GA26132@lst.de/),
  v6.0; with [Damien Le Moal insisting](https://lore.kernel.org/all/62b801e8-66b6-0af7-b0c9-195823bf9f62@opensource.wdc.com/)
  it cap the user-raisable soft limit, not `max_hw_sectors`), then
  3710e2b056cb for NVMe — which capped the hard limit.
- **Robin's recorded requirements** for any extension:
  [pay-for-what-you-use](https://lore.kernel.org/all/26fb1b79-2e46-09f6-1814-48fec4205f32@arm.com/)
  ("big high-throughput systems with plenty of memory can spend it on
  better performance, while small systems ... don't have to pay the (not
  insignificant) cost for caches they don't need"),
  [no decoupling of the pow-2 roundup from the caching bound](https://lore.kernel.org/all/e2d873d9-3529-caff-d4ae-cca456857ff1@arm.com/)
  (his example: a ~35MB video frame rounding to 64MB "would be hard to
  justify"), and end-user-reachable knobs over Kconfig. His own
  [flexible-depot rework](https://lore.kernel.org/all/cover.1692033783.git.robin.murphy@arm.com/)
  (v6.7, triggered by
  [Zhang Zekun's 256-CPU report](https://lore.kernel.org/all/20230811130246.42719-1-zhangzekun11@huawei.com/))
  removed the depot half of the memory objection — though
  [John re-raised the scaling math in-thread](https://lore.kernel.org/all/1aa1ecad-bdf0-84c8-a37f-94e1d0fb8a03@oracle.com/)
  (~17× more cacheable IOVAs than a 4096-deep controller needs), and Robin
  left the door open:
  ["we can absolutely come back and go to town on enhancements later"](https://lore.kernel.org/all/b9bda816-612c-b646-63e7-54cb3fedf1f4@arm.com/).
- **The original contention evidence** is per-request, shared-domain SAS:
  [Zhen Lei's revert request](https://lore.kernel.org/all/20210129092120.1482-1-thunder.leizhen@huawei.com/)
  (128-CPU Kunpeng, one hisi_sas HBA, 12 SSDs: 1655K→893K IOPS) and the
  [2019 rcache-ageing analysis](https://lore.kernel.org/all/20190815121104.29140-3-thunder.leizhen@huawei.com/).
- **The two-step DMA series never revisited the clamp**:
  [the v1 cover](https://lore.kernel.org/all/cover.1730037276.git.leon@kernel.org/)
  frames the IOVA angle as pre-computation; no thread discusses allocator
  lock pressure for block I/O (Robin's
  [initial NAK](https://lore.kernel.org/all/1166a5f5-23cc-4cce-ba40-5e10ad2606de@arm.com/)
  was about API genericity). Meanwhile
  [Luis Chamberlain's 8MB-single-I/O work](https://lore.kernel.org/all/20250320111328.2841690-1-mcgrof@kernel.org/)
  builds large-I/O support on that API without mentioning that translated
  systems still clamp at 128KB — **the collision this document addresses
  is latent and undiscussed upstream**.
- **The memory climate is currently hostile to eager caches**: an April
  2026 report from Alireza Haghdoost of **25TiB of fleet-wide `iommu_iova`
  slab** across 93 AmpereOne hosts (160-core arm64, 64K pages, DMA-FQ) has
  sharpened scrutiny. Any proposal that eagerly allocates per-CPU
  magazines for four extra classes on every domain will re-collect the
  2021 objection verbatim. Note arm64/64K pages already runs the rcache at
  2MB (`PAGE_SIZE << 5`), so the *size* is not novel — the allocation
  pattern is the issue.
- **The allocator is in play right now**: Rik van Riel (Meta) hit
  production soft-lockups in `alloc_iova()`'s linear search on AMD Bergamo
  and is iterating
  ["iova: convert from rbtree to maple tree"](https://lore.kernel.org/all/20260624030853.2340880-1-riel@surriel.com/)
  (v4, June 2026, unmerged;
  [v3](https://lore.kernel.org/all/20260603033653.4144138-1-riel@surriel.com/)).
  Robin
  [suggested the maple tree himself](https://lore.kernel.org/all/59e0476c-a2bf-42f3-8244-d8a4828da64a@arm.com/)
  ("improving the allocator has been on my to-do list for a very long
  time"), wants the `max32_alloc_size` bodge gone and 33–56-bit masks
  handled, and is "wary of growing struct iova" — but note the maple
  conversion fixes the O(n) *search*, not the single-lock serialization
  this document targets; the two are complementary.
- **Positions**:
  [Linus, Apr 2023](https://lore.kernel.org/all/CAHk-=whogEk1UJfU3E7aW18PDYRbdAzXta5J0ECg=CB5=sCe7g@mail.gmail.com/):
  "if this is actually an issue, to the point that it's now being
  discussed for a _second_ block driver subsystem, then shouldn't the
  queue handling just do this all automatically, instead of adding random
  crap to random block driver architectures?" — the clamp's *placement*
  was already contested when it landed. Robin's SAC-trick fix
  (791c2b17fb40, v6.6) has since removed the worst single rbtree
  pathology, the perpetual 32-bit retry walk.

## 6. What a mergeable fix looks like

The prototype proves the mechanism; it is not the patch to send. The
recorded objections define the shape:

1. **Per-domain cache range, set before first map** (the fatal UX of
   John's v5 was *live* resizing). At DMA-domain init the range is
   computed from the attached device(s): a domain whose masters advertise
   multi-MB `max_hw_sectors` gets classes to cover them (capped at MDTS or
   a sane bound); a USB/NIC domain stays at 6 classes and pays nothing.
   No unbind dance — the information exists at probe time.
2. **Lazy magazines above 128KB.** Allocate per-CPU magazines for the
   large classes on first use, and let the existing reaper reclaim them.
   This answers both the 2021 objection and the 2026 slab climate: idle
   large classes cost a pointer array, not 2KB × CPUs × domains.
3. **Free-side is covered by construction**: `fq_ring_free →
   free_iova_fast → iova_rcache_insert` for cached sizes — the 448-core
   trigger path never reaches the rbtree. This must be said explicitly in
   the changelog, because reviewers who remember 3710e2b will look for it.
4. **Shared-domain honesty**: VMD and SAS-HBA domains aggregate many
   drives onto one lock. The per-domain range helps them the same way, but
   their *uncached* residue (sizes between classes, cache misses under
   fragmentation) still lands on one rbtree. That residue is where the
   longer-term work goes: the vmalloc precedent (per-node zones with
   owner-computable free, merged v6.9 for the identical problem shape)
   is the community-sanctioned template if and when someone needs it. It
   is not a prerequisite for NVMe-class topologies.
5. **32-bit coexistence**: with large classes cached, sub-4GB frees stop
   passing through `__cached_rbnode_delete_update()`, so the
   `max32_alloc_size` fail-fast stops resetting and a 32-bit-masked device
   sharing the domain can starve persistently. Per-domain enablement
   (item 1) must therefore decline large classes on domains with any
   32-bit-limited master — a one-line policy, but a correctness one.
6. **Roundup waste**: accepted and bounded — this proposal caches only up
   to MDTS-class sizes (2–4MB), where pow-2 roundup waste is a fraction of
   a class, not Robin's 35MB→64MB scenario. Sizes above the range stay
   uncached and unrounded, exactly as today.

## 7. Verdict

**Is it safe to switch nvme-pci from `dma_opt_mapping_size()` to
`dma_max_mapping_size()` today, on stock kernels? No.** Mixed sizes in the
129KB–1MB band at high IOPS put both allocs and FQ frees on one rbtree
lock, and shared-domain topologies (VMD, SAS) multiply consumers onto it.
That is the 448-core lockup waiting for a big enough machine — the
reproduction in §4 shows the mechanism alive at 32 threads.

**Is the limitation fundamental? Also no — demonstrably.** One line of
allocator change makes the entire MDTS range cache-covered, at which point
the opt/max distinction *evaporates by construction*: `opt == max`, the
NVMe clamp self-lifts with no driver change, the free-side trigger of the
original lockup never reaches a global lock from any context, and the
adversarial storm that previously defined the pathology runs with **three**
contended acquisitions in a minute. Measured cost: +256KB per domain
(eager form), 2.85× better tail latency at equal bandwidth, and — the
counterintuitive bonus — 160× *less* rbtree contention than the clamp's
own 128KB regime produces at equal bandwidth.

The mergeable form is per-domain and lazy (§6), sized at domain init from
device limits. Our three-patch series remains the right *near-term* shape —
it is default-safe and needs no IOMMU changes — but the allocator fix is
the right *end state*, and the two compose: when `opt == max`, the series'
opt-in knob simply becomes a no-op. The remaining honest caveat is not the
allocator at all: it is per-drive QoS (§7.2 of the study — some firmware
serves 4K readers dramatically worse next to 2MB streams), which is a
reason for per-device *policy*, never for a global 128KB ceiling.
