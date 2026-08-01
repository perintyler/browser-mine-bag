# browser-mine

Drives the user's **real, already-running Chrome**, with their existing cookies
and logins, via [`chrome-devtools-mcp`](https://github.com/ChromeDevTools/chrome-devtools-mcp).

    barry start --traits browser-mine

Requires a one-time opt-in at `chrome://inspect/#remote-debugging`. See
`skills/real-chrome-setup/` for the steps and the security tradeoff — this
exposes browser content, and `--autoConnect` can see every open tab.

Use this only when the task genuinely needs the user's logged-in state.
Otherwise prefer the `browser` (headless) pack, which has none of this exposure.

## Concurrent sessions share the real Chrome

`session-scoped: true` gives each Barry session its own server process, but every
process attaches to the same real Chrome — that is the whole point of the mode,
so tabs are genuinely shared. `--experimentalPageIdRouting` exposes `pageId` on
page-scoped tools so a session can pin the tab it is working on instead of
relying on which one happens to be selected.

## Different tool names from the Playwright packs

This server's vocabulary is `navigate_page`, `take_snapshot`, `click`/`fill`
with a **`uid`** element reference — not Playwright's `browser_navigate`,
`browser_snapshot`, and `target`. `skills/web-browsing/` documents both; using
the wrong set is the most common failure mode.

Chrome refuses `--remote-debugging-port` on the default profile, so the
`chrome://inspect` opt-in is the only way to reach the real logged-in profile.
