# How the per-domain rcache RFC overcomes the IOVA rbtree lock

Companion to [iova-feasibility.md](iova-feasibility.md) (why it's safe) and
[rfc-benchmarks.md](rfc-benchmarks.md) (measured results): this note explains
the *mechanism*. The short version: the patches don't make the rbtree faster
and don't shard it — they **remove its traffic** by widening an existing fast
path until the lock stops being on the I/O path at all.

## The mechanism that was already there

Every IOVA allocation goes through this funnel in `alloc_iova_fast()`
(`drivers/iommu/iova.c`):

```
size < (1 << (RANGE_CACHE_MAX - 1)) ?          <- the gate: 32 pages = 128KB
  yes -> roundup to pow2 -> iova_rcache_get()  <- per-CPU magazines
           loaded magazine has one?  pop, done    (per-CPU spinlock, ~0 contention)
           prev magazine?            swap, pop
           depot has a full mag?     per-class depot lock, pop 127 at once
  no  -> alloc_iova()                          <- global iova_rbtree_lock, tree walk
```

and the mirror on free: cacheable sizes go `free_iova_fast() ->
iova_rcache_insert()` (push into the per-CPU magazine); everything else goes
`free_iova()` — global lock, `private_find_iova()` lookup, `rb_erase()`.

The rcache is a magazine allocator: two 127-entry magazines per CPU per size
class, with a per-class depot exchanging *full* magazines — so even the
depot lock is touched once per 127 operations. When a size class is covered,
the global lock is only involved in **first-fill** (initially carving IOVAs
out of the tree) and depot trimming by the 100ms reaper.

The entire problem is the gate: it is a compile-time constant —
`IOVA_RANGE_CACHE_MAX_SIZE = 6`, six classes, 128KB at 4K pages. A 2MB
request fails the `size <` check on *both* sides: every allocation walks the
tree under the global lock, and, worse, every completion's deferred free
arrives via `fq_ring_free()` from **per-CPU flush-queue timers**, each drain
taking that one lock per entry, in timer context. That is the 448-core
lockup of 3710e2b056cb: hundreds of timers convoying on one spinlock, 85ms
single-acquisition waits. It is also why the clamp exists —
`dma_opt_mapping_size()` literally returns this gate's value, and nvme-pci
sizes itself to stay under it.

## What the six patches change

**Patch 1 — per-domain bound.** `IOVA_RANGE_CACHE_MAX_SIZE` becomes
`iovad->rcache_max_size`, with all class structures up to a hard limit
(2^10 pages) initialised at domain creation. That last detail is the trick
that killed every previous attempt: since the arrays, locks, and per-CPU
structs for the large classes *already exist*, "growing" the domain is
publishing one integer (`cmpxchg`, grow-only, `READ_ONCE` at the gates). No
live resizing, no unbind/reattach — the reconfiguration UX that sank the
2021–22 series after Will Deacon had already Acked it.

**Patch 2 — power-of-two insert guard.** The safety proof for concurrent
growth. An allocation that raced the bound update may be an *unrounded* odd
size (allocated under the old bound, freed under the new one);
`order_base_2()` would file it into a class of larger IOVAs — handing out
overlapping ranges later ("freeing odd-sized IOVAs back into the caches
causes havoc", the recorded objection to live resize). The insert path now
refuses non-power-of-two sizes, so the race degrades to "this one IOVA
takes the rbtree free path, as it always did" instead of address-space
corruption.

**Patch 3 — lazy magazines.** Answers the memory objection
(pay-for-what-you-use, and the 2026 fleet reports of multi-TiB iova slab):
magazines for classes above the default are allocated per-CPU on first
insert. A NIC's domain, or a CPU that never touches 2MB I/O, pays nothing.

**Patches 4–5 — plumbing.** `->opt_mapping_size()` learns which device is
asking, and `dma_set_opt_mapping_size(dev, size)` lets a driver grow its
domain's cached range — refused if any IOMMU-group member is 32-bit-masked,
because caching large ranges keeps sub-4GB frees out of the rbtree, which
is the only place the `max32_alloc_size` fail-fast is reset; a 32-bit
device sharing the domain could otherwise starve.

**Patch 6 — nvme self-sizing.** One call in nvme-pci: hint the transfer
limit *before* reading `dma_opt_mapping_size()` back. The existing clamp
line then self-sizes — `opt == max` for that device, no sysfs knob, no
driver-visible behaviour change anywhere else.

## The effect, in the funnel

After the hint, a 2MB allocation passes the gate: alloc pops a per-CPU
magazine; free — **including from `fq_ring_free()` in timer context, the
exact 448-core stack** — pushes into a per-CPU magazine. The global lock's
remaining traffic is first-fill and reaper housekeeping.

The [three-way measurement](rfc-benchmarks.md#5-lock-contention-the-reason-the-clamp-existed)
shows exactly this shape at identical device command counts (~209K, verified
via `nvmeq->sq_lock` acquisitions):

| | opted-in stock allocator | RFC |
|---|---|---|
| `iova_rbtree_lock` acquisitions | 436,753 (2 global-lock trips per I/O) | 46,228 (first-fill + reaper) |
| contentions | 2,886 | **16** |
| per-CPU cache ops / contentions | 61K / 0 | 452K / **0** |

And the reason it scales where the old world couldn't: contention on the
global lock grew with CPU count × drain depth (the 85ms waits at 448
cores), whereas per-CPU magazines have no cross-CPU waiter to multiply.
The convoy term isn't reduced — it is structurally absent. The rbtree
remains what it was always good at: the cold path that hands out address
space in bulk, 127 PFNs at a time, a few times a minute.
