# claude-statusline

Configure your Claude Code statusline to show limits, directory and git info

> Fork of [nilbuild/claude-statusline](https://github.com/nilbuild/claude-statusline) by Kamran Ahmed.
> Upstream has been unmaintained since 2026-04, so this fork tracks newer Claude Code payload fields and fixes a few broken ones.

## Layout

```
Opus 5 │ ✍️ 42% │ perf (main*) │ ⏱ 1h15m │ ⚡ fast │ 🧠 off │ 1M │ ● xhigh
$1.23 │ +128/-47 │ api 183s │ PR #4321 APPROVED │ ⑂ feat-x │ Explanatory │ my-session │ v2.1.234

current ●●●○○○○○○○  37% ⟳  2:53pm
weekly  ●●●●●●○○○○  68% ⟳  aug 23, 6:00am
fable   ○○○○○○○○○○   7% ⟳  aug 25, 1:00am
extra   ●○○○○○○○○○  $1.20/$50.00 ⟳  sep 1
```

Everything marked *conditional* is omitted when the field is absent, so the line stays short in a plain session.

**Line 1 — where you are**

| Segment | Source | |
|---|---|---|
| `Opus 5` | `model.display_name` | |
| `✍️ 42%` | `context_window` | share of the context window in use |
| `⚡` (prefix) | parent process | shown when Claude Code runs with `--dangerously-skip-permissions` |
| `perf (main*)` | `cwd` + git | `*` means the working tree is dirty |
| `⏱ 1h15m` | `cost.total_duration_ms` | wall clock since the session started |
| `⚡ fast` | `fast_mode` | conditional |
| `🧠 off` | `thinking.enabled` | conditional, shown only when thinking is off |
| `1M` | `exceeds_200k_tokens` | conditional, 1M context window |
| `● xhigh` | `effort.level` | live value, follows `/effort` |

**Line 2 — what this session cost**

| Segment | Source | |
|---|---|---|
| `$1.23` | `cost.total_cost_usd` | |
| `+128/-47` | `cost.total_lines_added` / `_removed` | |
| `api 183s` | `cost.total_api_duration_ms` | time spent waiting on the API, retries included |
| `PR #4321 APPROVED` | `pr.number` / `pr.review_state` | conditional |
| `⑂ feat-x` | `worktree.name` | conditional |
| `Explanatory` | `output_style.name` | conditional, hidden while set to `default` |
| `my-session` | `session_name` | conditional, set by `/rename` |
| `v2.1.234` | `version` | Claude Code version |

**Usage block — what's left**

| Row | Source | |
|---|---|---|
| `current` | `rate_limits.five_hour` | rolling 5-hour window |
| `weekly` | `rate_limits.seven_day` | |
| per-model | usage API `limits[]`, `kind: "weekly_scoped"` | conditional, one row per scoped model — not available from stdin |
| `extra` | usage API `extra_usage` | conditional, shown when extra usage is enabled |

The 5-hour and weekly numbers come from the statusline payload on stdin. The per-model and extra-usage rows only exist in the OAuth usage API, which is polled separately and cached for 60s under `/tmp/claude/`.

## What's different from upstream

Fixes:

- `.session.start_time` doesn't exist in the statusline payload, so the `⏱` session timer never rendered — now uses `cost.total_duration_ms`
- `jq`'s `//` operator treats `false` as absent, so `.thinking.enabled // true` was always `true`
- The usage API cache was only refreshed when stdin carried no `rate_limits`. Since current Claude Code always sends them, the cache (and anything read from it) went permanently stale — the API is now always polled, with a 60s cache, while stdin stays authoritative for the 5h/weekly numbers
- `xhigh` / `max` effort levels fell through to the `default` display branch

Additions:

- Effort level is read from the live payload (`.effort.level`, reflects `/effort`) instead of `settings.json`
- Per-model weekly caps from the usage API's `limits[]` (`kind: "weekly_scoped"`) — e.g. Fable — which stdin's `rate_limits` does not carry
- A second line with session cost, lines added/removed, API time, PR number + review state, worktree, output style, session name, and CLI version
- `⚡ fast` (fast mode), `🧠 off` (thinking disabled) and `1M` (1M context) indicators

## Install

```bash
git clone git@github.com:noah4520/claude-statusline.git
cd claude-statusline
node bin/install.js
```

It backups your old status line if any and copies the status line script to `~/.claude/statusline.sh` and configures your Claude Code settings.

## Requirements

- [jq](https://jqlang.github.io/jq/) — for parsing JSON
- curl — for fetching rate limit data
- git — for branch info

On macOS:

```bash
brew install jq
```

## Uninstall

```bash
node bin/install.js --uninstall
```

If you had a previous statusline, it restores it from the backup. Otherwise it removes the script and cleans up your settings.

## License

MIT — original copyright (c) 2026 Kamran Ahmed, see [LICENSE](./LICENSE).
