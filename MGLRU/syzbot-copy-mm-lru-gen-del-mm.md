# syzbot KCSAN: copy_mm / lru_gen_del_mm

Report: [syzbot c6e1b1b32ff2fc3b7d60](https://syzkaller.appspot.com/bug?extid=c6e1b1b32ff2fc3b7d60). Source read at mainline `067c5c24cef2` (2026-11-10) and the first crash commit `17e7b8eacf4c`. Static triage only; not posted upstream.

## Report

KCSAN sees an 8-byte `__list_del()` write inside the `mm_struct` that `dup_mm()` is copying with `memcpy()`. The dashboard's AI analysis claims `__mmput()` is tearing down the same mm while it is being copied, and labels the race harmful.

## Analysis

- `__list_del(prev, next)` writes `next->prev` and `prev->next`, that is, the **neighbouring** nodes. When another mm leaves the MGLRU mm list (`lru_gen_del_mm()` under the list lock), the write can land in the `lru_gen.list` of the parent mm being copied. The parent is not being freed.
- The same-mm teardown claim does not fit the lifetime rules: `copy_mm()` copies `current->mm`, which the forking task holds, and `__mmput()` needs `mm_users` to drop to zero.
- The child never uses the copied pointers: `dup_mm()` calls `mm_init()` right away, and `lru_gen_init_mm()` resets `list`, `bitmap` and `memcg`.
- The dashboard's AI `repro-c` job is marked reproduced, but the KCSAN pair it hits is `copy_mm / percpu_counter_destroy_many`, not this one.

This agrees with the earlier public discussion ([September](https://lists.openwall.net/linux-kernel/2026/09/21/381), [October](https://lists.openwall.net/linux-kernel/2026/10/28/2507)), where the race was considered harmless and an annotation was suggested. A July variant is labelled `race:benign`.

## Conclusion

A real data race in KCSAN's sense, but benign for this field: no use-after-free, corruption or cross-process effect. Not reproduced locally. This does not cover every field of a copied `mm_struct`.
