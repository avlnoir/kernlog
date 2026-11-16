# syzbot: task hung in migrate_pages_batch behind an unfinished NBD read

Report: [syzbot f523acb98cd782b4872e](https://syzkaller.appspot.com/bug?extid=f523acb98cd782b4872e), "INFO: task hung in migrate_pages_batch (4)". Source: local HEAD `551c722f4080`; `mm/migrate.c`, `mm/compaction.c`, `mm/filemap.c`, `mm/vmscan.c` and `pagemap.h` have the same blobs as the 2026-09-19/21 crash commits. Not posted upstream.

## Report

In the two recent samples, a manual compaction (`sysctl_compaction_handler` → … → `migrate_pages_sync` → `migrate_pages_batch` → `migrate_folio_unmap`) waits more than 143 s in `folio_lock()` on the source folio. This is before the writeback wait and before the destination lock is taken. The dashboard has no reproducer or bisection, and 308 events share the title.

## Static analysis

- Simpler explanations are not supported by the stacks: a lost wakeup or check/enqueue race in `folio_wait_bit_common()` (waiters is set under the queue lock before the lock bit is checked), a source lock leaked from the async batch into the sync phase, or a permanent sleep caused by `NR_ISOLATED_*`, since `shrink_inactive_list()` throttles at most once.
- `PG_locked` has no owner. Read completion releases it through `folio_end_read()`, not `folio_unlock()`, so tracing only `folio_unlock()` misses it, and the migrate tracepoints carry no folio identity.
- Making the wait killable is not a one-line change: `folio_lock_killable()` failure needs the existing unmap cleanup, `-EINTR` is counted as a per-folio failure and does not cancel the batch, and `compact_node()` ignores `compact_zone()`'s result. It would also not explain why the lock is held so long.

## Reproducer

A bounded guest test reproduces the same wait. 64 readers issue 128 KiB buffered `pread()`s on an NBD device whose userspace server deliberately holds the READ replies for 180 s, while another task triggers manual compaction 32 times.

- No delay: all reads and compactions complete.
- 180 s delay, syzbot-derived config: the compactor sits in `folio_wait_bit_common` → `migrate_pages_batch` at 10/30/60/120/170 s. One compaction write blocks for 176.9 s, then everything completes once the server replies.
- Same, with `hung_task_timeout_secs=60`: `INFO: task compact-burst blocked in I/O wait for more than 61 seconds` with the same stack, followed by the config's hung-task panic.
- Lean config with kprobes on readahead, page lock wait and read completion: the folio the compactor waits on is one of the NBD readahead folios, and after the reply `folio_end_read()` runs on that same address. The ring buffer reported no drops up to the last check before the reply.

## Conclusion

An unfinished NBD read keeps a readahead folio locked, and synchronous compaction waits on it for as long as the I/O takes. That matches the reported hang without requiring an MM bug. The delay here was artificial. Why the original NBD requests stalled, and whether the syzbot folio was such a folio, remains unknown. No fix was written.
