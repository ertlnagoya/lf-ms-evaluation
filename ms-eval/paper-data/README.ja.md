# 生の実験データ（TCRS 2026）

論文の表・図の元になった結果 CSV と図です。Raspberry Pi 5（PREEMPT_RT）での実行に
よるもので、reactor-c タグ ms-eval-v1.0、lfc 0.11.0、P=2 ms、ワーカ {1,2}、n=15 です。
条件ごとの生の行は `*_long.csv` にあります。

英語版は [README.md](README.md) にあります。

- e5/  worker スイープ（tcrs_bk_backlog_w{1,2}）: ワーカごとの summary と long CSV、
       および結合した backlog CSV — Table I, fig_hc_miss, fig_backlog の元。
- e6/  バジェット感度（tcrs_e6_policy_*）— バジェットの段落の元。
- figures/  fig_hc_miss.pdf, fig_backlog.pdf。
- validation/  周期非依存性（rpi_{1ms,2ms,10ms}）と、機能／コア割り当ての確認
       （rpi_check, rpi_4core, rpi_sweep_w2, ...）。

容量モデル E4 は e5 スイープからの解析的な導出です（専用ディレクトリはありません）。
サイズの大きい実行ごとの JSON/MS ログ（約 0.5 GB、ms-eval/logs/）はコミットして
いません。`ms-eval/scripts/run_baremetal.sh` で再生成してください。

## 列の注記（CSV を解釈する前に読んでください）

- `*_hc_miss_mean` — HC のみのデッドラインミス率（%）。HC リアクション
  （reaction_id < hc）を対象に、n=15 回の平均。`*_ci_low/high` を伴う。
- `*_hc_latency_p95_us` — HC のテール遅延 =（物理完了時刻 − 論理 release）。
  nearest-rank の 95 パーセンタイル、単位はマイクロ秒。
- `*_lc_completion_mean` — released された LC リアクションのうち実行された割合（%）
  （degrade でバジェット b=2/4 なら ≈50%）。
- `fallbacks_mean`, `fallback_missing_metadata_mean`, `fallback_no_candidate_mean`
  — デフォルトスケジューリングへの安全なフォールバック（例: メタデータ欠落）。
  **これらの実行では 0**。
- **`violations_mean`（および `degrade_violations`）— 名前に反して、これは
  `event=mismatch` を数えています。すなわち MS/runtime の *pick mismatch*
  （ランタイムが MS の推奨とは別の ready リアクションを実行した＝別の妥当なタグ内
  順序）です。これは良性で（LF の決定性は構成上保たれる）、意味的違反でも
  デッドライン違反でもありません。**（ここでは ≈0.3〜0.7/run。）列名は後方互換の
  ために従来のまま残しています。論文では "MS/runtime pick mismatches" と呼んでいます。
- 信頼区間は正規近似の 95%（平均 ± 1.96·sd/√n）。
