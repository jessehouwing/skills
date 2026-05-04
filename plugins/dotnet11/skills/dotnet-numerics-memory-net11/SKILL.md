---
name: dotnet-numerics-memory-net11
description: >
  Covers new .NET 11 APIs in System.Memory and System.Runtime.Numerics
  for BFloat16 data pipelines. Includes BFloat16 BinaryPrimitives read/write
  methods (big-endian, little-endian, Try variants), BigInteger to/from BFloat16
  explicit conversions, BigInteger UTF-8 TryFormat/TryParse, and implicit
  Complex from BFloat16 conversion. Use when serializing ML tensor data,
  converting between arbitrary-precision integers and BFloat16, or performing
  zero-allocation BigInteger formatting in .NET 11.
license: MIT
---

# Numerics & Memory — .NET 11 APIs

Target framework: `net11.0`

---

## BFloat16 BinaryPrimitives
Assembly: `System.Memory`

Eight new methods on `System.Buffers.Binary.BinaryPrimitives` for reading and
writing `BFloat16` values in big-endian and little-endian byte order.

### Read Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `ReadBFloat16BigEndian(ReadOnlySpan<byte>)` | `BFloat16` | Read big-endian |
| `ReadBFloat16LittleEndian(ReadOnlySpan<byte>)` | `BFloat16` | Read little-endian |
| `TryReadBFloat16BigEndian(ReadOnlySpan<byte>, out BFloat16)` | `bool` | Safe big-endian read |
| `TryReadBFloat16LittleEndian(ReadOnlySpan<byte>, out BFloat16)` | `bool` | Safe little-endian read |

### Write Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `WriteBFloat16BigEndian(Span<byte>, BFloat16)` | `void` | Write big-endian |
| `WriteBFloat16LittleEndian(Span<byte>, BFloat16)` | `void` | Write little-endian |
| `TryWriteBFloat16BigEndian(Span<byte>, BFloat16)` | `bool` | Safe big-endian write |
| `TryWriteBFloat16LittleEndian(Span<byte>, BFloat16)` | `bool` | Safe little-endian write |

### Example: Basic Read/Write

```csharp
using System.Buffers.Binary;
using System.Numerics;

BFloat16 value = (BFloat16)3.14f;

Span<byte> buffer = stackalloc byte[2];
BinaryPrimitives.WriteBFloat16LittleEndian(buffer, value);
BFloat16 read = BinaryPrimitives.ReadBFloat16LittleEndian(buffer);
```

### Example: Batch Processing ML Tensor Data

```csharp
using System.Buffers.Binary;
using System.Numerics;

byte[] fileData = File.ReadAllBytes("weights.bin");
int count = fileData.Length / 2;
var weights = new BFloat16[count];

for (int i = 0; i < count; i++)
    weights[i] = BinaryPrimitives.ReadBFloat16LittleEndian(
        fileData.AsSpan(i * 2, 2));
```

---

## BigInteger ↔ BFloat16 Conversions
Assembly: `System.Runtime.Numerics`

Explicit conversion operators between `BigInteger` and `BFloat16`.

| Operator | Description |
|----------|-------------|
| `explicit operator BigInteger(BFloat16)` | Truncates fractional part |
| `explicit operator BFloat16(BigInteger)` | May lose precision |

```csharp
using System.Numerics;

BFloat16 bf = (BFloat16)42.0f;
BigInteger big = (BigInteger)bf;     // 42
BFloat16 back = (BFloat16)(BigInteger)1000;  // 1000 (approx)
```

---

## BigInteger UTF-8 TryFormat/TryParse
Assembly: `System.Runtime.Numerics`

Zero-allocation UTF-8 formatting and parsing for `BigInteger`.

| Method | Returns | Description |
|--------|---------|-------------|
| `TryFormat(Span<byte> utf8Destination, out int bytesWritten, ...)` | `bool` | Format to UTF-8 bytes |
| `TryParse(ReadOnlySpan<byte> utf8Text, NumberStyles, IFormatProvider?, out BigInteger)` | `bool` | Parse from UTF-8 with style |
| `TryParse(ReadOnlySpan<byte> utf8Text, out BigInteger)` | `bool` | Parse from UTF-8 (defaults) |

```csharp
using System.Numerics;

BigInteger value = BigInteger.Parse("123456789012345678901234567890");

Span<byte> utf8Buffer = stackalloc byte[64];
if (value.TryFormat(utf8Buffer, out int bytesWritten))
    Console.WriteLine($"Formatted {bytesWritten} UTF-8 bytes");

if (BigInteger.TryParse("99999999999999999999"u8, out BigInteger parsed))
    Console.WriteLine(parsed);
```

---

## Complex ← BFloat16 Implicit Conversion
Assembly: `System.Runtime.Numerics`

| Operator | Description |
|----------|-------------|
| `implicit operator Complex(BFloat16)` | Converts to Complex (imaginary = 0) |

```csharp
using System.Numerics;

BFloat16 bf = (BFloat16)2.5f;
Complex c = bf;  // (2.5, 0)
```
