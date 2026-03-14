# Macros

The Macros page lets you define, edit, execute, and manage multi-step request sequences. Macros are useful for automating workflows like authentication flows, multi-step API interactions, and session management.

<!-- TODO: Add screenshot of the Macros page -->

## Macro list

The main page displays a table of all defined macros:

| Column | Description |
|--------|-------------|
| **Name** | Macro name (unique identifier) |
| **Description** | Optional description text |
| **Steps** | Number of steps in the macro (badge) |
| **Created** | Creation timestamp |
| **Updated** | Last modified timestamp |
| **Actions** | Run, Edit, Delete buttons |

Click any row to navigate to the macro detail/editor page.

### Actions

- **New Macro** -- Button in the header to create a new macro (navigates to `/macros/new`)
- **Run** -- Execute the macro immediately
- **Edit** -- Navigate to the macro editor
- **Delete** -- Delete the macro (with confirmation dialog)

The macro list polls for updates every 5 seconds.

## Macro editor

The detail page (`/macros/{name}`) provides a full editor with two tabs:

- **Editor** -- Define and modify macro steps
- **Run & Results** -- Execute the macro and view results

### Defining steps

Each macro consists of one or more steps. Each step specifies:

- **Step ID** -- Unique identifier within the macro
- **Flow ID** -- The captured flow to use as the base request
- **Override method** -- Optionally change the HTTP method
- **Override URL** -- Optionally change the request URL
- **Override headers** -- Key-value editor for header overrides
- **Override body** -- Optionally replace the request body
- **Error handling** -- What to do when a step fails:
    - **Abort** -- Stop the macro
    - **Skip** -- Skip the failed step and continue
    - **Retry** -- Retry the step (configurable retry count, delay, and timeout)

### Variable extraction

Each step can define extraction rules to capture values from the response for use in subsequent steps. Each extraction rule specifies:

- **Variable name** -- Name to store the extracted value (referenced as `{{variable_name}}` in later steps)
- **From** -- Whether to extract from the request or response
- **Source** -- Where to find the value:
    - **Header** -- Extract from a specific header
    - **Body (regex)** -- Extract using a regular expression with capture group
    - **Body (JSON path)** -- Extract using a JSON path expression
    - **Status code** -- Capture the response status code
    - **URL** -- Capture the request URL
- **Default value** -- Fallback value if extraction fails
- **Required** -- Whether the macro should fail if extraction fails

### Guard conditions

Steps can have conditional execution via guard conditions. A guard evaluates the result of a previous step and determines whether the current step should execute. Guard options include:

- **Step** -- The step whose result to evaluate
- **Status code** -- Match a specific status code
- **Status code range** -- Match a range of status codes
- **Header match** -- Match header key-value pairs
- **Body match** -- Match body content
- **Extracted variable** -- Check an extracted variable's value
- **Negate** -- Invert the condition

### Running a macro

From the **Run & Results** tab, click **Run** to execute the macro. Results are displayed as a table showing each step's outcome:

- Step ID and flow ID
- Response status code
- Duration
- Extraction results (variables captured from the response)
- Error information (if the step failed)

## Related pages

- [Macros feature](../features/macros.md) -- Detailed macros documentation
- [macro tool](../tools/macro.md) -- MCP tool reference
