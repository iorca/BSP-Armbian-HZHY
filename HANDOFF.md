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

## 六、下一步建议

1. 先看 CI 日志。若 U-Boot 阶段失败 → 按风险 2 处理（加 hook 调 `make.sh`）。
2. 若内核配置不对 → 核实 `LINUXCONFIG` 变量名（查 Armbian `config/sources/families/` 下 rk35xx 相关文件）。
3. 构建成功后 → 烧录验证串口登录（`ttyFIQ0:1500000`），再看 HDMI。
4. 阶段二再动 MIPI 屏。
