# syzbot: Bad page state in hugetlb_no_page, tested against urgent_fixes 02/10

Report: [syzbot d06e9fe6d6a7b537fe49](https://syzkaller.appspot.com/bug?extid=d06e9fe6d6a7b537fe49). Patch tested: "[PATCH 02/10] mm/hugetlb: fix use-after-free of a new anon folio" (`<20261015-urgent_fixes-v1-2-42796911043b@kernel.org>`). Kernel: `d135ab453a74` with the syzbot config. Test results were not posted to the thread.

## Result

Public C reproducer, QEMU/KVM, 2 vCPU, 4 GiB:

| Kernel | Runs | Runs with Bad page | Bad page reports |
| --- | ---: | ---: | ---: |
| baseline, round A | 10 | 8 | 257 |
| baseline, round B | 10 | 10 | 243 |
| baseline + patch 02 | 10 | 0 | 0 |

The patched kernel logged no KASAN, WARNING, BUG, lockdep or stall report, and the reproducer exited cleanly in all 30 runs.

## Mechanism

- `hugetlb_no_page()` drops the PTL and only then calls `folio_set_hugetlb_migratable()` and `folio_unlock()`. After `set_huge_pte_at()`, the only reference to the folio is the page table's.
- In a private `MAP_NORESERVE` mapping no `resv_map` is allocated, so `hugetlb_vma_lock_read()` is a no-op, and `MADV_DONTNEED` does not take the hugetlb fault mutex.
- The zapper clears the PTE and drops the rmap under `huge_pte_lock()`. A surplus folio is then freed immediately by `free_huge_folio()` (HVO is not on by default in this config), while the fault path still touches it.
- The patch moves `spin_unlock(vmf->ptl)` after both folio operations. For 2 MiB pages, both sides use the same split PMD lock. `hugetlb_wp()` returns with the PTL held on every path, and nothing after `out:` touches the folio. The object code confirms the reordering.

## Limits

This covers only the path the reproducer drives (surplus page, `MAP_PRIVATE|MAP_NORESERVE`, write faults racing `MADV_DONTNEED`). A race test lowers the chance of a miss but cannot prove absence. Built with clang 23.1.4 (syzbot used 22.1.8). Other trigger paths, merge status and performance were not checked.
