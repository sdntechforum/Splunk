# 🔧 Installation & Troubleshooting Guide: MCP Splunk Server

This guide documents the setup of the **MCP Server for Splunk** in a crowded Docker environment. It specifically covers how to resolve port conflicts and migrate sensitive credentials from configuration files to environment variables.

---

## 🚀 The Challenge: The "Port 8000" Battle

In a standard development environment, ports `8000` through `8008` are often occupied by other web services, Splunk instances, or dev servers.

### The Struggle

When first launching via Docker Compose, the system failed with:
`Error response from daemon: Bind for 0.0.0.0:8003 failed: port is already allocated`

This happened because the default **Traefik** entrypoint and the **MCP Inspector** were fighting for ports already claimed by the host machine.

---

## 💡 The Solution: Dynamic Port Re-routing

Instead of hunting through every file to change "8003" to "8010," we implemented a **Variable-First** approach using a `.env` file.

### Key Modifications

1. **Environment Centralization:** We defined `MCP_SERVER_PORT=8010` in the `.env` file.
2. **Docker Compose Logic:** We updated `docker-compose.yml` to use variable substitution: `${MCP_SERVER_PORT:-8003}`. This tells Docker: *"Use port 8010 if defined; otherwise, fall back to 8003."*
3. **Traefik Integration:** Traefik acts as the "Traffic Cop," listening on the new port (8010) and routing `/mcp` traffic to the internal container port (8001).

---

## 🛠 Step-by-Step Installation

### 1. Clone and Initialize

```bash
git clone <your-repo-url>
cd splunk-atl

```

### 2. Configure Environment Variables

Copy the example environment file and update your credentials. **This is where the magic happens.**

```bash
cp env.example .env

```

Edit `.env` to include:

```bash
# Shift the port to avoid conflicts
MCP_SERVER_PORT=8010

# Splunk Credentials (The server reads these directly)
SPLUNK_HOST=so1
SPLUNK_PORT=8089
SPLUNK_USERNAME=admin
SPLUNK_PASSWORD=YourSecurePassword

```

### 3. Deploy with Docker

```bash
docker compose down --remove-orphans
docker compose up -d --build

```

---

## 🔍 Understanding the Service Architecture

| Service | Host URL | Description |
| --- | --- | --- |
| **MCP Server** | `http://localhost:8010/mcp` | The core engine. (Note: Returns a JSON error in browsers—this is normal!) |
| **Traefik Dashboard** | `http://localhost:8280/dashboard/` | Visualizes your routing and entrypoints. |
| **MCP Inspector** | `http://localhost:6274` | Web UI to test tools like `run_splunk_search`. |

### Concept: The "Not Acceptable" Error

If you visit `http://localhost:8010/mcp` in a browser, you will see:
`"Not Acceptable: Client must accept text/event-stream"`

**This is a sign of success.** It confirms the server is alive but waiting for a **Server-Sent Events (SSE)** connection (like Cursor or the Inspector) rather than a standard web browser.

---

## 🤖 Connecting to Cursor (Security Best Practice)

To keep your `mcp.json` clean and secure, we do not declare passwords in the Cursor config. Because the server is already running in Docker with access to your `.env`, we only need the URL.

**Cursor Configuration:**

```json
{
  "mcpServers": {
    "splunk-in-docker": {
      "url": "http://localhost:8010/mcp",
      "headers": {
        "X-Session-ID": "splunk-production-session"
      }
    }
  }
}

```

---

## 📜 Summary of Lessons Learned

* **Don't hardcode:** Use `${VAR:-default}` in Compose files for portability.
* **Traefik is King:** Using a reverse proxy allows you to change external ports without touching the internal application logic.
* **Security First:** Use Docker's `env_file` capability to inject secrets into the container so that your client-side tools (like Cursor) don't need to hold sensitive data.
