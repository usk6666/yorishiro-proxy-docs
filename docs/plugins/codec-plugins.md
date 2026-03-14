# Custom codec plugins

In addition to hook plugins, yorishiro-proxy supports **codec plugins** -- Starlark scripts that define custom encode/decode transformations. Once registered, custom codecs integrate seamlessly with the built-in codec registry and can be used in the fuzzer, resender, macro templates, and from other Starlark plugins via the `codec` module.

## File format

A codec plugin is a `.star` file that defines:

- `name` (string, required): The name used to reference this codec in encoding chains
- `encode(s)` (function, required): Takes a string, returns the encoded string
- `decode(s)` (function, optional): Takes a string, returns the decoded string. If not defined, decode operations return an error

### Reversible codec example

```python
# codecs/sql_escape.star
name = "sql_escape"

def encode(s):
    return s.replace("'", "''")

def decode(s):
    return s.replace("''", "'")
```

### Encode-only codec example

For irreversible transformations, you can omit the `decode` function:

```python
# codecs/rot13.star
name = "rot13"

_upper = "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
_lower = "abcdefghijklmnopqrstuvwxyz"

def encode(s):
    result = []
    for c in s.elems():
        i = _upper.find(c)
        if i >= 0:
            result.append(_upper[(i + 13) % 26])
        else:
            i = _lower.find(c)
            if i >= 0:
                result.append(_lower[(i + 13) % 26])
            else:
                result.append(c)
    return "".join(result)
```

## Configuration

Codec plugins are configured in the `codec_plugins` section of the config file. Each entry specifies a `path` to a `.star` file or a directory containing `.star` files.

```json
{
  "codec_plugins": [
    {"path": "codecs/sql_escape.star"},
    {"path": "codecs/"}
  ]
}
```

- **File path**: Loads the specified `.star` file
- **Directory path**: Loads all `*.star` files in the directory (non-recursive)
- Paths are relative to the working directory

## Name collision

If a codec plugin defines a `name` that conflicts with a built-in codec (e.g., `base64`, `url_encode_query`) or another already-loaded codec plugin, the proxy returns an error at startup. Choose unique names for custom codecs.

## Error handling

| Error type | Behavior |
|-----------|----------|
| Syntax error in `.star` file | Warning logged, codec skipped |
| Missing `name` variable | Warning logged, codec skipped |
| Missing `encode` function | Warning logged, codec skipped |
| `encode`/`decode` runtime error | Error propagated to the encoding chain caller |
| Infinite loop (exceeds step limit) | Execution halted, error returned |
| Name conflict with existing codec | Startup error |

## Using custom codecs

Once loaded, custom codecs are available everywhere built-in codecs are used.

### In fuzzer encoding chains

```json
// fuzz
{
  "action": "fuzz",
  "params": {
    "flow_id": "...",
    "targets": [{"location": "query", "name": "q"}],
    "payloads": {
      "type": "values",
      "values": ["admin'--", "' OR 1=1--"],
      "encoding": ["sql_escape", "url_encode_query"]
    }
  }
}
```

### In resender

Custom codecs can be referenced in resender encoding chains the same way as built-in codecs.

### In macro templates

```
{{payload | sql_escape | url_encode_query}}
```

### In Starlark hook plugins via the `codec` module

The `codec` module automatically includes custom codecs:

```python
# Direct encode/decode
encoded = codec.sql_escape("admin'--")          # "admin''--"
decoded = codec.sql_escape_decode("admin''--")   # "admin'--"

# Chain encoding
result = codec.encode("admin'--", ["sql_escape", "url_encode_query"])
```

## Related pages

- [Overview](overview.md) -- Plugin system overview
- [Writing plugins](writing-plugins.md) -- How to write hook plugins
- [Examples](examples.md) -- Plugin samples including codec examples
- [Built-in codecs](../reference/built-in-codecs.md) -- List of built-in codecs
- [Fuzzer](../features/fuzzer.md) -- Using codecs in fuzzer encoding chains
