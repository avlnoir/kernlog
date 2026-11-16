# MGLRU debugfs eviction accepts a wrapped sequence number

Kernel: mainline `067c5c24cef2` (2026-11-10). Not posted upstream.

## Problem

The `lru_gen` debugfs command `- memcg_id node_id seq [swappiness [nr_to_reclaim]]` should reject a `seq` newer than `max_seq - MIN_NR_GENS`. `run_eviction()` in `mm/vmscan.c` checks this as:

```c
if (seq + MIN_NR_GENS > max_seq)
	return -EINVAL;
```

`seq` is parsed with `%lu` and is not bounded anywhere else. With `seq = ULONG_MAX`, `seq + 2` wraps to 1 and passes the check. `run_eviction()` then compares the unwrapped `seq` with the oldest generation and keeps calling `evict_folios()` until the reclaim limit or another stop condition.

## Result

QEMU/KVM, x86-64, `CONFIG_LRU_GEN_ENABLED=y`, memcg disabled:

- With `max_seq=3`, `- 0 0 3 100 1` returns `-EINVAL`, but the same command with `seq=ULONG_MAX` succeeds.
- After reading 16 MiB of file data and advancing `max_seq` to 5: `seq=5, nr_to_reclaim=1` → `-EINVAL`; `seq=ULONG_MAX, nr_to_reclaim=1` → success, and one file page is reclaimed. `pgsteal_file`, `pgscan_file` and the proactive counters each went 0 → 1, and `nr_file_pages` went 4354 → 4353.

## Impact

`scan_folios()` still stops at the two youngest generations, so protected generations are not evicted. The debugfs file is 0644 and normally needs admin access, and a valid `seq = max_seq - 2` already allows reclaiming the eligible old generations. This is wrong input validation for a privileged control, not a security issue.

## Fix direction

Compare without addition, for example reject `seq > max_seq` first and then `max_seq - seq < MIN_NR_GENS`. No fix was written.
