# STM32 CLI 工作流补充

> 编译、烧录基础命令见 SKILL.md，本文档为扩展操作。

## 环境验证

```powershell
arm-none-eabi-gcc --version
cmake --version
ninja --version
STM32_Programmer_CLI --version
```

## Flash 操作

```powershell
# 全片擦除
STM32_Programmer_CLI -c port=SWD -e all

# 扇区擦除（0-3:16KB, 4:64KB, 5-11:128KB）
STM32_Programmer_CLI -c port=SWD -e 1
STM32_Programmer_CLI -c port=SWD -e 0 1 2

# 回读
STM32_Programmer_CLI -c port=SWD -r 0x08000000 0x10000 dump.bin
STM32_Programmer_CLI -c port=SWD -r 0x20000000 256 sram_dump.bin
```

## Option Bytes

```powershell
STM32_Programmer_CLI -c port=SWD -ob displ    # 读取（AI 可执行）
STM32_Programmer_CLI -c port=SWD -ob RDP=0xBB # 修改（仅限人工！）
```

## 设备查询

```powershell
STM32_Programmer_CLI -l                    # 列出连接的调试器
STM32_Programmer_CLI -c port=SWD -i        # MCU 型号/Flash/电压
```

## Under Reset 模式（Standby 恢复专用）

```powershell
STM32_Programmer_CLI -c port=SWD mode=UR
STM32_Programmer_CLI -c port=SWD mode=UR -w firmware.elf -v -rst
```

## 串口监视

```powershell
$port = New-Object System.IO.Ports.SerialPort COM3, 115200, None, 8, One
$port.ReadTimeout = 2000
$port.Open()
try {
    $timeout = [DateTime]::Now.AddSeconds(5)
    while ([DateTime]::Now -lt $timeout) {
        Write-Host -NoNewline $port.ReadChar()
    }
} finally { $port.Close() }
```

前提：P10 跳线帽已安装（CH340↔PA9/PA10）、printf 重定向用 `__io_putchar`。

### 捕获启动消息

先开端口再复位 MCU，否则错过启动输出：

```powershell
$port = New-Object System.IO.Ports.SerialPort COM3, 115200, None, 8, One
$port.Open()
Start-Sleep -Milli 500
STM32_Programmer_CLI -c port=SWD -rst
Start-Sleep -Milli 4000
$buf = $port.ReadExisting()
```

## Flash 回读比对

```powershell
$binSize = (Get-Item build/Debug/project.bin).Length
STM32_Programmer_CLI -c port=SWD -r 0x08000000 $binSize build/Debug/flash_readback.bin
$bin = [System.IO.File]::ReadAllBytes("build/Debug/project.bin")
$rb  = [System.IO.File]::ReadAllBytes("build/Debug/flash_readback.bin")
$diff = 0; for ($i=0; $i -lt $bin.Count; $i++) { if ($bin[$i] -ne $rb[$i]) { $diff++ } }
"Total: $($bin.Count) bytes, Diff: $diff"
```

## 一键构建烧录

```powershell
$ErrorActionPreference = "Stop"
$buildDir = "build\Debug"
$elf = "$buildDir\project.elf"
$prog = "$env:STM32CubeCLT_PATH\STM32CubeProgrammer\bin\STM32_Programmer_CLI.exe"

cmake --build $buildDir
if ($LASTEXITCODE -ne 0) { exit 1 }
arm-none-eabi-size $elf
& $prog -c port=SWD -w $elf -v -rst
if ($LASTEXITCODE -ne 0) { exit 1 }
"BUILD+FLASH OK"
```

## 系统时钟 (STM32F407 标准)

- HSE: 8MHz → PLL: M=4, N=168, P=2 → SYSCLK=168MHz
- HCLK=168MHz, APB1=42MHz (PPRE1=÷4), APB2=84MHz (PPRE2=÷2)
- APB1 时钟：`hclk / (ppre1 >= 4 ? (1U << (ppre1 - 3)) : 1U)`

## 操作权限

### AI Agent 可执行
构建 / 烧录 / 调试 / 静态分析 / 设备查询 / 串口监视 / Flash 操作 / 外设测试 / Option Bytes 读取

### 仅限人工
- Option Bytes 写入（误操作可锁死芯片）
- ST-LINK 固件升级（失败可导致调试器变砖）
