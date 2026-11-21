# iOS 12.4 on SM-A300F (a3lte) - Hardware Bringup Report

## Session Start: 2026-11-21

### Device State
- Device: Samsung Galaxy A3 (2015) SM-A300F / a3lte
- Current mode: **fastboot** (18d1:d00d) - lk2nd confirmed
- Target: Download mode (04e8:685d) for PIT dump and partition backup

### Task Status

#### 1. Find mkpayload.py path
**Status:** LOCATED (external workspace)
- Referenced in build scripts as `../hakios/xnuloader/mkpayload.py`
- Actual location: `/home/laptop/Documents/Codex/2026-11-03/ho/work/hakios/xnuloader/mkpayload.py` (per REPORT.md from Core Prime work)
- **NOT in this repository** - must be copied or reimplemented for a3lte loader build

#### 2. Dump real PIT via heimdall
**Status:** PENDING - Device not in Download mode
- Required: Put device in Download mode (Vol Down + Home + Power)
- Command: `heimdall print-pit --no-reboot`
- Must record in this REPORT.md before any partition writes
- a3lte has **16 GB eMMC** (not 8 GB as in BoardConfig.mk)

#### 3. Determine lk2nd location (boot vs aboot)
**Status:** PENDING - Requires real PIT
- CLAUDE.md notes inconsistency: "lk2nd location (`boot` vs `aboot`) is inconsistent in this file"
- Writing `boot` partition overwrites lk2nd - treat as boot-chain write
- Must confirm from PIT before any write

#### 4. Identify touchscreen from android_device_samsung_a3lte
**Status:** PARTIAL - Core Prime reference only
From `ios-16-SM-G360T1/shim/phone.dts` (Core Prime SM-G360T1):
```
i2c@78b9000 (BLSP1 QUP5, GPIO 18/19)
  touchscreen@50:
    compatible = "imagis,ist3038"
    reg = <0x50>
    interrupts-extended = <&pinctrl 0x0d 0x02>  // GPIO 13
    touchscreen-size-x = <0x1e0>  // 480
    touchscreen-size-y = <0x320>  // 800
    vdd-supply = <&vdd_tsp_a>     // GPIO 75 (TSP_EN)
```

From `android_device_samsung_a3lte/overlay/` and `init.target.rc`:
- `/sys/class/sec/sec_touchkey/brightness` exists → Samsung `sec_touchkey` driver
- Lights HAL controls `sec_touchkey` brightness
- **a3lte touch controller NOT IST3038** - exact chip unidentified (Zinitix claimed but `tm2-touchkey` is keys driver)
- I2C address, GPIOs, compatible string: **TBD from a3lte hardware bringup**

Key differences from Core Prime:
- Display: 540×960 MIPI DSI (Super AMOLED) vs 480×800 parallel
- Touch controller: different chip, different I2C/GPIOs
- Touch buttons: integrated in `tm2-touchkey` (not separate gpio-keys)

#### 5. Verify GIC, timer, SDHCI, PSCI, crypto on a3lte
**Status:** PENDING - Requires hardware access

From Core Prime (SM-G360T1) REPORT.md - **assumptions for a3lte**:

| Component | Core Prime Value | Verified on a3lte |
|-----------|------------------|-------------------|
| GIC | GICv2 distributor 0x0b000000, CPU 0x0b002000 | no |
| Timer | Virtual timer PPI20 @ 19.2 MHz | no |
| SDHCI microSD | 0x07864900 / core 0x07864000 | no |
| SDHCI eMMC | 0x07824900 / core 0x07824000 | no |
| PSCI | **Absent** in stock TrustZone | no |
| Crypto Extensions | **NONE** (ISAR0=0x10000, AES=0, SHA1=0, SHA2=0) | no |
| CPU MIDR | 0x410fd030 (Cortex-A53 r0p0) | no |

**Need to verify on a3lte hardware:**
- MIDR/ISAR0/ISAR1/PFR0 via CPUID
- GIC register layout
- Timer frequency and PPI mapping
- SDHCI register bases
- PSCI availability (SMC calls)
- Crypto instruction trapping

#### 6. Create base commit and patch for xnu-4903
**Status:** READY TO START
- Current xnu-4903 HEAD: `494ffe1 xnu-4903.270.47`
- Need separate base commit from Core Prime's `96ffbc532e970f3540885755d457e401e04d4707` (xnu-7195)
- Will create `HAKIOS_BASE_COMMIT_A3LTE` and generate patch after changes

### Next Actions Required

1. **User action**: Put device in Download mode (Vol Down + Home + Power)
2. **Run**: `heimdall print-pit --no-reboot` and record output
3. **Backup**: Full eMMC partition backup to `~/a3lte-backup/` with sha256 verification
4. **Verify**: lk2nd partition location from PIT
5. **Hardware bringup**: Run CPUID, memory map, register probes on a3lte
6. **Create a3lte loader directory** (doesn't exist yet - only Core Prime reference)
7. **Port crypto emulation** to xnu-4903 (`coreprime_crypto_emul.h` → `sleh.c` hook)
8. **Generate xnu-4903 base commit and patch**

### Safety Reminders
- **No writes** until backup verified and Download mode confirmed working
- Never overwrite `sbl1`, `rpm`, `tz`, `aboot` (lk2nd) without verified backup AND recovery path
- EDL mode (05c6:9008) **unverified on SM-A300F** - do not rely on it
- Prefer `fastboot boot` for iteration; flash only verified working images
- Log all commands with target partition and image hash in this REPORT.md

### Reference: Core Prime Boot-args (DO NOT USE DIRECTLY ON A3LTE)
```
debug=0x8 serial=0 rd=md0 cpus=4 loader_return_after=600
sdmmc=1 sdmmc_start=2048 sdmmc_sectors=249733120
fbscale=2 fb_hz=20 ramfs_mb=256 coreprime_input=1
```

### a3lte Baseline Boot-args (unverified)
```
debug=0x8 serial=0 rd=md0 cpus=4 loader_return_after=600
sdmmc=1 sdmmc_start=2048 sdmmc_sectors=<COMPUTE_PER_CARD>
ramfs_mb=256
```
- Add `fb_addr=<addr> fb_width=540 fb_height=960 fb_stride=<stride>` once framebuffer mapped
- Touchscreen/touchkey args TBD from hardware bringup