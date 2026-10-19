# docs

Self-service documentation for the MCP server. Every help document that ships with yorishiro-proxy is reachable through this one tool, so your agent never has to guess parameter syntax.

## Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `topic` | string | No | | Document to return. Omit it to receive the topic index |
| `section` | string | No | | Return only one heading-anchored section instead of the whole document. Requires `topic` |

### topic

Every MCP tool name is a topic -- `proxy_start`, `proxy_stop`, `query`, `manage`, `macro`, `intercept`, `configure`, `security`, `resend_http`, `resend_ws`, `resend_grpc`, `resend_raw`, `fuzz_http`, `fuzz_ws`, `fuzz_grpc`, `fuzz_raw`, `plugin_introspect`, `grpc_schema` -- plus the concept topics `getting-started`, `examples`, `template-syntax`, and `docs` itself.

An unknown topic returns an error listing every valid topic.

### section

- Matching is case-insensitive and whitespace-trimmed.
- Both the literal heading text and its GitHub-style slug work: `"Variable substitution syntax"` and `"variable-substitution-syntax"` select the same section.
- Addressable levels are `##` through `####`. The `#` document title is not addressable -- `topic` alone already returns it.
- A match includes its sub-sections: selecting a `##` heading returns its `###` and `####` children too.
- When two sections share a heading text, the first is returned with a note saying how many share it. Fetch the whole document to reach the later ones.
- An unknown section returns an error listing that topic's `##`--`####` outline, spelled exactly as it must be passed back.

## Response

The tool returns Markdown text.

- **Index** (no `topic`) -- every topic with its one-line summary, an estimated token cost (`est_tokens`), and its `##` section names. Use it to decide between a full document and a section fetch. `est_tokens` is a `chars/4` heuristic, not a tokenizer count.
- **Document or section** -- returned verbatim from the binary's embedded copy. There is no filesystem or network read and no output size cap.

The Output SafetyFilter is not applied to `docs` results: these are static in-repo documents with no captured traffic, and masking would corrupt the sample values the `security` topic uses to explain the PII presets.

## Examples

### List every topic

```json
// docs
{}
```

### Read a whole tool reference

```json
// docs
{
  "topic": "fuzz_http"
}
```

### Read one section of a large document

```json
// docs
{
  "topic": "macro",
  "section": "Variable substitution syntax"
}
```

### The same section by slug

```json
// docs
{
  "topic": "macro",
  "section": "variable-substitution-syntax"
}
```

## From the CLI

The same documents are embedded in the binary, so you can read them without a running server:

```bash
yorishiro-proxy docs                                        # the topic index
yorishiro-proxy docs macro                                  # a whole document
yorishiro-proxy docs macro "Variable substitution syntax"   # one section
yorishiro-proxy docs macro --section variable-substitution-syntax
```

The topic and section are positional words; `--section` is an alternative to the second positional, and giving both is an error. The subcommand opens no socket, so it works before `proxy_start`, before MCP registration, and when nothing is listening.

Against a running server, the MCP tool is also reachable through the generic client path with `key=value` arguments:

```bash
yorishiro-proxy client docs topic=macro section="Variable substitution syntax"
```

## Notes

- The same documents remain registered as MCP resources under `yorishiro://help/<topic>` for hosts that browse resources. The `docs` tool exists because most hosts never fetch a resource on the model's behalf.
- Tool descriptions point to this tool (for example, "Full reference: call `docs(topic="proxy_start")`") instead of to the resource URIs.

## Related pages

- [MCP tools overview](overview.md) -- Catalog of every tool
- [MCP prompts](../guides/mcp-prompts.md) -- Vulnerability-verification playbooks delivered over MCP
- [CLI flags](../configuration/cli-flags.md) -- Command-line reference
