---
name: real-chrome-setup
description: One-time setup to let Barry drive the user's real Chrome with their existing logins. Use when browser-mine tools fail to connect, or when the user wants Barry to browse as themselves for the first time.
---

# Driving the real Chrome

The `browser-mine` mode attaches to the user's already-running Chrome, so pages
load with their real cookies and sessions. That requires a one-time opt-in
inside Chrome. Until it's granted, `browser-mine` tools will fail to connect.

## Why there is no way around this

Chrome refuses `--remote-debugging-port` on the default user-data-dir — Google's
docs state the regular browsing profile cannot be used that way, and it was
verified on this machine. Launching Chrome with a debugging flag therefore gets
you a *fresh* profile, which is exactly the logged-out browser the clean modes
already provide.

So the only routes to the real profile are Chrome's own opt-in
(`chrome-devtools-mcp --autoConnect`, Chrome 144+, what this pack uses) or the
`@playwright/mcp --extension` bridge.

## The opt-in (user does this once)

1. In Chrome, open `chrome://inspect/#remote-debugging`.
2. Enable remote debugging.
3. Accept the confirmation dialog.

It persists, so this is genuinely once per machine. Ask the user to do it — you
cannot click through a Chrome settings page on their behalf, and shouldn't try.

## What the user is agreeing to

Say this plainly before asking, rather than after:

- It exposes browser content to the MCP client. Google's own warning on that
  page is explicit, and it is a real security tradeoff, not a formality.
- `--autoConnect` attaches to the running browser and **can see every open tab**,
  including ones unrelated to the task — email, banking, work systems.
- Anything reachable while logged in is reachable by the agent, including
  actions that change state.

Treat it as borrowing someone's unlocked laptop: navigate deliberately, stay on
the task you were asked to do, don't rummage through other tabs, and don't
close or disturb the user's windows.

## When it isn't working

- **Pages render logged-out** — `--autoConnect` didn't attach to the real
  profile. It fell back to a fresh context; re-check the opt-in above.
- **Connection refused** — Chrome isn't running. It must already be open;
  this mode attaches, it does not launch.
- **Tool calls hang with no error** — the opt-in has lapsed. This is the most
  confusing failure, because nothing reports a problem: `initialize` succeeds,
  the pack connects, and then the first real call simply never returns. It
  reads like a slow page.

  Confirm it in one command:

      curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:9222/json

  `200` means remote debugging is live. **`404` means the opt-in is off** —
  Chrome still listens on the port, but refuses every DevTools endpoint, so
  the server waits forever. Connection-refused instead means Chrome is closed.

  Re-enable it at `chrome://inspect/#remote-debugging`. Observed lapsing
  *without* a Chrome restart, so treat it as something to re-check whenever
  this mode stops responding, not a one-time setup step.
- **Tools not found** — the session lacks the `browser-mine` trait, or is using
  Playwright's `browser_*` names. This mode's tools are `navigate_page`,
  `take_snapshot`, `click`, and friends. See the `web-browsing` skill.

If the real profile isn't required, prefer `browser-headless` — it has none of
this setup or exposure.
