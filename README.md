# navigating-hotwire-apps

An agent skill for driving, testing, and navigating **Turbo / Stimulus (Hotwire)** web applications using the **`firefox-devtools` MCP** server.

Designed for AI coding assistants such as **Claude Code**, Cursor, Codex, Antigravity, and other agents supporting the agent skills specification.

---

## Overview

Hotwire apps render interactive widgets as custom DOM trees driven by Stimulus rather than native form controls. Pages frequently host multiple instances of similar widgets simultaneously (e.g., multiple dropdowns, modal submit buttons overlapping with sidebar buttons).

This skill provides patterns and operational playbooks to eliminate common automation failures when driving Hotwire applications through browser DevTools MCP tools:

- **Locate once, act by reference:** Leverages `evaluate_script` with snapshot UIDs passed through `args: [{uid}]` to receive the actual resolved DOM element instead of guessing CSS selectors.
- **Handle `data-turbo-confirm`:** Prevents browser freezes by immediately accepting/dismissing native confirm dialogs on destructive actions.
- **Catch silent Stimulus errors:** Inspects Firefox console logs (`get_firefox_output`) for runtime exceptions thrown during `connect()` or action handlers.
- **Drive complex JS widgets:** Uses real keystrokes (`press_key`, `type_text`) on focused inputs for widgets like SlimSelect rather than synthetic click events.
- **Handle Turbo Drive cache & race conditions:** Navigates Turbo Drive cached snapshots and state transitions safely.

---

## Installation

### Method 1: Using Skills CLI (Recommended)

The [Skills CLI](https://skills.sh/) (`npx skills`) manages agent skills and handles symlinks and updates automatically across supported AI tools.

#### Install globally:

```bash
npx skills add carloskvasir/navigating-hotwire-apps -g
```

#### Install specifically for Claude Code:

```bash
npx skills add carloskvasir/navigating-hotwire-apps -g -a claude-code
```

#### Install for a single project:

Run inside your project directory:

```bash
npx skills add carloskvasir/navigating-hotwire-apps
```

---

### Updating

To keep the skill up to date:

```bash
# Update just this skill
npx skills update navigating-hotwire-apps

# Or update all global skills
npx skills update -g
```

---

### Method 2: Manual Installation (Claude Code)

If you prefer installing directly into Claude Code without `npx skills`:

#### User-wide (Global):

```bash
git clone https://github.com/carloskvasir/navigating-hotwire-apps.git ~/.claude/skills/navigating-hotwire-apps
```

To update:
```bash
git -C ~/.claude/skills/navigating-hotwire-apps pull
```

#### Project-level:

```bash
mkdir -p .claude/skills
git clone https://github.com/carloskvasir/navigating-hotwire-apps.git .claude/skills/navigating-hotwire-apps
```

Or as a git submodule:

```bash
git submodule add https://github.com/carloskvasir/navigating-hotwire-apps.git .claude/skills/navigating-hotwire-apps
```

---

## Prerequisites

This skill assumes your agent has access to the `firefox-devtools` MCP server tools:

- `take_snapshot` / `resolve_uid_to_selector`
- `click_by_uid` / `fill_by_uid` / `press_key` / `type_text`
- `evaluate_script`
- `accept_dialog` / `dismiss_dialog`
- `get_firefox_output`
- `get_page_text` / `screenshot_page` / `screenshot_by_uid`

---

## Quick Reference

| Symptom | Likely cause | Fix |
|---|---|---|
| Click hit the wrong button (e.g. logged user out) | Unscoped selector matched unrelated element | Resolve element via `take_snapshot` → `args: [{uid}]`, don't re-query by selector |
| Browser hangs after clicking delete/archive | `data-turbo-confirm` opened native dialog | Run `accept_dialog` (or `dismiss_dialog`) immediately after click |
| Widget silently not responding | Stimulus controller threw on connect | Inspect `get_firefox_output` grepped for controller name |
| `evaluate_script` returns null right after click | State did not survive call boundary | Merge open + read into single call with short delay |
| Form submission effect not in DB yet | Frame/stream still rendering or wrong element clicked | Short wait, verify target clicked by UID |
| Stale data after `navigate_history('back')` | Turbo Drive restored cached snapshot | Wait for refetch; re-`take_snapshot` (UIDs reset on navigation) |

---

## Author

**Carlos Kvasir**  
Website: [https://carloskvasir.dev](https://carloskvasir.dev)

---

## License

This project is licensed under the **Mozilla Public License Version 2.0 (MPL-2.0)**.  
See the [LICENSE](LICENSE) file for the full license text.
