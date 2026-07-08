# MS 評価スイート（E1/E2/E3）

このフォルダには、論文
"Requirements-Driven Design of a User-Space Master Scheduler for Deterministic Reactive CPS"
のための再現可能な評価ハーネスが入っています。

LF ベンチマークを生成し、実験 E1/E2/E3 を実行し、ログを CSV に整形し、グラフを
作成します。

英語版は [README.md](README.md) にあります。

## クイックスタート
リポジトリのルートで:

```bash
./run_all.sh
```

出力:
- `ms-eval/results/e1_overhead.csv`
- `ms-eval/results/e2_correctness.csv`
- `ms-eval/results/e2_miss_detection.csv`
- `ms-eval/results/e3_missrate.csv`
- `ms-eval/figures/fig2_overhead.pdf`
- `ms-eval/figures/fig3_missrate.pdf`
- `ms-eval/results/metadata.json`

## 必要環境
- Linux（x86_64）
- `PATH` の通った `lfc`（または `LFC=/path/to/lfc`）
- `python3` ＋ `matplotlib`

## 評価プログラム

### マイクロベンチ（E1/E2）
`ms-eval/scripts/gen_microbench.py` が `ms-eval/lf-gen/` に生成します。

挙動:
- 1 つのタイマが、論理時刻ごとに `N` 個の独立したリアクションを起動する。
- 各リアクションは固定時間のビジーウェイトを行う（`--workload-us`）。
- E2 ではデッドラインミスの注入（選んだリアクションに追加遅延）を任意で行える。
- `reaction_id` は生成器のインデックス（0..N-1）。

主なパラメータ:
- `--N`: ready set のサイズ（リアクション数）
- `--workload-us`: リアクションあたりの基本仕事量
- `--deadline-us`: ミス判定に用いる相対デッドライン
- `--period-us`: タイマ周期（論理時刻の間隔）
- `--iterations`: 実行する論理ステップ数

注入制御（E2）:
- `--inject-period`: k 論理時刻ごとにのみ注入
- `--inject-rate-pct`: 対象ステップごとに注入するリアクションの割合（%）
- `--inject-delay-us`: ミスを強制するための追加遅延

### 過負荷（E3）
`ms-eval/scripts/gen_overload.py` が `ms-eval/lf-gen/` に生成します。

挙動:
- 2 種類のリアクション:
  - 高 criticality（HC）: 仕事量とデッドラインは固定。
  - 低 criticality（LC）: 仕事量を `load_factor` でスケール。
- `load_factor` は実行ごとに設定（E3）。
- デグラデーションは MS ポリシファイル（`ms_policy.cfg`）で設定。

主なパラメータ:
- `--hc`, `--lc`: 高／低 criticality リアクションの数
- `--hc-work-us`, `--lc-work-us`: リアクションあたりの仕事量
- `--hc-deadline-us`, `--lc-deadline-us`: 相対デッドライン
- `--load-factors`: L 値のカンマ区切りリスト

## 実験

### E1: オーバーヘッド（観測 vs 介入）

```bash
./run_e1.sh --N 16,32,64,128 --iterations 1000 --workload-us 50 --deadline-us 200 --period-us 1000 --seed 1
```

モード:
- `baseline`: `LF_MS_DISABLE=1`
- `observe`: `LF_MS_OBSERVE_ONLY=1`（ログのみ、介入なし）
- `intervene`: MS の pick-next を強制

### E2: 意味的正しさ＋ミス検出

```bash
./run_e2.sh --N 64 --iterations 500 --workload-us 50 --deadline-us 200 \
  --inject-period 10 --inject-rate-pct 30 --inject-delay-us 500
```

- 正しさは MS の `ready` ＋ `pick_next` イベントを使い、メンバシップ／順序を確認。
- ミス検出は注入した遅延と `reaction_end.missed_deadline` を突き合わせる。

### E3: 過負荷下の制御的デグラデーション

```bash
./run_e3.sh --hc 8 --lc 8 --hc-work-us 80 --lc-work-us 60 \
  --load-factors 1.0,1.5,2.0,2.5 --lc-budget 4 --window-ns 1000000
```

- Baseline: `LF_MS_DISABLE=1`
- Degrade: 実行ごとに生成される MS ポリシ設定（`ms_policy.cfg`）

## 実行方法

### 一括（全実験）
```bash
./run_all.sh
```

### 実験ごと
```bash
./run_e1.sh --N 16,32,64,128 --iterations 1000 --workload-us 50 --deadline-us 200 --period-us 1000 --seed 1
./run_e2.sh --N 64 --iterations 500 --workload-us 50 --deadline-us 200 \
  --inject-period 10 --inject-rate-pct 30 --inject-delay-us 500
./run_e3.sh --hc 8 --lc 8 --hc-work-us 80 --lc-work-us 60 \
  --load-factors 1.0,1.5,2.0,2.5 --lc-budget 4 --window-ns 1000000
```

### 出力
各実行は `ms-eval/logs/<exp>/<timestamp>/...` にログを書き出し、続いて以下を生成します:
- `ms-eval/results/*.csv`
- `ms-eval/figures/*.pdf`
- `ms-eval/results/metadata.json`

## ロギング
- アプリログ: JSONL（`LF_APP_LOG`）
- MS ログ: key-value 形式（`LF_MS_LOG`）

詳細は `ms-eval/LOG_FORMAT.md` を参照してください。

## メモ / TODO
- `LF_MS_OBSERVE_ONLY` は observe-only モードのために `core/utils/master_scheduler.c`
  に追加されています。
- MS フックが明示的なデッドラインミスイベントを出す場合は、`reaction_end.missed_deadline`
  の代わりにそれを使うよう `scripts/parse_e2.py` を更新してください。
- MS 設定生成器は、リアクションのインデックスが生成器の順序と一致することを前提と
  しています。生成されるインデックスが異なる場合は、パーサか設定生成器にマッピング
  処理を追加してください。

## 構成
```
ms-eval/
  lf-templates/        # C ロギングヘルパ
  lf-gen/              # 生成された LF プログラム
  scripts/             # 生成器・ランナ・パーサ・プロット
  logs/                # 生ログ
  results/             # CSV ＋ メタデータ
  figures/             # PDF グラフ
```
