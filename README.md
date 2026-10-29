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
| `Ctrl+T` | Toggle the snapshot (when off, the side model only sees your question) |
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
  "maxAnswerTokens": 4000,
  "notifyOnComplete": true
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
- `maxAnswerTokens`: token limit for a single side answer.
- `notifyOnComplete`: whether to show a notification when an answer completes while
  the panel is closed.

Side requests always use `reasoning: "low"` to favor speed.

## How isolation works

1. The side conversation has its own message array and calls
   `ctx.modelRegistry.streamSimple(model, ...)` directly. It **does not** go through
   `session.prompt()`, so no message is written to the session file.
2. The request context declares no tools, so the model cannot modify files or run commands.
3. Opening a tab captures a **read-only snapshot**: the last assistant reply on the
   current branch plus the user request before it. The snapshot is sent as part of the
   system prompt and is a copy; later changes in the main conversation are not synced
   automatically (use `Ctrl+R`).
4. The panel is an overlay: the main transcript does not scroll or re-layout, so your
   reading position stays where it was.

The only action that writes back to the main conversation is `Ctrl+Y`. It adds an
`aside` custom message via `pi.sendMessage()`, which becomes part of the main context
for subsequent model calls.

## Known limits

- The snapshot contains text only — no tool output, file contents, or images. To
  explain a particular file, pasting the relevant excerpt into your question is more
  reliable.
- Token usage for side requests appears in the panel status line but is not counted
  in the main session's usage statistics.
- Side history lives in memory. Switching sessions (`/new`, `/resume`, `/fork`) or
  exiting pi clears it.
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
