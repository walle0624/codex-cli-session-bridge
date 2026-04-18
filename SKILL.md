---
name: codex-cli-session-bridge
description: Use when the user asks to call, continue, resume, query, or read Codex CLI sessions from OpenClaw. Handles merged session selection from OpenClaw-managed Codex indexes, distinguishes between continuing a live Codex conversation versus reading historical session content, and enforces the user's workflow for new sessions, named sessions, and relaying Codex replies back to the user.
---

# Codex CLI Session Bridge

Keep these local anchors in mind:
- Codex CLI executable: `/Applications/Codex.app/Contents/Resources/codex`
- OpenClaw-managed VS Code session index: `/Users/walle/.openclaw/workspace/state/codex/CodeX_Session.json`
- OpenClaw-managed CLI session map: `/Users/walle/.openclaw/workspace/state/codex/OpenClaw_Codex_CLI_Session_Map.json`

## Preflight check

Before any Codex-specific workflow, check whether these local files exist:
- `/Applications/Codex.app/Contents/Resources/codex`
- `/Users/walle/.openclaw/workspace/state/codex/CodeX_Session.json`
- `/Users/walle/.openclaw/workspace/state/codex/OpenClaw_Codex_CLI_Session_Map.json`

If Codex CLI exists, but one or both OpenClaw-managed JSON files are missing, create or refresh them locally instead of asking Codex to do it.

If Codex CLI itself is missing, do not pretend the bridge is available. Tell the user which prerequisite is missing and switch into onboarding guidance mode.

### Onboarding guidance for machines that do not have Codex set up

If the machine has not installed Codex CLI yet, guide the user in this order:
1. install Codex CLI or confirm the local executable path
2. verify `codex --help` works
3. create one test session manually in Codex
4. if needed, create the OpenClaw local state directory `state/codex/`
5. only after that, start using the bridge skill

If local Codex session export or OpenClaw-managed JSON files do not exist yet, do not block. Initialize the local files yourself when possible.

If you truly cannot operate locally, give the user a short copyable prompt they can send to Codex manually, then ask them to send the result back.

## Local source-of-truth split

Maintain two local data files:

1. **VS Code main-session index**
   - file: `/Users/walle/.openclaw/workspace/state/codex/CodeX_Session.json`
   - contains only Codex main sessions where `payload.source == "vscode"`
   - fields: `session_id`, `thread_name`, `updated_at`
   - sorted by `updated_at` descending

2. **CLI session map**
   - file: `/Users/walle/.openclaw/workspace/state/codex/OpenClaw_Codex_CLI_Session_Map.json`
   - contains sessions created or adopted by OpenClaw through Codex CLI
   - preferred fields: `session_id`, `title`, `source`, `cwd`, `created_at`, `last_seen_at`

Do not mix CLI-created sessions back into the VS Code main-session index.

## Decide the interaction mode

First decide whether the user wants one of these two modes:

1. **Continue talking to Codex**
   Treat phrases like “调用 codex”, “问下 codex”, “继续跟 codex 沟通”, “基于某个会话继续追问” as a live relay task.
   In this mode, send a prompt to Codex and return Codex's reply to the user.
   Do **not** answer by only reading historical logs, unless the user explicitly asked to read session content.

2. **Read session content**
   Treat phrases like “读一下我和 codex 的某个会话”, “整理会话内容”, “读取会话记录” as a history-reading task.
   In this mode, inspect local session data and summarize what was said.

If ambiguous, ask one short clarifying question.

## Session list construction

When you need to present selectable sessions, build one merged view from both local files.

### Merge rules

- Read `/Users/walle/.openclaw/workspace/state/codex/CodeX_Session.json`
- Read `/Users/walle/.openclaw/workspace/state/codex/OpenClaw_Codex_CLI_Session_Map.json`
- Merge by `session_id` only
- Title priority:
  1. OpenClaw CLI map `title`
  2. VS Code index `thread_name`
  3. fallback placeholder such as `未命名会话`
- Sort by the best available timestamp descending
- Always add one extra option at the end:
  `N. 启用新会话`

Do not dump raw JSON unless the user explicitly asks for it.

## When the user asks to call Codex

### Step 1: Check whether the user specified a session

Try to determine whether the user explicitly named a session/thread to continue.

If a session is specified, resolve it against the merged local view first.
Prefer exact title match before fuzzy matching.

If no session is specified, present the merged selectable list as a numbered Chinese list.

### Step 2: Handle “启用新会话”

If the user chooses `启用新会话`, ask one follow-up question to confirm the new session name.
Use a direct prompt such as:

`新会话准备叫什么？你给我一个名字，我再替你拉起它。`

Do not invent the final session title without confirmation.

When the user gives the new name, create a new Codex CLI session and register it locally like this:
- start the new conversation
- capture the returned `session id` directly from Codex CLI output
- immediately write a new row into `/Users/walle/.openclaw/workspace/state/codex/OpenClaw_Codex_CLI_Session_Map.json`
- do not depend on Codex internal `thread_name` generation for this path

This is the primary flow. Scanning local session files is only a fallback when the CLI output did not expose the `session id`.

### Step 3: Continue the live Codex conversation

For “继续沟通” tasks, use Codex CLI to send the user's message into the chosen `session_id` and wait for Codex's answer.
Preferred pattern:
- Resume an existing session with `codex exec resume <SESSION_ID> ...`
- For a new session, create the session, capture `session id`, register it locally, then use that id for future continuation

Return Codex's reply back to the user as Codex's answer.
Make it clear only when helpful that this is Codex's response, but do not add your own interpretation unless the user asks.

## When the user explicitly asks to read a Codex session

Only in this case, inspect local session data instead of relaying a live answer.
Recommended sources:
- `/Users/walle/.openclaw/workspace/state/codex/CodeX_Session.json`
- `/Users/walle/.openclaw/workspace/state/codex/OpenClaw_Codex_CLI_Session_Map.json`
- `~/.codex/sessions/.../rollout-*.jsonl`
- related Codex memory/summary files when needed

Summarize the requested conversation faithfully and distinguish clearly between:
- what the user said
- what Codex replied
- what task or automation was set up

## Output rules

- Default to concise replies in Chinese.
- If the user asked to continue a Codex conversation, the result should be the actual Codex reply, not your reconstructed answer.
- If you used logs instead of live relay, only do so because the user explicitly asked to read/summarize the session.
- If session selection is missing, stop and ask the user to choose from the merged numbered list plus `启用新会话`.
- If a requested session name does not exist in the merged local view, say so plainly and offer the numbered list again.

## Reliability notes

- Treat OpenClaw-managed local files as the primary session-selection source.
- Keep the VS Code index limited to `payload.source == "vscode"`.
- Keep CLI-created sessions in the separate local map.
- Prefer direct `session id` capture from Codex CLI output over filesystem rescans.
- If a live Codex call fails, report the actual CLI error briefly and do not pretend the relay succeeded.
- If a `session_id` disappears from Codex data later, keep the local row but degrade its display instead of hard-deleting it immediately.
