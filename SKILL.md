---
name: cloakbrowser-manager
description: "Manage CloakBrowser-Manager browser profiles and automate tasks via its REST API and CDP. Use this skill whenever the user mentions CloakBrowser, CloakBrowser-Manager, browser profiles, browser fingerprints, anti-detect browsers, multi-account browser management, stealth browsing, or wants to create/launch/stop/configure isolated browser instances. Also trigger when the user wants to connect Playwright or Puppeteer to a running CloakBrowser profile, write automation scripts that target CloakBrowser, or manage proxies/fingerprints/timezones across multiple browser profiles. Even if the user just says 'launch a browser profile', 'create a new profile with proxy', 'connect to my running browser', or 'write a scraper using my browser profiles', this skill should activate. 中文触发词：CloakBrowser 浏览器指纹、反检测浏览器、多账号浏览器管理、浏览器 profile 管理、CDP 端点连接、配置代理/时区/locale、批量创建 profile、localhost:8970、/api/profiles。"
---

# CloakBrowser-Manager Skill

Operate a locally-running CloakBrowser-Manager instance: manage browser profiles via its REST API and write Playwright/Puppeteer automation scripts that connect to running profiles through CDP.

## Environment

The user runs CloakBrowser-Manager via Docker Compose on **localhost:8970**. The base URL for all API calls is:

```
http://localhost:8970
```

No authentication is configured for local use. If the user mentions an AUTH_TOKEN, pass it as `Authorization: Bearer <token>` header.

## Quick Reference — API Endpoints

All endpoints are under `/api/`. Use `curl` or `httpx`/`requests` in Python.

### Profiles CRUD

| Method | Path | Body | Description |
|--------|------|------|-------------|
| GET | `/api/profiles` | — | List all profiles |
| POST | `/api/profiles` | ProfileCreate JSON | Create a new profile |
| GET | `/api/profiles/{id}` | — | Get profile details |
| PUT | `/api/profiles/{id}` | ProfileUpdate JSON | Update profile settings |
| DELETE | `/api/profiles/{id}` | — | Delete profile (must be stopped first) |

### Lifecycle

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/profiles/{id}/launch` | Launch browser instance |
| POST | `/api/profiles/{id}/stop` | Stop browser instance |
| GET | `/api/profiles/{id}/status` | Check running/stopped status |
| GET | `/api/status` | System status (running count, total profiles) |

### Clipboard & CDP

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/profiles/{id}/clipboard` | Set clipboard text `{"text": "..."}` |
| GET | `/api/profiles/{id}/clipboard` | Get clipboard text |
| GET | `/api/profiles/{id}/cdp` | Get CDP connection info for automation |

## Profile Configuration

When creating or updating a profile, these fields are available:

```json
{
  "name": "My Profile",
  "fingerprint_seed": 12345,
  "proxy": "http://user:pass@host:port",
  "timezone": "America/New_York",
  "locale": "en-US",
  "platform": "windows",
  "user_agent": null,
  "screen_width": 1920,
  "screen_height": 1080,
  "gpu_vendor": null,
  "gpu_renderer": null,
  "hardware_concurrency": null,
  "humanize": false,
  "human_preset": "default",
  "headless": false,
  "geoip": false,
  "clipboard_sync": true,
  "auto_launch": false,
  "color_scheme": "light",
  "launch_args": [],
  "notes": "optional notes",
  "tags": [{"tag": "scraping", "color": "#3b82f6"}]
}
```

Key fields explained:
- **fingerprint_seed** — integer that deterministically generates the browser fingerprint (canvas, WebGL, audio, fonts, etc.). Same seed = same fingerprint across restarts. Random if omitted.
- **proxy** — supports `http://`, `https://`, `socks5://` schemes, or shorthand `host:port:user:pass`.
- **platform** — `"windows"` | `"macos"` | `"linux"` — affects User-Agent and navigator.platform.
- **humanize** — enables human-like mouse movements and typing delays.
- **human_preset** — `"default"` or `"careful"` (slower, more natural).
- **headless** — run without VNC display (saves resources, useful for pure automation).
- **geoip** — auto-set timezone/locale from proxy IP geolocation.
- **auto_launch** — automatically start this profile when the Manager starts.
- **launch_args** — extra Chromium flags passed to the browser.
- **tags** — array of `{"tag": "name", "color": "#hex"}` for organizing profiles.

Only `name` is required for creation. All other fields have sensible defaults.

## Common Workflows

### 1. Create and launch a profile

```bash
# Create
curl -s -X POST http://localhost:8970/api/profiles \
  -H "Content-Type: application/json" \
  -d '{"name": "Worker-1", "proxy": "socks5://user:pass@proxy.example.com:1080", "timezone": "Europe/London", "locale": "en-GB", "platform": "windows"}' | jq .id

# Launch (use the returned ID)
curl -s -X POST http://localhost:8970/api/profiles/<profile-id>/launch | jq
```

### 2. Batch create profiles

When the user needs multiple profiles (e.g., 10 workers), write a script that loops through the creation endpoint. Vary the proxy, timezone, and locale per profile to create realistic diversity.

### 3. Connect Playwright (Python)

After a profile is launched, connect to it via CDP:

```python
from playwright.async_api import async_playwright
import asyncio

async def main():
    async with async_playwright() as pw:
        browser = await pw.chromium.connect_over_cdp(
            "http://localhost:8970/api/profiles/<profile-id>/cdp"
        )
        context = browser.contexts[0]
        page = context.pages[0]
        await page.goto("https://example.com")
        print(await page.title())

asyncio.run(main())
```

### 4. Connect Playwright (Node.js)

```javascript
const { chromium } = require("playwright");

(async () => {
  const browser = await chromium.connectOverCDP(
    "http://localhost:8970/api/profiles/<profile-id>/cdp"
  );
  const context = browser.contexts()[0];
  const page = context.pages()[0];
  await page.goto("https://example.com");
  console.log(await page.title());
})();
```

### 5. Multi-profile automation

A common pattern: launch several profiles, connect to each, and run tasks in parallel.

```python
import asyncio
import httpx
from playwright.async_api import async_playwright

BASE = "http://localhost:8970"

async def run_task(pw, profile_id: str, url: str):
    browser = await pw.chromium.connect_over_cdp(f"{BASE}/api/profiles/{profile_id}/cdp")
    page = browser.contexts[0].pages[0]
    await page.goto(url)
    title = await page.title()
    print(f"[{profile_id[:8]}] {title}")

async def main():
    async with httpx.AsyncClient() as client:
        # Get all profiles
        resp = await client.get(f"{BASE}/api/profiles")
        profiles = resp.json()

        # Launch all stopped profiles
        for p in profiles:
            if p["status"] == "stopped":
                await client.post(f"{BASE}/api/profiles/{p['id']}/launch")

        # Wait for browsers to start
        await asyncio.sleep(3)

    async with async_playwright() as pw:
        tasks = [run_task(pw, p["id"], "https://example.com") for p in profiles]
        await asyncio.gather(*tasks)

asyncio.run(main())
```

## Important Notes

- A profile must be **stopped** before it can be deleted or have its settings changed.
- Each running profile uses ~512 MB RAM. Plan resource allocation accordingly.
- The VNC viewer is accessible in the web GUI at `http://localhost:8970` — each running profile shows a live browser view.
- CDP URL format: `http://localhost:8970/api/profiles/{id}/cdp` — this is the URL you pass to `connect_over_cdp()`.
- Proxy formats accepted: `http://user:pass@host:port`, `socks5://host:port`, or shorthand `host:port:user:pass`.
- When checking if a profile is running, use the `status` field from GET `/api/profiles` (returns `"running"` or `"stopped"`) or GET `/api/profiles/{id}/status`.

## Troubleshooting

- **"Profile is already running"** — The profile was already launched. Use `/stop` first, then `/launch` again.
- **CDP connection fails** — Make sure the profile is in `"running"` state. It takes 1-2 seconds after launch for CDP to become available.
- **Profile won't delete** — Stop it first with POST `/api/profiles/{id}/stop`.
- **Proxy connection errors** — Validate proxy format. Use `http://user:pass@host:port` or `socks5://user:pass@host:port`.

## When to Use This Skill vs. agent-browser

Use **this skill** when working with CloakBrowser-Manager's own API (managing profiles, launching browsers, connecting via CDP). Use **agent-browser** when you need general browser automation that doesn't involve CloakBrowser specifically. If the user wants to automate a CloakBrowser profile, use this skill to set up the connection, then write the Playwright/Puppeteer script for the actual automation logic.
