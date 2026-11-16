# test_hmm copies a device-private page after re-faulting the mirror

Kernel: mainline `d89e1d186ec9`. The structure dates back to `b2ef9f5a5cb3` ("mm/hmm/test: add selftest driver for HMM", 2020). Not posted upstream.

## Problem

`lib/test_hmm.c` uses backing pages to emulate device memory. When a migration completes, the mirror stores `BACKING_PAGE(dpage)`. If the mirror is invalidated by something other than its own migration, `HMM_DMIRROR_READ`/`WRITE` re-fault through `dmirror_fault()` → `hmm_range_fault()`. Since `dev_private_owner` is this device, HMM returns the owner-matching device-private page itself. `dmirror_do_fault()` stores it without going through `BACKING_PAGE()`. `dmirror_do_read()`/`dmirror_do_write()` then `memcpy` to or from the device-private `struct page`, whose direct map is not accessible, and the kernel oopses.

The precondition is that the page is still on the test device but the mirror was cleared. mprotect, THP split and NUMA hinting all send notifier events other than `MMU_NOTIFY_MIGRATE`.

## Result

QEMU, `oops=panic`:

| Mode | Steps | Result |
| --- | --- | --- |
| control | migrate to device, read via mirror | reads correct data |
| read | migrate to device, `mprotect(RWX)`, read via mirror | oops in `memcpy` in `dmirror_do_read` |
| write-split | migrate THP to device, `MADV_DONTNEED` subpage 1 to split, write subpage 0 via mirror | oops in the write `memcpy` |

The read path triggers with read-only or writable entries. The write path needs a still-writable entry, such as the remaining subpages after a device-private THP split.

## Impact and fix

Only the test module (`CONFIG_TEST_HMM`, root-only device node) is affected, so this is not a security issue. Existing `hmm-tests` never access a page that is still on the device after its mirror was cleared, which is why it went unnoticed. A likely fix is to apply `BACKING_PAGE()` to device-private pages in `dmirror_do_fault()`; not written or tested.
