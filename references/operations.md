# Operations: daemon lifecycle and recovery

Read this only when a tool call can't reach the daemon, or the user explicitly asks to install / start / troubleshoot kimi-webbridge.

## The daemon

The `kimi-webbridge` binary lives at `~/.kimi-webbridge/bin/kimi-webbridge` (Windows: `%USERPROFILE%\.kimi-webbridge\bin\kimi-webbridge.exe`) and serves a local HTTP daemon on `127.0.0.1:10086`. Status, PID, and logs live under `~/.kimi-webbridge/`.

## Two runtime scenarios

| Scenario | Daemon lifecycle | Notes |
|----------|-----------------|-------|
| Kimi Desktop App | Auto-started with app, auto-stopped on close | Do NOT use CLI `start`/`stop`/`restart` — conflicts with Desktop App |
| Other local agents (WorkBuddy, Claude Desktop, etc.) | Agent manages lifecycle independently | Install: `irm https://cdn.kimi.com/webbridge/install.ps1 \| iex` (Windows) |

## /status JSON fields

- `running` (bool) — daemon listening on `:10086`
- `version` (string) — daemon build version
- `extension_connected` (bool) — a WebSocket client (the browser extension) is attached
- `extension_id` (string) — the Chrome/Edge extension ID, empty if none
- `extension_version` (string) — browser extension version
- `uptime_seconds` (int)

## Recovery — what to do when a tool call fails

1. **Daemon not reachable (connection refused)** → start it yourself, don't ask the user. `start` is idempotent: it no-ops if the daemon is already up.

   **macOS / Linux:**
   ```bash
   ~/.kimi-webbridge/bin/kimi-webbridge start
   ```

   **Windows (PowerShell):**
   ```powershell
   & "$env:USERPROFILE\.kimi-webbridge\bin\kimi-webbridge.exe" start
   ```

   Then retry the tool call.

2. **`command not found` / binary missing** → not installed. Point the user to the help page.

3. **Extension won't connect** (`extension_connected: false` after daemon is running):
   - Kimi Desktop App user: restart Kimi Desktop
   - Other agents: reinstall via `curl -fsSL https://kimi-web-img.moonshot.cn/webbridge/install_skill.sh | bash -s -- -y` then restart agent
   - If still broken → point user to https://www.kimi.com/zh-cn/features/webbridge

4. **Port 10086 occupied** → Desktop App + another agent running simultaneously. Close one, or share the existing daemon.

## Do NOT do automatically

Never run `stop` / `restart` / `uninstall` on your own. They kill the running daemon; if the user runs the **Kimi Desktop App** (which manages its own daemon), an external stop/restart also fights the app.

## Browser management (Windows)

### Multi-browser conflict

If both Edge and Chrome have the WebBridge extension installed AND both are running, WebBridge connects to either one randomly. Always close the non-target browser before starting a task.

### Profile detection

```bash
tasklist.exe | grep -i "msedge.exe" || echo "EDGE_NOT_RUNNING"
tasklist.exe | grep -i "chrome.exe" || echo "CHROME_NOT_RUNNING"
ls -la "$LOCALAPPDATA/Microsoft/Edge/User Data/" | grep -E "^d.*(Default|Profile)"
ls -la "$LOCALAPPDATA/Google/Chrome/User Data/" | grep -E "^d.*(Default|Profile)"
```

### Browser close (safe)

```bash
# Normal close — wait 8-10s for clean shutdown (saves cookies/sessions)
taskkill.exe /IM msedge.exe > /dev/null 2>&1
sleep 8

# Check residual — if still running, prompt user to close manually
tasklist.exe | grep -i "msedge.exe" > /dev/null
if [ $? -eq 0 ]; then
  echo "⚠️ Browser still running — please close manually"
fi
```

⚠️ **Never `taskkill /F` without user consent.** Forced kill loses cookies/login state. If normal close fails, pause and ask the user.

### Browser start (Windows)

**Recommended (CMD, most reliable):**
```bash
cmd.exe /c "start \"\" \"C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe\" --profile-directory=\"Profile 1\""
```

**Fallback (PowerShell):**
```powershell
powershell -Command "Start-Process 'C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe' -ArgumentList '--profile-directory=Profile 1'"
```

Wait 8-10s after start for extension to connect. Verify:
```bash
sleep 10 && curl.exe -s http://127.0.0.1:10086/status
```

⚠️ Sandbox startup may be unstable (process starts but window doesn't appear). If `extension_connected` stays `false`, ask user to open browser manually.

### Profile parameter format
- Edge: `--profile-directory="Profile 1"` (quoted)
- Chrome: `--profile-directory=Profile 1` (unquoted)
- `Default` = the default profile in both browsers

### Verify correct browser/profile
After startup: screenshot to confirm browser icon/account, snapshot to check page content. If wrong, pause and let user switch manually.

## Windows encoding issues

- **Git Bash + `taskkill`**: output is GBK-encoded, shows gibberish. Use `> /dev/null 2>&1` to suppress.
- **Git Bash + paths**: `MSYS_NO_PATHCONV=1` prefix prevents path mangling (e.g., `/F` becoming `F:/`).
- **Chinese characters in command line**: Always use file-body pattern (`--data-binary @file`), never inline.
