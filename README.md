# Agent Meter

An [Omarchy](https://omarchy.org) bar widget showing how much of your AI coding
limit you've used: a progress meter with the percentage inside it.

![The Agent Meter in the Omarchy bar, reading 23% of the 5-hour session window](docs/meter.png)

Hovering names the window, the percentage, and when it resets — and keeps that
text current while the pointer stays put:

![Hover text reading "Claude Code · Session 23% · resets in 4h 53m"](docs/hover.png)

Built on Omarchy's own **Agents** widget. Clicking still opens the same full
panel: every limit with its reset countdown, tokens by day, and tokens by
model. The bar shows the number at a glance, so you don't have to open it.

> **GuideCoded** by [JasonAdamHD](https://github.com/JasonAdamHD) and Claude.
> Jason directed the idea and the look; Claude wrote the code in Claude Code.

## Features

- **Meter with the number inside.** The fill carries an inverted copy of the
  label, so the digits stay readable as the fill passes under them.
- **Shows your current session by default.** That's the rolling 5-hour window,
  the one that decides whether your next prompt goes through right now. You can
  switch it to the weekly limit, or to whichever window is fullest.
- **Keeps up while you code.** With an agent CLI open in a terminal it refreshes
  every minute. Otherwise it stays on the slow cycle, so an idle machine isn't
  polling anything. See [How often it updates](#how-often-it-updates).
- **Warns you.** At 90% the meter switches to your theme's urgent color.
- **Hover for detail**, e.g. `Claude Code · Session 12% · resets in 3h 40m`.
- **Follows your theme** and hot-reloads like any Omarchy shell plugin.
- **Works with every agent Omarchy tracks**: Claude Code, Codex, and Fireworks.
  Middle-click switches between them. Agents without rate limits, and vertical
  bars, fall back to the regular icon.

## Requirements

- Omarchy with the Quickshell-based shell and its built-in Agents widget, which
  provides the plugin system and the `omarchy-agent-usage-update` collectors
  this widget reads from. Tested on Omarchy 4.0.3.
- At least one agent that Omarchy can see: a signed-in Claude Code CLI, Codex,
  or a Fireworks account. See [Where the numbers come from](#where-the-numbers-come-from).

## Install

There is nothing to build or compile. The widget is plain QML that the Omarchy
shell loads directly.

```bash
omarchy plugin add https://github.com/JasonAdamHD/omarchy-agent-meter.git --enable
```

The meter replaces the built-in Agents icon and takes its spot on the bar. Your
existing Agents settings (refresh interval, sync, enabled providers) carry over.
If Agents wasn't on your bar, the meter goes into the right section.

### Manual install

If you'd rather clone it yourself, for example to hack on it:

```bash
git clone https://github.com/JasonAdamHD/omarchy-agent-meter.git \
  ~/.config/omarchy/plugins/jasonadamhd.agent-meter
omarchy-shell shell rescanPlugins
omarchy plugin enable jasonadamhd.agent-meter
```

The folder name must match the plugin id, `jasonadamhd.agent-meter`.

### Update

```bash
omarchy plugin update jasonadamhd.agent-meter
```

### Uninstall

Switching the built-in back on puts the Agents icon back in the meter's spot:

```bash
omarchy plugin enable omarchy.agents
omarchy plugin remove jasonadamhd.agent-meter
```

## Usage

| Action | What it does |
| --- | --- |
| Left-click | Open the usage panel |
| Right-click | Launch an agent |
| Middle-click | Switch to the next agent |
| Hover | Show the window, percentage, and reset countdown |

In the panel, `h`/`l` switch agents, `j`/`k` scroll, `r` or Enter refreshes,
and Esc closes.

The widget answers to the built-in's IPC target, so existing keybindings keep
working:

```bash
omarchy-shell omarchy.agents toggle    # also: open, close, refresh, next
```

## How often it updates

Two cycles run side by side:

- **While you're coding**, every `activeRefreshIntervalSec` (60 by default), the
  widget re-reads your limits through the collectors' limits-only path. That
  asks each service for its current windows and reuses the last transcript
  scan, so it costs a fraction of a second.
- **Otherwise**, every `refreshIntervalSec` (900 by default), it does the full
  collection: limits plus a fresh walk of your local transcripts for the
  per-day and per-model token stats.

"While you're coding" means an agent CLI is open in a terminal. The widget
checks once per fast interval by looking for a `claude` or `codex` process that
holds a pty, which costs about 10ms. Background daemons and spare workers,
which these CLIs leave running after you quit, deliberately don't count — only
a session you actually have open. Opening the panel always refreshes the limits
too, whatever the cycle is doing.

## Settings

Settings live in the widget's entry in `~/.config/omarchy/shell.json`. Change
them with `omarchy bar set`, which hot-reloads.

| Key | Default | What it does |
| --- | --- | --- |
| `barWindow` | `"session"` | Which limit the meter shows: `session` (the 5-hour window), `weekly`, or `binding` (whichever is fullest) |
| `activeRefreshIntervalSec` | `60` | How often to re-read limits while an agent CLI is open in a terminal |
| `refreshIntervalSec` | `900` | How often to run the full collection, in seconds |
| `providers` | all enabled | Turn individual agents on or off |
| `syncMode`, `syncDir` | `"Off"`, `""` | Merge usage from other machines through a synced folder |

```bash
omarchy bar set jasonadamhd.agent-meter barWindow weekly
omarchy bar set jasonadamhd.agent-meter activeRefreshIntervalSec 30 --json
```

Numbers need `--json`, otherwise they are saved as strings. To switch the fast
cycle off, set `activeRefreshIntervalSec` to the same value as
`refreshIntervalSec`. An agent with no limit under the chosen name, such as a
prepaid account, falls back to the fullest limit it does report. Per-agent and
sync options work exactly as in the built-in Agents widget; see its README in
`/usr/share/omarchy/shell/plugins/agents/README.md` on your system.

## Where the numbers come from

This widget only draws. It collects nothing and makes no network requests
itself. Omarchy's `omarchy-agent-usage-update` does the collecting: it writes
one JSON record per agent to `~/.local/state/omarchy/agents/usage/`, and the
widget watches those files.

- **Claude Code**: session and weekly limits come from Anthropic's usage
  endpoint via your signed-in CLI. Local token stats come from
  `~/.claude/projects`.
- **Codex**: limits come from the Codex app server. Stats come from local
  session files.
- **Fireworks**: an estimated prepaid balance from the Fireworks billing API.

## Development

Edit the QML in your plugin folder. The shell reloads it on save. To check the
manifest before publishing a change:

```bash
omarchy plugin validate ~/.config/omarchy/plugins/jasonadamhd.agent-meter
```

Shell logs, including QML errors, are available with:

```bash
quickshell log -n -p /usr/share/omarchy/shell -t 40    # add -f to follow
```

A plugin rescan can keep drawing the widget instance that is already running,
which makes a code or settings change look ignored. `omarchy restart shell`
applies it for certain.

| File | Role |
| --- | --- |
| `Panel.qml` | Bar meter and popup panel |
| `Main.qml` | Finds and watches usage records, runs refreshes, handles sync |
| `Agent.qml` | Watches a single usage record file |
| `manifest.json` | Plugin id, entry point, defaults, and settings schema |

## License

Agent Meter is free software under the **GNU General Public License v3.0 or
later** ([LICENSE](LICENSE)). You can use, copy, modify, and share it. If you
distribute a modified version, you must release its source under the same
license.

It is based on the Agents plugin from [Omarchy](https://github.com/basecamp/omarchy),
Copyright (c) David Heinemeier Hansson, used under the MIT License. That
notice is kept in [LICENSE-OMARCHY](LICENSE-OMARCHY).

The Claude, Codex, and Fireworks marks in `assets/` belong to their respective
owners and are included only to identify each service.
