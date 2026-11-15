---
name: mcp-stateless-migration
description: Migrate an MCP server to the stateless MCP protocol revision 2026-07-28, or make it serve both 2026-07-28 and 2025-11-25 clients. Use when an MCP server still depends on initialize, Mcp-Session-Id, server-initiated sampling/createMessage, roots/list or elicitation/create, logging/setLevel, MCP ping or Last-Event-ID resumption; when moving an MCP server to SDK v2 (@modelcontextprotocol/server, Python mcp 2.x, go-sdk Stateless, rmcp 3); when asked about server/discover, resultType, ttlMs, cacheScope, requestState, subscriptions/listen or Multi Round-Trip Requests; or when an MCP server fails against a client that negotiates 2026-07-28. Not for sessions, pings or SSE outside MCP.
license: Apache-2.0
metadata:
  version: "0.1.0"
  homepage: "https://efaimo.com"
  verified_against: "2026-11-16"
---

# Migrating an MCP server to 2026-07-28

The MCP specification published on 2026-07-28 removes the `initialize`
handshake and protocol level sessions, and calls itself a breaking change.
Existing pairs kept working that day because nothing forces either side to
move: a 2025-11-25 client and a 2025-11-25 server still talk. Mixed pairs are
what break. A server that speaks only 2026-07-28 fails every client that still
opens with `initialize`, and those clients have no way to fall forward. The
spec's compatibility matrix spells out every pairing:
https://modelcontextprotocol.io/specification/2026-07-28/basic/versioning#compatibility-matrix

The spec sets no date until which 2025-11-25 must be served. SEP-2596 governs
deprecated features, not protocol revisions, and lists the revision support
window as an open question, so for now that is each SDK's and each client's
own policy. The floor it does set is for features: Roots, Sampling and Logging
stay in the spec until at least the first revision released on or after
2027-07-28.

**Do not answer this from memory.** This revision published on 2026-07-28, later
than many model training cutoffs; check yours before trusting recall about it.
Anything written between 2026-05-21 and 2026-07-28 describes the Release
Candidate, which is not the same document, and there is a one line test for
that: if a source says server identity is `DiscoverResult.serverInfo`, or names
error code `-32003` or `-32004`, it is reading the RC and is wrong about the
published spec. See `references/rc-vs-final.md`.

## Step 0: ask which clients still connect

Before changing anything, ask whether clients that open with `initialize`
(anything on 2025-11-25 or earlier) still connect to this server.

- **Dual-era, the default.** Recommend it unless the user knows every client
  speaks 2026-07-28. Serve requests that carry the per-request `_meta`
  statelessly, and keep answering `initialize` for clients that still send it.
  The spec allows both eras on one endpoint (a MAY in `basic/versioning`), and
  the TypeScript, Python, Go, C# and Rust SDKs do it by default once you use
  their modern entry point (`references/sdks.md`).
- **Modern only**, when every client is known to speak 2026-07-28: answer
  `initialize` with an error that names the versions you support, answer HTTP
  GET and DELETE on the endpoint with 405, and ignore `Mcp-Session-Id` and
  `Last-Event-ID`. Legacy clients will fail.

Write the choice down. Step 3 tests both eras if you chose dual-era.

## Step 1: measure, do not guess

Get the actual diff for this server before changing a line:

```bash
npx efaimo@0.6.0 check --mcp "npx -y <the-server>"      # stdio
npx efaimo@0.6.0 check --mcp https://host/mcp           # remote
npx efaimo@0.6.0 check --mcp "<server>" --repo ./src    # also scan source
```

`efaimo@0.6.0` is the `latest` release on npm as of 2026-11-16; the commands in
this skill describe what that release does.

It prints a quality grade and, separately, a `2026-07-28 readiness` list: each
item is a rule id (E101 to E118), what breaks, and the SEP that changed it. Work
that list. A server with zero items still needs Step 3.

If the server will not start, read `references/changes.md` and audit by hand
against the same list.

## Step 2: make the changes

Ordered by what blocks the rest. Full detail per item, with the shapes and the
SEP links, is in `references/changes.md`.

1. **Upgrade the SDK, then switch to its modern entry point.** In TypeScript
   and Go the upgrade alone puts nothing new on the wire.
   - TypeScript, `@modelcontextprotocol/server` 2.x: for the v1 to v2 API, run
     `npx @modelcontextprotocol/codemod@latest v1-to-v2 .` at the package root.
     The codemod does not adopt 2026-07-28. Replace
     `server.connect(new StdioServerTransport())` with
     `serveStdio(() => buildServer())`, and HTTP hosting with
     `createMcpHandler(factory)`.
   - Go, go-sdk v1.7 or later: set `StreamableHTTPOptions.Stateless = true`, or
     clients negotiate down to 2025-11-25.
   - Python, `mcp` 2.x: `FastMCP` is now `MCPServer`, so the old
     `mcp.server.fastmcp` import fails with `ModuleNotFoundError`. One
     `MCPServer` serves both eras with nothing to configure. There is no
     Python codemod.
   - Versions, the other languages, and the date all of it was checked:
     `references/sdks.md`.

   Behind the modern entry point the SDK emits `server/discover`, `resultType`
   and the cache fields for you. Check the cache defaults: TypeScript sends
   `ttlMs: 0` and `cacheScope: "private"` until you set hints.
2. **Answer requests with no `initialize`.** Every modern request carries
   `_meta["io.modelcontextprotocol/protocolVersion"]` and
   `_meta["io.modelcontextprotocol/clientCapabilities"]`. Read capabilities per
   request, never from a handshake. Keep the legacy path only if Step 0 said
   dual-era.
3. **Implement `server/discover`** (MUST). It returns `supportedVersions`,
   `capabilities`, and identity in
   `_meta["io.modelcontextprotocol/serverInfo"]`.
4. **Put `resultType` on every result**: `"complete"`, or `"input_required"`
   for a Multi Round-Trip interim result.
5. **Put `ttlMs` and `cacheScope` on every cacheable result**: the list results,
   the resource reads, and `server/discover` itself.
6. **Replace server-initiated requests with Multi Round-Trip Requests.**
   `roots/list`, `sampling/createMessage` and `elicitation/create` are gone as
   server-initiated calls. From `tools/call`, `prompts/get` or
   `resources/read` (and nothing else), return an `InputRequiredResult`:
   `resultType: "input_required"` with `inputRequests`, `requestState`, or
   both. The client retries the original request with a new JSON-RPC id,
   `inputResponses` under the same keys, and `requestState` echoed byte for
   byte. Ask only for what the request's `clientCapabilities` declare.
   `requestState` comes back through the client, so treat it as attacker
   controlled: if it affects authorization, resource access or business
   logic, seal it (HMAC or AEAD) and bind it to the principal, a short expiry
   and the original request. Details in `references/changes.md`, and the rule
   itself: https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/mrtr
7. **Replace cross-call state with server-minted handles** passed back as
   ordinary tool arguments. Nothing may live in a session.
8. **Move off the deprecated features** where you can: Roots, Sampling,
   Logging, and the HTTP+SSE transport if you still host it. The first three
   stay until at least the first revision released on or after 2027-07-28;
   the next revision is planned for 2027-03-30 (source in `references/sdks.md`),
   so in practice the one after it. HTTP+SSE can go sooner. Do these last, not
   first.

## Step 3: prove it, twice

A migration that has not been measured is not finished.

```bash
npx efaimo@0.6.0 check --mcp "<the-server>" --strict-readiness  # exits 1 unless readiness is clean
npx @modelcontextprotocol/conformance@0.2.0-alpha.12 server \
  --url https://host/mcp --requirements 2026-07-28
# dual-era: the legacy set too; one run does not cover the other
npx @modelcontextprotocol/conformance@0.2.0-alpha.12 server \
  --url https://host/mcp --requirements 2025-11-25
```

`0.2.0-alpha.12` is what the `alpha` dist-tag pointed at on npm on 2026-11-16.
`--requirements` runs exactly the scenarios that revision required when it
shipped. What `efaimo check --conformance` runs instead is in
`references/verify.md`. Expect the readiness list to be empty AND the suite to
pass. Either alone is a partial answer.

## Traps

- **Being lenient is not the same as being conformant.** A client should read
  identity from `_meta` first and fall back to the RC's top level `serverInfo`,
  because RC era servers are still out there. A server should emit only the
  published shape.
- **A dist-tag is not a pin.** `alpha` moved on 2026-07-27, 2026-08-07 and
  2026-10-01, and `latest` (0.1.16) predates this revision.
- **A retry can run a mutating tool twice.** Without `Last-Event-ID`, a broken
  stream loses the response and the client MUST re-issue the request with a
  new id (changelog, major change 9:
  https://modelcontextprotocol.io/specification/2026-07-28/changelog). A tool
  that charges, sends or writes
  needs its own idempotency key; the spec does not deduplicate, and that is
  still an open question:
  https://github.com/modelcontextprotocol/modelcontextprotocol/issues/3394
- **No `logLevel`, no logs.** A server MUST NOT send `notifications/message`
  for a request whose `_meta` lacks `io.modelcontextprotocol/logLevel`
  (changelog, major change 5). Logs that vanished after the migration are
  usually this.
- **On Streamable HTTP, a closed stream is the cancel.** There is no
  `notifications/cancelled` there any more; stop work when the request's SSE
  response stream closes.
- **Half migrated is worse than not migrated.** A server that negotiates
  2026-07-28 but omits `resultType`, `ttlMs` or `cacheScope` is broken for a
  modern client: Claude Code 2.1.302 had to work around servers in exactly
  that state showing no tools or failing every call
  (https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md). A server
  that is not modern yet just gets a 2025-11-25 connection. Do not advertise
  2026-07-28 until the shapes are in.
- **`Mcp-Name` is not required on every request.** Only on `tools/call`,
  `resources/read` and `prompts/get`. Sending a wrong value is worse than
  sending none: a mismatch with the body MUST be rejected with `-32020`.
- **Tool count does not predict context cost.** If you are touching the tool
  surface anyway, `npx efaimo@0.6.0 weigh "<server>"` prints per tool token
  cost.

## What this does not cover

`efaimo check --mcp` reports what it can reach over the wire and in source, and
two of the thirteen changes have no readiness rule behind them: the `Mcp-Method`
and `Mcp-Name` request headers (SEP-2243), and Tasks moving out of core into the
`io.modelcontextprotocol/tasks` extension (SEP-2663). **A clean readiness list
is not the same as all thirteen done.** Check those two by hand, and run the
official conformance suite, which is an independent reading of the same spec.
The changes that sit outside the thirteen (the removed elicitation completion
notification, the deprecated HTTP+SSE transport) are listed under "Also
changed" in `references/changes.md`.

## References

- `references/changes.md`: all thirteen changes, each with its SEP or PR, and
  what else changed outside that count.
- `references/sdks.md`: SDK versions, entry points and client behaviour, dated
  2026-11-16. Re-check it when the next revision ships.
- `references/rc-vs-final.md`: what moved between the locked RC and the
  published spec. Read this before trusting any guide written before 2026-07-28.
- `references/verify.md`: how to prove the migration landed.

<!-- generated:siblings -->

## Siblings

Every skill in this set is about a report that was true about the wrong thing. The set: https://efaimo.com/skills

- `unreleased-guard` - when the docs describe the migrated server and the published one has not migrated yet.
- `red-before-green` - before believing an empty readiness list, confirm the check can produce one.

<!-- /generated:siblings -->
