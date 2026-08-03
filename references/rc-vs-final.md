# What moved between the locked RC and the published spec

Read this before trusting any MCP migration guide dated before 2026-07-28.

The Release Candidate locked on 2026-05-21. The final published on 2026-07-28.
Anything written or built in those 68 days describes the RC, and the two are
not the same document.

Reproduce the whole delta yourself, in four commands:

```bash
# the RC kept it under draft/; the final gave it a dated home
curl -s https://raw.githubusercontent.com/modelcontextprotocol/modelcontextprotocol/2026-07-28-RC/schema/draft/schema.ts -o rc.ts
curl -s https://raw.githubusercontent.com/modelcontextprotocol/modelcontextprotocol/2026-07-28/schema/2026-07-28/schema.ts -o final.ts

# normalize the docs reorg, which rewrote every spec link, or it drowns the diff.
# no `sed -i` here: BSD/macOS sed requires an argument to -i and this would fail.
sed 's|/specification/draft/|/specification/2026-07-28/|g' rc.ts > rc-normalized.ts

git diff --no-index --stat rc-normalized.ts final.ts   # 209 insertions, 87 deletions
git diff --no-index rc-normalized.ts final.ts | less   # and here is what moved
```

The tags are 393 commits apart, which you can confirm without cloning:

```bash
gh api repos/modelcontextprotocol/modelcontextprotocol/compare/2026-07-28-RC...2026-07-28 --jq .total_commits
```

## The four that change working code

### 1. `DiscoverResult.serverInfo` was deleted

The RC carried server identity as an ordinary field on the discover result.
The published spec removed it and moved identity into
`_meta["io.modelcontextprotocol/serverInfo"]`, where it is **optional**.

```jsonc
// RC, and anything reading it
{ "supportedVersions": ["..."], "serverInfo": { "name": "x" } }

// published 2026-07-28
{ "supportedVersions": ["..."],
  "_meta": { "io.modelcontextprotocol/serverInfo": { "name": "x" } } }
```

Servers should emit only the published shape. Clients should read the published
shape first and fall back to the RC's field, because RC era servers exist and
dropping them is a regression dressed as conformance. A reader that only knew
the RC shape reports "identity unknown" for exactly the servers that finished
migrating, which is the worst direction for that error to point.

### 2. `DiscoverResult` became cacheable

RC: `interface DiscoverResult extends Result`.
Published: `interface DiscoverResult extends CacheableResult`.

So `ttlMs` and `cacheScope` are required on `server/discover` now and were not
under the RC. A server migrated against the RC passes on `tools/list` and fails
here, and most checkers do not look.

### 3. Three error codes were renumbered, one was added

| name | RC | published |
|---|---|---|
| MissingRequiredClientCapability | `-32003` | `-32021` |
| UnsupportedProtocolVersion | `-32004` | `-32022` |
| HeaderMismatch | `-32001`, in transport prose only | `-32020`, and now in the schema |

This table said HeaderMismatch "did not exist" in the RC, which was wrong in a
way worth naming: it existed at `-32001` in the transport prose and was absent
only from the RC's `schema.ts`. The changelog is explicit that all three moved
together: "Renumber the error codes introduced in this draft accordingly -
`HeaderMismatch` `-32001` -> `-32020`, `MissingRequiredClientCapability`
`-32003` -> `-32021`, `UnsupportedProtocolVersion` `-32004` -> `-32022`", and
separately that `HeaderMismatchError` was added to the schema, "which
previously existed only in transport prose". A team told the code "did not
exist" would leave a server returning `-32001` alone. The renumbering is
exactly the case they need to act on.

Also: `-32002` (resource not found) is retired in favour of the standard
`-32602`, and the spec promises it will never be reused.

The lesson is not the numbers, it is that they moved at all. Detect a condition
by its message or its shape, not by an implementation defined integer, and your
code survives the next renumbering too.

### 4. `clientInfo` became optional

RC: `"io.modelcontextprotocol/clientInfo": Implementation;` and the prose said
"Required."

Published: `"io.modelcontextprotocol/clientInfo"?: Implementation;` and the
prose says clients SHOULD include it unless configured not to, that it is self
reported and unverified, and that servers SHOULD NOT change behaviour or make
security decisions based on it.

If you wrote a server that rejects requests without `clientInfo`, that
rejection is no longer conformant.

## The rest, for completeness

- `CancelledNotification.params.requestId` went from optional to **required**.
- New `NotificationMetaObject` and `ResultMetaObject` narrow what `_meta` may
  carry on notifications and results respectively.
- `subscriptions/listen` gained `SubscriptionsListenResult`, whose `_meta`
  carries a **required** `io.modelcontextprotocol/subscriptionId`.
- `ElicitRequest.params.elicitationId` and
  `notifications/elicitation/complete` were removed outright.
- `ClientNotification` narrowed from `CancelledNotification |
  ProgressNotification` to just `CancelledNotification`.

## What did NOT change, so you can stop re-checking

- `resultType` is still required on every result.
- `CacheableResult` still carries `ttlMs` and `cacheScope`.
- `DiscoverResult` still carries `supportedVersions`.
- `METHOD_NOT_FOUND` is still `-32601`.
