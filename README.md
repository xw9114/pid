# STM32H0 远程 PID 调参

这个仓库提供一个 Python 命令行工具，用于通过 Horco `DAPLINK-WIRELESS / ESP32-S3` 远程调节 STM32H0 的 PID 参数，并实时接收控制误差、测量值和输出。

## 文件

- `tune_console.py`：远程调参脚本，支持 TCP/Wi-Fi 和 USB-TTL 串口。

脚本只使用 Python 标准库。使用串口模式时，需要安装 `pyserial`：

```powershell
python -m pip install pyserial
```

## TCP/Wi-Fi 运行

电脑和 DAPLINK-WIRELESS 接入同一网络后，将 `dsp_link` 和 `<TCP_PORT>` 替换为模块实际的主机名/IP 和 TCP 端口：

```powershell
python tune_console.py --tcp dsp_link <TCP_PORT> --loop-id 1
```

发送初始参数：

```powershell
python tune_console.py --tcp dsp_link <TCP_PORT> `
  --loop-id 1 --kp 10.0 --ki 0 --kd 0.15 --aux 0 --send-initial
```

脚本默认只连接并接收遥测，不会自动修改 MCU 参数。只有使用 `send` 命令或显式指定 `--send-initial` 才会发送参数。

## USB-TTL 直连

建议先绕过无线模块，验证 STM32H0 固件和 UART 协议：

```powershell
python tune_console.py --serial-port COM8 --baudrate 115200 --loop-id 1
```

接线：

```text
USB-TTL/DAPLINK TX  -> STM32H0 UART_RX
USB-TTL/DAPLINK RX  <- STM32H0 UART_TX
GND                 --- STM32H0 GND
```

使用 3.3 V TTL 电平，不要将 RS-232 电平直接接入 STM32H0。

## 交互命令

```text
select 1
set kp 10.0
set ki 0
set kd 0.15
set aux 0
send
step kp up
step kd down
show
stats
quit
```

`step` 会根据最近遥测帧中的 `abs(error)` 自动选择步长：大误差 `1.0`、中等误差 `0.2`、小误差 `0.05`。

记录遥测数据：

```powershell
python tune_console.py --tcp dsp_link <TCP_PORT> --log logs\run.csv
```

## 参数持久化

默认配置文件是当前目录下的 `config.json`。程序启动时自动加载，输入 `quit`、按 `Ctrl+C` 或执行 `save` 时保存全部控制环参数。保存采用临时文件替换，避免程序退出时留下半个 JSON 文件。

```powershell
python tune_console.py --tcp dsp_link <TCP_PORT> --config my_pid.json
```

配置文件格式：

```json
{
  "version": 1,
  "loops": {
    "1": {"kp": 10.0, "ki": 0.0, "kd": 0.15, "aux": 0},
    "2": {"kp": 1.0, "ki": 0.1, "kd": 0.0, "aux": 0}
  }
}
```

命令行显式传入的 `--kp`、`--ki`、`--kd`、`--aux` 会覆盖所选环从配置文件加载的值；未传入的参数保留配置文件值。

## MCU 协议要求

下行参数帧固定 18 字节：

```text
AA FF Loop_ID kp(float) ki(float) kd(float) aux(int16) checksum
```

所有多字节数据均为小端序，校验为 `sum(frame[0..16]) & 0xFF`。

上行遥测帧固定 20 字节，帧头为 `AA FE`，包含 `error`、`output`、`measurement`、`revision` 和状态位。只有遥测中的 `revision` 变化后，才表示 MCU 已在控制周期边界应用新参数。

无线链路只负责参数和遥测传输，不参与 5 ms 控制闭环。STM32H0 必须在 UART 中断/DMA 中接收数据，并在主循环或控制周期边界安全提交完整参数快照。
