# mcp-audit GitHub Action

[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-mcp--audit-purple?logo=github)](https://github.com/marketplace/actions/mcp-audit)
[![license: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](./LICENSE)
[![powered by @dj_abstract/mcp-audit](https://img.shields.io/badge/powered%20by-%40dj__abstract%2Fmcp--audit-cb3837?logo=npm)](https://www.npmjs.com/package/@dj_abstract/mcp-audit)

Static security audit for **Model Context Protocol (MCP) servers**, native to GitHub's Code Scanning UI. Catches prompt injection, tool poisoning, lethal trifecta capability combinations, schema permissiveness, and other AI-native vulnerabilities — all surfaced inline on your PRs via the Security tab.

Wraps the [`@dj_abstract/mcp-audit`](https://github.com/abregoarthur-star/mcp-audit) CLI in a single drop-in Action.

## What it does

1. Runs `mcp-audit scan` against your MCP server (manifest, stdio, or remote URL)
2. Emits a **SARIF 2.1.0** report that GitHub's code-scanning dashboard consumes natively
3. Uploads the SARIF so findings appear in the **Security tab** and inline on the **Files Changed** view
4. Fails the build at a configurable severity threshold so critical vulns block merge

## Usage

### Minimal — scan an extracted manifest

```yaml
name: MCP Audit
on: [push, pull_request]

permissions:
  contents: read
  security-events: write   # required for SARIF upload

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: abregoarthur-star/mcp-audit-action@v1
        with:
          manifest: ./dist/mcp-manifest.json
```

### Scan a local stdio server

```yaml
- uses: abregoarthur-star/mcp-audit-action@v1
  with:
    stdio: 'node ./packages/server/dist/index.js'
    fail-on: critical   # only fail on critical; warn on high
```

### Scan a deployed remote server

```yaml
- uses: abregoarthur-star/mcp-audit-action@v1
  with:
    url: https://mcp.example.com/sse
    bearer: ${{ secrets.MCP_BEARER }}
    fail-on: high
```

### Use the scan outputs in later steps

```yaml
- id: audit
  uses: abregoarthur-star/mcp-audit-action@v1
  with:
    manifest: ./mcp-manifest.json

- name: Post summary comment
  if: github.event_name == 'pull_request'
  uses: actions/github-script@v7
  with:
    script: |
      const body = `mcp-audit: ${{ steps.audit.outputs.finding-count }} findings — ${{ steps.audit.outputs.critical-count }} critical, ${{ steps.audit.outputs.high-count }} high`;
      github.rest.issues.createComment({ ...context.repo, issue_number: context.issue.number, body });
```

## Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `manifest` | one-of | — | Path to a static MCP manifest JSON |
| `stdio` | one-of | — | Command that spawns a local stdio MCP server |
| `url` | one-of | — | URL of a remote HTTP/SSE MCP server |
| `bearer` | no | — | Bearer token for authenticated remote servers |
| `fail-on` | no | `high` | Minimum severity that fails the build: `critical`, `high`, `medium`, `low` |
| `sarif-file` | no | `mcp-audit.sarif` | Where to write the SARIF report |
| `upload-sarif` | no | `true` | Whether to upload SARIF to GitHub Code Scanning |
| `mcp-audit-version` | no | `latest` | Pin to a specific `@dj_abstract/mcp-audit` version for reproducible CI |

## Outputs

| Name | Description |
|------|-------------|
| `sarif-file` | Path to the generated SARIF report |
| `finding-count` | Total findings |
| `critical-count` | Number of critical-severity findings |
| `high-count` | Number of high-severity findings |

## What the audit catches

- **prompt-injection** — instruction-override text in tool descriptions
- **invisible-instructions** — Unicode Tag smuggling, zero-width chars
- **tool-poisoning** — hidden capabilities, false read-only claims
- **unsafe-tool-combos** — lethal trifecta (shell + network; secret + network; file + network)
- **sensitive-output** — tools whose names suggest credential returns
- **destructive-no-confirm** — `delete_*`, `drop_*` without confirmation param
- **schema-permissiveness** — unbounded strings, missing `inputSchema`
- **unauthenticated-server** — remote servers with no auth
- **excessive-scope** — single server spanning too many capability domains

Full rule catalog: [mcp-audit README → What it checks](https://github.com/abregoarthur-star/mcp-audit#what-it-checks)

## Permissions required

```yaml
permissions:
  contents: read
  security-events: write
```

`security-events: write` lets the action upload SARIF to the Code Scanning dashboard. Without it, findings still appear in the Action log but don't reach the Security tab.

## Related tools

Part of a **detect → inventory → test → generate → defend** AI-security pipeline:

- **[`mcp-audit`](https://github.com/abregoarthur-star/mcp-audit)** — the CLI this Action wraps
- **[`mcp-audit-sweep`](https://github.com/abregoarthur-star/mcp-audit-sweep)** — reproducible audit of public MCP servers (methodology)
- **[`agent-firewall`](https://github.com/abregoarthur-star/agent-firewall)** — call-time defensive middleware (runtime)
- **[`prompt-eval`](https://github.com/abregoarthur-star/prompt-eval)** — prompt-injection eval harness
- **[`prompt-genesis`](https://github.com/abregoarthur-star/prompt-genesis)** — adversarial attack corpus generator

## License

MIT — see [LICENSE](./LICENSE).
