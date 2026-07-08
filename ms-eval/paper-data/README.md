# Raw experimental data (TCRS 2026)

Result CSVs and figures behind the paper's tables/figures, from the Raspberry
Pi 5 (PREEMPT_RT) runs: reactor-c tag ms-eval-v1.0, lfc 0.11.0, P=2 ms,
workers {1,2}, n=15. Per-condition raw rows are in the *_long.csv files.

日本語版は [README.ja.md](README.ja.md) にあります。

- e5/  worker sweep (tcrs_bk_backlog_w{1,2}): per-worker summary & long CSVs and
       the combined backlog CSV — behind Table I, fig_hc_miss, fig_backlog.
- e6/  budget sensitivity (tcrs_e6_policy_*) — behind the budget paragraph.
- figures/  fig_hc_miss.pdf, fig_backlog.pdf.
- validation/  period-independence (rpi_{1ms,2ms,10ms}) and the functional /
       core-allocation checks (rpi_check, rpi_4core, rpi_sweep_w2, ...).

Capacity model E4 is analytic from the e5 sweep (no separate dir). Bulky per-run
JSON/ms logs (~0.5 GB, ms-eval/logs/) are not committed; regenerate with
ms-eval/scripts/run_baremetal.sh.

## Column notes (read before interpreting the CSVs)

- `*_hc_miss_mean` — HC-only deadline-miss rate (%), over HC reactions
  (reaction_id < hc); mean over the n=15 repeats, with `*_ci_low/high`.
- `*_hc_latency_p95_us` — HC tail latency = (physical completion − logical
  release), nearest-rank 95th percentile, in microseconds.
- `*_lc_completion_mean` — fraction (%) of released LC reactions that executed
  (degrade with budget b=2 of 4 ⇒ ≈50%).
- `fallbacks_mean`, `fallback_missing_metadata_mean`, `fallback_no_candidate_mean`
  — safe fallbacks to default scheduling (e.g. missing metadata); **0 in these
  runs**.
- **`violations_mean` (and `degrade_violations`) — despite the name, this counts
  `event=mismatch`: MS/runtime *pick mismatches*, i.e. the runtime ran a
  different ready reaction than the MS's advisory pick (a different valid
  intra-tag order). These are benign — LF determinism holds by construction —
  and are NOT semantic or deadline violations.** (≈0.3–0.7/run here.) The column
  keeps its legacy name for backward compatibility; the paper refers to it as
  "MS/runtime pick mismatches".
- Confidence intervals are normal-approximation 95% (mean ± 1.96·sd/√n).
