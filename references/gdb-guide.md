# GDB 调试指南

## 架构

```
gdb-py.exe (客户端)            ST-LINK_gdbserver.exe (服务器)
  ├─ Python 3.8                  ├─ USB → SWD
  ├─ 符号解析                    └─ TCP :61234 → ST-LINK → MCU
  └─ TCP → 127.0.0.1:61234 ──────┘
```

## Python 环境变量

使用 GDB 客户端前必须设置：

```powershell
$py = $env:PYTHON38_PATH
$env:PATH = "$py;$py\Scripts;" + $env:PATH
$env:PYTHONHOME = $py
$env:PYTHONPATH = "$py\Lib;$py\Lib\site-packages"
```

## 启动 GDB Server（持久模式 + 后台）

```powershell
$clt = $env:STM32CubeCLT_PATH
& "$clt\STLink-gdb-server\bin\ST-LINK_gdbserver.exe" `
  -e `                    # 持久模式（必须）
  -p 61234 `              # 监听端口
  -d `                    # SWD 模式
  -cp "$clt\STM32CubeProgrammer\bin"
```

验证：输出 `Waiting for debugger connection...`
关闭：`Stop-Process -Name "ST-LINK_gdbserver" -Force`

## 批处理模式

### `-ex` 模式（推荐，逐命令容错）

```powershell
& $env:GDB_CLIENT_PATH --batch `
  -ex "file D:/path/project.elf" `
  -ex "target extended-remote 127.0.0.1:61234" `
  -ex "interrupt" `
  -ex "print/x var" `
  -ex "quit"
```

### `-x` 脚本文件（遇错即停）

```powershell
& $env:GDB_CLIENT_PATH --batch -x "D:/path/script.gdb"
```

脚本模板：
```
set confirm off
file D:/path/project.elf
target extended-remote 127.0.0.1:61234
interrupt
# ... commands ...
quit
```

## 关键规则

1. 路径用正斜杠 `D:/path/file`
2. `file` 在 `target` 之前（先加载符号表）
3. 用 `interrupt` 而非 `monitor halt` 暂停 MCU
4. 不要用 `load` 烧录（用 `STM32_Programmer_CLI`）
5. CP1252/UTF-32 编码警告可忽略
6. `monitor reset halt` 偶尔触发非致命 "Protocol error with Rcmd: 05"
7. `-batch` 模式不需要交互终端
8. `set confirm off` 避免 quit 确认提示

## 命令全集

### 运行控制
| 命令 | 功能 |
|------|------|
| `monitor reset halt` | 复位停在第一条指令 |
| `interrupt` | 暂停运行中的目标 |
| `continue` / `c` | 继续运行 |
| `step` / `s` | 单步进入 |
| `next` / `n` | 单步跳过 |
| `stepi` | 汇编级单步 |
| `finish` | 运行到当前函数返回 |
| `detach` | 断开连接（MCU 继续运行） |
| `quit` | 退出 GDB |

### 断点/观察点
| 命令 | 功能 |
|------|------|
| `break func` / `break file:line` | 设断点 |
| `thbreak func` | 临时断点（命中后自动删除） |
| `condition N expr` | 条件断点 |
| `watch var` | 硬件观察点（最多 4 个） |
| `info break` | 查看所有断点 |
| `delete N` / `delete` | 删除断点 |

### 数据检查
| 命令 | 功能 |
|------|------|
| `info reg pc sp lr` | 关键寄存器 |
| `info locals` / `info args` | 局部变量 / 函数参数 |
| `print var` / `print/x var` | 打印变量 / 十六进制 |
| `x/16wx 0x20000000` | 查看 SRAM |
| `x/1xw 0x40020000` | 查看 GPIOA MODER |
| `x/16bx addr` | 查看 addr 处 16 字节 |

### 栈/反汇编
| 命令 | 功能 |
|------|------|
| `backtrace` / `bt` | 调用栈 |
| `bt N` | 前 N 帧 |
| `frame N` | 切换到第 N 帧 |
| `disassemble func` | 反汇编函数 |
| `disassemble /m func` | 反汇编 + 源码交织 |

## FreeRTOS GDB 调试

`freertos tasks` 在 Windows 不可用（CP1252→UTF-8 编码冲突 → `gdb.MemoryError`）。

纯 GDB 替代：
```gdb
print/x uxCurrentNumberOfTasks
print/x xTickCount
x/16bx ((char*)pxCurrentTCB)+0x34    # 当前任务名
```

TCB 字段偏移（`configUSE_PORT_OPTIMISED_TASK_SELECTION=0`）：

| 偏移 | 字段 |
|------|------|
| 0x00 | pxTopOfStack |
| 0x34 | pcTaskName[16] |

需在调度器启动后读取（任务入口断点命中后），否则返回 0x0。

完整批处理：
```powershell
$py = $env:PYTHON38_PATH
$env:PATH = "$py;$py\Scripts;" + $env:PATH
$env:PYTHONHOME = $py
$env:PYTHONPATH = "$py\Lib;$py\Lib\site-packages"

& $env:GDB_CLIENT_PATH --batch `
  -ex "file D:/path/project.elf" `
  -ex "target extended-remote 127.0.0.1:61234" `
  -ex "interrupt" `
  -ex "print/x uxCurrentNumberOfTasks" `
  -ex "print/x xTickCount" `
  -ex "x/16bx ((char*)pxCurrentTCB)+0x34" `
  -ex "quit"
```

## 异常处理

### Server 启动即退出
原因：未指定 `-cp` 或父 shell 退出。解决：`-e` + 后台运行。

### ST-LINK 被占用 (DEV_CONNECT_ERR)
```powershell
Get-Process -Name "*gdbserver*" | Stop-Process -Force
Start-Sleep -Seconds 2
```

### GDB 连接 ECONNREFUSED
Server 未运行或已退出。检查 `-e` 持久模式是否生效。
