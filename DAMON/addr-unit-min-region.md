# DAMON_LRU_SORT/RECLAIM minimum region size ignores addr_unit

Source: mm-new `d865f2677c44` (2026-10-27). Not posted upstream.

## Problem

`mm/damon/lru_sort.c:175` and `mm/damon/reclaim.c:171` set the scheme's minimum region size to `PAGE_SIZE`, while region addresses and lengths are in `addr_unit`. With 4 KiB pages and `addr_unit=4096`, the minimum becomes 16 MiB instead of one page, so schemes skip every region smaller than that.

## Result

Four fresh QEMU TCG boots (`addr_unit` 1, 4096, 4096, 1). Each guest populated 64 MiB of anonymous memory and pointed DAMON_LRU_SORT at the 2 MiB physical range containing it, with endpoints scaled by `addr_unit`. At the third one-second snapshot:

| addr_unit | Tried bytes | Applied bytes |
| ---: | ---: | ---: |
| 1 | 37,748,736 | 2,727,936 |
| 4096 | 0 | 0 |

Both runs of each setting gave the same numbers. The kernel image was built from `085d70326dec`; the relevant module and paddr sources are identical to the reviewed tree.

## Limits

- Applied bytes count `folio_activate()` calls; LRU membership and reclaim behaviour were not observed directly.
- DAMON_RECLAIM has the same code but was not run.
- A fix would convert one page to `addr_unit` when building the scheme, keeping granularity and alignment constraints. It touches only parameter setup, not per-page work. No fix was written.
