# Claude Code skills integration

yorishiro-proxy ships with a set of Claude Code skills that teach the AI agent how to use the proxy effectively for vulnerability assessment workflows. This guide explains how to install the skills, what they contain, and how to interact with the agent.

## Installing skills

### Quick install

The simplest way to install skills is with the `install` command:

```bash
yorishiro-proxy install skills
```

This copies the skill files to `.claude/skills/yorishiro/` in your current project directory.

### Full setup

To install everything (MCP configuration, CA certificate, and skills) in one step:

```bash
yorishiro-proxy install
```

To also register the CA certificate in your OS trust store:

```bash
yorishiro-proxy install --trust
```

### User-scope installation

By default, skills are installed at the project level. To install them in your user-level settings (`~/.claude/settings.json`):

```bash
yorishiro-proxy install skills --user-scope
```

### Updating skills

After upgrading yorishiro-proxy, update the skills to match the new version:

```bash
yorishiro-proxy upgrade
yorishiro-proxy install skills
```

If a previous version of the skills exists, it is automatically backed up to `.claude/skills/yorishiro.bak.<timestamp>/` before the new version is installed.

## What gets installed

The `install skills` command creates the following files in `.claude/skills/yorishiro/`:

```
.claude/skills/yorishiro/
  SKILL.md                              # Main skill definition
  references/
    payload-patterns.md                 # Non-destructive attack payloads
    playwright-capture.md               # Browser traffic capture workflow
    self-contained-iteration.md         # Stateful fuzzing patterns
    verify-vulnerability.md             # 5-phase verification workflow
```

### SKILL.md -- Main skill

The main skill file (`SKILL.md`) is the entry point. It provides Claude Code with:

- **MCP tool overview** -- a complete reference of all 11 MCP tools with parameters and usage examples
- **MCP resource URIs** -- how to access help and schema resources for each tool
- **Decision tree** -- a workflow selection guide that helps the agent choose the right approach
- **SafetyFilter guidance** -- preset selection and configuration for safe testing

The skill is triggered when you ask the agent to perform tasks like:

- "Test this endpoint for IDOR vulnerabilities"
- "Check for SQL injection on the login API"
- "Fuzz the search parameter with XSS payloads"
- "Verify that CSRF protection is working"

### references/payload-patterns.md -- Attack payloads

A curated collection of non-destructive attack payloads organized by vulnerability type:

| Vulnerability type | Strategy |
|---|---|
| IDOR | Integer range enumeration, path parameter swapping |
| SQL injection (time-based) | SLEEP-based payloads for safe detection |
| SQL injection (error-based) | Syntax error triggers |
| SQL injection (UNION-based) | Column count detection and data extraction |
| XSS (reflected) | Marker tag injection with `KTP_` prefix |
| CSRF | Token removal, invalidation, and replacement |
| Authentication bypass | Header removal and invalid token injection |
| Authorization bypass | Role-based token swapping |

The payload reference enforces strict safety rules:

- No destructive SQL (DROP, DELETE, UPDATE, INSERT, TRUNCATE)
- No `OR 1=1` on write endpoints (POST/PUT/PATCH/DELETE)
- Time-based blind techniques preferred for safe detection on all methods

### references/playwright-capture.md -- Traffic capture

A step-by-step workflow for capturing browser traffic using playwright-cli:

1. Start the proxy with a focused capture scope
2. Open pages with playwright-cli (proxied through yorishiro-proxy)
3. Verify proxy connectivity by checking flow count
4. Review and map captured flows to test operations

### references/self-contained-iteration.md -- Stateful fuzzing

Documents the "Self-Contained Iteration" pattern for testing stateful APIs:

- Each fuzz iteration independently sets up prerequisites (login, CSRF token, resource creation)
- Pre-send hooks run before the main request
- Post-receive hooks clean up after (e.g., logout)
- KV Store variables flow from pre-send macros to the main request and post-receive macros

### references/verify-vulnerability.md -- Verification workflow

A complete 5-phase vulnerability verification workflow:

1. **Traffic capture** -- capture target application traffic
2. **Macro design** -- build macros for stateful operations
3. **Payload selection** -- choose appropriate non-destructive payloads
4. **Testing** -- single-shot resend, then fuzz for coverage
5. **Analysis** -- review results, compare responses, report findings

## Interaction patterns

### Natural language requests

With the skills installed, you can describe testing goals in natural language:

**Basic testing:**

- "Start the proxy and capture traffic from api.target.com"
- "Show me all POST requests to /api/users"
- "Resend this request with a different Authorization header"

**Vulnerability testing:**

- "Test the /api/users/{id} endpoint for IDOR by enumerating IDs 1-20"
- "Check if the search parameter is vulnerable to SQL injection"
- "Fuzz the login endpoint for authentication bypass"
- "Verify CSRF protection on all state-changing endpoints"

**Workflow management:**

- "Set up a macro for the login flow and extract the session cookie"
- "Run a fuzz campaign with the auth macro as a pre-send hook"
- "Export the assessment results as HAR"

### Agent decision flow

When the agent receives a testing request, the skill guides it through a decision tree:

1. **Does traffic need to be captured?** -- If yes, follow the playwright-capture workflow
2. **Is the operation stateful?** -- If yes, design macros using the self-contained-iteration pattern
3. **Are attack payloads needed?** -- If yes, consult the payload-patterns reference
4. **Single test or comprehensive?** -- Use `resend` for single-shot, `fuzz` for coverage
5. **Need response comparison?** -- Use `resend compare` for structural diff

### MCP resource access

The skills teach the agent to use MCP resources for detailed help. When the agent needs more information about a tool, it can access:

**Help resources** (usage, parameters, examples):

- `yorishiro://help/proxy_start`
- `yorishiro://help/query`
- `yorishiro://help/resend`
- `yorishiro://help/fuzz`
- `yorishiro://help/macro`
- `yorishiro://help/security`
- `yorishiro://help/examples`

**Schema resources** (JSON Schema):

- `yorishiro://schema/proxy_start`
- `yorishiro://schema/fuzz`
- `yorishiro://schema/resend`

## Example session

Here is a typical interaction flow with the skills installed:

**You:** "Test the /api/users endpoint for IDOR vulnerabilities"

**Agent actions:**

1. Checks if the proxy is running (`query status`)
2. Finds flows matching `/api/users` (`query flows`)
3. Identifies the flow with user ID in the path
4. Runs a single resend with a different user ID to validate the approach
5. Launches a fuzz campaign with an ID range
6. Reviews results, filtering for outliers and 200 responses
7. Reports findings with evidence

**You:** "Now check the same endpoint for SQL injection"

**Agent actions:**

1. Selects time-based blind SQLi payloads (safe for all HTTP methods)
2. Runs a fuzz campaign targeting the user ID parameter
3. Sorts results by `duration_ms` to identify timing anomalies
4. Reports whether any payloads caused a significant delay

## Related pages

- [Quick setup](../getting-started/quick-setup.md) -- initial installation guide
- [MCP configuration](../getting-started/mcp-configuration.md) -- MCP server setup
- [Vulnerability assessment](vulnerability-assessment.md) -- full assessment workflow
- [MCP tools overview](../tools/overview.md) -- all MCP tools reference
