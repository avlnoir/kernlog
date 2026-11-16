# remove_migration_pmd() warns on read-only device-private PMDs

Kernel: mainline `cbbddb6b08cc` (2026-10-31). Introduced by `65edfda6f3f2` ("mm/rmap: extend rmap and migration support device-private entries", v6.19-rc1). Not posted upstream.

## Problem

In `remove_migration_pmd()` (`mm/huge_memory.c`), when the target folio is device-private, `pmde` is converted to a swap-format entry before this check runs:

```c
VM_WARN_ON_ONCE(pmd_write(pmde) && folio_test_anon(folio) &&
		!PageAnonExclusive(&folio->page));
```

`pmd_write()` means nothing on a swap entry. On x86, `_PAGE_SWP_SOFT_DIRTY` is the same bit as `_PAGE_RW`, so a read-only device-private PMD with soft-dirty set looks writable. `remove_migration_pte()` does not have this problem.

The trigger is an anonymous PMD-mapped THP that is not AnonExclusive (for example after `fork()`), with a soft-dirty migration entry, migrated to device-private memory through `migrate_vma_*`. It needs `CONFIG_DEBUG_VM` and a device-private driver (nouveau, xe or test_hmm).

## Result

- Baseline: triggered in 2/2 QEMU runs with test_hmm. Two controls (an exclusive THP, and a shared THP with soft-dirty cleared first) migrated 512 pages without a warning.
- Moving the rmap update and the check before the device-private conversion: 0/2, and the post-migration page-table snapshot matched the baseline. The mm `hmm-tests` TAP output was identical before and after (42 skipped for missing coherent device, hugetlb or gup_test).
- With `DEBUG_VM` off, `remove_migration_pmd()` grew by 2 instructions (+21 bytes). Runtime cost was not measured.

## Impact

Debug builds only. It is a WARNING since v7.3-rc1 (`8e658ecc3be2`), a `VM_BUG_ON` from v6.19 to v7.2, and a panic with `panic_on_warn`. The PMD that gets installed is correct.

## Related, unconfirmed

`migrate_vma_collect_hole()` and `migrate_vma_insert_huge_pmd_page()` only check alignment and `thp_vma_suitable_order()` before installing a PMD-sized device page. They do not check `VM_NOHUGEPAGE` or `PR_SET_THP_DISABLE`. This is a policy question and was not reproduced.
