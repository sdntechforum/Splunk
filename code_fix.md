# Splunk-ATL MCP Server Fix – What Happened

## 1. The Problem

You had two Splunk MCP servers:

- **splunk-atl** – Connects to Splunk at `10.100.0.41` (your local Splunk)
- **Splunk-MCP-Server** – Connects to `ciscovalidated.com`

splunk-atl kept returning:

> "Splunk connection not available: Splunk service is not available. MCP server is running in degraded mode."

Even though:

- `curl` from your Mac to `10.100.0.41` worked
- `.env` had the right host, port, username, and password

So the issue was inside the splunk-atl MCP server, not in Splunk itself.

---

## 2. How the Splunk-ATL MCP Server Works

### Architecture

```
┌─────────────────┐     HTTP (port 8010)      ┌──────────────────┐
│  Cursor IDE     │ ◄──────────────────────► │  splunk-atl      │
│  (MCP Client)   │                           │  MCP Server      │
└─────────────────┘                           │  (Docker)        │
                                              └────────┬─────────┘
                                                       │
                                                       │ HTTPS (8089)
                                                       ▼
                                              ┌──────────────────┐
                                              │  Splunk          │
                                              │  10.100.0.41     │
                                              └──────────────────┘
```

- Cursor talks to the MCP server over HTTP.
- The MCP server runs in Docker and talks to Splunk over HTTPS.

### How `list_indexes` Gets a Splunk Connection

When you call `list_indexes`, the flow is:

1. Tool handler runs.
2. It calls `get_splunk_service(ctx)`.
3. That uses `check_splunk_available(ctx)` to see if Splunk is reachable.
4. `check_splunk_available` uses `_get_splunk_context(ctx)` to find the Splunk connection.

So the critical piece is: where does `_get_splunk_context` get the Splunk connection from?

---

## 3. What I Checked First

### 3.1 Network and Credentials

I ran these inside the splunk-atl container:

```bash
# Can the container reach Splunk?
docker exec mcp-server python3 -c "socket.connect(('10.100.0.41', 8089))"
# Result: Reachable ✓

# Are env vars correct?
docker exec mcp-server python3 -c "print(os.getenv('SPLUNK_HOST'))"
# Result: 10.100.0.41 ✓

# Does curl work from inside the container?
docker exec mcp-server curl -sk -u "splunk:Shark01!" "https://10.100.0.41:8089/services/server/info"
# Result: XML response ✓
```

So:

- Network: OK
- Env vars: OK
- Splunk API: OK

The problem was not connectivity or credentials.

### 3.2 Startup Logs

```text
2026-03-09 16:42:11 - src.client.splunk_client - INFO - Connecting to Splunk at https://10.100.0.41:8089
2026-03-09 16:42:11 - src.client.splunk_client - INFO - Successfully connected to Splunk
2026-03-09 16:42:11 - src.server - INFO - Splunk connection established for module initialization using server environment
```

So at startup the server **does** connect to Splunk and stores that in `server._splunk_context`. But when `list_indexes` ran, it still reported “Splunk service is not available”.

That suggested the tool was not using the same context that was set at startup.

---

## 4. How the Code Was Choosing the Splunk Context

### 4.1 Original `_get_splunk_context` Logic

```python
def _get_splunk_context(self, ctx: Context):
    # 1. Try lifespan context first
    if hasattr(ctx.request_context, "lifespan_context"):
        return ctx.request_context.lifespan_context

    # 2. Fallback: server._splunk_context
    server = get_server()
    if hasattr(server, "_splunk_context"):
        return server._splunk_context

    return None
```

So the order was:

1. Use `lifespan_context` if it exists.
2. Otherwise use `server._splunk_context`.

### 4.2 What “lifespan” Means Here

In FastMCP/Starlette, the lifespan is a context that runs when the app starts and when it shuts down. It’s often used to open connections once and reuse them.

In splunk-atl’s `server.py` there is a comment:

```python
# Initialize FastMCP server without lifespan (components loaded at startup instead)
# Note: lifespan causes issues in HTTP mode as it runs for each SSE connection
```

So splunk-atl intentionally does **not** use a custom lifespan for Splunk. Instead it:

- Loads components at startup
- Connects to Splunk during that startup
- Stores the connection in `server._splunk_context`

But FastMCP still provides a `lifespan_context` for each HTTP/SSE connection. That context is **not** the one splunk-atl fills with Splunk data. So:

- `ctx.request_context.lifespan_context` existed
- It was not the Splunk context
- It had no `service` or `is_connected`
- The code preferred it over `server._splunk_context` and returned it
- `check_splunk_available` saw “no service” and reported degraded mode

So the bug was: the code always preferred `lifespan_context` over `server._splunk_context`, even when the latter had a valid Splunk connection.

---

## 5. The Fixes

### Fix 1: Prefer `server._splunk_context` When It Has a Connection

The goal was: when `server._splunk_context` has a valid Splunk connection, use it first.

```python
def _get_splunk_context(self, ctx: Context):
    # NEW: Prefer server._splunk_context when it has a valid connection
    try:
        server = get_server()
        if hasattr(server, "_splunk_context"):
            sctx = server._splunk_context
            if sctx and getattr(sctx, "is_connected", False) and getattr(sctx, "service", None):
                return sctx  # Use this first!
    except Exception:
        pass

    # Original: Try lifespan context
    try:
        if hasattr(ctx.request_context, "lifespan_context"):
            return ctx.request_context.lifespan_context
    except Exception:
        pass

    # Fallback: server instance
    try:
        server = get_server()
        if hasattr(server, "_splunk_context"):
            return server._splunk_context
    except Exception:
        pass

    return None
```

So now:

1. If `server._splunk_context` has `is_connected` and `service`, we use it.
2. Otherwise we fall back to lifespan context and then to `server._splunk_context` again.

This fixed the case where the startup connection was stored in `server._splunk_context` but the tool was using the wrong context.

### Fix 2: Fresh Connection Fallback

There was still a risk: in some setups (e.g. multiple workers or different processes), `server._splunk_context` might not be set or might be from another process. So a second fallback was added: if no stored context has a connection, try to create a new one from env vars.

```python
# In check_splunk_available(), when splunk_ctx has no connection:
if not is_connected or not service:
    # NEW: Final fallback - try fresh connection from env vars
    try:
        from src.client.splunk_client import get_splunk_service_safe
        service = get_splunk_service_safe(None)  # None = use env vars
        if service:
            return True, service, ""
    except Exception:
        pass

    return (False, None, "Splunk service is not available...")
```

`get_splunk_service_safe(None)` reads `SPLUNK_HOST`, `SPLUNK_PORT`, `SPLUNK_USERNAME`, `SPLUNK_PASSWORD` from the environment and connects. Since we already knew env vars and network were correct, this fallback should work when the stored context is missing or invalid.

### Fix 3: `.env` Quotes

Earlier, `.env` had:

```env
SPLUNK_HOST='10.100.0.41'
SPLUNK_USERNAME='splunk'
```

Some env parsers treat quotes as part of the value, so the host could become `'10.100.0.41'` (with quotes). That was changed to:

```env
SPLUNK_HOST=10.100.0.41
SPLUNK_USERNAME=splunk
```

This avoids potential connection issues from malformed values.

---

## 6. Summary of Changes

| File | Change |
|------|--------|
| `src/core/base.py` | 1. Prefer `server._splunk_context` when it has a valid connection. 2. Add a fallback that creates a fresh Splunk connection from env vars when no stored context is usable. |
| `.env` | Remove quotes from Splunk-related values. |

---

## 7. Why It Broke in the First Place

- splunk-atl uses **HTTP/streamable** transport.
- FastMCP provides a `lifespan_context` per connection.
- splunk-atl does not populate that context with Splunk; it uses startup loading and `server._splunk_context`.
- The original code always preferred `lifespan_context` over `server._splunk_context`.
- The default `lifespan_context` had no Splunk connection, so the tool always saw “no connection” and reported degraded mode.

The fix was to:

1. Prefer the context that actually has a Splunk connection (`server._splunk_context`).
2. Add a last-resort fallback to connect from env vars.
3. Clean up `.env` so values are parsed correctly.

---

## 8. How to Verify It’s Working

After rebuilding and restarting the container:

```bash
cd /Users/amitsi4/MCP/splunk-atl
docker compose up -d --build mcp-server
```



