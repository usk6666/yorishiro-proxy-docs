# CA certificate

yorishiro-proxy performs HTTPS interception (MITM) by dynamically generating server certificates signed by its own CA. To inspect HTTPS traffic, you must install this CA certificate in your operating system or browser trust store.

## Automatic generation

On first startup, the CA certificate is automatically generated and saved to:

```
~/.yorishiro-proxy/ca/ca.crt   (certificate)
~/.yorishiro-proxy/ca/ca.key   (private key)
```

No manual action is required to generate the CA -- it is created the first time yorishiro-proxy runs.

!!! tip
    Running `yorishiro-proxy install ca --trust` generates the CA certificate and registers it in your OS trust store automatically. You can skip the manual steps below.

## Check the CA certificate path

Use the `query` tool to confirm the CA certificate location:

```json
// query
{"resource": "ca_cert"}
```

The response includes `cert_path` (file path) and `fingerprint` (SHA-256 hash) fields.

## OS-specific installation

=== "macOS"

    ```bash
    sudo security add-trusted-cert -d -r trustRoot \
      -k /Library/Keychains/System.keychain \
      ~/.yorishiro-proxy/ca/ca.crt
    ```

=== "Linux"

    ```bash
    sudo cp ~/.yorishiro-proxy/ca/ca.crt \
      /usr/local/share/ca-certificates/yorishiro-proxy.crt
    sudo update-ca-certificates
    ```

=== "Windows"

    Run the following command in an Administrator command prompt:

    ```cmd
    certutil -addstore "Root" %USERPROFILE%\.yorishiro-proxy\ca\ca.crt
    ```

## Alternative: skip CA installation with playwright-cli

If you are using playwright-cli for browser automation, you can skip CA certificate installation by enabling `ignoreHTTPSErrors` in the playwright configuration. This tells the browser to accept any certificate without verification.

Create `.playwright/cli.config.json` in your project root:

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

With this configuration, playwright-cli will route traffic through the proxy and accept yorishiro-proxy's dynamically generated certificates without needing to install the CA in the OS trust store.

## Ephemeral CA

If you do not want to persist the CA certificate to disk, use the `-ca-ephemeral` flag:

```json
{
  "mcpServers": {
    "yorishiro-proxy": {
      "command": "/path/to/bin/yorishiro-proxy",
      "args": ["server", "-stdio-mcp", "-ca-ephemeral"]
    }
  }
}
```

An ephemeral CA is generated in memory on each launch. This is useful for testing, but the certificate changes on every restart, so you cannot install it in a trust store.

## Next steps

- [First capture](first-capture.md) -- capture your first traffic
- [HTTPS MITM](../protocols/https-mitm.md) -- how HTTPS interception works
- [TLS configuration](../configuration/tls.md) -- advanced TLS settings
