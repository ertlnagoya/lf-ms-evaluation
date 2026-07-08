# lf-ms-evaluation

Lingua Franca（LF）の C ランタイム上で動く**ユーザ空間 Master Scheduler（MS）**の
評価ハーネスです。MS は LF ランタイムの **ready-set 境界**で、観測・介入・制御的
デグラデーション・OS レベル協調といった段階的な制御を加えます。これらはすべて
LF の**論理時間（決定的リアクティブ）セマンティクスを保ったまま**行われ、カーネル・
RTOS・ハイパーバイザに一切手を加えずユーザ空間だけで動作します。

このリポジトリは論文の評価アーティファクトの基盤です。評価対象のランタイムは
[`reactor-c`](https://github.com/ertlnagoya/reactor-c) を submodule としてタグ固定して
いるので、チェックアウトすれば「どの MS 実装を測ったか」を正確に再現できます。

英語版は [README.md](README.md) にあります。

## リポジトリ構成

```
lf-ms-evaluation/
├── reactor-c/                 # submodule: MS ランタイム本体（タグ ms-eval-v1.0 に固定）
├── ms-eval/                   # 評価ハーネス（scripts, config, lf-templates, figures）
├── run_e1.sh run_e2.sh run_e3.sh run_all.sh
└── README.md
```

## セットアップ

```bash
git clone --recursive https://github.com/ertlnagoya/lf-ms-evaluation.git
# もしくは通常の clone のあとで:
git submodule update --init --recursive
```

必要なもの: `PATH` の通った `lfc`（Lingua Franca。または `LFC=/path/to/lfc`）、
`cmake`、C ツールチェイン、Python 3（＋ `matplotlib`）、そして物理実機／コア固定
実行のための `util-linux`（`taskset`）。

## 実行

```bash
./run_e3.sh --steps 5 --load-factors 1.0        # ビルド＋MSパッチの簡易動作確認
./run_all.sh                                     # E1/E2/E3 を通しで実行
ms-eval/scripts/run_baremetal.sh backlog 2-3     # E5 過負荷オンセット（物理実機、コア 2-3）
ms-eval/scripts/run_baremetal.sh final           # E4/E5/E6 本実験
```

結果と図は `ms-eval/` 以下に書き出されます。`run_e3.sh` はランタイムのソースを
`reactor-c` submodule から読み込みます（`REACTOR_C=/abs/path` で上書き可能）。

## 実験一覧

| ID | 内容 | ドライバ／スクリプト |
|----|------|----------------------|
| E1 | MS ランタイムのオーバーヘッド（マイクロベンチ） | `run_e1.sh` |
| E2 | HC デッドラインミス率: LF baseline vs MS / MS+RT | `run_e2.sh` |
| E3 | 過負荷下の制御的デグラデーション（baseline / ms / degrade） | `run_e3.sh` |
| E4 | 過負荷の検証: 容量モデル＋周期ロバスト性 | `ms-eval/scripts/run_e4_e6_main.py` |
| E5 | backlog × worker/負荷スイープ（過負荷オンセット） | `ms-eval/scripts/run_e5_backlog_worker_sweep.py` |
| E6 | ポリシ感度: LC バジェット, degrade lag, ready-queue 長 | `ms-eval/scripts/run_e6_policy_sensitivity.py` |

## 評価対象ランタイム（reactor-c）と論文

評価対象の MS ランタイムは `reactor-c` submodule で、タグにより固定されています。
下表は各論文と、使用したランタイムのコミット・報告した実験の対応です。

| 日付 | 発表・文書 | タイトル | reactor-c タグ → コミット（日付） | 実験 |
|------|-----------|----------|-----------------------------------|------|
| 2026-03 | 技術論文（`TechnicalPaper-MS-v1.1.pdf`） | A Semantics-Preserving Master Scheduler for Lingua Franca | `ms-v1.0` / `v1.0` → `6a5ba2fc`（2026-03-12） | E1, E2, E3 |
| 2026-05 | [JSAE 2026 春季大会](https://tech.jsae.or.jp/paperinfo/en/content/p202601.039/) / 自動車技術会 2026年春季大会 | A Semantics-Preserving Master Scheduler for Mixed-Criticality Control in Lingua Franca | `ms-v1.0` → `6a5ba2fc`（2026-03-12） | E1, E2, E3 |
| 2026-06 | TCRS 2026 | Controlled Degradation, Not Dispatch Reordering: Semantics-Preserving Overload Control in Lingua Franca | `ms-eval-v1.0` → `997a8df4`（2026-06-24） | E3, E4, E5, E6 |

3 本とも著者は共通です: **松原 豊**（名古屋大学）、**Wenhung Kevin Huang**、
**岩井 明史**（株式会社デンソー）。

ある論文のランタイムを正確に再現するには、
`git -C reactor-c checkout <tag>`（TCRS 2026 なら `ms-eval-v1.0`）としてからビルド
してください。

### 評価プラットフォーム（TCRS 2026）

| 項目 | 詳細 |
|------|------|
| ホスト | Raspberry Pi 5（クアッドコア Arm Cortex-A76）、64-bit Raspberry Pi OS（Docker を使わずネイティブ実行） |
| カーネル | `PREEMPT_RT` `6.18.34+rpt-rpi-v8-rt`、performance ガバナ、コア 2,3 を隔離、全4コアに固定して実行 |
| ジッタ | cyclictest の最悪スケジューリング遅延 ≤ 3 µs（コア 2、1 ms、60 秒） |
| LF コンパイラ | `lfc` 0.11.0 |
| ランタイム | reactor-c タグ `ms-eval-v1.0` |
| ワーカ | {1, 2}（4コアの Pi。4 ワーカだと飽和し、ランタイム/MS スレッド用の余裕がなくなる） |
| 反復 | E4/E5/E6 = 各条件 15 回、平均と 95% 信頼区間 |
| 周期 | 自然な P = 2 ms（主点）。1 ms（容量端でわずかに残差）と 10 ms でも確認 |

> 同じハーネスの初期の内部実行では MacBook Air（Apple M3）上の Docker Linux
> コンテナを使いました。TCRS 2026 では上記の物理実機（Pi）の数値がそれに取って
> 代わります。

初期の技術論文／JSAE の結果（タグ `ms-v1.0`）は別の構成（例: n = 30）で取得して
います。詳細は各論文を参照してください。

## ドキュメント

- [`ms-eval/README.md`](ms-eval/README.md)（日本語: [`.ja`](ms-eval/README.ja.md)）
  — 評価スイートの詳細: クイックスタート、評価プログラム、必要環境、出力ファイル。
- [`ms-eval/LOG_FORMAT.md`](ms-eval/LOG_FORMAT.md)（日本語:
  [`.ja`](ms-eval/LOG_FORMAT.ja.md)） — パーサが用いる構造化ログ形式
  （アプリ JSONL と MS ログ）。
- [`README_ms_performance_test.md`](README_ms_performance_test.md)
  （日本語: [`.ja`](README_ms_performance_test.ja.md)）
  — 再現手順ガイド（E1/E2/E3）とタグ付きアーティファクトの再現手順。
- [`ms-eval/RUN_NATIVE_RT_RPI.md`](ms-eval/RUN_NATIVE_RT_RPI.md)（日本語:
  [`.ja`](ms-eval/RUN_NATIVE_RT_RPI.ja.md)） — Raspberry Pi での物理実機評価:
  OS 構築、PREEMPT_RT カーネル、コア隔離、実行、自然な 1 ms チェック。
- [`reactor-c/README_ms.md`](reactor-c/README_ms.md) — MS 実装の概要（Phase 0〜4）。
  ランタイム submodule 内にあります。

## 再現性 / タグ

- `reactor-c` submodule はタグ
  [`ms-eval-v1.0`](https://github.com/ertlnagoya/reactor-c/tree/ms-eval-v1.0)
  に固定（評価対象ランタイム）。
- MS 実装タグ:
  [`ms-v1.0`](https://github.com/ertlnagoya/reactor-c/tree/ms-v1.0)。
- 後でランタイムを更新するには
  `git -C reactor-c fetch && git -C reactor-c checkout <ref>` としてから、
  更新された submodule の gitlink をこのリポジトリでコミットします。
