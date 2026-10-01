# caravelle-mcp-fix

**Fix for an MCP server over SSE that authenticates once and then silently fails every later tool call in the same session.**

Symptoms on the client side:

```
Server not initialized
MCP error -32000
No transport found for sessionId
Already connected to a transport. Call close() before connecting to a new transport
```

Self-hosted Node.js MCP server on a local network, reverse-proxied through Caddy, connected to claude.ai as a custom connector.

---

## Symptom

The connector is added and authenticates. The first tool call works. Every later
call in the same session fails, usually with `Server not initialized` and JSON-RPC
error `-32000`, or with `No transport found for sessionId`. The server logs show no
error: from its point of view the session simply no longer exists.

Restarting the server does not help. Removing and re-adding the connector makes it
work once more, then the same thing happens again.

## Root cause

Two independent things go wrong at the same time, which is why the usual advice
about proxy timeouts does not fix it.

**1. Client side — the SSE stream is dropped between requests and not re-established.**
The browser keeps a stale session identifier and sends it on the next tool call.

**2. Server side — the session map is deleted when the HTTP response closes.**
This is the part that is almost never mentioned. Example of the offending pattern:

```js
app.get("/sse", async (req, res) => {
  const transport = new SSEServerTransport("/messages", res);
  transports[transport.sessionId] = transport;

  res.on("close", () => {
    delete transports[transport.sessionId];   // <-- wrong
  });

  await server.connect(transport);
});
```

`res.on("close")` fires whenever that single HTTP response ends — which happens
routinely behind a reverse proxy, on network hiccups, on tab backgrounding. The MCP
protocol requires the **session to persist across multiple requests**: it is not
bound to the lifetime of one HTTP response. Deleting the entry there destroys a
session the client still considers valid, and every later `POST /messages` carrying
that `sessionId` is answered as if the server had never been initialized.

## Fix

**Server side.** Do not tie the session map to the response lifetime. Keep the
transport until the session is actually finished, and expire it on a timer instead:

```js
app.get("/sse", async (req, res) => {
  const transport = new SSEServerTransport("/messages", res);
  transports[transport.sessionId] = transport;
  lastSeen[transport.sessionId] = Date.now();

  // no delete here

  await server.connect(transport);
});

app.post("/messages", async (req, res) => {
  const id = req.query.sessionId;
  const transport = transports[id];
  if (!transport) return res.status(404).send("No transport found for sessionId");
  lastSeen[id] = Date.now();
  await transport.handlePostMessage(req, res);
});

// reap genuinely dead sessions, not merely closed responses
setInterval(() => {
  const now = Date.now();
  for (const id of Object.keys(lastSeen)) {
    if (now - lastSeen[id] > 30 * 60 * 1000) {
      transports[id]?.close?.();
      delete transports[id];
      delete lastSeen[id];
    }
  }
}, 60 * 1000);
```

**Reverse proxy (Caddy).** Disable buffering and allow a long-lived stream:

```
handle /sse* {
    reverse_proxy 127.0.0.1:3000 {
        flush_interval -1
        transport http {
            read_timeout 0
            write_timeout 0
        }
    }
}
```

**Client side**, to recover a connector already stuck:

1. Remove the connector.
2. Clear the site's session data for claude.ai.
3. Add the connector again and choose **Always allow**.
4. Restart the browser completely before testing.
5. Expect the first call to time out; the second one succeeds.

## Why the usual advice does not apply

Most write-ups on "MCP SSE timeout" blame idle timeouts in proxies, load balancers
and CDNs, and recommend keep-alive heartbeats or moving to Streamable HTTP. Those
are real problems, but they produce a *connection* failure. Here the connection is
fine and the *session* is gone: the server answers promptly, and answers that it
does not know you. Heartbeats do not fix a session the server has already deleted.

## Environment where this was observed

- Node.js MCP server, SSE transport, self-hosted on a local network
- Caddy as reverse proxy with TLS
- claude.ai custom connector
- Reproduced repeatedly over several months; the failure returns after any event
  that ends the HTTP response

## License

MIT.
