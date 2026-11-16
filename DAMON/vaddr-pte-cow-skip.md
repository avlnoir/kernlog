# DAMON vaddr skips a COW page inside a large folio's PTE range

Source: mm-new `d865f2677c44` (2026-10-27). Run on a kernel built from `085d70326dec`, where the two affected functions are byte-identical. Not posted upstream.

## Problem

When iterating PTEs, `mm/damon/vaddr.c` advances over the remaining pages of a folio based on the folio's physical size (`vaddr.c:1054-1055` in STAT/filter handling, `888-889` in migration). It assumes the following virtual PTEs still map that folio. A single-page COW breaks that assumption: the parent keeps the large folio, and one PTE in the middle now maps a separate order-0 folio. That page is skipped.

The September fix `359093bdffbb` handles starting in the middle of a folio. It does not cover a folio whose PTE mapping is interrupted.

## Result

Reproduced in a QEMU TCG guest. A 2 MiB anonymous THP was PTE-mapped (one-page `mprotect()` splits), then after `fork()` the child wrote page 1, giving the layout THP head / separate order-0 page / THP tail. `/proc/self/pagemap` and `/proc/kpageflags` confirmed it before and after the test. An `fvaddr` STAT scheme with a `hugepage_size` filter that only admits 4 KiB folios reported:

| Region starts at | Expected | Observed |
| --- | ---: | ---: |
| THP start | 4096 | 0 |
| COW page | 4096 | 4096 |

## Limits

- The migration path has the same advance, but its runtime effect was not tested.
- A fix would bound the skip by the actual consecutive PTE run (for example with `folio_pte_batch()`). It must also keep STAT accounting correct when a large folio appears on both sides of another folio, since `last_applied` only remembers the previous folio. The extra PTE checks cost work that was not measured. No fix was written.
