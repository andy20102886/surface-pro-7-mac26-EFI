# Surface Pro 7 Hackintosh macOS 26 Tahoe — 完整调试记录（合并版）

> 合并时间: 2026-07-03 21:15 CST
> 来源: Windows 端 WorkBuddy 调试记录 + macOS 端 WorkBuddy 调试总结 + EFI 版本变更日志
> 设备: Surface Pro 7 (i5-1035G4 / Iris Plus G4 0x8A5A / 8GB / 256GB)
> macOS: 26.5 Tahoe (25F71)
> OpenCore: r1.0.5

---

## 一、阶段 1：初始安装与禁止符号（2026-06-29 ~ 2026-07-01）

### 1.1 首次尝试 — balopez83 v2.5.0 EFI（I 盘）

| 时间 | 事件 | 结果 |
|------|------|------|
| 2026-06-29 | 用户用 balopez83/Surface-Pro-7-Hackintosh v2.5.0 EFI 引导安装 macOS 26 | ⛔ **禁止符号** |
| 2026-06-29 | 分析: SMBIOS=MacBookPro13,3 不在 macOS 26 支持列表 | — |
| 2026-06-29 | 修改: SecureBootModel=Disabled→x86legacy, boot-args 加 amfi=0x80, APFS MinDate/MinVersion=0 | ❌ 连 OC 都进不去 |
| 2026-06-29 | 恢复备份: config.plist.backup_20260701_042200 | — |
| 2026-06-29 | 根因: x86legacy 导致 OC 验证签名，未签名 kext 被拒绝加载 | — |
| 2026-06-29 | 修复: SMBIOS=MacBookPro13,3→**MacBookPro16,3**（macOS 26 原生支持） | ✅ 待用户测试（用户切换到 F 盘 EFI） |

### 1.2 换用 OpCore-Simplify EFI（F 盘）

| 时间 | 事件 | 结果 |
|------|------|------|
| 2026-07-01 ~19:19 | 用户改用 OpCore-Simplify 生成的 EFI（`F:\EFI\OC\`） | macOS 26 能进设置界面 |
| 2026-07-01 ~19:20 | **分析发现**: AAPL,ig-platform-id=`0200518a`(LE)=`0x8A510002`(BE) — 不是标准 Apple Ice Lake framebuffer! | ❌ 屏幕严重闪烁, 黑屏 >80% |
| 2026-07-01 ~19:20 | SMBIOS=MacBookAir9,1 ✅ | macOS 26 兼容 |
| 2026-07-01 ~19:20 | Lilu=1.7.3（OpCore 定制）, WEG=1.7.1（OpCore 定制） | — |

---

## 二、阶段 2：GPU 调试 — 崩溃到 VESA 安全模式（2026-07-01 19:19 ~ 22:27）

### v1 → v2：修复 framebuffer + Lilubetaall

| 版本 | 时间 | 修改 | 结果 |
|------|------|------|------|
| **v1** | 19:19 | OpCore-Simplify 初版 (framebuffer=8A510002, SMBIOS=MacBookAir9,1, Lilu 1.7.3/WEG 1.7.1) | ❌ 屏幕闪烁 |
| **v2** | 20:20 | framebuffer: 8A510002→**00009B3E**; boot-args: +**-lilubetaall** | ⚠️ 闪烁修复但触发了 WEG→AppleIntelICLGraphics kernel panic |

### v3：开启深度日志

| 版本 | 时间 | 修改 | 结果 |
|------|------|------|------|
| **v3** | 20:46 | Debug: Target=3→67, DisplayLevel=max, AppleDebug/Panic=true, boot-args: +-liludump all | 📋 **确认 crash：WhateverGreen→AppleIntelICLGraphics** |

### v4：VESA 安全模式 — 首次进系统

| 版本 | 时间 | 修改 | 结果 |
|------|------|------|------|
| **v4** | 22:27 | boot-args: +**-igfxvesa**（VESA模式）, debug=0x100→**0x104**（禁止自动重启）, 删 -igfxblr; SMBIOS=MacBookAir9,1 | ✅ **首次进入 macOS 设置界面** |
| | 22:27 | 问题: 2736x1824 VESA 下字极小, CPU 软件渲染过热, 最终卡死 | — |
| | 22:29 | 开启 UIScale=2（HiDPI 大字模式） | ✅ 字变大可操作 |

### v5：尝试恢复 GPU 加速（官方 kext）

| 版本 | 时间 | 修改 | 结果 |
|------|------|------|------|
| **v5** | 22:50 | Lilu: 1.7.3→**1.7.2**（官方）, WEG: 1.7.1→**1.7.0**（官方）; 去 -igfxvesa | ❌ **AppleIntelICLGraphics 仍 kernel panic** |

### 崩溃根因发现：#1 — 缺少 fbmem/stolenmem 补丁

| 时间 | 事件 | 结果 |
|------|------|------|
| 22:52 | 回顾: 删 WEG 后原生驱动也崩溃（IOAcceleratorFamily2 panic） | — |
| 22:52 | 加入 framebuffer-patch-enable, fbmem=0x9000, stolenmem=0x3001, device-id 注入 | ❌ 仍 panic |
| 22:54 | **发现崩溃真正元凶**: `-noDC9` boot-arg! 删掉后用 WEG+IGLVESA 不崩溃了 | ✅ **WEG 加载无 crash** |
| 22:54 | 但去 -igfxvesa 后仍黑屏 | — |

---

## 三、阶段 3：跨系统协作调试（2026-07-02 04:00 ~ 23:00）

### 3.1 macOS 端修改 config（macOS WorkBuddy）

| 时间 | 事件 | 结果 |
|------|------|------|
| ~04:00 | macOS 内 WorkBuddy 修改: ig-platform-id→**0x8A520000**, SMBIOS→**MacBookPro16,2**, 删 -igfxvesa, 注 device-id=0x8A52 | ⚠️ 代码跑完黑屏（无panic） |
| ~05:00 | **Reset NVRAM 后修复成功!** VRAM 19MB→1536MB, Metal 3 | ✅ **GPU 加速首次成功** |
| ~05:00 | ig-platform-id 优化: 0x8A520000→**0x8A5A0000**（原生device-id）, 无需 device-id 注入 | ✅ **正确 framebuffer** |

### 3.2 Windows 端误操作与修正

| 时间 | 事件 | 结果 |
|------|------|------|
| 19:33 | Windows WorkBuddy 将 framebuffer 恢复为 07009B3E（错误!） | ❌ 黑屏 |
| 19:40 | 发现 EFI/OC/.contentVisibility="Disabled"→OC 不显示启动项 | ✅ 修正为 Auxiliary |
| 20:12 | 发现 macOS 工具改了 SMBIOS+device-id, 又改回去 | 来回拉扯 |
| 22:34 | 恢复 SMBIOS=MacBookAir9,1, 去 -igfxvesa | ❌ WEG panic |
| 22:43 | **加入 device-id=0x8A53, 关掉 WEG 所有 ICL 补丁** | ❌ 仍 panic |
| 22:54 | 删 WEG — 原生 ICLGraphics 也崩 | ❌ IOAcceleratorFamily2 panic |

### 3.3 上网调研

| 时间 | 事件 | 结果 |
|------|------|------|
| 22:54 | 搜索社区方案: balopez83 仓库 + 远景论坛 + Dortania 文档 | 📋 **Dortania: 删 WEG 全删**; 远景 laobamac: 定制WEG |
| 22:56 | 尝试 balopez83 v12: 删 WEG | ❌ 原生也崩 |
| 22:57 | v13: 恢复 WEG - 去 -noDC9 + 保留 -igfxvesa | ✅ **WEG 不崩了 — 证明元凶是 -noDC9** |
| 23:07 | v14: 去 -igfxvesa → 黑屏（WEG 不崩但显示无输出） | ⚠️ 非崩溃型失败 |
| 23:23 | **读取 macOS 诊断报告** → 发现 macOS WorkBuddy 成功配置 | — |
| 23:25 | **正确 framebuffer 解析**: 用户 i5-1035G4 的 device-id 是 **0x8A5A**, 不是 0x8A52! | — |
| 23:31 | v15: 恢复 macOS WorkBuddy 配置 (8A520000 + MBP16,2 + 无 -igfxvesa) | ❌ 黑屏 |

### 3.4 转折点 — balopez83 原版 EFI

| 时间 | 事件 | 结果 |
|------|------|------|
| 23:54 | 用户下载 balopez83 v2.4.2 EFI 到本地 | — |
| 23:55 | **分析 balopez83 i5 config**: ig-platform-id=**AABaig==(0x8A5A0000)** — 完全匹配用户 GPU! | 💡 **突破!** |
| 23:55 | 关键差异: i5 config 使用 **0x8A5A0000**（真实 G4 device-id）, i7 config 用 0x8A520000 | — |
| 23:55 | **缺失 3 个关键属性**: igfxfw=2(GPU 固件), rps-control=1(电源状态), enable-backlight-registers-fix | — |
| 23:56 | 其他差异: framebuffer-patch-enable=**integer 1**(非 data), stolenmem=**0x3C000**(非 0x3001), 无 fbmem | — |
| 23:58 | v16: **完整复制 balopez83 i5 config** 到 F 盘 | — |
| ~23:59 | 用户测试 | ✅ **画面正常! GPU 加速成功!** |

---

## 四、阶段 4：声卡调试（2026-07-02 23:30 ~ 2026-07-03 21:00）

### 4.1 AppleALC 方案

| 时间 | 事件 | 结果 |
|------|------|------|
| 2026-07-02 23:30 | v17: 添加 AppleALC.kext + layout-id=35（balopez83 值）+ UEFI Audio 配置 | ❌ 无声 |
| 2026-07-03 00:34 | **根因查明**: HDAS class-code=0x040380(Intel SST) ≠ 0x040100(标准 HDA) | — |
| 2026-07-03 00:35 | class-code伪装: 注入 0x040100 + device-id=0x9DC8 | ❌ 无效 |
| 2026-07-03 01:00 | **深层根因**: macOS 26 完全移除了 AppleHDA.kext 和 AppleHDAController | ❌ AppleALC 无依赖可绑 |
| 2026-07-03 01:30 | 清理无效伪装, config 恢复干净 | — |

### 4.2 OCLP Root Patch 方案

| 时间 | 事件 | 结果 |
|------|------|------|
| 2026-07-03 01:00 | 写入 OCLP 修复指南 + SIP=>03000000 | — |
| 2026-07-03 15:00 | 手动安装 macOS 15 AppleHDA.kext | ❌ 内核拒绝加载 |
| 2026-07-03 16:00 | OCLP 2.0.1 → "No Patches Required" | ❌ SMBIOS 白名单拦截 |
| 2026-07-03 16:30 | OCLP-Plus → 同样 "No Patches Required" | ❌ |

### 4.3 VoodooHDA 方案

| 时间 | 事件 | 结果 |
|------|------|------|
| 2026-07-03 17:00 | 编译安装 VoodooHDA 3.0.3 | ✅ **加载成功** |
| 2026-07-03 20:45 | 用户反馈: 有声, 音量键可用, 麦克风可用 | ⚠️ 音量略小 |

---

## 五、阶段 5：后续修复（2026-07-03）

### 5.1 亮度调节

| 时间 | 事件 | 结果 |
|------|------|------|
| 20:35 | 下载 SSDT-PNLF.aml（Dortania 通用版）, 加入 ACPI/Add | — |
| 20:41 | 用户确认亮度可调 | ✅ **已修复** |

### 5.2 CPUFriend 降温 — 失败

| 时间 | 事件 | 结果 |
|------|------|------|
| 19:30 | 开始实施 CPUFriend ResourceConverter.sh + 修改频率向量 | — |
| 19:31 | LFM=800→1200MHz, HighFrequencyLimit=2400 | — |
| 19:33 | 安装到 EFI + config.plist | — |
| 20:21 | **无法启动** | ❌ 失败 |
| 20:24 | 用户自行恢复, 删除所有 CPUFriend 文件 | ✅ 恢复 |

### 5.3 合盖睡眠

| 时间 | 事件 | 结果 |
|------|------|------|
| 20:50 | 合盖测试 → 屏幕熄灭无法唤醒 | ❌ 不可用 |
| 20:52 | 诊断: AppleACPILid 存在但 Surface 睡眠后无法唤醒 | — |
| 20:53 | 建议: `sudo pmset lidwake 0 sleep 0` | ⏸ 待用户执行 |

---

## 六、版本迭代总表（17 个版本）

| 版本 | 时间 | 关键修改 | 结果 |
|------|------|----------|------|
| v1 | 07-01 19:19 | OpCore-Simplify 初版 (8A510002, MacBookAir9,1, Lilu1.7.3/WEG1.7.1) | ❌ 屏幕闪烁 |
| v2 | 07-01 20:20 | framebuffer 8A510002→00009B3E, +-lilubetaall | ⚠️ WEG panic |
| v3 | 07-01 20:46 | 深度日志 (Target=67, AppleDebug=true) | 📋 确认 WEG→ICL crash |
| v4 | 07-01 22:27 | +-igfxvesa, debug=0x104, 删 -igfxblr | ✅ 进设置界面 |
| v5 | 07-01 22:50 | Lilu=1.7.2, WEG=1.7.0 (官方), 去 -igfxvesa | ❌ WEG panic |
| v6 | 07-02 ~04:00 | macOS WorkBuddy: 8A520000+MBP16,2+device-id | ⚠️ 黑屏(非panic) |
| v7 | 07-02 19:33 | framebuffer 8A520000→07009B3E, 删 device-id | ❌ 黑屏 |
| v8 | 07-02 20:12 | +-igfxvesa, EnableWriteUnprotector=true, 删 device-id | ✅ VESA稳定 |
| v9 | 07-02 22:34 | SMBIOS MacBookPro16,2→MacBookAir9,1, 去 -igfxvesa | ❌ WEG panic |
| v10 | 07-02 22:46 | device-id=0x8A53注入, 关闭 ICL 补丁 | ❌ WEG panic |
| v11 | 07-02 22:53 | 恢复 VESA 稳定版 (-igfxvesa) | ✅ 稳定 |
| v12 | 07-02 22:57 | **删WEG** → 原生驱动 | ❌ IOAccelerator panic |
| v13 | 07-02 23:07 | 恢复WEG, **删 -noDC9**, 保留 -igfxvesa | ✅ **WEG不崩了!** |
| v14 | 07-02 23:14 | 去 -igfxvesa (恢复加速尝试) | ❌ 黑屏 |
| v15 | 07-02 23:35 | 恢复 macOS WorkBuddy 配置 (8A520000+MBP16,2) | ❌ 黑屏 |
| **v16** | **07-02 23:58** | **balopez83 i5 config 完整复制** (0x8A5A0000, igfxfw, rps-control, backlight-fix, stolenmem=0x3C000) | ✅ **GPU加速成功!** |
| v17 | 07-03 00:18 | +AppleALC+layout-id=35 (音频) | ⚠️ 无声但已尽力 |

---

## 七、关键根因总结

### 7.1 GPU 加速 — 找出 3 层嵌套问题

```
第1层 (最浅): OpCore-Simplify 生成了错误 framebuffer 0x8A510002
    → 屏幕闪烁
    修复: 改为 00009B3E

第2层 (中间): -noDC9 boot-arg 导致 WhateverGreen 触发 ICLGraphics kernel panic
    → WEG kernel panic
    修复: 删除 -noDC9

第3层 (最深): macOS WorkBuddy 给 i5 配置用了 i7 的 framebuffer 0x8A520000
    → 黑屏 (无panic)
    最终修复: balopez83 i5 原生 framebuffer 0x8A5A0000 + igfxfw=2 + rps-control=1
```

### 7.2 声卡 — 不可逾越的架构问题

```
AppleALC:       需要 AppleHDA.kext → macOS 26 已删除 → ❌
class-code伪装:  需要 AppleHDAController → macOS 26 已删除 → ❌  
OCLP Root Patch: MBP16,2 SMBIOS 在白名单 → 跳过 → ❌
VoodooHDA:      绕过 HDA 框架, 直接驱动 SST → ✅ 有声但音量略小
```

---

## 八、当前驱动状态（最终）

| 组件 | 状态 | 备注 |
|------|------|------|
| GPU (Iris Plus G4 0x8A5A) | ✅ **正常** | Metal 3, 1536MB VRAM, 2736x1824 HiDPI |
| 亮度调节 | ✅ **已修复** | SSDT-PNLF 通用版 |
| WiFi (Intel AX201) | ✅ **正常** | itlwm 2.3.0 + HeliPort |
| 蓝牙 | ✅ **正常** | A2DP/HFP, 可连耳机音箱 |
| 触摸屏/手写笔 | ✅ **正常** | BigSurface + VoodooInput |
| NVMe SSD | ✅ **正常** | NVMeFix 1.1.4 |
| USB | ✅ **正常** | USBToolBox 1.2.0 |
| 传感器 | ✅ **正常** | SMCProcessor + SMCSuperIO |
| CPU 电源管理 | ✅ **正常** | XCPM, 400MHz-3.7GHz |
| 电池 | ✅ **正常** | 78% 健康, 388 循环 |
| 声卡(内置) | ⚠️ **可用** | VoodooHDA 3.0.3, 有声+唛, 音量略小 |
| 音量键 | ✅ **可用** | Fn+F3/F4 |
| CPU 温度 | ⚠️ **偏高(~90°C)** | BigSurface + WindowServer 导致 |
| 合盖睡眠 | ❌ **不可用** | 睡眠后无法唤醒 |
| 摄像头 | ❌ **不可用** | I2C 无 macOS 驱动 |
| 键盘背光 | ❌ **不可用** | Surface 特有 |
| AirDrop | ❌ **不可用** | itlwm 不支持 awdl0 |

---

## 九、关键命令集

```bash
# 显卡检测
system_profiler SPDisplaysDataType | grep -E "Metal|VRAM|Chipset"

# 全驱动检测
kextstat | grep -E "Intel|Lilu|Whatever|AppleALC|Voodoo|BigSurface"

# CPU/电源
sysctl machdep.cpu.brand_string
pmset -g

# 声卡
sudo defaults write /Library/Preferences/VoodooHDA OutputGain -int 85
sudo kextunload /Library/Extensions/VoodooHDA.kext
sudo kextload /Library/Extensions/VoodooHDA.kext

# 禁用合盖睡眠（解决唤醒问题）
sudo pmset -a lidwake 0 sleep 0 displaysleep 1
```

---

## 十、文件清单

### EFI 分区 (`F:\EFI\`)

| 文件 | 作用 |
|------|------|
| `workbuddy-debug.log` | Windows 端调试记录 |
| `OC/v17 声卡修复指南.txt` | OCLP 声卡修复操作步骤 |
| `OC/版本/CHANGELOG.md` | 17 个版本完整迭代记录 |
| `OC/版本/config_v*.plist` | 17 个版本 config 原始文件 |

### 工作区

| 文件 | 作用 |
|------|------|
| `Surface_Pro7_macOS26_完整调试总结.md` | 本文 |

---

*本文件由 Windows 端 WorkBuddy 与 macOS 端 WorkBuddy 协作生成*
*跨越 4 天、17 个配置版本、100+ 次重启调试*
*2026-07-03 21:15 CST*
