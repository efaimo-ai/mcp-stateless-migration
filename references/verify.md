# Proving the migration landed

Two independent checks, because each misses what the other catches. A linter
reads shapes it knows about; the conformance suite drives real scenarios. Green
on one is a partial answer.

## 1. The readiness diff

```bash
npx efaimo@0.6.1 check --mcp "npx -y <the-server>"
```

`efaimo@0.6.1` is the `latest` release on npm as of 2026-11-16. Expect:

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
- `--strict-readiness` to exit non-zero when the readiness list is not empty.
  This is the one to put in CI for the migration.
- `--strict` exits non-zero on quality warnings only; it does not look at
  readiness.

## 2. The official conformance suite

Run it directly, pinned to an exact version and to a revision's requirement
set:

```bash
npx @modelcontextprotocol/conformance@0.2.0-alpha.12 server \
  --url https://host/mcp --requirements 2026-07-28
# dual-era servers: the legacy set as well
npx @modelcontextprotocol/conformance@0.2.0-alpha.12 server \
  --url https://host/mcp --requirements 2025-11-25
```

`0.2.0-alpha.12` is what the `alpha` dist-tag pointed at on npm on 2026-11-16.
Do not write `@alpha` in a script: it moved on 2026-07-27, 2026-08-07 and
2026-10-01. Without a version at all you get `latest`, which on that date was
0.1.16 from 2026-03-30, older than this revision, so it will happily test your
2026-07-28 server against the old protocol and report success.

`--requirements` (0.2.0-alpha.11 and later) runs exactly the scenarios that
revision required when it shipped, at that revision's wire version, and
replaces `--suite`, `--spec-version` and `--scenario`. The two eras use
different wires, so a scenario that belongs to both runs twice, once per set.
Passing one says nothing about the other.

`efaimo@0.6.1` can run the suite for you:

```bash
npx efaimo@0.6.1 check --mcp https://host/mcp --conformance
```

It prints the exact command line before running it, so you can rerun it by
hand or in CI. It needs an HTTP target; a stdio server is skipped with a note.
That release pins `@modelcontextprotocol/conformance@0.2.0-alpha.10` with
`--spec-version 2026-07-28`. alpha.10 is the release the frozen 2026-07-28 set
is anchored to, so it approximates the `--requirements 2026-07-28` run above,
and it does not run the 2025-11-25 set.

## 3. Check the five things checkers miss

These are shape changes a passing suite can still leave wrong, because they are
about which shape you emit rather than whether you answer:

1. `server/discover` carries `ttlMs` and `cacheScope`. It only became cacheable
   in the published spec.
2. Identity is in `_meta["io.modelcontextprotocol/serverInfo"]`, not at the top
   level of the discover result.
3. `cacheScope` is `"private"` on anything that varies per user. `"public"` on
   a per user result is a cache poisoning bug, not a style choice.
4. Nothing rejects a request for missing `clientInfo`, which is optional.
5. If you chose dual-era, a legacy client still connects. Run the
   `2025-11-25` requirement set as well, and open one `initialize` session by
   hand.

## A note on trusting green

A check you have never seen fail is not a check. Before you believe any of the
above, break the server on purpose once: delete `resultType` from one result,
or drop `cacheScope` from `server/discover`, and confirm the tool goes red and
names the right surface. Then put it back. That takes a minute and it is the
difference between a verified migration and a hopeful one.
