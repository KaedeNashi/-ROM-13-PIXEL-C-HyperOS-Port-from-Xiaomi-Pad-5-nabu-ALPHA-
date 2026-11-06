# Pixel C — HyperOS Android 13 (Xiaomi Pad 5 / nabu) Porting Log

> **声明**
>
> 本移植项目全部使用 Cherry Studio 配合 DeepSeek V4.1 Flash 完成。系统目前仅在初始可用阶段，存在许多不确定因素和 BUG。本项目 system 已做过重签名修改，部分 BUG 有动手能力的用户可以自行解决。本人使用的机器已改装为 8GB DRAM，本项目在 3GB 设备上的流畅度不做保证。本项目无后期维护的可能性。

> Device: Google Pixel C (dragon / sphynx, NVIDIA Tegra X1 T210, 8 GB RAM mod)
> Base: HyperOS Android 13, Xiaomi Pad 5 (nabu), `OS1.0.3.0.TKXCNXM`
> Vendor: LineageOS 20 Pixel C vendor (T210)
> Images: `boot.img`, `vendor.img`, `system.img`
> Date: 2026-11-07

This document is split into two parts: **Part 1 — 中文**, then **Part 2 — English**. Commands, hashes and file paths are identical in both parts.

---

# Part 1 — 中文

## 0. 概要

本版本保留 Pixel C 硬件，使用小米平板 5 的 HyperOS 系统。重力感应/平板布局的修复分为两层：

1. **传感器数据层（vendor）**：CrosEC 内核模块 + NVIDIA sensor HAL + VINTF + SELinux，让加速度计、陀螺仪、磁力计真正出数据。
2. **MIUI 设备类型层（product）**：`ro.build.characteristics=tablet` 让 MIUI 使用 Pad 布局并允许自动旋转。

这两层缺一不可。只改 `ro.build.characteristics` 不会产生传感器数据；只修 vendor 传感器，MIUI 桌面仍可能套手机竖屏布局。

**不要把这些镜像刷到其他设备。**

## 1. 设备与镜像

| Image | Size (bytes) | SHA-256 |
|---|---:|---|
| `boot.img` (KERN-A, mmcblk0p1) | 33,554,432 | `280059679EAA3481165490CBE57853D25175FEF5585239616F5A62355352DE1E` |
| `vendor.img` (VNR, mmcblk0p5) | 318,767,104 | `60CABF265FAE374A24F31280DD26D6AC223E3AE97695A3E2C074FC6DC0446DF1` |
| `system.img` (APP, mmcblk0p4) | 8,589,934,592 | `9D0D524ABAEBDF9D297396CE9146EF681C53EE5EDF38B0D80F98D281D42B14F5` |

### 各分区里有什么

| 分区 | 内容 |
|---|---|
| `boot.img` | KERN-A 启动镜像/内核。 |
| `vendor.img` | CrosEC 传感器模块、NVIDIA sensor HAL、VINTF、SELinux；EMC 初始化行；vendor `pm.dexopt.*`；vendor WiFi supplicant 修复。 |
| `system.img` | `/system`、`/product`、`/system_ext`：ThemeManager/WallpaperT 补丁、百度输入法、Settings、overlay、WebView、删除 FindDevice、`ro.build.characteristics=tablet`、`ro.miui.notch=0`、system `pm.dexopt.*`、音频策略补丁。 |
| `/data` | 用户应用、`/data/app` 更新、设置、数据库、dex2oat 产物。格式化 data 会清空。 |

所以“所有修复都在 system”是不准确的：**传感器修复在 vendor，启动镜像在 boot，UI/APK 修复在 system**。只格式化 data 不会改变这三个分区；只要不重刷它们，当前修复都还在。

## 2. 重力感应/平板布局完整修复

### 2.1 症状

- 桌面像手机：竖屏、网格和边距不对。
- 屏幕不随重力旋转，或只有部分应用能转。
- 横屏顶部中间出现固定不动的黑色椭圆/“挖孔”。
- 如果 `dumpsys sensorservice` 里没有 `CrosEC Accelerometer`，说明传感器 HAL 没起来，先修第 1 层。

### 2.2 根因

**传感器层：** 原版 LOS20/LOS22.2 的 vendor 只有 `android.hardware.sensors::ISensors` 的 VINTF 字符串，没有 `cros_ec_sensors*.ko`，没有 `sensors.dragon.so`，也没有 NVIDIA sensor HAL 服务，所以传感器服务起不来。

**MIUI UI 层：** `ro.build.characteristics` 为空或不是 `tablet` 时，`MiuiHomePadS` 通过 `miui.os.Build.IS_TABLET` 判断成手机，启动器套手机布局、锁竖屏，并按手机逻辑画圆角/挖孔。Pixel C 原生面板是 2560×1800 横屏，MIUI 的竖屏坐标假设会同时导致“重力感应不工作”的假象。

### 2.3 第 1 层：Vendor 传感器栈

Pixel C 的加速度计/陀螺仪/磁力计挂在 ChromeOS EC 上。当前 HyperOS 的 `vendor.img` 已经包含下面整套文件；如果你要在 LOS20/LOS22.2 或干净的 vendor 上复现，按以下步骤做。

**准备：**

- 与启动镜像同一棵 **kernel 4.9.141** 源码树（当前模块 `vermagic=4.9.141 SMP preempt mod_unload modversions aarch64`）。
- 你自己的 Pixel C stock/LOS vendor 里的 NVIDIA sensor HAL 文件（`sensors.dragon.so`、`android.hardware.sensors@1.0-impl.so`、`vendor.nvidia.hardware.sensors@1.0-service`）。这些是闭源文件，不要从不明来源下载。
- TWRP，能读写 `/vendor`（Pixel C 上是 `/dev/block/mmcblk0p5`）。
- 先备份整个 vendor 分区。

**步骤 1：编译/放置内核模块**

```sh
# 同一棵 kernel 4.9.141 树
# 确认 CONFIG_IIO_CROS_EC_SENSORS_CORE=m
#      CONFIG_IIO_CROS_EC_SENSORS=m
make O="$KOBJ" ARCH=arm64 CROSS_COMPILE="$TC" \
    M=drivers/iio/common/cros_ec_sensors modules
```

本版本使用的模块：

| File | Size | SHA-256 |
|---|---:|---|
| `cros_ec_sensors_core.ko` | 367,832 | `d4d69fe8ac37f050e9225a263cf992d5a91201563b80e3b75a7fd5a096e68450` |
| `cros_ec_sensors.ko` | 312,912 | `2a1eb3ee4fd6fdbc3a49d37f6d46028aec0475db6a1e724dcf7af9d002869391` |

复制到 vendor：

```sh
mount /dev/block/mmcblk0p5 /vendor
cp cros_ec_sensors_core.ko /vendor/lib/modules/
cp cros_ec_sensors.ko      /vendor/lib/modules/
chmod 644 /vendor/lib/modules/cros_ec_sensors*.ko
chown root:root /vendor/lib/modules/cros_ec_sensors*.ko
chcon u:object_r:vendor_file:s0 /vendor/lib/modules/cros_ec_sensors*.ko
```

**步骤 2：让 init 自动加载**

`/vendor/bin/lkm_loader.sh` 会读取 `/vendor/lib/modules/modules.load`。在文件末尾追加（注意不带 `.ko`）：

```text
cros_ec_sensors_core
cros_ec_sensors
```

并在 `/vendor/lib/modules/modules.dep` 追加：

```text
/vendor/lib/modules/cros_ec_sensors_core.ko:
/vendor/lib/modules/cros_ec_sensors.ko: /vendor/lib/modules/cros_ec_sensors_core.ko
```

然后：

```sh
chcon u:object_r:vendor_file:s0 \
  /vendor/lib/modules/modules.load \
  /vendor/lib/modules/modules.dep
```

**步骤 3：放置 NVIDIA sensor HAL**

需要这 5 个文件（来自你自己的 Pixel C vendor）：

| Destination | SHA-256 |
|---|---|
| `/vendor/bin/vendor.nvidia.hardware.sensors@1.0-service` | `4c0da9eb5fde450be2b2ea3f6d9bc34f64ac404322b5b52acc3f89964100d3d6` |
| `/vendor/lib64/hw/sensors.dragon.so` | `f1058fd0b232f043a1e89e89ef9c3e79877d12e5be67bee0ec4ebf6fdfeb8687` |
| `/vendor/lib64/hw/android.hardware.sensors@1.0-impl.so` | `559b48ab0f745582db5f338800bb241ceb31dee7ceb2eb78e1d4f19535b44c6e` |
| `/vendor/etc/init/vendor.nvidia.hardware.sensors@1.0-service.rc` | 见下方 |
| `/vendor/etc/permissions/android.hardware.sensor.*.xml` | accelerometer/compass/gyroscope/light |

`.rc` 必须是 root + DAC 权限，因为 HAL 要写 `/sys/bus/iio/...` 的 ring 属性：

```text
service vendor.sensors-hal-1-0 /vendor/bin/vendor.nvidia.hardware.sensors@1.0-service
    class hal
    user root
    group root
    capabilities BLOCK_SUSPEND DAC_OVERRIDE DAC_READ_SEARCH
    rlimit rtprio 10 10
```

权限和 SELinux context：

```sh
chmod 755 /vendor/bin/vendor.nvidia.hardware.sensors@1.0-service
chmod 644 /vendor/lib64/hw/sensors.dragon.so \
          /vendor/lib64/hw/android.hardware.sensors@1.0-impl.so \
          /vendor/etc/init/vendor.nvidia.hardware.sensors@1.0-service.rc \
          /vendor/etc/permissions/android.hardware.sensor.*.xml
chcon u:object_r:hal_sensors_tegra_exec:s0 \
  /vendor/bin/vendor.nvidia.hardware.sensors@1.0-service
chcon u:object_r:vendor_file:s0 \
  /vendor/lib64/hw/sensors.dragon.so \
  /vendor/lib64/hw/android.hardware.sensors@1.0-impl.so
chcon u:object_r:vendor_configs_file:s0 \
  /vendor/etc/init/vendor.nvidia.hardware.sensors@1.0-service.rc \
  /vendor/etc/permissions/android.hardware.sensor.*.xml
```

**步骤 4：VINTF manifest**

在 `/vendor/etc/vintf/manifest.xml` 里加入（或放一个 fragment 到 `/vendor/etc/vintf/manifest/`）：

```xml
<hal format="hidl">
    <name>android.hardware.sensors</name>
    <transport>hwbinder</transport>
    <version>1.0</version>
    <interface>
        <name>ISensors</name>
        <instance>default</instance>
    </interface>
    <fqname>@1.0::ISensors/default</fqname>
</hal>
```

否则 `checkvintf` 会报 missing HAL，Android 不会启动 sensor service。

**步骤 5：SELinux**

在 vendor policy（`vendor_sepolicy.cil` 或 `vendor/sepolicy`）加入以下规则。当前 Android 13 vendor 使用 `iio_device_33_0`；LOS22.2/Android 15 的类型可能是 `iio_device_35_0`，用 `sesearch`/`secilc` 确认后替换，不要盲目照抄：

```text
(allow hal_sensors_tegra sysfs (file (open read write getattr)))
(allow hal_sensors_tegra sysfs_iio (file (ioctl read write getattr lock append map open watch watch_reads)))
(allow hal_sensors_tegra iio_device_33_0 (chr_file (ioctl read write getattr lock append map open watch watch_reads)))
(allow hal_sensors_tegra sysfs_nvi2c (dir (ioctl read getattr lock open watch watch_reads search)))
(allow hal_sensors_tegra sysfs_nvi2c (file (ioctl read write getattr lock append map open watch watch_reads)))
(allow hal_sensors_tegra self (capability (dac_override dac_read_search)))
```

同时确保 `vendor.nvidia.hardware.sensors@1.0-service` 的 file context 是 `hal_sensors_tegra_exec`，`/dev/iio:device*` 是 `iio_device_33_0`。

**步骤 6：重启并验证**

```sh
adb shell dumpsys sensorservice | grep -A8 'Active sensors'
adb shell logcat -b all -d | grep -iE 'cros|sensor|iio' | tail -50
adb shell settings get system accelerometer_rotation
adb shell settings put system accelerometer_rotation 1
```

应看到：

```text
CrosEC Accelerometer
CrosEC Gyroscope
CrosEC Magnetometer
```

### 2.4 第 2 层：MIUI 平板属性

备份 `/product/etc/build.prop`，确认/写入：

```properties
ro.build.characteristics=tablet
ro.miui.notch=0
```

TWRP 示例（注意先确认文件末尾有换行）：

```sh
mount /dev/block/bootdevice/by-name/APP /system_root
cp /system_root/product/etc/build.prop /data/local/tmp/product-build.prop.bak
grep -n 'ro.build.characteristics' /system_root/product/etc/build.prop || \
    printf '\nro.build.characteristics=tablet\n' >> /system_root/product/etc/build.prop
printf '\nro.miui.notch=0\n' >> /system_root/product/etc/build.prop
chmod 644 /system_root/product/etc/build.prop
restorecon /system_root/product/etc/build.prop
sync
```

重启后应变为 Pad 布局、横屏，并能随重力旋转。验证：

```sh
adb shell getprop ro.build.characteristics     # tablet
adb shell settings get system accelerometer_rotation  # 1
```

### 2.5 挖孔/黑椭圆

这个黑椭圆不是 `DisplayCutout`，截图也截不到；它来自 MIUI SystemUI 的受保护 `RoundCornerTop` 圆角层。最终方案是 `PixelCRoundCorner.apk` 静态 RRO（目标 `com.android.systemui`），覆盖圆角资源，黑椭圆消失，屏幕四角保持直角。已内置在 `/product/overlay/PixelCRoundCorner.apk` 并启用。

可选方案：`support_round_corner=false` 也能去掉椭圆，但底部圆角会一起消失；开发者选项里的“模拟显示切口”只能处理标准 cutout。

### 2.6 电源键

Pixel C 需要 `key 116 POWER`，不要加 `WAKE`。当前 `Generic.kl` 第 138 行为：

```text
key 116   POWER
```

`gpio-keys.kl` 保留音量键 114/115。

### 2.7 手势导航

MIUI HomePadS 的 `GestureOperationHelper.updateRegion()` 用 `min(2560,1800)=1800` 作为宽、`max=2560` 作为高，并按手机 `rotation=0` 逻辑计算底部区域，导致原生横屏平板的底部上划区域错位。若手势不灵，先用三键导航；直接替换 `MiuiHomePadS.apk` 会因 MIUI 签名校验失败。当前 `navigation_mode=2`（手势）；底部上划仍不命中时，这是已知的 MiuiHomePadS 限制。

### 2.8 LOS20 / LOS22.2 兼容性

| ROM | 传感器 vendor 栈 | 说明 |
|---|---|---|
| LOS20 (Android 13) | 基本可以，成功率很高 | 内核同为 4.9.141 时可直接复用；用 `iio_device_33_0` 策略。 |
| LOS22.2 (Android 15) | 原理上可以，但不是直接照抄 | 内核模块可能可复用；HAL/VINTF/SELinux 要按 API 35 适配。LOS22.2 原厂 vendor 缺少 `android.hardware.sensors@1.0.so`，必要时从 Android 13 vendor 补库。不要刷本目录的 HyperOS vendor。 |
| HyperOS/MIUI（本版本） | 本目录 `vendor.img` 已包含 | 还需要 product 的 tablet 属性才有 Pad UI。 |

LOS20/LOS22.2 不需要 `ro.build.characteristics=tablet` 和 `ro.miui.notch=0`；那是 MIUI 专用。LOS 只要传感器出数据并打开 `accelerometer_rotation`。

## 3. 其他成功修复

| 修复 | 说明 |
|---|---|
| 黑屏/扬声器爆音 | 补丁 `libaudiopolicymanagerdefault.so` 偏移 `0x90D34`（`00 4b 00 34` → `1f 20 03 d5`），并加 `service.bootanim.exit=1`。 |
| 设置去掉移动网络与 SIM | 修改 `Settings.apk` 的 `res/xml/wireless_settings.xml`、`res/xml/functionindex.xml`、`assets/index.json`；`config_mobile_data_capable=false` overlay。Settings SHA-256 `23103bae92c17557af2c6581d1028f1710e133e7dbf0dd2a2ef4e87f91a3dfa1`。 |
| 仅 Wi-Fi overlay | `/product/overlay/PixelcWifiOnlyOverlay.apk`；SHA-256 `5aa5f440f13ccdd10df969a60296522adf2bfa4da9d45793423f46ae52ca0e50`。 |
| WebView | AOSP WebView 156.0.8078.47，`/product/app/webview/webview.apk`；SHA-256 `32ee1b3919bb2a8b243d204583326de8c245bd9ad330bbaceaef5beb4b0f8c3c`。 |
| 精简 | 删除 MiShop、MIMediaEditor、SmartHome、MIUIYoupin、MiuiScanner、MIUIHuanji、MIUIVipAccountPad、MIUIDuokanReaderPad、MIUIGameCenterPad、MIUIEmail、Creation、旧百度输入法、MIUIXiaoAiSpeechEngine、MSA、MIUIGuardProvider，以及通话/SIM 相关 UI。Google 套件保留。 |
| 删除 FindDevice | 删除 `/system_ext/priv-app/FindDevice`；此前约 8.6% CPU 常驻。 |
| ThemeManager 慢 | 给 `MIUIThemeManagerPad.apk` 加 `android.permission.INTERACT_ACROSS_USERS`，平台签名；首次 `Displayed` 从约 1.9 s 降到约 0.8 s。补丁 SHA-256 `185581d7aad0c9544f0ef3add3649764c5e2696e13399f552511042cc3d4943f`。 |
| 亮屏壁纸黑遮罩 | `WallpaperServiceController.shouldDoBlackoutAnim()` 强制返回 `false`；避免 `mRevealValue=0.6699` 卡住。补丁 SHA-256 `b95c97ced35ace64b37c62429355acf46e3e2b236f6e8a8354d37742a917a755`。 |
| 百度输入法 | 用户安装的 `com.baidu.input` 13.3.18.31 内置为 `/system/app/BaiduInput/BaiduInput.apk`；SHA-256 `92915ffbaf9a60751aadf5819d8dd0707398b0599dfc1f24764bab85e8a13ec6`。格式化后需选为默认输入法。 |
| 8GB EMC | `/vendor/etc/init/hw/init.t210_common.rc` 写 `emc_force_max_rate 1`；EMC 表最高 1600 MHz，无 1862/2133 表，不要盲目超频。 |
| 系统分区 | 已扩到完整 8 GB。 |
| AOT/dexopt | system 和 vendor `build.prop` 追加 `pm.dexopt.first-boot/install/boot-after-ota/bg-dexopt=speed`；原版 vendor 没有这四行。格式化 data 后首次开机会全量 dex2oat，慢且发热。 |
| 8GB RAM/BCT | 基于 T210/Switch 的 8GB BCT；SPI 签名区未动。 |

## 4. 刷写

```sh
# 8GB system 建议用 fastboot 29.0.6
fastboot flash boot boot.img
fastboot flash vendor vendor.img
fastboot flash system system.img
# 推荐在 TWRP 格式化 data，或：
fastboot -w
fastboot reboot
```

- 首次开机因 `pm.dexopt.first-boot=speed` 会全量 AOT，可能 10–30 分钟，发热大；请插电、散热。
- 不要用新版 fastboot 刷 8GB system；部分版本会 `std::bad_alloc`。
- 格式化 data 后，`/data/app` 更新消失，system 里的 ThemeManager/WallpaperT/BaiduInput 生效；ContactsProvider 数据库重建。
- 传感器修复在 vendor；只刷 system 不会带上它。

### 4.1 System 扩容与分区注意事项（重要）

> **绝对不能删的最小分区：`PST`（512 KiB）。** 它保存 ChromeOS/bootloader 的 persistent 状态；删除或重建会导致 bootloader 状态异常，**可能直接不开机**。`MSC`（4 MiB，`/misc`）同样不能删，它保存 BCB。扩容时只动 APP/UDA，其他小分区一律保留。

**原机 APP/system 分区只有 3.5 GiB：**

```text
3,758,096,384 B = 3.5 GiB
```

本目录的 `system.img` 是 **8,589,934,592 B = 8 GiB**，是为扩容后的 APP 分区做的。**不要把它 dd 到原机 3.5 GiB 的 APP 分区**：`dd` 不会在分区边界停下，会继续写进后面的 UDA，破坏 data，甚至导致无法开机。

当前布局（本机）：

| 分区 | 设备节点 | 大小 |
|---|---|---:|
| APP/system | `mmcblk0p4` | 8,589,934,592 B = 8 GiB |
| VNR/vendor | `mmcblk0p5` | 318,767,104 B = 304 MiB |
| UDA/data | `mmcblk0p7` | 从原 24.7 GiB 缩小，给 APP 让出空间 |

**扩容方式：** Pixel C 的 eMMC 原本几乎分满，没有未分配空间；把 APP 从 3.5 GiB 扩到 8 GiB，必须**重写 GPT 分区表**，把 APP 的结束位置后移，同时把 UDA 的起始位置后移（相当于从 UDA 头部让出空间）。这不是单纯的 `resize2fs`，而是改分区表。

**必须保留的注意事项：**

1. **先备份 GPT，再动分区表。** TWRP 下：
   ```sh
   sgdisk --print /dev/block/mmcblk0
   sgdisk --backup=/sdcard/gpt-$(date +%Y%m%d-%H%M).bin /dev/block/mmcblk0
   ```
   把备份复制到电脑。**绝对不要刷入另一台 Pixel C 的 GPT 备份**；GPT 里有本机分区 offset/GUID，刷错会找不到 KERN-A/APP，直接进不去系统。
2. **先备份 UDA/data。** 缩小 UDA 会移动它的起始位置；原有 data 可能无法挂载，扩容后通常要格式化 data。
3. **不要用整包 `flashall` 或别的 ROM 的 GPT。** 只改 APP/UDA 两项，KERN-A、KERN-B、CAC、VNR 等保持原样。
4. **确认活动启动槽是 KERN-A。** Pixel C 有 KERN-A/KERN-B；当前系统从 KERN-A 启动。`fastboot flash boot` 默认写 KERN-A；不要误写 KERN-B 后以为没生效。
5. **写 system 前先确认 APP 大小：**
   ```sh
   adb shell blockdev --getsize64 /dev/block/mmcblk0p4
   # 必须是 8589934592
   ```
   不是 8 GiB 就不要写这个 8 GiB 镜像。
6. **写入后校验：**
   ```sh
   adb shell blockdev --getsize64 /dev/block/mmcblk0p4
   adb shell blockdev --getsize64 /dev/block/mmcblk0p5
   adb shell sync
   ```
7. **分区表错了的症状：** 卡 LOGO、黑屏、fastboot 能进但 Android 挂不上 `/system`，或 `/data` 加密/挂载失败。此时不要反复格式化；先用本机 GPT 备份恢复分区表，再重刷 boot/vendor/system。
8. **`/dev/block/by-name/system` 可能不存在或指向不同布局。** 不确定时用 `mmcblk0p4`（APP）和 `mmcblk0p5`（VNR），不要用整盘 `/dev/block/mmcblk0` 作为 `dd` 输出。

**不能删的小分区（就是你说的“特别小的占位”）：**

| 分区 | 大小 | 作用 / 注意 |
|---|---:|---|
| `MSC` | 4 MiB | Android `/misc`，存 bootloader control block（BCB）。**不要 `fastboot erase misc`，不要在重新分区时删掉它。** 删了会导致 recovery/重启到 bootloader 等命令失效；深度链（depthcharge）虽然可能允许普通启动，但 misc 读失败会报 disk error。 |
| `PST` | 512 KiB | ChromeOS persistent 状态分区，最小的一块，**删除会导致 bootloader 状态异常、可能不开机**。保留原样，不要删、不要格式化；它太小，也没有空间给你放 system/vendor。 |
| `MD1` | 64 MiB | Android `/metadata`，metadata 加密相关。删了/重建会导致 data 解密、OTA、部分系统服务异常。 |
| `LNX` | 32 MiB | 启动/平台相关保留分区。不要动。 |
| `KERN-A` / `KERN-B` | 各 32 MiB | 两个启动槽。当前从 KERN-A 启动；不要为了“省空间”删 KERN-B。 |

本机 GPT 实测：`MSC` 是 `mmcblk0p10`（4096 KiB），`PST` 是 `mmcblk0p11`（512 KiB，磁盘最后一块）。`PST` 头部是非零数据，不是空白占位，说明它存有 bootloader/persistent 状态；**不要清空它**。

扩容时**只改 APP 和 UDA**，`MSC`、`PST`、`MD1`、`LNX`、`KERN-A/B` 全部保持原样。不要“重新生成一张全新 GPT”再从零建分区；优先在原 GPT 上调整 APP/UDA 的边界。这样即使扩容失败，也还能从 KERN-A 和原有小分区恢复。

**关于 `/data`、FBE 和 `fastboot -w`：** MoKee/MIUI 的 fstab 里 `/data` 是 `forcefdeorfbe=.../by-name/MD1,quota`，也就是 **FBE 加密 + MD1 存 metadata**。`fastboot -w` 不只是“格式化 data”，它还会 erase/重建 data、cache 和 metadata（MD1）。所以：

- 刷完 `-w` 后，TWRP 不解密旧 data 是正常的；旧 FBE 密钥/状态已经没了。
- 如果 TWRP 里再 format 一次 data 才挂得上，也正常——这是在清掉残留的 FBE/metadata 状态。
- 想保留旧 data 就不要 `-w`；换 ROM/换 FBE 状态时，`-w` 反而是干净路线。
- `/system` 挂不上通常不是“system 被加密”，更可能是 ext4 状态、挂载点冲突或分区表/镜像不匹配；不要把它和 MD1/FBE 混为一谈。

## 5. 已知限制

- EMC 最高 1600 MHz；1862/2133 需要新 BCT/EMC timing，未包含。
- 磁力计有数据但未校准，指南针精度可能较差。
- 全部传感器关闭后 EC 可能仍按约 400 Hz 采样，可能增加耗电。
- MiuiHomePadS 手势区域 bug 可能仍在；三键导航最稳。
- 本目录 vendor 的传感器栈是手工整合的；从 LOS 设备树重编 vendor 若没加模块/HAL/策略会丢失。
- ThemeManager/WallpaperT 是平台签名修改版；后续系统更新可能覆盖。
- 百度输入法是用户签名版；格式化后需选默认。
- Google 套件保留；需要 Play 服务就不要删。

## 6. 致谢

- 感谢 **@npjohnson** 和 **@Electimon**：本次使用的 boot + vendor 来自他们维护的 **LineageOS 20–22** 开源项目，本项目的 boot 和 vendor 都是在这基础上进行修改的。
- 感谢 **@汽水°** 大佬提供的 **8GB SPI（bootloader）**。
- 感谢 **@lumingyu0423** 大佬的 **[Pixel-C_custom_bootloader](https://github.com/lumingyu0423/Pixel-C_custom_bootloader/blob/main/README.md)**。
- Pixel C / Tegra X1 社区、LineageOS 20 vendor、hekate/T210 8GB SDRAM 参考。
- Xiaomi Pad 5 (nabu) HyperOS 底包。
- 所有测试和镜像验证都在实际 Pixel C 8GB 硬件上完成。

---

# Part 2 — English

> **Disclaimer**
>
> This porting project was completed entirely with Cherry Studio using DeepSeek V4.1 Flash. The system is only at an initial usable stage and has many uncertainties and bugs. The system partition has been re-signed and modified; users with the necessary skills may fix some bugs themselves. The test device has been modified to 8GB DRAM; smoothness on 3GB devices is not guaranteed. There is no possibility of later maintenance for this project.

## 0. Summary

This build keeps Pixel C hardware working while using the Xiaomi Pad 5 HyperOS system. The gravity-sensor/tablet-layout fix has two independent layers:

1. **Sensor data layer (vendor)**: CrosEC kernel modules + NVIDIA sensor HAL + VINTF + SELinux, so the accelerometer, gyroscope and magnetometer actually produce data.
2. **MIUI device-class layer (product)**: `ro.build.characteristics=tablet` makes MIUI use the Pad layout and allows auto-rotation.

Both layers are required. Changing only `ro.build.characteristics` does not create sensor data; fixing only the vendor sensor stack may still leave MIUI in a phone-style portrait layout.

**Do not flash these images to another device.**

## 1. Device & Images

| Image | Size (bytes) | SHA-256 |
|---|---:|---|
| `boot.img` (KERN-A, mmcblk0p1) | 33,554,432 | `280059679EAA3481165490CBE57853D25175FEF5585239616F5A62355352DE1E` |
| `vendor.img` (VNR, mmcblk0p5) | 318,767,104 | `60CABF265FAE374A24F31280DD26D6AC223E3AE97695A3E2C074FC6DC0446DF1` |
| `system.img` (APP, mmcblk0p4) | 8,589,934,592 | `9D0D524ABAEBDF9D297396CE9146EF681C53EE5EDF38B0D80F98D281D42B14F5` |

### What lives in which image

| Partition | Contents |
|---|---|
| `boot.img` | KERN-A boot image/kernel. |
| `vendor.img` | CrosEC sensor modules, NVIDIA sensor HAL, VINTF, SELinux; EMC init line; vendor `pm.dexopt.*`; vendor WiFi supplicant fix. |
| `system.img` | `/system`, `/product`, `/system_ext`: ThemeManager/WallpaperT patches, Baidu input, Settings, overlays, WebView, FindDevice removal, `ro.build.characteristics=tablet`, `ro.miui.notch=0`, system `pm.dexopt.*`, audio policy patch. |
| `/data` | User apps, `/data/app` updates, settings, databases, dex2oat artifacts. Wiped by data format. |

"All fixes are in system" is not accurate: **the sensor stack is in vendor, the boot image is in boot, and UI/APK fixes are in system**. Formatting data does not touch these three partitions; as long as you do not reflash them, the current fixes remain.

## 2. Complete Gravity Sensor / Tablet Layout Fix

### 2.1 Symptoms

- Desktop looks like a phone: portrait, wrong grid, wrong padding.
- Screen does not rotate with gravity, or rotates only in some apps.
- A black oval/"punch hole" appears at the top center in landscape and does not move.
- If `dumpsys sensorservice` does not list `CrosEC Accelerometer`, the sensor HAL is not up; fix layer 1 first.

### 2.2 Root cause

**Sensor layer:** Stock LOS20/LOS22.2 vendor images only contain the VINTF string for `android.hardware.sensors::ISensors`; they have no `cros_ec_sensors*.ko`, no `sensors.dragon.so`, and no NVIDIA sensor HAL service. The sensor service cannot start.

**MIUI UI layer:** When `ro.build.characteristics` is empty or not `tablet`, `MiuiHomePadS` sees `miui.os.Build.IS_TABLET == false`, applies the phone layout, locks portrait, and draws phone-style corner/cutout overlays. Pixel C's native panel is 2560×1800 landscape; MIUI's portrait coordinate assumptions make it look like the gravity sensor is broken.

### 2.3 Layer 1 — Vendor sensor stack

Pixel C's accelerometer/gyroscope/magnetometer are exposed through the ChromeOS EC. The `vendor.img` in this folder already contains the complete stack. To reproduce it on LOS20/LOS22.2 or a clean vendor, follow these steps.

**Prerequisites:**

- The **same kernel 4.9.141 source tree** as the boot image (current modules: `vermagic=4.9.141 SMP preempt mod_unload modversions aarch64`).
- NVIDIA sensor HAL files from your own Pixel C stock/LOS vendor (`sensors.dragon.so`, `android.hardware.sensors@1.0-impl.so`, `vendor.nvidia.hardware.sensors@1.0-service`). These are proprietary; do not download them from unknown sources.
- TWRP with writable `/vendor` (Pixel C: `/dev/block/mmcblk0p5`).
- Back up the vendor partition first.

**Step 1 — Build/copy kernel modules**

```sh
# Same kernel 4.9.141 tree
# CONFIG_IIO_CROS_EC_SENSORS_CORE=m
# CONFIG_IIO_CROS_EC_SENSORS=m
make O="$KOBJ" ARCH=arm64 CROSS_COMPILE="$TC" \
    M=drivers/iio/common/cros_ec_sensors modules
```

Files used by this build:

| File | Size | SHA-256 |
|---|---:|---|
| `cros_ec_sensors_core.ko` | 367,832 | `d4d69fe8ac37f050e9225a263cf992d5a91201563b80e3b75a7fd5a096e68450` |
| `cros_ec_sensors.ko` | 312,912 | `2a1eb3ee4fd6fdbc3a49d37f6d46028aec0475db6a1e724dcf7af9d002869391` |

Copy to vendor:

```sh
mount /dev/block/mmcblk0p5 /vendor
cp cros_ec_sensors_core.ko /vendor/lib/modules/
cp cros_ec_sensors.ko      /vendor/lib/modules/
chmod 644 /vendor/lib/modules/cros_ec_sensors*.ko
chown root:root /vendor/lib/modules/cros_ec_sensors*.ko
chcon u:object_r:vendor_file:s0 /vendor/lib/modules/cros_ec_sensors*.ko
```

**Step 2 — Auto-load**

`/vendor/bin/lkm_loader.sh` reads `/vendor/lib/modules/modules.load`. Append (without `.ko`):

```text
cros_ec_sensors_core
cros_ec_sensors
```

Append dependency lines to `/vendor/lib/modules/modules.dep`:

```text
/vendor/lib/modules/cros_ec_sensors_core.ko:
/vendor/lib/modules/cros_ec_sensors.ko: /vendor/lib/modules/cros_ec_sensors_core.ko
```

Then:

```sh
chcon u:object_r:vendor_file:s0 \
  /vendor/lib/modules/modules.load \
  /vendor/lib/modules/modules.dep
```

**Step 3 — NVIDIA sensor HAL**

Required files (from your own Pixel C vendor):

| Destination | SHA-256 |
|---|---|
| `/vendor/bin/vendor.nvidia.hardware.sensors@1.0-service` | `4c0da9eb5fde450be2b2ea3f6d9bc34f64ac404322b5b52acc3f89964100d3d6` |
| `/vendor/lib64/hw/sensors.dragon.so` | `f1058fd0b232f043a1e89e89ef9c3e79877d12e5be67bee0ec4ebf6fdfeb8687` |
| `/vendor/lib64/hw/android.hardware.sensors@1.0-impl.so` | `559b48ab0f745582db5f338800bb241ceb31dee7ceb2eb78e1d4f19535b44c6e` |
| `/vendor/etc/init/vendor.nvidia.hardware.sensors@1.0-service.rc` | see below |
| `/vendor/etc/permissions/android.hardware.sensor.*.xml` | accelerometer/compass/gyroscope/light |

The `.rc` must run as root with DAC capabilities because the HAL writes the IIO ring attributes under `/sys/bus/iio/...`:

```text
service vendor.sensors-hal-1-0 /vendor/bin/vendor.nvidia.hardware.sensors@1.0-service
    class hal
    user root
    group root
    capabilities BLOCK_SUSPEND DAC_OVERRIDE DAC_READ_SEARCH
    rlimit rtprio 10 10
```

Modes and SELinux contexts:

```sh
chmod 755 /vendor/bin/vendor.nvidia.hardware.sensors@1.0-service
chmod 644 /vendor/lib64/hw/sensors.dragon.so \
          /vendor/lib64/hw/android.hardware.sensors@1.0-impl.so \
          /vendor/etc/init/vendor.nvidia.hardware.sensors@1.0-service.rc \
          /vendor/etc/permissions/android.hardware.sensor.*.xml
chcon u:object_r:hal_sensors_tegra_exec:s0 \
  /vendor/bin/vendor.nvidia.hardware.sensors@1.0-service
chcon u:object_r:vendor_file:s0 \
  /vendor/lib64/hw/sensors.dragon.so \
  /vendor/lib64/hw/android.hardware.sensors@1.0-impl.so
chcon u:object_r:vendor_configs_file:s0 \
  /vendor/etc/init/vendor.nvidia.hardware.sensors@1.0-service.rc \
  /vendor/etc/permissions/android.hardware.sensor.*.xml
```

**Step 4 — VINTF manifest**

Add to `/vendor/etc/vintf/manifest.xml` (or a fragment under `/vendor/etc/vintf/manifest/`):

```xml
<hal format="hidl">
    <name>android.hardware.sensors</name>
    <transport>hwbinder</transport>
    <version>1.0</version>
    <interface>
        <name>ISensors</name>
        <instance>default</instance>
    </interface>
    <fqname>@1.0::ISensors/default</fqname>
</hal>
```

Without it, `checkvintf` reports a missing HAL and Android will not start the sensor service.

**Step 5 — SELinux**

Add these rules to vendor policy (`vendor_sepolicy.cil` or `vendor/sepolicy`). Android 13 uses `iio_device_33_0`; LOS22.2/Android 15 may use `iio_device_35_0`. Verify with `sesearch`/`secilc`; do not blindly copy the type name:

```text
(allow hal_sensors_tegra sysfs (file (open read write getattr)))
(allow hal_sensors_tegra sysfs_iio (file (ioctl read write getattr lock append map open watch watch_reads)))
(allow hal_sensors_tegra iio_device_33_0 (chr_file (ioctl read write getattr lock append map open watch watch_reads)))
(allow hal_sensors_tegra sysfs_nvi2c (dir (ioctl read getattr lock open watch watch_reads search)))
(allow hal_sensors_tegra sysfs_nvi2c (file (ioctl read write getattr lock append map open watch watch_reads)))
(allow hal_sensors_tegra self (capability (dac_override dac_read_search)))
```

Also ensure the service file context is `hal_sensors_tegra_exec` and `/dev/iio:device*` is `iio_device_33_0` (or the API-35 equivalent).

**Step 6 — Reboot and verify**

```sh
adb shell dumpsys sensorservice | grep -A8 'Active sensors'
adb shell logcat -b all -d | grep -iE 'cros|sensor|iio' | tail -50
adb shell settings get system accelerometer_rotation
adb shell settings put system accelerometer_rotation 1
```

Expected:

```text
CrosEC Accelerometer
CrosEC Gyroscope
CrosEC Magnetometer
```

### 2.4 Layer 2 — MIUI tablet property

Back up `/product/etc/build.prop`, then ensure:

```properties
ro.build.characteristics=tablet
ro.miui.notch=0
```

TWRP example (make sure the file ends with a newline before appending):

```sh
mount /dev/block/bootdevice/by-name/APP /system_root
cp /system_root/product/etc/build.prop /data/local/tmp/product-build.prop.bak
grep -n 'ro.build.characteristics' /system_root/product/etc/build.prop || \
    printf '\nro.build.characteristics=tablet\n' >> /system_root/product/etc/build.prop
printf '\nro.miui.notch=0\n' >> /system_root/product/etc/build.prop
chmod 644 /system_root/product/etc/build.prop
restorecon /system_root/product/etc/build.prop
sync
```

After reboot the launcher should use the Pad layout, landscape, and rotate with gravity. Verify:

```sh
adb shell getprop ro.build.characteristics     # tablet
adb shell settings get system accelerometer_rotation  # 1
```

### 2.5 Cutout / black oval

The black oval is not a `DisplayCutout`; it cannot be captured by `screencap`. It comes from MIUI SystemUI's protected `RoundCornerTop` layer. The final fix is the static RRO `PixelCRoundCorner.apk` (target `com.android.systemui`), which overrides the corner resources; the oval disappears and the screen is square-cornered. It is already in `/product/overlay/PixelCRoundCorner.apk` and enabled.

Alternative: `support_round_corner=false` removes the oval too, but also removes the bottom rounded corners. The developer option "Simulate display cutout" only handles standard cutouts.

### 2.6 Power key

Pixel C needs `key 116 POWER` without the `WAKE` modifier. Current `Generic.kl` line 138 is:

```text
key 116   POWER
```

`gpio-keys.kl` keeps volume keys 114/115.

### 2.7 Gesture navigation

MiuiHomePadS `GestureOperationHelper.updateRegion()` uses `min(2560,1800)=1800` as width and `max=2560` as height, then applies phone `rotation=0` logic to the bottom gesture region. This is wrong for a native-landscape tablet. If gestures fail, use 3-button navigation. Replacing `MiuiHomePadS.apk` directly fails MIUI signature verification. Current `navigation_mode=2` (gesture); bottom swipe may still miss, which is a known MiuiHomePadS limitation.

### 2.8 LOS20 / LOS22.2 compatibility

| ROM | Sensor vendor stack | Notes |
|---|---|---|
| LOS20 (Android 13) | Likely works, high confidence | Reuse modules if the kernel is also 4.9.141; use `iio_device_33_0` policy. |
| LOS22.2 (Android 15) | Possible, but not a drop-in | Kernel modules may be reusable; HAL/VINTF/SELinux must be adapted for API 35. Stock LOS22.2 vendor lacks `android.hardware.sensors@1.0.so`; copy it from the Android 13 vendor if needed. Do not flash this HyperOS vendor. |
| HyperOS/MIUI (this build) | Included in this `vendor.img` | Also needs the product tablet property for Pad UI. |

LOS20/LOS22.2 do not need `ro.build.characteristics=tablet` or `ro.miui.notch=0`; those are MIUI-specific. On LOS, just make the sensor work and enable `accelerometer_rotation`.

## 3. Other Successful Fixes

| Fix | Details |
|---|---|
| Black screen/speaker pop | Patched `libaudiopolicymanagerdefault.so` at offset `0x90D34` (`00 4b 00 34` → `1f 20 03 d5`), plus `service.bootanim.exit=1`. |
| Settings without mobile/SIM | Modified `Settings.apk`: `res/xml/wireless_settings.xml`, `res/xml/functionindex.xml`, `assets/index.json`; `config_mobile_data_capable=false` overlay. Settings SHA-256 `23103bae92c17557af2c6581d1028f1710e133e7dbf0dd2a2ef4e87f91a3dfa1`. |
| Wi-Fi-only overlay | `/product/overlay/PixelcWifiOnlyOverlay.apk`; SHA-256 `5aa5f440f13ccdd10df969a60296522adf2bfa4da9d45793423f46ae52ca0e50`. |
| WebView | AOSP WebView 156.0.8078.47 at `/product/app/webview/webview.apk`; SHA-256 `32ee1b3919bb2a8b243d204583326de8c245bd9ad330bbaceaef5beb4b0f8c3c`. |
| Debloat | Removed MiShop, MIMediaEditor, SmartHome, MIUIYoupin, MiuiScanner, MIUIHuanji, MIUIVipAccountPad, MIUIDuokanReaderPad, MIUIGameCenterPad, MIUIEmail, Creation, old BaiduIME, MIUIXiaoAiSpeechEngine, MSA, MIUIGuardProvider, and call/SIM UI. Google suite kept. |
| FindDevice removed | Deleted `/system_ext/priv-app/FindDevice`; it was consuming ~8.6% CPU. |
| ThemeManager slow | Added `android.permission.INTERACT_ACROSS_USERS` to `MIUIThemeManagerPad.apk`, platform-signed; first `Displayed` improved from ~1.9 s to ~0.8 s. Patched SHA-256 `185581d7aad0c9544f0ef3add3649764c5e2696e13399f552511042cc3d4943f`. |
| Wallpaper blackout | `WallpaperServiceController.shouldDoBlackoutAnim()` forced to `false`; prevents the stuck `mRevealValue=0.6699` black frame. Patched SHA-256 `b95c97ced35ace64b37c62429355acf46e3e2b236f6e8a8354d37742a917a755`. |
| Baidu input | User-installed `com.baidu.input` 13.3.18.31 baked as `/system/app/BaiduInput/BaiduInput.apk`; SHA-256 `92915ffbaf9a60751aadf5819d8dd0707398b0599dfc1f24764bab85e8a13ec6`. Select as default after data wipe. |
| 8 GB EMC | `/vendor/etc/init/hw/init.t210_common.rc` writes `emc_force_max_rate 1`; EMC table max is 1600 MHz, no 1862/2133 table; do not blindly overclock. |
| System partition | Resized to full 8 GB. |
| AOT/dexopt | system and vendor `build.prop` append `pm.dexopt.first-boot/install/boot-after-ota/bg-dexopt=speed`; original vendor had none. After data wipe, first boot runs full dex2oat (slow/hot). |
| 8 GB RAM/BCT | T210/Switch-derived 8 GB BCT; SPI signed region untouched. |

## 4. Flashing

```sh
# Use fastboot 29.0.6 for the 8 GB system image
fastboot flash boot boot.img
fastboot flash vendor vendor.img
fastboot flash system system.img
# Recommended: format data in TWRP, or:
fastboot -w
fastboot reboot
```

- First boot runs full AOT because `pm.dexopt.first-boot=speed`; expect 10–30+ minutes and heat. Use a charger and good cooling.
- Do not use newer fastboot for the 8 GB system; some versions fail with `std::bad_alloc`.
- After data wipe, `/data/app` updates are gone; system ThemeManager/WallpaperT/BaiduInput take effect; ContactsProvider databases rebuild.
- The sensor stack is in vendor; flashing only system will not include it.

### 4.1 System partition expansion and GPT cautions (important)

> **The smallest partition that must never be deleted: `PST` (512 KiB).** It stores ChromeOS/bootloader persistent state; deleting or recreating it corrupts that state and **can make the device unbootable**. `MSC` (4 MiB, `/misc`) must also be kept because it holds the BCB. Expand only APP/UDA; leave every other small partition untouched.

**The stock Pixel C APP/system partition is only 3.5 GiB:**

```text
3,758,096,384 B = 3.5 GiB
```

The `system.img` in this folder is **8,589,934,592 B = 8 GiB**, built for an expanded APP partition. **Do not `dd` it onto a stock 3.5 GiB APP partition**: `dd` does not stop at the partition boundary. It will continue into UDA, destroy data, and can make the device unbootable.

Current layout on this device:

| Partition | Device node | Size |
|---|---|---:|
| APP/system | `mmcblk0p4` | 8,589,934,592 B = 8 GiB |
| VNR/vendor | `mmcblk0p5` | 318,767,104 B = 304 MiB |
| UDA/data | `mmcblk0p7` | Shrunk from the original ~24.7 GiB to make room for APP |

**How it was expanded:** Pixel C's eMMC was almost fully allocated. Expanding APP from 3.5 GiB to 8 GiB requires **rewriting the GPT partition table**: move the APP end forward and move the UDA start forward (taking space from the beginning of UDA). This is not just `resize2fs`; it changes the partition table.

**Cautions you must keep:**

1. **Back up GPT before touching the partition table.** In TWRP:
   ```sh
   sgdisk --print /dev/block/mmcblk0
   sgdisk --backup=/sdcard/gpt-$(date +%Y%m%d-%H%M).bin /dev/block/mmcblk0
   ```
   Copy the backup off-device. **Never flash a GPT backup from another Pixel C.** GPT contains this device's offsets/GUIDs; a wrong GPT can make the bootloader unable to find KERN-A/APP and the device will not boot.
2. **Back up UDA/data first.** Shrinking UDA moves its start; the old data may not mount. A data format is normally required after expansion.
3. **Do not use a full `flashall` or another ROM's GPT.** Change only APP/UDA; keep KERN-A, KERN-B, CAC and VNR intact.
4. **Confirm the active boot slot is KERN-A.** Pixel C has KERN-A/KERN-B; this system boots from KERN-A. `fastboot flash boot` normally writes KERN-A; do not flash KERN-B and assume it took effect.
5. **Check the APP size before writing system:**
   ```sh
   adb shell blockdev --getsize64 /dev/block/mmcblk0p4
   # must be 8589934592
   ```
   If it is not 8 GiB, do not write this 8 GiB image.
6. **Verify after writing:**
   ```sh
   adb shell blockdev --getsize64 /dev/block/mmcblk0p4
   adb shell blockdev --getsize64 /dev/block/mmcblk0p5
   adb shell sync
   ```
7. **Symptoms of a wrong partition table:** stuck at LOGO, black screen, fastboot works but Android cannot mount `/system`, or `/data` fails to mount/decrypt. Do not keep formatting; restore this device's GPT backup first, then reflash boot/vendor/system.
8. **`/dev/block/by-name/system` may be missing or may point to a different layout.** If unsure, use `mmcblk0p4` (APP) and `mmcblk0p5` (VNR); never use the whole disk `/dev/block/mmcblk0` as a `dd` output.

**Small partitions that must not be deleted (the “tiny placeholder”):**

| Partition | Size | Role / warning |
|---|---:|---|
| `MSC` | 4 MiB | Android `/misc`, holds the bootloader control block (BCB). **Do not `fastboot erase misc` and do not delete it while repartitioning.** Without it, recovery/reboot-to-bootloader commands can fail; depthcharge may still allow a normal boot on a misc read error, but it reports a disk error. |
| `PST` | 512 KiB | ChromeOS persistent state, the smallest partition. **Deleting it causes a bootloader state error and can make the device unbootable.** Keep it untouched; do not delete or format it. It is far too small to hold system/vendor anyway. |
| `MD1` | 64 MiB | Android `/metadata`, used for metadata encryption. Deleting/recreating it can break data decryption, OTA and some system services. |
| `LNX` | 32 MiB | Boot/platform reserved partition. Do not touch. |
| `KERN-A` / `KERN-B` | 32 MiB each | The two boot slots. This system boots from KERN-A; do not delete KERN-B just to save space. |

On this device, GPT reports `MSC` as `mmcblk0p10` (4096 KiB) and `PST` as `mmcblk0p11` (512 KiB, the last partition on the disk). The first bytes of `PST` are non-zero, so it is not an empty placeholder — it contains bootloader/persistent state. **Do not wipe it.**

When expanding, **change only APP and UDA**. Keep `MSC`, `PST`, `MD1`, `LNX` and `KERN-A/B` exactly as they are. Do not “generate a brand-new GPT” and recreate every partition from scratch; adjust the APP/UDA boundaries in the existing GPT. That way, if expansion fails, KERN-A and the small partitions are still intact for recovery.

**About `/data`, FBE and `fastboot -w`:** In MoKee/MIUI fstab, `/data` uses `forcefdeorfbe=.../by-name/MD1,quota`, i.e. **FBE encryption with metadata stored in MD1**. `fastboot -w` is not just “format data”; it erases/recreates data, cache and metadata (MD1). Therefore:

- After `-w`, TWRP cannot decrypt the old data — the old FBE keys/state are gone.
- If a second format in TWRP is needed before `/data` mounts, that is expected: it clears the stale FBE/metadata state.
- Do not use `-w` if you need to keep old data; when switching ROMs/FBE state, `-w` is the clean path.
- `/system` failing to mount is usually not “system encryption”; it is more likely ext4 state, a mount-point conflict, or a partition-table/image mismatch. Do not confuse it with MD1/FBE.

## 5. Known Limitations

- EMC max is 1600 MHz; 1862/2133 need new BCT/EMC timing data and are not included.
- Magnetometer data is present but not calibrated; compass accuracy may be poor.
- When all sensors are disabled, the EC may still sample at ~400 Hz; this can cost battery.
- MiuiHomePadS gesture region bug may still affect bottom gesture navigation; 3-button navigation is the safest workaround.
- The sensor stack in the included `vendor.img` was integrated manually. Rebuilding vendor from the LOS device tree without adding the modules/HAL/policy will lose it.
- ThemeManager/WallpaperT are platform-signed modified APKs; future system updates may overwrite them.
- Baidu input is user-signed; select it as default after data wipe.
- Google suite remains installed; do not remove it if you need Play services.

## 6. Credits

- Thanks to **@npjohnson** and **@Electimon**: the boot + vendor images used here come from the **LineageOS 20–22** open-source projects they maintain, and this project's boot and vendor are modified from that base.
- Thanks to **@汽水°** for the **8GB SPI (bootloader)**.
- Thanks to **@lumingyu0423** for **[Pixel-C_custom_bootloader](https://github.com/lumingyu0423/Pixel-C_custom_bootloader/blob/main/README.md)**.
- Pixel C / Tegra X1 community, LineageOS 20 vendor, hekate/T210 8 GB SDRAM references.
- Xiaomi Pad 5 (nabu) HyperOS base.
- All testing and image validation on actual Pixel C 8 GB hardware.

---

*End of document.*
