# Bare-metal evaluation on Raspberry Pi (PREEMPT_RT + core isolation)

Goal: reproduce the key TCRS result (HC protection and bounded backlog under
overload) on real hardware at a natural timescale, removing the "virtualized
host / period scaled to escape jitter" threat to validity. The harness runs
natively (Docker was only packaging).

Use a Raspberry Pi 5 (quad-core aarch64); a Pi 4 also works but the Pi 5 is
faster and throttles less. The steps below assume a Pi 5 with 64-bit Raspberry
Pi OS (Bookworm or later).

Kernel versions and the way RT kernels are shipped change over time. At each step
check the actual values on your board (`uname -r`, `apt search`) before
proceeding.

---

## 0. Hardware

- Raspberry Pi 5 (4 GB+), official 27 W USB-C supply, active cooling (e.g. the
  official Active Cooler). Inadequate cooling or power causes thermal throttling,
  which injects jitter and corrupts the measurement.
- microSD (A2, 32 GB+) or, preferably, an NVMe SSD (the Pi 5 can boot from NVMe
  via a PCIe HAT).
- A host PC for Raspberry Pi Imager; wired Ethernet + SSH is preferred.

---

## 1. Install the OS

1. With Raspberry Pi Imager, write **Raspberry Pi OS (64-bit)**.
   - In the settings, set hostname, user/password, enable SSH, and set
     Wi-Fi/locale. A Lite (no-desktop) image is fine and reduces background load.
2. Boot, SSH in, and update:
   ```sh
   sudo apt update && sudo apt full-upgrade -y
   sudo reboot
   ```
3. Optionally stop unneeded daemons (lower jitter):
   ```sh
   sudo systemctl disable --now bluetooth ModemManager cups avahi-daemon 2>/dev/null || true
   ```

Note the running kernel (you will match the build branch to it):
```sh
uname -r          # e.g. 6.12.xx-v8-16k+
```

---

## 2. PREEMPT_RT kernel

PREEMPT_RT was mainlined in Linux 6.12. If the Pi kernel is **6.12 or newer** you
do **not** need an external RT patch — just enable "Fully Preemptible Kernel
(Real-Time)" in the kernel config. For older kernels, apply the kernel.org RT
patch (step 2C).

### 2A. (Check first) Is a prebuilt RT kernel available?

Depending on the date, an RT kernel may be packaged via apt / `rpi-update`:
```sh
apt search linux-image 2>/dev/null | grep -iE 'rt|realtime|preempt'
```
If there is a match, `sudo apt install <package>`, reboot, and jump to 2D.
Otherwise build from source (2B).

### 2B. Build from source (6.12+, mainline RT)

```sh
# dependencies
sudo apt install -y git bc bison flex libssl-dev make libncurses-dev

# clone the branch matching your running kernel (example: 6.12.y)
git clone --depth=1 -b rpi-6.12.y https://github.com/raspberrypi/linux
cd linux

# default config for the Pi 5 (BCM2712)
KERNEL=kernel_2712
make bcm2712_defconfig          # Pi 4: KERNEL=kernel8 / make bcm2711_defconfig

# enable RT
make menuconfig
#   General setup ->
#     Preemption Model ->
#       Fully Preemptible Kernel (Real-Time)   then Save/Exit
#   (if this option is missing, the kernel is < 6.12 -> apply the patch in 2C first)

# tag the build (optional, recommended)
scripts/config --set-str CONFIG_LOCALVERSION "-v8-rt"

# build (native on a Pi 5 is roughly 30-60 min; watch temp: vcgencmd measure_temp)
make -j4 Image.gz modules dtbs 2>&1 | tee ~/kbuild.log

# install
sudo make -j4 modules_install
sudo cp /boot/firmware/$KERNEL.img /boot/firmware/$KERNEL-backup.img   # back up
sudo cp arch/arm64/boot/Image.gz /boot/firmware/$KERNEL.img
sudo cp arch/arm64/boot/dts/broadcom/*.dtb /boot/firmware/
sudo cp arch/arm64/boot/dts/overlays/*.dtb* /boot/firmware/overlays/
sudo cp arch/arm64/boot/dts/overlays/README /boot/firmware/overlays/
sudo reboot
```

> You can cross-compile on the host (aarch64 toolchain, `ARCH=arm64
> CROSS_COMPILE=aarch64-linux-gnu-`) to save time, but native building is simpler
> for a first pass.

### 2C. (Only for kernels < 6.12) Apply the RT patch

If `uname -r` is below 6.12, apply the matching RT patch before the menuconfig in
2B. Pick the patch version that matches the kernel (if none exists, use a kernel
series that has one).
```sh
# example: 6.6 series; check https://cdn.kernel.org/pub/linux/kernel/projects/rt/ for versions
cd ~/linux
curl -OL https://cdn.kernel.org/pub/linux/kernel/projects/rt/6.6/patch-6.6.xx-rtyy.patch.gz
gzip -d patch-6.6.xx-rtyy.patch.gz
patch -p1 < patch-6.6.xx-rtyy.patch
# then continue with make menuconfig (Fully Preemptible ...) -> build -> install as in 2B
```

### 2D. Confirm RT

```sh
uname -a          # should contain "PREEMPT_RT" (e.g. ... SMP PREEMPT_RT ... aarch64)
```

---

## 3. Core isolation and timing hygiene

### 3.1 Isolate cores (cmdline.txt)

Keep the LF run undisturbed by removing cores 2,3 from the OS scheduler/tick/RCU.
`/boot/firmware/cmdline.txt` is a **single line**; append (no newline) at the end:
```
isolcpus=2,3 nohz_full=2,3 rcu_nocbs=2,3
```
Edit and reboot:
```sh
sudo nano /boot/firmware/cmdline.txt   # append the above on the same line
sudo reboot
# verify:
cat /sys/devices/system/cpu/isolated   # -> 2-3
```
Run the experiment pinned to cores 2-3. To steer interrupts to 0-1, you may also
add `irqaffinity=0-1` to cmdline.

### 3.2 Performance governor

```sh
sudo apt install -y linux-cpupower
sudo cpupower frequency-set -g performance
cpupower frequency-info | grep -i governor      # expect performance
```
(`run_baremetal.sh` also attempts this on a best-effort basis.)

### 3.3 Cooling / power / throttling check

```sh
vcgencmd measure_temp        # stay below ~80C even under load
vcgencmd get_throttled       # 0x0 is ideal (non-zero = throttling / undervoltage)
```
If `get_throttled` is non-zero, fix cooling/power before trusting results.

---

## 4. Toolchain

```sh
sudo apt install -y git cmake build-essential default-jdk python3 python3-matplotlib \
                    util-linux linux-cpupower rt-tests
```
- Install `lfc` (the Lingua Franca compiler; needs Java) and put its `bin/` on
  `PATH`, or pass `LFC_BIN=/path/to/lfc` at run time. Verify:
  ```sh
  lfc --version
  ```
  (The evaluated runtime is reactor-c tag `ms-eval-v1.0`; `lfc` 0.11.x was used.)

---

## 5. Get the repository

```sh
cd ~
git clone --recursive https://github.com/ertlnagoya/lf-ms-evaluation.git
cd lf-ms-evaluation
git submodule update --init --recursive       # if not cloned with --recursive
git -C reactor-c describe --tags              # expect ms-eval-v1.0
```

---

## 6. Build smoke test

```sh
cd ~/lf-ms-evaluation
./run_e3.sh --steps 5 --load-factors 1.0
```
Expect `e3_overload` to build, the MS patch to be applied and rebuilt, and logs
under `ms-eval/logs/e3/<timestamp>/`. If you see `could not locate the generated
runtime tree ... aborting`, note the actual path under `src-gen/` and report it so
`run_e3.sh`'s discovery can be adjusted.

---

## 7. Run the experiment

With cores 2,3 isolated, pin the run there.
```sh
cd ~/lf-ms-evaluation

# plan only (no execution)
ms-eval/scripts/run_baremetal.sh dry 2-3

# overload onset (the decisive E5 backlog run)
ms-eval/scripts/run_baremetal.sh backlog 2-3

# E4/E5/E6 main experiment
ms-eval/scripts/run_baremetal.sh final 2-3
```
Results land in `ms-eval/tcrs-data/{e4,e5,e6}/`. Confirm the run log does **not**
print `failed to derive rid->runtime index mapping; falling back to 0..N-1` (that
only appears on very short runs).

---

## 8. Natural-timescale (1 ms) check — the point of bare metal

On isolated cores, jitter should be << 1 ms, so you can run at a **natural 1 ms
period** (no 10 ms scaling) and show the result is not an artifact of the scaled
operating point. Use the `periods` mode:
```sh
taskset -c 2-3 python3 ms-eval/scripts/run_e4_e6_main.py --mode periods \
  --pr-periods 1000,2000,5000 --pr-loads 4.0,5.0 --pr-repeats 15 --pr-steps 150
```
First check that at low load the per-tag wall time tracks the period and exceeds
it under overload (as designed); inspect a `baseline` log under
`ms-eval/logs/e3/<timestamp>/`. If 1 ms is jitter-dominated, fall back to 2-5 ms.

### Measure jitter (optional, useful for reviewers)
```sh
sudo cyclictest -m -p99 -t1 -a2 -i1000 -D60   # pin to core 2, 1 ms, 60 s, report max latency
```
With PREEMPT_RT + isolation, a max latency in the tens of microseconds justifies
the 1 ms operating point.

---

## 9. Regenerate figures/tables

```sh
python3 ms-eval/scripts/plot_paper_figs.py     # reads ms-eval/tcrs-data/{e4,e5}
# figures are written under ms-eval/tcrs-data/figures/
```
For the paper, copy the regenerated `fig_hc_miss.pdf` / `fig_backlog.pdf` into
`paper/figures/` and rebuild the `.xbb` with `extractbb` (upLaTeX + dvipdfmx).

---

## 10. Update the manuscript

- Replace the platform sentence in §Experimental Setup with the real values:
  "reactor-c `ms-eval-v1.0` with `lfc` 0.11.x run natively on a Raspberry Pi 5
  (PREEMPT_RT kernel `<uname -r>`, cores 2-3 isolated, performance governor),
  period P = 1 ms (2-5 ms if needed)."
- Update §Threats to Validity: change "virtualized host / future bare-metal" to
  "confirmed on a Raspberry Pi 5 with PREEMPT_RT and isolated cores."
- Update n and P to the real values (e.g. n = 15, P = 1 ms; cite the cyclictest
  max latency).

---

## Troubleshooting

- **lfc not found**: export `LFC_BIN=/path/to/lfc`, or put the LF `bin/` on `PATH`.
- **cmake/Java version mismatch**: most issues show up on the first build; get
  `./run_e3.sh --steps 5` passing first.
- **Throttling**: `vcgencmd get_throttled` non-zero -> fix cooling/power.
- **Isolation not applied**: check `cat /sys/devices/system/cpu/isolated` is `2-3`;
  cmdline.txt must stay a single line (no mid-line newline).
- **No RT option in menuconfig**: kernel is < 6.12; apply the RT patch (2C) first.
- **`uname` lacks PREEMPT_RT**: the new `Image.gz` was not copied to
  `/boot/firmware/$KERNEL.img`, or config.txt points at another kernel. Recheck
  the image copy and reboot.

A Japanese version of this guide is in `RUN_BAREMETAL_RPI.ja.md`.
