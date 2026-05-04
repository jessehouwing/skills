---
name: system-io-compression-net11
description: >
  Provides guidance on new Zstandard (zstd) compression APIs and ZipArchiveEntry
  improvements added to System.IO.Compression in .NET 11.
  It covers ZstandardStream for stream-based compression and decompression,
  ZstandardEncoder and ZstandardDecoder for low-level operations,
  ZstandardCompressionOptions and ZstandardDictionary for configuration,
  and new ZipArchiveEntry.Open overloads with FileAccess and async support.
  Use when working with zstd-compressed data, configuring advanced compression
  options, or leveraging the improved ZipArchiveEntry API in .NET 11.
license: MIT
---

# System.IO.Compression — Zstandard Support & ZipArchiveEntry Improvements

Target framework: `net11.0`

## Overview

.NET 11 introduces first-class Zstandard (zstd) compression support in
`System.IO.Compression`. Additionally, `ZipArchiveEntry` gains new methods for
opening entries with explicit file-access modes and async support.

## New APIs

### ZstandardStream

Stream-based Zstandard compression and decompression. Extends `Stream`.

**Constructors:**

| Signature | Description |
|-----------|-------------|
| `ZstandardStream(Stream, CompressionLevel)` | Compress with a standard level |
| `ZstandardStream(Stream, CompressionLevel, bool leaveOpen)` | Compress, optionally leave base stream open |
| `ZstandardStream(Stream, CompressionMode)` | Decompress |
| `ZstandardStream(Stream, CompressionMode, bool leaveOpen)` | Decompress, optionally leave base stream open |
| `ZstandardStream(Stream, ZstandardCompressionOptions, bool leaveOpen)` | Compress with advanced options |
| `ZstandardStream(Stream, ZstandardEncoder, bool leaveOpen)` | Compress with a pre-configured encoder |
| `ZstandardStream(Stream, ZstandardDecoder, bool leaveOpen)` | Decompress with a pre-configured decoder |
| `ZstandardStream(Stream, ZstandardDictionary, CompressionLevel)` | Compress using a pre-trained dictionary |

**Properties:** `BaseStream`, `CanRead`, `CanWrite`, `CanSeek`

**Methods:** `SetSourceLength(long)` — hint the uncompressed size for better compression.

### ZstandardEncoder

Low-level encoder. Implements `IDisposable`.

**Constructors:**

- `ZstandardEncoder()` — default quality
- `ZstandardEncoder(int quality)`
- `ZstandardEncoder(int quality, int windowLog)`
- `ZstandardEncoder(ZstandardCompressionOptions options)`
- `ZstandardEncoder(ZstandardDictionary dictionary)`

**Instance methods:** `Compress()`, `Flush()`, `Reset()`, `SetPrefix()`,
`SetSourceLength()`, `GetMaxCompressedLength()`

**Static methods:** `TryCompress(ReadOnlySpan<byte>, Span<byte>, out int)`

### ZstandardDecoder

Low-level decoder. Implements `IDisposable`.

**Constructors:**

- `ZstandardDecoder()`
- `ZstandardDecoder(int maxWindowLog)`
- `ZstandardDecoder(ZstandardDictionary dictionary)`
- `ZstandardDecoder(ZstandardDictionary dictionary, int maxWindowLog)`

**Instance methods:** `Decompress()`, `Reset()`, `SetPrefix()`

**Static methods:** `TryDecompress(ReadOnlySpan<byte>, Span<byte>, out int)`,
`TryGetMaxDecompressedLength(ReadOnlySpan<byte>, out long)`

### ZstandardCompressionOptions

Configuration for Zstandard compression.

| Property | Type | Description |
|----------|------|-------------|
| `Quality` | `int` | Compression quality level |
| `WindowLog` | `int` | Window log size |
| `EnableLongDistanceMatching` | `bool` | Enable LDM for large inputs |
| `AppendChecksum` | `bool` | Append integrity checksum |
| `TargetBlockSize` | `int` | Target block size hint |
| `Dictionary` | `ZstandardDictionary?` | Pre-trained dictionary |

**Static fields:** `DefaultQuality`, `MinQuality`, `MaxQuality`,
`DefaultWindowLog`, `MinWindowLog`, `MaxWindowLog`

### ZstandardDictionary

Pre-trained dictionary for improved compression of small, similar payloads.
Implements `IDisposable`.

| Member | Description |
|--------|-------------|
| `Create(ReadOnlySpan<byte>)` | Create from raw dictionary data (static) |
| `Train(ReadOnlySpan<byte>[], int)` | Train a dictionary from sample data (static) |
| `Data` | The raw dictionary bytes (property) |

### ZipArchiveEntry Improvements

| Member | Description |
|--------|-------------|
| `Open(FileAccess access)` | Open with explicit read or write access |
| `OpenAsync(FileAccess, CancellationToken)` | Async open with access mode |
| `CompressionMethod` | Get the `ZipCompressionMethod` used |

### ZipCompressionMethod Enum

| Value | Description |
|-------|-------------|
| `Stored` | No compression |
| `Deflate` | Deflate algorithm |
| `Deflate64` | Enhanced Deflate |

## Examples

### Basic ZstandardStream Compression and Decompression

```csharp
using System.IO;
using System.IO.Compression;

// Compress a file with ZstandardStream
await using var sourceStream = File.OpenRead("data.bin");
await using var output = File.Create("data.zst");
await using (var zstStream = new ZstandardStream(output, CompressionLevel.Optimal))
{
    await sourceStream.CopyToAsync(zstStream);
}

// Decompress a file with ZstandardStream
await using var input = File.OpenRead("data.zst");
await using var zstDecompress = new ZstandardStream(input, CompressionMode.Decompress);
await using var result = new MemoryStream();
await zstDecompress.CopyToAsync(result);
Console.WriteLine($"Decompressed {result.Length} bytes");
```

### Using ZstandardEncoder and ZstandardDecoder Directly

```csharp
using System.IO.Compression;

byte[] sourceData = File.ReadAllBytes("data.bin");

// Compress with the low-level encoder
var encoder = new ZstandardEncoder(quality: 3);
byte[] compressed = new byte[encoder.GetMaxCompressedLength(sourceData.Length)];
bool ok = ZstandardEncoder.TryCompress(sourceData, compressed, out int written);
Console.WriteLine($"Compressed {sourceData.Length} → {written} bytes");

// Decompress with the low-level decoder
byte[] decompressed = new byte[sourceData.Length];
bool decoded = ZstandardDecoder.TryDecompress(
    compressed.AsSpan(0, written), decompressed, out int decompressedSize);
Console.WriteLine($"Decompressed {decompressedSize} bytes");
```

### Advanced Options and Dictionary Training

```csharp
using System.IO.Compression;

// Configure advanced compression options
var options = new ZstandardCompressionOptions
{
    Quality = 6,
    WindowLog = 22,
    EnableLongDistanceMatching = true,
    AppendChecksum = true
};

await using var src = File.OpenRead("large-dataset.bin");
await using var dst = File.Create("large-dataset.zst");
await using var zst = new ZstandardStream(dst, options, leaveOpen: false);
await src.CopyToAsync(zst);
```

### ZipArchiveEntry New Methods

```csharp
using System.IO.Compression;

// Inspect and read zip entries with new APIs
using var archive = ZipFile.OpenRead("archive.zip");
foreach (var entry in archive.Entries)
{
    // Query the compression method
    Console.WriteLine($"{entry.FullName}: {entry.CompressionMethod}");

    // Open with explicit access mode
    using var stream = entry.Open(FileAccess.Read);
    using var reader = new StreamReader(stream);
    string content = await reader.ReadToEndAsync();
}

// Async open
using var archive2 = ZipFile.OpenRead("archive.zip");
var first = archive2.Entries[0];
await using var asyncStream = await first.OpenAsync(
    FileAccess.Read, CancellationToken.None);
```
