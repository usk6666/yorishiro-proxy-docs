# Intercept

The Intercept page lets you review, modify, and control HTTP requests that match your intercept rules before they are forwarded to the target server. It operates as a request queue where you can inspect each intercepted request and decide what to do with it.

<!-- TODO: Add screenshot of the Intercept page -->

## Tabs

The page has two tabs:

- **Queue** -- Shows intercepted requests waiting for your action
- **Rules** -- Displays information about intercept rule configuration

## Intercept queue

The Queue tab displays a table of all currently intercepted requests. The queue polls every second for new entries. Each row shows:

| Column | Description |
|--------|-------------|
| **ID** | First 8 characters of the intercept entry ID |
| **Method** | HTTP method with color-coded badge (GET=green, POST=blue, PUT/PATCH=yellow, DELETE=red) |
| **URL** | Request path and query string |
| **Host** | Target hostname extracted from the URL |
| **Rules** | Badges showing which intercept rules matched this request |
| **Time** | Timestamp when the request was intercepted |

The page header shows a badge with the current queue count, highlighted in yellow when items are pending.

When the queue is empty, a message prompts you to configure intercept rules.

## Working with intercepted requests

Click a row in the queue to select it. The detail panel opens below the queue table, showing the full request with editable fields.

### Request editing

The detail panel provides inline editing for all request components:

- **Method** -- Dropdown to change the HTTP method (GET, POST, PUT, PATCH, DELETE, HEAD, OPTIONS)
- **URL** -- Text input to modify the full request URL
- **Headers** -- Key-value editor where you can:
    - Edit existing header names and values
    - Remove headers with the **x** button
    - Add new headers with **+ Add Header**
- **Body** -- Textarea for editing the request body

### Actions

Three action buttons appear in the detail header:

#### Release

Forwards the request to the target server **as-is**, without applying any of your edits. Use this when you want to let the request through unchanged.

#### Modify & forward

Applies your edits (method, URL, headers, body) and forwards the modified request to the target server. The modified headers are merged using comma concatenation for duplicate header names per RFC 7230.

#### Drop

Discards the intercepted request entirely. The request is never forwarded to the target server. A warning toast confirms the drop action.

## Rules tab

The Rules tab provides information about intercept rule management. Intercept rules are configured through the MCP `configure` tool rather than directly in the Web UI. The tab explains how to use the configure tool to add, remove, enable, or disable intercept rules.

!!! tip
    You can manage intercept rules from the [Settings](settings.md) page under the **Intercept Rules** tab, which provides a visual interface for rule management.

## Related pages

- [Intercept feature](../features/intercept.md) -- Detailed intercept documentation
- [intercept tool](../tools/intercept.md) -- MCP tool reference
- [Settings: Intercept rules](settings.md) -- Configure intercept rules via the Settings page
