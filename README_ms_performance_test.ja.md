# MS 評価 — 再現ガイド

このハーネスは 2 世代の成果にまたがります。再現したい結果に対応する節を使って
ください。最新版（TCRS 2026）の手順を先に示し、初期の技術論文／JSAE 版（E1/E2/E3）
の手順はそれらの論文を再現するために後半に残してあります。

英語版は [README_ms_performance_test.md](README_ms_performance_test.md) にあります。

## 最新 — TCRS 2026（物理実機でのネイティブ実行）

論文: *Controlled Degradation, Not Dispatch Reordering: Semantics-Preserving
Overload Control in Lingua Franca*（TCRS 2026 / IEEE Embedded Systems Letters）。

- **ランタイム**: `reactor-c` submodule のタグ `ms-eval-v1.0`、`lfc` 0.11.0。
- **プラットフォーム**: Raspberry Pi 5（クアッドコア Arm Cortex-A76）、64-bit
  Raspberry Pi OS に `PREEMPT_RT` カーネルと performance ガバナ。ネイティブ実行
  （非仮想化）。最悪スケジューリング遅延の実測値は ≤ 3 µs（cyclictest）。
- **動作点**: 周期 P = 2 ms（1 ms と 10 ms でも確認）、ワーカ ∈ {1, 2}、
  タグあたり LC バジェット b = 2、過負荷閾値 θ_ℓ = 300 µs / θ_q = 2、
  各条件 n = 15 回。
- **実験**: E4（過負荷の検証＋容量モデル）、E5（worker スイープ＋backlog 伝播 —
  Table I と両図）、E6（バジェット感度）。

OS 導入、PREEMPT_RT カーネルのビルド／選択、コア隔離、実行、自然な周期（1〜2 ms）
チェックまでの全手順は
[`ms-eval/RUN_NATIVE_RT_RPI.md`](ms-eval/RUN_NATIVE_RT_RPI.md)
（日本語: [`ms-eval/RUN_NATIVE_RT_RPI.ja.md`](ms-eval/RUN_NATIVE_RT_RPI.ja.md)）
にあります。

クイックスタート:

```bash
git clone --recursive https://github.com/ertlnagoya/lf-ms-evaluation.git
cd lf-ms-evaluation
./run_e3.sh --steps 5 --load-factors 1.0           # ビルド＋MSパッチの簡易確認
ms-eval/scripts/run_baremetal.sh final 0-3         # E4/E5/E6 本実行（コア 0-3）
python3 ms-eval/scripts/plot_paper_figs.py         # fig_hc_miss / fig_backlog を再生成
```

論文の表・図の元になった生の結果 CSV と図は
[`ms-eval/paper-data/`](ms-eval/paper-data/)（`e5/`, `e6/`, `figures/`,
`validation/`）にアーカイブされ、目録は `ms-eval/paper-data/README.md` にあります。

## 初期版 — 技術論文 / JSAE（E1/E2/E3、reactor-c モノレポ＋Docker）

以下のガイドは、初期の技術論文／JSAE 版の **E1/E2/E3** 実験を再現します。`Dockerfile`
を含む元の **reactor-c モノレポ**のタグ `ms-v1.0` を前提とします。この節の `git` /
`docker` コマンドはその（この評価リポジトリではなく）リポジトリを対象とします。
ここで触れる結果 CSV／図は、その Docker（x86-64）環境のもので、上記の Raspberry Pi
実行のものではありません。

---

### MS 評価テストガイド（E1 / E2 / E3）— reactor-c モノレポ（Docker）

この文書は、MS 性能評価の再現手順を示します:

- E1: MS 介入のランタイムオーバーヘッド
- E2: LF baseline と 2 つの MS+RT 構成の HC ミス率比較
- E3: 過負荷下の制御的デグラデーション

## 0. タグ付きアーティファクトの再現

論文参照に使った正確なコード／データ状態を再現するには、固定タグをチェックアウト
します:

```bash
git checkout v1.0
```

参照タグ:

- MS 実装: `ms-v1.0`
- 評価コードとアーティファクト: `ms-eval-v1.0`
- 統合スナップショット: `v1.0`

参照 URL:

- https://github.com/ertlnagoya/reactor-c/tree/ms-v1.0
- https://github.com/ertlnagoya/reactor-c/tree/ms-eval-v1.0
- https://github.com/ertlnagoya/reactor-c/tree/v1.0

## 1. 共通ワークフロー（Docker の再ビルドとシェル）

すべての評価コマンドはリポジトリのルート（コンテナ内では `/workspace/reactor-c`）
から実行します。

```bash
docker build --no-cache -t lf-ms-phase4 .
docker run --rm -it \
  -v "$PWD":/workspace/reactor-c \
  -w /workspace/reactor-c \
  lf-ms-phase4 \
  bash
```

コンテナ内で確認:

```bash
pwd
ls
```

## 2. E1: MS オーバーヘッド測定

### 目的

MS 介入を有効（`intervene`）にしたときのステップ時間オーバーヘッドを、LF baseline
（`baseline`）と比較して測定します。

### テスト条件

- マイクロベンチのサイズ: `N=16,32,64,128`
- ステップ数: `1000`
- リアクションの仕事量: `50 us`
- 相対デッドライン: `200 us`
- 周期: `1000 us`
- N ごとのモード:
  - `baseline`: `LF_MS_DISABLE=1`
  - `observe`: `LF_MS_DISABLE=0, LF_MS_OBSERVE_ONLY=1`
  - `intervene`: `LF_MS_DISABLE=0, LF_MS_OBSERVE_ONLY=0`

### 実行

```bash
./run_e1.sh
```

期待される末尾出力:

```text
E1 logs written under /workspace/reactor-c/ms-eval/logs/e1/<timestamp>
```

### 整形とプロット

`<timestamp>` を上で表示された値に置き換えてください。

```bash
python3 ms-eval/scripts/parse_e1.py \
  --logs ms-eval/logs/e1/<timestamp> \
  --out ms-eval/results/e1_overhead_<timestamp>.csv

python3 ms-eval/scripts/plot_fig2.py \
  --in ms-eval/results/e1_overhead_<timestamp>.csv \
  --out ms-eval/figures/fig2_overhead_<timestamp>.png
```

簡易確認:

```bash
column -s, -t < ms-eval/results/e1_overhead_<timestamp>.csv
```

解釈:

- `overhead_pct` は同じ `(N, workload_us)` の baseline 行に対して計算される。
- 論文本文での主な比較は `intervene` vs `baseline`。

## 3. E2: HC ミス率比較（baseline vs RT single-worker vs RT worker-group）

### 目的

同じストレス条件下で HC ミス率を比較します:

1. LF baseline（MS 無効パス。両実行からプール）
2. MS + RT（single-worker）
3. MS + RT（worker-group、提案手法）

### 再現に不可欠な設定

E2 スクリプトは次の制御ノブを設定・使用します:

- `LF_MS_OS_ENABLE`
- `LF_MS_OS_RT_ENABLE`
- `LF_MS_OS_RT_GROUP_ENABLE`
- `LF_MS_OS_RT_PRIO_HC`
- `LF_MS_OS_RT_PRIO_LC`
- `LF_MS_OS_HC_NICE_DELTA`
- `LF_MS_OS_LC_BASE_NICE_DELTA`
- `LF_MS_HC_GUARD_ENABLE`
- `LF_MS_HC_GUARD_LAG_NS`
- `LF_MS_HC_GUARD_READY_Q_LEN`
- `LF_MS_MINIMAL_LOG=1`
- `MS_EVAL_LOG_REACTION_START=0`（ロギングのオーバーヘッド削減）

### E2 の共通ワークロード／ストレスパラメータ

- `repeats=30`
- `load=1.15`
- `steps=1600`
- `workers=2`, `hc-workers=1`
- `hc=1`, `lc=5`
- `hc-work-us=90`, `lc-work-us=230`
- `deadline-us=1800`
- `stress-ng`: `--cpu 1 --cpu-load 80 --timeout 360s --taskset 2`
- CPU セット: コンテナ `0-3`、LF `0-1`
- `drop-initial-tags=20`
- RT/nice 操作のための Docker capability: `--cap-add SYS_NICE`

### 3.1 比較群: RT single-worker

```bash
python3 ms-eval/scripts/run_e1_triplet_compare.py \
  --repeats 30 \
  --load 1.15 \
  --steps 1600 \
  --workers 2 \
  --hc-workers 1 \
  --hc 1 \
  --lc 5 \
  --hc-work-us 90 \
  --lc-work-us 230 \
  --deadline-us 1800 \
  --stress-cpu 1 \
  --stress-load 80 \
  --stress-timeout-s 360 \
  --stress-warmup-s 3 \
  --container-cpuset 0-3 \
  --lf-cpu-set 0-1 \
  --stress-cpu-set 2 \
  --drop-initial-tags 20 \
  --inter-arm-sleep-ms 200 \
  --os-lag-ns 300000 \
  --os-ready-q-len 4 \
  --os-lc-base-nice-delta 0 \
  --os-nice-delta 2 \
  --os-hc-nice-delta 0 \
  --os-rt-enable 1 \
  --os-rt-group-enable 0 \
  --os-rt-prio-hc 10 \
  --os-rt-prio-lc 2 \
  --hc-guard-enable 1 \
  --hc-guard-lag-ns 300000 \
  --hc-guard-ready-q-len 4 \
  --docker-cap-sys-nice 1 \
  --out-prefix e1_rtaxis_fix_rt_guard_r30
```

生成物:

- `ms-eval/results/e1_rtaxis_fix_rt_guard_r30_long.csv`
- `ms-eval/results/e1_rtaxis_fix_rt_guard_r30_summary.csv`

### 3.2 提案群: RT worker-group

```bash
python3 ms-eval/scripts/run_e1_triplet_compare.py \
  --repeats 30 \
  --load 1.15 \
  --steps 1600 \
  --workers 2 \
  --hc-workers 1 \
  --hc 1 \
  --lc 5 \
  --hc-work-us 90 \
  --lc-work-us 230 \
  --deadline-us 1800 \
  --stress-cpu 1 \
  --stress-load 80 \
  --stress-timeout-s 360 \
  --stress-warmup-s 3 \
  --container-cpuset 0-3 \
  --lf-cpu-set 0-1 \
  --stress-cpu-set 2 \
  --drop-initial-tags 20 \
  --inter-arm-sleep-ms 200 \
  --os-lag-ns 300000 \
  --os-ready-q-len 4 \
  --os-lc-base-nice-delta 0 \
  --os-nice-delta 2 \
  --os-hc-nice-delta 0 \
  --os-rt-enable 1 \
  --os-rt-group-enable 1 \
  --os-rt-prio-hc 10 \
  --os-rt-prio-lc 2 \
  --hc-guard-enable 1 \
  --hc-guard-lag-ns 300000 \
  --hc-guard-ready-q-len 4 \
  --docker-cap-sys-nice 1 \
  --out-prefix e1_rtgroup_main_r30
```

生成物:

- `ms-eval/results/e1_rtgroup_main_r30_long.csv`
- `ms-eval/results/e1_rtgroup_main_r30_summary.csv`

### 3.3 E2 の図生成

```bash
python3 ms-eval/scripts/plot_e2_rt_comparison_v2.py \
  --comparison-long ms-eval/results/e1_rtaxis_fix_rt_guard_r30_long.csv \
  --proposed-long ms-eval/results/e1_rtgroup_main_r30_long.csv \
  --out-png ms-eval/figures/fig_e2_rt_comparison_v2.png \
  --out-pdf ms-eval/figures/fig_e2_rt_comparison_v2.pdf \
  --out-csv ms-eval/results/e2_rt_comparison_plotdata_v2.csv
```

図中のラベル:

- `LF baseline (MS disabled)`
- `MS + RT (single-worker)`
- `MS + RT (worker-group, proposed)`

論文用の図ファイル:

- `ms-eval/figures/fig_e2_rt_comparison_v2.png`

標本サイズの注記:

- 図中の baseline は両方の long CSV からプール（`n=60`）
- 各 RT 条件は 1 つの long CSV を使用（`n=30`）

### 3.4 E2 スレッド解析拡張

E2 ワークロードのスレッドごとの CPU 使用率、コンテキストスイッチ、リアクション
遅延分布を収集するには:

```bash
python3 ms-eval/scripts/run_e2_thread_analysis.py \
  --repeats 30 \
  --load 1.15 \
  --steps 1600 \
  --workers 2 \
  --hc-workers 1 \
  --hc 1 \
  --lc 5 \
  --hc-work-us 90 \
  --lc-work-us 230 \
  --deadline-us 1800 \
  --stress-cpu 1 \
  --stress-load 80 \
  --stress-timeout-s 360 \
  --stress-warmup-s 3 \
  --container-cpuset 0-3 \
  --lf-cpu-set 0-1 \
  --stress-cpu-set 2 \
  --drop-initial-tags 20 \
  --inter-arm-sleep-ms 200 \
  --os-lag-ns 300000 \
  --os-ready-q-len 4 \
  --os-lc-base-nice-delta 0 \
  --os-nice-delta 2 \
  --os-hc-nice-delta 0 \
  --os-rt-enable 1 \
  --os-rt-prio-hc 10 \
  --os-rt-prio-lc 2 \
  --hc-guard-enable 1 \
  --hc-guard-lag-ns 300000 \
  --hc-guard-ready-q-len 4 \
  --docker-cap-sys-nice 1 \
  --out-prefix e2_thread_analysis_r30

python3 ms-eval/scripts/parse_e2_thread_analysis.py \
  --manifest ms-eval/results/e2_thread_analysis_r30_manifest.csv \
  --out-thread ms-eval/results/e2_thread_utilization.csv \
  --out-latency ms-eval/results/e2_reaction_latency_distribution.csv \
  --out-single ms-eval/results/e2_single_worker_thread_comparison.csv

python3 ms-eval/scripts/summarize_e2_thread_analysis.py \
  --out ms-eval/results/e2_thread_analysis_summary.csv
```

生成物:

- `ms-eval/results/e2_thread_analysis_r30_manifest.csv`
- `ms-eval/results/e2_thread_utilization.csv`
- `ms-eval/results/e2_reaction_latency_distribution.csv`
- `ms-eval/results/e2_single_worker_thread_comparison.csv`
- `ms-eval/results/e2_thread_analysis_summary.csv`

## 4. E3: 過負荷下の制御的デグラデーション

### 4.1 E3 強過負荷ラン

```bash
./run_e3.sh
```

ログの出力先:

- `ms-eval/logs/e3/<timestamp>`

### 4.2 E3 フォローアップ負荷スイープ（`n=30`）

```bash
python3 ms-eval/scripts/run_e3_degradation_compare.py \
  --loads 0.4,0.6,0.8,1.0,1.2,1.6,2.0 \
  --repeats 30 \
  --steps 120 \
  --workers 2 \
  --hc 1 \
  --lc 4 \
  --hc-work-us 150 \
  --lc-work-us 260 \
  --deadline-us 1000 \
  --period-us 1000 \
  --degrade-lag-ns 120000 \
  --degrade-ready-q-len 10 \
  --drop-initial-tags 10 \
  --out-prefix e3_followup_n30

python3 ms-eval/scripts/postprocess_e3_followup_n30.py \
  --in-long ms-eval/results/e3_followup_n30_long.csv \
  --out-raw ms-eval/results/e3_followup_raw_n30.csv \
  --out-summary ms-eval/results/e3_followup_summary_n30.csv \
  --out-tests ms-eval/results/e3_followup_statistical_tests.csv
```

生成物:

- `ms-eval/results/e3_followup_n30_long.csv`
- `ms-eval/results/e3_followup_n30_summary.csv`
- `ms-eval/results/e3_followup_raw_n30.csv`
- `ms-eval/results/e3_followup_summary_n30.csv`
- `ms-eval/results/e3_followup_statistical_tests.csv`

### 4.3 論文図の再生成

```bash
python3 ms-eval/scripts/summarize_e1_three_conditions.py \
  --in-raw ms-eval/results/e1_overhead_n30.csv \
  --out-summary ms-eval/results/e1_three_conditions_summary.csv \
  --out-diff ms-eval/results/e1_observe_vs_intervene.csv

python3 ms-eval/scripts/plot_paper_figures.py \
  --e1-summary ms-eval/results/e1_three_conditions_summary.csv \
  --e2-plotdata ms-eval/results/e2_rt_comparison_plotdata_v2.csv \
  --e3-summary ms-eval/results/e3_followup_summary_n30.csv
```

論文用の出力:

- `ms-eval/figures/fig4_e1_overhead.png`
- `ms-eval/figures/fig4_e1_overhead.pdf`
- `ms-eval/figures/fig5_e2_missrate.png`
- `ms-eval/figures/fig5_e2_missrate.pdf`
- `ms-eval/figures/fig_e3_loadsweep.png`
- `ms-eval/figures/fig_e3_loadsweep.pdf`

## 5. 再現性のための実務メモ

- データ収集中は Docker イメージとホストの負荷条件を固定しておく。
- `ms-eval/logs` は大きくなりやすい。アーカイブには必要なタイムスタンプだけを残す。
- `os_policy_apply` が想定外に 0 の場合は、コンテナの capability（`--cap-add SYS_NICE`）
  と RT フラグを確認する。
- 論文更新の際は必ず次を記録する:
  - コマンドライン
  - 使用した正確な CSV ファイル名
  - 図生成コマンド

原稿更新でよく使う現行の E1 図:

- `ms-eval/figures/fig2_overhead_20260227_013034.png`
