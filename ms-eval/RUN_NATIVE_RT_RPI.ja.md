# Raspberry Pi ネイティブ実機評価（PREEMPT_RT・コア隔離込み）完全手順

目的: TCRS の主要結果（過負荷下の HC 保護と有界 backlog）を実機・自然な時間スケールで
再現し、「仮想化ホスト／ジッタ回避のための周期スケーリング」という妥当性への脅威を取り除く。
ハーネスはネイティブ実行（Docker は不要。Docker は配布の都合だっただけ）。

対象は Raspberry Pi 5（クアッドコア aarch64）を推奨。Pi 4 でも可だが Pi 5 の方が高速で
スロットリングしにくい。以下は Pi 5・64bit Raspberry Pi OS（Bookworm 以降）を前提に書く。

注意: カーネルのバージョンや RT カーネルの提供方法は時期で変わる。各手順の冒頭で
`uname -r` や `apt search` で実機の実際の値を確認してから進めること。

---

## 0. 用意するもの

- Raspberry Pi 5（4GB 以上）、公式 27W USB-C 電源、能動冷却（公式 Active Cooler 等）。
  冷却と電源が不十分だとサーマルスロットリングでジッタが入り、評価が汚れる。
- microSD（A2, 32GB+）または NVMe SSD（Pi 5 は PCIe HAT で NVMe 起動可・推奨）。
- 母艦 PC（Raspberry Pi Imager 用）、有線 LAN かつ SSH 運用が望ましい。

---

## 1. OS インストール

1. 母艦で Raspberry Pi Imager を使い、**Raspberry Pi OS (64-bit)** を書き込む。
   - 設定（歯車/編集）で、ホスト名・ユーザ名・パスワード・SSH 有効化・Wi‑Fi/ロケールを設定。
   - 評価専用なのでデスクトップ無し（Lite）でも可。GUI 不要なら Lite を推奨（余計な負荷が減る）。
2. 起動して SSH 接続し、最新化:
   ```sh
   sudo apt update && sudo apt full-upgrade -y
   sudo reboot
   ```
3. 余計な常駐を止める（任意・ジッタ低減）:
   ```sh
   sudo systemctl disable --now bluetooth ModemManager cups avahi-daemon 2>/dev/null || true
   ```

現在のカーネルを確認（後でビルドするブランチを合わせる）:
```sh
uname -r          # 例: 6.12.xx-v8-16k+ など
```

---

## 2. PREEMPT_RT カーネル

Linux 6.12 で PREEMPT_RT がメインライン化された。Raspberry Pi のカーネルが **6.12 以降**なら
外部 RT パッチは不要で、カーネル設定で「Fully Preemptible Kernel (Real-Time)」を選ぶだけでよい。
6.12 未満なら kernel.org の RT パッチを当てる（手順 2C）。

### 2A.（まず確認）配布済み RT カーネルがないか

時期によっては apt / `rpi-update` で RT 版が提供されている。あれば最短:
```sh
apt search linux-image 2>/dev/null | grep -iE 'rt|realtime|preempt'
```
該当があれば `sudo apt install <パッケージ>` 後に再起動し、手順 2D の確認へ。
無ければ 2B（ソースビルド）に進む。

### 2B. ソースからビルド（6.12 以降・メインライン RT）

```sh
# 依存
sudo apt install -y git bc bison flex libssl-dev make libncurses-dev

# 実機のカーネルに合うブランチを取得（例は 6.12 系。uname -r に合わせる）
git clone --depth=1 -b rpi-6.12.y https://github.com/raspberrypi/linux
cd linux

# Pi 5 (BCM2712) の既定 config
KERNEL=kernel_2712
make bcm2712_defconfig          # ※ Pi 4 の場合: KERNEL=kernel8 / make bcm2711_defconfig

# RT を有効化
make menuconfig
#   General setup ->
#     Preemption Model ->
#       Fully Preemptible Kernel (Real-Time)   を選択して Save/Exit
#   （6.12 未満でこの選択肢が無い場合は手順 2C のパッチを先に当てる）

# 識別用にローカルバージョンを付ける（任意・推奨）
scripts/config --set-str CONFIG_LOCALVERSION "-v8-rt"

# ビルド（Pi 5 ネイティブでおおむね 30〜60 分。温度に注意: vcgencmd measure_temp）
make -j4 Image.gz modules dtbs 2>&1 | tee ~/kbuild.log

# インストール
sudo make -j4 modules_install
sudo cp /boot/firmware/$KERNEL.img /boot/firmware/$KERNEL-backup.img   # 退避
sudo cp arch/arm64/boot/Image.gz /boot/firmware/$KERNEL.img
sudo cp arch/arm64/boot/dts/broadcom/*.dtb /boot/firmware/
sudo cp arch/arm64/boot/dts/overlays/*.dtb* /boot/firmware/overlays/
sudo cp arch/arm64/boot/dts/overlays/README /boot/firmware/overlays/
sudo reboot
```

> NVMe/SD の空きとビルド時間を節約したい場合は母艦でクロスコンパイルも可（aarch64
> ツールチェーンを使い `ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-` を付ける）。ネイティブ
> ビルドの方が手間が少ないので、初回はネイティブを推奨。

### 2C.（6.12 未満のときだけ）RT パッチを当てる

`uname -r` が 6.12 未満なら、対応する RT パッチを当ててから 2B の menuconfig に進む。
RT パッチのバージョンはカーネルに一致するものを選ぶ（無い場合は近い系列のカーネルにする）。
```sh
# 例: 6.6 系。https://cdn.kernel.org/pub/linux/kernel/projects/rt/ で版を確認
cd ~/linux
curl -OL https://cdn.kernel.org/pub/linux/kernel/projects/rt/6.6/patch-6.6.xx-rtyy.patch.gz
gzip -d patch-6.6.xx-rtyy.patch.gz
patch -p1 < patch-6.6.xx-rtyy.patch
# 以降は 2B の make menuconfig（Fully Preemptible …）→ build → install と同じ
```

### 2D. RT 化の確認

```sh
uname -a          # "PREEMPT_RT" が含まれていれば成功（例: ... SMP PREEMPT_RT ... aarch64）
```

---

## 3. コア隔離とタイミング衛生

### 3.1 コア隔離（cmdline.txt）

LF 実行を邪魔されないよう、コア 2,3 を OS のスケジューラ/タイマ/RCU から外す。
`/boot/firmware/cmdline.txt` は**1行**。末尾に（改行を入れず）追記する:
```
isolcpus=2,3 nohz_full=2,3 rcu_nocbs=2,3
```
編集して再起動:
```sh
sudo nano /boot/firmware/cmdline.txt   # 上記を同一行末尾に追記
sudo reboot
# 確認:
cat /sys/devices/system/cpu/isolated   # → 2-3
```
以降、実験はコア 2-3 に固定して実行する（`cpuset 2-3`）。割り込みを 0-1 に寄せたい場合は
cmdline に `irqaffinity=0-1` も追加可。

### 3.2 周波数ガバナを performance に

```sh
sudo apt install -y linux-cpupower
sudo cpupower frequency-set -g performance
cpupower frequency-info | grep -i governor      # performance を確認
```
（ネイティブ実機評価ランナー `run_baremetal.sh` もこれを best-effort で試みる。）

### 3.3 冷却・電源・スロットリング確認

```sh
vcgencmd measure_temp        # 負荷時も 80℃ 未満が目安
vcgencmd get_throttled       # 0x0 が理想（0以外はスロットリング/電圧低下の痕跡）
```
`get_throttled` が 0 以外なら冷却・電源を見直す（結果が汚れる）。

---

## 4. ツールチェーン

```sh
sudo apt install -y git cmake build-essential default-jdk python3 python3-matplotlib \
                    util-linux linux-cpupower rt-tests
```
- `lfc`（Lingua Franca コンパイラ、Java 必須）を導入し `bin/` を PATH に通す。
  もしくは実行時に `LFC_BIN=/path/to/lfc` を渡す。確認:
  ```sh
  lfc --version
  ```
  （評価対象は reactor-c タグ `ms-eval-v1.0`。lfc は 0.11.x 系で検証済み。）

---

## 5. リポジトリ取得

```sh
cd ~
git clone --recursive https://github.com/ertlnagoya/lf-ms-evaluation.git
cd lf-ms-evaluation
# サブモジュール未取得なら:
git submodule update --init --recursive
git -C reactor-c describe --tags    # ms-eval-v1.0 を確認
```

---

## 6. ビルド疎通（スモーク）

```sh
cd ~/lf-ms-evaluation
./run_e3.sh --steps 5 --load-factors 1.0
```
期待: `e3_overload` がビルドされ、MS パッチが適用・再ビルドされ、
`ms-eval/logs/e3/<timestamp>/` にログが出る。
`could not locate the generated runtime tree ... aborting` が出たら、`src-gen/` 配下の
実際のパスを控えて連絡（`run_e3.sh` の探索を合わせる）。

---

## 7. 本実験

コア 2,3 を隔離している前提で、そこへ固定して実行する。
```sh
cd ~/lf-ms-evaluation

# 計画だけ表示（実行なし）
ms-eval/scripts/run_baremetal.sh dry 2-3

# 過負荷オンセット（E5 backlog の決め手）
ms-eval/scripts/run_baremetal.sh backlog 2-3

# E4/E5/E6 本実験
ms-eval/scripts/run_baremetal.sh final 2-3
```
結果は `ms-eval/tcrs-data/{e4,e5,e6}/` に出る。実行ログに
`failed to derive rid->runtime index mapping; falling back to 0..N-1` が**出ない**ことを確認
（短すぎる走でのみ出るはず。本実験の steps なら出ない）。

---

## 8. 自然な時間スケール（1ms）チェック ＝ ネイティブ実機の本丸

隔離コアならジッタは << 1ms のはずなので、10ms へスケールせず **自然な 1ms 周期**でも
結果が崩れないことを示せる。`periods` モードで周期をスイープ:
```sh
taskset -c 2-3 python3 ms-eval/scripts/run_e4_e6_main.py --mode periods \
  --pr-periods 1000,2000,5000 --pr-loads 4.0,5.0 --pr-repeats 15 --pr-steps 150
```
まず低負荷で `baseline` ログの per-tag wall time が周期に追従し、過負荷で周期を超える
（設計どおり）ことを確認。1ms でジッタが支配的なら 2〜5ms に落とす。

### ジッタの実測（任意・査読対策に有効）
```sh
sudo cyclictest -m -p99 -t1 -a2 -i1000 -D60   # コア2に固定, 1ms, 60秒, max latency を見る
```
PREEMPT_RT＋隔離で max latency が数十µs オーダなら 1ms 周期評価は妥当。

---

## 9. 図・表の再生成

```sh
python3 ms-eval/scripts/plot_paper_figs.py     # ms-eval/tcrs-data/{e4,e5} を読む
# 図は ms-eval/tcrs-data/figures/ に出力
```
論文用に使う場合は、生成図を `paper/figures/` に反映（`fig_hc_miss.pdf`, `fig_backlog.pdf`）。
upLaTeX+dvipdfmx なら `extractbb` で `.xbb` を作り直すこと。

---

## 10. 論文への反映

- §Experimental Setup の環境記述を、実機の値へ差し替え:
  「reactor-c `ms-eval-v1.0` ＋ lfc 0.11.x を、Raspberry Pi 5（PREEMPT_RT カーネル
  `<uname -r>`、コア 2-3 隔離、performance ガバナ）でネイティブ実行。周期 P=1ms（必要なら 2–5ms）」。
- §Threats to Validity の「virtualized host／native real-hardware は今後」を、
  「Raspberry Pi 5・PREEMPT_RT・隔離コアで確認済み」に更新。
- 反復数 n と P は実機の実値で更新（例: n=15、P=1ms、cyclictest の max latency 併記）。

---

## トラブルシュート

- **lfc が見つからない**: `LFC_BIN=/path/to/lfc` を export、または LF の `bin/` を PATH に。
- **cmake/Java の版ずれ**: 初回ビルドで大半の問題が出る。まず `./run_e3.sh --steps 5` を通す。
- **スロットリング**: `vcgencmd get_throttled` が 0 以外 → 冷却・電源を見直す。
- **隔離が効かない**: `cat /sys/devices/system/cpu/isolated` が `2-3` か確認。cmdline.txt は
  必ず1行（途中改行禁止）。
- **RT 選択肢が menuconfig に無い**: カーネルが 6.12 未満。手順 2C の RT パッチを先に当てる。
- **uname に PREEMPT_RT が出ない**: `/boot/firmware/$KERNEL.img` の差し替え漏れ、または
  config.txt が別カーネルを指している。Image コピーと再起動を再確認。

参考: Raspberry Pi 公式「Building the kernel」、kernel.org PREEMPT_RT プロジェクト、
コミュニティの PREEMPT_RT ビルド事例（本文の Sources を参照）。
