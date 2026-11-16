# DAMON pages out memory a GPU is actively reading through HMM

Kernel: Arch `7.3.4-arch1-2`, x86-64. GPU: RTX 4060 Laptop (AD107), `nvidia-open 620.70.06`, `nvidia_uvm` defaults (access counters off). Swap: 8 GiB zram. Not posted upstream.

## Question

Once a device maps host memory through its own page tables (HMM/SVM), device accesses no longer set the CPU accessed bit. Does DAMON then treat that memory as cold, and does this cause real damage?

## Setup

A 512 MiB anonymous buffer filled with random data (so zram cannot shortcut same-filled pages), read in full by a GPU kernel through HMM over and over (about 40 ms per launch). Control: the CPU reads the same buffer with `zlib.crc32`. DAMON `fvaddr` monitored only that buffer: 5 ms / 100 ms / 1 s intervals, 10–1000 regions, a pageout scheme for `nr_accesses == 0` and `age >= 5 s`, plus a STAT scheme for snapshots. Each run: 10 s baseline, then 60 s with or without DAMON.

## Result

| Run | DAMON's view of buffer | Paged out | pswpout / pswpin (pages) | GPU throughput (baseline → DAMON) |
| --- | --- | ---: | --- | --- |
| GPU + DAMON #1 | all 512 MiB `nr_accesses=0` | 2472 MiB | +632906 / +632891 | 12.56 → 12.09 GB/s (−3.8%) |
| GPU + DAMON #2 | all 512 MiB `nr_accesses=0` | 2407 MiB | +616275 / +616246 | 12.51 → 12.07 GB/s (−3.5%) |
| GPU, no DAMON | – | – | 0 / 0 | 12.46 → 12.43 GB/s (−0.2%) |
| CPU read + DAMON | mostly `nr_accesses=1` | 0 | 0 / 0 | 3.76 → 3.84 GB/s |

- DAMON saw the whole buffer as cold. It first paged it out about 11.7 s after starting, then roughly every 10 s: about 4.8 times the buffer size in 60 s.
- The GPU faulted everything straight back. Swap-in matched swap-out and `VmSwap` stayed 0, so reclaim freed nothing and only caused round trips.
- GPU throughput dropped during each pageout window, to a minimum of about 10 GB/s per launch.

## Cause

DAMON's young/mkold checks call `mmu_notifier_clear_young()`/`test_young()`, but UVM, like every device SVM/ODP driver in the tree except KVM, does not implement them. They also cannot be implemented by scanning GPU page tables. Public NVIDIA (VER2/VER3), AMD (GFX9–GFX12) and Intel (xe/i915, Arc PRM) PTE definitions have no hardware accessed bit, and no driver reads one. Alternatives are access counters (see [nvidia-uvm-access-counters.md](nvidia-uvm-access-counters.md)) or unmap-and-refault sampling.

From the source only, not measured: MGLRU cannot protect these pages either. `PG_workingset` needs rmap to see an access twice, and refaults land in unprotected tier 0. DAMOS pageout goes through `reclaim_pages()` with `ignore_references=true`, bypassing LRU reference checks.

## Limits and status

- The damage here is small: zram swap-in is fast and there was no other memory pressure. Slower swap, file writeback, tiered-memory demotion or shorter ages would likely hurt more; not tested.
- `min_age` 5 s amplifies the effect. DAMON_RECLAIM defaults to 120 s and has quota and PSI goals. System-wide paddr DAMON_RECLAIM was not run.
- kdamond, zram and GPU fault-handling CPU time were not measured.
- No simple and correct kernel fix was found, and the impact looked small, so this direction was paused.
