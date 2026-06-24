# lf-ms-evaluation

Evaluation harness for the user-space Master Scheduler (MS) in Lingua Franca.
The evaluated runtime is pinned as the `reactor-c` submodule.

## Setup
    git clone --recursive https://github.com/ertlnagoya/lf-ms-evaluation.git
    # or after a plain clone: git submodule update --init --recursive

## Run
    ./run_e3.sh --steps 5 --load-factors 1.0       # build + MS-patch smoke test
    ms-eval/scripts/run_baremetal.sh backlog 2-3   # E5 overload onset
    ms-eval/scripts/run_baremetal.sh final         # E4/E5/E6 main experiment
