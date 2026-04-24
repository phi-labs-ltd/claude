# claude

Claude Code plugins by and for Phi Labs Ltd.

## Plugins

| Plugin | Ships | Description |
| --- | --- | --- |
| [`personalities`](./plugins/personalities) | Output styles | Different voices for Claude. Currently: **Crypto Bro**. |
| [`developer`](./plugins/developer) | Agents, skills | Developer tooling. Currently: the **`second-set-of-eyes`** review agent and the **`karpathy`** engineering-discipline skill. |
| [`bolt-theme`](./plugins/bolt-theme) | Theme | A dark color theme with Bolt Liquidity's signature yellow accent. |

## Install

Add the marketplace once:

```
/plugin marketplace add phi-labs-ltd/claude
```

Then install any plugin you want:

```
/plugin install personalities@phi-labs-ltd
/plugin install developer@phi-labs-ltd
/plugin install bolt-theme@phi-labs-ltd
```

## Using each plugin

### `personalities`

Pick a voice with `/config` → **Output style** → **personalities:crypto-bro**. Revert with `/config` → **Output style** → **Default**.

### `developer`

- **`second-set-of-eyes`** — an independent-review agent that reads your pending diff with the critical eye of a senior engineer. Ask for a "second opinion", "sanity check", or "review my changes" and Claude will delegate to it. Read-only: it cannot edit, stage, or commit.
- **`karpathy`** — an engineering-discipline skill (think before coding, simplicity first, surgical changes, goal-driven execution). Loads automatically on non-trivial code tasks.

### `bolt-theme`

Run `/theme` and pick **Bolt Liquidity** — a dark base with Bolt's signature yellow (`#FFED2C`) for Claude's voice.
