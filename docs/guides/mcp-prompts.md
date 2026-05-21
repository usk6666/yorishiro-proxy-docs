# MCP prompts (vulnerability-verification playbooks)

yorishiro-proxy ships nine vulnerability-verification playbooks as **MCP Prompts** — self-contained, parameterised instructions that the MCP host delivers to the model on demand. Each playbook walks the agent through a focused workflow (verify IDOR, verify SQL injection, fuzz an endpoint, ...) using the typed `query` / `resend_*` / `fuzz_*` / `macro` tools.

Prompts are part of the proxy binary. They are delivered to the host over the standard MCP `prompts/list` and `prompts/get` calls — there is no filesystem install step and nothing to keep in sync with the binary version.

!!! info "Migrated from `install skills`"
    Earlier releases copied a `.claude/skills/yorishiro/` skill tree into your project as part of `yorishiro-proxy install`. The `install skills` subcommand and the `--skills-dir` flag were removed when the playbooks moved to MCP Prompts. There is nothing to delete in existing checkouts -- leftover `.claude/skills/yorishiro/` directories are inert and can be removed at your convenience.

## Why MCP Prompts

- **No install step.** The model sees the playbooks as soon as your MCP host (Claude Code, Claude Desktop, ...) connects to a running yorishiro-proxy server.
- **Versioned with the binary.** Upgrading yorishiro-proxy automatically upgrades the playbooks; there is no drift between the proxy version and a separately-installed skill tree.
- **Triggered explicitly.** A prompt is invoked by name with structured arguments -- you (or the model on your behalf) pick the playbook and supply its inputs, instead of the host guessing from a free-text description.
- **One delivery model.** Anything an agent needs to do with yorishiro-proxy is either an MCP tool or an MCP prompt -- there is no third surface.

## Available playbooks

The following nine prompts are registered by the MCP server. Required arguments must be non-empty; the handler returns an error otherwise.

| Prompt | Purpose | Required arguments |
|--------|---------|--------------------|
| `verify-idor` | Sweep candidate ids in body or path to verify Insecure Direct Object Reference / privilege escalation | `target_flow_id`, `id_field`, `candidate_ids` |
| `verify-sqli` | Verify SQL injection with method-aware payload guardrails (time-based default; error-based; UNION) | `target_flow_id`, `injection_point` |
| `verify-xss` | Verify reflected XSS using `YP_`-prefixed marker payloads -- no scripts execute | `target_flow_id`, `injection_point` |
| `verify-csrf` | Verify whether a state-changing flow validates its CSRF token (empty / forged / missing) | `target_flow_id`, `token_header_name` |
| `audit-auth` | Audit authentication bypass and role-downgrade / privilege escalation on a recorded flow | `target_flow_id`, `auth_header_name` |
| `fuzz-endpoint` | General-purpose fuzz workflow that picks `fuzz_http` / `fuzz_ws` / `fuzz_grpc` / `fuzz_raw` from the captured flow's protocol | `target_flow_id`, `position_path`, `payload_set` |
| `replay-with-mods` | Single-shot replay of a recorded request with selective header / body modifications, using the matching typed `resend_*` tool | `target_flow_id`, `mods_description` |
| `capture-traffic` | Capture HTTP / WebSocket / gRPC traffic via playwright-cli through yorishiro-proxy with a scoped capture filter | `target_host` |
| `stateful-fuzz-loop` | Drive a setup -> resend -> teardown loop around stateful endpoints that cannot be safely fuzzed in bulk (repeated DELETE, single-use tokens, rotating CSRF) | `setup_macro_name`, `teardown_macro_name`, `target_flow_id` |

Optional arguments (e.g. `verify-sqli.technique`, `audit-auth.low_priv_token`, `capture-traffic.listen_addr`) substitute their default behaviour when omitted.

## How a host invokes a prompt

In an MCP host that surfaces prompts (e.g. Claude Code), the playbooks appear in the host's prompt picker. Pick one, fill in the required fields, and the host expands the prompt body into a single user message that drives the agent through the workflow.

Programmatically, the call looks like this:

```json
// prompts/get
{
  "name": "verify-idor",
  "arguments": {
    "target_flow_id": "f-3a8d4c1e",
    "id_field": "$.user_id",
    "candidate_ids": "1,2,3,42,999"
  }
}
```

The server returns a single `PromptMessage` whose `content` is a `TextContent` block containing the full playbook with `{{argument_name}}` placeholders already substituted. The model then executes the playbook step by step, issuing the typed MCP tool calls described inside it.

!!! note "Placeholder syntax"
    Prompt bodies use `{{arg_name}}` double-brace placeholders that match each `PromptArgument` name. The handler substitutes them with `strings.ReplaceAll` at `prompts/get` time. Required arguments missing or empty cause the handler to error out before substitution; optional arguments missing or empty are substituted with the empty string.

## Talking to the agent

You do not have to invoke a prompt by name yourself -- describing a task in natural language usually works. Examples:

- "Verify IDOR on flow `f-3a8d4c1e` by sweeping `$.user_id` over 1..50." -> host picks `verify-idor`.
- "Check that flow for SQL injection in the body." -> host picks `verify-sqli`.
- "Replay this request as the low-privilege user." -> host picks `replay-with-mods`.
- "Capture traffic from `*.target.example.com` via playwright-cli." -> host picks `capture-traffic`.

In a host that does not surface MCP prompts in its UI, invoke them by name directly through the host's prompt interface, or ask the agent to "use the `verify-idor` prompt with these arguments".

## Playbook structure (what's inside a prompt)

Every playbook is short (50--180 lines) and follows the same shape:

1. **Inputs** -- list of required and optional arguments. The agent is instructed to ask the user for any required input that is empty.
2. **Safety guardrails** (where applicable) -- method-aware payload tables, marker-payload conventions, "never send DROP/DELETE/UPDATE" reminders.
3. **Plan** -- numbered steps that issue typed MCP tool calls. Each tool call appears in the playbook body as a `// tool_name` comment followed by a JSON object.
4. **Evaluation** -- how to interpret tool results.
5. **Reporting** -- what to summarise back to the user once the playbook is complete.

The tool calls inside the playbooks use the same typed surface documented in [MCP tools](../tools/overview.md): `query`, `resend_http` / `resend_ws` / `resend_grpc` / `resend_raw`, `fuzz_http` / `fuzz_ws` / `fuzz_grpc` / `fuzz_raw`, and `macro`.

## Discovering prompts from a custom MCP client

If you are building a custom MCP client, the standard `prompts/list` call returns the full catalogue (name, title, description, and argument schema). Use this to populate your UI's prompt picker or to enumerate which arguments need to be collected before issuing `prompts/get`.

```json
// prompts/list
{}
```

The MCP server advertises the `prompts: {listChanged: true}` capability on connect.

## Related pages

- [MCP tools overview](../tools/overview.md) -- typed tools that the playbooks invoke
- [Quick setup](../getting-started/quick-setup.md) -- initial installation and CA setup
- [MCP configuration](../getting-started/mcp-configuration.md) -- transport modes and host configuration
- [Vulnerability assessment](vulnerability-assessment.md) -- end-to-end vulnerability assessment workflow
