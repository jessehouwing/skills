---
name: dotnet-security-cryptography-net11
description: >
  Covers new .NET 11 APIs in System.Security.Cryptography and
  System.Security.Cryptography.Pkcs. Includes constant-time hash verification
  methods (VerifyHmac, HMAC/KMAC/IncrementalHash Verify), and cross-platform
  CMS/PKCS#7 signing (SignedCms), encryption (EnvelopedCms), PKCS#12 archive
  building, PKCS#8 private key info, and RFC 3161 timestamps. Use when
  implementing hash verification, digital signatures, certificate-based
  encryption, or PFX/PKCS#12 operations in .NET 11 applications.
license: MIT
---

# Security & Cryptography — .NET 11 APIs

Target framework: `net11.0`

---

## Constant-Time Hash Verification
All HMAC, KMAC, and incremental hash classes now include built-in `Verify()`
methods with constant-time comparison, eliminating manual `FixedTimeEquals`.

### Migration Pattern

```csharp
// ❌ Before .NET 11
byte[] computed = HMACSHA256.HashData(key, data);
bool valid = CryptographicOperations.FixedTimeEquals(computed, expected);

// ✅ .NET 11
bool valid = HMACSHA256.Verify(key, data, expected);
```

### CryptographicOperations.VerifyHmac

Static one-shot HMAC verification with algorithm selection.

| Method | Description |
|--------|-------------|
| `VerifyHmac(HashAlgorithmName, byte[] key, byte[] source, byte[] hash)` | Byte array overload |
| `VerifyHmac(HashAlgorithmName, ReadOnlySpan<byte>, ReadOnlySpan<byte>, ReadOnlySpan<byte>)` | Span overload |
| `VerifyHmac(HashAlgorithmName, byte[] key, Stream source, byte[] hash)` | Stream overload |
| `VerifyHmacAsync(HashAlgorithmName, byte[] key, Stream, byte[], CancellationToken)` | Async stream |

```csharp
byte[] key = RandomNumberGenerator.GetBytes(32);
byte[] data = "Hello, World!"u8.ToArray();
byte[] hash = HMACSHA256.HashData(key, data);

bool ok = CryptographicOperations.VerifyHmac(
    HashAlgorithmName.SHA256, key, data, hash);
```

### HMAC Classes: Verify and VerifyAsync

Added to: `HMACMD5`, `HMACSHA1`, `HMACSHA256`, `HMACSHA384`, `HMACSHA512`,
`HMACSHA3_256`, `HMACSHA3_384`, `HMACSHA3_512`.

Each class gains:

| Method | Description |
|--------|-------------|
| `Verify(byte[] key, byte[] source, byte[] hash)` | Byte array verify |
| `Verify(ReadOnlySpan<byte>, ReadOnlySpan<byte>, ReadOnlySpan<byte>)` | Span verify |
| `VerifyAsync(byte[] key, Stream, byte[], CancellationToken)` | Async stream |
| `VerifyAsync(ReadOnlyMemory<byte>, Stream, ReadOnlyMemory<byte>, CancellationToken)` | Async memory |

```csharp
byte[] key = RandomNumberGenerator.GetBytes(32);
byte[] data = "Sensitive payload"u8.ToArray();
byte[] expected = HMACSHA256.HashData(key, data);

bool valid = HMACSHA256.Verify(key, data, expected);

// Async stream verification
await using var stream = File.OpenRead("data.bin");
bool streamValid = await HMACSHA256.VerifyAsync(key, stream, expected);
```

### IncrementalHash: VerifyCurrentHash and VerifyHashAndReset

| Method | Description |
|--------|-------------|
| `VerifyCurrentHash(ReadOnlySpan<byte>)` | Verifies without resetting state |
| `VerifyHashAndReset(ReadOnlySpan<byte>)` | Verifies and resets for reuse |

```csharp
using var hash = IncrementalHash.CreateHash(HashAlgorithmName.SHA256);
hash.AppendData("Part 1"u8);
hash.AppendData("Part 2"u8);
bool match = hash.VerifyHashAndReset(expectedHash);
```

### KMAC Classes: Verify Methods

`Kmac128`, `Kmac256`, `KmacXof128`, `KmacXof256` all gain `Verify()`,
`VerifyAsync()`, `VerifyCurrentHash()`, and `VerifyHashAndReset()`.

```csharp
byte[] key = RandomNumberGenerator.GetBytes(32);
byte[] data = "KMAC input"u8.ToArray();
byte[] customization = "MyApp"u8.ToArray();
byte[] kmacHash = Kmac128.HashData(key, data, 32, customization);

bool valid = Kmac128.Verify(key, data, kmacHash, customization);
```

---

## CMS/PKCS#7 — Cross-Platform
Assembly `System.Security.Cryptography.Pkcs` is now available cross-platform
for ASP.NET Core in .NET 11, removing the previous Windows-only limitation.

### SignedCms (CMS/PKCS#7 Signing)

| Member | Description |
|---|---|
| `SignedCms(ContentInfo)` | Constructs a signed CMS message |
| `ComputeSignature(CmsSigner)` | Computes and adds a signature |
| `CheckSignature(bool)` | Verifies all signatures |
| `Decode(byte[])` / `Encode()` | Round-trip encoding |
| `SignerInfos` | Collection of signer info objects |
| `Certificates` | Embedded certificates |

Supports RSA, ECDsa, and post-quantum algorithms (MLDsa, SlhDsa,
CompositeMLDsa — experimental `SYSLIB5006`).

```csharp
byte[] data = Encoding.UTF8.GetBytes("Hello, cross-platform CMS!");
var cms = new SignedCms(new ContentInfo(data));
cms.ComputeSignature(new CmsSigner(SubjectIdentifierType.IssuerAndSerialNumber, cert));
byte[] signed = cms.Encode();

// Verify
var verify = new SignedCms();
verify.Decode(signed);
verify.CheckSignature(verifySignatureOnly: true);
```

### EnvelopedCms (CMS Encryption)

| Member | Description |
|---|---|
| `EnvelopedCms(ContentInfo)` | Constructs an enveloped message |
| `Encrypt(CmsRecipient)` | Encrypts for a recipient |
| `Decrypt()` | Decrypts using certs from the store |
| `ContentEncryptionAlgorithm` | Algorithm used for encryption |

```csharp
var env = new EnvelopedCms(new ContentInfo(plaintext));
env.Encrypt(new CmsRecipient(recipientCert));
byte[] encrypted = env.Encode();

var dec = new EnvelopedCms();
dec.Decode(encrypted);
dec.Decrypt();
byte[] decrypted = dec.ContentInfo.Content;
```

### PKCS#12 (PFX) Archive Building

| Type | Description |
|---|---|
| `Pkcs12Builder` | Builds a PKCS#12/PFX archive |
| `Pkcs12Info` | Reads PKCS#12 data |
| `Pkcs12SafeContents` | Container for safe bags |
| `Pkcs12CertBag` / `Pkcs12KeyBag` / `Pkcs12ShroudedKeyBag` | Bag types |

```csharp
var builder = new Pkcs12Builder();
var contents = new Pkcs12SafeContents();
var pbeParams = new PbeParameters(
    PbeEncryptionAlgorithm.Aes256Cbc, HashAlgorithmName.SHA256, 100_000);

contents.AddCertificate(cert);
contents.AddShroudedKey(key, "password", pbeParams);
builder.AddSafeContentsEncrypted(contents, "password", pbeParams);
builder.SealWithMac("password", HashAlgorithmName.SHA256, 100_000);
byte[] pfx = builder.Encode();
```

### Additional Types

- **PKCS#8:** `Pkcs8PrivateKeyInfo` for private key information
- **PKCS#9 Attributes:** `Pkcs9ContentType`, `Pkcs9SigningTime`, `Pkcs9MessageDigest`,
  `Pkcs9DocumentName`, `Pkcs9DocumentDescription`, `Pkcs9LocalKeyId`
- **RFC 3161:** `Rfc3161TimestampRequest`, `Rfc3161TimestampToken`,
  `Rfc3161TimestampTokenInfo`
- **Recipients:** `CmsRecipient`, `CmsRecipientCollection`,
  `KeyTransRecipientInfo`, `KeyAgreeRecipientInfo`
