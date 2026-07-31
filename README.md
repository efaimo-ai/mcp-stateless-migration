# mcp-stateless-migration

An Agent Skill for migrating an MCP server to the stateless MCP specification
published on **2026-07-28**.

## Why a skill and not a blog post

The revision published on 2026-07-28. That is after the training cutoff of the
models doing the migration, so a coding agent asked to "make this server
2026-07-28 ready" is working from the old protocol or from guesses. Worse, most
migration writing on the web was produced during the Release Candidate window
that ran from 2026-05-21, and the RC is not the published spec: server identity
moved, `DiscoverResult` became cacheable, and three error codes were
renumbered. Guidance that names `DiscoverResult.serverInfo` or error `-32003`
is reading the RC.

This skill carries the published shapes, the deltas from the RC, and the two
commands that prove the migration landed.

Everything in it is verified against `schema/2026-07-28/schema.ts` at tag
`2026-07-28` in `modelcontextprotocol/modelcontextprotocol`, not against a
summary. `references/rc-vs-final.md` includes the commands to reproduce the
RC-to-final diff yourself.

## Install

Copy the directory into your skills location, for example:

```bash
git clone https://github.com/efaimo-ai/mcp-stateless-migration \
  ~/.claude/skills/mcp-stateless-migration
```

## What is in it

| file | loaded | contents |
|---|---|---|
| `SKILL.md` | on trigger | the three step procedure: measure, change, prove |
| `references/changes.md` | on demand | all twelve changes with their SEP or PR, and the shapes to emit |
| `references/rc-vs-final.md` | on demand | what moved between the locked RC and the published spec |
| `references/verify.md` | on demand | how to prove it landed, and what a passing suite still misses |

## Its own numbers

Audited by [efaimo](https://github.com/efaimo-ai/efaimo), which is the same
tool this skill tells you to run:

```
$ npx efaimo check --skill ./mcp-stateless-migration
grade A (100)   0 errors  0 warnings  0 info
  no findings. clean.

$ npx efaimo weigh ./mcp-stateless-migration
  skill                        metadata      body  lines  refs
  mcp-stateless-migration           104     1,173     95  3 files 3,139
```

104 tokens sit in your context at all times, which is what every installed
skill costs you whether or not you use it. The 1,173 token body loads only when
the skill triggers, and the 3,139 tokens of reference material only when it is
actually read. Re-run both commands yourself; that is the point of them.

## Licence

Apache-2.0.
