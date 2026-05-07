---
name: stm32f4-cli-dev
description: STM32F4 嵌入式软件开发全流程指导。覆盖 CubeMX 项目创建、CLI 编译烧录、GDB 调试、外设开发、FreeRTOS 等。当用户需要进行 STM32 嵌入式开发、编译烧录固件、调试 STM32 程序、编写外设驱动、排查 HardFault、使用 FreeRTOS、或询问 STM32 相关问题时应使用此 skill。
---

# STM32F4 嵌入式开发指南

> 目标芯片：STM32F407ZGT6 (Cortex-M4, 1MB Flash, 192KB SRAM)
> 工具链：STM32CubeCLT 1.21.0 + GCC 14.3.1 (arm-none-eabi) + STM32CubeProgrammer 2.22.0
> 开发板：正点原子探索者

## 核心原则

1. **最小化修改** — 只改任务相关的代码
2. **不编写兼容性代码** — 针对当前芯片和工具链
3. **printf 重定向用 `__io_putchar`** — `_write → __io_putchar` 调用链，fputc 无效
4. **GDB Server 必须 `-e` 持久模式** — 非持久模式在客户端断连后自动退出
5. **烧录用 STM32_Programmer_CLI** — 不用 GDB `load`
6. **暂停 MCU 用 `interrupt`** — 比 `monitor halt` 更可靠

## 工具链定位

首次使用时检测工具链并设置环境变量。检测不到则报错，由开发者自行解决。

```powershell
if ($env:STM32CubeCLT_PATH -and $env:GDB_CLIENT_PATH -and $env:PYTHON38_PATH) { return }

# 1. STM32CubeCLT — 从 PATH 中的 exe 推导根目录，再扫常见位置
$clt = $null
foreach ($exe in @('STM32_Programmer_CLI.exe','arm-none-eabi-gcc.exe','ST-LINK_gdbserver.exe')) {
    $cmd = Get-Command $exe -ErrorAction SilentlyContinue
    if ($cmd) {
        $p = Split-Path $cmd.Source
        while ($p -and (Split-Path $p -Leaf) -notmatch '^STM32CubeCLT') { $p = Split-Path $p }
        if ($p) { $clt = $p; break }
    }
}
if (-not $clt) {
    $roots = @("${env:ProgramFiles}\STMicroelectronics", "${env:ProgramFiles(x86)}\STMicroelectronics", "C:\ST")
    $clt = $roots | Where-Object { $_ } | ForEach-Object {
        Get-ChildItem $_ -Directory -Filter "STM32CubeCLT*" -ErrorAction SilentlyContinue
    } | Sort-Object Name -Descending | Select-Object -First 1 -ExpandProperty FullName
}
if ($clt) {
    $env:STM32CubeCLT_PATH = $clt
    $env:PATH = "$clt\GNU-tools-for-STM32\bin;$clt\STM32CubeProgrammer\bin;$clt\STLink-gdb-server\bin;$clt\CMake\bin;$clt\Ninja\bin;$env:PATH"
}
else { throw "STM32CubeCLT not found. Set `$env:STM32CubeCLT_PATH manually." }

# 2. GDB 客户端 — arm-none-eabi-gdb-py.exe（含 Python 支持）
$gdb = Get-Command arm-none-eabi-gdb-py.exe -ErrorAction SilentlyContinue
if (-not $gdb) {
    $gdb = Get-ChildItem "${env:ProgramFiles}" -Recurse -Filter "arm-none-eabi-gdb-py.exe" `
        -Depth 3 -ErrorAction SilentlyContinue | Select-Object -First 1
}
if ($gdb) { $env:GDB_CLIENT_PATH = if ($gdb.Source) { $gdb.Source } else { $gdb.FullName } }
else { throw "arm-none-eabi-gdb-py.exe not found. Set `$env:GDB_CLIENT_PATH manually." }

# 3. Python 3.8
$py = Get-Command python.exe -ErrorAction SilentlyContinue
if ($py) {
    $ver = (& python.exe -c "import sys; print('{}.{}'.format(*sys.version_info[:2]))" 2>$null).Trim()
    if ($ver -eq '3.8') { $env:PYTHON38_PATH = Split-Path (Split-Path $py.Source) }
}
if (-not $env:PYTHON38_PATH) {
    $py38 = @("C:\Python\Python38", "C:\Python38", "$env:LOCALAPPDATA\Programs\Python\Python38") `
        | Where-Object { Test-Path "$_\python.exe" } | Select-Object -First 1
    if ($py38) { $env:PYTHON38_PATH = $py38 }
}
if (-not $env:PYTHON38_PATH) { throw "Python 3.8 not found. Set `$env:PYTHON38_PATH manually." }
```

`$env:STM32CubeCLT_PATH` 下结构：
```
├── CMake\bin\                        cmake.exe
├── Ninja\bin\                        ninja.exe
├── GNU-tools-for-STM32\bin\          arm-none-eabi-gcc
├── STM32CubeProgrammer\bin\          STM32_Programmer_CLI.exe
├── STLink-gdb-server\bin\            ST-LINK_gdbserver.exe
├── drivers\
├── jre\
└── st-arm-clang\
```

## 工作流

### 1. 创建项目（唯一 GUI 步骤）

CubeMX → 选 MCU → 配外设/时钟 → Project Manager 设 Toolchain=CMake → Generate Code

项目结构：
```
project.ioc          → CubeMX 配置
CMakeLists.txt       → 用户级（可自由改）
CMakePresets.json    → Debug/Release 预设 (Ninja + GCC)
cmake/
  gcc-arm-none-eabi.cmake
  stm32cubemx/CMakeLists.txt  → CubeMX 管理（重新生成时覆盖）
Core/Inc/, Core/Src/ → 用户代码（写在 USER CODE BEGIN/END 之间）
Drivers/             → HAL/CMSIS
```

### 2. 编译

```powershell
cmake --preset Debug
cmake --build --preset Debug
# Release: cmake --preset Release && cmake --build --preset Release
```

产物：`build/Debug/project.{elf,hex,bin,map}`

### 3. 烧录

```powershell
STM32_Programmer_CLI -c port=SWD -w build/Debug/project.elf -v -rst
# 其他格式: .hex / .bin 0x08000000
# 全片擦除: -e all
# 设备列表: -l
# MCU 信息: -c port=SWD -i
# Option Bytes(只读): -c port=SWD -ob displ
```

烧录前：`Get-Process -Name "*gdbserver*" -ErrorAction SilentlyContinue | Stop-Process -Force`

### 4. 调试

架构：`gdb-py.exe → TCP:61234 → ST-LINK_gdbserver.exe → SWD → MCU`

启动 Server：
```powershell
$clt = $env:STM32CubeCLT_PATH
& "$clt\STLink-gdb-server\bin\ST-LINK_gdbserver.exe" -e -p 61234 -d -cp "$clt\STM32CubeProgrammer\bin"
```
关闭：`Stop-Process -Name "ST-LINK_gdbserver" -Force`

GDB 批处理：
```powershell
& $env:GDB_CLIENT_PATH --batch `
  -ex "file D:/path/project.elf" `
  -ex "target extended-remote 127.0.0.1:61234" `
  -ex "interrupt" `
  -ex "print/x var" `
  -ex "quit"
```

关键规则：路径用正斜杠 `D:/`、`file` 在 `target` 之前、简单任务用 `-ex`（容错）复杂用 `-x` 脚本、CP1252 警告可忽略。

## GDB 常用命令

| 命令 | 功能 |
|------|------|
| `monitor reset halt` | 复位停在第一条指令 |
| `interrupt` | 暂停 MCU |
| `continue / c` | 继续运行 |
| `break main` / `thbreak func` | 断点 / 临时断点 |
| `step / next / stepi / finish` | 单步 |
| `info reg pc sp lr` / `info locals` / `info args` | 寄存器 / 局部变量 / 参数 |
| `print var` / `print/x var` | 打印 |
| `x/16wx 0x20000000` | 查看 SRAM |
| `backtrace` / `frame N` | 调用栈 |
| `watch var` / `condition N expr` | 硬件观察点 / 条件断点 |
| `delete` / `disassemble func` | 删除断点 / 反汇编 |
| `detach` / `quit` | 断开 / 退出 |

## FreeRTOS GDB 调试

`freertos tasks` 在 Windows 不可用（CP1252→UTF-8 编码冲突）。用纯 GDB 替代：

```gdb
print/x uxCurrentNumberOfTasks
print/x xTickCount
x/16bx ((char*)pxCurrentTCB)+0x34    # 当前任务名
# TCB: pxTopOfStack@0x00, pcTaskName[16]@0x34
```

## 静态分析

| 操作 | 命令 |
|------|------|
| 段大小 | `arm-none-eabi-size project.elf` |
| 反汇编 | `arm-none-eabi-objdump -d -S project.elf` |
| 反汇编函数 | `arm-none-eabi-objdump -d --disassemble=main project.elf` |
| 符号表(地址) | `arm-none-eabi-nm -n project.elf` |
| 符号表(大小) | `arm-none-eabi-nm --print-size --size-sort --radix=d project.elf` |
| 段头 | `arm-none-eabi-readelf -S project.elf` |
| 向量表 | `arm-none-eabi-objdump -s -j .isr_vector project.elf` |
| 预定义宏 | `cmd /c "arm-none-eabi-gcc -mcpu=cortex-m4 -mthumb -dM -E - < nul"` |
| BIN 生成 | `arm-none-eabi-objcopy -O binary project.elf project.bin` |

## Flash 扇区 (STM32F407ZGT, 1MB)

| 扇区 | 地址 | 大小 |
|------|------|------|
| 0-3 | 0x08000000-0x0800FFFF | 各 16KB |
| 4 | 0x08010000-0x0801FFFF | 64KB |
| 5-11 | 0x08020000-0x080FFFFF | 各 128KB |

## HardFault 关键寄存器

| 寄存器 | 地址 | 用途 |
|--------|------|------|
| CFSR | 0xE000ED28 | 故障状态 (UFSR/BFSR/MMFSR) |
| HFSR | 0xE000ED2C | HardFault 状态 (FORCED bit) |
| BFAR | 0xE000ED38 | 总线故障地址 |

## 参考文件

需要详细命令时加载对应文件：

- **[workflow.md](references/workflow.md)** — 烧录 Flash 操作 / 串口监视 / CI/CD 脚本 / 系统时钟 / 操作权限
- **[gdb-guide.md](references/gdb-guide.md)** — GDB Server 完整指南 / Python 环境变量 / 异常处理
- **[lessons.md](references/lessons.md)** — 7 条开发教训（根因+解决方案）
