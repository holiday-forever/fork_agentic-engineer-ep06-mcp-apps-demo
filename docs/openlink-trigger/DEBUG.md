# Debugging `window.openLink` in MCP App widgets

If a widget button doesn't trigger Claude Desktop's "You're leaving Claude
to visit an external link" modal, it is almost always one of the items
below — in order of likelihood.

## 1. Skipped the handshake

`ui/openLink` only works *after* `ui/initialize` completes **and**
`ui/notifications/initialized` has been sent.

If you call `openLink` on page load, race conditions silently swallow it.
Either await initialization, or only call from a user-gesture event
handler (button click) — which by definition fires after init.

## 2. CSP killed the script

Claude Desktop blocks `script-src` from third-party origins. The script
**must be inline** — no `<script src="https://cdn...">`. `'unsafe-inline'`
is permitted.

## 3. Used `window.open` directly

`window.open(url)` does nothing inside the sandboxed widget iframe. You
**must** go through the postMessage JSON-RPC call (`ui/openLink`).

## 4. Used `<a href target="_blank">`

Same problem — the iframe sandbox blocks top-level navigation. Intercept
the click with `event.preventDefault()` and call `window.openLink(url)`
instead.

```html
<a href="https://example.com" onclick="event.preventDefault(); window.openLink(this.href)">
  Visit
</a>
```

## 5. Host doesn't advertise `openLinks` capability

Older hosts and non-Claude hosts (Goose, MCPJam) may not implement
`ui/openLink`. After `ui/initialize`, inspect the returned
`hostCapabilities.openLinks` — if it's `undefined`, fall back to
rendering the URL as plain text the user can copy.

## 6. URL is malformed or non-http

Claude Desktop validates the URL. Non-`http(s)` schemes (`file://`,
`javascript:`, etc.) are rejected without showing the modal. No error
surfaces in the widget — check Claude Desktop's dev console.

## 7. `postMessage` listener attached too late

Attach the `message` listener **before** sending `ui/initialize`. The
provided `openlink-inline.html` does this in the right order — preserve
it.

## Quick smoke test

Use `smoke-test.html`. Serve it as your MCP App's UI resource and trigger
it in Claude Desktop.

- If clicking "Test open link" shows the modal: the protocol works, the
  bug is in the caller's button-wiring code.
- If it doesn't: the handshake hasn't completed. Open Claude Desktop's
  dev tools and look for `[widget] ui/initialize failed` in the console.
