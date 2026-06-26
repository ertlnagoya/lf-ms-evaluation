# Raw experimental data (TCRS 2026)

Result CSVs and figures behind the paper's tables/figures, from the Raspberry
Pi 5 (PREEMPT_RT) runs: reactor-c tag ms-eval-v1.0, lfc 0.11.0, P=2 ms,
workers {1,2}, n=15. Per-condition raw rows are in the *_long.csv files.

- e5/  worker sweep (tcrs_bk_backlog_w{1,2}): per-worker summary & long CSVs and
       the combined backlog CSV — behind Table I, fig_hc_miss, fig_backlog.
- e6/  budget sensitivity (tcrs_e6_policy_*) — behind the budget paragraph.
- figures/  fig_hc_miss.pdf, fig_backlog.pdf.
- validation/  period-independence (rpi_{1ms,2ms,10ms}) and the functional /
       core-allocation checks (rpi_check, rpi_4core, rpi_sweep_w2, ...).

Capacity model E4 is analytic from the e5 sweep (no separate dir). Bulky per-run
JSON/ms logs (~0.5 GB, ms-eval/logs/) are not committed; regenerate with
ms-eval/scripts/run_baremetal.sh.
