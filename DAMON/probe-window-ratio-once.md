# Computing the DAMON probe window ratio once per call

Base: mm-new `d865f2677c44` (2026-10-27). Not posted upstream.

## Idea

The probe weighted-sum helper computes the moving-window ratio from the context intervals inside the per-probe loop, which costs three variable divisions per probe. The change computes the ratio once per call when the probe list is non-empty and the call is neither `last` nor `mv` only. Per-probe rounding, `unsigned char` truncation, sum overflow and `last` precedence are unchanged, and no field is added to the context or region.

## Result

- Equivalence: the real source, extracted and built natively for 64-bit and 32-bit, was compared against the original over 3,945,732 inputs each, including exhaustive u8 values, interval wrap and non-divisible intervals. 50 DAMON KUnit cases passed on the real kernel. Of the 15 DAMON selftests, 14 passed; `quota` also fails on the unmodified base.
- Code: the three divisions leave the loop. The function grew from 293 to about 404 bytes, and the explicit stack reservation shrank from 64 to 16 bytes.
- Timing: source extracted into separate translation units, GCC 16 -O2, no LTO, pinned CPU, 7 interleaved A/A and ABBA rounds.
  - Moving-window calls: −47.6 to −49.6% with 2 probes, −66.5 to −73.9% with 4.
  - Empty list (after an early-return fix): −15 to −17%.
  - 1 probe: no general gain.
  - 2 probes, `last` and `last && mv` paths: about +0.17–0.19 ns per call (+12.8% and +11.7% paired), above the A/A noise.

## Status

A real local gain with a measured cost on the control path. These are per-call microbenchmarks with a simplified context layout, not whole-kdamond results, so the change was kept as a candidate to evaluate on real probe configurations. It was not proposed.
