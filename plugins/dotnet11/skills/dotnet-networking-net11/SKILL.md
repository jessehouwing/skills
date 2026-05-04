---
name: dotnet-networking-net11
description: >
  Covers new .NET 11 APIs in System.Net.Sockets and System.Net.Primitives.
  Includes the ConnectAlgorithm enum and Socket.ConnectAsync overload for Happy
  Eyeballs (RFC 8305) parallel IPv4/IPv6 connections, IPEndPoint UTF-8
  Parse/TryParse/TryFormat for zero-allocation endpoint handling, and the
  DecompressionMethods.Zstandard enum value for HttpClient. Use when implementing
  low-latency dual-stack connections, high-performance endpoint parsing, or
  enabling Zstandard HTTP decompression in .NET 11.
license: MIT
---

# Networking — .NET 11 APIs

Target framework: `net11.0`

---

## Happy Eyeballs Parallel Connect
Assembly: `System.Net.Sockets`

A new `Socket.ConnectAsync` overload and `ConnectAlgorithm` enum enable the
Happy Eyeballs algorithm (RFC 8305) for parallel IPv4/IPv6 connection attempts,
reducing connection latency on dual-stack networks.

### ConnectAlgorithm Enum

```csharp
namespace System.Net.Sockets;

public enum ConnectAlgorithm
{
    Default = 0,   // Sequential (existing behavior)
    Parallel = 1   // Happy Eyeballs: concurrent IPv4/IPv6 attempts
}
```

### Socket.ConnectAsync

```csharp
public static bool ConnectAsync(
    SocketType socketType, ProtocolType protocolType,
    SocketAsyncEventArgs e, ConnectAlgorithm algorithm);
```

### Example: Parallel Connect

```csharp
using System.Net;
using System.Net.Sockets;

var args = new SocketAsyncEventArgs();
args.RemoteEndPoint = new DnsEndPoint("example.com", 443);
args.Completed += (sender, e) =>
{
    if (e.SocketError == SocketError.Success)
    {
        Socket connected = e.ConnectSocket!;
        Console.WriteLine($"Connected to {connected.RemoteEndPoint}");
    }
    else
    {
        Console.WriteLine($"Connection failed: {e.SocketError}");
    }
};

Socket.ConnectAsync(SocketType.Stream, ProtocolType.Tcp,
    args, ConnectAlgorithm.Parallel);
```

### When to Use

- **Dual-stack hosts:** Hostname resolves to both IPv4 and IPv6 — parallel
  attempts use whichever succeeds first.
- **Latency-sensitive apps:** Reduces worst-case connection time when one
  address family is slower or unreachable.
- **Client applications:** HTTP clients, database connectors, and any scenario
  where connection setup latency matters.

---

## IPEndPoint UTF-8 Support
Assembly: `System.Net.Primitives`

New methods on `IPEndPoint` for parsing and formatting with UTF-8 byte spans,
avoiding string allocations in hot paths.

### New APIs

| Method | Returns | Description |
|--------|---------|-------------|
| `IPEndPoint.Parse(ReadOnlySpan<byte> utf8Text)` | `IPEndPoint` | Parse from UTF-8 bytes |
| `IPEndPoint.TryParse(ReadOnlySpan<byte> utf8Text, out IPEndPoint?)` | `bool` | Try-parse from UTF-8 bytes |
| `IPEndPoint.TryFormat(Span<byte> utf8Destination, out int bytesWritten)` | `bool` | Format to UTF-8 bytes |
| `IPEndPoint.TryFormat(Span<char> destination, out int charsWritten)` | `bool` | Format to char span |

### Example

```csharp
using System.Net;

// Parse from UTF-8 bytes
ReadOnlySpan<byte> utf8 = "192.168.1.1:8080"u8;
IPEndPoint endpoint = IPEndPoint.Parse(utf8);

// TryParse
if (IPEndPoint.TryParse("10.0.0.1:443"u8, out IPEndPoint? parsed))
    Console.WriteLine($"Address: {parsed.Address}, Port: {parsed.Port}");

// TryFormat to UTF-8
Span<byte> buffer = stackalloc byte[64];
if (endpoint.TryFormat(buffer, out int bytesWritten))
    Console.WriteLine($"Wrote {bytesWritten} UTF-8 bytes");

// TryFormat to char span
Span<char> charBuffer = stackalloc char[64];
if (endpoint.TryFormat(charBuffer, out int charsWritten))
    Console.WriteLine(charBuffer[..charsWritten]);
```

---

## DecompressionMethods.Zstandard
Assembly: `System.Net.Primitives`

A new enum value enables Zstandard (zstd) decompression in HTTP handlers.

| Member | Value | Description |
|--------|-------|-------------|
| `DecompressionMethods.Zstandard` | `8` | Enables Zstandard decompression |

### Example

```csharp
using System.Net;
using System.Net.Http;

// Enable Zstandard decompression
var handler = new HttpClientHandler
{
    AutomaticDecompression = DecompressionMethods.Zstandard
};
using var client = new HttpClient(handler);

// Combine multiple methods
var multiHandler = new HttpClientHandler
{
    AutomaticDecompression = DecompressionMethods.GZip
                           | DecompressionMethods.Brotli
                           | DecompressionMethods.Zstandard
};
```
