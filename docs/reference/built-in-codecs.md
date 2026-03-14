# Built-in codecs

yorishiro-proxy includes 14 built-in codecs for encoding, decoding, and hashing payloads. You can use these codecs in [Fuzzer](../features/fuzzer.md), [Resender](../features/resender.md), and [Macro](../features/macros.md) encoding chains to transform payloads before sending.

## Codec catalog

| Name | Category | Reversible | Description |
|------|----------|------------|-------------|
| `base64` | Encoding | Yes | Standard Base64 (RFC 4648) |
| `base64url` | Encoding | Yes | URL-safe Base64 (RFC 4648 §5) |
| `url_encode_query` | Encoding | Yes | URL query encoding (spaces as `+`) |
| `url_encode_path` | Encoding | Yes | URL path encoding (spaces as `%20`) |
| `url_encode_full` | Encoding | Yes | Encode all non-alphanumeric characters to `%XX` |
| `double_url_encode` | Encoding | Yes | Apply URL query encoding twice |
| `hex` | Encoding | Yes | Hexadecimal encoding |
| `html_entity` | Encoding | Yes | Numeric HTML entities (`&#xNN;`) |
| `html_escape` | Encoding | Yes | Named HTML entities (`&amp;`, `&lt;`, etc.) |
| `unicode_escape` | Encoding | Yes | Unicode escape sequences (`\uXXXX`) |
| `md5` | Hash | No | MD5 hash (hex-encoded output) |
| `sha256` | Hash | No | SHA-256 hash (hex-encoded output) |
| `lower` | Case | No | Convert to lowercase |
| `upper` | Case | No | Convert to uppercase |

## Encoding codecs

### base64

Standard Base64 encoding per RFC 4648. Uses the standard alphabet (`A-Z`, `a-z`, `0-9`, `+`, `/`) with `=` padding.

| Direction | Input | Output |
|-----------|-------|--------|
| Encode | `Hello World` | `SGVsbG8gV29ybGQ=` |
| Decode | `SGVsbG8gV29ybGQ=` | `Hello World` |

### base64url

URL-safe Base64 encoding per RFC 4648 §5. Uses `-` and `_` instead of `+` and `/`, making it safe for URLs and filenames.

| Direction | Input | Output |
|-----------|-------|--------|
| Encode | `Hello World` | `SGVsbG8gV29ybGQ=` |
| Decode | `SGVsbG8gV29ybGQ=` | `Hello World` |

### url_encode_query

URL encoding for query string values. Spaces are encoded as `+`, special characters as `%XX`.

| Direction | Input | Output |
|-----------|-------|--------|
| Encode | `key=value&foo=bar baz` | `key%3Dvalue%26foo%3Dbar+baz` |
| Decode | `key%3Dvalue%26foo%3Dbar+baz` | `key=value&foo=bar baz` |

### url_encode_path

URL encoding for path segments. Spaces are encoded as `%20` (not `+`).

| Direction | Input | Output |
|-----------|-------|--------|
| Encode | `/path/with spaces` | `%2Fpath%2Fwith%20spaces` |
| Decode | `%2Fpath%2Fwith%20spaces` | `/path/with spaces` |

### url_encode_full

Encodes all characters except unreserved characters (RFC 3986 §2.3: `A-Z`, `a-z`, `0-9`, `-`, `.`, `_`, `~`) to `%XX` format. Useful for WAF bypass testing.

| Direction | Input | Output |
|-----------|-------|--------|
| Encode | `<script>` | `%3Cscript%3E` |
| Decode | `%3Cscript%3E` | `<script>` |

### double_url_encode

Applies URL query encoding twice. The `%` characters from the first encoding are themselves encoded, producing `%25XX` sequences. Useful for testing applications that decode URL encoding multiple times.

| Direction | Input | Output |
|-----------|-------|--------|
| Encode | `<script>` | `%253Cscript%253E` |
| Decode | `%253Cscript%253E` | `<script>` |

### hex

Hexadecimal encoding of the raw bytes.

| Direction | Input | Output |
|-----------|-------|--------|
| Encode | `ABC` | `414243` |
| Decode | `414243` | `ABC` |

### html_entity

Encodes every character as a numeric HTML entity (`&#xNN;`). Useful for XSS payload construction and WAF bypass.

| Direction | Input | Output |
|-----------|-------|--------|
| Encode | `<img>` | `&#x3C;&#x69;&#x6D;&#x67;&#x3E;` |
| Decode | `&#x3C;&#x69;&#x6D;&#x67;&#x3E;` | `<img>` |

### html_escape

Encodes the five special HTML characters using named entities: `&` → `&amp;`, `<` → `&lt;`, `>` → `&gt;`, `"` → `&#34;`, `'` → `&#39;`.

| Direction | Input | Output |
|-----------|-------|--------|
| Encode | `<a href="x">` | `&lt;a href=&#34;x&#34;&gt;` |
| Decode | `&lt;a href=&#34;x&#34;&gt;` | `<a href="x">` |

### unicode_escape

Encodes characters as `\uXXXX` Unicode escape sequences. Supplementary characters (above U+FFFF) use surrogate pairs.

| Direction | Input | Output |
|-----------|-------|--------|
| Encode | `Hello` | `\u0048\u0065\u006C\u006C\u006F` |
| Decode | `\u0048\u0065\u006C\u006C\u006F` | `Hello` |

## Hash codecs

Hash codecs are **one-way** — `Encode` produces a hash, but `Decode` returns an error (`ErrIrreversible`). The output is always a hex-encoded string.

### md5

Computes the MD5 digest (128-bit) and returns a 32-character hex string.

| Direction | Input | Output |
|-----------|-------|--------|
| Encode | `password` | `5f4dcc3b5aa765d61d8327deb882cf99` |

### sha256

Computes the SHA-256 digest (256-bit) and returns a 64-character hex string.

| Direction | Input | Output |
|-----------|-------|--------|
| Encode | `password` | `5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8` |

## Case codecs

Case codecs are **one-way** — the original casing cannot be recovered, so `Decode` returns an error.

### lower

Converts all characters to lowercase.

| Direction | Input | Output |
|-----------|-------|--------|
| Encode | `Hello World` | `hello world` |

### upper

Converts all characters to uppercase.

| Direction | Input | Output |
|-----------|-------|--------|
| Encode | `Hello World` | `HELLO WORLD` |

## Encoding chains

You can combine multiple codecs into a chain. Codecs are applied in order during encoding and in reverse order during decoding.

### How chains work

When you specify `encoding: ["base64", "url_encode_query"]`:

1. **Encode**: value → base64 → url_encode_query → encoded payload
2. **Decode**: encoded payload → url_encode_query (decode) → base64 (decode) → original value

### Usage in MCP tools

Specify encoding chains in the `encoding` parameter of fuzzer payload sets, resender body patches, or macro steps:

```json
// fuzz
{
  "flow_id": "abc123",
  "positions": [
    {
      "location": "body",
      "name": "payload",
      "start": 10,
      "end": 20
    }
  ],
  "payload_sets": [
    {
      "name": "payload",
      "type": "wordlist",
      "values": ["<script>alert(1)</script>", "' OR 1=1 --"],
      "encoding": ["base64"]
    }
  ]
}
```

### Practical chain examples

#### Base64 + URL encoding

Encode a payload in Base64, then URL-encode the result for safe inclusion in a query parameter:

```json
"encoding": ["base64", "url_encode_query"]
```

`<script>` → `PHNjcmlwdD4=` → `PHNjcmlwdD4%3D`

#### Double URL encoding for WAF bypass

```json
"encoding": ["double_url_encode"]
```

`<script>` → `%253Cscript%253E`

#### Hash generation

Generate SHA-256 hashes of wordlist entries (e.g., for testing hash-based authentication):

```json
"encoding": ["sha256"]
```

`password123` → `ef92b778bafe771e89245b89ecbc08a44a4e166c06659911881f383d4473e94f`

#### Unicode escape for filter bypass

```json
"encoding": ["unicode_escape"]
```

`alert` → `\u0061\u006C\u0065\u0072\u0074`

## Custom codec plugins

You can extend the codec registry with custom codecs written in Starlark. Custom codecs are registered through the `codec_plugins` section in the config file and become available in encoding chains alongside built-in codecs.

See [Codec plugins](../plugins/codec-plugins.md) for details on writing custom codecs.

!!! warning "Name conflicts"
    If a custom codec uses the same name as a built-in codec, registration will fail with an error. Choose unique names for custom codecs.

## Related pages

- [Fuzzer](../features/fuzzer.md) — Use encoding chains with payload sets
- [Resender](../features/resender.md) — Apply codecs to body patches
- [Macros](../features/macros.md) — Use codecs in macro steps
- [Codec plugins](../plugins/codec-plugins.md) — Write custom codecs in Starlark
