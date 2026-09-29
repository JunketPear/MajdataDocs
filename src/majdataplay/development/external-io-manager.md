# 外部IO管理器

::: info

`Pipe`输入在MajdataPlay`2.0.4-20260930-b939e80`中可用
:::

本文档说明如何开发一个独立的外部IO管理器，通过命名管道与MajdataPlay的`Pipe`设备后端通信。

外部管理器可以接管以下设备：

- 外键输入
- 触摸输入
- LED输出

> 本文以当前实现中的 `PipePacket`、`InputManager.ButtonRing`、`InputManager.TouchPanel` 和 `OutputManager.LedDevice` 为协议依据。若注释与实际序列化代码不一致，应以代码实际读写顺序为准。

## 1. 原理

MajdataPlay会作为**命名管道客户端**连接外部IO管理器。因此，外部IO管理器必须为每个启用的设备创建对应的**命名管道服务端**。

通信关系如下：

| 设备 | 数据方向 | MajdataPlay 的行为 | 管理器的职责 |
| --- | --- | --- | --- |
| 外键 | 外部管理器 → MajdataPlay | 读取状态包 | 周期性或状态变化时发送按键位图 |
| 触摸 | 外部管理器 → MajdataPlay | 读取状态包 | 周期性或状态变化时发送触摸位图 |
| LED | MajdataPlay → 外部管理器 | 发送颜色包或心跳包 | 读取数据并更新实际灯光 |

虽然管道使用双向模式 `InOut`，当前输入设备实现只从管道读取，LED实现只向管道写入。

## 2. 管道名称

管道名称包含Player编号

`{PlayerIndex}` 通常为 `1` 或 `2`。

| 设备 | 管道名称 |
| --- | --- |
| 外键 | `MajdataPlay.IO.ButtonRing.{PlayerIndex}P` |
| 触摸 | `MajdataPlay.IO.TouchPanel.{PlayerIndex}P` |
| LED | `MajdataPlay.IO.Led.{PlayerIndex}P` |

例如，1P 的三个完整管道名称分别为：

```text
\\.\pipe\MajdataPlay.IO.ButtonRing.1P
\\.\pipe\MajdataPlay.IO.TouchPanel.1P
\\.\pipe\MajdataPlay.IO.Led.1P
```

在dotnet中创建服务端时，只需将不带`\\.\pipe\`前缀的名称传给`NamedPipeServerStream`。

## 3. 通用数据包格式

所有设备均使用相同的 `PipePacket` 帧格式。多字节整数采用**小端序**。

| 偏移 | 长度 | 字段 | 说明 |
| ---: | ---: | --- | --- |
| `0` | 2 bytes | Identity | 固定为 `0x0448`，线上字节为 `48 04` |
| `2` | 1 bytes | Type | `0x00` 心跳，`0x01` 报告 |
| `3` | 2 bytes | Version | 协议版本，默认写入 `0` |
| `5` | 2 bytes | Length | Payload 长度，小端序 |
| `7` | Length bytes | Payload | 设备数据 |

固定包头长度为 **7 bytes**。

### 3.1 数据包类型

| 值 | 名称 | 用途 |
| ---: | --- | --- |
| `0x00` | HeartBeat | 保持连接活跃；Payload 通常为空 |
| `0x01` | Report | 携带按键、触摸或灯光数据 |

输入端会直接忽略心跳包。未知类型目前不会被解析器拒绝，但设备回调仅应处理已定义类型，因此实现方不应发送其他类型。

### 3.2 帧同步与分包

命名管道是字节流，单次读取不保证正好得到一个完整包。管理器必须支持：

- 一个包被拆分为多次读取；
- 一次读取包含多个包；
- 在数据中搜索同步字节 `48 04`；
- 收到异常长度后丢弃错误帧并重新同步；
- 保留末尾不完整数据，等待下一次读取。

MajdataPlay输入端允许的最大Payload为`1024 bytes`

外键与触摸的有效Payload必须严格为`8 bytes`。

## 4. 外键输入

### 4.1 Payload

按键环报告是一个 **8 字节小端无符号整数**。每一位表示一个按钮状态：

- `1`：按下；
- `0`：释放。

报告包总长度为 `7 + 8 = 15` 字节。

### 4.2 位定义

| Bit | 按钮 |
| ---: | --- |
| 0 | BA1 |
| 1 | BA2 |
| 2 | BA3 |
| 3 | BA4 |
| 4 | BA5 |
| 5 | BA6 |
| 6 | BA7 |
| 7 | BA8 |
| 8 | Test |
| 9 | Select P1 |
| 10 | Service |
| 11 | Select P2 |
| 12–63 | 保留，建议写 `0` |

例如，BA1、BA3 和 Test 同时按下时：

```text
mask = (1 << 0) | (1 << 2) | (1 << 8) = 0x0000000000000105
payload = 05 01 00 00 00 00 00 00
```

完整报告包为：

```text
48 04 01 00 00 08 00 05 01 00 00 00 00 00 00
```

### 4.3 发送

- 在连接建立后立即发送一次完整状态。
- 状态变化时立即发送，不要只发送变化的按钮。
- 即使状态不变，也建议每100–500 ms发送一次完整状态或心跳。
- 断开连接后MajdataPlay会清空全部按钮状态，并将释放事件标记为已发生。

## 5. 触摸输入

### 5.1 Payload

触摸报告同样是一个 **8 字节小端无符号整数**。当前实现读取 Bit 0–34，共 35 个物理传感器状态。

- `1`：正在触摸；
- `0`：未触摸。

### 5.2 位定义

| Bit 范围 | 区域 |
| --- | --- |
| 0–7 | A1–A8 |
| 8–15 | B1–B8 |
| 16–17 | C1–C2 |
| 18–25 | D1–D8 |
| 26–33 | E1–E8 |
| 34 | 当前实现保留的内部位，公开区域映射不使用，建议写 `0` |
| 35–63 | 保留，建议写 `0` |

> MajdataPlay逻辑中的 `SensorArea.C` 会合并 C1 和 C2：任一按下即视为 C 区按下；两者都释放才视为 C 区释放。管理器仍应分别发送 Bit 16 和 Bit 17。

例如，A1、B1、C1、D1、E1 被触摸时：

```text
mask = (1 << 0) | (1 << 8) | (1 << 16) | (1 << 18) | (1 << 26)
     = 0x0000000004050101
payload = 01 01 05 04 00 00 00 00
```

### 5.3 发送

- 每个报告必须包含所有34个公开传感器的完整快照；内部保留位 34 建议置零。
- 触摸输入对延迟敏感，建议在硬件产生新数据后立即发送。
- 不建议仅依赖低频心跳；空闲时可发送上次报告的状态或心跳。
- 断开连接后MajdataPlay会清空全部触摸状态，并记录释放状态。

## 6. LED输出

### 6.1 数据方向

LED管道由MajdataPlay写入，管理器读取。连接成功后，MajdataPlay会强制发送一次八个灯区的完整颜色。之后是否只发送变化项取决于 `Throttler` 设置。

### 6.2 Report Payload

Payload 由零个或多个 4 字节记录连续组成：

| 记录偏移 | 长度 | 字段 | 说明 |
| ---: | ---: | --- | --- |
| `0` | 1 bytes | LED Index | 灯区编号 `0–7` |
| `1` | 1 bytes | Red | 红色分量 `0–255` |
| `2` | 1 bytes | Green | 绿色分量 `0–255` |
| `3` | 1 bytes | Blue | 蓝色分量 `0–255` |

因此：

- Payload 长度应为 4 的倍数；
- 单包最多包含 8 条记录，即 32 字节 Payload；
- 未出现在包中的灯区应保持上一次颜色；
- 外部管理器应忽略或记录超出 `0–7` 的索引，避免访问越界。

例如，将 LED 0 设置为红色、LED 3 设置为蓝色：

```text
payload = 00 FF 00 00 03 00 00 FF
packet  = 48 04 01 00 00 08 00 00 FF 00 00 03 00 00 FF
```

### 6.3 心跳与刷新率

当没有颜色需要更新时，MajdataPlay会发送`7 bytes`心跳包：

```text
48 04 00 00 00 00 00
```

LED Pipe模式的发送周期使用LED `RefreshRateMs`，但最大会被限制为 500 ms。因此外部管理器应持续读取管道，不能只在预期颜色变化时读取。

### 6.4 亮度说明

当前MajdataPlay的Pipe实现直接把Unity颜色分量乘以255后发送，并未在Pipe序列化阶段应用全局 `_brightness`。

管理器应把收到的RGB视为最终的8位颜色值；若需要额外亮度控制，应在硬件适配层中明确实现。

## 7. 连接、重连与生命周期

MajdataPlay的Pipe客户端具有以下行为：

1. 等待全局设备重连间隔；
2. 尝试连接对应管道，连接超时约 2000 ms；
3. 连接后持续读写；
4. 遇到 EOF、`IOException` 或其他通信异常后关闭连接；
5. 返回外层循环并再次尝试连接。

建议管理器：

- 每个设备和Player使用独立服务端循环；
- 客户端断开后释放当前 `NamedPipeServerStream`，再创建新实例等待重连；
- 不要因为单个设备断开而终止整个IO管理器；
- 使用取消令牌支持正常退出；
- 对输入设备保留最新硬件状态，并在MajdataPlay重连后立即重发；
- 对LED设备在断开时选择安全策略，例如保持最后颜色或熄灭。

## 8. C# 参考实现

以下代码展示协议的编解码方式

```csharp
using System.Buffers.Binary;

public enum PipePacketType : byte
{
    HeartBeat = 0,
    Report = 1
}

public readonly record struct PipeFrame(
    PipePacketType Type,
    ushort Version,
    byte[] Payload);

public static class MajdataPipeProtocol
{
    public const int HeaderLength = 7;

    public static byte[] Encode(
        PipePacketType type,
        ReadOnlySpan<byte> payload,
        ushort version = 0)
    {
        if (payload.Length > ushort.MaxValue)
            throw new ArgumentOutOfRangeException(nameof(payload));

        var packet = new byte[HeaderLength + payload.Length];
        packet[0] = 0x48;
        packet[1] = 0x04;
        packet[2] = (byte)type;
        BinaryPrimitives.WriteUInt16LittleEndian(packet.AsSpan(3, 2), version);
        BinaryPrimitives.WriteUInt16LittleEndian(
            packet.AsSpan(5, 2), (ushort)payload.Length);
        payload.CopyTo(packet.AsSpan(HeaderLength));
        return packet;
    }

    public static byte[] EncodeState(ulong state)
    {
        Span<byte> payload = stackalloc byte[8];
        BinaryPrimitives.WriteUInt64LittleEndian(payload, state);
        return Encode(PipePacketType.Report, payload);
    }

    public static bool TryReadFrame(
        ref List<byte> buffer,
        out PipeFrame frame,
        int maxPayloadLength = 1024)
    {
        frame = default;

        while (true)
        {
            var sync = -1;
            for (var i = 0; i + 1 < buffer.Count; i++)
            {
                if (buffer[i] == 0x48 && buffer[i + 1] == 0x04)
                {
                    sync = i;
                    break;
                }
            }

            if (sync < 0)
            {
                var keep48 = buffer.Count > 0 && buffer[^1] == 0x48;
                buffer = keep48 ? new List<byte> { 0x48 } : new List<byte>();
                return false;
            }

            if (sync > 0)
                buffer.RemoveRange(0, sync);

            if (buffer.Count < HeaderLength)
                return false;

            var length = buffer[5] | (buffer[6] << 8);
            if (length > maxPayloadLength)
            {
                buffer.RemoveRange(0, 2);
                continue;
            }

            var packetLength = HeaderLength + length;
            if (buffer.Count < packetLength)
                return false;

            var payload = buffer.GetRange(HeaderLength, length).ToArray();
            var version = (ushort)(buffer[3] | (buffer[4] << 8));
            frame = new PipeFrame((PipePacketType)buffer[2], version, payload);
            buffer.RemoveRange(0, packetLength);
            return true;
        }
    }
}
```

### 8.1 输入设备服务端示例

```csharp
using System.IO.Pipes;

static async Task RunButtonRingServerAsync(
    int playerIndex,
    Func<ulong> readButtonMask,
    CancellationToken cancellationToken)
{
    var pipeName = $"MajdataPlay.IO.ButtonRing.{playerIndex}P";

    while (!cancellationToken.IsCancellationRequested)
    {
        await using var pipe = new NamedPipeServerStream(
            pipeName,
            PipeDirection.InOut,
            1,
            PipeTransmissionMode.Byte,
            PipeOptions.Asynchronous);

        await pipe.WaitForConnectionAsync(cancellationToken);

        while (pipe.IsConnected && !cancellationToken.IsCancellationRequested)
        {
            var packet = MajdataPipeProtocol.EncodeState(readButtonMask());
            await pipe.WriteAsync(packet, cancellationToken);
            await pipe.FlushAsync(cancellationToken);
            await Task.Delay(5, cancellationToken);
        }
    }
}
```

触摸服务端只需改用管道名 `MajdataPlay.IO.TouchPanel.{playerIndex}P`，并让状态函数返回Bit 0–33的触摸位图，Bit 34置零。

### 8.2 LED数据处理示例

```csharp
static void ApplyLedReport(ReadOnlySpan<byte> payload, Span<Rgb24> leds)
{
    if (payload.Length % 4 != 0)
        return;

    for (var offset = 0; offset < payload.Length; offset += 4)
    {
        var index = payload[offset];
        if (index >= leds.Length)
            continue;

        leds[index] = new Rgb24(
            payload[offset + 1],
            payload[offset + 2],
            payload[offset + 3]);
    }
}

public readonly record struct Rgb24(byte R, byte G, byte B);
```

## 9. 配置要求

在MajdataPlay的IO配置中，需要：

1. 将Manufacturer设置为 `Pipe`；
2. 启用所需的按键环、触摸面板和 LED 设备；
3. 确认玩家编号与外部管理器创建的管道名称一致；
4. 根据硬件能力设置输入轮询率与 LED 刷新率；
5. 在启动MajdataPlay前或启动后尽快创建管道服务端。客户端会自动重试，因此启动顺序并非强制。

## 10. 兼容性与安全建议

- 当前实现面向Windows Named Pipe；其他平台需要额外传输层。
- 将`Version`写为`0`，除非双方未来明确支持其他版本。
- 始终校验 Payload 长度、LED 索引及保留位。
- 不要假设一次`Read`返回完整数据包。
- 限制缓存大小，避免异常客户端导致无限内存增长。
- 只允许本机可信进程访问管道，并按部署环境设置合适的Pipe ACL。
- 硬件操作应与协议线程隔离，避免慢速USB/串口写入阻塞管道读取。

## 11. 联调检查表

- [ ] 三个管道名称与Player编号正确。
- [ ] 包头固定字节为 `48 04`。
- [ ] Version 和 Length 使用小端序。
- [ ] 按键环和触摸报告 Payload 恰好为 8 字节。
- [ ] 按键位映射为 Bit 0–11。
- [ ] 触摸位映射为 Bit 0–33，Bit 34 置零。
- [ ] LED Payload 按 `[Index, R, G, B]` 四字节分组。
- [ ] 能处理粘包、拆包和无效同步字节。
- [ ] MajdataPlay 重启或断线后可以自动重连。
- [ ] 空闲 LED 连接能持续接收心跳。
- [ ] 退出时能够取消等待并释放所有管道和硬件资源。
