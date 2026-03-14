# Technology detection

yorishiro-proxy automatically detects the technology stack of target applications by analyzing HTTP response headers, cookies, and body content. Detection results are stored as flow tags and aggregated per host for easy querying.

## How it works

When HTTP responses pass through the proxy, the fingerprinting engine analyzes:

- HTTP response headers (e.g. `Server`, `X-Powered-By`, `X-AspNet-Version`)
- Cookies (e.g. `PHPSESSID`, `JSESSIONID`, `csrftoken`)
- Response body content (e.g. generator meta tags, framework-specific patterns)

Detection results are stored in each flow's tags under the `technologies` key as a JSON array.

## Detection categories

The fingerprinting engine detects technologies across several categories:

| Category | Examples |
|----------|---------|
| Framework | Django, Rails, Express, Laravel, Spring |
| Language | PHP, Python, Java, Ruby, ASP.NET |
| Server | nginx, Apache, IIS, Cloudflare |
| CMS | WordPress, Drupal, Joomla |
| CDN | Cloudflare, AWS CloudFront, Akamai |
| Security | WAF signatures, CSP configurations |

## Querying detected technologies

Use the `query` tool to retrieve aggregated technology detection results per host:

```json
// query
{
  "action": "technologies"
}
```

The response groups detected technologies by host:

```json
{
  "hosts": [
    {
      "host": "api.target.com",
      "technologies": [
        {
          "name": "nginx",
          "version": "1.24.0",
          "category": "server",
          "confidence": "high"
        },
        {
          "name": "Django",
          "version": "",
          "category": "framework",
          "confidence": "medium"
        }
      ]
    },
    {
      "host": "www.target.com",
      "technologies": [
        {
          "name": "Apache",
          "version": "2.4",
          "category": "server",
          "confidence": "high"
        },
        {
          "name": "PHP",
          "version": "8.2",
          "category": "language",
          "confidence": "high"
        },
        {
          "name": "WordPress",
          "version": "6.4",
          "category": "cms",
          "confidence": "high"
        }
      ]
    }
  ],
  "count": 2
}
```

## Detection details

Each detection entry includes:

| Field | Description |
|-------|-------------|
| `name` | Technology name |
| `version` | Detected version (empty if unknown) |
| `category` | Technology category (server, framework, language, etc.) |
| `confidence` | Detection confidence level |

### Confidence levels

Detection results have a confidence level based on the signal strength:

- **high** -- detected from explicit headers (e.g. `Server: nginx/1.24.0`) or known unique indicators
- **medium** -- detected from indirect signals (e.g. default cookie names, common patterns)
- **low** -- detected from heuristic analysis

### Version detection

When version information is available in headers or response content, it is extracted and included. For example:

- `Server: nginx/1.24.0` produces version `"1.24.0"`
- `X-Powered-By: PHP/8.2.0` produces version `"8.2.0"`

If the version cannot be determined, the `version` field is empty.

### Deduplication

Technologies are deduplicated per host by name and category. When multiple flows for the same host detect the same technology, the entry with version information is preferred over one without.

## Practical use cases

### Reconnaissance

Technology detection provides automatic fingerprinting during the reconnaissance phase. After browsing a target application through the proxy, query the detected technologies to understand the stack:

```json
// query
{
  "action": "technologies"
}
```

### Targeted vulnerability testing

Use detection results to focus testing on known vulnerabilities for the detected stack. For example, if WordPress is detected, prioritize WordPress-specific vulnerability checks.

### Change detection

Compare technology detection results across different hosts or time periods to identify infrastructure changes or inconsistencies in a target's environment.

## Related pages

- [Query tool reference](../tools/query.md) -- Technologies query action
- [Flows](../concepts/flows.md) -- Flow tags and metadata
