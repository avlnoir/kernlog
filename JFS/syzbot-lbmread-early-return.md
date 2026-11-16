# syzbot: JFS lbmRead() returns before the read completion finishes

Report: [syzbot 6cc93ec9a4035badb85f](https://syzkaller.appspot.com/bug?extid=6cc93ec9a4035badb85f), "KASAN: use-after-free Read in copy_folio_from_iter_atomic (2)". Source: `5b0764a53235` (2026-10-24 crash). Source review and build check only; not posted upstream.

## Report

A loop worker reads a freed page in `copy_folio_from_iter_atomic()`. The page was allocated through `lbmLogInit()` and freed through `lmLogShutdown()` → `lbmLogShutdown()`.

## Candidate path

In `fs/jfs/jfs_logmgr.c`:

1. `lbmRead()` waits outside any lock with `wait_event(bp->l_ioevent, bp->l_flag != lbmREAD)`.
2. The READ completion in `lbmIODone()` clears READ and wakes the waiter, and only sets DONE at its common exit later. The reader does not take the LCACHE lock, so it can return between those two updates. An I/O error flag also makes the `!=` test true early.
3. `lmLogShutdown()` then calls `lbmDirectWrite()` on the same superblock buffer, which overwrites `l_flag` and submits a write.
4. The old read completion's final DONE update can land after the new write's flags are set. `lbmIOWait()` then takes it for write completion and returns the buffer too early. When the pool is freed, the write I/O may still reference the page.

This path fits the report and does not depend on group commit. It was not tied to the crash with a trace.

## Candidate fix

```diff
-	wait_event(bp->l_ioevent, (bp->l_flag != lbmREAD));
+	lbmIOWait(bp, 0);
```

`lbmIOWait()` checks DONE under `jfsLCacheLock`, and the completion holds that lock until after the wake and the last flags update. Only changing the condition to DONE would not be enough: the completion wakes before it sets DONE, so an unlocked waiter could miss the wakeup. Passing 0 keeps buffer ownership with the caller. Keeping `return 0` preserves the current error semantics, because some callers do not release a buffer after a failed read. All four `lbmRead()` callers can sleep and hold neither LCACHE nor gclock.

## Status

Both the original and patched `fs/jfs/` build with the same config (LLVM). The reproducer was not run, so there is no before/after KASAN result and no RT/lockdep run. This is a reviewed candidate, not a confirmed fix.
