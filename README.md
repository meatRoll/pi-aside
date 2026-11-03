# pi-aside

[English](README.md) | [简体中文](README.zh-CN.md)

A [Pi](https://pi.dev) extension: **read and ask questions alongside the main conversation**.

The main agent keeps doing its thing; you ask your questions in an overlay panel. The Q&A never enters the main session, never steers or queues the main agent, and never scrolls or reflows the transcript — close the panel and you are back on the same screen.

A good fit: the AI is writing a long document or running a task, and you want to ask "what does this term mean" or "why is this code written this way" while reading, without polluting the main context or pushing the output you are reading away.

## Features

- **Context isolation**: the side conversation has its own message array and calls
  `ctx.modelRegistry.streamSimple()` directly. It bypasses `session.prompt()` and never writes to the session file.
- **No interruption**: side requests are independent. You can ask while the main agent is streaming; nothing is injected into the current run.
- **Truly detachable**: `Esc` only hides the panel — answers keep streaming in the background. The footer shows progress and a notification appears when done. Multiple tabs can run in parallel (up to 9).
- **Reading position preserved**: the panel is an overlay, so the main transcript does not move. Reopening it returns you to where you were.
- **Automatic read-only snapshot**: opening a tab captures the last assistant reply and the user request before it as reference material for the side model.
- **Selection quote**: select text in the main transcript, then open Aside; the excerpt gets a tab of its own (an untouched tab is reused) and is sent with exactly one question, then dropped. The selection is consumed on open too, so reopening without selecting again does not repeat it; `Ctrl+Q` can exclude the quote from the next question.
- **Read-only tools**: when the snapshot is not enough, the side model can inspect the repo or system files with `read` / `grep` / `find` / `ls`, read git history and commit data with `git`, and fetch a URL with `web_fetch`. There is no `bash`, `edit`, or `write` tool, so it cannot modify files.
- **Optional write-back**: only `Ctrl+Y` writes a Q&A turn into the main conversation. That is the only step that enters the main context.
- **Independent model**: use a cheap/fast model for explanations without spending the main model's budget.
- **Bilingual UI**: follows the system language by default — Chinese only when the system locale is Chinese, English otherwise. The panel title shows the current mode and shortcut (`Language: Auto (EN) · Ctrl+L` / `语言：跟随系统（中文）· Ctrl+L`); press `Ctrl+L` (or click the label in fullscreen mode) to cycle through Auto / English / Chinese. The choice is saved to `aside.json`.

## Install

```bash
git clone https://github.com/meatRoll/pi-aside.git
cp pi-aside/aside.ts ~/.pi/agent/extensions/aside.ts
```

Then run `/reload` in pi (or restart pi).

Update:

```bash
cd pi-aside && git pull && cp aside.ts ~/.pi/agent/extensions/aside.ts
```

To try it without a global install, load it for one invocation:

```bash
pi --extension /path/to/pi-aside/aside.ts
```

Uninstall:

```bash
rm ~/.pi/agent/extensions/aside.ts
```

> Keep a single copy. If the same extension is loaded both as a user extension and
> from a package, Pi registers `aside:1` / `aside:2` and the older copy may win.
> Remove one of them, then run `/reload`.

## Usage

- `Ctrl+Shift+S` opens the panel
- `/aside what does this command do?` in the main editor opens it with a first question
- Inside the panel, the title shows the current mode and shortcut (`Language: Auto (EN) · Ctrl+L` / `Language: EN · Ctrl+L` / `语言：跟随系统（中文）· Ctrl+L` / `语言：中文 · Ctrl+L`); press `Ctrl+L` to cycle Auto / English / Chinese, or click the label in fullscreen mode
- `/aside-model [provider/model]` picks the side model

You can use it while the main agent is streaming.

To quote the conversation: select text in the main transcript (fullscreen mode, the default; Pi owns the selection), then press `Ctrl+Shift+S` or run `/aside`. The excerpt opens in a tab of its own and is sent with the first question in that tab, then dropped — it is never inherited by new tabs, and once sent it does not come back when you reopen the panel. The selection is consumed on open, so reopening without selecting anything keeps the current tab instead of quoting the old text again. `Ctrl+Q` excludes the quote from the next question if you do not want it.

| Key | Action |
|---|---|
| `Enter` | Send the question |
| `Shift+Enter` / `Ctrl+J` | New line |
| `Esc` | Close the panel (the request keeps running) |
| `Ctrl+C` | Cancel the active tab's generation |
| `Ctrl+N` | New tab |
| `Ctrl+X` | Close the current tab |
| `Shift+←` / `Shift+→` | Switch tabs |
| `PageUp` / `PageDown` | Scroll the side answer |
| `Alt+↑` / `Alt+↓` | Scroll one line |
| `Ctrl+End` | Jump to the latest output |
| `Ctrl+R` | Re-capture the main-conversation snapshot |
| `Ctrl+T` | Toggle the snapshot (when off, the side model only sees your question and any selected quote) |
| `Ctrl+Q` | Toggle the selected-text quote for the current tab |
| `Ctrl+G` | Toggle read-only tools for the current tab |
| `Ctrl+L` | Cycle interface language: Auto / English / Chinese (saved to `aside.json`) |
| `Ctrl+Y` | Write the latest Q&A turn into the main conversation (**only this enters the main context**) |

The mouse wheel also scrolls.

## Configuration

Optional file: `~/.pi/agent/aside.json`

```json
{
  "language": "auto",
  "model": "provider/model-id",
  "maxContextChars": 24000,
  "maxSelectionChars": 8000,
  "notifyOnComplete": true,
  "tools": true,
  "git": true,
  "network": true,
  "maxToolResultChars": 12000
}
```

- `language`: interface and system-prompt language — `"auto"` (default) follows the
  system locale: Chinese only when the system language is Chinese, English otherwise.
  `"en"` / `"zh"` force a language. Press `Ctrl+L` inside the panel to cycle
  Auto / English / Chinese; the choice is written back to this file.
- `model`: model used only for Aside Q&A; when omitted, the main conversation's
  current model is used. You can also pick one with `/aside-model`; the choice is
  written back to this file.
- `maxContextChars`: maximum characters kept in the snapshot (when exceeded, keep the
  first 65% and the last 35%, with the omitted middle marked).
- `maxSelectionChars`: maximum characters kept from the text selected in the main
  conversation (default 8000; the middle is omitted when exceeded).
- `maxAnswerTokens`: optional cap for a single side answer; when omitted the model's own
  output limit applies (same as the main conversation).
- `notifyOnComplete`: whether to show a notification when an answer completes while
  the panel is closed.
- `tools`: enable the read-only tools (`read`, `grep`, `find`, `ls`, `git`). Default `true`.
  Press `Ctrl+G` inside the panel to toggle them per tab.
- `git`: enable the read-only git tool. Default `true`; it is only registered when
  `tools` is on. It exposes `show`, `log`, `diff`, `blame`, `file`, `status`,
  `branches`, `tags`, and `ls`; there is no arbitrary git command execution.
- `network`: enable `web_fetch` for specific http(s) URLs. Default `true`; it is only
  registered when `tools` is on. There is no web search.
- `maxToolResultChars`: maximum characters kept from one tool result (default 12000).
  Built-in tools already truncate large outputs; this is a second token budget.

Aside behaves like the main conversation apart from the two deliberate limits: no
`write`/`edit`/`bash` tools, and a separate message array that never touches the
session file. The model follows the main session (override with `model`), side
requests use the main thinking level, and there is no tool-call budget: tools stay
declared for the whole turn, so the provider keeps parsing tool calls.

Compaction does not invent its own policy; it follows whatever manages the main
conversation:

- **pi-compactor owns compaction** (`~/.pi/agent/compactor/lifecycle.json` enabled): the
  `trigger` / `target` from `~/.pi/agent/compactor/settings.json` are used, including the
  repository layer and per-model rules, so the numbers match the main chat.
- **Otherwise**: pi's own `compaction` settings (`enabled`, `reserveTokens`,
  `keepRecentTokens`, `modelOverrides`).

The summary itself comes from pi's summarizer (`generateSummary`), using the same model
and thinking level as the main conversation. On a context-overflow error Aside folds once
and retries that round, like the main chat. The panel keeps showing the full history;
only the message array sent to the model is compacted.

## How isolation works

1. The side conversation has its own message array and calls
   `ctx.modelRegistry.streamSimple(model, ...)` directly. It **does not** go through
   `session.prompt()`, so no message is written to the session file.
2. The side agent runs its own tool loop with a closed set of tools: `read`, `grep`,
   `find`, `ls`, `git`, and optionally `web_fetch`. `bash`, `edit`, and `write` are
   never registered, so the model has no tool that can modify files. The `git` tool
   only accepts fixed read-only actions and fixed arguments; `web_fetch` only performs
   GET requests and refuses localhost/private addresses.
3. Opening the panel captures a **read-only snapshot**: the last assistant reply on
   the current branch plus the user request before it, and any text currently selected in
   the transcript. Both are sent as part of the system prompt and are copies; later
   changes in the main conversation are not synced automatically (use `Ctrl+R` for the
   snapshot; reselect and reopen for the quote).
4. The panel is an overlay: the main transcript does not scroll or re-layout, so your
   reading position stays where it was.

The only action that writes back to the main conversation is `Ctrl+Y`. It adds an
`aside` custom message via `pi.sendMessage()`, which becomes part of the main context
for subsequent model calls.

## Known limits

- There is no web search. `web_fetch` works only when you have a specific http(s) URL;
  searching needs a search API and is not implemented.
- Reads are not restricted to the project: the side model can read anything the pi
  process can read (same as the main agent). It still cannot write, but do not point it
  at secrets you would not send to your model provider.
- `grep` and `find` use `rg` / `fd`; like the main agent, pi may download them on first
  use if they are missing.
- The `git` tool is deliberately narrow: it can inspect commits, diffs, files, and
  history, but it cannot commit, checkout, stash, branch, tag, or run arbitrary git
  commands.
- Selection quote requires fullscreen mode, where Pi owns mouse selection. In regular
  mode the terminal owns the selection and Aside cannot read it. A quoted selection is
  cleared from the transcript when the panel opens and dropped from the tab after it is
  sent with one question; select again to quote again.
- A quoted selection is sent to the model provider, like the snapshot. Do not select
  secrets you would not send to your provider.
- Tool calls add tokens and latency. Toggle tools off with `Ctrl+G` when the snapshot is
  enough.
- Token usage for side requests appears in the panel status line but is not counted
  in the main session's usage statistics.
- Side history lives in memory. Switching sessions (`/new`, `/resume`, `/fork`) or
  exiting pi clears it.
- There is no separate side-context cap. The trigger is the main conversation's, so with
  pi-compactor it is that config's `trigger` (a percentage or token count per model), and
  without it, `contextWindow - reserveTokens` from the `compaction` settings. Retained
  text follows `target` / `keepRecentTokens`. pi-compactor's session-scoped overrides are
  in memory only and are not visible to Aside.
- TUI mode only. In RPC/print mode, `/aside` shows a hint instead.
- `Ctrl+Shift+S` may be captured by some terminals; `/aside` always opens the panel.

## Development

Type-checking can reuse the pi source repository's `tsconfig.json` and path mappings:

```bash
cd /path/to/pi
cat > tsconfig.aside.json <<'JSON'
{ "extends": "./tsconfig.json", "include": ["/path/to/pi-aside/aside.ts"] }
JSON
./node_modules/.bin/tsc -p tsconfig.aside.json --noEmit 2>&1 | grep aside
rm tsconfig.aside.json
```

To test a local build in pi:

```bash
cp aside.ts ~/.pi/agent/extensions/aside.ts
# then run /reload inside pi
```

> Note: pi's launcher refuses to `git pull` when the source repository has untracked
> files. Delete the temporary `tsconfig.aside.json` when you are done.

## License

MIT
