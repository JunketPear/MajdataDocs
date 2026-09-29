# External IO Manager

::: info

`Pipe` IO mode is available in MajdataPlay `2.0.4-20260930-b939e80`
:::

This document explains how to develop a standalone external IO manager that communicates with the MajdataPlay `Pipe` device backend through Windows Named Pipes. The external manager can provide the following devices:

- Button ring input
- Touch panel input
- Eight-zone LED output

> This document is based on the current implementations of `PipePacket`, `InputManager.ButtonRing`, `InputManager.TouchPanel`, and `OutputManager.LedDevice`. If a source comment differs from the actual serialization code, the byte order used by the code takes precedence.

## 1. Architecture

When the configured device manufacturer is `Pipe`, MajdataPlay acts as a **Named Pipe client**. The external IO manager must therefore create a **Named Pipe server** for every enabled device.

| Device | Data direction | MajdataPlay behavior | External manager responsibility |
| --- | --- | --- | --- |
| Button ring | External manager → MajdataPlay | Reads state packets | Sends a complete button bitmap |
| Touch panel | External manager → MajdataPlay | Reads state packets | Sends a complete touch bitmap |
| LED | MajdataPlay → External manager | Sends color or heartbeat packets | Reads packets and updates the physical LEDs |

The pipes are opened with bidirectional `InOut` access. However, the current input implementations only read from their pipes, while the LED implementation only writes to its pipe.

## 2. Pipe Names

Pipe names include the player index. `{PlayerIndex}` is normally `1` or `2`.

| Device | Pipe name |
| --- | --- |
| Button ring | `MajdataPlay.IO.ButtonRing.{PlayerIndex}P` |
| Touch panel | `MajdataPlay.IO.TouchPanel.{PlayerIndex}P` |
| LED | `MajdataPlay.IO.Led.{PlayerIndex}P` |

For player 1, the full Windows paths are:

```text
\\.\pipe\MajdataPlay.IO.ButtonRing.1P
\\.\pipe\MajdataPlay.IO.TouchPanel.1P
\\.\pipe\MajdataPlay.IO.Led.1P
```

When using .NET, pass the name without the `\\.\pipe\` prefix to `NamedPipeServerStream`.

## 3. Common Packet Format

All devices use the same `PipePacket` frame format. Multi-byte integers are encoded in **little-endian** order.

| Offset | Size | Field | Description |
| ---: | ---: | --- | --- |
| `0` | 2 bytes | Identity | Fixed value `0x0448`; bytes on the wire are `48 04` |
| `2` | 1 byte | Type | `0x00` heartbeat, `0x01` report |
| `3` | 2 bytes | Version | Protocol version; current sender defaults to `0` |
| `5` | 2 bytes | Length | Payload length, little-endian |
| `7` | Length bytes | Payload | Device-specific data |

The fixed header length is **7 bytes**.

### 3.1 Packet Types

| Value | Name | Purpose |
| ---: | --- | --- |
| `0x00` | HeartBeat | Keeps the connection active; normally has an empty payload |
| `0x01` | Report | Carries button, touch, or LED data |

Input handlers ignore heartbeat packets. The parser does not currently reject unknown type values, but integrations should only send the defined types.

### 3.2 Framing and Stream Handling

A Named Pipe is a byte stream. One read operation is not guaranteed to return exactly one packet. Implementations must handle:

- one packet split across multiple reads;
- multiple packets returned by one read;
- synchronization by searching for `48 04`;
- invalid lengths followed by resynchronization;
- incomplete trailing data retained for the next read.

MajdataPlay input parsers allow a maximum payload of 1024 bytes. Valid button ring and touch panel report payloads must be exactly 8 bytes.

## 4. Button Ring Input

### 4.1 Payload

A button ring report contains one **8-byte little-endian unsigned integer**. Each bit represents one button:

- `1`: pressed;
- `0`: released.

The complete report packet is `7 + 8 = 15` bytes.

### 4.2 Bit Mapping

| Bit | Button |
| ---: | --- |
| 0 | A1 |
| 1 | A2 |
| 2 | A3 |
| 3 | A4 |
| 4 | A5 |
| 5 | A6 |
| 6 | A7 |
| 7 | A8 |
| 8 | Test |
| 9 | Select P1 |
| 10 | Service |
| 11 | Select P2 |
| 12–63 | Reserved; write `0` |

For example, if A1, A3, and Test are pressed:

```text
mask = (1 << 0) | (1 << 2) | (1 << 8) = 0x0000000000000105
payload = 05 01 00 00 00 00 00 00
```

The complete packet is:

```text
48 04 01 00 00 08 00 05 01 00 00 00 00 00 00
```

### 4.3 Sending Recommendations

- Send one complete state immediately after the connection is established.
- Send immediately when state changes; do not send only the changed buttons.
- Even when unchanged, send a complete state or heartbeat every 100–500 ms.
- When disconnected, MajdataPlay clears all button states and records release transitions.

## 5. Touch Panel Input

### 5.1 Payload

A touch panel report also contains one **8-byte little-endian unsigned integer**. The current implementation reads bits 0–34, representing 35 physical sensor states.

- `1`: touched;
- `0`: not touched.

### 5.2 Bit Mapping

| Bit range | Area |
| --- | --- |
| 0–7 | A1–A8 |
| 8–15 | B1–B8 |
| 16–17 | C1–C2 |
| 18–25 | D1–D8 |
| 26–33 | E1–E8 |
| 34 | Internal reserved bit retained by the current implementation; it is not used by the public area mapping, so write `0` |
| 35–63 | Reserved; write `0` |

> `SensorArea.C` in game logic combines C1 and C2. It is considered on when either sensor is on, and off only when both sensors are off. The external manager must still transmit bits 16 and 17 separately.

For example, if A1, B1, C1, D1, and E1 are touched:

```text
mask = (1 << 0) | (1 << 8) | (1 << 16) | (1 << 18) | (1 << 26)
     = 0x0000000004050101
payload = 01 01 05 04 00 00 00 00
```

### 5.3 Sending Recommendations

- Every report should contain a complete snapshot of the 34 public sensors; write `0` to internal reserved bit 34.
- Touch input is latency-sensitive; send as soon as new hardware data is available.
- Do not rely only on a low-frequency heartbeat. Send an all-zero state or heartbeat while idle.
- When disconnected, MajdataPlay clears all touch states and records release states.

## 6. LED Output

### 6.1 Data Direction

MajdataPlay writes to the LED pipe and the external manager reads from it. After connecting, MajdataPlay forces one full update containing all eight LED zones. Later packets may contain only changed zones when `Throttler` is enabled.

### 6.2 Report Payload

The payload contains zero or more consecutive 4-byte records:

| Record offset | Size | Field |
| ---: | ---: | --- |
| `0` | 1 byte | LED Index | Zone index `0–7` |
| `1` | 1 byte | Red | Red component `0–255` |
| `2` | 1 byte | Green | Green component `0–255` |
| `3` | 1 byte | Blue | Blue component `0–255` |

Consequently:

- payload length should be a multiple of 4;
- one packet can contain up to 8 records, or 32 payload bytes;
- zones omitted from a packet retain their previous colors;
- out-of-range indices should be ignored or logged to avoid invalid memory access.

For example, set LED 0 to red and LED 3 to blue:

```text
payload = 00 FF 00 00 03 00 00 FF
packet  = 48 04 01 00 00 08 00 00 FF 00 00 03 00 00 FF
```

### 6.3 Heartbeats and Refresh Rate

When no color needs to be updated, MajdataPlay sends a 7-byte heartbeat packet:

```text
48 04 00 00 00 00 00
```

Pipe-mode LED transmission uses the configured LED `RefreshRateMs`, capped at 500 ms. The external manager must continuously read the pipe instead of reading only when a color change is expected.

### 6.4 Brightness Behavior

The current Pipe LED implementation multiplies Unity color components directly by 255. It does not apply the global `_brightness` value during Pipe serialization. Treat the received RGB values as final 8-bit color values. If additional brightness control is required, implement it explicitly in the hardware adapter.

## 7. Connection, Reconnection, and Lifecycle

The MajdataPlay Pipe client behaves as follows:

1. Waits for the global IO reconnection interval.
2. Attempts to connect to the corresponding pipe with an approximately 2000 ms timeout.
3. Continuously reads or writes after connecting.
4. Closes the connection after EOF, `IOException`, or another communication failure.
5. Returns to the outer loop and attempts to connect again.

Recommended external manager behavior:

- Run an independent server loop for every device and player.
- After a client disconnects, dispose the current `NamedPipeServerStream`, create a new one, and wait for reconnection.
- Do not terminate the entire IO manager because one device disconnected.
- Use cancellation tokens for clean shutdown.
- Retain the latest physical input state and resend it immediately after MajdataPlay reconnects.
- Choose an explicit LED disconnect policy, such as retaining the last color or turning all LEDs off.

## 8. C# Reference Implementation

The following code demonstrates the core protocol encoding and decoding logic. Production code should also include logging, timeouts, cancellation, exception isolation, and hardware driver integration.

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

### 8.1 Input Server Example

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

A touch panel server uses `MajdataPlay.IO.TouchPanel.{playerIndex}P` and returns a touch bitmap in bits 0–33, with bit 34 set to zero.

### 8.2 LED Report Handling Example

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

## 9. Configuration Requirements

In the MajdataPlay/NapCat IO configuration:

1. Set the device manufacturer to `Pipe`.
2. Enable the required button ring, touch panel, and LED devices.
3. Ensure the player index matches the pipe names created by the external manager.
4. Configure input polling and LED refresh rates for the attached hardware.
5. Create the pipe servers before MajdataPlay starts or shortly afterward. Startup order is not strict because the clients automatically retry.

## 10. Compatibility and Security Recommendations

- The current implementation targets Windows Named Pipes; other platforms require another transport layer.
- Write `0` to `Version` unless both sides explicitly support another version.
- Validate payload lengths, LED indices, and reserved bits.
- Never assume one `Read` returns one complete packet.
- Limit accumulation buffer size to prevent unbounded memory use.
- Restrict pipe access to trusted local processes and configure appropriate Pipe ACLs for the deployment environment.
- Isolate hardware operations from the protocol loop so slow USB or serial writes do not block pipe reads.

## 11. Integration Checklist

- [ ] All pipe names use the correct player index.
- [ ] Header identity bytes are `48 04`.
- [ ] Version and Length are little-endian.
- [ ] Button and touch report payloads are exactly 8 bytes.
- [ ] Button states use bits 0–11.
- [ ] Touch states use bits 0–33, with bit 34 set to zero.
- [ ] LED records use `[Index, R, G, B]` groups.
- [ ] The parser handles packet coalescing, packet splitting, and invalid synchronization bytes.
- [ ] The manager automatically recovers after MajdataPlay restarts or disconnects.
- [ ] The LED server continuously consumes heartbeat packets while idle.
- [ ] Shutdown cancels pending waits and releases every pipe and hardware resource.
