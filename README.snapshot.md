# kernel_xiaomi_raphael-4.14 (ban-sky snapshot)

Redmi K20 Pro / Mi 9T Pro（代号 **raphael**，SM8150）的 4.14 第三方内核，来源：

- 上游：<https://github.com/HeliumStudio-Dev/kernel_xiaomi_raphael>
- 分支：`zundamon-evo-public`，快照提交 `5c85421 treewide: Zundamon-Evo kernel changes for raphael`
- 本仓库为**单分支快照**（不是 fork），保留 `upstream` 远端。

## 为什么 4.14 这条线重要

目前跑澎湃 OS 4 移植包的 K20 Pro 实测内核就是 **`4.14.357-Zundamon-Evo-v20260827-1`**（见 dsshhgg/raphael-nfc-fix 的 README：`OS4.0.0.16.XPCCNXM` / `CP2A.260605.016` / Android 17）。也就是说：想改 OS4 上的行为，4.14 才是"真·适配"的那条线；5.4 是另一条面向 AOSP/OSS vendor 的线。

布局与 5.4 差别很大：显示是 `drivers/gpu/drm/msm/{sde,dsi-staging}`，视频是 `drivers/media/platform/msm/vidc`（还有未编译的 `vidc_3x`），NFC 是老驱动 `drivers/nfc/nq-nci.c`（`CONFIG_NFC_NQ=y`）。

## 本快照做的修复

1. **NFC 固件下载帧被截断**（高）
   - `drivers/nfc/nq-nci.c`：`MAX_BUFFER_SIZE` 320 → 558。旧头文件注释写明 558 = `Frame len(2)+Header(6)+DATA(512)+HASH(32)+CRC(2)+RFU(4)`，是 FW download 模式 554 字节帧的尺寸；320 会让 HAL 的 554 字节读被静默截断、写直接 `-ENOMEM`。
2. **NFC 中断触发方式**（很可能）
   - 同文件 `IRQF_TRIGGER_RISING` → `IRQF_TRIGGER_HIGH`：DT 里 INT 是 `interrupts = <47 0>` + pinctrl 上拉，电平有效；上升沿会在 handler 屏蔽中断期间丢包。
3. **QC3 18W 不上报**（确定）
   - `drivers/power/supply/qcom/smb5-lib.c`：class A（18W）分支补 `power_supply_changed(chg->usb_psy)`，和 class B（27W）保持一致；之前 `HVDCP3_TYPE` 停在 NONE。
4. **FG 关机延迟用未初始化电压**（确定/低危）
   - `drivers/power/supply/qcom/qpnp-fg-gen4.c`：`int vbatt_uv = 0;`，并且只有 `!rc` 时才判断 `vbatt_uv/1000 > SHUTDOWN_DELAY_VOL`。
5. **`reboot edl` 失效**（确定）
   - `drivers/power/reset/msm-poweroff.c`：恢复 `if (!strncmp(cmd, "edl", 3)) enable_emergency_dload_mode();`（原来被改成 `if (0)`），defconfig 补 `CONFIG_QCOM_DLOAD_MODE=y`，否则整块 dload 代码变 stub。
6. **背光映射空指针/除零**（高）
   - `drivers/gpu/drm/msm/sde/sde_connector.c`：`if (!display || !display->panel) return -ENODEV;`；`bl_max_level <= 1 || bl_range <= 0` 时直接用 `brightness`，否则 `bl_min + DIV_ROUND_CLOSEST((brightness - 1) * bl_range, bl_max_level - 1)`。
7. **ESD IRQ 不平衡 + 裸改 istate**（中)
   - `drivers/gpu/drm/msm/dsi-staging/dsi_panel.c`：`irq_to_desc()` 结果判空；disable 前检查 `irqd_irq_disabled()`，避免重复 `disable_irq_nosync()` 把计数压到 0 以下导致 IRQ 永久关闭。

## CI 构建链（这段是踩坑记录）

`.github/workflows/build-kernel.yml`，ubuntu-22.04 + GCC 交叉工具链（`gcc-aarch64-linux-gnu` + `gcc-arm-linux-gnueabi` 给 compat vDSO），`make O=out ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- CROSS_COMPILE_COMPAT=arm-linux-gnueabi- raphael_defconfig && make ... Image dtbs`。

为让这棵老树在 22.04 上编过，额外打了三个构建兼容补丁（与功能无关）：

- `arch/arm64/Makefile` 的 `CROSS_COMPILE_COMPAT` 检查（未改源码）：CI 显式传 `CROSS_COMPILE_COMPAT=arm-linux-gnueabi-`，否则 4.14 直接 `*** CROSS_COMPILE_COMPAT not defined` 停编；
- `scripts/dtc/Makefile` 支持 `NO_YAML=1`：22.04 的 libyaml 链接方式让 host `dtc` 报 `undefined reference to yaml_*`；
- `include/linux/compiler_types.h` 给 `__has_attribute()` 加回退：vDSO 的 `.S`/`.lds` 预处理不认这个内建，报 `missing binary operator before token "("`。

构建失败时 workflow 会把日志推到 `ci-logs` 分支（`ci: build failure log (run <id>)`），方便匿名查看；绿色运行则直接下载 artifact `raphael-4.14-image`（`Image`、`Image.gz`、`*.dtb`、`*.dtbo`、config）。

构建结果：**run 44987507044（commit `89c602c58`）success**，耗时约 10.5 分钟，artifact `raphael-4.14-image`（14.75 MB）。

## 构建验证的边界

CI 只证明能编过；NFC 大帧/`reboot edl`/QC3 18W 上报/背光这些修复的实机行为仍需真机验证（本仓库没有本机编译，也没有真机日志）。

## 已知未修 / 待验证

- **`nq-nci` 删掉了 `.compat_ioctl`**：32 位 NFC HAL 调 `NFC_SET_PWR`/`ESE_SET_PWR`/`ESE_GET_PWR` 会 `-ENOTTY`；全 64 位环境无影响。
- **`SCM_DLOAD_MINIDUMP` 0x20 → 0x80**：这块 TZ 的 minidump 位是否真是 0x80，树内无法证实（5.4 没有这个文件）；位错的话 panic dump 类型会全错。
- **pstore 文件名互换**：`fs/pstore/inode.c:340-348` 把 DMESG 记录命名为 `console-ramoops`、CONSOLE 记录命名为 `dmesg-ramoops`；按 AOSP 约定读 `console-ramoops` 会拿到 panic 记录（是否 Xiaomi 有意，待验证）。
- **ESD IRQ 的根因**（`dsi_drm.c` seamless enable 早返回跳过 enable）只在驱动层做了防御，没有改调用时序。
- **背光除零**：本机 DT 是 2047/2/2047，不会触发；改了只是防御。
- **SIM/「卡二」**：内核无 SIM/UIM 代码，`pil_modem` 与 glink 完整；「卡二」指卡在第二个开机画面/开机动画，不是 SIM 卡二，怀疑在 NON-HLOS/基带固件、qcril/HAL 或移植包本身，需 radio log。
- SELinux 的 `androidboot.selinux=` 接管（`security/selinux/selinuxfs.c:173-175`）只影响显式传 permissive 的启动，默认 enforcing 不受影响，故未改。

## 刷机提醒

- 4.14 与澎湃 OS 4 移植包同代，但仍需与 ROM 的 vendor 匹配；刷前备份 `boot`/`dtbo`。
- 确认版本：`magiskboot unpack boot.img` 或 `unpack_bootimg --boot_img boot.img`。
- 出问题先看 `console-ramoops`（ramoops 区域 0xa1600000，8MB）。
