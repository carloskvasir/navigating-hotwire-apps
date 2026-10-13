---
name: navigating-hotwire-apps
description: Use when driving a Turbo/Stimulus (Hotwire) web app through firefox-devtools MCP tools — custom JS-rendered widgets (SlimSelect-style selects), Turbo Frame/Stream modals, data-turbo-confirm delete buttons, a click that seems to hang, or evaluate_script/selector-based scoping producing wrong or flaky results.
license: MPL-2.0
author: Carlos Kvasir
homepage: https://carloskvasir.dev
---

# Navigating Hotwire Apps with firefox-devtools MCP

## Overview

Hotwire apps render interactive widgets as custom DOM trees driven by Stimulus, not native form controls, and a page commonly has several instances of the same widget at once (multiple dropdowns, a modal's submit button plus the page's own). The biggest lever available is one most people skip: `evaluate_script` takes snapshot UIDs as `args` and hands your function the *actual resolved element* — this replaces almost all selector-guessing and `document.querySelector` scoping mistakes.

## Locate once, act by reference — the core pattern

1. `take_snapshot` (scoped with `selector` to the relevant container if the page is busy) to get a UID for the element you mean.
2. Either act on it directly (`click_by_uid`, `fill_by_uid`, `press_key` with `uid`), or pass it into `evaluate_script` via `args: [{uid}]` — the function receives the real element, not a string to re-find:

```js
// function receives the resolved element for uid "e42" as its first argument
evaluate_script({
  function: "(ssMain) => ssMain.click()",
  args: [{uid: "e42"}]
})
```

This is why a bare `document.querySelectorAll('button[type=submit]')` is risky on a page with a modal AND a sidebar "sign out" button — it grabs whichever comes first in DOM order (this has logged out the test session mid-flow). Resolving by UID sidesteps that category of mistake entirely. When you need an actual CSS selector string instead (e.g. to scope a second `take_snapshot`), `resolve_uid_to_selector` converts a UID without you inventing one by hand.

## Rails/Turbo confirm dialogs will hang you if ignored

`data-turbo-confirm` (and legacy `data-confirm`) on a destructive link/button — virtually every delete button in a Rails app — triggers a native `confirm()` before Turbo proceeds. A plain `click_by_uid` leaves the browser blocked on it. Call `accept_dialog` (or `dismiss_dialog` to cancel) immediately after that click.

## Read the log instead of guessing

Stimulus logs a console error when a controller throws during `connect()` or an action, and that failure doesn't always show up visually — the widget just silently doesn't work. Before concluding "it isn't wired up," check `get_firefox_output` (`grep` for the controller identifier) — faster and more conclusive than introspecting `window.Stimulus` by hand, and it catches errors that broke rendering without any visible symptom.

## Cheap verification first, screenshots for visual questions

`get_page_text` returns the page's visible text in one call — use it to confirm a flash message, a turbo_stream update, or a "no results" string landed, before reaching for a screenshot. Save `screenshot_page`/`screenshot_by_uid` for genuinely visual questions (did dark mode apply, is an icon actually rendered) — those are ground truth text can't give you.

## Drive JS widgets with real keys, not just synthetic clicks

Libraries like SlimSelect wire ArrowUp/ArrowDown/Enter/Escape on their search input. Once a dropdown is open, `press_key` (ArrowDown, ArrowDown, Enter) on the focused input is often more robust than locating and clicking a specific option node in the rendered list. Use `type_text` instead of `fill_by_uid` for inputs that specifically listen for keydown/keyup (autocomplete fields, rich text editors) — `fill_by_uid` sets `.value` and fires one `input` event, which some components don't react to the same way as real keystrokes.

## Other Hotwire-adjacent situations

- **Transient UI state:** bundle open + read + type into ONE `evaluate_script` call when steps depend on a toggle or an outside-click handler — splitting them across calls is flaky because the state can change between calls. Add short `await new Promise(r => setTimeout(r, 300))` pauses inside the single call instead.
- **File uploads:** `upload_file_by_uid` for Active Storage-backed file fields.
- **Exports (CSV/PDF):** `set_download_behavior({behavior: 'allowed', downloadFolder})` then `list_downloads` to verify an export landed, instead of fighting the native save dialog.
- **New tabs / external redirects:** `target="_blank"` links, OAuth popups, payment redirects (e.g. Stripe Checkout) open a new tab — use `list_pages` / `select_page` / `new_page` to follow them.
- **Turbo Drive cache:** `navigate_history('back')` can restore a *cached* (stale) snapshot of the page before Turbo's background refetch completes — don't assert against the page immediately after a back-navigation, and always re-`take_snapshot` since UIDs go stale on any navigation, history included.
- **Drag-to-reorder lists** (acts_as_list-style sortable UIs): `drag_by_uid_to_uid` with two snapshot UIDs.
- **Context budget:** `evaluate_script`, `take_snapshot`, `get_page_text`, and the screenshot tools all accept `saveTo` to write large results to a file (with `preview` for a quick inline peek) instead of flooding the conversation — use it for big DOM dumps or long page text.
- **Widgets the snapshot doesn't represent well** (SlimSelect's generated `.ss-main` / `.ss-search` / `.ss-list` / `.ss-addable`): scope a `take_snapshot` with `selector` to the field's container to get real UIDs for these generated nodes.

## Authenticated flows

1. Create a throwaway user/account via `bin/rails runner`, mirroring the app's own test factory/helper (skip email confirmation, fill whatever scaffolding the app's models require).
2. Log in through the real form — snapshot it, `fill_form_by_uid`, `click_by_uid` submit — rather than faking a session, so CSRF and the real auth stack get exercised too.
3. Clean up afterward in dependency order: destroy records created during the test before the account/user, or a foreign key will block it.

## Quick reference

| Symptom | Likely cause | Fix |
|---|---|---|
| Click hit the wrong button (e.g. logged the user out) | Unscoped selector matched an unrelated element elsewhere on the page | Resolve the exact element via `take_snapshot` → `args: [{uid}]`, don't re-query by selector |
| Browser seems stuck after clicking delete/archive/etc. | `data-turbo-confirm`/`data-confirm` opened a native dialog | `accept_dialog` (or `dismiss_dialog`) right after the click |
| A widget silently "isn't wired up" | A Stimulus controller threw on connect — check before assuming | `get_firefox_output` grepped for the controller name |
| `evaluate_script` says an element is null right after a click that should produce it | State didn't survive the call boundary (toggle closed, focus moved) | Merge open + read into one call with a short delay |
| A just-submitted form's effect isn't in the DB yet | Frame/stream hadn't finished rendering, or the click hit the wrong element | Re-check after a short wait; confirm the right element was clicked via UID, not guessed selector |
| Page looks right after `navigate_history('back')` but data is stale | Turbo Drive restored its cached snapshot before refetching | Wait, then re-verify; re-`take_snapshot` since UIDs reset on any navigation |

## Example: resolving and driving a SlimSelect field by UID

```js
// 1. take_snapshot scoped to the field's container to get a uid for .ss-main, e.g. "e42"
// 2. open it and read the generated list by passing that uid through args:
evaluate_script({
  function: "async (ssMain) => { ssMain.click(); await new Promise(r => setTimeout(r, 300)); return ssMain.parentElement.querySelector('.ss-search input') }",
  args: [{uid: "e42"}]
})
// 3. take_snapshot scoped to '.ss-search' to get a uid for the real input, then:
//    fill_by_uid(that uid, "typed value")
//    press_key("ArrowDown"); press_key("Enter")   // drive the result list by keyboard
```

Confirm with `get_page_text` or a container-scoped screenshot before deciding an option (like an "add new" affordance) is or isn't present — don't rely solely on script introspection of framework internals.
