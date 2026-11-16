# syzbot KCSAN: maple tree dead-node parent read/write

Report: [syzbot 38a879f4a73497f2dfef](https://syzkaller.appspot.com/bug?extid=38a879f4a73497f2dfef); latest sample from `fa4ad798f6f6`. Source read at mainline `067c5c24cef2` (2026-11-10), where `lib/maple_tree.c` is unchanged from the sample. Static review only; not posted upstream.

## Report

During `mprotect`, `mte_set_node_dead()` does an 8-byte write that races with an 8-byte read in `ma_dead_node()` from the `lock_vma_under_rcu()` page-fault lookup.

## Analysis

- The writer publishes the replacement node with `rcu_assign_pointer()`, then marks the old node dead by setting its `parent` to itself, then frees it with `kfree_rcu()`.
- The reader holds `rcu_read_lock()` in `mas_walk()`, uses the node, then checks `ma_dead_node()` after `smp_rmb()` and restarts if the node is dead. A successful lookup is still checked by `vma_start_read()` and the address range check, and failure falls back to the mmap lock.
- The protocol is intended, but both accesses are plain C, so the KCSAN report matches the source. Per `memory-barriers.txt` and `access-marking.txt`, plain racy accesses leave tearing and compiler transformations unconstrained. RCU and barriers alone do not prove every compiled form safe.

## Prior work

- The [2026-04 syzbot discussion](https://groups.google.com/g/syzkaller-bugs/c/i3R7Pph_6IA/m/fuStsxXjAAAJ) covered an in-place pivot update, which was considered benign. It does not cover this `parent` pair.
- A [2023 experimental patch](https://lists.infradead.org/pipermail/maple-tree/2023-October/002913.html) already proposed `WRITE_ONCE()`/`READ_ONCE()` for these accesses.
- The [2026-08 pivot-read KCSAN patch](https://lkml.iu.edu/2608.3/04636.html) is about `rcu_head` overlaying a dead node's pivots, not this pair.

## Conclusion

The data race is real. No functional failure, use-after-free or corruption is shown, and nothing was run locally. Marking the two accesses would silence KCSAN and constrain the compiler; whether there is any real miscompilation risk would need an LKMM/compiler analysis.
