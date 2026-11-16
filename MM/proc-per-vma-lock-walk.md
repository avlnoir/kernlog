# /proc address-space walks under per-VMA locks: no issue found

Kernel: mainline `cbbddb6b08cc` (2026-10-31). `fs/proc/task_mmu.c`, `mm/mmap_lock.c`, `include/linux/mmap_lock.h` and `mm/pagewalk.c` were identical in mm-new, mm-unstable and mm-hotfixes-unstable at the time. Negative result; not posted upstream.

## Question

smaps, numa_maps, maps and `PROCMAP_QUERY` now walk another process's address space holding only per-VMA read locks. Can a VMA that is freed, reused, split or merged during the walk lead to a wrong VMA, a use-after-free or sleeping under RCU?

## Static review

| Hypothesis | Why it does not hold |
| --- | --- |
| `lock_next_vma()`/`vma_start_read()` return a wrong VMA after free/reuse, split or merge | After taking a reference they check `vm_mm` and the sequence number. A range mismatch or a possible gap falls back to speculation under the mmap lock or a retry. |
| The sequence-wrap fix `7135a05838e8` misses a VMA re-inserted with a stale sequence | New VMAs, `vm_area_dup()`, split, downgrade and munmap rollback all reset it or finish within the same write-locked section. |
| `reacquire_rcu()` dereferences NULL at the gate VMA | smaps returns before `drop_rcu()`; numa_maps returns early on `vm_mm == NULL`. |
| `anon_vma_name`, `vm_policy` or `vm_file` freed under the VMA read lock | Replacing the name or policy takes the VMA write lock first; the file is pinned by the VMA. |
| `PROCMAP_QUERY` copies to user while locked, or out of bounds | The name is formatted into a kernel buffer and copied after unlock and `mmput()`; the length checks hold. |
| Sleeping in the maps `show` RCU section | The `->name()` callbacks and `d_dname()` of mmap-able pseudo files do not sleep; `__get_vma_policy()` only takes a spinlock. |
| Touching the address space with only `mm_count` (the shape fixed by `4fb975471717`) | Other users in task_mmu.c use `mmget_not_zero()`. munmap detaches VMAs before downgrading, and `exit_mmap()` only runs after `mm_users` reaches 0. |
| The new lockless `vma_set_flags()`/`vma_clear_flags()` lose an update against `VM_MAYBE_GUARD` set under the read lock | All 18 callers act on a VMA not yet in the tree or take `vma_start_write()` first. `remap_pfn_range()` lost an implicit write-lock assertion; that is only a weaker debug check. |

## Stress test

QEMU/KVM, 4 vCPUs, 2 NUMA nodes, KASAN, lockdep/PROVE_RCU, DEBUG_ATOMIC_SLEEP, DEBUG_VM, PAGE_TABLE_CHECK. For 120 s, 3 mutator threads (mmap of anon/shared/memfd/hugetlb, munmap, mprotect, mremap including DONTUNMAP, madvise including guard and COLLAPSE, anon VMA naming, mbind, mlock) ran against 3 readers (smaps, smaps_rollup, numa_maps, maps, `PROCMAP_QUERY`, mincore, pagemap, `PAGEMAP_SCAN`), plus a forking thread whose children were read through /proc while they churned VMAs.

Test-only assertions were added: lockless flag helpers on in-tree VMAs must hold the write lock, `reacquire_rcu()` must see a non-NULL `locked_vma`, and new mms start `mm_lock_seq` at `0xFFFFF000` to force wraps. The run did about 124k mutations, 48–52k reads of each kind, 1689 forks and 1653 sequence wraps: no KASAN, lockdep, RCU or BUG report, and the assertions never fired.

## Limits

One 120 s run with one seed, x86-64 only. A negative result, not proof of absence.
