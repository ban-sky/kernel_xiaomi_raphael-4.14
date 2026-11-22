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
8. **seamless 模式下 ESD 保护失效**（中）
   - `drivers/gpu/drm/msm/dsi-staging/dsi_drm.c`：`dsi_bridge_disable()` 对 seamless/VRR/dyn-clk 模式提前返回，和 enable 路径对称；否则 enable 跳过、disable 照样关 ESD IRQ。
9. **DC 调光把低亮度锁死**（rc2，待真机确认）
   - `drivers/gpu/drm/msm/dsi-staging/dsi_panel.c`：DC 调光（`DISPPARAM_DC_ON` / `DISPPARAM_DC_LAYER_ON`）打开后，低于 `dc_threshold`（raphael 是 610/2047）的背光请求会被 **整条丢弃**（`skip set backlight bacase dc enable`），或**钉死在 dc_threshold**。Xiaomi 原本靠显示处理器在底层压暗画面；部分第三方 ROM（如移植的澎湃 OS 3）只打开了 DC 图层却没做画面压暗，于是亮度条在低段完全没反应、屏幕停在 ~30%。
   - 现在改成：DC 地板默认关闭，背光始终按请求值下发（DC 图层命令 `DIMMINGON/OFF` 仍照发）。需要恢复 Xiaomi 老行为时，给面板节点加 `qcom,mdss-dsi-panel-dc-floor-enable;` 即可。
10. **最低亮度接近全黑**（rc3）
   - `arch/arm64/boot/dts/qcom/xiaomi-sm8150-common.dtsi` 的 `qcom,mdss-dsi-bl-min-level` 从 `<2>` 提到 `<60>`（2047 的约 3%）。
   - `drivers/gpu/drm/msm/dsi-staging/dsi_panel.c` 的 `dsi_panel_set_backlight()` 对任何非零 `bl_temp` 钳制到 `bl_min_level`；`0` 仍是 `0`，关屏/AOD 不受影响。觉得还暗就继续调这个 60。

11. **DC 调光开关没有实际效果**（rc5，待真机确认）
   - 根因：ROM 的 DC 开关走 `disp_param` 的 `DISPPARAM_DC_ON/OFF`，原来只置 `panel->dc_enable`，而真正驱动 SDE dim layer 的是 `panel->dc_dimming_enabled`（由 `DISPPARAM_DC_LAYER_ON/OFF` 置位）。`dc_enable` 在 rc2 之后就没人消费，所以开关毫无反应。
   - rc5 起 `DISPPARAM_DC_ON/OFF` 同时置/清 `dc_dimming_enabled`，DC 一开就按 `dc_panel_alpha.h` 的 LUT 叠加黑色 dim layer（亮度 < 610 时生效，610 及以上 alpha=0）。
   - 也可以手动验证：`echo 1 > /sys/class/drm/card0-DSI-1/dc_dimming`，再切亮度看是否变暗；`cat` 同一个节点看状态。
12. **NFC 无法识别时便于定位**（rc5，诊断）
   - `drivers/nfc/nq-nci.c` 在 `NFC_SET_PWR` 上电成功后打一行：`nq-nci: powered on ven=.. firm=.. irq=.. clk_vote=.. chip=0x..`。
   - 若 `ven=0` 说明 VEN 没拉起来（电源/GPIO 问题）；`irq=1` 长时间不变说明芯片没发中断（可能没上电/固件没跑）；`chip=0x0` 是本树把 `nfcc_hw_check` 关掉（原厂注释：避免阻塞 FW download）导致的，管理器/HAL 若依赖 `NFCC_GET_INFO` 需要这一项。

## 版本与产物

- 版本串：`CONFIG_LOCALVERSION="-Zundamon-v1.0-rc5"`（`1.0` 前带 `v`）且去掉了 `Makefile` 里的 `EXTRAVERSION = -openela`，`uname -r` 为 **`4.14.357-Zundamon-v1.0-rc5`**。
- 每次 CI 出两种口味：
  - `Zundamon-4.14.357-v1.0-rc5.zip`——标准版；
  - `Zundamon-4.14.357-v1.0-rc5-docker.zip`——同内核 + Docker/容器支持；
  - 都是 AnyKernel3，只替换 boot 内 kernel，不含 dtbo.img，同目录有 `.sha256`。
- Docker 版是在 `raphael_defconfig` 上再叠 `arch/arm64/configs/raphael_docker.config`：UTS/IPC/PID/NET/USER 命名空间、cgroup pids/device、veth/macvlan/vxlan/bridge + iptables NAT、overlayfs、seccomp filter。标准版不启用这些（尤其 `USER_NS` 会扩大攻击面），两个包互不影响。
- 命名规则 `Zundamon-<内核版本>-<版本号>[-docker]`，由 workflow 顶部 `KNAME/KV` 控制。

## Releases 与 root 管理器

- `main` 分支构建成功后，workflow 的 `release` job 用 Actions 自带的 `GITHUB_TOKEN` 自动创建/更新 **prerelease** `v1.0-rc5-4.14`，资产包括：
  - `Zundamon-4.14.357-v1.0-rc5.zip` / `Zundamon-4.14.357-v1.0-rc5-docker.zip` + `.sha256`；
  - `KernelSU-v3.3.1_32680.apk`——唯一匹配的管理器。
- 这棵树自带的是 **backslashxx/KernelSU v3.3.1**（KernelSU-Next 的 fork）：`drivers/staging/kernelsu/Makefile:11` `KSU_VERSION=32680`，`include/uapi/app_profile.h:4` `KSU_APP_PROFILE_VER 4`，包名 `me.weishu.kernelsu`。
- 所以管理器必须用 backslashxx 的 **build 32680** APK：`Natives.kt:27` `MINIMAL_SUPPORTED_KERNEL=32513`，app-profile ABI 为 v4（`uapi/app_profile.h:6`），包名一致。
- KernelSU-Next 官方 3.x 管理器全都对不上：v3.1.0 过得了版本门（32310）但 app-profile ABI 是 **v2**，添加超级用户时内核 `ksu_set_app_profile()` 直接 `-EINVAL`，界面就是「**更新应用配置失败**」；v3.2.0 是 ABI v3 且要求内核 33110；v3.4.x 是 ABI v4 但要求 33188。**不要装 KernelSU-Next 3.x。**
- v2 签名白名单在 `manager/apk_sign.c:322`，常量在 `drivers/staging/kernelsu/Makefile:13-15`：现在为 `me.weishu.kernelsu` / `0x0363`（867 字节）/ `4359c171…`，正好是 backslashxx 32680 APK 的证书。换管理器必须同步改这三个值，否则报「无法在内核找到v2签名」（长度不等时也会报这句，且 `dmesg` 里看不到 `sha256:` 行）。
- 刷机包内置 `KernelSU.apk`，`anykernel.sh` 刷完会拷到 `/sdcard/Download/Zundamon/KernelSU.apk`。recovery 里无法静默 `pm install`，重启后手动点安装，再打开管理器授权即可。

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
- 包是 AnyKernel3：TWRP/OrangeFox 直接刷 zip，只替换 boot 里的 kernel。**不含 `dtbo.img`**（boot 的 dtb 要和 ROM 的 dtbo 同代，覆盖 ROM dtbo 风险大），dtbo 镜像单独放在 `raphael-4.14-image` artifact。打包后 CI 会 `unzip -l` 校验 `Image`/`anykernel.sh`/`tools/ak3-core.sh`，缺一个即失败。
