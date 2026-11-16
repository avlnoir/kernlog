# Avoiding the list allocation for single-destination DAMOS migration

Base: mm-new `d865f2677c44` (2026-10-27). Not posted upstream; dropped.

## Idea

`damos_va_migrate()` kmallocs an array of folio lists even when there is only one destination. The change used a list on the stack when `nr_dests <= 1` and kept kmalloc/kfree for multiple destinations. The callbacks are synchronous, so the list does not escape.

## Result

- Functionally equivalent: 33 migration scenarios passed (target_nid fallback, explicit one and many destinations, zero weights, hot/cold, PTE and THP, filters, paddr), checked with `move_pages()`, full buffer contents and DAMOS statistics. Per-call kmalloc/kfree dropped from 3/3 to 0/0 for one destination.
- One behaviour difference: the old allocation-failure path disappears, so under allocation failure the new code migrates where the old code skipped.
- Cost: GCC 16 code for `damos_va_migrate` grew from 520 to 550 bytes, and the stack reservation from 120 to 144 bytes.
- Timing (4096 regions × 256 snapshots, fixed work, interleaved ABBA with an A/A control): no stable gain. The final fallback case was +2.2%, and earlier rounds were also slower. The explicit-target case was +9.5%, but the A/A spread there was 28%.

## Conclusion

The allocation saved is one small kmalloc per scheme application, which is not visible next to the migration work. The larger frame and code did not pay off in any measurement, so the change is not worth making. The measurements cover vaddr only.
