---
name: mcp-stateless-migration
description: Migrate an MCP server to the stateless MCP specification published 2026-07-28. Use when a server still calls initialize or Mcp-Session-Id, when it uses Sampling, Roots, Logging, ping, Last-Event-ID or elicitation/create, when the user asks about server/discover, resultType, ttlMs, cacheScope or Multi Round-Trip Requests, or when an MCP server stopped working against a 2026-07-28 client.
license: Apache-2.0
metadata:
  version: "0.1.0"
  homepage: "https://efaimo.ai"
---

# Migrating an MCP server to 2026-07-28

The MCP specification published on 2026-07-28 removes the `initialize`
handshake and protocol level sessions. The spec calls it a breaking change in
its own words. Nothing stopped working that day: the same release adopts a
minimum twelve month deprecation window (SEP-2596).

**Do not answer this from memory.** This revision published on 2026-07-28, later
than many model training cutoffs; check yours before trusting recall about it.
Anything written between 2026-05-21 and 2026-07-28 describes the Release
Candidate, which is not the same document, and there is a one line test for
that: if a source says server identity is `DiscoverResult.serverInfo`, or names
error code `-32003` or `-32004`, it is reading the RC and is wrong about the
published spec. See `references/rc-vs-final.md`.

## Step 1: measure, do not guess

Get the actual diff for this server before changing a line:

```bash
npx efaimo check --mcp "npx -y <the-server>"      # stdio
npx efaimo check --mcp https://host/mcp           # remote
npx efaimo check --mcp "<server>" --repo ./src    # also scan source
```

It prints a quality grade and, separately, a `2026-07-28 readiness` list: each
item is a rule id (E101 to E118), what breaks, and the SEP that changed it. Work
that list. A server with zero items still needs Step 3.

If the server will not start, read `references/changes.md` and audit by hand
against the same list.

## Step 2: make the changes

Ordered by what blocks the rest. Full detail per item, with the shapes and the
SEP links, is in `references/changes.md`.

1. **Upgrade the SDK first.** TypeScript `@modelcontextprotocol/server` 2.x,
   Python `mcp` 2.x; both cut 2.0.0 on 2026-07-27, the day before the spec
   published. The 2.x lines implement `server/discover`, `resultType` and the
   cache fields for you, which closes most of the list without hand written
   code. On TypeScript, `npx @modelcontextprotocol/codemod` rewrites the v1
   call sites to v2 for you. Do this before anything below: diff, codemod,
   re-check.
2. **Answer requests with no `initialize`.** Every request now carries
   `_meta["io.modelcontextprotocol/protocolVersion"]` and
   `_meta["io.modelcontextprotocol/clientCapabilities"]`. Drop the handshake
   and the `Mcp-Session-Id` header.
3. **Implement `server/discover`** (MUST). It returns `supportedVersions`,
   `capabilities`, and identity in
   `_meta["io.modelcontextprotocol/serverInfo"]`.
4. **Put `resultType` on every result**: `"complete"`, or `"input_required"`
   for a Multi Round-Trip interim result.
5. **Put `ttlMs` and `cacheScope` on every cacheable result**: the list results,
   the resource reads, and `server/discover` itself.
6. **Replace server initiated requests with Multi Round-Trip Requests.**
   `roots/list`, `sampling/createMessage` and `elicitation/create` are gone as
   server initiated calls. Return an `InputRequiredResult` instead.
7. **Replace cross call state with server minted handles** passed back as
   ordinary tool arguments. Nothing may live in a session.
8. **Move off the deprecated primitives** where you can: Roots, Sampling,
   Logging. These still work and have until 2027-07-28 at the earliest, so do
   them last, not first.

## Step 3: prove it, twice

A migration that has not been measured is not finished.

```bash
npx efaimo check --mcp "<the-server>"     # expect: readiness clean
npx efaimo check --mcp <url> --conformance # also runs the official suite
```

`--conformance` runs the official `@modelcontextprotocol/conformance` scenarios
against a remote server and prints the exact command first, so the result is
reproducible by hand. Expect the readiness list to be empty AND the conformance
suite to pass. Either alone is a partial answer.

## Traps

- **Being lenient is not the same as being conformant.** A client should read
  identity from `_meta` first and fall back to the RC's top level `serverInfo`,
  because RC era servers are still out there. A server should emit only the
  published shape.
- **A dist-tag is not a pin.** If you pin the conformance suite, pin an exact
  version. The `alpha` tag moved on 2026-07-27.
- **`Mcp-Name` is not required on every request.** Only on `tools/call`,
  `resources/read` and `prompts/get`. Sending a wrong value is worse than
  sending none: a mismatch with the body MUST be rejected with `-32020`.
- **Tool count does not predict context cost.** If you are touching the tool
  surface anyway, `npx efaimo weigh "<server>"` prints per tool token cost.

## What this does not cover

`efaimo check --mcp` reports what it can reach over the wire and in source, and
two of the twelve changes have no readiness rule behind them: the `Mcp-Method`
and `Mcp-Name` request headers (SEP-2243), and Tasks moving out of core into the
`io.modelcontextprotocol/tasks` extension (SEP-2663). **A clean readiness list
is not the same as all twelve done.** Check those two by hand, and run the
official conformance suite, which is an independent reading of the same spec.

## References

- `references/changes.md`: all twelve changes, each with its SEP or PR.
- `references/rc-vs-final.md`: what moved between the locked RC and the
  published spec. Read this before trusting any guide written before 2026-07-28.
- `references/verify.md`: how to prove the migration landed.
