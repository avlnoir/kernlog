# DAMOS max_nr_snapshots only covers the first target

Source: mm-new `d865f2677c44` (2026-10-27). Not posted upstream.

## Observation

One `fvaddr` context with two process targets (1 MiB and 2 MiB of mapped memory) and a STAT scheme matching every region, no quota or filters:

- `max_nr_snapshots=1`: only the first target's bytes appear in `sz_tried`. Swapping the target order changes which size is counted.
- `max_nr_snapshots=2`: all 3 MiB are counted.

Each configuration was run three times in isolated QEMU guests with the same result.

## Open question

This depends on whether a snapshot is defined per target or per context. In an [earlier discussion](https://lore.kernel.org/damon/20260807135300.90965-1-sj@kernel.org/) the maintainer stressed completing full attempts, so the behaviour is not clearly a bug. It needs clarification before any change.
