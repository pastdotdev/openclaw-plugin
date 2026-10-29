# past.dev for OpenClaw: contributor guide

An OpenClaw memory plugin (`kind: "memory"` in `openclaw.plugin.json`), loaded in process by the
Gateway. The whole plugin is `index.js`: JavaScript that `tsc` checks (`// @ts-check`, `npm run
typecheck`), with no build step and no runtime dependency, so the file OpenClaw loads is the file in
this folder, whatever the install source. Node 24, as OpenClaw requires.

## How it fits together

```
OpenClaw turn
  before_prompt_build     POST /api/v1/recall, returned as prependContext: the block between
                          "=== past · recalled from memory ===" and "=== end of past.dev recall ==="
  past_recall tool        POST /api/v1/recall, as JSON
  agent_end               the run's messages merged into ~/.past/openclaw/sessions/<session>.json, nothing sent
  quiet for a sitting, session_end, before_compaction, gateway_stop
                          POST /api/v1/ingest/batch: every sitting whose content changed
  gateway_start           sends sessions a crash or a stop left behind
```

## The rules this code keeps

They are the rules every past.dev connector keeps, applied to OpenClaw:

1. **Never break the session.** Every hook catches its own failures; a failed send is kept in
   `~/.past/openclaw/state.json` as `lastError`. The recall hook gives up after 6 seconds, well
   inside the 15 seconds OpenClaw waits.
2. **Only prose travels.** User text and the assistant's text blocks; tool calls, tool results,
   thinking, OpenClaw's runtime context and the recall block are not sent.
3. **Send at boundaries, never every turn.** A session that grew is a new revision, priced on its
   whole content, so turns are kept on disk and sent once the conversation is quiet for a sitting's
   length, when the session ends or compacts, and when the Gateway stops.
4. **The hash gate.** Each sitting's content hash is kept; an unchanged sitting is not sent. A
   sitting is marked sent only after past.dev accepted it.
5. **Sittings.** A session is cut where it was silent for `sittingMinutes` (30, at least 5), pinned
   per session at its first send. The first sitting's id is `openclaw:<session>`, the next ones
   `openclaw:<session>:<n>`; each is timed at its first turn.
6. **Turns are added, never replaced.** A run's messages may be the whole session or what
   compaction left of it, so `agent_end` merges by role, timestamp and content hash.
7. **The owner's turns only.** past.dev recalls as one identity. The plugin acts only where
   OpenClaw resolves the turn's `memoryAudience` as `owner-private`, started by a person
   (`trigger` `user`, provenance `external_user` or absent). A conversation audience, a missing
   audience, cron, heartbeat and inter-session input get nothing, unless `ownerOnly` is false.
8. **Redact the query and the content alike.** Same patterns as past.dev's other plugins.
9. **`~/.past` is private.** Files are created `600` in a `700` folder. The key is read from the
   environment, the plugin's config or the shared `~/.past/config.json`, in that order.
10. **The product is written past.dev.** Every sentence a person or the model reads says past.dev.
    Identifiers keep `past`: the plugin id and memory slot (`past`), the tool `past_recall`,
    `~/.past`, the key prefix `past_sk_`, and the recall block's header, which other past.dev
    plugins drop by its exact text.

## Contracts

`POST /api/v1/recall` (needs `identity`), `POST /api/v1/ingest/batch` (`id` for replacement,
`timestamp`, `label`, `metadata`, optional `audience`), and OpenClaw's typed hooks:
`before_prompt_build`, `agent_end`, `before_compaction`, `session_end`, `gateway_start`,
`gateway_stop`, read from the plugin SDK's types in OpenClaw 2026.10.8. The prompt and
`agent_end` hooks run only with `plugins.entries.past.hooks.allowConversationAccess` set to true.
It is checked by hand against a real OpenClaw before each release, like the other connectors.
