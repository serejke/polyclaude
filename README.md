# polyclaude

A terminal launcher for Claude Code. One screen shows the sessions that run now, the directories you use, and the sessions you can resume.

```
  Start   Resume   Settings                          Tab:switch  Enter:attach/launch  q:quit

  Running
 > ● busy     fix-billing-retry   ~/code/acme-api   bg  ·  up 12m
   ● waiting  dark-mode           ~/code/acme-web   bg  ·  up 2h
   ● idle     notes               ~/code/notes      terminal  ·  up 1d

  New session in
   ~/code/acme-api   current dir  ·  1 running  ·  12 sessions  ·  5m ago
   ~/code/acme-web   1 running  ·  47 sessions  ·  2h ago
   ~/code/sandbox    3 sessions  ·  4d ago
```

## Features

- **Sessions that continue.** polyclaude starts each session in the background. The session continues when your terminal or your ssh connection closes.
- **One entry point.** Attach to a running session, start a new session, or resume an old session from the same screen.
- **Live status.** Each running session shows its status: `busy`, `waiting` or `idle`.
- **Fast start.** The screen opens in approximately 0.2 s, also with gigabytes of session history.
- **Mouse and keyboard.** Click, scroll, or use the arrow keys and `j` `k`.
- **No dependencies.** polyclaude is one Python file. It uses only the standard library.

## Install

You need [uv](https://docs.astral.sh/uv/) and Claude Code.

```sh
uv tool install git+https://github.com/serejke/polyclaude
```

To install a specific release, add the tag:

```sh
uv tool install git+https://github.com/serejke/polyclaude@v0.7.1
```

To update, run the same command with `--reinstall`. uv does not update a git source without this flag.

## Use

```sh
polyclaude
```

Select an item and push Enter:

| Item                         | Enter                                                            |
| ---------------------------- | ---------------------------------------------------------------- |
| A running `bg` session       | Attaches to the session.                                         |
| A running `terminal` session | Shows its terminal. See [Terminal sessions](#terminal-sessions). |
| A directory                  | Starts a new session in the directory.                           |
| An old session (Resume tab)  | Opens the chat. Enter again resumes the session.                 |

### Options

```sh
polyclaude 50           # show 50 sessions on the Resume tab (default: 20)
polyclaude --fg         # start claude in the foreground for this run
polyclaude --unsafe     # add --dangerously-skip-permissions for this run
polyclaude -b my-claude # use a different claude binary
```

You can also set the binary with the `CLAUDE_BINARY` environment variable.

## Tabs

### Start

The Start tab has two sections:

- **Running** shows the sessions that run now. The sessions in the current directory are first.
- **New session in** shows the directories from your session history. The current directory is first.

### Resume

The Resume tab shows your recent sessions. The session with the most recent activity is first.

```
  ── In this directory ──
 > 5m ago    fix-billing-retry   ● busy (bg)
             ~/code/acme-api  24m  you:18  claude:40  tools:42  agents:3
             first: fix the flaky retry test in the billing client
              last: run the suite again
             reply: All 212 tests pass.
```

Each row shows:

- The time of the last activity. Up to 7 days, the time is relative (`5m ago`, `2d ago`). After 7 days, the time is a date.
- The session name. If the session has no name, the row shows the path.
- The duration, and the number of messages and tool calls.
- Your first and last messages, and the last reply of the agent.

Push `/` to search the full text of the sessions. Push Enter or click a session to open the full chat. The chat opens at the end, so you see the latest messages first.

### Settings

| Setting                      | Default | Effect                                             |
| ---------------------------- | ------- | -------------------------------------------------- |
| Dangerously skip permissions | off     | Adds `--dangerously-skip-permissions` to claude.   |
| Launch in background         | on      | Starts sessions with `claude --bg`, then attaches. |

polyclaude saves each change immediately to `~/.config/polyclaude/config.json`.

## Background sessions

By default, polyclaude does not run claude in your terminal. It runs `claude --bg`, which gives the session to the Claude background service. Then it runs `claude attach`, which shows the session in your terminal.

Thus your terminal only shows the session. The terminal does not own the session. When the terminal closes, the session continues. To see the session again, run `polyclaude` and select the session in **Running**.

In an attached session, push `←` to go to the agent view. Push `Ctrl+Z` to go back to the shell.

### Workspace trust

`claude --bg` cannot show the trust dialog, so it does not start in a directory that Claude Code does not trust yet. Inside a git repository, a trusted parent directory does not count: the repository needs its own trust. Before a background launch, polyclaude reads the trust records in `~/.claude.json`. If the directory is not trusted, polyclaude starts claude in the foreground instead, and claude shows its own trust dialog. To move that session to the background, type `/bg` in it. The next launch in the directory is a background launch again. polyclaude never writes to Claude Code's config.

### Terminal sessions

A session that you start with plain `claude` is a `terminal` session. Its terminal owns the session. You cannot attach to it from a different terminal.

To change a terminal session into a background session, type `/bg` in that session.

## Keys

| Key               | Action                             |
| ----------------- | ---------------------------------- |
| `Tab`             | Go to the next tab.                |
| `↑` `↓` / `j` `k` | Move the cursor.                   |
| `Enter`           | Attach, start, open or resume.     |
| `Space`           | Change a setting.                  |
| `/`               | Search the sessions (Resume tab).  |
| `Esc` / `←`       | Go back, or clear the search.      |
| `g` / `G`         | Go to the top or bottom of a chat. |
| `q` / `Ctrl+C`    | Quit.                              |

| Mouse          | Action                                   |
| -------------- | ---------------------------------------- |
| Click a tab    | Open the tab.                            |
| Click (Start)  | Select. Double-click to attach or start. |
| Click (Resume) | Open the chat.                           |
| Wheel          | Move the cursor, or scroll the chat.     |

polyclaude uses the mouse. To select text, hold `Shift` (or `Option`) in your terminal.

## How it works

Claude Code keeps each session as a JSONL file in `~/.claude/projects/`. Some files are larger than 100 MB. Thus polyclaude reads as little as possible before it shows the screen:

1. It reads the directory of each session one time, and keeps it in `~/.cache/polyclaude/cwds.json`.
2. It reads only the start and the last 256 KB of each recent session.
3. It counts the messages and tool calls on a background thread.
4. It runs `claude agents --json` every 2 s to get the running sessions.
5. It reads the full text of a session only when you search or open the chat.

## License

MIT
