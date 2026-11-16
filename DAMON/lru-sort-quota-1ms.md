# DAMON_LRU_SORT quota_ms=1 disables the quota

Source: mm-new `d865f2677c44` (2026-10-27). Not posted upstream.

## Problem

`mm/damon/lru_sort.c` gives each of its hot and cold schemes half of `quota_ms`. With `quota_ms=1`, integer division gives 0 to both. The size quota is 0 by default and quota goals are only set when `active_mem_bp` is enabled, so `damos_quota_is_set()` returns false and both schemes run without a quota. The documentation only says that 0 disables the limit.

## Result

- The two functions, copied verbatim into a userspace harness, turn `quota_ms` 0/1/2/3/10 into per-scheme quotas 0/0/1/1/5. The quota is considered set only for inputs of 2 and above.
- Four fresh QEMU TCG boots with `quota_ms` 0, 1, 2, 1, all observed within the first quota window. With 0 and both runs of 1, DAMON kept trying regions; with 2 it stopped after the initial budget was used up. Cumulative tried bytes at the third snapshot were 9,124,507,648 versus 4,194,304. These counts include repeated work on the same memory and are not CPU time.
- The kernel image was built from `085d70326dec`. `lru_sort.c`, `paddr.c` and `modules-common.h` are identical to the reviewed source.

## Possible fix

Reject 1 when applying parameters and document 2 ms as the smallest non-zero total. Rounding each half up to 1 would allow 2 ms in total, which is not what the user asked for. A real 1 ms budget would need a finer time unit. No fix was written.
