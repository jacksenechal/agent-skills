# Browser Automation Setup

This skill requires an MCP server that provides Playwright-style browser tools
(`browser_navigate`, `browser_snapshot`, `browser_click`, `browser_type`, etc.).
Two options are supported.

## Option A: browsermcp (Simple)

[browsermcp](https://browsermcp.com/) controls your real desktop browser via the Chrome
DevTools Protocol. It's the fastest way to get started.

**Pros:** Zero infrastructure. Uses your real browser with your real sessions (LinkedIn
already logged in). Minimal setup.

**Cons:** Cannot do file uploads (OS-level file dialogs are unreachable). OAuth popups
(e.g., "Apply with LinkedIn") may not work reliably. Tied to your real browser state —
automation and personal browsing share a session.

### Setup

1. Install the browsermcp Chrome extension from https://browsermcp.com/
2. Add to your Claude Code project settings (`.claude/settings.local.json`):

```json
{
  "mcpServers": {
    "browsermcp": {
      "command": "npx",
      "args": ["@anthropic-ai/browsermcp@latest", "--browserName", "chrome"]
    }
  }
}
```

3. The browser tools will be prefixed with `mcp__browsermcp__` (e.g.,
   `mcp__browsermcp__browser_snapshot`). Add permissions as needed:

```json
{
  "permissions": {
    "allow": [
      "mcp__browsermcp__browser_navigate",
      "mcp__browsermcp__browser_snapshot",
      "mcp__browsermcp__browser_click",
      "mcp__browsermcp__browser_type",
      "mcp__browsermcp__browser_press_key",
      "mcp__browsermcp__browser_wait",
      "mcp__browsermcp__browser_screenshot",
      "mcp__browsermcp__browser_hover",
      "mcp__browsermcp__browser_select_option"
    ]
  }
}
```

### File uploads with browsermcp

browsermcp **cannot** handle file uploads — clicking a file input triggers a native OS
dialog that the browser DevTools Protocol cannot interact with. When a form requires a
resume upload, the user must upload manually.

---

## Option B: Dockerized Playwright + noVNC (Recommended)

A Docker container running [Microsoft Playwright MCP](https://github.com/microsoft/playwright-mcp)
with a headed Chromium browser visible via [noVNC](https://novnc.com/). Based on
[mcp-playwright-novnc](https://github.com/xtr-dev/mcp-playwright-novnc).

**Pros:** Programmatic file uploads via `browser_file_upload` tool. Persistent browser
profile (LinkedIn stays logged in across container restarts). Container isolation (no risk
to your real browser). Real-time monitoring via noVNC web UI. Same DOM-level Playwright
tools as browsermcp.

**Cons:** Requires Docker. Uses ~1GB RAM. Initial setup takes a few minutes.

### Prerequisites

- Docker and Docker Compose
- ~2GB disk for the image

### Setup

Create a directory for the Docker config (e.g., `docker/playwright/` in your job-search repo).

#### 1. `docker-compose.yml`

```yaml
services:
  playwright-display:
    image: ghcr.io/xtr-dev/mcp-playwright-novnc:latest
    container_name: playwright-display
    ports:
      - "6080:6080"   # noVNC web interface
      - "3080:3080"   # MCP SSE endpoint
    volumes:
      # Your resume repo — so the browser can access resume.pdf for uploads.
      # Change this path to wherever your resume repo lives.
      - ${RESUME_REPO_PATH:-~/workspace/resume}:/home/pwuser/resume:ro
      # Browser profile — persists LinkedIn sessions across container restarts
      - playwright-profile:/home/pwuser/persistent-profile
      # Custom start script for session persistence
      - ./start-mcp.sh:/usr/local/bin/start-mcp.sh:ro
    environment:
      - SCREEN_WIDTH=1920
      - SCREEN_HEIGHT=1080
      - MCP_BROWSER=chromium
      - MCP_PORT=3080
    shm_size: '2gb'
    # Required: Docker's default seccomp profile is too restrictive for Chromium.
    # Without these, Chromium crashes with SIGTRAP on launch.
    cap_add:
      - SYS_PTRACE
    security_opt:
      - seccomp:unconfined
    restart: unless-stopped
    networks:
      - playwright-network

volumes:
  playwright-profile:

networks:
  playwright-network:
    name: playwright-network
```

#### 2. `start-mcp.sh`

This replaces the upstream start script to enable session persistence. The upstream uses
`--isolated` (in-memory profile, fresh on every connection). We remove that and add
`--user-data-dir` and `--shared-browser-context` so LinkedIn cookies survive restarts and
all MCP clients share one browser instance.

```bash
#!/bin/bash
set -e

# Wait for X11 to be ready
sleep 2

# Kill any orphaned browser processes from a previous crash
for pid_exe in /proc/[0-9]*/exe; do
    if ls -l "$pid_exe" 2>/dev/null | grep -q "chrome\|chromium"; then
        target_pid=$(echo "$pid_exe" | cut -d/ -f3)
        echo "Cleaning up stale browser process: $target_pid"
        kill -9 "$target_pid" 2>/dev/null || true
    fi
done

# Profile directory mapped to a Docker volume
USER_DATA_DIR="/home/pwuser/persistent-profile"
mkdir -p "$USER_DATA_DIR"

# Clean stale Chromium lock files from ungraceful shutdowns
rm -f "$USER_DATA_DIR/SingletonLock" \
      "$USER_DATA_DIR/SingletonCookie" \
      "$USER_DATA_DIR/SingletonSocket" 2>/dev/null || true

VIEWPORT_WIDTH=$((${SCREEN_WIDTH:-1920} - 0))
VIEWPORT_HEIGHT=$((${SCREEN_HEIGHT:-1080} - 0))

echo "Starting Playwright MCP server..."
echo "  Port: ${MCP_PORT:-3000}"
echo "  Browser: ${MCP_BROWSER:-chromium}"
echo "  Display: ${DISPLAY:-:99}"
echo "  Viewport: ${VIEWPORT_WIDTH}x${VIEWPORT_HEIGHT}"
echo "  User Data Dir: $USER_DATA_DIR"

# --user-data-dir: persistent profile (sessions survive restarts)
# --shared-browser-context: all SSE clients share one browser instance
# No --isolated: enables persistence to disk
exec node /app/cli.js \
    --host 0.0.0.0 \
    --port "${MCP_PORT:-3000}" \
    --browser "${MCP_BROWSER:-chromium}" \
    --config /etc/playwright-config.json \
    --allowed-hosts "*" \
    --viewport-size "${VIEWPORT_WIDTH}x${VIEWPORT_HEIGHT}" \
    --user-data-dir "$USER_DATA_DIR" \
    --shared-browser-context
```

Make it executable: `chmod +x start-mcp.sh`

#### 3. Start the container

```bash
cd docker/playwright
docker compose up -d
```

- noVNC UI: http://localhost:6080
- MCP SSE endpoint: http://localhost:3080/sse

#### 4. MCP client configuration

Add to `.claude/settings.local.json`:

```json
{
  "mcpServers": {
    "playwright": {
      "command": "docker",
      "args": [
        "run", "--rm", "-i",
        "--network=playwright-network",
        "ghcr.io/xtr-dev/mcp-playwright-novnc:latest",
        "mcp-proxy",
        "http://playwright-display:3080/sse"
      ]
    }
  }
}
```

This uses the image's built-in `mcp-proxy` (a stdio-to-SSE bridge) to connect Claude Code
to the long-running container. Tools will be prefixed with `mcp__playwright__`.

Add permissions:

```json
{
  "permissions": {
    "allow": [
      "mcp__playwright__*"
    ]
  }
}
```

#### 5. LinkedIn session setup (one-time)

1. Open http://localhost:6080 in your browser (noVNC)
2. In the noVNC view, the agent navigates Chromium to `https://linkedin.com/login`
3. **You** manually type your LinkedIn credentials in the noVNC window (the agent NEVER
   handles credentials)
4. LinkedIn session cookies persist in the `playwright-profile` Docker volume
5. Subsequent container restarts preserve the login

#### 6. File uploads

The Playwright MCP server exposes a `browser_file_upload` tool. When a form has a file
input, use it to upload the resume from `/home/pwuser/resume/resume.pdf` (the mounted
resume repo).

### Monitoring

Open http://localhost:6080 at any time to watch the browser in real time via noVNC. This
is useful for reviewing form fills before the user clicks Submit.

### Container lifecycle

The container runs with `restart: unless-stopped`, so it persists across reboots. To
manage it:

```bash
cd docker/playwright
docker compose up -d      # Start
docker compose restart    # Restart (preserves profile volume)
docker compose down       # Stop (preserves profile volume)
docker compose down -v    # Stop AND delete profile volume (resets LinkedIn session)
```
