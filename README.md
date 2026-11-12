# OpenCore EFI — 华硕天选5 Pro 锐龙版（ASUS TUF Gaming A16 2024 / FA607PV）

适用于 **华硕天选5 Pro 锐龙版（ASUS TUF Gaming A16 2024，型号 FA607PV）** 的黑苹果 EFI，基于 **OpenCore**，运行 **macOS 26 Tahoe**

> 本 EFI 针对**单一机型**定制。其他型号、其他 BIOS 版本或硬件配置（无线网卡、触摸板、EC 布局不同）都必须重新提取 DSDT 后再做适配，请勿直接套用。

***

## 硬件信息

| 部件        | 规格                                                              |
| --------- | --------------------------------------------------------------- |
| Model     | 华硕天选5 Pro 锐龙版（ASUS TUF Gaming A16 2024，FA607PV）                 |
| CPU | AMD Ryzen 9 7940HX，16 核 32 线程，Zen 4，55W 基础功耗 |
| iGPU      | AMD Radeon 610M（RDNA 2，2 CU）                                    |
| dGPU | NVIDIA GeForce RTX 4060 Laptop 8GB GDDR6 |
| Memory | DDR5-5600 SO-DIMM，双通道，2 个插槽（运行在5200MHz）  |
| Storage | WD PC SN560 SDDPNQE-1T00-1102 M.2 2280 PCIe 4.0 NVMe |
| Storage2 | WD_BLACK SN770 1TB × M.2 2280 PCIe 4.0 NVMe |
| Screen | 16 英寸 2560 × 1600，165Hz IPS 内屏（eDP）华星光电CSW1632（MNG007DA5-1） |
| Wireless | MediaTek Wi-Fi 6/6E（PCI `14c3:7961 / 14c3:7922` 系列）MT7921 |
| Bluetooth | MediaTek 蓝牙（与无线网卡组合） |
| Ethernet | Realtek RTL8111/8168 GbE Family |
| Sound | Realtek HD Audio + 独显 HDMI/DP 音频（`layout-id 56`） |
| Touchpad  | Synaptics I2C HID（`\_SB.I2CA.TPD0`，地址 `0x15`，GPIO 中断引脚 `0x08`）  |
| Keyboard | PS/2（内置），Fn 亮度/音量快捷键 |
| Battery | 华硕 EC，内存映射地址 `0xFE700600`，长度 `0x100` |
| Reader    | 不支持                                                             |

软件基线：

- **OpenCore** 1.0.x（config 使用 1.0 系列的 Booter / UEFI Quirks）
- **macOS** 26.x Tahoe
- 图形引导界面：OpenCanopy（主题 `Acidanthera\GoldenGate`）

***

## 目录结构

```
OC/
├── ACPI/
│   ├── SSDT-PLUG-ALT.aml
│   ├── SSDT-EC.aml
│   ├── SSDT-USB-Reset.aml
│   ├── SSDT-USBX.aml
│   ├── SSDT-XOSI.aml
│   ├── SSDT-BATT.aml
│   └── SSDT-Disable_Network_GPP6.aml   （未启用）
├── Drivers/
│   ├── OpenRuntime.efi
│   ├── OpenCanopy.efi
│   ├── HfsPlus.efi
│   └── ResetNvramEntry.efi
├── Kexts/
├── Resources/
├── Tools/         
├── OpenCore.efi
└── config.plist
```

***

## ACPI 说明

| 表                               | 作用                                                                                        |
| ------------------------------- | ----------------------------------------------------------------------------------------- |
| `SSDT-PLUG-ALT.aml`             | 为 16 个处理器对象（`C000`–`C01F`）注入 `plugin-type = 1`，使 XNU 启用正常的 CPU 电源管理。                      |
| `SSDT-EC.aml`                   | 伪嵌入式控制器（`_HID = ACID0001`），通过 `_OSI("Darwin")` 仅在 macOS 下暴露，避免系统枚举 `_STA` 返回 0 的真实 `EC0`。 |
| `SSDT-USB-Reset.aml`            | USB 复位与属性辅助。                                                                              |
| `SSDT-USBX.aml`                 | 为已映射的 USB 端口注入供电属性（充电电流）。                                                                 |
| `SSDT-XOSI.aml`                 | 配合 `_OSI → XOSI` 重命名使用的兼容层。                                                               |
| `SSDT-BATT.aml`                 | **自研电池实现**，详见https://github.com/VisualTechStudio/SSDT-BATT-ASUS。                                                                          |
| `SSDT-Disable_Network_GPP6.aml` | 保留但**未启用**。                                                                               |

### ACPI 补丁（`ACPI → Patch`）

三条电池重命名均限定 **仅 DSDT**（`TableSignature = DSDT`），不会影响其他 ACPI 表：

| 查找     | 替换     | 目的                                 |
| ------ | ------ | ---------------------------------- |
| `_BST` | `XBST` | 释放名称，使 `SSDT-BATT` 可以定义自己的 `_BST`。 |
| `_BIF` | `XBIF` | 同上，用于 `_BIF`。                      |
| `_PSR` | `XPSR` | 同上，用于交流电检测 `_PSR`。                 |

`_OSI → XOSI` 重命名作用于全部表，需配合 `SSDT-XOSI.aml`。

### `SSDT-BATT.aml`

原生 DSDT 通过 `Field (IT55, ByteAcc, Lock, Preserve)` 读取电池。macOS 下这类带 `Lock` 限定的 EC 字段常常读回 0，因此常见的"重命名后委托给 `XBST()`"方案会得到错误的电量（1% 或卡在 100%）。

本 SSDT 完全绕开 DSDT 字段：

- 自行声明 `OperationRegion (ZBT0, SystemMemory, 0xFE700600, 0x0100)`，以 **`NoLock`** 方式直接读取原始 EC 偏移：
  - `0x42` bit 4 —— 充电中标志；`0x43` bit 0 —— 放电中标志
  - `0x4E` —— 电量百分比（%）
  - `0x53/0x54` —— 当前电压（mV）
  - `0x55/0x56` —— 当前电流（mA）
  - `0x57/0x58` —— 满充容量（mAh）
  - `0x5F/0x60` —— 设计容量（mAh）
  - `0x63/0x64` —— 设计电压（mV）
- 直接从根级 `EXBU` 区域（`0x96CBF018`，偏移 `0x13`）读取交流电状态 `ACPF`。
- 在 `\_SB.PCI0.SBRG.BAT0` / `ADP0` 下实现 `_BIF`、`_BST`、`_PSR`，返回符合规范的包，使 `AppleSmartBattery`（经 `VirtualSMC` + `SMCBatteryManager` + `ECEnabler`）能正确显示电量、剩余时间和电源状态。

***

## Kext 清单
| ------------------------------------------------ | ----------- | ---------------------------------------------------------- |
| Kext                                                      | 版本          | 作用                                                        |
| --------------------------------------------------------- | ----------- | --------------------------------------------------------- |
| Lilu                                                      | 1.7.3       | 补丁框架，以下多数 kext 依赖它。                                       |
| VirtualSMC                                                | 1.3.9       | SMC 模拟（启动必需）。                                             |
| SMCBatteryManager                                         | 1.3.8       | 向 `AppleSmartBattery` 上报电池与电源状态。                          |
| ECEnabler                                                 | 1.0.6       | 允许多字节 EC 字段读取。                                            |
| SMCAMDProcessor                                           | 1.0.1       | Ryzen 处理器传感器与功耗数据。                                        |
| AMDRyzenCPUPowerManagement                                | 0.7.2       | Zen 4 P-State 管理。                                         |
| AppleALC                                                  | 1.9.9       | HD Audio 布局注入（`layout-id 56`）。                            |
| AppleMCEReporterDisabler                                  | 1.2         | 禁用 `AppleMCEReporter`（与 AMD 不兼容）。                         |
| RestrictEvents                                            | 1.1.7       | CPU 名称 / 内存 / 主板信息修正。                                     |
| ForgedInvariant                                           | 1.5.0       | AMD 的 `cpuid` / 不变量 TSC 修正。                               |
| NVMeFix                                                   | 1.1.4       | NVMe 电源管理与 APST 修正。                                       |
| VoodooPS2Controller（含 Keyboard/Mouse/Trackpad）            | 2.3.8       | 内置 PS/2 键盘与 Fn 快捷键。                                       |
| VoodooI2C（含 VoodooI2CServices / VoodooGPIO / VoodooInput） | 2.9.1       | I2C-HID 传输与 GPIO 驱动。                                      |
| VoodooI2CHID                                              | 1.0         | Synaptics I2C 触摸板（多点触控手势）。                                |
| USBToolBox + UTBDefault                                   | 1.2.0 / 1.0 | USB 端口映射。                                                 |
| RealtekRTL8111                                            | 2.4.2       | 千兆有线网卡。                                                   |
| AirportMtkWlan                                            | 1.5.0       | MediaTek 无线网卡（AirportItlwm 分支，Tahoe 版）。                   |
| SimpleMtkBtFw / SimpleMtkBtPatcher                        | 1.0.0       | MediaTek 蓝牙固件上传/补丁。                                       |
| BlueToolFixup                                             | 2.7.3       | 蓝牙协议栈修正。                                                  |
| BrightnessKeys                                            | 1.0.4       | Fn 亮度键映射。                                                 |
| SMCLightSensor                                            | 1.3.9       | 环境光传感器。                                                   |
| AMFIPass                                                  | 1.4.x       | AMFI 放行（配合 `amfi_get_out_of_my_way=0x1`）。                 |
| NMHDA                                                     | 1.0.0       | 注入 HDA 控制器 device-id，使独显 HDMI 音频绑定到 `AppleHDAController`。 |
| WhateverGreen                                             | 1.7.2       | **因iGPU不可驱动故未启用**。                                                  |

启动参数：
-v amfi_get_out_of_my_way=0x1 amfi=0x80 ipc_control_port_options=0
```

`Misc → Debug` 已设为 `Target = 0`、`DisplayLevel = 0`、`AppleDebug = false`，以减少启动日志开销；仅保留 `-v` 便于排错。

`Misc → Debug` 已设为 `Target = 0`、`DisplayLevel = 0`、`AppleDebug = false`，以减少启动日志开销；仅保留 `-v` 用于排错。

***

## 当前驱动状态
| -------------------- | -- | --------------------------------------------------------- |
| 功能                   | 状态 | 说明                                                |
| -------------------- | -- | ------------------------------------------------- |
| CPU（16 核 32 线程）+ SMT | √  | `SSDT-PLUG-ALT` + `AMDRyzenCPUPowerManagement`。   |
| CPU 电源管理 / 传感器       | √  | `SMCAMDProcessor`。                                |
| 电池电量、剩余时间、电源状态       | √  | 自研 `SSDT-BATT`。                                   |
| 内置键盘 + Fn 快捷键        | √  | `VoodooPS2Controller`。                            |
| 触摸板（多点触控）            | √  | Synaptics I2C-HID，经 `VoodooI2C` / `VoodooI2CHID`。 |
| 内置扬声器 / 耳机           | √  | `AppleALC`，`layout-id 56`。                        |
| HDMI/DP 音频           | √  | `NMHDA` device-id 注入。                             |
| 亮度调节（Fn 键）           | √  | `BrightnessKeys` + `SMCLightSensor`。              |
| 有线网络                 | √  | `RealtekRTL8111`。                                 |
| USB 端口 + 内置摄像头       | √  | `USBToolBox` 端口映射。                                |
| NVMe 硬盘              | √  | `NVMeFix`。                                        |
| Wi-Fi                | √  | `AirportMtkWlan`。                                 |
| 蓝牙                   | √  | `BlueToolFixup`。                                 |
| iGPU               | ×  | 无法驱动。     |
| dGPU        | √  | 已驱动。 详见 https://github.com/nullmoth/nvidia-macos-driver   |
| 睡眠 / 唤醒              | ❓  | 尚未验证。                                             |
| 读卡器                  | ×  | 未包含驱动。                                            |
| 隔空投送 / 接力            | ×  | 需要受支持的苹果兼容无线/蓝牙网卡。                                |

***

| 项目                | 设置                           |
| ----------------- | ---------------------------- |
- Secure Boot：**关闭**
- Fast Boot：**关闭**
- CSM / Legacy：**关闭**（仅 UEFI）
- Above 4G Decoding：**开启**
- Resizable BAR：启动异常时**关闭**，否则保持默认
- IOMMU / SVM：默认即可
- VT-d：**关闭**（macOS 不需要，且可避免 DMA 重映射问题）

***

- [OpenCore](https://github.com/acidanthera/OpenCorePkg) / [Lilu](https://github.com/acidanthera/Lilu) / [VirtualSMC](https://github.com/acidanthera/VirtualSMC) / [AppleALC](https://github.com/acidanthera/AppleALC) / [WhateverGreen](https://github.com/acidanthera/WhateverGreen) —— Acidanthera
- [ECEnabler](https://github.com/1Revenger1/ECEnabler) —— 1Revenger1
- [OpenCore](https://github.com/acidanthera/OpenCorePkg)、[Lilu](https://github.com/acidanthera/Lilu)、[VirtualSMC](https://github.com/acidanthera/VirtualSMC)、[AppleALC](https://github.com/acidanthera/AppleALC)、[WhateverGreen](https://github.com/acidanthera/WhateverGreen) —— Acidanthera
- [VoodooI2C](https://github.com/VoodooI2C/VoodooI2C)、[VoodooPS2Controller](https://github.com/acidanthera/VoodooPS2) —— VoodooI2C 项目 / Acidanthera
- [USBToolBox](https://github.com/USBToolBox/tool) —— USBToolBox
- [RealtekRTL8111](https://github.com/Mieze/RTL8111_driver_for_OS_X) —— Mieze
- [AirportMtkWlan / SimpleMtkWlan](https://github.com/laobamac) —— laobamac、pengubow（AirportItlwm 的 MediaTek HAL 分支）
- [SSDT-BATT-ASUS](https://github.com/VisualTechStudio/SSDT-BATT-ASUS) —— 电池实现思路参考；`SSDT-BATT.aml` 依据本机 DSDT 转储重写
- [AirportMtkWlan / SimpleMtkWlan](https://github.com/laobamac) —— laobamac、pengubow（基于 AirportItlwm 的 MediaTek HAL 分支）
- `SSDT-BATT.aml` —— 依据本机 DSDT 转储编写。

***

本 EFI 仅针对上述机型。修改 ACPI、引导配置与刷写 EFI 均存在风险，请务必备份可用的 EFI，并保留 Windows 与 U 盘恢复手段。作者不对任何后果负责。
