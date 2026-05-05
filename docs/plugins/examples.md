# Examples

Ready-to-use Starlark plugins for common scenarios. Each example is a complete script body — drop it into a `.star` file, point a `plugins[]` config entry at it, and restart the proxy.

For the syntax of `register_hook`, the `msg` mutation API, and the sandbox modules, see [Writing plugins](writing-plugins.md).

## Add an Authorization header on outbound HTTP

Stamp `Authorization` onto every outbound HTTP request, including resends and Macro fan-outs. Registering at `phase="post_pipeline"` ensures the hook fires once per variant, after Transform — so the header is present whether the request originated from the wire, a `resend_http` MCP call, or a Macro.

```python
def stamp_auth(msg, ctx):
    msg["headers"].delete_first("Authorization")
    msg["headers"].append("Authorization", config["token"])
    return None  # CONTINUE

register_hook("http", "on_request", stamp_auth, phase="post_pipeline")
```

Configuration:

```json
{
  "plugins": [
    {
      "path": "plugins/add_auth_header.star",
      "vars": {
        "token": "Bearer eyJhbGciOiJIUzI1NiJ9.example"
      }
    }
  ]
}
```

`delete_first` is called before `append` so re-runs of this hook on the same envelope (which can happen when both pre and post phases are active for different plugins) do not stack duplicate headers.

## Filter inbound WebSocket messages by content

WebSocket message events accept `CONTINUE` only — there is no `DROP` for `ws.on_message` because mid-stream termination uses native Close frames, not an action enum. To suppress a forbidden command from reaching the upstream server, replace the payload with an empty binary frame.

```python
BLOCKED = "FORBIDDEN_COMMAND"

def filter_ws(msg, ctx):
    # Opcode 0x1 = text, 0x2 = binary.
    if msg["opcode"] != 0x1:
        return None
    if BLOCKED in str(msg["payload"]):
        # Replace the frame with an empty payload so the wire framing stays
        # well-formed but the forbidden command never reaches the server.
        msg["payload"] = b""
        print("ws filter: stripped %s" % BLOCKED)
    return None

register_hook("ws", "on_message", filter_ws, phase="pre_pipeline")
```

Configuration:

```json
{
  "plugins": [
    {"path": "plugins/ws_filter.star"}
  ]
}
```

To genuinely terminate the channel from a plugin, send a Close frame from the upstream side via your own application logic — `ws.on_message` is the wrong layer for connection-level decisions.

## Synthesize a 401 for unauthorized requests

Use `action.RESPOND` to short-circuit unauthorized requests with a synthesized 401 response. The upstream is never contacted; the client sees the synthetic envelope as if the server had returned it.

```python
def require_auth(msg, ctx):
    # Exempt the health endpoint.
    if msg["path"] == "/health":
        return None

    auth = msg["headers"].get_first("Authorization")
    if auth == None or not auth.startswith("Bearer "):
        return action.RESPOND(
            401,
            [
                ("WWW-Authenticate", "Bearer"),
                ("Content-Type", "application/json"),
            ],
            b'{"error":"unauthorized"}',
        )
    return None

register_hook("http", "on_request", require_auth, phase="pre_pipeline")
```

Configuration:

```json
{
  "plugins": [
    {"path": "plugins/require_auth.star"}
  ]
}
```

`pre_pipeline` is the right phase for access-control checks — they should fire on the pristine wire-fresh request, before Transform rules or Macro variants observe it.

## Tag flows for grouping using `ctx.transaction_state`

`ctx.transaction_state` is a per-transaction KV: for HTTP, two hooks on the same envelope (a pre-phase and a post-phase invocation) share it; for streaming protocols, it spans the channel's lifetime. Use it to stash values on a request that a response hook then reads.

```python
def tag_request_start(msg, ctx):
    # Stamp a per-flow correlation ID into transaction_state. Both this
    # hook and tag_response below run on the same HTTP envelope (request
    # and response respectively, sharing FlowID-keyed transaction_state).
    correlation = msg["headers"].get_first("X-Correlation-ID")
    if correlation == None:
        return None
    ctx.transaction_state.set("correlation", correlation)

def tag_response(msg, ctx):
    correlation = ctx.transaction_state.get("correlation")
    if correlation == None:
        return None
    # Mirror the correlation header onto the response so the client sees it.
    msg["headers"].delete_first("X-Correlation-ID")
    msg["headers"].append("X-Correlation-ID", correlation)
    print("flow %s tagged" % correlation)

register_hook("http", "on_request", tag_request_start, phase="pre_pipeline")
register_hook("http", "on_response", tag_response, phase="post_pipeline")
```

Configuration:

```json
{
  "plugins": [
    {"path": "plugins/tag_flow.star"}
  ]
}
```

The Layer releases the transaction scope at terminal-state transitions; after release, captured references read empty and writes silently drop. You don't need to clean up entries yourself.

## See also

- [Writing plugins](writing-plugins.md) — `register_hook`, mutation API, sandbox modules.
- [Hook reference](hook-reference.md) — The 17-entry surface table.
- [Data map reference](data-map-reference.md) — Per-protocol `msg` dict shape.
