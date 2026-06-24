# lf-ms-evaluation

Evaluation harness for a **user-space Master Scheduler (MS)** on the Lingua
Franca (LF) C runtime. The MS adds staged control — observation, intervention,
controlled degradation, and OS-level coordination — at the LF runtime
**ready-set boundary**, so it **preserves LF logical-time (deterministic
reactive) semantics** and is deployable entirely in user space, with no kernel,
RTOS, or hypervisor modifications.

This repository is the evaluation artifact base for the accompanying paper. The
evaluated runtime is pinned as the [`reactor-c`](https://github.com/ertlnagoya/reactor-c)
submodule, so a checkout reproduces exactly which MS implementation was measured.

## Repository layout

```
lf-ms-evaluation/
├── reactor-c/                 # submodule: the MS runtime, pinned at tag ms-eval-v1.0
├── ms-eval/                   # the harness (scripts, config, lf-templates, figures)
├── run_e1.sh run_e2.sh run_e3.sh run_all.sh
└── README.md
```

## Setup

```bash
git clone --recursive https://github.com/ertlnagoya/lf-ms-evaluation.git
# or, after a plain clone:
git submodule update --init --recursive
```

Requirements: `lfc` (Lingua Franca) on `PATH` (or `LFC=/path/to/lfc`), `cmake`,
a C toolchain, Python 3 (+ `matplotlib`), and `util-linux` (`taskset`) for
pinned/bare-metal runs.

## Run

```bash
./run_e3.sh --steps 5 --load-factors 1.0        # quick build + MS-patch smoke test
./run_all.sh                                     # E1/E2/E3 end-to-end
ms-eval/scripts/run_baremetal.sh backlog 2-3     # E5 overload onset (bare metal, cores 2-3)
ms-eval/scripts/run_baremetal.sh final           # E4/E5/E6 main experiment
```

Results and figures are written under `ms-eval/`. `run_e3.sh` reads the runtime
sources from the `reactor-c` submodule (override with `REACTOR_C=/abs/path`).

## Experiments

| ID | Focus | Driver / script |
|----|-------|-----------------|
| E1 | MS runtime overhead (microbenchmark) | `run_e1.sh` |
| E2 | HC deadline miss-rate: LF baseline vs MS / MS+RT | `run_e2.sh` |
| E3 | Controlled degradation under overload (baseline / ms / degrade) | `run_e3.sh` |
| E4 | Overload validation: capacity model + period robustness | `ms-eval/scripts/run_e4_e6_main.py` |
| E5 | Backlog vs worker/load sweep (overload onset) | `ms-eval/scripts/run_e5_backlog_worker_sweep.py` |
| E6 | Policy sensitivity: LC budget, degrade lag, ready-queue length | `ms-eval/scripts/run_e6_policy_sensitivity.py` |

## Evaluated runtime (reactor-c) and publications

The MS runtime under evaluation is the `reactor-c` submodule, pinned by tag. The
table maps each publication to the runtime commit it used and the experiments it
reported.

| Date | Venue / document | Title | reactor-c tag → commit (date) | Experiments |
|------|------------------|-------|-------------------------------|-------------|
| 2026-03 | Technical Paper (`TechnicalPaper-MS-v1.1.pdf`) | A Semantics-Preserving Master Scheduler for Lingua Franca | `ms-v1.0` / `v1.0` → `6a5ba2fc` (2026-03-12) | E1, E2, E3 |
| 2026-04 | JSAE (自動車技術会) — _event/authors TBD_ | _title TBD_ | `ms-v1.0` → `6a5ba2fc` (2026-03-12) | E1, E2, E3 |
| 2026-06 | TCRS 2026 | Semantics-Preserving Controlled Degradation for Overload Control: Bounding Logical-Time Backlog in Lingua Franca | `ms-eval-v1.0` → `997a8df4` (2026-06-24) | E3, E4, E5, E6 |

To reproduce a publication's runtime exactly:
`git -C reactor-c checkout <tag>` (e.g. `ms-eval-v1.0` for TCRS 2026), then build.

## Documentation

- [`ms-eval/README.md`](ms-eval/README.md) — evaluation suite details: quick
  start, evaluation programs, requirements, output files.
- [`ms-eval/LOG_FORMAT.md`](ms-eval/LOG_FORMAT.md) — structured log format
  (application JSONL and MS log) used by the parsers.
- [`README_ms_performance_test.md`](README_ms_performance_test.md) — reproducible
  test guide (E1/E2/E3) and tagged-artifact reproduction steps.
- [`ms-eval/RUN_BAREMETAL_RPI.md`](ms-eval/RUN_BAREMETAL_RPI.md) — bare-metal
  evaluation on Raspberry Pi 4/5 (timing hygiene, core isolation, natural 1 ms).
- [`reactor-c/README_ms.md`](reactor-c/README_ms.md) — MS implementation summary
  (Phases 0–4), in the runtime submodule.

## Reproducibility / tags

- `reactor-c` submodule is pinned at tag
  [`ms-eval-v1.0`](https://github.com/ertlnagoya/reactor-c/tree/ms-eval-v1.0)
  (the evaluated runtime).
- MS implementation tag:
  [`ms-v1.0`](https://github.com/ertlnagoya/reactor-c/tree/ms-v1.0).
- To bump the runtime later: `git -C reactor-c fetch && git -C reactor-c checkout <ref>`,
  then commit the updated submodule gitlink here.
