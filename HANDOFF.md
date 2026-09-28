# HZ-EVM-RK3588 Armbian BSP 镜像 — 阶段一交接

**目标**：用厂家 BSP 内核（Linux 6.1）构建可启动的 Armbian 镜像。
**阶段一范围**：MIPI / DSI / 背光 PWM / LCD_RESX / 触摸(sec,sec_ts I2C2) **全部不碰**，板级 dts 保持厂家 SDK 原样。

---

## 一、仓库结构

仓库：`https://github.com/iorca/BSP-Armbian-HZHY`

| 分支 | 内容 | Commit |
|------|------|--------|
| `kernel` | 厂家 BSP 内核 6.1.99（清理后 1.6G） | `564ee2a` |
| `u-boot` | 厂家 U-Boot 2017.09（HZ_EVM_RK3588_v1.0.0_20250515） | `2df5897` |
| `rkbin` | Rockchip DDR/BL31/BL32 blob（65M） | `25959be` |
| `main` | Armbian 配置 + CI workflow | `dbe8891` |

Release：`toolchain-v1` → `HZ-EVM-RK3588-Linux-GCC.tar.gz`（263M，`aarch64-buildroot-linux-gnu`）

---

## 二、已验证的 SDK 关键事实（来自源码原文，非推断）

### 内核
- 路径：`SDK/kernel-6.1`，版本 **6.1.99**
- 板级 dts：`arch/arm64/boot/dts/rockchip/HZ-EVM-RK3588.dts`
- defconfig：`arch/arm64/configs/HZ-EVM-RK3588_defconfig`

### 板级 lunch 配置
`device/rockchip/.chips/rk3588/rockchip_HZ-EVM-RK3588_defconfig`（lunch 选项 3）：
```
RK_KERNEL_DTS_NAME="HZ-EVM-RK3588"
RK_USE_FIT_IMG=y
RK_KERNEL_CFG="HZ-EVM-RK3588_defconfig"
```
> lunch 选项 2 是 `rockchip_HZ-EVM-RK3588_Ubuntu_defconfig`（Ubuntu 桌面版）。
> Armbian 做 minimal server 镜像，**应对齐选项 3**，不用 Ubuntu 版。

### U-Boot defconfig 推导链（已验证，非猜测）
`RK_UBOOT_CFG` 未在板级/芯片级 defconfig 中显式设置，走 Kconfig 默认值。
`device/rockchip/common/configs/Config.in.loader`：
```
default RK_CHIP_FAMILY if RK_CHIP_FAMILY = "rk3308"
    || RK_CHIP_FAMILY = "rk3288" || RK_CHIP_FAMILY = "rk3588"
```
RK3588 命中 → `RK_UBOOT_CFG` = `rk3588` → `make.sh rk3588` → `configs/rk3588_defconfig`。

**结论：board config 里 `BOOTCONFIG="rk3588_defconfig"` 正确。**

### rkbin 打包参数（`u-boot/make.sh` 365 行）
```
INI_LOADER = rkbin/RKBOOT/${RKCHIP_LOADER}MINIALL.ini
```
- Loader ini：`RKBOOT/RK3588MINIALL.ini`
  - DDR：`rk3588_ddr_lp4_2112MHz_lp5_2400MHz_v1.21.bin`
  - SPL：`rk3588_spl_v1.13.bin`（**SPL 方案，非传统 MiniLoader**）
  - 输出：`rk3588_spl_loader_v1.18.113.bin`
- Trust ini：`RKTRUST/RK3588TRUST.ini`
  - BL31：`rk3588_bl31_v1.53.elf`
  - BL32：`rk3588_bl32_v1.19.bin`

### 编译方式（厂家手册）
```
./build.sh lunch        # 选 3（rockchip_HZ-EVM-RK3588_defconfig）
./build.sh              # 全编译
./build.sh uboot        # 单独编 U-Boot
./build.sh kernel       # 单独编内核
```
产物在 `rockdev/` 下，`update.img` 为完整镜像。要求 Ubuntu 主机 ≥8G RAM，**完整重编约 2 小时**（i5-12400）。

---

## 三、Armbian 侧配置

### `userpatches/lib.config`
覆盖工具链与源码来源：
- `KERNEL_COMPILER` / `UBOOT_COMPILER` / `ATF_COMPILER` = `aarch64-buildroot-linux-gnu-`
  （Armbian 默认 `aarch64-linux-gnu-`，厂家工具链前缀不同，必须覆盖）
- `KERNELSOURCE`/`KERNELBRANCH` → `iorca/BSP-Armbian-HZHY` 的 `kernel` 分支
- `BOOTSOURCE`/`BOOTBRANCH` → 同仓库 `u-boot` 分支
  （否则 Armbian 会去拉默认的 armbian/linux-rockchip 与 Radxa U-Boot）
- 工具链解压到 `/opt/toolchain` 后追加 PATH

### `config/sources/boards/hz-evm-rk3588.conf`
```
BOARDFAMILY="rk35xx"
BOOTCONFIG="rk3588_defconfig"
LINUXCONFIG="HZ-EVM-RK3588_defconfig"   # 待确认变量名*
BOOT_FDT_FILE="rockchip/HZ-EVM-RK3588.dtb"
SERIALCON="ttyFIQ0:1500000"
```
\* `LINUXCONFIG` 变量名尚未在 Armbian 源码中核实——`HAS_` 需查证。若 Armbian 不识别，内核 defconfig 需通过其他机制指定。

### `.github/workflows/build.yml`
1. checkout `armbian/build@main`
2. 从 Release 下载工具链 → `/opt/toolchain`
3. 下载 `lib.config` + `hz-evm-rk3588.conf` 到对应目录
4. `./compile.sh BOARD=hz-evm-rk3588 BRANCH=vendor RELEASE=jammy BUILD_MINIMAL=yes`
5. 上传镜像 + build log artifact

不预 clone 源码（Armbian 的 `fetch_from_repo` 自己管理 `cache/sources/` 布局，猜目录名容易错）。

---

## 四、CI 状态

- 运行中：`https://github.com/iorca/BSP-Armbian-HZHY/actions`
- 最近 run：`36337905147`（push `dbe8891` 触发）
- 超时放宽到 360 分钟（1.6G kernel clone + 编译）

---

## 五、已知风险 / 待验证（按优先级）

1. **`LINUXCONFIG` 变量名未核实** — Armbian 是否用这个名字选内核 defconfig，需查证。
   若不对，内核会用默认 defconfig 而非 `HZ-EVM-RK3588_defconfig`。
2. **厂家 U-Boot 2017.09 用 `make.sh`（Rockchip 特有流程）**，不是标准 `make`。
   Armbian 默认按标准 U-Boot 流程构建，可能需要 hook 调用 `make.sh`。
   这是**最可能失败的点**。
3. **rkbin 路径** — Armbian 构建 U-Boot 时能否找到 DDR/BL31 blob，需看日志确认。
4. **SPL vs MiniLoader** — 厂家用 SPL 方案，Armbian rockchip64 的打包逻辑是否匹配，待验证。
5. 未做 SDK 基线编译（按要求跳过），若 CI 失败，无法快速区分是 SDK 本身问题还是 Armbian 集成问题。

---

## 六、CI 调试记录（已修 5 个问题）

| # | 现象 | 根因 | 修复 |
|---|------|------|------|
| 1 | run 显示 **success 但只跑 1m53s**（假绿灯） | `./compile.sh \| tee build.log` 的退出码取 `tee`（恒为 0） | 加 `set -o pipefail` |
| 2 | `error! stdin is not a terminal. can't use dialog`（exit 43） | `interactive_config_ask_kernel_configure()` 仅在 `KERNEL_CONFIGURE` 非空时跳过 | 加 `KERNEL_CONFIGURE=no` |
| 3 | `No such BOARD 'hz-evm-rk3588'` | board config 放在 `config/sources/boards/`（该目录 404 不存在） | 改到 **`config/boards/`** |
| 4 | `BRANCH='vendor' not valid ... KERNEL_TARGET='Image'` | `KERNEL_TARGET` 是**分支列表**不是镜像类型 | 改为 `KERNEL_TARGET="vendor"` |
| 5 | `KERNEL_ONLY is set. This is not supported anymore` | 新版 Armbian 改用**子命令 CLI**，`KERNEL_ONLY` 已废弃 | 用 `./compile.sh kernel BOARD=... BRANCH=vendor` |

### 关键认知：新版 Armbian 是子命令 CLI

```
./compile.sh build  BOARD=... BRANCH=...   # 完整镜像（需 rootfs/binfmt）
./compile.sh kernel BOARD=... BRANCH=...   # 只编内核+dtb（跳过 rootfs）
```
`kernel` 子命令可绕过 binfmt 阶段（`NEEDS_BINFMT` 只在 image/rootfs 构建时置 yes）。

## 七、当前阻塞：hash-files.sh:74

```
Error 1 occurred in SUBSHELL at /armbian/lib/functions/general/hash-files.sh:74
```

第 74 行是：
```bash
full_hash="$(cd "${SRC}" && sha256sum "${files_to_hash_sorted[@]}")"
```
`sha256sum` 返回 1 = 列表中至少一个文件读不到/不存在，或**参数列表超限**。

怀疑方向（未证实，需日志）：
1. Armbian 对整个 kernel 源码目录做文件 hash —— 我们的 kernel 分支 1.6G / 数万文件，
   一次性 `sha256sum` 会超过 ARG_MAX（约 2MB 命令行上限）。
2. 仍在套用 `KERNELPATCHDIR='rk35xx-vendor-6.1'`（来自 rockchip-rk3588 / rk35xx family）。
   **这些 patch 是针对 armbian/linux-rockchip 的，打到厂家 BSP 内核上必然冲突** ——
   我们换了 KERNELSOURCE，但 family 的 patch 目录没换，这是个设计问题。

### 下一步（需要真实日志才能定论）
1. 看完整 build.log（需 GitHub token 或开机 VM 用 `gh run view --log-failed`）。
2. 若确认是 patch 冲突 → **不要复用 rockchip-rk3588 vendor 分支**，改为自定义 family
   （干净指向厂家 BSP 源码，不带 Armbian 的 rk35xx patch）。
3. 若确认是 hash 参数超限 → 需看 Armbian 是否有跳过源码 hash 的开关。

> 注：CI 日志需认证才能下载，本地无 gh / 无 token，VM 已关机 —— 这是当前最大障碍。

## 八、下一步建议

1. **先解决 binfmt 阻塞**（第七节），这是当前唯一卡点，尚未进入真正的源码编译。
2. 之后最可能失败的是 **U-Boot 构建**：厂家 U-Boot 2017.09 用 `make.sh`（Rockchip 流程），
   Armbian 默认走标准 `make`，可能需 hook 调 `make.sh`。
3. 核实 `LINUXCONFIG` 变量名（board config 中用了它指定 `HZ-EVM-RK3588_defconfig`，未在 Armbian 源码核实）。
4. 构建成功后 → 烧录验证串口登录（`ttyFIQ0:1500000`），再看 HDMI。
5. 阶段二再动 MIPI 屏。
