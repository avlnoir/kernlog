# Size-normalized scoring for DAMON access reports

Base: Ravi Jonnalagadda's [RFC v4](https://lore.kernel.org/all/20261005-damon-perf-rfc-v3-send-2026-10-03-v4-0-b03452e137f3@gmail.com/) (snapshot `92e67371c34e`). Mainline v7.4-rc1 has no report scoring path. The idea was raised as a question in the [report infrastructure design thread](https://lore.kernel.org/damon/20261017195747.89961-1-sj@kernel.org/); the data below was not posted.

## Problem

With report-based probes, a region scores a hit if any of its pages is reported in the sampling interval. If each page is reported with probability q, a region of s pages hits with probability 1 - (1 - q)^s. Large regions therefore score higher than small regions with the same access density. The page-table accessed-bit check samples one page per region and has no such size bias.

## Variants compared

- **E**: RFC v4 rule, at most one hit per region per interval.
- **C**: reported bytes / region bytes, in fixed point.
- **U-fx**: distinct reported pages / region pages (needs per-interval page de-duplication).
- **U-bn, S**: integer-count versions (random rounding; one random page checked against reports).
- **A**: one random page per region against all real accesses. Reference only; it sees about four times as many accesses as the reports carry.

## Results (offline replay, synthetic traces)

1. **Size bias confirmed.** In the contiguous hot-set workload, cold-only regions with a true access density of 0.005 and 25% report sampling: E's hit rate rises from 0.00125 (1 page) to 0.30 (256 pages) and 0.997 (4096 pages). C and U-fx stay at 0.00125 for every size.
2. **Adaptive-region closed loop** (5 ms sample, 100 ms aggregation, min 64 regions; parameters tuned on training seeds, then frozen): held-out macro F1 went from 0.217 (E) to 0.343 (C) and 0.342 (U-fx). Almost all of the gain comes from the contiguous hot-set workload (F1 0.513 → 0.995). Scattered, bursty and moving workloads stay at 0.07–0.12 under every adaptive scoring rule; that is a spatial-resolution limit, not a scoring issue.
3. **The counter must be wider than u8.** The integer versions (U-bn, S) only reach about 0.26. Cold regions hit about 0.00125 per interval, which 20 samples per window cannot resolve. Mainline `probe_hits` is `unsigned char`.
4. **C is as good as U-fx** in these workloads and needs no per-interval de-duplication.
5. Normalized scores let neighbouring cold regions merge much more easily: about 185 regions on average versus about 1266 for E. This helps the contiguous hot set; its effect on other distributions is unknown.

## Kernel cost (microbenchmark)

Prototype on the RFC v4 snapshot: `probe_hits`/`last_probe_hits` widened to u32, each report adds 1e6 × report size / region size (saturating). A "lean" version also drops the now unneeded per-interval de-duplication fields. Setup: QEMU/KVM, 1 vCPU pinned to a P-core, 12 rounds per variant, with an A/A control within ±3%.

- Scoring itself is cheap. With 64 regions and 255 reports, the drain path is 7% faster because it no longer reads and writes the per-region de-duplication mark.
- The larger `struct damon_region` costs more: 80 → 104 bytes. Per-region walks get slower: reset +12%, merge +6%, wsum scan +3–6% (2048/8192 regions). At 512 regions, reset is +45% and merge +16%. That is probably because the baseline's 41 KiB fits in a 48 KiB L1d and the prototype's 53 KiB does not; this was not confirmed with hardware counters.
- The lean version (88 bytes) is not better everywhere: wsum scan is +75% at 2048/8192 regions even though its code is effectively identical. The cause is unknown.
- Estimated at 8192 regions, 5 ms sampling, 100 ms aggregation: about 0.2 ms extra CPU per second and 192 KiB extra memory. This is extrapolated from the microbenchmarks, not measured end to end.

## Limits

Synthetic traces with uniform cold density, ideal report delivery, order-0 pages only, no migration. Scoring semantics follow the RFC v4 snapshot; the final report API may differ. The microbenchmark lays regions out sequentially and does not cover the NMI producer path or multi-CPU rings.
