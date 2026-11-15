# SDKs and clients, as checked on 2026-11-16

Versions go stale; the entry points do not. Every version and date below was
read from the package registry or the SDK's own release page on 2026-11-16.
Re-check them before quoting them, and re-check this whole file when the next
revision ships (planned for 2027-03-30:
https://blog.modelcontextprotocol.io/posts/mcp-release-cycle/).

## What to change, per SDK

| SDK | first release speaking 2026-07-28 | latest on 2026-11-16 | what you must change |
|---|---|---|---|
| TypeScript `@modelcontextprotocol/server` | 2.0.0, 2026-07-27 (UTC) | 2.4.0, 2026-10-23 | Serve through `serveStdio(factory)` or `createMcpHandler(factory)`. A server connected to `StdioServerTransport` by hand stays 2025 only. Both entries serve both eras by default; `{ legacy: 'reject' }` makes them modern only. |
| Python `mcp` | 2.0.0, 2026-07-28 | 2.3.0, 2026-10-02 | `FastMCP` became `MCPServer` (`mcp.server.fastmcp` no longer imports). Both eras are served with nothing to configure. 1.x gets security fixes only; `mcp>=1.28,<2` keeps you there meanwhile. |
| Go `github.com/modelcontextprotocol/go-sdk` | v1.7.0, 2026-07-28 | v1.8.0, 2026-09-14 | `StreamableHTTPOptions.Stateless = true`, or clients negotiate down to 2025-11-25. Legacy clients keep working on every endpoint. |
| C# `ModelContextProtocol` | v2.0.0, 2026-07-28 | v2.4.0, 2026-11-07 | `HttpServerTransportOptions.Stateless` defaults to `true` from 2.0.0; peers on 2025-11-25 and earlier still interoperate. |
| Rust `rmcp` | 3.0.0, 2026-07-28 | 3.10.0, 2026-11-12 | `stateful_mode` is now `legacy_session_mode` and only governs older revisions. State that lived in the handler moves outside it. |
| Java `io.modelcontextprotocol.sdk` | preview in v2.2.0-M1, 2026-10-16 | v2.2.0-M2, 2026-11-13 (milestone) | The modern server is a preview in `io.modelcontextprotocol.modern`, separate from the legacy schema; the release notes call it the first step. The stable 2.1.0 has no modern server. |
| FastMCP (`fastmcp` on PyPI, not the SDK) | 4.0.0, 2026-08-31 | 4.2.0, 2026-11-13 | One deployment negotiates the best revision per connection. `ctx.elicit()` works on the old protocol only; server-initiated sampling and roots are gone. |

Sources, one per row:

- TypeScript: https://github.com/modelcontextprotocol/typescript-sdk/blob/main/docs/migration/support-2026-07-28.md
- Python: https://github.com/modelcontextprotocol/python-sdk/releases/tag/v2.0.0 and
  https://github.com/modelcontextprotocol/python-sdk/blob/v2.0.0/docs/migration.md
- Go: https://github.com/modelcontextprotocol/go-sdk/releases/tag/v1.7.0
- C#: https://github.com/modelcontextprotocol/csharp-sdk/releases/tag/v2.0.0
- Rust: https://github.com/modelcontextprotocol/rust-sdk/releases/tag/rmcp-v3.0.0
- Java: https://github.com/modelcontextprotocol/java-sdk/releases/tag/v2.2.0-M1
- FastMCP: https://github.com/PrefectHQ/fastmcp/releases/tag/v4.0.0

## The TypeScript codemod and what it leaves

```bash
npx @modelcontextprotocol/codemod@2.4.0 v1-to-v2 .
```

2.4.0 is the version npm reported for the codemod on 2026-11-16 (the SDK's own
docs write `@latest`). Run it at the package root. It rewrites the v1 SDK
surface to v2 (imports, renames). It does not adopt 2026-07-28: the TypeScript migration guide calls
that step "architectural and not codemod-automatable". After the codemod,
swap the entry point as in the table, then look at the cache hints. When
serving 2026-07-28 the SDK always emits `ttlMs` and `cacheScope`, defaulting to
`0` and `"private"`; set `ServerOptions.cacheHints` to advertise a real policy.

The SDK also publishes the v1 to v2 guide as an agent skill,
`mcp-typescript-sdk-upgrade-to-v2`. It runs the codemod first and covers what
is left of the API change; this skill covers the protocol decision and the
proof. How to install it is in the SDK's migration index:
https://github.com/modelcontextprotocol/typescript-sdk/blob/main/docs/migration/index.md

Python has no codemod. Its migration guide lists each error message next to
the change that causes it, which is the fastest way through.

## Clients

- **Claude Code** negotiates 2026-07-28 by default: local (stdio) servers from
  2.1.292, claude.ai connectors from 2.1.295, cloud session and Remote Control
  connectors from 2.1.299. `MCP_PROTOCOL_NEGOTIATION=legacy` opts out. It
  probes first and falls back to the legacy handshake for a server that does
  not answer as a modern one (since 2.1.318 a stdio server that exits on the
  probe is no longer re-probed on every start). Source:
  https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md
- **TypeScript SDK `Client`** still opens with the 2025 handshake unless
  `versionNegotiation` says otherwise, and does not attach `logLevel` by
  default, so server logs on a modern exchange go nowhere until it does
  (support-2026-07-28.md above).
- **Python SDK `Client`** negotiates the version automatically (v2.0.0
  release notes above).
- VS Code, Cursor, ChatGPT and Codex: not confirmed from a primary source on
  2026-11-16. Ask the user which of them connect, and test with each.
