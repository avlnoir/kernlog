# DAMON paddr probes do not see hugetlb folios

Kernel: mm-new `243e8cdab641` (2026-10-30); the DAMON code matched mainline `cbbddb6b08cc` apart from three unrelated hotfixes. Not posted upstream.

## Problem

`damon_pa_apply_probes()` in `mm/damon/paddr.c` looks up the sampled folio with `damon_get_folio()`, which only accepts LRU folios. Since `e67684454bbe`, the paddr access check (`damon_pa_mkold()`/`damon_pa_young()`) uses `damon_get_monitor_folio()` and handles hugetlb, and the vaddr probe handles hugetlb since `f72c1f220d3d`. The paddr probe path is the only one that gets a NULL folio for hugetlb memory, so probe filters such as `pgidle_*` and `hugepage_size` never match there.

The probe filters themselves work on hugetlb folios (`damon_folio_young()` supports them), so the exclusion does not look intentional.

## Result

Verified on an unmodified kernel through the normal sysfs interface, 4/4 runs consistent (x86-64, 2 MiB hugetlb, QEMU/KVM, KASAN + lockdep + DEBUG_VM):

- paddr with a zero-weight `pgidle_unset` probe: hugetlb regions show `nr_accesses` 20/20 but probe hits 0; THP regions show 20/20 for both.
- paddr in attribute-only mode (weight 1 + `set_pgidle` prep): hugetlb hits 0, THP hits 20.
- fvaddr: hugetlb and THP hits both equal `nr_accesses`.

The fix is to use `damon_get_monitor_folio()` in the probe path as well. A local version was started but not finished or reviewed.

## Limits

- Inferred, not measured: in attribute-only mode, region merge/split uses the weighted probe sum, so hugetlb regions are also adjusted on wrong data.
- Not tested: 1 GiB pages, PMD-shared hugetlb, arm64 contiguous-PTE hugetlb.
- DAMOS actions do not operate on hugetlb, and no effect on memory contents was seen.
