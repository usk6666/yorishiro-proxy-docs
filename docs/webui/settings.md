# Settings

The Settings page provides a tabbed interface for configuring all aspects of the proxy. It uses the MCP `configure`, `proxy_start`, and `proxy_stop` tools to apply changes.

![Settings](../images/webui_settings.png)

## Tabs

The Settings page is organized into 12 tabs:

| Tab | Description |
|-----|-------------|
| **Proxy** | Listener management |
| **Capture Scope** | Control which traffic is captured |
| **TLS Passthrough** | Bypass MITM for specific hosts |
| **TCP Forwards** | Port forwarding rules |
| **gRPC Schemas** | Register `.proto` schemas for schema-aware gRPC decode |
| **Intercept Rules** | Request interception rules and intercept queue timeout |
| **Auto-Transform** | Automatic request/response modifications |
| **SOCKS5 Auth** | SOCKS5 authentication credentials |
| **TLS Fingerprint** | TLS fingerprinting settings |
| **Client Certificate** | mTLS client certificate configuration |
| **Connection** | Connection limits, timeouts, and per-listener upstream proxy |
| **CA Certificate** | CA certificate management |

## Proxy

Manage proxy listeners (start/stop). The panel shows currently running listeners with their name, listen address, active connection count, and uptime.

To start a new listener:

1. Click **Start New Listener** to reveal the form
2. Configure the listener:
    - **Name** -- Optional listener name
    - **Listen address** -- Address and port to bind (default: `127.0.0.1:8080`)
    - **Upstream proxy** -- Optional upstream proxy, set with the [upstream proxy editor](#upstream-proxy-editor)
3. Click **Start**

To stop a listener, click the **Stop** button next to it (with confirmation).

!!! note "Protocol selection removed (USK-870)"
    The previous "Protocols" checkbox group on this panel was removed alongside the deletion of `proxy_start.protocols`. The half-implemented protocol allow-list had no use case that wasn't already covered by `target_scope` (host), `intercept_rules` (request pattern), `tls_passthrough` (MITM target), and `capture_scope` (recording target). Use those knobs to scope what the proxy handles.

### Upstream proxy editor

Both the **Start New Listener** form and the [Connection](#connection) tab use the same upstream proxy editor. A radio group switches between two modes:

- **Simple URL** -- A single upstream proxy URL (e.g., `http://host:port` or `socks5://host:port`)
- **Rotation template** -- A URL template that is expanded per resolution, plus a rotation policy:
    - **Rotation URL Template** -- e.g., `http://session-§__nonce§:pass@proxy:8080`. `§__nonce§` expands to a fresh UUID on each resolution; other `§var§` macros are also accepted
    - **Rotation Policy** -- When to resolve a new URL: **Per request**, **Per connection**, **Per target host**, or **Sticky**

The editor shows warnings as toasts when you save a URL or template that doesn't start with `http://`, `https://`, or `socks5://`, or a rotation template that contains no `§var§` macro (every resolution would then return the same URL). Credentials in the URL are redacted (`xxxxx`) in the confirmation toast.

See [Upstream proxy](../configuration/upstream-proxy.md) for rotation semantics.

## Capture scope

Configure rules that determine which traffic is captured and stored. You can add include/exclude patterns to scope the capture to specific hosts, paths, or other criteria.

## TLS passthrough

Manage hosts that should bypass TLS interception (MITM). Connections to these hosts are forwarded directly without decrypting the traffic. Use this for:

- Certificate-pinned applications
- Hosts you don't need to inspect
- Services that break under MITM

You can add and remove passthrough rules from this tab.

## TCP forwards

Configure TCP port forwarding rules that map local ports to upstream targets. Each forward rule specifies:

- **Local port** -- The port to listen on (1--65535)
- **Upstream host** -- The target hostname (e.g., `db.example.com`)
- **Upstream port** -- The target port (1--65535)
- **Protocol** -- Protocol hint for L7 parsing:

    | Value | Description |
    |-------|-------------|
    | `auto` | Peek-based protocol detection (default) |
    | `raw` | No L7 parsing, forward raw bytes |
    | `http` | Force HTTP/1.x parsing |
    | `http2` | Force HTTP/2 parsing |
    | `grpc` | Force gRPC parsing (over HTTP/2) |
    | `websocket` | Force WebSocket frame parsing |
    | `sse` | Force Server-Sent Events parsing |

- **TLS termination** -- **Enable TLS MITM** checkbox. When checked, the proxy terminates the client-side TLS using the target hostname for certificate generation, then applies L7 parsing to the decrypted traffic.
- **Upstream TLS** -- **Dial upstream over TLS** checkbox. When checked, the proxy connects to the upstream target over TLS, independently of the client-side TLS termination setting.

TCP forwards are listener start-time configuration. Adding a forward calls `proxy_start` with the updated mapping set, so a confirmation dialog (**Apply TCP Forward**) appears first: applying it resets the listener's in-session settings (intercept rules, transform rules, capture scope, TLS fingerprint, connection limits) and may interrupt in-flight connections on that listener. To remove a forward, restart the proxy with the desired configuration.

Each configured forward is displayed in the mapping list with:

- A port badge (e.g., `:3306`)
- The upstream target (`host:port`)
- A protocol badge -- green for an explicit protocol, gray for `auto`
- A **TLS** badge (yellow) when TLS termination is enabled
- An **Upstream TLS** badge (yellow) when the upstream is dialed over TLS

This is useful for forwarding non-HTTP protocols (databases, gRPC services, custom TCP services) through the proxy for inspection.

## gRPC schemas

Register `.proto` schemas that drive schema-aware decode in `query` (`body_decoded_encoding: "proto-json"`) and accept `body_encoding: "proto-json"` on `resend_grpc`. The card lists the registered schemas and provides:

- **Upload .desc** -- Upload a `FileDescriptorSet` (`.desc`) file, with an optional source label
- **Discover** -- Probe a target's gRPC server reflection service (target address, scheme, optional metadata rows and timeout)
- **Clear all** -- Remove every registered schema

Schemaless fallback applies when no schema matches a flow. See the [grpc_schema tool](../tools/grpc-schema.md).

## Intercept rules

Manage rules that determine which requests are intercepted and held for review. You can:

- Add new intercept rules
- Enable or disable existing rules
- Remove rules

Intercepted requests appear in the [Intercept](intercept.md) page queue.

### Intercept queue

The **Intercept Queue** card on the same tab configures what happens to items that wait in the queue too long:

- **Timeout (ms)** -- How long an item is held before the timeout behavior applies
- **Timeout behavior** -- **Auto Release** (forward unchanged) or **Auto Drop**

The form is prefilled from the running configuration (`query` with `resource: "config"`), so it shows the current values rather than defaults.

## Auto-transform

Configure automatic request and response transformation rules. Auto-transform rules modify traffic in-flight without manual intervention. Each rule specifies match conditions and transformations to apply.

## SOCKS5 auth

Configure SOCKS5 proxy authentication credentials. When the proxy operates as a SOCKS5 server, you can set username/password requirements for connecting clients.

## TLS fingerprint

View and configure TLS fingerprinting settings. TLS fingerprinting uses uTLS to make the proxy's upstream TLS ClientHello mimic a browser, helping avoid JA3/JA4-based detection by WAFs or bot protection systems.

The available profiles are **None (Default Go TLS)**, **Chrome**, **Firefox**, **Safari**, **Edge**, and **Random**. The proxy defaults to `firefox`; choose **None** to use the standard Go TLS stack. See [TLS configuration](../configuration/tls.md) for details.

## Client certificate

Configure mTLS client certificates for connections to upstream servers that require mutual TLS authentication. You can set:

- Client certificate file path
- Client private key file path

## Connection

The Connection tab has two cards.

**Connection & Timeouts** -- Edit and **Save** the connection limits:

- **Max connections** -- Maximum concurrent connection limit (must be positive)
- **Peek timeout (ms)** -- Protocol detection timeout (at least 100 ms)
- **Request timeout (ms)** -- HTTP request timeout (at least 100 ms)

**Upstream Proxy** -- Route outgoing traffic through an upstream HTTP or SOCKS5 proxy, per listener:

- **Listener (scope)** -- Select `default` or any running listener. The editor below is prefilled from that listener's current setting: its rotation template if one is set, otherwise its literal upstream proxy, otherwise the global default
- The [upstream proxy editor](#upstream-proxy-editor) -- Simple URL or rotation template. Leave the URL empty to disable the upstream proxy

Click **Save** to apply the value to the selected listener through `configure`.

!!! note
    The intercept queue timeout settings moved to the [Intercept rules](#intercept-queue) tab.

## CA certificate

Manage the proxy's Certificate Authority used for TLS interception:

- **View certificate details** -- Subject, issuer, validity dates, and serial number
- **Download CA certificate** -- Download the root CA certificate in PEM format for installation in your operating system or browser trust store
- **Installation instructions** -- Platform-specific guides for macOS, Windows, Linux, Firefox, and Android
- **Regenerate CA** -- Generate a new CA certificate (requires reinstallation in trust stores)

!!! warning
    Regenerating the CA certificate invalidates all previously installed certificates. You will need to reinstall the new CA certificate on all devices.

## Plugins

Plugin management is no longer in Settings. The read-only Plugins view is at the `/plugins` route. Plugins load once at proxy boot from `config.plugins`; to change the loaded set, edit the config and restart the proxy.

## Related pages

- [CLI flags](../configuration/cli-flags.md) -- Command-line configuration reference
- [Config file](../configuration/config-file.md) -- Configuration file reference
- [TLS configuration](../configuration/tls.md) -- TLS setup guide
- [CA certificate](../getting-started/ca-certificate.md) -- CA certificate installation guide
- [proxy_start tool](../tools/proxy-start.md) -- MCP tool for starting listeners
- [configure tool](../tools/configure.md) -- MCP tool for configuration changes
- [plugin_introspect tool](../tools/plugin-introspect.md) -- read-only MCP tool for plugin introspection
