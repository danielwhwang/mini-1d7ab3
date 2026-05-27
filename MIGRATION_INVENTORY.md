# Daisy → Mac Mini Migration Inventory

Generated: 2026-05-27 · pre-migration audit on MacBook

This is the complete list of what needs to move from the MacBook to the Mac Mini for Daisy to function end-to-end.

---

## 1. LaunchAgents (16 total + 1 hidden dependency)

All live in `~/Library/LaunchAgents/com.daisy.*.plist`. Combined size: trivial (~30 KB).

### Casing inconsistency (decision needed)

7 plists reference `~/daisy/` (lowercase). 9 plists reference `~/Daisy/` (uppercase). On macOS this works because the filesystem is case-insensitive. On a case-sensitive volume (Linux server, some backup disks), only one casing would work.

**Lowercase `~/daisy/` (7):** awardwatch, bigthree, groupme-saturday, healthcheck, kremmel-check, morning, wedprep

**Uppercase `~/Daisy/` (9):** authcheck, checkusage, gmailunsubscribe, guidebaseline, guidecheck-noon, onyourradar, telegramcheck, tripdebrief, weeklyreview

**Recommended fix:** standardize on lowercase `~/daisy/` (matches the actual on-disk directory, requires no file moves, just plist edits).

### Hidden dependency: `com.daisy.community-group`

Doesn't live in `~/daisy/` — points to `~/.scripts/community_group_message.sh`. Easy to miss in a "just rsync `~/daisy`" plan.

---

## 2. Daisy code & data (`~/daisy/`, 932 KB)

- All Python and shell scripts (`.py`, `.sh`)
- Config files (`settings.json`, `settings.local.json`, `config.toml`)
- Helper CLIs in `~/daisy/bin/`:
  - `daisy-imessage` — iMessage send/read
  - `daisy-gmail-attachment` — Gmail attachment download
- State files (preserve, but stale state is recoverable):
  - `.telegram_last_restart` — Telegram monitor state
  - `.telegram_probe_stuck` — Telegram monitor state
- Logs (`*.log`, `*_stdout.log`, `*_stderr.log`) — **safe to skip during migration**, will be regenerated
- Backup files (`*.bak-*`) — **safe to delete pre-migration** if you want a cleaner state

---

## 3. Secrets & tokens (critical)

All must be copied with strict file permissions preserved (`chmod 600`).

| File | Purpose | Migrates how |
|---|---|---|
| `~/daisy/.seats-aero-token` | seats.aero API key (award flight watcher) | Direct copy — API key, not OAuth |
| `~/daisy/.todoist-token` | Todoist API token | Direct copy — API key, not OAuth |
| `~/gmail-cleanup/credentials.json` | Gmail OAuth client (app credentials) | Direct copy |
| `~/gmail-cleanup/token.json` | Gmail OAuth refresh token | Direct copy — published-to-Production fix means it doesn't expire on a timer |
| `~/.config/google-calendar-mcp/credentials.json` | Calendar OAuth client | Direct copy |
| `~/.config/google-calendar-mcp/tokens.json` | Calendar OAuth refresh token | Direct copy — may require re-auth on Mini (TBD) |
| `~/.scripts/.env` | Environment vars (community group automation) | Direct copy |
| `~/.claude/channels/telegram/access.json` | Telegram MCP plugin auth | Direct copy — but bot token is binary cutover (can't run on both machines) |

**OAuth migration risk:** Google's OAuth doesn't formally machine-bind refresh tokens, so they *should* survive a copy. But there's a small chance Google's risk-detection sees the IP change and forces a re-auth. **Easy mitigation:** have the Mini's browser ready for a re-auth flow on migration day. Takes ~2 minutes if needed.

---

## 4. Hidden dependencies outside `~/daisy/` (easy to miss!)

| Path | What it is | Why it matters |
|---|---|---|
| `~/.scripts/` | community_group_message.sh + .env | `com.daisy.community-group` LaunchAgent depends on this |
| `~/gmail-cleanup/` | Gmail OAuth + bulk-trash Python script | Used by `daisy-gmail-attachment` and gmail unsubscribe automation |
| `~/.config/google-calendar-mcp/` | Calendar MCP OAuth | Calendar-aware automations (wedprep, briefings) |
| `~/.claude/channels/telegram/` | Telegram MCP plugin (2 MB) | Daisy's Telegram interface |
| `~/.claude/channels/groupme/` | GroupMe channel data | GroupMe host check automation |
| `~/.claude/plugins/data/telegram-claude-plugins-official/` | Telegram plugin install | Required by Telegram channel |
| `~/.claude/plugins/data/telegram-inline/` | Telegram inline plugin | Same |
| `~/.claude/settings.json` + `settings.local.json` | Claude Code config (29 permission allow rules, hooks, statusLine, enabled plugins) | All custom permissions and hooks |
| `~/.claude/CLAUDE.md` | User-global Claude instructions | About Daniel, output prefs, etc. |
| `~/.claude/projects/-Users-hwangda/memory/` | Auto-memory (this whole index + 30+ memory files) | Daisy's institutional knowledge |
| `~/pharmacy-schedule/` | Pharmacy scheduling rules | Future scheduling skill foundation |

---

## 5. Software to install fresh on Mini (no migration, install clean)

Already covered in [Day 1 checklist](https://danielwhwang.github.io/mini-1d7ab3/), but for reference:

- Xcode CLT, Homebrew
- `brew install tmux gh node python@3.12`
- Claude Code (official installer)
- Anthropic account sign-in (same as MacBook for Max subscription)
- gh auth login as `danielwhwang`

---

## 6. Post-migration verification (smoke tests)

Manually trigger each automation type to confirm it works:

| Trigger | Command | Expects |
|---|---|---|
| Morning briefing | `bash ~/daisy/morning_briefing.sh` | Telegram message arrives |
| Award watch | `bash ~/daisy/award_flight_watch.sh` | Telegram message arrives (or "no new finds") |
| Telegram health | `bash ~/daisy/telegram_check.sh` | Exit 0, log shows OK |
| iMessage send | `~/daisy/bin/daisy-imessage send <self> "test"` | Message arrives |
| Gmail attachment | `~/daisy/bin/daisy-gmail-attachment --help` | Help text prints (no auth call) |

Then `launchctl list | grep daisy` should show all 16 agents loaded, exit code 0.

---

## 7. Cutover constraint (the only true blocker)

**The Telegram bot token can only run in one place at a time.** Daisy's Telegram interface uses long-polling, so two machines pulling with the same token will conflict.

**Cutover sequence:**
1. On MacBook: `launchctl unload ~/Library/LaunchAgents/com.daisy.*.plist` (stops the agents)
2. On MacBook: kill Daisy's tmux session
3. On Mini: `launchctl load ~/Library/LaunchAgents/com.daisy.*.plist`
4. On Mini: start Daisy's tmux session
5. Test: send a Telegram message to Daisy → should respond from the Mini

If anything goes wrong: reverse the above. MacBook is back online in ~5 minutes.

---

## Summary of decisions still needed

- [ ] **Casing:** standardize on lowercase `~/daisy/`?
- [ ] **Tailscale:** include as default-yes in the migration plan, or keep optional?
- [ ] **Backup files:** delete `*.bak-*` files in `~/daisy/` before migration to keep the package clean?
