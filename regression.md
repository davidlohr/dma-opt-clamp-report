# The −8% large-command regression: a four-phase investigation

**Question**: the study found that opting the Samsung PM9A3 into 2MB requests
costs ~8% of streaming bandwidth, while the Micron 7450 showed no such loss.
Is that penalty a universal property of commands at a drive's maximum
transfer size (MDTS) — meaning every drive pays it at *its* limit — or is it
PM9A3-specific firmware behavior?

**Answer, up front**: it is **not universal, and it is bidirectional**. The
PM9A3 penalty is real, portable across units and conditions (−4.6% to −7.0%
at its 2MB MDTS in every arm tested), and lives in the controller's command
handling, not in NAND access. The Micron 7450 does the opposite: with
byte-exact 4MB commands at its own MDTS it *gains* +31–35% over 512K
commands, on two physical units, on written and unwritten media alike. The
"every drive pays at its MDTS" hypothesis — this document's own starting
prediction — was tested and falsified.

All numbers below were recomputed from the raw logs
(`data/mdts-probe{,2,3,4}.log`) at writing time; medians of n=5 with
worst-case spreads noted where they matter.

## 1. The original observation (measurement campaign, box #1)

Bare metal, translated AMD-Vi domain, custom kernels, preconditioned PM9A3,
hugepage-backed fio buffers (medians, n=3):

| case | 128K commands | opt-in | Δ |
|---|---|---|---|
| seq 2MB QD8 | 6607 | 6064 | **−8.2%** |
| rand 1MB QD16 | 6608 | 6554 | −0.8% |
| seq 2MB, opt-in kernel forced back to 128K (`probe128`) | 6603 | — | isolates kernel code: not the variable |
| seq 2MB at QD16/QD64 (`probeqd`) | — | 6109 | queue depth does not recover it |

Two later experiments on the same box reproduced the signature accidentally:
a 2MB request from a `malloc` buffer, split by the segment limit into two 1MB
commands, ran at 6535; the same request as a single 2MB command ran at 6055.
Meanwhile the Micron 7450, tested through a file over its md-raid1 OS mirror,
showed flat bandwidth. This produced the initial (wrong) framing
"drive-specific quality difference", later sharpened to the (also wrong)
prediction "every drive pays at its own MDTS".

## 2. Why the first two follow-up attempts were invalid

A fresh box was provisioned (same hardware class: 2× PM9A3 1.92TB blank,
2× Micron 7450 480GB carrying the OS mirror; stock Ubuntu kernel,
`iommu=pt`). The first two experiment designs contained disqualifying
confounds — both caught by review before conclusions were drawn from them:

1. **Deallocated-LBA reads measure the zero-engine, not media.** The fresh
   PM9A3 was factory-blank and the Micron mostly unwritten; reads of
   deallocated LBAs are served from controller metadata without touching
   NAND. (This turned out to be salvageable as a *deliberate* arm — see
   phase 1 — because a penalty that appears on the zero path cannot be a
   NAND effect.)
2. **The file-over-md arm measures two drives, not one.** md-raid1
   load-balances reads across both mirror legs: the file arm peaked at
   9826 MiB/s, ~2× the single-leg ceiling. This also retroactively explains
   the original campaign's "flat" Micron result — that method never isolated
   a single drive's command-size response. A second defect in the same arm:
   command sizes could not be cleanly controlled through fs+md (verified
   average command size reached only 2422KB at a 4096KB setting).

Two further methodological rules were adopted for all subsequent arms:
**achieved command size is verified per run** from `/proc/diskstats` deltas
(never assumed from the sysfs setting), and full-size commands are
guaranteed by hugepage-backed fio buffers (`iomem=mmaphuge`), since
`malloc`-backed buffers fragment against the segment limit and produce
blends (e.g. avg 1638KB at a 2048KB setting).

## 3. Experimental design (final form)

Four phases on the fresh box, read-only throughout on OS-carrying devices
(`fio --readonly` as a hard guard), 4 jobs × QD8 over disjoint regions, 20s
runs + 3s ramp, n=5 per point, sizes interleaved within each rep so drift
spreads evenly across sizes:

| phase | Samsung arm | Micron arm |
|---|---|---|
| P1 | blank (zero path), `malloc` buffers | mostly-blank raw leg, `malloc` |
| P2 | **preconditioned** (160GB written), `malloc` | file over md (invalidated, kept for the record) |
| P3 | preconditioned + **hugepage** (clean commands) | raw active leg + hugepage |
| P4 | **second unit**, blank + hugepage | **mirror deliberately degraded** (user-approved): freed leg preconditioned with 120GB of real writes, then swept raw with clean commands; plus a second region as a nominally-blank arm |

Geometry (captured this time): PM9A3 `mdts=9` → 2MB; 7450-480GB `mdts=10` →
4MB; `noiob=0` on both (no advertised optimal boundary that would explain a
slow path). Thermal excluded: 35°C composite on the Micron immediately after
the heaviest phase (throttle thresholds are ~70°C+).

## 4. Results

### Samsung PM9A3 — the dip is in every arm

Medians (MiB/s); "dip" is the 2MB (at-MDTS) point vs the 512K point of the
same arm:

| arm | 512K | 1M | 2M | dip |
|---|---|---|---|---|
| P1 blank, malloc (unit A) | 6725 | 6645 | 6252 | **−7.0%** |
| P2 written, malloc (unit A) | 6308 | 6282 | 5984 | **−5.1%** |
| P3 written, hugepage (unit A) | 6309 | 6248 | 6018 | **−4.6%** |
| P4 blank, hugepage (**unit B**) | 6725 | 6726 | 6310 | **−6.2%** |
| campaign box (third unit, written, hugepage) | — | 6608* | 6064 | −8.2% |

\* rand1m vs seq2m across cases; the campaign's 1MB and 512K points are
within noise of each other.

Spreads are tight (the widest interval in the table is 6196–6333 for P1 at
2MB). Two cross-checks give high confidence in the harness:

- **Cross-unit identity on the blank path**: unit A (P1) and unit B (P4)
  both measure *exactly* 6725 at 512K — different physical drives, different
  buffer types, same value to four digits.
- The clean-command arms report achieved command size == the setting,
  byte-exact, in every row.

Because the dip appears on the **blank (deallocated) path** — which never
touches NAND — with the same magnitude as on written media, the mechanism is
in the controller's **command/DMA handling at maximum transfer size**, not
in flash access or internal striping of media reads. It also survives:
custom vs stock kernel, translated vs passthrough IOMMU, blended vs clean
commands, queue depths 8–64, three physical units, and two provisioning
generations of the same machine class.

### Micron 7450 — monotonic gain through its MDTS

Clean commands (achieved == set, byte-exact), medians:

| arm | 512K | 1M | 2M | **4M = its MDTS** | 512K→4M |
|---|---|---|---|---|---|
| P3 raw leg, hugepage (unit A) | 3629 | 3813 | 4274 | **4746** | **+30.8%** |
| P4 **written** raw, hugepage (unit B) | 3468 | 3666 | 3880 | **4682** | **+35.0%** |
| P4 second-region raw, hugepage (unit B) | 3442 | 3611 | 3840 | **4624** | **+34.3%** |

No dip anywhere; the largest command is the fastest in all three arms, on
both units. The written arm is the decisive one: media state is known with
certainty (we wrote it ourselves minutes earlier), commands are byte-exact,
the device is raw and single, and the box was otherwise idle.

## 5. Anomalies, stated honestly

1. **P1's Micron arm is flat at ~5030 MiB/s at every size** — higher than
   any later single-leg arm. The most economical explanation: in P1 the
   test region was still in its deallocated/pristine mapping state and was
   served from the fast metadata path (flat, size-insensitive, like the
   Samsung zero path but at this controller's ceiling); P2 then wrote a
   100GB file through the mirror, landing real data in the same LBA ranges,
   after which reads became NAND-bound and size-sensitive. This is a
   consistent story but the box was destroyed before it could be
   dispositively confirmed; it does not affect the at-MDTS conclusion,
   which rests on the clean-command arms.
2. **P4's "blank" Micron region matches the written region within 1.3%** —
   almost certainly because a mirror member's LBA space is written (by the
   initial md resync and/or the mirrored file), so the intended
   written-vs-blank axis collapsed on the Micron: both P4 arms read written
   media. The axis was properly realized only on the Samsung (via the
   never-touched second unit).
3. **The 480GB 7450's absolute bandwidth (~3.4–4.7 GiB/s NAND-bound)** is
   parallelism-limited by its capacity class; nothing here compares
   absolute ceilings across the two models — only each drive's *response to
   command size*.

## 6. Verdict and guidance

- The PM9A3's at-MDTS penalty is **real, portable, and firmware-located**:
  −4.6% to −8.2% across every condition tested. Avoid it by opting in one
  notch below MDTS: `max_sectors_kb=1024` keeps full streaming bandwidth
  (−0.8%) and still cuts command count 8× vs the 128KB clamp.
- The 7450 **rewards** maximum-size commands: `max_sectors_kb=4096` is
  worth +31–35% on this drive versus stopping at 512K, and ~+21% versus
  stopping at 2MB.
- Therefore: **sweep the opt-in value per drive model**; the behavior at
  and near MDTS is firmware-specific and can point in either direction.
  In shadow-vIOMMU guests, 2048 remains the right value regardless of
  drive, because the IOMMU superpage effect (10×, study §7.3) dominates
  firmware effects, and 1MB requests cannot map as superpages.
- For the patch series, this is an argument *for* its shape: a per-device,
  admin-chosen value is exactly the right knob for bidirectional
  firmware behavior; no single default can be right.

## 7. Artifacts

- `data/mdts-probe.log` — phase 1 (geometry capture + blank-path sweeps)
- `data/mdts-probe2.log` — phase 2 (preconditioned Samsung; invalidated
  file-over-md Micron arm, kept for the record)
- `data/mdts-probe3.log` — phase 3 (hugepage clean-command arms)
- `data/mdts-probe4.log` — phase 4 (degraded-mirror written Micron, second
  Samsung, all clean commands)
- Campaign-era anchors: `data/box-artifacts.tgz` (fio JSON per case/rep)

Hardware: Latitude m4-metal-medium (EPYC 9124), 2× Samsung PM9A3 1.92TB
(MZQL21T9HCJR-00A07, mdts=9), 2× Micron 7450 480GB (MTFDKBA480TFR,
mdts=10), Ubuntu 24.04 stock kernel, `amd_iommu=on iommu=pt`. Total
machine time for the follow-up: under three hours.
