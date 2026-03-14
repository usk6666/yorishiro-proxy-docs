# Writing plugins

This guide covers how to write Starlark hook plugins for yorishiro-proxy. You will learn the basic structure of a plugin, the available built-in modules, return formats, error handling, and debugging techniques.

## Starlark basics

Starlark is a subset of Python designed for configuration and extension. Key differences from Python:

- No `import` statement -- use the predeclared modules (`action`, `codec`, `crypto`, `config`, `state`, `store`, `proxy`)
- No classes, only functions and basic data types
- Dictionaries, lists, tuples, strings, ints, floats, booleans, `None`
- `print()` outputs to the proxy log
- No file I/O, network access, or system calls
- All module-level variables are frozen after script load (see [Module-level freeze constraint](#module-level-freeze-constraint))

## Basic structure

A plugin is a `.star` file that defines one or more hook functions. Each function receives a `data` dictionary containing protocol-specific keys and returns a result dictionary.

```python
def on_receive_from_client(data):
    # Inspect or modify the data
    url = data.get("url", "")
    print("Request to: %s" % url)

    # Return an action
    return {"action": action.CONTINUE, "data": data}
```

## The `action` module

Every plugin has access to the predeclared `action` module with three constants:

| Constant | Description |
|----------|-------------|
| `action.CONTINUE` | Continue processing with the (optionally modified) data |
| `action.DROP` | Silently drop the connection (only in `on_receive_from_client`) |
| `action.RESPOND` | Send a custom response to the client (only in `on_receive_from_client`, HTTP/HTTPS only) |

## Return format

Hook functions must return one of:

- `None` (or no explicit return) -- treated as `action.CONTINUE` with no modifications
- A dictionary with the following structure:

```python
# Continue with modified data
{"action": action.CONTINUE, "data": modified_data}

# Continue without modifications
{"action": action.CONTINUE}

# Drop the connection
{"action": action.DROP}

# Respond directly (HTTP/HTTPS only, on_receive_from_client only)
{
    "action": action.RESPOND,
    "response": {
        "status_code": 200,
        "headers": {"Content-Type": "application/json"},
        "body": '{"ok": true}',
    },
}
```

### RESPOND response fields

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `status_code` | int | Yes | HTTP status code |
| `headers` | dict | No | Response headers |
| `body` | string | No | Response body |

## The `config` module

The `config` dict provides read-only access to the `vars` map defined in the plugin configuration. This lets you pass external configuration (e.g., API keys, environment settings) to your plugin without hardcoding values.

```json
{
  "plugins": [
    {
      "path": "plugins/add_auth_header.star",
      "protocol": "http",
      "hooks": ["on_before_send_to_server"],
      "vars": {
        "token": "Bearer eyJhbGciOiJIUzI1NiJ9.example",
        "environment": "staging"
      }
    }
  ]
}
```

```python
def on_before_send_to_server(data):
    token = config.get("token", "")
    if token:
        headers = data.get("headers", {})
        headers["Authorization"] = token
        data["headers"] = headers
    return {"action": action.CONTINUE, "data": data}
```

The `config` dict is frozen -- attempting to modify it raises a runtime error.

## The `state` module

The `state` module provides a per-plugin in-memory key-value store. State is volatile (lost on process restart) but survives plugin reloads. Each plugin has its own isolated namespace.

| Function | Description |
|----------|-------------|
| `state.get(key)` | Returns the value for `key`, or `None` if not found |
| `state.set(key, value)` | Sets `key` to `value`. Only primitives: string, bytes, int, float, bool |
| `state.delete(key)` | Deletes `key`. No-op if `key` does not exist |
| `state.keys()` | Returns a list of all keys |
| `state.clear()` | Removes all keys |

Resource limits:

- Maximum 10,000 keys per plugin
- Maximum 1 MB per string/bytes value

```python
def on_receive_from_client(data):
    url = data.get("url", "")
    count = state.get("request_count")
    if count == None:
        count = 0
    state.set("request_count", count + 1)
    print("Request #%d: %s" % (count + 1, url))
    return {"action": action.CONTINUE}
```

## The `store` module

The `store` module provides a per-plugin persistent key-value store backed by SQLite. It has the same API as `state` but values survive process restarts. Only string and bytes values are accepted.

| Function | Description |
|----------|-------------|
| `store.get(key)` | Returns the value for `key`, or `None` if not found |
| `store.set(key, value)` | Sets `key` to `value`. Only string and bytes are accepted |
| `store.delete(key)` | Deletes `key`. No-op if `key` does not exist |
| `store.keys()` | Returns a list of all keys |
| `store.clear()` | Removes all keys for this plugin |

Resource limits are the same as `state`: max 10,000 keys, max 1 MB per value.

!!! note
    The `store` module is only available when the proxy is started with a database connection. If no database is configured, `store` is not injected into the Starlark runtime.

## The `codec` module

The `codec` module exposes encode/decode functions for all registered codecs (both built-in and custom codec plugins).

```python
# Encode/decode with a specific codec
encoded = codec.base64("hello")             # "aGVsbG8="
decoded = codec.base64_decode("aGVsbG8=")   # "hello"

# Chain encoding/decoding
result = codec.encode("payload", ["url_encode_query", "base64"])
original = codec.decode(result, ["url_encode_query", "base64"])

# List all available codecs
names = codec.list()
```

## The `crypto` module

The `crypto` module provides cryptographic functions wrapping Go's standard library. All hash and HMAC functions accept `bytes` and return `bytes`.

### Hash functions

| Function | Description |
|----------|-------------|
| `crypto.md5(data)` | MD5 hash |
| `crypto.sha1(data)` | SHA-1 hash |
| `crypto.sha256(data)` | SHA-256 hash |
| `crypto.sha512(data)` | SHA-512 hash |

### HMAC functions

| Function | Description |
|----------|-------------|
| `crypto.hmac_md5(key, message)` | HMAC-MD5 |
| `crypto.hmac_sha1(key, message)` | HMAC-SHA1 |
| `crypto.hmac_sha256(key, message)` | HMAC-SHA256 |
| `crypto.hmac_sha512(key, message)` | HMAC-SHA512 |

### AES functions

| Function | Description |
|----------|-------------|
| `crypto.aes_encrypt_cbc(key, iv, plaintext)` | AES-CBC encryption with PKCS#7 padding |
| `crypto.aes_decrypt_cbc(key, iv, ciphertext)` | AES-CBC decryption with PKCS#7 unpadding |
| `crypto.aes_encrypt_gcm(key, nonce, plaintext, aad)` | AES-GCM authenticated encryption |
| `crypto.aes_decrypt_gcm(key, nonce, ciphertext, aad)` | AES-GCM authenticated decryption |

### Encoding functions

| Function | Description |
|----------|-------------|
| `crypto.hex_encode(data)` | Encode bytes to hex string |
| `crypto.hex_decode(string)` | Decode hex string to bytes |
| `crypto.base64_encode(data)` | Encode bytes to standard base64 string |
| `crypto.base64_decode(string)` | Decode standard base64 string to bytes |
| `crypto.base64url_encode(data)` | Encode bytes to URL-safe base64 (no padding) |
| `crypto.base64url_decode(string)` | Decode URL-safe base64 string to bytes |

```python
def on_before_send_to_server(data):
    body = data.get("body", b"")
    digest = crypto.sha256(body)
    hex_digest = crypto.hex_encode(digest)
    headers = data.get("headers", {})
    headers["X-Body-SHA256"] = hex_digest
    data["headers"] = headers
    return {"action": action.CONTINUE, "data": data}
```

## The `proxy` module

The `proxy` module provides control over the proxy runtime.

| Function | Description |
|----------|-------------|
| `proxy.shutdown(reason)` | Triggers proxy shutdown with the given reason string |

```python
def on_receive_from_client(data):
    url = data.get("url", "")
    if "/emergency-stop" in url:
        proxy.shutdown("Emergency stop triggered by plugin")
    return {"action": action.CONTINUE}
```

## Transaction context (`ctx`)

Data hooks within the same transaction (e.g., one HTTP request-response pair) share a transaction context via the `ctx` key in the data dictionary. You can use `ctx` to pass data between hooks.

```python
def on_receive_from_client(data):
    ctx = data.get("ctx", {})
    ctx["start_time"] = "captured"
    data["ctx"] = ctx
    return {"action": action.CONTINUE, "data": data}

def on_before_send_to_client(data):
    ctx = data.get("ctx", {})
    start = ctx.get("start_time", "unknown")
    print("Transaction context: start_time=%s" % start)
    return {"action": action.CONTINUE}
```

Mutations to `ctx` are automatically propagated back to Go even if the plugin does not explicitly return `data`.

## Module-level freeze constraint

All module-level variables are **frozen** (immutable) after the script is loaded. Attempting to mutate a frozen value inside a hook function causes a runtime error.

```python
# BAD: _counter is frozen after load -- mutating it raises an error.
_counter = [0]

def on_before_send_to_server(data):
    _counter[0] = _counter[0] + 1  # Runtime error: frozen list
    return {"action": action.CONTINUE}
```

With `on_error: "skip"` (the default), frozen-mutation errors are silently skipped, making them hard to diagnose. Use `on_error: "abort"` during development to surface these errors immediately.

Module-level **constants** (strings, ints, tuples) are safe because they are inherently immutable:

```python
# OK: immutable constants.
AUTH_TOKEN = "Bearer ..."
BLOCKED_PATHS = ("/admin", "/internal")
```

To maintain mutable state across hook calls, use the `state` module instead of module-level variables.

## on_error setting

| Value | Behavior |
|-------|----------|
| `"skip"` (default) | Log the error, skip the plugin, continue processing |
| `"abort"` | Stop the hook chain and return the error to the caller |

!!! tip
    Use `on_error: "abort"` during development. Switch to `"skip"` in production to ensure a buggy plugin does not block traffic.

## Debugging

### print() for logging

`print()` statements in Starlark output to the proxy log:

```python
def on_receive_from_client(data):
    print("Method: %s, URL: %s" % (data.get("method", ""), data.get("url", "")))
    return {"action": action.CONTINUE}
```

Log output appears as structured log entries with `plugin` and `message` fields.

### Use on_error: "abort" during development

Set `on_error: "abort"` to immediately see errors instead of having them silently swallowed:

```json
{
  "plugins": [
    {
      "path": "my_plugin.star",
      "protocol": "http",
      "hooks": ["on_receive_from_client"],
      "on_error": "abort"
    }
  ]
}
```

### Hot reload

Use the `plugin` MCP tool to reload plugins without restarting the proxy:

```json
// plugin
{"action": "reload", "params": {"name": "my_plugin"}}
```

## Related pages

- [Hook reference](hook-reference.md) -- All hook functions and their call timing
- [Data map reference](data-map-reference.md) -- Protocol-specific data keys
- [Codec plugins](codec-plugins.md) -- Custom encode/decode transformations
- [Examples](examples.md) -- Ready-to-use plugin samples
