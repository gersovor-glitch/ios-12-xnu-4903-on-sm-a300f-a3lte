# CLAUDE.md

Guidance for Claude Code when working in this repository.

## Goal and Target

- **Goal**: iOS 12.4 as the **only, primary OS** on the device. Android is not used and not kept.
- **OS**: iOS 12.4 (build 16G77), kernel `xnu-4903.270.47`
- **Device**: Samsung Galaxy A3 (2015), **SM-A300F / a3lte**
- **SoC**: Snapdragon 410 (MSM8916), 4x Cortex-A53, **no ARMv8 Crypto Extensions**
- **Bootloader**: lk2nd, already flashed. Keep it as the boot chain base.
- Everything not about SM-A300F + iOS 12.4 is reference only.

## Repository Layout

```
ios-12-xnu-4903-on-sm-a300f-a3lte/
├── xnu-4903/                     # XNU source (target kernel)
├── android_device_samsung_a3lte/ # Hardware reference only (GPIO, regulators, I2C). Android is not run.
├── ios-16-SM-G360T1/             # REFERENCE ONLY: other device (Core Prime), other OS version
│   ├── loader/                   # lk2nd -> XNU loader, bringup-launchd, packaging
│   ├── shim/                     # FDT / framebuffer / ramoops / PSCI helpers
│   ├── hakios-patches/           # kernel patches + drivers
│   └── docs/                     # logs (REPORT.md, HANDOFF-*.md)
└── IPhone_6_ios_12.4_16G77_Restore.ipsw   # not in git
```

Re-verify anything from `ios-16-SM-G360T1/` before using it on a3lte. Do not copy device-specific values (touch controller, display, boot-args) blindly.

## Safety

Writing to eMMC/microSD is allowed **only when performed by the human**. Claude never runs flash/erase/`dd`/`heimdall`/`fastboot flash` itself: it prepares the exact command (target partition, image, sha256) and waits for explicit confirmation for each write. Rules to avoid an unrecoverable device:

- **Before the first write**, make a full backup of eMMC partitions and store it outside git. Pay special attention to: lk2nd/aboot, **sbl1, rpm, tz** (TrustZone), SBL/bootloader chain, `modemst1`, `modemst2`, `fsg`, `persist`, EFS (IMEI, calibration, MAC addresses). These are device-unique and cannot be re-created.
- **Backup procedure**: store backups outside the repo (e.g. `~/a3lte-backup/`), record `sha256` of every image, verify by read-back/re-dump. No write until the backup is verified.
- **lk2nd location (`boot` vs `aboot`) is inconsistent in this file** — confirm from the real PIT before any write. Writing the `boot` partition overwrites lk2nd: treat it as boot-chain write.
- **Never overwrite `sbl1`, `rpm`, `tz`, `aboot` (lk2nd) without a verified backup AND a confirmed recovery path** (Download mode 04e8:685d with Heimdall/Odin working, or EDL 05c6:9008 verified on this device). If the recovery path is untested, do not write these partitions.
- Always know the recovery path: Download mode (04e8:685d, Odin/Heimdall) and EDL (05c6:9008, Qualcomm) are different interfaces. **EDL is unverified on SM-A300F** — do not rely on it until confirmed on device. Confirm Download mode works before flashing the boot chain.
- Prefer `fastboot boot` for iteration; flash only a verified, working image.
- Log every flash/erase/dd command with target partition and image hash in `docs/REPORT.md`.
- Never commit or print: IPSW contents, dyld cache, identity blobs, serials, ECID, keys, EFS backups, stock firmware, vendor blobs. Check `.gitignore` before `git add`; do not use `git add -A` blindly.
- Required `.gitignore` entries: `*.ipsw`, `*.img`, `*.bin`, `*.pit`, `backup*/`, `*efs*`, `*dyld*`, `*.key`. Run `git status` and review the diff before every commit; consider a pre-commit hook that rejects these patterns and any ECID/serial string.
- Ask before destructive git operations (`reset --hard`, `clean -fd`, force push).

## Build

### XNU (`xnu-4903/`, macOS host required)

```bash
cd xnu-4903
make SDKROOT=<path/to/iPhoneOS.sdk> ARCH_CONFIGS=ARM64 KERNEL_CONFIGS=DEVELOPMENT
```

- X86_64 builds are irrelevant for this target.
- Output: `BUILD/obj/<CONFIG>/ARM64/kernel.<config>` and `.unstripped`.
- Extra targets: `make KERNEL_CONFIGS=RELEASE`, `make checkstyle`, `make restyle`.

### Loader

```bash
cd ios-16-SM-G360T1/loader
export PATH=/usr/lib/llvm-18/bin:$PATH
mkdir -p out
for file in start.S loader.c; do
  clang --target=aarch64-none-elf -mcpu=cortex-a53 -O2 -ffreestanding -fno-builtin \
    -fno-stack-protector -fno-pic -mstrict-align -mgeneral-regs-only -Wall -Wextra \
    -I../shim -c "$file" -o "out/${file%.*}.o"
done
ld.lld -T shim.ld --build-id=none -o out/loader.elf out/start.o out/loader.o
llvm-objcopy -O binary out/loader.elf out/loader-Image
python3 - <<'PY'
from pathlib import Path
import subprocess,struct
symbols=subprocess.check_output(['llvm-nm','out/loader.elf'],text=True)
end=next(int(line.split()[0],16) for line in symbols.splitlines() if line.split()[-1]=='_end')
p=Path('out/loader-Image')
with p.open('r+b') as f:f.truncate(end)
h=p.read_bytes()[:64]
assert h[56:60]==b'ARM\x64'
assert struct.unpack_from('<Q',h,16)[0]==end
print('ARM64 loader Image:',end,'bytes')
PY
llvm-readelf -r out/loader.o > out/loader-relocations.txt
if grep -q 'R_AARCH64_ABS64' out/loader-relocations.txt; then
 echo 'Unexpected absolute C relocation' >&2; exit 1
fi
```

- Toolchain: LLVM 18 (`/usr/lib/llvm-18/bin`) with `clang`, `ld.lld`, `llvm-objcopy`, `llvm-nm`, `llvm-readelf`
- Target: `aarch64-none-elf`, `-mcpu=cortex-a53`
- Output: `out/loader-Image` (raw binary), `out/loader.elf` (with symbols)
- Relocation guard: rejects `R_AARCH64_ABS64` C relocations (position-dependent code fails at runtime)
- Payload build (XNUP): see the payload section of `build.sh`; it references `../hakios/xnuloader/mkpayload.py`
  - **Location**: External workspace at `/home/laptop/Documents/Codex/2026-11-03/ho/work/hakios/xnuloader/mkpayload.py`
  - **Not in this repository** — must be copied or reimplemented for a3lte
- This builds the **Core Prime reference loader**. An a3lte loader directory does not exist yet. Host note: XNU needs macOS; this loader build assumes a Linux LLVM path.

## Boot Flow and Stages

```
lk2nd -> loader (start.S, loader.c: FDT fixup, framebuffer, ramoops)
  -> patched XNU -> bringup-launchd (pivot root to APFS, exec launchd)
```

- **Stage A (bringup)**: `fastboot boot <loader_image>`, root APFS on microSD. Fast iteration.
- **Stage B (permanent)**: loader in boot partition, kernel + ramdisk loaded from a separate partition (e.g., repurposed `recovery` or `system`), root APFS on eMMC (and/or microSD). 
  - **Partition layout (from fstab.qcom) — Android reference, NOT the real PIT**. The listed sizes are from an 8 GB reference but **a3lte has 16 GB eMMC** — the real PIT must be dumped before any writes: `heimdall print-pit --no-reboot` and record it in `docs/REPORT.md`.
    - `boot` (p16): 13,631,488 bytes (13 MiB) — holds lk2nd (512 KiB prefix) + loader (~300 KB). **Kernel and ramdisk do NOT fit here** — must be loaded from another partition.
    - `recovery` (p17): 15,728,640 bytes (15 MiB) — **too small for a 256 MB ramdisk** and repurposing it removes the stock recovery fallback. Do not use until payload size is known and Download mode is verified.
    - `system`: 1,568,669,696 bytes (1.5 GiB) — repurposable for APFS root or kernel+ramdisk
    - `userdata`: 10,537,124,096 bytes (10 GiB) — **Android reference only**; real size depends on 16 GB PIT
    - `cache`: 314,572,800 bytes (300 MiB)
    - `persist`: 33,554,432 bytes (32 MiB) — **must preserve** (calibration, Wi-Fi MAC, etc.)
    - `misc`: small
    - `efs`: **must preserve** (IMEI, radio calibration, MAC addresses)
    - `modemst1`, `modemst2`, `fsg`: **must preserve** (modem NVRAM)
    - `apnhlos`: modem firmware
    - `modem`: modem firmware
  - **Flash procedure**: Use Heimdall in Download mode (04e8:685d) for full partition writes. `fastboot flash boot` via lk2nd preserves its 512 KiB prefix — NOT suitable for full BOOT restore. Full BOOT rollback: `heimdall flash --BOOT <original_boot.img>`.
- Android partitions (`system`, `userdata`, `cache`, `recovery`) may be repurposed after backup, after Stage A is stable, and after real PIT is recorded.

### Boot-args (Core Prime SM-G360T1 — reference only)

```
debug=0x8 serial=0 rd=md0 cpus=4 loader_return_after=600
sdmmc=1 sdmmc_start=2048 sdmmc_sectors=249733120
fbscale=2 fb_hz=20 ramfs_mb=256 coreprime_input=1
```

| Arg | Meaning (from loader.c) |
|-----|-------------------------|
| `debug=0x8` | Enable serial console output (framebuffer + ramoops always on) |
| `serial=0` | Diagnostic serial mode (0 = loader console, 3 = kernel console in Core Prime build) |
| `rd=md0` | Root device is first ramdisk (md0) |
| `cpus=N` | Number of CPUs to start (max 4; loader enumerates cpu@0..3) |
| `loader_return_after=600` | Kernel returns to loader after N seconds (timed diagnostic return) |
| `sdmmc=1` | Enable read-only microSD SDHCI driver (virtio_blk backend) |
| `sdmmc_start=2048` | First sector of partition 1 (MBR partition start) |
| `sdmmc_sectors=249733120` | **Partition 1 size in 512-byte sectors** — compute per card (SK128: 249,737,216 physical sectors, MBR p1 at 2048 length 249,733,120) |
| `fbscale=N` | Framebuffer upscale factor (2 = 750x1624 logical on 480x800 panel) |
| `fb_hz=N` | Framebuffer target refresh rate |
| `ramfs_mb=N` | Ramfs size cap in MB (capped at max_mem/2 in ramfs.c) |
| `coreprime_input=1` | Enable Core Prime touch + button driver (IST3038 + gpio-keys) |

- `sdmmc_sectors` **must be computed per microSD**, not reused. Formula: `(partition_end_sector - partition_start_sector)` from MBR.
- `debug=0x144` enables KDP on panic (remote kernel memory access): not used in bringup, never enable outside an isolated debug setup.
- **The above are Core Prime-specific.** Do not use `fbscale`, `fb_hz`, `coreprime_input` on a3lte until verified.

### Boot-args (a3lte SM-A300F — unverified, start from this baseline)

```
debug=0x8 serial=0 rd=md0 cpus=4 loader_return_after=600
sdmmc=1 sdmmc_start=2048 sdmmc_sectors=<COMPUTE_PER_CARD>
ramfs_mb=256
```

- `fbscale`, `fb_hz`, `coreprime_input` **removed** — display is 540×960 qHD MIPI DSI (Super AMOLED, Samsung AMS452EF01 / S6E88A0), not Core Prime's 480×800 parallel panel. Touch controller: not IST3038, exact chip unidentified (see Hardware table), GPIOs unverified.
- `sdmmc_sectors` **must be computed per microSD** from its MBR.
- Add `fb_addr=<addr> fb_width=540 fb_height=960 fb_stride=<stride>` once framebuffer is mapped and verified.
- Touchscreen controller and touchkeys will need separate boot-args or device tree entries for I2C address, GPIOs, compatible string — TBD from hardware bringup.

## Hardware (SM-A300F / a3lte)

| Item | Value | Source / Notes | Verified on a3lte |
|------|-------|----------------|-------------------|
| RAM | 1 GB LPDDR3 (single-channel, 533 MHz) | Device Reference | no |
| Display | 540 × 960 (qHD, 16:9), 4.5" Super AMOLED | Device Reference | no |
| Display Driver / Panel | Samsung AMS452EF01 / S6E88A0 (MIPI DSI) | Device Reference | no |
| eMMC | Internal, 16 GB, eMMC 4.5 | Device Reference | no |
| microSD | Slot 2 (sdhci@7864900) | fstab.qcom, phone.dts (Core Prime) | no |
| SoC | MSM8916 (Snapdragon 410) | Device Reference | partial (MIDR read confirms core type only) |
| CPU | 4x Cortex-A53 r0p0 @ 1.2 GHz (MIDR 0x410fd030) | Device Reference, REPORT.md (Core Prime cpu-id) | no |
| GPU | Qualcomm Adreno 306 @ 400 MHz (OpenGL ES 3.0) | Device Reference | no |
| Crypto Extensions | **NONE** (ISAR0=0x10000, AES=0, SHA1=0, SHA2=0) | REPORT.md (Core Prime) | no |
| GIC | GICv2 distributor 0x0b000000, CPU 0x0b002000 | REPORT.md (Core Prime) | no |
| Timer | Virtual timer PPI20 @ 19.2 MHz | REPORT.md (Core Prime) | no |
| Framebuffer | TBD — MIPI DSI panel, 540x960, format/stride/addr unknown | — | no |
| SDHCI microSD | 0x07864900 / core 0x07864000 | REPORT.md (Core Prime) | no |
| SDHCI eMMC | 0x07824900 / core 0x07824000 | REPORT.md (Core Prime) | no |
| PMIC | PM8916 via SPMI (spmi@200f000 pmic@0) | phone.dts (Core Prime) | no |
| PON (Power On) | PM8916 PON via SPMI: pwrkey (IRQ 8, linux,code=0x74), resin (IRQ 9, linux,code=0x72) | phone.dts (Core Prime) | no |
| Touchscreen Controller | **Not IST3038** (Core Prime uses `imagis,ist3038` at I2C 0x50 on BLSP1 QUP5 GPIO 18/19). a3lte: exact chip unidentified — Zinitix claimed but `tm2-touchkey` is keys driver only. I2C address/GPIOs/compatible: TBD from a3lte bringup | Core Prime phone.dts (ist3038); a3lte init.target.rc shows `/sys/class/sec/sec_touchkey/`; check mainline `msm8916-samsung-a3u` dts | no |
| Touch Button Controller | Integrated in `sec_touchkey` / `tm2-touchkey` (capacitive keys, not separate gpio-keys). Brightness via `/sys/class/sec/sec_touchkey/brightness` | init.target.rc, lights.c | no |
| Charge Controller / Fuel Gauge | Richtek RT5033 | Device Reference | no |
| USB / MUIC | Silicon Mitus SM5502 (supports service UART) | Device Reference | no |
| Wireless (Wi-Fi/BT/FM) | Qualcomm WCNSS subsystem (WCN36xx transceiver) | Device Reference | no |
| LED Flash Driver | SG Micro SGM3140 (charge pump) | Device Reference | no |
| Vibration Motor | PWM-controlled (`pwm-vibrator`) | Device Reference | no |
| Accelerometer / Compass / Thermometer | Bosch BMC150 (`bmc150_accel`, `bmc150_magn`) | Device Reference | no |
| Proximity / Ambient Light / RGB | Capella CM36652 | Device Reference | no |
| Hall Effect Sensor | GPIO Hall Effect Sensor | Device Reference | no |
| Rear Camera | 8 MP, IMX219, LED flash (SGM3140) | Device Reference | no |
| Front Camera | 5 MP, S.LSI S5K5E3YX | Device Reference | no |
| PSCI | **Absent** in stock TrustZone | REPORT.md, loader.c (Core Prime) | no |
| CPU bringup | lk2nd spin-table for CPU1 (shared release word @ 0x86700088); loader powers CPU2/3 via APCS ACC (0x0b088000 + 0x10000*n) and routes them to loader holding pen (`cp_pen_slot[Aff0]`) | REPORT.md, loader.c `spin_table_release()`, `apcs_power_cpu()` | no |

*"Verified" = confirmed by a read on a3lte. Rows sourced from Core Prime (GIC, timer, SDHCI, PSCI, CPU bringup addresses incl. `0x86700088`) are assumptions.*

**Key differences from Core Prime (SM-G360T1) that need a3lte verification:**
- Display: 540x960 MIPI DSI (Super AMOLED) vs 480x800 parallel → framebuffer address, stride, panel timings, DSI controller config differ
- Touch controller: **not IST3038**; exact chip unidentified (Zinitix claimed, but `tm2-touchkey` is a keys driver) — different I2C address, GPIOs, compatible string
- Touch buttons: integrated in `tm2-touchkey` (not separate gpio-keys)
- eMMC: **16 GB** (not 8 GB) — partition layout must be re-dumped via `heimdall print-pit`
- Additional peripherals: RT5033 (charging), SM5502 (MUIC/UART), WCN36xx (Wi-Fi/BT), SGM3140 (flash), BMC150/CM36652 (sensors), PWM vibrator
- No device tree (.dts/.dtb) for a3lte in this repo — use Android device tree from `msm8916-common` as reference
- Check `android_device_samsung_a3lte/overlay/` and `init/init_a3.cpp` for a3lte-specific GPIO/regulator config

## Crypto Emulation

A53 lacks AES/SHA/PMULL; XNU corecrypto uses them unconditionally. Emulation lives in `ios-16-SM-G360T1/hakios-patches/new-files/coreprime_crypto_emul.h` (built against **iOS 16 / xnu-7195** source), hooked from `osfmk/arm64/sleh.c` via `coreprime_emulate_user_crypto()` in `handle_uncategorized`.

- Implements: AESE/AESD/AESMC/AESIMC, SHA1C/P/M/SU0/H/SU1, SHA256H/H2/SU0/SU1, PMULL/PMULL2 (1Q)
- Host KAT: `loader/test-crypto-emul.c` — 9/9 pass (FIPS-197 AES-128, SHA-1, SHA-256, PMULL)
- **Verified on hardware**: First Apple `/bin/df` completed successfully with emulation active (REPORT.md, Core Prime)

### Porting to xnu-4903 (iOS 12.4)

1. Copy `coreprime_crypto_emul.h` into `xnu-4903/osfmk/arm64/` (or a new `hakios/` subdirectory).
2. In `xnu-4903/osfmk/arm64/sleh.c`, locate `handle_uncategorized` (or equivalent exception handler) and add the call to `coreprime_emulate_user_crypto()` before the default `EXC_BAD_INSTRUCTION` path.
3. Verify syndrome decoding against xnu-4903's own `ESR_EC_*` definitions (names differ from Linux `ESR_ELx_EC_*`); do not assume iOS 16 and iOS 12.4 match. `sleh.c` differs substantially between xnu-7195 and xnu-4903 — port by hand.
4. Re-run host KAT (`loader/test-crypto-emul.c`) against the xnu-4903 build.
5. **Known gaps**: the hook covers user-mode only, while kernel corecrypto also uses these instructions unconditionally (would panic). Prefer first clearing crypto capability bits in commpage/`hw.optional` so userland avoids them. Table-based AES emulation is not constant-time (side-channel risk).
6. **TODO**: Test on a3lte hardware — unverified.

## APFS Storage

- Stage A: read-only microSD, multi-block CMD18 (~11 MB/s vs ~1.6 MB/s single-sector PIO).
- Root mounted by `bringup-launchd.c` via pivot_root (or chroot + `kern.hakios_switch_root`).
- eMMC driver for Stage B: **TODO** — requires retained-state init (lk2nd deselects eMMC with CMD7, sleeps with CMD5, disables HC mode; loader must re-init clock/power/registers).
- microSD model in testing: SK128, 249,737,216 physical sectors, MBR p1 at 2048 length 249,733,120.

## Working Rules

### XNU changes
- Prefer `CONFIG_*` over platform macros. Use `KERNEL_PRIVATE`, `BSD_KERNEL_PRIVATE`, `MACH_KERNEL_PRIVATE`, `XNU_KERNEL_PRIVATE` for visibility.
- Run `make checkstyle` before committing.
- Patch base: `HAKIOS_BASE_COMMIT=96ffbc532e970f3540885755d457e401e04d4707` belongs to the **xnu-7195 (Core Prime)** patch set, not xnu-4903. Record a separate base commit and patch file for xnu-4903 and regenerate it after kernel changes.
  - xnu-4903 current HEAD: `494ffe1 xnu-4903.270.47`
  - Will create `HAKIOS_BASE_COMMIT_A3LTE` for a3lte-specific patches

### Loader/shim changes
- Verify hardware values against a3lte, not Core Prime.
- Log hardware results in `docs/REPORT.md`.

## Verification

- Software: XNU ARM64 build succeeds, loader builds, `make checkstyle` passes.
- Device: serial log reaches launchd; record result, date and image hash in `docs/REPORT.md`.
- Report failures as they are. Do not claim "boots" without serial evidence.

## Key Files

| File | Purpose |
|------|---------|
| `xnu-4903/README.md` | XNU build system, style, debugging |
| `ios-16-SM-G360T1/loader/loader.c` | Loader logic (reference) |
| `ios-16-SM-G360T1/loader/bringup-launchd.c` | pivot_root + launchd |
| `ios-16-SM-G360T1/loader/build.sh` | Loader build command + toolchain |
| `ios-16-SM-G360T1/shim/phone.dts` | Core Prime device tree (reference) |
| `ios-16-SM-G360T1/shim/diag.c` | FDT parse, framebuffer, ramoops, PSCI |
| `ios-16-SM-G360T1/hakios-patches/HAKIOS_BASE_COMMIT` | Base commit hash |
| `ios-16-SM-G360T1/hakios-patches/hakios-tracked-changes.patch` | Kernel diff |
| `ios-16-SM-G360T1/hakios-patches/new-files/coreprime_crypto_emul.h` | Crypto emulation (port to xnu-4903) |
| `ios-16-SM-G360T1/hakios-patches/new-files/hakios.coreprime_input.c` | Touch + button driver (Core Prime IST3038 — **not used on a3lte**) |
| `/home/laptop/Documents/Codex/2026-11-03/ho/work/hakios/xnuloader/mkpayload.py` | Payload builder (XNUP) — **external, not in repo** |
| `android_device_samsung_a3lte/BoardConfig.mk` | Hardware reference (partition sizes, **eMMC size wrong: says 8 GB, actual 16 GB**) |
| `android_device_samsung_a3lte/rootdir/etc/fstab.qcom` | eMMC partition layout (Android reference, not real PIT) |
| `android_device_samsung_a3lte/init/init_a3.cpp` | a3lte variant detection |
| `android_device_samsung_a3lte/overlay/` | a3lte-specific GPIO, regulator, pinmux config — check for touch, sensors, PMIC |
| `ios-16-SM-G360T1/docs/REPORT.md` | Hardware bringup log (authoritative for Core Prime only; a3lte entries must be added) |
| `ios-16-SM-G360T1/docs/HANDOFF-start-of-session.md` | Latest session state |
| `docs/REPORT.md` | **This repo's a3lte bringup log** (created 2026-11-21) |

## Licensing

XNU is under APSL 2.0. Apple and Samsung proprietary files are not redistributed.
