# Z790 Hackintosh — OpenCore EFI

适用于 **Z790 + Intel 12 代 + AMD RX 6000 系列** 的 OpenCore 配置，运行 macOS 26 Tahoe。
这份 EFI 是从零调试出来的可用配置，包含 WiFi、蓝牙、声卡、睡眠、显卡加速的完整解决方案。

> **三码已脱敏**：仓库里的序列号、MLB、UUID、ROM 都是占位符，直接使用前必须换成你自己的，
> 否则 iMessage、FaceTime、App Store 登录都会失败。详见 [换三码](#️-使用前必须换三码)。

---

## 硬件配置

| 部件 | 型号 |
|---|---|
| 主板 | CVN Z790M FROZEN V20
| CPU | Intel Core i5-12600KF 
| 显卡 | AMD Radeon RX 6700 XT 12GB 
| 无线 | Intel AX201 
| 有线 | Intel I225-V 
| 声卡 | Realtek ALC897 

---

## 软件版本

| 组件 | 版本 |
|---|---|
| macOS | 26.5.1 Tahoe（Darwin 25.x） |
| OpenCore | 见 `OpenCore.efi` |
| Lilu | 1.7.3 |
| VirtualSMC | 1.3.8 |
| WhateverGreen | — |
| AppleALC | 1.9.8 |
| AirportItlwm | 2.4.0（Tahoe 专用分支） |
| IntelBluetoothFirmware | 2.5.2 |
| IntelBTPatcher | 2.5.2 |
| BlueToolFixup | 2.7.2 |
| RestrictEvents | 1.1.7 |
| NootRX | 1.0.0 |
| SimpleHDA | 1.0.0 |

**boot-args**

```
keepsyms=1 debug=0x100 agdpmod=pikera amfi=0x80 ipc_control_port_options=0 \
alcid=11 revcpu=1 revpatch=cpuname itlwm_he=1 shda-lang=zh_CN
```

---

## ⚠️ 使用前必须换三码

`config.plist` → `PlatformInfo` → `Generic` 里的这四项都是占位符：

| 键 | 当前值 | 说明 |
|---|---|---|
| `SystemSerialNumber` | `XXXXXXXXXXXX` | 序列号 |
| `MLB` | `XXXXXXXXXXXXXXXXX` | 主板序列号 |
| `SystemUUID` | `00000000-0000-0000-0000-000000000000` | |
| `ROM` | `000000000000` | 网卡 MAC（6 字节） |

**生成方法**（任选其一）：

1. **GenSMBIOS**（推荐）— 选机型 `MacPro7,1`，自动生成并可直接写入 config
2. **OpenCore Configurator** → PlatformInfo → 机型选择 `MacPro7,1` → 点「生成」
3. 手动填：序列号需在 [Apple 保修查询](https://checkcoverage.apple.com) 返回「无法验证」才算未注册的可用序列号

`ROM` 建议填你网卡的真实 MAC 地址（十六进制 6 字节），或从 MLB 派生。

> 换完三码后，首次登录 iCloud 建议先确认网卡为 `en0`，否则 FaceTime/iMessage 可能无法激活。

---

## 方案要点

### WiFi — AirportItlwm（非 itlwm + HeliPort）

用的是 [laobamac/itlwm](https://github.com/laobamac/itlwm) 的 **AirportItlwm-Tahoe** 分支，
它适配 macOS 26 的 `IO80211Family` 新接口，能直接出现在系统原生 WiFi 菜单里，**不需要 HeliPort**。

关键点：

- `Kernel → Block` 必须为**空**。之前 WiFi 打不开的真正原因就是这里屏蔽了 `com.apple.iokit.IOSkywalkFamily`，
  驱动加载了却无法工作
- `MinKernel` / `MaxKernel` 设为 `25.0.0` / `25.99.99`
- `itlwm_he=1` 开启 802.11ax（实验性，若出现断流可删掉此参数）

`Kexts/` 里保留了另外两套备用方案（默认禁用）：

- `itlwm.kext` 2.3.0 — 走 IOEthernet 路径，需配合 HeliPort
- `AirportItlwm_Sequoia.kext` — 给 macOS 15 用的旧分支

### 蓝牙 — IntelBluetoothFirmware

AX201 走 USB 通道（挂在 `HS14`），所以 USB 端口映射必须包含该端口，否则蓝牙设备根本不会出现。

三个 kext 的**加载顺序**不能错：

```
IntelBluetoothFirmware  →  IntelBTPatcher  →  BlueToolFixup
```

`MinKernel` 统一设为 `21.0.0`。固件已内嵌在 kext 里（`ibt-19-*.sfi`），不需要额外下载。

### 声卡 — SimpleHDA（Tahoe 关键）

**macOS 26 从 Beta 2 起彻底移除了 `AppleHDA.kext`**，所以 AppleALC 加载了也没有可打补丁的对象，
声卡会完全不工作。这是 Tahoe 上最容易卡住的地方。

解决办法是用 [laobamac/SimpleHDA](https://github.com/laobamac/SimpleHDA)：一个 Lilu 插件，
通过 OpenCore 注入把 AppleHDA 栈补回来，**不改系统卷、不重建内核集合**，系统更新也不会丢。

- 顺序：`Lilu → AppleALC → SimpleHDA`
- `alcid=11` 对应 ALC897
- `shda-lang=zh_CN` 让设备名显示中文

> 不要用 OCLP 根补丁或 MyKextInstaller 装 AppleHDA，会破坏系统卷密封，导致无法更新。

### 显卡 — NootRX

RX 6700 XT 属于 Navi 22，苹果从未官方支持，需要 **NootRX**（WhateverGreen 的衍生分支）才能驱动。
`agdpmod=pikera` 用于解决黑屏问题。

### 睡眠

EFI 侧已配置妥当，系统侧 pmset 建议值（本机实测可用）：

```bash
sudo pmset -a hibernatemode 0     # 不写休眠镜像，纯 S3 睡眠
sudo pmset -a powernap 0          # 关闭 Power Nap，避免睡眠中被唤醒联网
sudo pmset -a tcpkeepalive 0
sudo pmset -a womp 0              # 关闭网络唤醒
sudo pmset -a proximitywake 0     # 关闭 proximity 唤醒，否则手机靠近就亮屏
sudo pmset -a disksleep 0
```

**「碰一下就亮屏」的排查方向**：多半不是配置问题，而是
①`proximitywake` 没关；② 无线鼠标/键盘的 HID 事件在进入睡眠的 15 秒转换窗口内打断了过程；
③ 有程序持有 `PreventUserIdleSystemSleep` 断言（用 `pmset -g assertions` 查）。

---

## 已知问题

这些是目前**没有解决**的，不是配置错误：

| 问题 | 原因 |
|---|---|
| 系统信息里处理器显示 `Unknown` | macOS 26 移除了 `System Information` / `AboutExtension` 里的 `10-Core Intel Xeon W` 等字面量，RestrictEvents 的字符串替换补丁无靶可打。**内核层型号是正常的**，`sysctl machdep.cpu.brand_string` 能读到正确值 |
| 显卡待机 34W（比 Windows 高） | macOS 的 AMDRadeon 驱动对非官方型号的 PowerPlay 表不完整，核心能降到 233MHz 但显存锁在满频 1988MHz。软件层无法调整 |
| AirDrop 不可用 | Intel 网卡在 macOS 下无法支持 AWDL |
| 蓝牙偶尔搜不到设备 | 确认 `HS14` 端口在 USB 映射里，且 `BlueToolFixup` 已加载 |

---

## 目录结构

```
EFI/
├── ACPI/             # SSDT 补丁
├── Drivers/          # HfsPlus / OpenRuntime / OpenCanopy / ResetNvramEntry / apfs_aligned
├── Kexts/            # 全部驱动（含 3 个默认禁用的备用方案）
├── Resources/        # OpenCanopy 主题资源
├── Tools/            # OpenShell.efi
├── OpenCore.efi
├── config.plist      # ★ 已脱敏，使用前换三码
└── oldConfig.plist   # 历史配置存档（同样已脱敏）
```

**Kernel 注入 18 项启用 / 21 项总计**，默认禁用：`IOSkywalkFamily.kext`、`IO80211FamilyLegacy.kext`、`AirportItlwm_Sequoia.kext`。

---

## 配套工具：GPU Monitor

macOS 上没有任何现成软件能完整显示 AMD 显卡的功耗、频率、温度
（实测 Stats 3.0.23 的 GPU 模块只实现了使用率和显存频率两个字段）。

配套的菜单栏小工具直接读 `IOKit` 注册表里的 `PerformanceStatistics` 字典，
**不需要 root、不需要后台守护进程**，每次读取仅 0.7 毫秒。

它已独立成仓库：**[xiaoji12/gpu-monitor](https://github.com/xiaoji12/gpu-monitor)**

```bash
git clone https://github.com/xiaoji12/gpu-monitor.git
cd gpu-monitor && ./build.sh --install
```

---

## 使用方法

```bash
# 1. 克隆
git clone https://github.com/xiaoji12/hackintosh-z790-efi.git

# 2. 换三码（用 GenSMBIOS 或 OpenCore Configurator，机型选 MacPro7,1）

# 3. 备份现有 EFI
cp -R /Volumes/EFI/EFI /Volumes/EFI/EFI.backup

# 4. 覆盖
rsync -a --delete EFI/ /Volumes/EFI/EFI/

# 5. 清 NVRAM（首次切换机型时必须）
#    开机进 OpenCore 菜单 → 空格 → Reset NVRAM
```

> 换机型或换三码后**必须** Reset NVRAM，否则残留的旧序列号会导致 iCloud 登录异常。

---

## 免责声明

黑苹果属于非官方使用方式，可能因硬件差异导致无法启动、数据丢失或硬件损坏。
本配置仅在一台具体机器上验证可用，**不保证在你的硬件上能工作**。
使用前请务必备份原 EFI 与重要数据，并准备好 PE 或第二台电脑用于救援。

刷入前建议用 `ocvalidate` 检查配置：

```bash
ocvalidate EFI/config.plist
```
