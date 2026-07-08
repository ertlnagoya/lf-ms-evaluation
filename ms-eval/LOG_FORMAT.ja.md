# MS 評価ログ形式

この評価スイートは、実行ごとに 2 つのログストリームを使います:

1) master scheduler が出力する **MS ログ**（`LF_MS_LOG`）。既存の key-value 行形式。
2) ベンチマークプログラムが出力する **アプリログ**（`LF_APP_LOG`）。JSONL 形式。

どちらのログも `ms-eval/logs/<exp>/<timestamp>/<run>/` 以下に保存されます。

英語版は [LOG_FORMAT.md](LOG_FORMAT.md) にあります。

## アプリログ（JSONL）
各行は `type` フィールドを持つ JSON オブジェクトです。

### `run_start`
フィールド:
- `ts_mono_ns`: モノトニックなタイムスタンプ（ns）
- `experiment`: `e1`, `e2`, `e3` のいずれか
- `mode`: `baseline`, `observe`, `intervene`, `degrade` のいずれか
- `reactions`: リアクション数（ready set のサイズ）
- `steps`: 反復回数（論理ステップ数）
- `workload_us`: 基本ワークロード（us）
- `deadline_us`: 相対デッドライン（us）
- `load_factor`: 低 criticality の負荷係数（E3）
- `seed`: 乱数シード

### `reaction_start`
フィールド:
- `ts_mono_ns`: モノトニックなタイムスタンプ（ns）
- `logical_time_ns`: 論理時刻タグ（ns）
- `reaction_id`: ベンチマークが生成したリアクションのインデックス

### `reaction_end`
フィールド:
- `ts_mono_ns`: モノトニックなタイムスタンプ（ns）
- `logical_time_ns`: 論理時刻タグ（ns）
- `reaction_id`
- `deadline_us`
- `missed_deadline`（0/1）: ベンチマークが計算（物理時刻 vs 論理時刻＋デッドライン）

### `injection`
フィールド:
- `ts_mono_ns`
- `logical_time_ns`
- `reaction_id`
- `injected_delay_us`
- `deadline_us`
- `expected_miss`（0/1）

### `run_end`
フィールド:
- `ts_mono_ns`

## MS ログ（key-value 行）
ランタイムは次のような行を出力します:
```
<mono_ns>,<LEVEL>,event=ready env=0 reaction_index=5 logical=1000000 deadline=5000 is_input=0 updated=0
```

パーサが使う主なイベント:
- `event=ready`: リアクションが ready set に入る。`logical` と `reaction_index` を含む。
- `event=ready_drop`: ready エントリの削除（例: デグラデーションによる skip）。
- `event=pick_next`: スケジューラの決定。`logical` と `candidate` を含む。
- `event=mismatch`: ランタイムが MS の推奨とは別の ready リアクションを実行した
  （良性の、別の妥当なタグ内順序。意味的違反でもデッドライン違反でもない）。
- `event=runtime_selected_missing`: MS が追跡していないリアクションをランタイムが実行した。
- `event=degrade`: デグラデーション動作が適用された。

## メモ / TODO
- ランタイムが明示的なデッドラインミスイベントを出す場合は、それを検出に使うよう
  `scripts/parse_e2.py` を更新してください。
- MS ログのリアクションインデックスが生成器の ID と一致しない場合は、パーサに
  マッピング処理を追加してください。
