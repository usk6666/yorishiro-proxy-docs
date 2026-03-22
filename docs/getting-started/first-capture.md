# First capture

This guide walks you through capturing your first HTTP traffic with yorishiro-proxy. You can use either a manual browser setup or playwright-cli for automated capture.

## Manual browser capture

### Step 1: Start the proxy

Ask Claude Code to start the proxy, or use the `proxy_start` tool directly:

```json
// proxy_start
{
  "listen_addr": "127.0.0.1:8080"
}
```

The proxy is now listening on `127.0.0.1:8080`.

### Step 2: Configure your browser

Set your browser or system HTTP proxy to `http://127.0.0.1:8080`. You can also set environment variables for command-line tools:

```bash
export HTTP_PROXY=http://127.0.0.1:8080
export HTTPS_PROXY=http://127.0.0.1:8080
```

### Step 3: Generate traffic

Browse to any website or make HTTP requests through the proxy. For a quick test:

```bash
curl -x http://127.0.0.1:8080 http://httpbin.org/get
```

For HTTPS (requires the CA certificate installed, or use `-k` to skip verification):

```bash
curl -x http://127.0.0.1:8080 -k https://httpbin.org/get
```

## Playwright-cli capture

If you have playwright-cli installed, you can capture browser traffic without manual browser configuration.

### Step 1: Configure playwright-cli

Run `install playwright` to auto-detect your system browser and generate the configuration:

```bash
yorishiro-proxy install playwright
```

This command:

- **Auto-detects** an installed browser (priority: chromium > firefox > chrome)
- **Generates** `.playwright/cli.config.json` with the detected browser settings
- **Auto-installs** the browser via `npx playwright install` if not found
- **Applies `--no-sandbox`** automatically in container environments (Docker, Codespaces, Gitpod)

The generated configuration looks like this (values depend on your environment):

```json
{
  "browser": {
    "browserName": "chromium",
    "launchOptions": {
      "channel": "chromium",
      "proxy": {
        "server": "http://127.0.0.1:8080"
      }
    },
    "contextOptions": {
      "ignoreHTTPSErrors": true
    }
  }
}
```

The `ignoreHTTPSErrors: true` option bypasses SSL certificate errors, so you do not need to install the CA certificate when using playwright-cli.

You can also create or edit this file manually if you need a specific browser or custom settings.

### Step 2: Start the proxy

```json
// proxy_start
{
  "listen_addr": "127.0.0.1:8080"
}
```

### Step 3: Open a page

Use playwright-cli to open a browser that routes through the proxy:

```bash
playwright-cli open https://httpbin.org/get
```

All browser traffic flows through yorishiro-proxy and is recorded as flows.

## View captured flows

List all captured flows:

```json
// query
{"resource": "flows"}
```

This returns all captured flows with their IDs, methods, URLs, status codes, and timestamps.

## Filter flows

To find specific requests, use the `filter` parameter:

```json
// query
{
  "resource": "flows",
  "filter": {"url_pattern": "httpbin.org", "method": "GET"},
  "limit": 10
}
```

## Inspect request details

Drill into an individual flow to see the full request and response:

```json
// query
{"resource": "flow", "id": "<flow-id>"}
```

This returns the complete request and response, including headers, body, status code, and timing information.

For streaming protocols (WebSocket, gRPC, HTTP/2), flows contain multiple messages. List them with:

```json
// query
{
  "resource": "messages",
  "id": "<flow-id>",
  "limit": 50
}
```

## Resend a request

Replay a captured request as-is:

```json
// resend
{
  "action": "resend",
  "params": {
    "flow_id": "<flow-id>"
  }
}
```

Or resend with modifications -- for example, changing a header:

```json
// resend
{
  "action": "resend",
  "params": {
    "flow_id": "<flow-id>",
    "override_headers": {"Authorization": "Bearer <different-token>"}
  }
}
```

Use `"dry_run": true` to preview changes without sending:

```json
// resend
{
  "action": "resend",
  "params": {
    "flow_id": "<flow-id>",
    "override_method": "PUT",
    "dry_run": true
  }
}
```

## Next steps

With the basic setup complete, explore these topics:

- [Resender](../features/resender.md) -- advanced request replay and mutation
- [Fuzzer](../features/fuzzer.md) -- automated payload injection
- [Target scope](../features/target-scope.md) -- control which traffic is captured
- [Intercept](../features/intercept.md) -- hold and modify requests in real time
- [Web UI](../webui/overview.md) -- visual interface for browsing flows and running tests
