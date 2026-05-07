# STM32 CLI 开发教训


## 1. printf 重定向 — 必须用 `__io_putchar`

**问题**：`fputc` 重定向在 CubeMX 生成的项目中无效。

**根因**：CubeMX 生成的 `syscalls.c` 中 `printf` 调用链是 `printf → _write_r → _write → __io_putchar`，不是 `printf → fputc`。

**解决**：只需实现 `__io_putchar`：
```c
int __io_putchar(int ch) {
  while (!(USART1->SR & USART_SR_TXE)) {}
  USART1->DR = ch;
  return ch;
}
```

**诊断**：在输出函数中加 LED 翻转代码，看 LED 是否快速闪烁判断函数是否被调用。

## 2. 串口测试必须包含周期性输出

**问题**：固件烧录后打开串口总是无数据。

**根因**：MCU 复位后几毫秒内 printf 启动消息已发送完毕，PowerShell 打开 COM3 需数秒。

**解决**：主循环中加周期性 printf（如每秒心跳）：
```c
if (++tick % 5 == 0) printf("heartbeat %lu\r\n", tick);
```

## 3. DMA 方向位常量含义陷阱

**问题**：`DMA_SxCR_DIR_0` 做 M→M 传输死循环。

**根因**：`_0`/`_1` 后缀表示枚举取值，不是位号。

| 常量 | 位值 | 实际含义 |
|------|------|----------|
| `DMA_SxCR_DIR_0` | 0x40 | 存储器→外设 |
| `DMA_SxCR_DIR_1` | 0x80 | 存储器→存储器 |

## 4. GDB `-ex` 优于 `-x` 脚本文件

- `-ex`：逐命令容错，非致命错误不中断后续命令
- `-x`：遇错即停，`monitor reset halt` 的 "Protocol error with Rcmd: 05" 会导致后续跳过

选择：命令少（<10 个）→ `-ex`；命令多或需条件逻辑 → `-x` 脚本 + 持久模式 Server。

## 5. 串口捕获启动消息 — 先开端口再复位

正确顺序：(1) 打开串口 (2) 复位 MCU (3) 等待输出：
```powershell
$port.Open()
Start-Sleep -Milli 500
STM32_Programmer_CLI -c port=SWD -rst
Start-Sleep -Milli 4000
$buf = $port.ReadExisting()
```
先烧录后开串口必然错过启动阶段。

## 6. FreeRTOS 任务栈不足 — 无报错直接死机

CubeMX 默认栈 128*4=512 bytes。printf（通过 newlib `_write → __io_putchar`）调用链需 800+ bytes。栈溢出→HardFault→FreeRTOS 默认 HardFault_Handler 是死循环→无任何输出。

**解决**：栈增加到 `256*4`（1024 bytes）。实测 defaultTask ~400 bytes，workerTask ~392 bytes。剩余 624/632 bytes 安全。

**诊断**：GDB 连接→看 PC（若在 Reset_Handler 说明持续复位）→逐步简化代码→回退到只有 osDelay+GPIO_TogglePin→读 pxCurrentTCB（若为空说明任务未创建就崩溃）。

## 7. freertos-gdb 在 Windows 上不可用 — CP1252 编码冲突

`freertos tasks` 抛出 `gdb.MemoryError: Cannot access memory at address 0xa1080094`。

**根因**：Windows 版 GDB 内部用 CP1252，FreeRTOS 内核符号从 C 层→Python3(UTF-8) 编码转换失败，指针值被破坏。

**替代**：纯 GDB 命令读取 TCB：
```gdb
print/x uxCurrentNumberOfTasks
print/x xTickCount
x/16bx ((char*)pxCurrentTCB)+0x34
```
TCB 偏移：pxTopOfStack@0x00, pcTaskName[16]@0x34

**正确的 GDB 启动**：必须用 `arm-none-eabi-gdb-py.exe` + 需设 PYTHONHOME/PYTHONPATH/PATH + Python 3.8。

## 8. GDB Server 必须用 `-e` 持久模式

非持久模式下每次 `detach` 后 Server 自动退出。

**解决**：
```powershell
ST-LINK_gdbserver.exe -e -p 61234 -d -cp "<CUBEPROGRAMMER_PATH>"
```
持久模式下 Server 在客户端断开后保持运行，实测处理 11+ 次连续连接。第一次连接时可能端口占用，先 `Stop-Process` 再启动。
