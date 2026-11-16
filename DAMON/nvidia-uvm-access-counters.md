# NVIDIA UVM access counters as a DAMON access source

Kernel: Arch `7.3.4-arch1-2`, x86-64, 4 KiB pages. GPU: RTX 4060 Laptop (AD107, PCIe), `nvidia-open 620.70.06`; driver source read from 595.112.07. Not posted upstream.

## Question

GPU accesses to host memory are invisible to DAMON (see [hmm-gpu-access-invisible.md](hmm-gpu-access-invisible.md)). Can UVM access counters provide that signal instead?

## Findings

**Enabling.** With `uvm_perf_access_counter_migration_enable=1`, counters are on and notifications arrive, but on 4 KiB-page kernels UVM never migrates (`uvm_gpu_access_counters.c`, "Bug 4299018"). It only broadcasts notifications to the UVM tools event queue and does not clear the counter. Migrated pages stayed at 0 and bandwidth stayed at PCIe remote level. Receiving the events from a separate process needed `uvm_enable_builtin_tests=1`.

**Semantics** (managed memory + `SET_ACCESSED_BY`, 2 MiB granularity, threshold 256):
- A notification carries the threshold value (256, rarely 257–259), not a frequency. Addresses are 64 KiB aligned, with a 64 KiB sub-block mask within the 2 MiB region.
- Without clearing, a working set of up to about 192 regions notifies once and then goes silent. Above about 256 regions it keeps notifying, probably because hardware counter table entries are evicted and recounted; not confirmed.
- With `UVM_TEST_RESET_ACCESS_COUNTERS` before each launch, every region notifies once per launch. That gives a binary "accessed at least threshold times this period" sample, which matches DAMON's mkold/young model. The reset ioctl takes 6–13 µs (median).
- Local VRAM accesses produce no notifications. Pinned and HMM memory keep notifying without reset, and behave less regularly after reset; not explained.

**Cost.** Each notification costs about 4.5 µs of UVM bottom-half CPU, whether or not anyone consumes it. Streaming 1 GiB of pinned memory used about 36% of one core. Granularity × reset period matrix (1 GiB streamed, 2 s, 3 rounds):

| Memory | Granularity | Notifications/s | BH % of a core |
| --- | --- | ---: | ---: |
| managed | 64K | ~189k | ~85 |
| managed | 2M | ~12k | 4.5–6.8 |
| managed | 16M | 3.8k–8k | 1.3–3.7 |
| pinned | 2M | 82–88k | 37–41 |
| pinned | 16M | 53–74k | 25–35 |

Notifications follow remote traffic volume, not granularity: above 2M they do not drop in proportion. The reset period barely matters except at 16G granularity (one counter for everything), which is cheap but gives no spatial resolution. GPU bandwidth showed no measurable difference in any combination (12.36–12.55 GB/s).

## Conclusion

The signal exists and can be shaped into DAMON-style samples with periodic resets. The CPU cost scales with remote traffic and is too high at useful granularities. The threshold (up to 65535) is the only untested knob that might reduce the notification rate proportionally. Not tested: the production configuration without builtin tests, top-half interrupt cost, impact on CPU-side workloads, and real applications.
