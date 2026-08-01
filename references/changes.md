# The thirteen changes

A curated list of the changes that break working code, not the changelog's own
enumeration (it counts differently, and splits major from minor). The number in
this heading is the number of rows in the tables below; if you add a row,
change it. It said twelve until 2026-08-02, when the row replacing the HTTP GET
endpoint and `resources/subscribe` with `subscriptions/listen` turned out to be
missing, which is a breaking wire change and exactly the kind this file exists
to list.

Every row links the pull request that made it. Verified against
`schema/2026-07-28/schema.ts` at tag `2026-07-28` in
`modelcontextprotocol/modelcontextprotocol`, not against a summary.

Link form: `https://github.com/modelcontextprotocol/modelcontextprotocol/pull/<n>`

## Removed

| what | consequence | PR |
|---|---|---|
| `initialize`, `notifications/initialized` | Every request carries its own protocol version and client capabilities in `_meta`. | 2575 |
| `Mcp-Session-Id`, protocol level sessions | Cross call state becomes server minted handles, passed back as ordinary tool arguments. | 2567 |
| `Last-Event-ID`, SSE resumability | A broken response stream loses the in flight request. Clients re-issue it as a new one. | 2575 |
| The HTTP GET endpoint, `resources/subscribe`, `resources/unsubscribe` | Replaced by `subscriptions/listen`: one long lived POST response stream carrying the change notifications a client opted into. A server that still exposes GET for notifications, or answers `resources/subscribe`, is talking to nobody. | 2575 |
| `ping`, `logging/setLevel`, `notifications/roots/list_changed` | Log level moves per request into `_meta["io.modelcontextprotocol/logLevel"]`. | 2575 |

## Added

| what | consequence | PR |
|---|---|---|
| `server/discover` | Servers MUST implement it, to advertise supported versions, capabilities and identity. | 2575 |
| `resultType` on every result | `"complete"`, or `"input_required"` for a multi round trip interim result. | 2322 |
| Multi Round-Trip Requests | Replaces every server initiated request: `roots/list`, `sampling/createMessage`, `elicitation/create`. | 2322 |
| `Mcp-Method`, `Mcp-Name` headers | On Streamable HTTP POST. See the note below: they are not required equally. | 2243 |
| `ttlMs`, `cacheScope` | Required on every list and resource read result, so clients can cache and stop polling. | 2549 |

## Deprecated (still functional)

| what | consequence | PR |
|---|---|---|
| Roots, Sampling, Logging | New implementations should not adopt them. Minimum removal 2027-07-28. | 2577 |
| Dynamic Client Registration | In favour of Client ID Metadata Documents. Kept for servers without CIMD. | 2858 |

## Moved

| what | consequence | PR |
|---|---|---|
| Tasks | Out of the core protocol into an official extension, `io.modelcontextprotocol/tasks`. | 2663 |

## The shapes you actually have to emit

### `_meta` keys in the protocol

These are the seven reserved keys in the published schema. Anything else you
put in `_meta` is yours, and must not collide with this namespace.

```
io.modelcontextprotocol/protocolVersion     request  (client states the revision)
io.modelcontextprotocol/clientCapabilities  request
io.modelcontextprotocol/clientInfo          request, OPTIONAL (SHOULD send)
io.modelcontextprotocol/serverInfo          result,  OPTIONAL (SHOULD send)
io.modelcontextprotocol/logLevel            request
io.modelcontextprotocol/subscriptionId      notification, on subscription streams
io.modelcontextprotocol/tasks               extension settings
```

`clientInfo` and `serverInfo` are self reported and unverified. The spec says
explicitly that the other side SHOULD NOT change behaviour based on them and
SHOULD NOT use them for security decisions.

### `server/discover`

```jsonc
{
  "supportedVersions": ["2026-07-28"],
  "capabilities": { "tools": {} },
  "resultType": "complete",
  "ttlMs": 60000,
  "cacheScope": "public",
  "_meta": {
    "io.modelcontextprotocol/serverInfo": { "name": "your-server", "version": "1.0.0" }
  }
}
```

`DiscoverResult` extends `CacheableResult` in the published spec, so `ttlMs`
and `cacheScope` are required here too. Under the RC it extended plain
`Result` and they were not. This is the single most commonly missed item.

### Cacheable results

`ttlMs` (number) and `cacheScope` (`"public"` or `"private"`) are required on
`tools/list`, `prompts/list`, `resources/list`, `resources/read`,
`resources/templates/list` and `server/discover`.

`"public"` means any client or intermediary may cache and serve it to any user.
`"private"` means only the requesting user's client may cache it, and shared
caches MUST NOT serve it to a different user. If a result varies per user,
`"private"` is not optional.

### Multi Round-Trip Requests

Instead of the server calling the client, the server returns a result whose
`resultType` is `"input_required"`:

```jsonc
{
  "resultType": "input_required",
  "inputRequests": { "<key>": { /* a full elicitation or sampling request */ } },
  "requestState": "<opaque string the client echoes back unmodified>"
}
```

At least one of `inputRequests` or `requestState` MUST be present. Because
`requestState` carries everything needed to resume, the client's retry may land
on a different server instance. That is the point of the whole revision: put
the state in the message so the server can be ordinary infrastructure.

### Headers on Streamable HTTP POST

| header | value | required on |
|---|---|---|
| `MCP-Protocol-Version` | the revision | all requests |
| `Mcp-Method` | the body's `method` | all requests |
| `Mcp-Name` | `params.name` or `params.uri` | `tools/call`, `resources/read`, `prompts/get` only |

Servers MUST reject a request whose header disagrees with the body, or which
is missing a required header, with `-32020` (HeaderMismatch).

### Error codes

`-32000` to `-32019` stay implementation defined. `-32020` onward are allocated
by MCP, sequentially:

```
-32020  HeaderMismatch
-32021  MissingRequiredClientCapability
-32022  UnsupportedProtocolVersion
```

`-32002` (resource not found) is retired and replaced by the standard `-32602`.
The spec states it will never be reused. Do not match on `-32002`.
