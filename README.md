# uuid

UUID values and secure UUID v4/v7 generation for GoML. Requires GoML 0.1.57 or newer.

The implementation uses two private `u64` fields and pure GoML parsing, formatting, ordering, hashing and bit layout. Secure entropy comes from `std::crypto::rand`; wall-clock time comes from `std::time`. There are no ecosystem or native binding dependencies.

```toml
[dependencies]
"ecosystem::uuid" = "0.1.0"
```

The dependency version is the version used by the ecosystem verification registry. This repository does not imply an immutable registry release has been published.

```goml
use ecosystem::uuid::{Uuid};

fn event_id() -> Result[string, string] {
    let id = Uuid::new_v7().map_err(|error| error.to_string())?;
    Result::Ok(id.to_string())
}
```

## API

| API | Behavior |
| --- | --- |
| `Uuid::parse(text)` | Exactly 36 ASCII bytes in `8-4-4-4-12` hexadecimal groups; accepts upper, lower or mixed case. |
| `Uuid::parse_simple(text)` | Exactly 32 hexadecimal bytes without hyphens. |
| `Uuid::parse_urn(text)` | Exactly `urn:uuid:` followed by the hyphenated form. Scheme and namespace are case insensitive. |
| `to_string()`, `hyphenated()`, `simple()`, `urn()` | Canonical lowercase output. |
| `Uuid::from_bytes(slice)`, `to_bytes()` | Exactly 16 bytes in network byte order; bytes are copied, never retained as a mutable alias. |
| `Uuid::from_guid_bytes(slice)`, `to_guid_bytes()` | Microsoft GUID byte layout: the first 4-, 2-, and 2-byte fields use little-endian order; the final eight bytes retain their order. Exactly 16 bytes, with independent storage. |
| `Uuid::from_u64_pair(high, low)`, `to_u64_pair()` | Two big-endian halves without version or variant rewriting. |
| `Uuid::nil()`, `Uuid::max()`, `is_nil()`, `is_max()` | All-zero and all-one special UUID values. `Default` is nil. |
| `variant()` | `Ncs`, `Rfc9562`, `Microsoft` or `Future`. |
| `version()` | The four version bits for the RFC variant, including currently unused values; `None` for other variants. |
| `unix_milliseconds()` | The 48-bit Unix timestamp for RFC-variant v7 values; `None` otherwise. |
| `Uuid::new_v4()`, `Uuid::new_v7()` | Secure random generation with recoverable errors. |
| `Uuid::v4_from_random(bytes)` | Exactly 16 entropy bytes; overwrites version and variant bits, leaving 122 random bits. |
| `Uuid::v7_from_random(milliseconds, bytes)` | Exactly 10 entropy bytes; the upper four bits of byte 0 and upper two bits of byte 2 are discarded, leaving 74 random bits. |
| `Generator::secure()` | Reusable generator using the standard cryptographic random source and system time. |
| `Generator::new(entropy, clock)` | Explicit entropy and millisecond-clock injection; `v4()` and `v7()` verify byte counts and propagate source errors. |

`Uuid` implements `PartialEq`, `Eq`, `Hash`, `std::cmp::PartialOrd`, `std::cmp::Ord`, `Debug`, `ToString`, `Default`, and Serde `Serialize`/`Deserialize`. `cmp()` compares all 128 bits in unsigned network order, matching canonical textual order. UUIDs can be used as `HashMap` keys. The hash is not a cryptographic digest.

Serde always represents UUIDs as canonical strings, including in binary formats. Deserialization accepts only the strict hyphenated form. Applications requiring a raw 16-byte database or wire representation should use `from_bytes`/`to_bytes` explicitly.

The parsers preserve every 128-bit value, including legacy variants, nil, max, and future versions. Parsing establishes representation validity; applications can inspect `variant()` and `version()` when a protocol requires v4 or v7 specifically. Whitespace, braces, malformed separators, non-ASCII digits and trailing data are rejected. Invalid-character positions are zero-based byte offsets in the original input.

## Generation guarantees

The v4/v7 layouts follow [RFC 9562](https://www.rfc-editor.org/rfc/rfc9562.html). Tests include its Appendix A.3 and A.6 reference vectors. Generation never falls back to a deterministic PRNG when secure entropy fails.

V7 accepts Unix milliseconds in `0..=281474976710655`. Negative and overflowing timestamps fail before entropy is requested. Each call uses fresh entropy for the remaining 74 bits. Values sort by their timestamp, but calls in the same millisecond have random order; clock rollback is not clamped. There is no shared counter, monotonicity promise, or absolute uniqueness guarantee. Secure generators may be shared between concurrent callers. Injected callbacks must themselves be safe for concurrent use if their generator is shared.

`v4_from_random`, `v7_from_random`, and `Generator::new` trust the caller to supply fresh cryptographically secure entropy when unpredictability matters. Deterministic bytes are useful for test fixtures, not production ID generation. UUIDs are identifiers, not authentication tokens; v7 also exposes creation time.

The library can parse and preserve every UUID version, but generates only v4 and v7. It does not generate MAC-address, name-based or custom-version UUIDs.

The explicit GUID methods interoperate with the byte layout returned by
[.NET `Guid.ToByteArray()`](https://learn.microsoft.com/en-us/dotnet/api/system.guid.tobytearray).
For example, `00112233-4455-6677-8899-aabbccddeeff` becomes
`33 22 11 00 55 44 77 66 88 99 aa bb cc dd ee ff`. This is a field-wise conversion,
not reversal of the entire UUID. It preserves every bit without checking or
rewriting the variant/version. Canonical text, Serde and `to_bytes()` keep their
existing network-order representation.

## Checks

```sh
goml fmt --check
goml check
goml test
GOFLAGS=-race goml test --target-dir _artifact/race
goml run --example basic
```

`examples/basic` is a separate consumer package that uses only public API. It stores typed JSON events with UUID keys; its tests share a secure generator across eight tasks. The root tests cover RFC vectors, every malformed digit/separator position, entropy failures, timestamp limits, variant boundaries, byte alias isolation, order, hashing and Serde rejection.
