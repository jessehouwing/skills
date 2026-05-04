---
name: aspnetcore-middleware-services-net11
description: >
  Covers new .NET 11 APIs for ASP.NET Core middleware and services:
  zero-allocation data protection (ISpanDataProtector, ISpanAuthenticatedEncryptor),
  Zstandard response compression (ZstandardCompressionProvider), and passkey
  AAGUID support in Identity (UserPasskeyInfo.Aaguid, IdentityPasskeyData.Aaguid).
  Use when configuring high-performance data protection, enabling zstd HTTP
  response compression, or working with passkey authenticator identification
  in .NET 11 ASP.NET Core applications.
license: MIT
---

# ASP.NET Core Middleware & Services — .NET 11 APIs

Target framework: `net11.0`

---

## Zero-Allocation Data Protection
Assembly: `Microsoft.AspNetCore.DataProtection.Abstractions`,
`Microsoft.AspNetCore.DataProtection`

### ISpanDataProtector

Extends `IDataProtector` with span-based protect/unprotect using
`IBufferWriter<byte>` to avoid heap allocations.

```csharp
public interface ISpanDataProtector : IDataProtector, IDataProtectionProvider
{
    void Protect<TWriter>(ReadOnlySpan<byte> plaintext, ref TWriter destination)
        where TWriter : IBufferWriter<byte>;
    void Unprotect<TWriter>(ReadOnlySpan<byte> protectedData, ref TWriter destination)
        where TWriter : IBufferWriter<byte>;
}
```

### ISpanAuthenticatedEncryptor

Extends `IAuthenticatedEncryptor` with span-based encrypt/decrypt.

```csharp
public interface ISpanAuthenticatedEncryptor : IAuthenticatedEncryptor
{
    void Encrypt<TWriter>(ReadOnlySpan<byte> plaintext,
        ReadOnlySpan<byte> aad, ref TWriter destination)
        where TWriter : IBufferWriter<byte>;
    void Decrypt<TWriter>(ReadOnlySpan<byte> ciphertext,
        ReadOnlySpan<byte> aad, ref TWriter destination)
        where TWriter : IBufferWriter<byte>;
}
```

### Example: Zero-Allocation Protect/Unprotect

```csharp
using System.Buffers;
using Microsoft.AspNetCore.DataProtection;

IDataProtector protector = provider.CreateProtector("MyPurpose");

if (protector is ISpanDataProtector spanProtector)
{
    ReadOnlySpan<byte> plaintext = "sensitive data"u8;

    var protectWriter = new ArrayBufferWriter<byte>();
    spanProtector.Protect(plaintext, ref protectWriter);

    var unprotectWriter = new ArrayBufferWriter<byte>();
    spanProtector.Unprotect(protectWriter.WrittenSpan, ref unprotectWriter);
}
```

### Example: Minimal API with Fallback

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddDataProtection();
var app = builder.Build();

app.MapGet("/protect", (IDataProtectionProvider provider) =>
{
    var protector = provider.CreateProtector("Demo");
    if (protector is ISpanDataProtector spanProtector)
    {
        var writer = new ArrayBufferWriter<byte>();
        spanProtector.Protect("secret"u8, ref writer);
        return Results.Ok(Convert.ToBase64String(writer.WrittenSpan));
    }
    return Results.Ok(protector.Protect("secret"));
});

app.Run();
```

---

## Zstandard Response Compression
Assembly: `Microsoft.AspNetCore.ResponseCompression`

### ZstandardCompressionProvider

Implements `ICompressionProvider` for Zstandard compression.

| Member | Type | Description |
|--------|------|-------------|
| `EncodingName` | `string` | Returns `"zstd"` |
| `SupportsFlush` | `bool` | Whether the provider supports flushing |
| `CreateStream(Stream)` | `Stream` | Wraps output with zstd compression |

### ZstandardCompressionProviderOptions

| Property | Type | Description |
|----------|------|-------------|
| `CompressionOptions` | `ZstandardCompressionOptions` | Quality and settings |

### Example: Basic Zstd Compression

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddResponseCompression(options =>
{
    options.Providers.Add<ZstandardCompressionProvider>();
    options.MimeTypes = ResponseCompressionDefaults.MimeTypes;
});

var app = builder.Build();
app.UseResponseCompression();
app.MapGet("/", () => "Hello, compressed with Zstandard!");
app.Run();
```

### Example: Combine with Brotli and Gzip

```csharp
builder.Services.AddResponseCompression(options =>
{
    options.Providers.Add<ZstandardCompressionProvider>();
    options.Providers.Add<BrotliCompressionProvider>();
    options.Providers.Add<GzipCompressionProvider>();
    options.EnableForHttps = true;
});

builder.Services.Configure<ZstandardCompressionProviderOptions>(opt =>
    opt.CompressionOptions = new ZstandardCompressionOptions { Quality = 3 });
```

---

## Passkey AAGUID Support
Assembly: `Microsoft.Extensions.Identity.Core`, `Microsoft.Extensions.Identity.Stores`

The **AAGUID** (Authenticator Attestation Globally Unique Identifier) is a
16-byte value identifying the make and model of a passkey authenticator.

### New APIs

| Type | Property | Description |
|------|----------|-------------|
| `UserPasskeyInfo` | `byte[]? Aaguid` | AAGUID in Identity.Core |
| `IdentityPasskeyData` | `virtual byte[]? Aaguid` | AAGUID in Identity.Stores (overridable) |

### Example: Identify Authenticator

```csharp
using Microsoft.AspNetCore.Identity;

var passkeyInfo = new UserPasskeyInfo
{
    Aaguid = new byte[]
    {
        0xCB, 0x69, 0x48, 0x1E, 0x8F, 0xF7, 0x40, 0x39,
        0x93, 0xEC, 0x0A, 0x27, 0x29, 0xA1, 0x54, 0xA8
    }
};

if (passkeyInfo.Aaguid is { Length: 16 } aaguid)
{
    var guid = new Guid(aaguid);
    Console.WriteLine($"Authenticator AAGUID: {guid}");
}
```

### Example: Filter Passkeys by Authenticator

```csharp
private static readonly Dictionary<Guid, string> KnownAuthenticators = new()
{
    [new Guid("cb69481e-8ff7-4039-93ec-0a2729a154a8")] = "YubiKey 5 Series",
    [new Guid("08987058-cadc-4b81-b6e1-30de50dcbe96")] = "Windows Hello",
};

public static string GetAuthenticatorName(this UserPasskeyInfo passkey)
{
    if (passkey.Aaguid is not { Length: 16 } aaguid)
        return "Unknown Authenticator";
    var guid = new Guid(aaguid);
    return KnownAuthenticators.GetValueOrDefault(guid, $"Unknown ({guid})");
}
```
