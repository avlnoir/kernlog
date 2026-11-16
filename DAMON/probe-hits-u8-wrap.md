# DAMON zero-weight probe hit counter wraps at 256

Kernel: mm-new `243e8cdab641` (2026-10-30). Present since v7.2-rc1. Not posted upstream.

## Problem

`probe_hits[]` in `struct damon_region` is `unsigned char`, and the probes run on every sampling interval. `damon_valid_probe_params()` limits samples per aggregation interval to `U8_MAX` only when some probe has a non-zero weight (`93508425db11`). A zero-weight probe without filters hits on every sample, so with more than 255 samples per aggregation the counter wraps.

## Result

Verified on an unmodified kernel through sysfs, 4/4 runs consistent: with a 1 ms sampling interval and a filter-free zero-weight paddr probe, 200/255/256/300 samples per aggregation report 200/255/**0**/**44** hits. The same setup with weight 1 and 256 samples is rejected with `EINVAL`. The `damon_region_aggregated` tracepoint reports the same counter.

## Open question

It is not clear whether this is a bug. The KUnit test `damon_test_valid_probe_params()` explicitly asserts that validation is skipped without weights, and the commit adding the check only talks about attribute-only mode. The documentation describes the counter as the number of hit samples without mentioning a limit. Whether zero-weight probes are meant to be bounded needs the maintainer's view; no fix was written.
