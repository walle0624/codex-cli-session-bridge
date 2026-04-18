# Codex CLI Session Bridge Usage

## Local anchors

- Codex CLI executable: `/Applications/Codex.app/Contents/Resources/codex`
- OpenClaw-managed VS Code session index: `/Users/walle/.openclaw/workspace/state/codex/CodeX_Session.json`
- OpenClaw-managed CLI session map: `/Users/walle/.openclaw/workspace/state/codex/OpenClaw_Codex_CLI_Session_Map.json`

## Data model

### File A: VS Code main sessions

`/Users/walle/.openclaw/workspace/state/codex/CodeX_Session.json`

Rules:
- only include `payload.source == "vscode"`
- fields: `session_id`, `thread_name`, `updated_at`
- sort by `updated_at` descending

### File B: OpenClaw CLI sessions

`/Users/walle/.openclaw/workspace/state/codex/OpenClaw_Codex_CLI_Session_Map.json`

Rules:
- store CLI-created or OpenClaw-adopted sessions separately
- preferred fields:
  - `session_id`
  - `title`
  - `source`
  - `cwd`
  - `created_at`
  - `last_seen_at`

## Preflight check

Before session resolution or CLI relay, check these paths first:
- `/Applications/Codex.app/Contents/Resources/codex`
- `/Users/walle/.openclaw/workspace/state/codex/CodeX_Session.json`
- `/Users/walle/.openclaw/workspace/state/codex/OpenClaw_Codex_CLI_Session_Map.json`

### If Codex CLI exists

Proceed with local session management.
If either JSON file is missing, create it locally rather than asking Codex to generate it.

### If Codex CLI is missing

Do not fake local access.
Report what is missing, then give the user a short prompt they can copy into Codex themselves.

## Decision flow

### A. Continue talking to Codex

Use this path when the user asks to:
- 调用 codex
- 问一下 codex
- 基于某个会话继续沟通
- 继续追问 codex
- 把我的问题发给 codex

Meaning:
- Send the user's message to Codex through the CLI
- Wait for Codex's actual reply
- Return Codex's reply to the user

Do not replace this with log reading.

### B. Read/summarize a Codex conversation

Use this path only when the user explicitly asks to:
- 读一下某个会话
- 整理我和 codex 的会话
- 看看会话里说了什么
- 读取会话内容

Meaning:
- Read local session artifacts
- Summarize faithfully
- Do not pretend this is a live Codex answer

## Session resolution rules

### Merged view

Build one merged list from both local files.

Title priority:
1. CLI map `title`
2. VS Code index `thread_name`
3. fallback placeholder

Deduplicate by `session_id` only.
Sort by the best available timestamp descending.

### If the user specified a session

1. Read both local files
2. Build the merged view
3. Match title first
4. Prefer exact match
5. If multiple fuzzy matches exist, present a short numbered choice list
6. Resolve to `session_id`

### If the user did not specify a session

Present the merged list as:

1. 会话A
2. 会话B
3. 会话C
...
N. 启用新会话

Rules:
- Use Chinese numbering style
- Show only names unless the user asks for ids
- Always append `启用新会话`

## New session workflow

If the user chooses `启用新会话`:

1. Ask for the new session name
2. Wait for confirmation
3. Start a new Codex conversation
4. Capture the returned `session id` directly from Codex CLI output
5. Write a new row into `OpenClaw_Codex_CLI_Session_Map.json`
6. Then continue relaying messages in that session

Suggested prompt:

`新会话准备叫什么？你给我个名字，我替你拉起来。`

Do not finalize the title without user confirmation.

Important rule:
- direct CLI output `session id` capture is the primary registration path
- filesystem scanning is fallback only when CLI output failed to expose the id

## CLI patterns

### Continue an existing session

Preferred pattern:

```bash
printf '<USER_PROMPT>\n' | /Applications/Codex.app/Contents/Resources/codex exec resume <SESSION_ID> --skip-git-repo-check --ephemeral -o /tmp/codex-last-message.txt -
```

### Create a new session

Use a non-interactive invocation that returns a new `session id`.
Then:
- extract `session id` from CLI output
- append the local CLI map row immediately
- store the confirmed user title in the local map

## Relay rules

For live relay tasks:
- Preserve the user's intent accurately
- Send the prompt to Codex, do not paraphrase away important constraints
- Return Codex's answer as the main result
- Add your own comments only if the user asks for analysis

## History-reading sources

Use these only when the user asked to read/summarize content:
- `/Users/walle/.openclaw/workspace/state/codex/CodeX_Session.json`
- `/Users/walle/.openclaw/workspace/state/codex/OpenClaw_Codex_CLI_Session_Map.json`
- `~/.codex/sessions/.../rollout-*.jsonl`
- Codex summary or memory files when clearly relevant

## Error handling

### Session not found

Say plainly that the requested session name was not found, then offer the numbered list again.

### CLI output missing session id

Use fallback registration logic only then.
Do not prefer filesystem scanning when the CLI already returned a usable id.

### Codex CLI command fails

Report the actual CLI error briefly.
Do not claim the message was relayed.

### Ambiguous match

Present only the candidate session names as a numbered list and ask the user to choose.

### Long Codex output

Prefer the final assistant message only.
If the response is still too long, return a concise note plus the most important portion, and offer to continue.

## What not to do

- Do not answer from historical logs when the user asked to continue talking to Codex
- Do not skip session selection when the user did not specify a session
- Do not create a new session without asking for the session name first
- Do not dump raw JSON by default
- Do not mix your own reconstructed answer with Codex's live reply without labeling the difference
- Do not rely on Codex `thread_name` generation for CLI-created session titles
