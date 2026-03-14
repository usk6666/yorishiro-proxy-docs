# Plugin examples

This page provides ready-to-use Starlark plugin examples for common use cases. Each example includes the plugin code and the required configuration.

## Hook plugin examples

### add_auth_header.star

Injects an Authorization header into every outgoing HTTP request. Uses the `config` dict to read the token from plugin configuration.

```python
def on_before_send_to_server(data):
    token = config.get("token", "")
    if token:
        headers = data.get("headers", {})
        headers["Authorization"] = token
        data["headers"] = headers
    return {"action": action.CONTINUE, "data": data}
```

Configuration:

```json
{
  "plugins": [
    {
      "path": "plugins/add_auth_header.star",
      "protocol": "http",
      "hooks": ["on_before_send_to_server"],
      "vars": {
        "token": "Bearer eyJhbGciOiJIUzI1NiJ9.example"
      }
    }
  ]
}
```

### grpc_logger.star

Logs all gRPC method calls for monitoring:

```python
def on_receive_from_client(data):
    method = data.get("url", "unknown")
    print("gRPC call: method=%s" % method)
    return {"action": action.CONTINUE}
```

Configuration:

```json
{
  "plugins": [
    {
      "path": "plugins/grpc_logger.star",
      "protocol": "grpc",
      "hooks": ["on_receive_from_client"]
    }
  ]
}
```

### ws_filter.star

Drops WebSocket messages containing a blocked pattern:

```python
BLOCKED_PATTERN = "FORBIDDEN_COMMAND"

def on_receive_from_client(data):
    payload = data.get("payload", "")
    is_text = data.get("is_text", False)
    if is_text and BLOCKED_PATTERN in payload:
        return {"action": action.DROP}
    return {"action": action.CONTINUE}
```

Configuration:

```json
{
  "plugins": [
    {
      "path": "plugins/ws_filter.star",
      "protocol": "websocket",
      "hooks": ["on_receive_from_client"]
    }
  ]
}
```

### http_mock.star

Returns a mock response for requests to a specific path:

```python
MOCK_PATH = "/api/v1/health"

def on_receive_from_client(data):
    url = data.get("url", "")
    if MOCK_PATH in url:
        return {
            "action": action.RESPOND,
            "response": {
                "status_code": 200,
                "headers": {"Content-Type": "application/json"},
                "body": '{"status":"ok","mocked":true}',
            },
        }
    return {"action": action.CONTINUE}
```

Configuration:

```json
{
  "plugins": [
    {
      "path": "plugins/http_mock.star",
      "protocol": "http",
      "hooks": ["on_receive_from_client"]
    }
  ]
}
```

### socks5_logger.star

Logs SOCKS5 tunnel establishment details:

```python
def on_socks5_connect(data):
    target = data.get("target", "unknown")
    auth_method = data.get("auth_method", "unknown")
    auth_user = data.get("auth_user", "")
    client_addr = data.get("client_addr", "unknown")

    if auth_user:
        print("SOCKS5 CONNECT: target=%s auth=%s user=%s client=%s" % (
            target, auth_method, auth_user, client_addr))
    else:
        print("SOCKS5 CONNECT: target=%s auth=%s client=%s" % (
            target, auth_method, client_addr))

    return {"action": action.CONTINUE}
```

Configuration:

```json
{
  "plugins": [
    {
      "path": "plugins/socks5_logger.star",
      "protocol": "socks5",
      "hooks": ["on_socks5_connect"]
    }
  ]
}
```

### request_counter.star

Counts requests per URL path using the `state` module:

```python
def on_receive_from_client(data):
    path = data.get("path", "/")
    count = state.get(path)
    if count == None:
        count = 0
    count = count + 1
    state.set(path, count)
    print("Path %s: request #%d" % (path, count))
    return {"action": action.CONTINUE}
```

Configuration:

```json
{
  "plugins": [
    {
      "path": "plugins/request_counter.star",
      "protocol": "http",
      "hooks": ["on_receive_from_client"]
    }
  ]
}
```

### body_hash_logger.star

Computes and logs the SHA-256 hash of request bodies using the `crypto` module:

```python
def on_receive_from_client(data):
    body = data.get("body", b"")
    if body:
        digest = crypto.sha256(body)
        hex_digest = crypto.hex_encode(digest)
        method = data.get("method", "")
        url = data.get("url", "")
        print("%s %s body_sha256=%s" % (method, url, hex_digest))
    return {"action": action.CONTINUE}
```

Configuration:

```json
{
  "plugins": [
    {
      "path": "plugins/body_hash_logger.star",
      "protocol": "http",
      "hooks": ["on_receive_from_client"]
    }
  ]
}
```

## Codec plugin examples

### sql_escape

Escapes single quotes for SQL:

```python
# codecs/sql_escape.star
name = "sql_escape"

def encode(s):
    return s.replace("'", "''")

def decode(s):
    return s.replace("''", "'")
```

### rot13

ROT13 letter substitution (encode-only):

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

Codec plugin configuration:

```json
{
  "codec_plugins": [
    {"path": "codecs/sql_escape.star"},
    {"path": "codecs/rot13.star"}
  ]
}
```

## Related pages

- [Overview](overview.md) -- Plugin system overview
- [Writing plugins](writing-plugins.md) -- How to write plugins
- [Hook reference](hook-reference.md) -- All hook functions
- [Data map reference](data-map-reference.md) -- Protocol data keys
- [Codec plugins](codec-plugins.md) -- Custom codec plugin details
