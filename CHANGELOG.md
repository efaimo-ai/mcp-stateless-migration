# Changelog

All notable changes to mcp-stateless-migration are documented here. Format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/); this project uses
[Semantic Versioning](https://semver.org/).

## [0.1.1] - 2026-11-16

### Changed

- **Commands pin `efaimo@0.6.2`**, two efaimo releases newer
  than the pin in 0.1.0. Between them they mask keys in the `check --mcp`
  output (in a URL's query or username, or after `Bearer`), end a stdio server
  whose handshake fails instead of leaving it running, and keep what a working
  server writes to stderr out of the report.

## [0.1.0] - 2026-11-16

> **Published without provenance.** This is the first version of `mcp-stateless-migration` on
> npm, and it came from an authenticated session: npm's trusted publishing is
> configured from an existing package's settings page, so it cannot create a
> package that has never been published. From the next version the release
> workflow publishes it, with SLSA provenance naming the commit that built it.

### Added

- **The skill is installable.** `npx mcp-stateless-migration` copies `SKILL.md` and its
  `references/` into `./.claude/skills/mcp-stateless-migration/`, with `--global`, `--dir`,
  `--check`, `--print`, `--uninstall` and `--force`. The installer reads every
  byte back after writing it and fails if what landed is not what it wrote, and
  it prints how many files it installed rather than only that it succeeded.
- Three exit codes rather than two: 0 installed or already identical, 1 present
  and different, 2 could not run at all. "It differs" and "I could not do this"
  are different answers and collapsing them is how a broken install reads as a
  clean one.
- CI on four cells (Ubuntu and Windows, Node 20 and 24), including an end to end
  run that packs the package the way npm would, installs it from that tarball,
  and installs the skill from there.

### Changed

- The skill's content was re-verified on 2026-11-16 against the published
  spec, the SDK releases and the conformance suite, and revised. It now asks
  first which clients still connect and recommends serving 2025-11-25 and
  2026-07-28 on one endpoint unless every client speaks 2026-07-28. It names
  the SDK entry points that actually switch the wire (TypeScript
  `serveStdio` / `createMcpHandler`, Go `Stateless = true`, Python
  `MCPServer`), seals `requestState` and binds it to the request, warns that a
  retry can run a mutating tool twice and that a request without `logLevel`
  gets no logs, and proves the result with the conformance `--requirements`
  sets.
- Step 1 starts with a hand audit against `references/changes.md`.
  `efaimo check --mcp` is the option for when you can start or reach the
  server and want the list generated. The `efaimo weigh` suggestions (the last
  trap, and the cost section of `references/verify.md`) are gone: the migration
  does not need them.
- No longer says SEP-2596 keeps 2025-11-25 servers working until a date. It
  governs deprecated features; how long a revision is served is still open.
- Commands name the published `efaimo@0.6.0` and an exact conformance version,
  each with the date it was read from npm.
- New `references/sdks.md`: entry points, versions and client behaviour per
  SDK, dated 2026-11-16, to re-check when the next revision ships.
- `references/changes.md` keeps its thirteen and adds an "Also changed"
  section (the removed elicitation completion notification, the deprecated
  HTTP+SSE transport), and corrects the `_meta` key list to six.

### Notes

- The package version and the version in `SKILL.md` frontmatter are one
  number, pinned by a test.

[0.1.1]: https://github.com/efaimo-ai/mcp-stateless-migration/releases/tag/v0.1.1
[0.1.0]: https://github.com/efaimo-ai/mcp-stateless-migration/releases/tag/v0.1.0
