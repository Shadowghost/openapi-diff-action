# OpenAPI Diff Action

A GitHub Action to compare two OpenAPI specifications and detect breaking changes using [openapitools/openapi-diff](https://github.com/OpenAPITools/openapi-diff).

## Features

- Compare local files or remote URLs
- Automatic flattening of specs to handle circular `$ref` references
- Multiple output formats: HTML, Markdown, JSON, Asciidoc, Text
- Automatic artifact upload for report files (when changes detected)
- Collapsible sections in markdown output for better readability
- Fail conditions for CI/CD gates
- Built-in PR comment integration (creates or updates existing comment)
- GitHub job summary integration
- Authentication support for private spec URLs

## Quick Start

```yaml
- uses: Shadowghost/openapi-diff@v1
  with:
    old-spec: 'specs/openapi-v1.yaml'
    new-spec: 'specs/openapi-v2.yaml'
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `old-spec` | Yes | - | Path or URL to the old/base OpenAPI specification |
| `new-spec` | Yes | - | Path or URL to the new/head OpenAPI specification |
| `html` | No | - | Path to write HTML diff report |
| `markdown` | No | - | Path to write Markdown diff report |
| `json` | No | - | Path to write JSON diff report (warning: can be very large for complex APIs) |
| `asciidoc` | No | - | Path to write Asciidoc diff report |
| `text` | No | - | Path to write plain text diff report |
| `fail-on-breaking` | No | `false` | Fail if breaking changes detected |
| `fail-on-changed` | No | `false` | Fail if any changes detected |
| `headers` | No | - | HTTP headers for fetching remote URL specs (format: `Header=Value,Header2=Value2`) |
| `log-level` | No | `ERROR` | Log level (TRACE, DEBUG, INFO, WARN, ERROR, OFF) |
| `add-pr-comment` | No | `false` | Post results as PR comment (updates existing comment if present) |
| `github-token` | No | - | Token for PR comments (required if `add-pr-comment` is true) |
| `artifact-name` | No | `openapi-diff-report` | Name for the uploaded artifact containing report files |

## Outputs

| Output | Description |
|--------|-------------|
| `state` | Diff state: `no_changes`, `compatible`, or `incompatible` |
| `has-changes` | Whether changes were detected (`true`/`false`) |
| `is-breaking` | Whether breaking changes were detected (`true`/`false`) |

Report files are automatically uploaded as artifacts when changes are detected. Use the `artifact-name` input to customize the artifact name (required when using the action multiple times in the same workflow).

## Examples

### Basic Comparison

```yaml
name: API Diff
on: [push]

jobs:
  diff:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: Shadowghost/openapi-diff@v1
        id: diff
        with:
          old-spec: 'api/openapi-old.yaml'
          new-spec: 'api/openapi-new.yaml'
          markdown: 'diff-report.md'

      - name: Check results
        run: |
          echo "State: ${{ steps.diff.outputs.state }}"
          echo "Has Changes: ${{ steps.diff.outputs.has-changes }}"
          echo "Breaking: ${{ steps.diff.outputs.is-breaking }}"
```

### Fail on Breaking Changes

Use this to prevent merging PRs that introduce breaking API changes:

```yaml
name: API Compatibility Check
on: pull_request

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Get base spec
        run: git show origin/${{ github.base_ref }}:openapi.yaml > base-spec.yaml

      - uses: Shadowghost/openapi-diff@v1
        with:
          old-spec: 'base-spec.yaml'
          new-spec: 'openapi.yaml'
          fail-on-breaking: 'true'
```

### PR Comment with Diff Results

Automatically post the diff results as a comment on pull requests:

```yaml
name: API Review
on: pull_request

jobs:
  api-diff:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Get base spec
        run: git show origin/${{ github.base_ref }}:openapi.yaml > base-spec.yaml

      - uses: Shadowghost/openapi-diff@v1
        with:
          old-spec: 'base-spec.yaml'
          new-spec: 'openapi.yaml'
          add-pr-comment: 'true'
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

### Compare Remote Specs with Authentication

```yaml
- uses: Shadowghost/openapi-diff@v1
  with:
    old-spec: 'https://api.example.com/v1/openapi.json'
    new-spec: 'https://api.example.com/v2/openapi.json'
    headers: 'Authorization=Bearer ${{ secrets.API_TOKEN }}'
```

### Generate Reports with Automatic Artifact Upload

When you specify output file paths, the action automatically uploads them as artifacts (only when changes are detected):

```yaml
- uses: Shadowghost/openapi-diff@v1
  with:
    old-spec: 'old.yaml'
    new-spec: 'new.yaml'
    html: 'reports/diff.html'
    markdown: 'reports/diff.md'
    json: 'reports/diff.json'
    # Artifacts are automatically uploaded as 'openapi-diff-report'
```

If using the action multiple times in the same workflow, specify unique artifact names:

```yaml
- uses: Shadowghost/openapi-diff@v1
  with:
    old-spec: 'service-a/old.yaml'
    new-spec: 'service-a/new.yaml'
    markdown: 'reports/service-a.md'
    artifact-name: 'service-a-diff'

- uses: Shadowghost/openapi-diff@v1
  with:
    old-spec: 'service-b/old.yaml'
    new-spec: 'service-b/new.yaml'
    markdown: 'reports/service-b.md'
    artifact-name: 'service-b-diff'
```

### Conditional Steps Based on Changes

```yaml
- uses: Shadowghost/openapi-diff@v1
  id: diff
  with:
    old-spec: 'old.yaml'
    new-spec: 'new.yaml'

- name: Notify on breaking changes
  if: steps.diff.outputs.is-breaking == 'true'
  run: echo "Breaking changes detected!"

- name: Skip if no changes
  if: steps.diff.outputs.has-changes == 'false'
  run: echo "No API changes detected"
```

## Understanding Diff States

| State | Description |
|-------|-------------|
| `no_changes` | The specifications are identical |
| `compatible` | Changes detected but they are backward compatible |
| `incompatible` | Breaking changes detected that may affect API consumers |

### What Constitutes a Breaking Change?

- Removing an endpoint
- Removing a required request parameter
- Adding a new required request parameter
- Changing the type of a parameter
- Removing a response field that clients may depend on
- Changing authentication requirements

### What Are Compatible Changes?

- Adding a new endpoint
- Adding an optional request parameter
- Adding a new response field
- Deprecating (but not removing) an endpoint

## Technical Notes

### Circular Reference Handling

This action automatically flattens OpenAPI specifications using [oasdiff](https://github.com/oasdiff/oasdiff) before comparison. This resolves circular `$ref` references that would otherwise cause `StackOverflowError` in the underlying openapi-diff tool (see [issue #124](https://github.com/OpenAPITools/openapi-diff/issues/124)).

### Output Format

The markdown output includes collapsible `<details>` blocks for Request and Return Type sections, making large diffs more readable. When used with PR comments, the entire diff is wrapped in a collapsible section.

### PR Comment Behavior

When `add-pr-comment` is enabled, the action will:
- Look for an existing comment with the marker `<!--openapi-diff-workflow-comment-->`
- Update the existing comment if found, or create a new one if not
- This prevents multiple comments from accumulating on PRs with multiple pushes

### Size Limits and Large Diffs

For very large API diffs:
- The job summary and PR comments are truncated to ~950KB with a warning message directing users to the artifacts
- Full reports are uploaded as artifacts (untruncated)
- Artifacts are only uploaded when changes are detected
- JSON output can be extremely large for complex APIs (potentially gigabytes) - use with caution
