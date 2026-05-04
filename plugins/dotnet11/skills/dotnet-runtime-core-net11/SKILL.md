---
name: dotnet-runtime-core-net11
description: >
  Covers new .NET 11 APIs in System.Runtime and
  System.Runtime.InteropServices. Includes BFloat16 numeric type, DivisionRounding
  enum, String/StringBuilder Rune methods, RunePosition, Char/Rune.Equals with
  StringComparison, Base64 char-based encode/decode, BitConverter BFloat16 support,
  File.CreateHardLink, File.OpenNullHandle, Uri.UriSchemeData, IdnMapping span
  methods, AsyncHelpers, ExtendedLayoutAttribute with CStruct/CUnion layouts,
  CurrencyWrapper un-deprecated, and PosixSignal.SIGKILL. Use when working with
  brain floating-point arithmetic, Unicode Rune text processing, Base64, file
  system hard links, or C-compatible interop layouts on net11.0.
license: MIT
---

# Runtime Core & Interop — .NET 11 APIs

Target framework: `net11.0`

---

## BFloat16 Type
Assembly: `System.Runtime`

`System.Numerics.BFloat16` is a 16-bit brain floating-point type for ML
workloads. Implements `IFloatingPointIeee754<BFloat16>` and all standard
numeric interfaces.

**Key members:** Arithmetic operators (`+`, `-`, `*`, `/`, `%`), conversions
to/from `float`, `double`, `Half`, `int`, `long`, etc., `Parse()`,
`TryParse()`, `ToString()`, `TryFormat()`, static helpers (`IsZero`,
`IsNaN`, `IsInfinity`, `IsNormal`, etc.), constants (`Epsilon`, `MaxValue`,
`MinValue`, `NaN`, `PositiveInfinity`, `NegativeInfinity`).

```csharp
using System.Numerics;

BFloat16 a = (BFloat16)1.5f;
BFloat16 b = (BFloat16)2.5f;
BFloat16 sum = a + b;
Console.WriteLine((float)sum); // 4.0
Console.WriteLine(BFloat16.IsNaN(a)); // false
```

---

## DivisionRounding Enum
Assembly: `System.Runtime`

Controls rounding behavior for integer division via new
`IBinaryInteger<TSelf>` methods: `Divide()`, `DivRem()`, `Remainder()`.

| Value | Description |
|-------|-------------|
| `Truncate` | Round toward zero (C# default) |
| `Floor` | Round toward negative infinity |
| `Ceiling` | Round toward positive infinity |
| `AwayFromZero` | Round away from zero |
| `Euclidean` | Non-negative remainder |

```csharp
int floor = int.Divide(-7, 2, DivisionRounding.Floor); // -4
var (q, r) = int.DivRem(-7, 2, DivisionRounding.Euclidean); // q=-4, r=1
```

---

## String Rune Methods
Assembly: `System.Runtime`

New overloads on `String` for first-class `Rune` support.

| Method | Overloads |
|--------|-----------|
| `Contains(Rune)`, `Contains(Rune, StringComparison)` | 2 |
| `IndexOf(Rune, ...)` | 8 |
| `LastIndexOf(Rune, ...)` | 8 |
| `StartsWith(Rune)`, `StartsWith(Rune, StringComparison)` | 2 |
| `EndsWith(Rune)`, `EndsWith(Rune, StringComparison)` | 2 |
| `Replace(Rune, Rune)` | 1 |
| `Split(Rune, ...)` | 2 |
| `Trim(Rune)`, `TrimStart(Rune)`, `TrimEnd(Rune)` | 3 |

```csharp
string text = "Hello 🌍 World";
Rune globe = new Rune(0x1F30D);

bool has = text.Contains(globe);           // true
int idx = text.IndexOf(globe);             // 6
string replaced = text.Replace(globe, new Rune('X'));
string[] parts = "a🌍b🌍c".Split(globe);
string trimmed = "***data***".Trim(new Rune('*'));
```

## Char.Equals and Rune.Equals
```csharp
char ch = 'a';
bool eq = ch.Equals('A', StringComparison.OrdinalIgnoreCase); // true

Rune r = new Rune('ß');
bool runeEq = r.Equals(new Rune('ß'), StringComparison.CurrentCulture);
```

---

## StringBuilder Rune Methods
| Method | Description |
|--------|-------------|
| `Append(Rune)` | Appends a Rune |
| `GetRuneAt(int)` | Gets the Rune at a position |
| `TryGetRuneAt(int, out Rune)` | Try-gets the Rune at a position |
| `Insert(int, Rune)` | Inserts a Rune |
| `Replace(Rune, Rune)` | Replaces all occurrences |
| `EnumerateRunes()` | Enumerates all Runes |

---

## RunePosition Struct
Enables enumerating Runes with position metadata.

| Member | Type | Description |
|--------|------|-------------|
| `Rune` | `Rune` | The Rune value |
| `StartIndex` | `int` | Start index in source |
| `Length` | `int` | Length in code units |
| `WasReplaced` | `bool` | Whether it was a replacement char |

Static methods: `RunePosition.EnumerateUtf16(ReadOnlySpan<char>)`,
`RunePosition.EnumerateUtf8(ReadOnlySpan<byte>)`.

---

## TextInfo and TextWriter Rune Support
```csharp
Rune lower = CultureInfo.InvariantCulture.TextInfo.ToLower(new Rune('A'));
Rune upper = CultureInfo.InvariantCulture.TextInfo.ToUpper(new Rune('a'));

using var writer = new StringWriter();
writer.Write(new Rune(0x1F30D));
await writer.WriteAsync(new Rune('X'));
writer.WriteLine(new Rune('!'));
```

---

## Base64 Improvements
Assembly: `System.Runtime`

`System.Buffers.Text.Base64` gains char-based and simplified overloads.

| Method | Description |
|--------|-------------|
| `EncodeToString(ReadOnlySpan<byte>)` | Encode bytes to Base64 string |
| `EncodeToChars(ReadOnlySpan<byte>, Span<char>, ...)` | Encode to char span |
| `DecodeFromChars(ReadOnlySpan<char>, Span<byte>, ...)` | Decode from char span |
| `EncodeToUtf8(ReadOnlySpan<byte>)` | Simplified UTF-8 encode |
| `GetEncodedLength(int)` / `GetMaxDecodedLength(int)` | Length calculations |
| `TryEncodeToChars` / `TryDecodeFromChars` / `TryEncodeToUtf8` / `TryDecodeFromUtf8` | Try variants |

```csharp
using System.Buffers.Text;

byte[] data = { 1, 2, 3, 4, 5 };
string b64 = Base64.EncodeToString(data); // AQIDBAU=

int len = Base64.GetEncodedLength(data.Length);
Span<char> buf = stackalloc char[len];
Base64.TryEncodeToChars(data, buf, out int written);
```

---

## BitConverter BFloat16 Support
| Method | Description |
|--------|-------------|
| `BFloat16ToInt16Bits(BFloat16)` | BFloat16 → Int16 bits |
| `Int16BitsToBFloat16(short)` | Int16 bits → BFloat16 |
| `GetBytes(BFloat16)` | BFloat16 → byte[] |
| `ToBFloat16(byte[], int)` | bytes → BFloat16 |

---

## File Operations
### Hard Links
| Method | Description |
|--------|-------------|
| `File.CreateHardLink(string path, string pathToTarget)` | Creates a hard link |
| `FileInfo.CreateAsHardLink(string pathToTarget)` | Creates file as hard link |

### Null Handle
| Method | Description |
|--------|-------------|
| `File.OpenNullHandle()` | Opens `SafeFileHandle` to null device |

```csharp
File.CreateHardLink("link.txt", "original.txt");
using var nullHandle = File.OpenNullHandle();
```

---

## Uri.UriSchemeData
```csharp
string scheme = Uri.UriSchemeData; // "data"
```

---

## IdnMapping Span Methods
| Method | Returns |
|--------|---------|
| `TryGetAscii(ReadOnlySpan<char>, Span<char>, out int)` | `bool` |
| `TryGetUnicode(ReadOnlySpan<char>, Span<char>, out int)` | `bool` |

```csharp
var idn = new IdnMapping();
Span<char> output = stackalloc char[256];
if (idn.TryGetAscii("münchen.de", output, out int written))
    Console.WriteLine(output[..written]); // xn--mnchen-3ya.de
```

---

## StringSyntaxAttribute Constants
| Constant | Value |
|----------|-------|
| `StringSyntaxAttribute.CSharp` | `"csharp"` |
| `StringSyntaxAttribute.FSharp` | `"fsharp"` |
| `StringSyntaxAttribute.VisualBasic` | `"visualbasic"` |

---

## AsyncHelpers
| Method | Description |
|--------|-------------|
| `AsyncHelpers.HandleAsyncEntryPoint(Task)` | Handles async Main returning Task |
| `AsyncHelpers.HandleAsyncEntryPoint(Task<int>)` | Handles async Main returning Task&lt;int&gt; |

---

## ExtendedLayoutAttribute & C-Compatible Layouts
Assembly: `System.Runtime`, `System.Runtime.InteropServices`

New attribute and enum for C-compatible struct and union memory layouts.

| Type | Description |
|------|-------------|
| `ExtendedLayoutAttribute` | Specifies extended layout kind |
| `ExtendedLayoutKind.CStruct` | C-compatible sequential struct layout |
| `ExtendedLayoutKind.CUnion` | C-compatible union (all fields at offset 0) |
| `LayoutKind.Extended` | New LayoutKind enum value |
| `TypeAttributes.ExtendedLayout` | New type attribute flag |

```csharp
using System.Runtime.InteropServices;

// C-compatible union: all fields overlap at offset 0
[ExtendedLayout(ExtendedLayoutKind.CUnion)]
struct MyUnion
{
    public int IntValue;
    public float FloatValue;
    public byte ByteValue;
}

// C-compatible struct with platform-specific padding
[ExtendedLayout(ExtendedLayoutKind.CStruct)]
struct MyCStruct
{
    public byte Flags;
    public int Value;    // padded per C ABI rules
    public short Extra;
}
```

---

## CurrencyWrapper Un-deprecated
Assembly: `System.Runtime.InteropServices`

The `[Obsolete]` attribute has been removed from `CurrencyWrapper`. It is
once again fully supported for COM interop currency values.

```csharp
var wrapper = new CurrencyWrapper(99.95m);
Console.WriteLine(wrapper.WrappedObject); // 99.95
```

---

## PosixSignal.SIGKILL
Assembly: `System.Runtime.InteropServices`

| Member | Value | Description |
|--------|-------|-------------|
| `PosixSignal.SIGKILL` | `-11` | Force-terminate a process |

> SIGKILL cannot be caught or ignored. This value is for sending signals
> to other processes.

```csharp
PosixSignal signal = PosixSignal.SIGKILL;
Console.WriteLine($"Signal value: {(int)signal}"); // -11
```
