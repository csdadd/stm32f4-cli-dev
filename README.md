# stm32f4-cli-dev

适用于 Claude Code 的 STM32F4 嵌入式开发 SKILL — 覆盖 CubeMX 项目创建、CLI 编译烧录、GDB 调试、外设开发、FreeRTOS 全流程。

## 前置工具

| 工具 | 用途 | 免费 |
|------|------|------|
| STM32CubeMX | 图形化项目生成（引脚/时钟/外设配置） | 是 |
| STM32CubeCLT | GCC 工具链 + STM32_Programmer_CLI + ST-LINK GDB Server + CMake + Ninja | 是 |
| Arm GNU Toolchain (mingw-w64) | 带 Python 支持的 GDB 客户端 (`arm-none-eabi-gdb-py.exe`) | 是 |
| Python 3.8 | GDB Python 脚本运行时（FreeRTOS 调试等） | 是 |

## 工具安装

### 1. STM32CubeMX

下载并安装：https://www.st.com/en/development-tools/stm32cubemx.html

> 需要注册 ST 账号（免费）。

### 2. STM32CubeCLT

下载并安装：https://www.st.com/en/development-tools/stm32cubeclt.html

> 包含 GCC (arm-none-eabi)、STM32_Programmer_CLI、ST-LINK GDB Server、CMake、Ninja、USB 驱动。
>
> 安装后验证：
> ```powershell
> arm-none-eabi-gcc --version
> STM32_Programmer_CLI --version
> cmake --version
> ninja --version
> ```

### 3. Arm GNU Toolchain (mingw-w64)

下载页面：https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads

选择 **mingw-w64-x86_64-arm-none-eabi** 版本（Windows 64-bit）。

> 提取到任意目录，例如 `C:\ARM\GNU Toolchain mingw-w64-x86_64-arm-none-eabi\`

**注意：STM32CubeCLT 自带的 `arm-none-eabi-gdb.exe` 不含 Python 支持，必须使用此独立工具链中的 `arm-none-eabi-gdb-py.exe`。**

### 4. Python 3.8

下载：https://www.python.org/downloads/release/python-3810/

> GDB 内嵌 Python 绑定到特定版本，当前工具链要求 Python 3.8。
>
> 安装时勾选 "Add Python to PATH" 或记录安装路径（如 `C:\Python\Python38`）。

## 环境配置

SKILL 首次使用时自动检测工具链位置。检测逻辑：

1. 从 `PATH` 环境变量中查找对应可执行文件，推导安装根目录
2. 若 PATH 中找不到，扫描常见安装目录
3. 仍找不到则**报错停止**，不会替你去下载或安装

如果自动检测失败，手动设置后即可使用。将 `<路径>` 替换为本机实际位置：

```powershell
# 临时设置（仅当前 PowerShell 窗口有效）
$env:STM32CubeCLT_PATH = "<STM32CubeCLT 安装目录>"
$env:GDB_CLIENT_PATH  = "<arm-none-eabi-gdb-py.exe 完整路径>"
$env:PYTHON38_PATH    = "<Python 3.8 安装目录>"

# 永久写入（之后新开的窗口自动生效）
[Environment]::SetEnvironmentVariable("STM32CubeCLT_PATH", "<STM32CubeCLT 安装目录>", "User")
[Environment]::SetEnvironmentVariable("GDB_CLIENT_PATH",  "<arm-none-eabi-gdb-py.exe 完整路径>", "User")
[Environment]::SetEnvironmentVariable("PYTHON38_PATH",    "<Python 3.8 安装目录>", "User")
```

## SKILL 安装

将此仓库克隆到 Claude Code 的 SKILL 目录：

```powershell
# 克隆到 Claude Code skills 目录
git clone https://github.com/csdadd/stm32f4-cli-dev.git "$env:USERPROFILE\.claude\skills\stm32f4-cli-dev"
```

或手动下载解压到 `%USERPROFILE%\.claude\skills\stm32f4-cli-dev\`。

安装后重启 Claude Code 或执行 `/skills` 确认加载。

## 使用方式

在 Claude Code 对话中提及 STM32 相关关键词即可自动激活此 SKILL。

触发词：STM32、STM32F4、CubeMX、HAL、固件、烧录、GDB、ST-LINK、FreeRTOS、嵌入式开发、外设驱动、GPIO/ADC/TIM/PWM/DMA/SPI/I2C/UART/CAN 等。

## 适用硬件

- **芯片**：STM32F407ZGT6 (Cortex-M4, 1MB Flash, 192KB SRAM)
- **开发板**：正点原子探索者（其他 STM32F4 板卡也可参考）
- **调试器**：ST-LINK V2 (SWD)
- **OS**：Windows 11 + PowerShell

## SKILL 覆盖范围

- CubeMX CMake 项目创建与配置
- CLI 编译 (CMake + Ninja + GCC)
- CLI 烧录与 Flash 操作 (STM32_Programmer_CLI)
- GDB 调试 (ST-LINK GDB Server + arm-none-eabi-gdb-py)
- 串口监视 (PowerShell SerialPort)
- 外设驱动 (GPIO/ADC/TIM/PWM/DMA/SPI/I2C/UART 等)
- FreeRTOS 任务调试与栈分析
- HardFault 排查 (CFSR/HFSR/BFAR)
- 静态分析 (objdump/nm/readelf/size)
- 7 条实战开发教训

## 文件结构

```
├── SKILL.md                    # SKILL 主文件
├── README.md
└── references/
    ├── workflow.md             # CLI 工作流完整指南
    ├── gdb-guide.md            # GDB 调试完整指南
    └── lessons.md              # 开发教训速查
```

## 许可

MIT
