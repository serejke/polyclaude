# polyclaude

> The entry point to Claude Code — pick any directory, resume any session.

polyclaude shows your running Claude Code sessions, starts new sessions, and resumes old sessions. It has three tabs.

### Start

```
┌─[ Start ]─[ Resume ]─[ Settings ]─────────── Enter:attach/launch  q:quit ─┐
│  Running                                                                  │
│ > ● busy     fix-billing-retry   ~/code/acme-api   bg  ·  up 12m          │
│   ● waiting  dark-mode           ~/code/acme-web   bg  ·  up 2h           │
│   ● idle     notes               ~/code/notes      terminal  ·  up 1d     │
│                                                                           │
│  New session in                                                           │
│   ~/code/acme-api      current dir  ·  1 running  ·  12 sessions  ·  5m   │
│   ~/code/acme-web      1 running  ·  47 sessions  ·  2h ago               │
│   ~/code/sandbox       3 sessions  ·  4d ago                              │
└───────────────────────────────────────────────────────────────────────────┘
```

Enter on a running `bg` session attaches to it. Enter on a directory starts a new session there.

### Resume

```
┌─[ Start ]─[ Resume ]─[ Settings ]────────── /:search  Enter:expand  q ─┐
│  ── In this directory ──                                               │
│ > ~/code/acme-api  2026-11-26 09:14:02  ● busy (bg)                    │
│      24m  you:18  claude:40  tools:42  agents:3                        │
│      first: fix flaky retry test in billing client                     │
│       last: run the suite again                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Settings

- **Dangerously skip permissions.** Adds `--dangerously-skip-permissions`.
- **Launch in background.** Starts sessions with `claude --bg` and attaches to them. This is the default.

## Background sessions

By default polyclaude starts each session with `claude --bg` and then runs `claude attach`. The Claude background service owns the session, not your terminal. Thus the session continues when the terminal or the ssh connection closes. To open it again, run `polyclaude` and select it in *Running*.

A session that you started with plain `claude` is a `terminal` session. You cannot attach to it from a different terminal. Type `/bg` in that session to move it to the background.

## Install

```sh
uv tool install git+https://github.com/serejke/polyclaude
```

Or:

```sh
curl -sSL https://raw.githubusercontent.com/serejke/polyclaude/main/install.sh | bash
```

Requires [uv](https://docs.astral.sh/uv/). Zero runtime deps — pure Python stdlib.

## Use

```sh
polyclaude              # open the UI
polyclaude 50           # show 50 sessions on the Resume tab (default 20)
polyclaude --fg         # start claude in the foreground for this run
polyclaude --unsafe     # add --dangerously-skip-permissions for this run
polyclaude -b my-claude # use a different claude binary
```

## Keys

| Key               | Action                                 |
| ----------------- | -------------------------------------- |
| `Tab`             | go to the next tab                     |
| `j` `k` / `↑` `↓` | move the cursor                        |
| `Enter`           | attach / start / expand / resume       |
| `Space`           | toggle a setting                       |
| `/`               | search the session text (Resume)       |
| `Esc` / `←`       | go back / clear the search             |
| `g` `G`           | top / bottom (expanded session)        |
| `q` / `^C` / `^D` | quit                                   |

## Config

The config file is `~/.config/polyclaude/config.json`. The Settings tab saves each change immediately.

```json
{
  "skip_permissions": false,
  "background": true
}
```

## How it works

Claude Code keeps each session as a JSONL file in `~/.claude/projects/<path>/<uuid>.jsonl`. polyclaude:

1. Reads the cwd of each session file one time and caches it in `~/.cache/polyclaude/cwds.json`.
2. Reads only the start and the last 256 KB of the recent sessions, so the screen opens fast.
3. Counts the messages and tools on a background thread.
4. Runs `claude agents --json` every 2 s to show the running sessions.
5. Starts or attaches to claude with `chdir` + `execvp`.

## License

MIT
