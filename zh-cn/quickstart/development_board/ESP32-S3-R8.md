# ESP32-S3-R8 移植与使用指南

本文介绍 openvela 对 ESP32-S3-R8 板级平台的适配结构、配置方法、构建烧录流程和基础验证方法。

本文对应独立的 `esp32s3-r8` 板级目标，不修改也不替代 `esp32s3-eye`、`esp32s3-box` 或其他 ESP32-S3 开发板适配。文中硬件信息以当前 R8 适配代码和已确认的板级连接为准；未确认的外设不会在默认配置中启用。

## 一、平台概述

### 1. 硬件信息

| 项目 | 配置 |
| --- | --- |
| SoC/模块 | ESP32-S3-WROOM-2-N32R8V |
| CPU 架构 | Xtensa LX7，ESP32-S3 |
| Flash | 32 MB |
| PSRAM | 8 MB，Octal 模式，80 MHz |
| 显示屏 | ST7789V，240 × 320，RGB565 |
| 显示接口 | SPI2，SPI mode 0 |
| LCD 颜色顺序 | BGR |
| LCD 数据顺序 | RGB565 高低字节交换 |

### 2. 适配范围

当前 R8 适配包提供以下基础能力：

- 独立的 `esp32s3-r8` 板级目录和板级 Kconfig 入口。
- NSH 启动配置，以及通过 SPI2 初始化 ST7789V LCD 的配置。
- 板级启动、bringup、按键、复位、SPI 总线和 LCD 初始化代码。
- 与 openvela 原生工具链和 ESP32-S3 构建脚本兼容的 `Make.defs`。

## 二、openvela 移植分层

openvela 的新平台适配按照架构层、芯片层和板级层组织：

1. **架构层**：提供 Xtensa 处理器相关的启动、上下文切换、中断和编译支持。
2. **芯片层**：提供 ESP32-S3 的时钟、GPIO、SPI、USB Serial、PSRAM 和其他 SoC 外设支持。
3. **板级层**：提供 R8 的引脚映射、外设连接、初始化顺序和板级配置。

R8 只新增板级目录，并复用现有 ESP32-S3 架构层和芯片层实现。板级代码不应把 R8 的引脚或外设假设写入公共 EYE、BOX 或 ESP32-S3 芯片代码。

## 三、代码和配置位置

### 1. NuttX 板级代码

NuttX 仓库中的 R8 代码位于：

```text
nuttx/boards/xtensa/esp32s3/esp32s3-r8/
```

主要文件和职责如下：

| 路径 | 职责 |
| --- | --- |
| `Kconfig` | 定义 R8 LCD 开关及其依赖联动 |
| `include/board.h` | 定义板级时钟和基础硬件宏 |
| `scripts/Make.defs` | 选择 ESP32-S3 工具链、链接脚本和编译选项 |
| `src/Make.defs` | 按 Kconfig 选择板级源文件 |
| `src/esp32s3_boot.c` | 板级启动入口 |
| `src/esp32s3_bringup.c` | procfs、计时器、看门狗、按键、无线和 LCD 等板级初始化入口 |
| `src/esp32s3_board_spi.c` | SPI2 设备状态及 LCD command/data 控制 |
| `src/esp32s3_board_lcd.c` | LCD GPIO、复位、背光和 ST7789V 初始化 |
| `src/esp32s3_buttons.c` | 板级按键支持 |
| `src/esp32s3_reset.c` | `boardctl` 复位支持 |
| `src/esp32s3_appinit.c` | `boardctl` 应用初始化入口 |
| `configs/nsh/defconfig` | 最小 NSH 启动配置 |
| `configs/lcd/defconfig` | 启用 LCD 的基础验证配置 |

### 2. vendor 配置

vendor/espressif 仓库中的 R8 配置位于：

```text
vendor/espressif/boards/esp32s3/esp32s3-r8/
```

其中 `configs/openvela/defconfig` 是 vendor 构建入口使用的完整配置，`scripts/Make.defs` 与 NuttX 板级链接脚本选择保持一致。NuttX 板级代码和 vendor 配置必须使用相同的板名：

```text
esp32s3-r8
```

不要把 R8 配置放入 `esp32s3-eye` 或 `esp32s3-box` 目录，也不要通过修改 EYE 的 defconfig 来间接启用 R8。

## 四、板级连接与显示配置

### 1. LCD 引脚映射

| 信号 | GPIO | 说明 |
| --- | ---: | --- |
| SPI SCK | 11 | SPI2 时钟 |
| SPI MOSI | 5 | LCD 写数据 |
| SPI MISO | 4 | SPI2 MISO；当前 LCD 写屏流程不依赖读回 |
| SPI CS | 12 | LCD 片选 |
| LCD DC | 6 | 命令/数据选择 |
| LCD RST | 8 | LCD 复位，低有效 |
| LCD BL | 7 | 背光控制，高电平打开 |

LCD 初始化顺序为：配置 DC、RST、BL GPIO，拉低并释放 LCD 复位，打开背光，初始化 SPI2，创建 ST7789V LCD 设备并设置显示电源。

### 2. SPI 和像素格式

R8 的 LCD 配置必须同时满足以下条件：

- SPI mode 为 0。
- ST7789V 分辨率为 240 × 320。
- 颜色顺序使用 BGR。
- RGB565 像素发送启用高低字节交换。
- SPI 频率配置为 40 MHz；若实际屏幕或连线不稳定，应先在硬件验证中降低频率，不应修改其他板卡的公共默认值。

`CONFIG_LCD_ST7789_SWAP_BYTES=y` 只在 R8 的配置中启用。该选项和驱动中的条件编译共同保证 16 位 RGB565 数据按 LCD 所需的字节顺序发送；不要通过修改应用像素缓冲区来替代板级显示配置。

## 五、Kconfig 联动机制

R8 LCD 开关定义在 R8 自己的板级 `Kconfig` 中：

```text
CONFIG_ESP32S3_R8_LCD
```

启用该开关时，Kconfig 自动选择：

- `ESP32S3_SPI2`
- `SPI_CMDDATA`
- `LCD`
- `LCD_DEV`
- `LCD_ST7789`

SPI2 的具体 GPIO 和 LCD 参数仍由 defconfig 或 menuconfig 保存到最终配置中：

```text
CONFIG_ESP32S3_SPI2_CLKPIN=11
CONFIG_ESP32S3_SPI2_MOSIPIN=5
CONFIG_ESP32S3_SPI2_MISOPIN=4
CONFIG_ESP32S3_SPI2_CSPIN=12
CONFIG_LCD_ST7789_SPIMODE=0
CONFIG_LCD_ST7789_FREQUENCY=40000000
CONFIG_LCD_ST7789_SWAP_BYTES=y
```

板级源文件通过 `src/Make.defs` 按开关选择：

- `CONFIG_ESP32S3_SPI` 启用 `esp32s3_board_spi.c`。
- `CONFIG_ESP32S3_R8_LCD` 启用 `esp32s3_board_lcd.c`。
- `CONFIG_BOARDCTL` 启用应用初始化和可选复位代码。
- `CONFIG_ARCH_BUTTONS` 启用板级按键代码。

修改配置时应使用 `menuconfig` 或由配置目录重新生成，不要直接编辑自动生成的 `defconfig` 注释头所标记的构建产物。保存配置后检查 `.config`、生成的头文件和目标板名，确保没有残留 `esp32s3-eye` 配置。

## 六、工具链与构建

### 1. 工具链要求

ESP32-S3 必须使用仓库配套的 Xtensa ESP32-S3 工具链和 ESP32-S3 构建脚本。主机系统的 `/usr/bin/gcc` 不能替代 Xtensa 交叉编译器；出现 `-mlongcalls` 等 Xtensa 参数不识别时，说明工具链没有正确进入 `PATH`，不是 R8 源码编译结论。

如仓库已提供工具链或工具目录，可先按项目环境配置 `PATH`。不要为了绕过工具链错误修改 R8 的 `Make.defs` 或删除架构编译选项。

### 2. 使用 vendor 配置构建

在 openvela 根目录执行：

```sh
./build.sh vendor/espressif/boards/esp32s3/esp32s3-r8/configs/openvela -j$(nproc)
```

如果需要进入配置界面：

```sh
./build.sh vendor/espressif/boards/esp32s3/esp32s3-r8/configs/openvela menuconfig
```

构建前确认 NuttX 和 vendor/espressif 工作树处于期望版本，并确认配置路径来自 R8。构建产物名称和位置以当前 openvela 构建脚本输出为准，不要从其他板卡目录复制固件或链接脚本。

### 3. 只验证 NuttX 基础配置

也可以使用 NuttX 原生配置工具验证板级入口：

```sh
cd nuttx
./tools/configure.sh -L esp32s3-r8:nsh
./tools/configure.sh -L esp32s3-r8:lcd
```

`nsh` 配置用于确认板级启动和 NSH 入口；`lcd` 配置在此基础上确认 LCD 相关 Kconfig 联动、SPI2 引脚和 ST7789V 参数。

## 七、烧录与启动

构建完成后，在当前 openvela 工作区按项目提供的原生流程烧录。典型命令如下：

```sh
cd /home/jmz/openvela
export PATH="/home/jmz/openvela/prebuilts/tools/linux/x86_64/fs:$PATH"
make -C nuttx flash ESPTOOL_PORT=/dev/ttyACM0 ESPTOOL_BAUD=921600
```

实际串口设备可能不是 `/dev/ttyACM0`，应先通过 `ls /dev/ttyACM*` 或系统设备日志确认。烧录前确认 `nuttx/.config` 对应 `esp32s3-r8`，避免将其他板卡固件写入目标板。

启动后通过串口终端观察 NSH：

```sh
minicom -D /dev/ttyACM0 -b 115200
```

基础启动成功的标志是出现：

```text
NuttShell (NSH)
nsh>
```

## 八、验证项目

### 1. 配置验证

- 检查 `CONFIG_ARCH_BOARD="esp32s3-r8"`。
- 检查芯片选择为 `CONFIG_ARCH_CHIP_ESP32S3WROOM2N32R8V=y`。
- 检查 PSRAM 为 Octal、80 MHz。
- LCD 配置中检查 SPI2 GPIO、SPI mode、分辨率、BGR 和 byte swap。
- 检查生成配置中没有意外混入 EYE、BOX、MIST 或其他板级目录配置。

### 2. NSH 验证

进入 NSH 后可先执行：

```text
 uname
 free
 mount
```

重点确认系统可以稳定进入 NSH，堆内存统计正常，且启动阶段没有板级初始化错误。

### 3. LCD 验证

使用 `lcd` 配置启动后，检查：

- LCD 复位和背光控制时序正常。
- SPI2 初始化成功。
- LCD 设备注册成功。
- 画面颜色、方向和像素排列正常。
- 连续刷新时没有明显花屏或字节错位。

如果画面出现整体颜色异常或 RGB565 像素错位，优先检查 LCD 面板连接、SPI mode、BGR 和 `CONFIG_LCD_ST7789_SWAP_BYTES`，不要先修改通用 ST7789 默认行为。

### 4. 回归边界

本适配包未对 EYE、BOX 或 ESP32-S3 公共功能做替换。验证 R8 时应单独记录：

- 使用的仓库提交和配置目录。
- 实际硬件型号及 LCD 连接。
- 工具链版本和主机环境。
- 烧录端口和串口波特率。
- NSH 启动结果及 LCD 显示结果。

## 九、交付说明

R8 代码和配置按 NuttX 板级仓库与 vendor/espressif 配置仓库分别维护，板级目录、板名、Kconfig 符号和构建入口保持一一对应。文档中的命令均以独立的 `esp32s3-r8` 目标为对象，不应改用其他板卡的配置路径。

主机缺少 Xtensa 工具链或构建依赖时，应记录为构建环境问题；不能将主机原生 GCC 的参数错误归因于 R8 板级实现。

## 十、相关参考

- [新平台适配指南](../../chip_porting/porting_guide.md)
- [ESP32-S3 架构和平台代码](../../../../nuttx/tree/trunk/arch/xtensa/esp32s3)
- [R8 NuttX 板级代码](../../../../nuttx/tree/trunk/boards/xtensa/esp32s3/esp32s3-r8)
- [R8 vendor 配置](../../../../vendor_espressif/tree/trunk/boards/esp32s3/esp32s3-r8)
