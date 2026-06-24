# Bare-metal evaluation on Raspberry Pi 4/5

Goal: reproduce the key result (HC protection + bounded backlog under overload)
on real hardware at a natural timescale, removing the "virtualized host / period
scaled to escape jitter" threat to validity. The harness runs natively (no
Docker; Docker was only packaging). RPi 4 and 5 are quad-core aarch64.

RPi 5 is preferred (faster cores, less throttling). Either works.

## 1. OS and toolchain
- 64-bit OS: Raspberry Pi OS (64-bit) or Ubuntu Server 24.04 (arm64).
- Packages: `sudo apt install -y git cmake build-essential default-jdk python3 python3-matplotlib util-linux linux-cpupower`
- Lingua Franca compiler `lfc` (needs Java). Install the LF release and put its
  `bin/` on `PATH`, or pass `LFC_BIN=/path/to/lfc` to the runner. Verify:
  `lfc --version`.
- Clone the evaluation repo with its runtime submodule:
  ```sh
  git clone --recursive https://github.com/ertlnagoya/lf-ms-evaluation.git
  cd lf-ms-evaluation
  # if you cloned without --recursive:
  git submodule update --init --recursive
  ```
  Paths are derived from the script locations; no environment variables are
  required (override with `REACTOR_C` / `TCRS_DATA` if needed).

## 2. Timing hygiene (the point of bare metal)
- **Cooling + PSU**: use active cooling and the official PSU. Throttling injects
  jitter. Check after a run: `vcgencmd get_throttled` (want `0x0`).
- **Performance governor**: `sudo cpupower frequency-set -g performance` (the
  runner attempts this).
- **Isolate cores** (recommended) so the LF run is undisturbed. Edit the kernel
  cmdline (`/boot/firmware/cmdline.txt`) and append on the single line:
  `isolcpus=2,3 nohz_full=2,3 rcu_nocbs=2,3`
  then reboot. Run pinned to the isolated cores (`cpuset 2-3`).
- Optional (stronger): a PREEMPT_RT kernel. Not required for a first result;
  isolation + performance governor already removes most jitter.

## 3. Run
From the repo root (`lf-ms-evaluation/`). The runner signature is
`run_baremetal.sh <mode> [cpuset]`; it sets the governor (best effort), exports
the data dir, and pins the run with `taskset`.

```sh
# quick build/patch smoke test (no experiment):
./run_e3.sh --steps 5 --load-factors 1.0

# plan only (no execution):
ms-eval/scripts/run_baremetal.sh dry 0-3

# decisive confirmation (w2 overload onset, scaled-period sweep):
ms-eval/scripts/run_baremetal.sh backlog 0-3
# if cores 2,3 are isolated, pin there instead:
ms-eval/scripts/run_baremetal.sh backlog 2-3
```

`backlog` writes `ms-eval/tcrs-data/e5/tcrs_bk_backlog_*`. `validate` / `final` /
`periods` work the same way.

## 4. Natural-timescale check (the bare-metal payoff)
On the Pi, isolated cores should have << 1 ms jitter, so you can also run at a
**natural 1 ms period** (no 10 ms scaling) to show the result is not an artifact
of the scaled operating point. Use the `periods` mode, which sweeps periods and
scales work proportionally:

```sh
taskset -c 2-3 python3 ms-eval/scripts/run_e4_e6_main.py --mode periods \
  --pr-periods 1000,2000,5000 --pr-loads 4.0,5.0 --pr-repeats 12 --pr-steps 150
```

First sanity-check that the system actually overloads and that jitter is small:
inspect `wall/tag` vs `period` in a `baseline` log under
`ms-eval/logs/e3/<timestamp>/`. If at 1 ms the per-tag wall time tracks the
period at low load and exceeds it under overload (as designed), the
natural-timescale result is valid; if jitter still dominates at 1 ms, fall back
to 2–5 ms.

## 5. Refresh figures/tables
After a clean run on the Pi, regenerate the figures:

```sh
python3 ms-eval/scripts/plot_paper_figs.py   # reads ms-eval/tcrs-data/{e4,e5}
# figures are written under ms-eval/tcrs-data/figures/
```

Report the Pi result as a confirmation (a sentence + the w2 onset numbers), and
change the manuscript's Threats text from "virtualized host / future bare-metal"
to "confirmed on Raspberry Pi N (isolated cores, performance governor)".

## Caveats / what may need iteration on the actual Pi
- `lfc` install and `cmake`/Java versions can need adjustment; the first build is
  where most issues appear. Build once via `./run_e3.sh --steps 5 ...` and confirm
  the `e3_overload` binary is produced and the MS patch is applied.
- On very short runs the harness may print `failed to derive rid->runtime index
  mapping; falling back to 0..N-1`; with realistic step counts (150+) the mapping
  should derive. Confirm this warning is absent in the real runs.
- The user-space MS has no OS preemption; on bare metal without a competing load
  the three modes should separate cleanly, but new effects (e.g., scheduler
  behavior) may appear and should be reported honestly.
- Expect 1–2 dial-in iterations (toolchain, period vs jitter, core isolation)
  before the numbers are publication-clean.
