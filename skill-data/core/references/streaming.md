# Live Streaming

Stream a session's viewport over WebSocket and drive it with remote input. This is what a remote preview or embedded dashboard connects to: the browser runs wherever the daemon runs (a sandbox, a container, a CI box), and the client renders frames and sends clicks back.

**Related**: [commands.md](commands.md) for full command reference, [SKILL.md](../SKILL.md) for quick start.

## Contents

- [Enabling the stream](#enabling-the-stream)
- [Connecting](#connecting)
- [Messages from the server](#messages-from-the-server)
- [Messages from the client](#messages-from-the-client)
- [Frame rate and staleness](#frame-rate-and-staleness)
- [Limitations](#limitations)

## Enabling the stream

Streaming is always available; the server binds an OS-assigned localhost port unless told otherwise.

```bash
agent-browser stream status --json     # Report enabled state, port, client count
agent-browser stream enable            # Create the server (--port to pin one)
agent-browser stream disable           # Tear it down
```

`AGENT_BROWSER_STREAM_PORT` pins the port for the whole daemon instead of passing `--port`.

Read the port from `stream status --json` rather than assuming one; the OS-assigned default changes per daemon.

## Connecting

Connect a WebSocket client to `ws://127.0.0.1:<port>`. Frame delivery starts automatically once a client attaches, so there is no subscribe message. Connections from a browser page are subject to an origin allowlist.

## Messages from the server

Every message is JSON text with a `type` field.

- `frame`: a viewport image plus its metadata. Delivered latest-first (see below).
- `status`: connection state, screencasting flag, viewport size, engine, recording flag. Sent once on connect and again on change.
- `tabs`: the current tab list, sent on connect when tabs are known and on change.
- `url`, `console`: navigation and console events.

Status, tabs, url, and console travel on an ordered channel and are never dropped. Only frames are subject to dropping.

## Messages from the client

```json
{"type": "input_mouse", "eventType": "mousePressed", "x": 40, "y": 40, "button": "left", "clickCount": 1}
{"type": "input_keyboard", "eventType": "keyDown", "key": "a", "text": "a"}
{"type": "input_touch", "eventType": "touchStart", "touchPoints": []}
{"type": "config", "maxFps": 10}
```

Input dispatches to the browser on a task of its own, separate from frame delivery, so a click is not queued behind a frame write. Mouse, keyboard, and touch input also reset the daemon idle timer, so an actively driven preview is not shut down by the idle timeout.

`config` sets a per-client frame cap: 1 to 120, or `0` for uncapped (the default). It takes effect immediately, including when it loosens the cap. Each client's cap is its own; other connected clients are unaffected. Out-of-range and non-numeric values are ignored rather than rejecting the connection.

## Frame rate and staleness

The server holds only the newest frame per client and reads it at send time. A frame produced while an earlier one is still being written is skipped, not queued, so the application never builds a backlog.

The transport underneath is still ordered: frames already accepted by the socket are delivered in order. A client that stops reading entirely drains whatever the kernel buffered before the writer blocked. `maxFps` bounds that window, because the server only produces frames for that client at the requested rate. A preview that expects to stall (a background tab, a throttled render loop) should set a low `maxFps` rather than rely on uncapped delivery.

## Limitations

- Localhost only. Exposing the stream beyond the machine is the embedder's job (tunnel, proxy, or port forward), and the origin allowlist applies to browser clients.
- Frames are images, not a video codec. Bandwidth scales with viewport size and page activity; cap the rate for constrained links.
- There is no delivery acknowledgement, so the server cannot tell a slow renderer from a fast one beyond transport backpressure.
