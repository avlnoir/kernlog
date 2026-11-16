# syzbot: VM_BUG_ON_FOLIO(folio_mapped) in filemap_unaccount_folio

Report: [syzbot 17a207d226b8a5fb0fd9](https://syzkaller.appspot.com/bug?extid=17a207d226b8a5fb0fd9), open since 2023. Investigated 2026-10-29/30; not posted upstream.

## Summary

The bucket groups unrelated triggers under one assertion: page-cache deletion finds a folio that is still mapped. Three instances were examined; they do not share an established root cause.

| Instance | Path | Status |
| --- | --- | --- |
| 2026-03-13 linux-next `5c9e55fecf93`, shmem + userfaultfd | `shmem_evict_inode()` | Reproduced 5/5; covered by a known fix |
| 2026-08-08 `a59f57e2aa12`, `/dev/nullb0` shared mapping + `BLKZEROOUT` | `truncate_bdev_range()` | Not reproduced; root-cause candidate only |
| 2026-10-23 `2a08195e4db5`, KVM guest_memfd | `kvm_gmem_punch_hole()` | No C reproducer; root-cause candidate only |

## March shmem instance: known fix

The public C reproducer hit `kernel BUG at mm/filemap.c:155` in 5/5 fresh boots of the exact syzbot kernel. The folio was shmem, order 0, index `0x470`, `mapcount:1`, and the kernel was not tainted before. In that tree, `mfill_state` in `mm/userfaultfd.c` was initialized without `.len = len`, so `uffd_mfill_lock()` checked the VMA end with length 0 while the fill loop used the full length. A request crossing an already-unmapped VMA end therefore filled pages outside the VMA. Harry Yoo's [fix](https://groups.google.com/g/syzkaller-bugs/c/ITw6SXEddYQ) was folded into `db0062d2c035`. On `d135ab453a74` (2026-10-26), `UFFDIO_ZEROPAGE` returns `ENOENT` and three runs were clean. The one-line fix was not tested alone on the old commit.

## August and October instances: candidate

Both configs have `PREEMPT_RT`, `PT_RECLAIM` and `DEBUG_VM`, and the inputs use advice 24 (`MADV_DONTNEED_LOCKED`), which sets `reclaim_pt`. In the old `zap_pte_range()`, rmap updates delayed until TLB flush could happen after the PMD was cleared, and `zap_pmd_range()` skipped a `pmd_none()` PMD in the `single_folio` case. Truncation's `unmap_mapping_folio()` could then find empty page tables while `folio_mapped()` was still true. Baolin Wang's `0bb02bb1a72a` moves the rmap flush before clearing the PMD.

This is only a candidate. The August crash iteration shows no advice 24 call, the October report has no concurrent stack, and the August C reproducer did not trigger in 90 s runs on either `d135ab453a74` or the original RT kernel (static and dynamic builds). Without a positive baseline, a clean run with the patch says nothing.

## Impact

Observed only as a `DEBUG_VM` BUG; syzbot's configs add `PANIC_ON_OOPS`. The triggers need a writable block device or `/dev/kvm`. The dashboard's "unprivileged" AI label is not backed by a permission analysis. Production-config consequences were not examined.
