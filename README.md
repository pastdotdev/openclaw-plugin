# past.dev memory for OpenClaw

Gives your [OpenClaw](https://openclaw.ai) agent a long-term memory in [past.dev](https://past.dev):
before each message, what past.dev remembers about it is added as background, and your conversations
are kept in past.dev, cut into sittings where they went quiet. The agent can also search its memory
with the `past_recall` tool.

## Install

```bash
openclaw plugins install git:github.com/pastdotdev/openclaw-plugin@main
openclaw config set plugins.slots.memory past
openclaw config set plugins.entries.past.hooks.allowConversationAccess true
openclaw config set plugins.entries.past.config.apiKey past_sk_...    # a project key from the past.dev console
openclaw config set plugins.entries.past.config.identity you@example.com
```

The second line makes past.dev OpenClaw's memory: one plugin holds that slot at a time. The third
lets the plugin read the conversation, which OpenClaw asks of every plugin that recalls before a
message or keeps what was said. Without it, the plugin recalls nothing and sends nothing.

`openclaw plugins inspect past --runtime --json` shows what the plugin registered. The environment
works as well: `PAST_API_KEY`, `PAST_IDENTITY`.

If another past.dev plugin already runs on this machine, the key and the identity are already set:
this plugin reads the same `~/.past/config.json`. `PAST_API_URL`, or `apiUrl` in the plugin's
config, points it at a self-hosted past.dev.

## What it sends

- **Recall:** the message you send, with obvious secrets (keys, tokens, `PASSWORD=...`) removed,
  as the query. Nothing is recalled until an identity is set.
- **Memory:** what you and OpenClaw said, without tool calls, their results or the agent's
  thinking, with the same redaction. A conversation is kept on this machine (`~/.past/openclaw/`)
  and sent once it has been quiet for 30 minutes, when its session ends, before it is compacted
  and when the Gateway stops. One that a crash left behind is sent at the next start.
- **Only your own conversations.** past.dev recalls as one identity, so the plugin works on the
  turns OpenClaw puts in your private memory, and leaves out group chats, other people's chats,
  cron runs, heartbeats and subagents. If every conversation this agent has is yours, set
  `"ownerOnly": false` in the plugin's config.

`"recall": false` or `"ingest": false` turns either off. `"sittingMinutes"` (at least 5) changes
how long a quiet spell cuts a conversation, and `"audience"` sends to one audience of the project
rather than to the whole project.

MIT licensed.
