# Proving the migration landed

Two independent checks, because each misses what the other catches. A linter
reads shapes it knows about; the conformance suite drives real scenarios. Green
on one is a partial answer.

## 1. The readiness diff

```bash
npx efaimo check --mcp "npx -y <the-server>"
```

Expect:

```
2026-07-28 readiness  clean, nothing to migrate
```

Anything else prints a rule id (E101 to E118), what breaks, the SEP behind it,
and a fix hint. The grade above it is quality only and does not move with
readiness, so an A does not mean migrated.

Useful flags:

- `--repo ./src` also greps source for removed primitives, which catches code
  paths a live probe never reaches.
- `--json` for CI.
- `--strict` to exit non-zero on warnings.

## 2. The official conformance suite

```bash
npx efaimo check --mcp https://host/mcp --conformance
```

This runs the official `@modelcontextprotocol/conformance` scenarios and prints
the exact command line before running it, so you can rerun it by hand or in CI.
It needs an HTTP target; a stdio server is skipped with a note.

To run it directly, pin an exact version rather than the `alpha` dist-tag,
which moves:

```bash
npx @modelcontextprotocol/conformance@<exact-version> server \
  --url https://host/mcp --spec-version 2026-07-28
```

`--spec-version` matters. Without it the suite runs its default selection, and
the `latest` line predates this revision entirely, so it will happily test your
2026-07-28 server against the old protocol and report success.

## 3. Check the four things checkers miss

These are shape changes a passing suite can still leave wrong, because they are
about which shape you emit rather than whether you answer:

1. `server/discover` carries `ttlMs` and `cacheScope`. It only became cacheable
   in the published spec.
2. Identity is in `_meta["io.modelcontextprotocol/serverInfo"]`, not at the top
   level of the discover result.
3. `cacheScope` is `"private"` on anything that varies per user. `"public"` on
   a per user result is a cache poisoning bug, not a style choice.
4. Nothing rejects a request for missing `clientInfo`, which is optional.

## 4. Do not skip the cost question

Migrating is a good moment to look at what the server costs the context window,
because the tool surface is already open in front of you:

```bash
npx efaimo weigh "npx -y <the-server>"
```

Tool count does not predict cost, and the gap is not small. In a measured
loadout of eight real servers, `notion-mcp-server` and `@playwright/mcp` both
expose exactly 24 tools and cost 17,218 and 3,453 tokens: a factor of five for
the same tool count. The difference is almost always description and schema
weight rather than the number of entries. Version pins and a per row reproduce
command are in
<https://github.com/efaimo-ai/efaimo/blob/main/research/mcp-stack-cost/REPORT.md>.

## A note on trusting green

A check you have never seen fail is not a check. Before you believe any of the
above, break the server on purpose once: delete `resultType` from one result,
or drop `cacheScope` from `server/discover`, and confirm the tool goes red and
names the right surface. Then put it back. That takes a minute and it is the
difference between a verified migration and a hopeful one.
