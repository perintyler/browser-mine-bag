---
name: web-browsing
description: Browse and interact with web pages. Use when asked to open a URL, read a page, fill a form, click through a site, or check how something renders in a real browser.
---

# Web Browsing

Barry browses in one of three modes. A session gets exactly one, chosen by its trait:

| Trait | Pack | Browser | Logged in as you? |
|---|---|---|---|
| `browser` | `browser` | Clean Chromium, headless | No |
| `browser-mine` | `browser-mine` | Your real, already-running Chrome | **Yes** |

If you need a site the user is logged into, you need `browser-mine`. The
`browser` mode starts from a blank profile with no cookies, so a logged-in page
will render logged-out. Sessions can't switch modes mid-flight — the user has to
start a session with the right trait.

## The loop: snapshot → ref → act

Never guess CSS selectors. Take a snapshot, read the element reference out of
it, and pass that reference back to the action tool. A snapshot is an
accessibility tree — it is both cheaper in tokens and far more reliable than a
screenshot, because it gives you stable handles to act on.

**Screenshots are for humans.** Take one when the user wants to *see* something,
or to judge visual layout. You cannot click based on a screenshot.

**Refs go stale.** Any navigation, click, or DOM update can invalidate every
reference from the previous snapshot. Re-snapshot after anything that changes
the page, rather than reusing older refs.

## Tool names differ by mode — check before you call

The `browser` mode and the real-Chrome mode are different upstream servers with
**different tool vocabularies**. Using the wrong one is the single
most common failure. When unsure, list your available tools and match the
prefix.

### `browser` (@playwright/mcp)

Tools are prefixed `browser_`. Element references go in a **`target`** field,
alongside a human-readable `element` description.

| Task | Tool |
|---|---|
| Go to a URL | `browser_navigate` (`url`) |
| Read the page | `browser_snapshot` |
| Find one thing on a big page | `browser_find` (`text` or `regex`) |
| Click | `browser_click` (`target`, `element`) |
| Type | `browser_type` (`target`, `text`, `submit`) |
| Fill a whole form at once | `browser_fill_form` (`fields[]`) |
| Dropdown | `browser_select_option` (`target`, `values`) |
| Wait | `browser_wait_for` (`text`, `textGone`, or `time`) |
| Screenshot | `browser_take_screenshot` |
| Tabs | `browser_tabs` (`action`: list/new/close/select) |
| Back | `browser_navigate_back` |
| Console / network | `browser_console_messages`, `browser_network_requests` |
| Finish | `browser_close` |

Prefer `browser_find` over `browser_snapshot` on large pages — it returns just
the matching nodes with their refs, instead of the whole tree.

Prefer `browser_fill_form` over field-by-field typing on multi-field forms: one
call, and it handles checkboxes, radios and comboboxes by `type`.

`browser_run_code_unsafe` and `browser_evaluate` execute arbitrary JavaScript.
Reach for them only when no structured tool covers the task.

### `browser-mine` (chrome-devtools-mcp)

**Different names, and element references are `uid`, not `target`.**

| Task | Tool |
|---|---|
| Go to a URL | `navigate_page` (`url`, `type`) |
| Read the page | `take_snapshot` |
| Click | `click` (`uid`) |
| Type into a field | `fill` (`uid`, `value`) |
| Type at the keyboard | `type_text` (`text`) |
| Fill a form | `fill_form` (`elements[]`) |
| Wait | `wait_for` (`text`) |
| Screenshot | `take_screenshot` |
| Tabs | `list_pages`, `new_page`, `select_page`, `close_page` |
| Console / network | `list_console_messages`, `list_network_requests` |
| Performance | `performance_start_trace`, `lighthouse_audit` |

Many of these accept `includeSnapshot` — set it to get the refreshed tree back
in the same call instead of a separate `take_snapshot` round trip.

This mode drives the user's actual browser. See the `real-chrome-setup` skill
for the one-time opt-in and the cautions that come with it.

## Watching the browser work (headed mode)

There is no headed *trait*. Both Playwright modes would export the same 24
`browser_*` tool names, and session filtering matches bare names, so a second
Playwright server would leak into any session traited for the first — the model
would see every tool twice, unable to tell which browser it was driving.

So headed is a config change, not a mode to switch into mid-session. Drop
`--headless` from the `browser` server args in
`~/repos/packs/browser/barry-pack.yaml`, then restart MCP:

    launchctl kickstart -k gui/$UID/com.barry.mcp.barry

New sessions get a visible window; everything else is identical. Put the flag
back when done. If the user just wants to *see* what happened, prefer
`browser_take_screenshot` — it needs no restart.

## Cautions

- First call in a mode is slow: `npx` fetches and caches the pinned server.
- Close the browser when you're done in `browser` mode.
- In `browser-mine` you are sharing a browser with a human. Don't close their
  tabs, and treat anything you can see there as sensitive.
