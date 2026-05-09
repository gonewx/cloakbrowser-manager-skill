# CloakBrowser-Manager API Reference

Complete REST API reference. Base URL: `http://<host>:<port>` (user-configured, default `http://localhost:8080`)

## Table of Contents

1. [Authentication](#authentication)
2. [Profiles CRUD](#profiles-crud)
3. [Profile Lifecycle](#profile-lifecycle)
4. [Clipboard](#clipboard)
5. [CDP (Chrome DevTools Protocol)](#cdp)
6. [System Status](#system-status)
7. [Response Models](#response-models)

## Authentication

By default, no auth is required (local use). When `AUTH_TOKEN` env var is set:

- API: `Authorization: Bearer <token>` header
- Web UI: login page with token
- Exempt endpoints: `/api/auth/status`, `/api/auth/login`, `/api/status`

### GET /api/auth/status

Returns whether auth is enabled and if the current request is authenticated.

### POST /api/auth/login

```json
{"token": "your-secret-token"}
```

Sets `auth_token` cookie on success. Returns `{"authenticated": true}`.

### POST /api/auth/logout

Clears the auth cookie.

## Profiles CRUD

### GET /api/profiles

Returns `ProfileResponse[]` — all profiles with their current status.

### POST /api/profiles

Create a new profile.

**Required fields:**
- `name` (string)

**Optional fields:**

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| fingerprint_seed | int | random 10000-99999 | Deterministic fingerprint generation |
| proxy | string | null | `http://`, `https://`, `socks5://`, or `host:port:user:pass` |
| timezone | string | null | IANA timezone e.g. `America/New_York` |
| locale | string | null | BCP-47 e.g. `en-US` |
| platform | string | `"windows"` | `"windows"` \| `"macos"` \| `"linux"` |
| user_agent | string | null | Custom User-Agent (auto-generated if null) |
| screen_width | int | 1920 | |
| screen_height | int | 1080 | |
| gpu_vendor | string | null | Custom WebGL vendor |
| gpu_renderer | string | null | Custom WebGL renderer |
| hardware_concurrency | int | null | navigator.hardwareConcurrency |
| humanize | bool | false | Human-like mouse/keyboard |
| human_preset | string | `"default"` | `"default"` \| `"careful"` |
| headless | bool | false | No VNC display |
| geoip | bool | false | Auto timezone/locale from proxy IP |
| clipboard_sync | bool | true | Sync clipboard between host and browser |
| auto_launch | bool | false | Auto-start on Manager boot |
| color_scheme | string | null | `"light"` \| `"dark"` \| `"no-preference"` |
| launch_args | string[] | [] | Extra Chromium flags |
| notes | string | null | Free-text notes |
| tags | TagCreate[] | [] | `[{"tag": "name", "color": "#hex"}]` |

Returns `ProfileResponse` (201).

### GET /api/profiles/{profile_id}

Returns `ProfileResponse`.

### PUT /api/profiles/{profile_id}

Update profile settings. All fields optional — only send what you want to change. Profile must be stopped.

To clear a field, send `null` explicitly (e.g., `{"proxy": null}`).

Returns updated `ProfileResponse`.

### DELETE /api/profiles/{profile_id}

Delete a profile and its data. Profile must be stopped first.

Returns `{"detail": "deleted"}`.

## Profile Lifecycle

### POST /api/profiles/{profile_id}/launch

Launch the browser instance. Returns:

```json
{
  "profile_id": "uuid",
  "status": "running",
  "vnc_ws_port": 6100,
  "display": ":100",
  "cdp_url": "ws://localhost:5100"
}
```

### POST /api/profiles/{profile_id}/stop

Stop the browser instance. Returns `{"detail": "stopped"}`.

### GET /api/profiles/{profile_id}/status

```json
{
  "status": "running",
  "vnc_ws_port": 6100,
  "display": ":100",
  "cdp_url": "ws://localhost:5100"
}
```

## Clipboard

### POST /api/profiles/{profile_id}/clipboard

Set clipboard text for a running profile.

```json
{"text": "content to paste"}
```

Max 1 MB.

### GET /api/profiles/{profile_id}/clipboard

Returns `{"text": "current clipboard content"}`.

## CDP

### GET /api/profiles/{profile_id}/cdp

Returns CDP connection info. The URL to use with Playwright/Puppeteer `connect_over_cdp()` is:

```
http://localhost:8970/api/profiles/{profile_id}/cdp
```

The Manager proxies the WebSocket connection to the browser's internal CDP port.

### WebSocket /api/profiles/{profile_id}/cdp

WebSocket endpoint for CDP protocol. Used internally by `connect_over_cdp()`.

### GET /api/profiles/{profile_id}/cdp/json/version

CDP version info (proxied from browser).

### GET /api/profiles/{profile_id}/cdp/json/list

List of CDP targets (tabs/pages) in the profile.

## System Status

### GET /api/status

```json
{
  "running_count": 3,
  "binary_version": "133.0.6943.0",
  "profiles_total": 10
}
```

No authentication required (used for Docker healthcheck).

## Response Models

### ProfileResponse

```json
{
  "id": "uuid-string",
  "name": "Profile Name",
  "fingerprint_seed": 42567,
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
  "color_scheme": null,
  "launch_args": [],
  "notes": null,
  "user_data_dir": "/data/profiles/uuid-string",
  "created_at": "2026-01-01T00:00:00+00:00",
  "updated_at": "2026-01-01T00:00:00+00:00",
  "tags": [{"tag": "work", "color": "#3b82f6"}],
  "status": "stopped",
  "vnc_ws_port": null,
  "cdp_url": null
}
```
