# Installation

yorishiro-proxy is distributed as a single binary. You can download a pre-built release or build from source.

## Prerequisites

- **Claude Code** -- yorishiro-proxy operates as an MCP server for Claude Code
- **Go 1.25+** -- required only if building from source
- **playwright-cli** (optional) -- for automated browser-based traffic capture

## Download from GitHub Releases

Download the latest pre-built binary from the [GitHub Releases page](https://github.com/usk6666/yorishiro-proxy/releases/latest). Assets are named `yorishiro-proxy-{version}-{os}-{arch}` (e.g., `yorishiro-proxy-v1.0.0-linux-amd64`; Windows binaries have an `.exe` suffix).

=== "macOS / Linux"

    Visit the [Releases page](https://github.com/usk6666/yorishiro-proxy/releases/latest) and download the binary for your platform, or use curl:

    ```bash
    # Replace VERSION, OS, and ARCH with your values.
    # OS: linux, darwin  ARCH: amd64, arm64
    curl -Lo yorishiro-proxy \
      "https://github.com/usk6666/yorishiro-proxy/releases/download/VERSION/yorishiro-proxy-VERSION-OS-ARCH"
    chmod +x yorishiro-proxy
    sudo mv yorishiro-proxy /usr/local/bin/
    ```

=== "Windows"

    Download the `.exe` binary from the [Releases page](https://github.com/usk6666/yorishiro-proxy/releases/latest) and place it in a directory on your `PATH`.

## Build from source

Clone the repository and build with `make`:

```bash
git clone https://github.com/usk6666/yorishiro-proxy.git
cd yorishiro-proxy
make build
```

This produces the binary at `bin/yorishiro-proxy`. Move it to a directory on your `PATH`:

```bash
sudo mv bin/yorishiro-proxy /usr/local/bin/
```

## Verify the installation

Run the help command to confirm the binary is working:

```bash
yorishiro-proxy -h
```

You should see the version, usage information, and available flags.

## Next steps

- [Quick setup](quick-setup.md) -- one-command setup with `yorishiro-proxy install`
- [MCP configuration](mcp-configuration.md) -- manual MCP client configuration
