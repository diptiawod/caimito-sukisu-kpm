# caimito-sukisu-kpm

GitHub Actions 云端构建 **Pixel 9 (tokay / caimito / Tensor G4 zumapro)** 的 GKI 内核,集成 **SukiSU-Ultra v4.1.3(编入内核)+ `CONFIG_KPM=y` + SUSFS**,用于自有设备的内核级研究与 root。参照本人 [`raviole-sukisu-kpm`](https://github.com/diptiawod/raviole-sukisu-kpm)(Pixel 6 Pro)的 CICD 模板适配而来。

- **源码**:`android-gs-caimito-6.1-android16`,KMI `android14-6.1`。
- **构建**:Kleaf `build_caimito.sh`,`BUILD_AOSP_KERNEL=1` 从源码编 `aosp/`(含 SukiSU/KPM),关 KMI 裁剪保持 CRC 不变、原厂 vendor 模块照常加载。
- **产物**:`boot_kpm.img`(已打 KPM patch,可直接刷)、`boot.img`(未打 KPM)、`Image`。

## ★ 本次核心修复:`CONFIG_RFKILL=y`

自编内核曾把 `CONFIG_RFKILL` 从原厂 `=y` 降成 `=m`,导致:开机 `modules.load` 不加载 `rfkill.ko` → 内核缺 `rfkill_blocked` 符号 → 原厂 `cfg80211.ko` / `bcmdhd4390.ko`(WiFi 主驱动)无法加载 → 没有 `wlan0` → **WiFi 开关打开却搜不到任何网络列表**。改回 `=y`(内置)根治。

## 处理的已知坑(全部在 `build.yml` 对应步骤)

| 坑 | 处理 |
|---|---|
| `use_prebuilt_gki` 默认 true → 只用预编译 GKI、忽略改动 | 改所有 `*.bazelrc` 为 false + `.d6.bazelrc` 追加 `build:caimito --use_prebuilt_gki=false` |
| `setup.sh` 建符号链接 `drivers/kernelsu`,Kleaf 沙箱 `include/uapi` 悬空 → `feature.h not found` | 删符号链接,`cp -rL` 真实拷贝 |
| `KSU_VERSION` 由 git 提交数算,沙箱算错 → 与管理器 v4.1.3(40796)不匹配、KPM 页被隐藏 | 钉死 `KSU_VERSION := 40796` |
| `check_defconfig` 拒绝非规范格式的追加 config | `build.config*` 里的调用中和为 `true` |

## 刷机(重要)

**Pixel 9 上 `fastboot boot`(临时 RAM 启动)不工作**——连原厂内核都拒绝,会掉进 "cannot load android system" recovery(重启即回原厂,无损)。必须刷入:

```bash
fastboot flash boot boot_kpm.img
fastboot reboot
```

只刷了当前使用的 slot。**系统 OTA 会覆盖 boot / 破坏 root,每次 OTA 后需重新构建并重刷。**

刷后验证 WiFi 修复:

```bash
adb shell su -c "grep -w rfkill_blocked /proc/kallsyms"   # 应有输出
adb shell "ls /sys/class/net/ | grep wlan"                # 应有 wlan0
adb shell cmd wifi status                                  # 应为 enabled
```

## 触发构建

Actions → **"Build caimito SukiSU+KPM kernel"** → Run workflow。可覆盖 manifest 分支 / SukiSU tag / `KSU_VERSION` / Magisk tag。

> 仅用于本人自有测试设备的授权研究。
