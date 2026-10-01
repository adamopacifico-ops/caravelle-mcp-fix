# Materiale di contorno — caravelle-mcp-fix

## 1. Descrizione del repository (campo "About")

```
Fix for MCP SSE connector: session deleted on HTTP response close — "Server not initialized" (-32000) on claude.ai. Node.js + Caddy.
```

## 2. Topics da aggiungere (rotella accanto ad "About")

```
mcp
model-context-protocol
sse
server-sent-events
claude
mcp-server
nodejs
caddy
reverse-proxy
session-management
```

## 3. Commento da lasciare sulle issue altrui

Testo breve, uguale per tutte, con il link in fondo.

> Same symptom here, different cause — worth checking before chasing proxy timeouts.
>
> In our case the connection was fine; the **session** was gone. The server was
> deleting its session map inside `res.on("close", ...)` on the SSE route. That
> handler fires whenever a single HTTP response ends, which behind a reverse proxy
> happens routinely — but the MCP protocol requires the session to persist across
> multiple requests. The client then sends a `sessionId` the server has already
> dropped, and every later call is answered as `Server not initialized` / `-32000`
> or `No transport found for sessionId`.
>
> Keep-alive heartbeats and Streamable HTTP do not help here, because the transport
> is healthy and the session is the thing that disappeared.
>
> Write-up with the offending pattern, the corrected server code and the Caddy
> configuration: https://github.com/adamopacifico-ops/caravelle-mcp-fix

Dove lasciarlo:

- https://github.com/anthropics/claude-code/issues/21032  (Todoist MCP — "No transport found for sessionId")
- https://github.com/0xsline/OpenChatCut/issues/195       (MCP SSE session permanently destroyed)
- https://github.com/supercorp-ai/supergateway/pull/154   (transport non chiuso alla riconnessione — caso speculare)
- https://github.com/mark3labs/mcp-go/issues/967          (SSE transport Start dopo Close)

## 4. Titolo per una issue nell'SDK ufficiale

Repository: `modelcontextprotocol/typescript-sdk`

Titolo:

```
SSE example deletes the session in res.on("close"), breaking sessions behind a reverse proxy
```

Corpo, in sintesi: il frammento di esempio con `delete transports[transport.sessionId]`
dentro `res.on("close")` lega la sessione alla durata di una singola risposta HTTP,
mentre il protocollo richiede che sopravviva a più richieste. Proposta: rimuovere la
cancellazione dall'handler e usare una scadenza a tempo, come nel README allegato.

## 5. Nota sul titolo della issue #1 esistente

Attuale: "MCP SSE connector fix — session deleted on HTTP close"

Più trovabile:

```
"Server not initialized" (-32000) after first tool call — MCP SSE session deleted on HTTP response close
```

Il messaggio d'errore letterale all'inizio del titolo è ciò che la gente incolla
nei motori di ricerca.
