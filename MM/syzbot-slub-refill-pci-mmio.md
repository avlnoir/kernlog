# syzbot: SLUB crash in __refill_objects_node after raw PCI BAR access

Report: [syzbot 73e34786d639a8eb1324](https://syzkaller.appspot.com/bug?extid=73e34786d639a8eb1324); source read at arm64 `for-kernelci` `58c1eb84e977` (2026-11-09 crash). Static review only; not posted upstream.

## Report

Both crashes fault on a NULL-derived pointer in `get_freepointer()` during `__refill_objects_node()`. The older one hit while allocating in `l2cap_chan_create`, the newer one in `dup_mm`. The dashboard AI labels it exploitable and unprivileged and blames Bluetooth.

## Analysis

- The newer syz reproducer contains no Bluetooth operation. It opens the PCI device's sysfs `resource0` read/write, maps it, and passes an address inside that mapping as a userspace read buffer, with `sandbox: none`.
- In both consoles, NVMe completion errors (generation counter mismatches, invalid IDs) come right before the SLUB fault. The minimization log shows crashes in many unrelated functions.
- In SLUB, `get_partial_node_bulk()` uses a racy free count only to choose slabs; `get_freelist_nofreeze()` recomputes `count` from the slab, and `__refill_objects_node()` walks that many objects. A truncated or corrupted chain explains the crash but not its cause.
- `resourceN` is mode 0600 and checked against `LOCKDOWN_PCI_ACCESS`, and the crashing task was UID 0. Root writing to the active NVMe controller's BAR is outside the threat model.

## Conclusion

Most likely memory corruption from privileged raw access to the VM's own NVMe controller, not an allocator bug or an unprivileged path. This agrees with Andrew Morton's reply on the report and with syzkaller [PR #8111](https://github.com/google/syzkaller/pull/8111), merged by 2026-11-13, which filters such PCI endpoints. An independent SLUB defect is not ruled out: that would need repeated runs on the same commit with the raw PCI access removed. Nothing was run.
