# Salesforce MCP Server — Auto-Auth From Chrome

[![PyPI](https://img.shields.io/pypi/v/salesforce-mcp-auto-auth-chrome?label=pypi&color=blue)](https://pypi.org/project/salesforce-mcp-auto-auth-chrome/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Python 3.12+](https://img.shields.io/badge/python-3.12+-blue.svg)](https://www.python.org/downloads/)
[![macOS](https://img.shields.io/badge/platform-macOS-lightgrey.svg)](#)

A local MCP server that exposes **14 Salesforce tools** (SOQL, SOSL, CRUD, Apex execute, tooling API, REST) inside Claude Desktop — and **auto-refreshes the session from your Chrome browser** so you never have to paste a token into your config again. Wraps [`mcp-salesforce-connector`](https://pypi.org/project/mcp-salesforce-connector/) on PyPI; runs as a stdio MCP launched per-org by Claude Desktop.

No connected app, no OAuth dance, no copy-pasting session ids — just stay logged into the org in Chrome.

---

## How it fits together

```
Claude Desktop ── stdio ──▶ this package (Python) ── HTTPS ──▶ Salesforce REST API
       │                            │
       │                            └─ reads `sid` cookie ──▶ Chrome cookies on disk
       │                                  on every API call         (macOS Keychain)
       │
       └── one entry in claude_desktop_config.json per Salesforce org
```

The package is a thin Python shim around `mcp-salesforce-connector`. On every API call, it reads the `sid` cookie that Chrome stores when you're logged into Salesforce, and uses it as the `Authorization` header. That cookie value is identical to a Salesforce session id and is accepted as a Bearer token by the REST API. As long as you keep a Chrome tab open on the org, the session stays alive and Claude inherits the freshness.

---

## Files in this repo

- **`src/salesforce_mcp_auto_auth_chrome/__main__.py`** — entry point. Reads `SALESFORCE_INSTANCE_URL` from env, seeds a placeholder access token, installs the per-call patch, and hands off to `mcp-salesforce-connector`.
- **`src/salesforce_mcp_auto_auth_chrome/patch.py`** — monkey-patches `simple_salesforce.Salesforce._call_salesforce` so every API call gets a fresh `sid` from Chrome.
- **`src/salesforce_mcp_auto_auth_chrome/auth.py`** — reads the `sid` cookie from Chrome via `pycookiecheat`. Returns `None` on any failure so callers can defer the error to tool-call time.
- **`pyproject.toml`** — package metadata + entry point. `uvx` reads this when launching.
- **`examples/claude_desktop_config.example.json`** — copy-paste-ready Claude Desktop config snippet.
- **`docs/how-it-works.md`** — full architecture write-up with the design decisions, what we tried and discarded, and the lessons that generalize to other MCP wrappers.

---

## Setup — Part 1: Add to Claude Desktop (~1 min, one-time)

### Fastest path — have Claude do it for you

If Claude Desktop already has filesystem access to your home directory (Cowork mode users do by default), you don't need to touch JSON at all. Just paste the following prompt into a fresh Claude chat — edit the org name and URL to taste:

```
Add a Salesforce MCP entry to my Claude Desktop config named
`salesforce-mydevorg` for the org at
`https://yourdomain.my.salesforce.com`. The MCP command should be
`uvx salesforce-mcp-auto-auth-chrome` (it's on PyPI). Don't set a
SALESFORCE_ACCESS_TOKEN — the wrapper reads it from Chrome. Make a
backup of my existing config first.
```

Claude reads your existing `claude_desktop_config.json`, adds the new server entry alongside any existing ones, backs up the original, and tells you when to restart. When Claude confirms it's done, skip to **Restart Claude Desktop** below.

### Alternate route — edit `claude_desktop_config.json` yourself

If you'd rather edit JSON by hand:

**1. Open your Claude Desktop config.** On macOS the file lives at:

```
~/Library/Application Support/Claude/claude_desktop_config.json
```

**2. Add one entry per Salesforce org.**

```jsonc
{
  "mcpServers": {
    "salesforce-mydevorg": {
      "command": "uvx",
      "args": ["salesforce-mcp-auto-auth-chrome"],
      "env": {
        "SALESFORCE_INSTANCE_URL": "https://yourdomain.my.salesforce.com"
      }
    }
  }
}
```

Replace `yourdomain.my.salesforce.com` with your actual My Domain (the URL host when you're logged into Salesforce — everything before `/lightning`). **Don't set `SALESFORCE_ACCESS_TOKEN`** — the wrapper pulls it from Chrome.

To add multiple orgs, repeat the block under a different key — `salesforce-prod`, `salesforce-sandbox`, `salesforce-dev1`, etc. Each gets its own `SALESFORCE_INSTANCE_URL`. See [`examples/claude_desktop_config.example.json`](examples/claude_desktop_config.example.json) for a complete template.

### Restart Claude Desktop

Cmd+Q (a full quit — not just closing the window) and reopen. Claude Desktop reads the config at startup.

### Sanity check

In a new Claude conversation, ask: *"using salesforce-mydevorg, what's the org name?"* — Claude will call the `run_soql_query` tool with `SELECT Name FROM Organization`. If it returns your org's name, the wiring is correct.

The first time the wrapper runs, macOS will prompt for Keychain access (the dialog says *"Claude wants to use your confidential information stored in 'Chrome Safe Storage'"*). Click **Always Allow** to suppress future prompts.

---

## Setup — Part 2: Stay logged in to Salesforce in Chrome

This is the "config" for this MCP: just keep a Chrome tab open on your Salesforce org. The wrapper reads cookies from Chrome on every API call — if you log out (or your session times out without any browser activity), the next tool call returns:

> *Not logged into Salesforce in Chrome for https://yourdomain.my.salesforce.com. Open that org in Chrome, sign in, then retry.*

Sign back in and the next tool call works again — no need to restart Claude Desktop.

---

## The 14 tools

All 14 come from the underlying [`mcp-salesforce-connector`](https://pypi.org/project/mcp-salesforce-connector/) — this package just adds auto-auth on top.

### Query (2)

| Tool | Purpose |
| --- | --- |
| `run_soql_query` | Run any SOQL `SELECT` against the org |
| `run_sosl_search` | Cross-object full-text search via SOSL |

### Records — single (5)

| Tool | Purpose |
| --- | --- |
| `get_record` | Read one record by Id from any SObject |
| `create_record` | Insert one record |
| `update_record` | Update one record by Id |
| `delete_record` | Delete one record by Id |
| `get_object_fields` | Describe an SObject's fields, picklists, relationships |

### Records — bulk (3)

| Tool | Purpose |
| --- | --- |
| `bulk_create_records` | Bulk insert via the Composite REST API |
| `bulk_update_records` | Bulk update via the Composite REST API |
| `bulk_delete_records` | Bulk delete via the Composite REST API |

### Code & metadata (3)

| Tool | Purpose |
| --- | --- |
| `apex_execute` | Run anonymous Apex |
| `tooling_execute` | Tooling API access (deploy, query metadata, etc.) |
| `list_sobjects` | List every standard + custom SObject |

### Raw REST (1)

| Tool | Purpose |
| --- | --- |
| `restful` | Make a raw REST call against any Salesforce endpoint |

Each tool's full schema is advertised through the MCP `tools/list` method — Claude reads it automatically.

---

## How auth actually flows

```
You: "salesforce-mydevorg, run SELECT Name FROM Account LIMIT 5"
  │
  ▼
Claude Desktop → stdio → salesforce-mcp-auto-auth-chrome process (Python)
  │
  ▼
mcp-salesforce-connector calls simple_salesforce.Salesforce.query()
  │
  ▼
simple_salesforce calls Salesforce._call_salesforce()  ← OUR MONKEY-PATCH RUNS HERE
  │
  ├─ Read Chrome's `sid` cookie for kuga-dev-ed.my.salesforce.com
  │  (decrypts Chrome's cookie store using macOS Keychain)
  │
  ├─ If sid present → overwrite session_id + Authorization header
  ├─ If sid absent → raise RuntimeError → surfaces as tool error in chat
  │
  ▼
HTTPS GET https://kuga-dev-ed.my.salesforce.com/services/data/v59.0/query?q=...
  Authorization: Bearer <fresh sid from Chrome>
  │
  ▼
Salesforce returns rows → MCP tool result → Claude → you
```

The patch is class-level, runs once at startup, and is invisible to the connector. Upstream improvements to `mcp-salesforce-connector` flow through without changes here.

---

## Why a local wrapper (vs. alternatives we tried)

- **A remote MCP (Cloudflare Worker, etc.) can't read your local Chrome cookies.** The whole value here is "use my browser's session as the API session" — that requires local file-system access to Chrome's encrypted cookie store. A remote server would still need you to paste a token.
- **Forking `mcp-salesforce-connector` to add a `--auth-from-chrome` flag** means maintaining a fork forever and diverging from upstream. Monkey-patching at the `simple_salesforce._call_salesforce` boundary stays out of the way.
- **OAuth (connected app per org)** is the "right" answer for production automation but is heavy for sandbox/dev work where you change orgs frequently and the session token IS what you want. For prod, stick with the standard connector's OAuth config.
- **Caching the `sid` for N minutes** sounds smart but causes more stale-token errors than the per-call read adds latency. Reading from Chrome's encrypted cookie store costs sub-100ms. We do it per call.
- **Failing loudly at startup** when no session was found gave annoying "Server disconnected" warnings every time you restarted Claude Desktop with orgs you weren't actively using. We now defer auth errors to the moment a tool is invoked.

The full architecture write-up — including each of these and how we picked the current shape — is in [`docs/how-it-works.md`](docs/how-it-works.md).

---

## Local development

If you want to test changes before pushing:

```bash
# Install Python 3.12+ and uv if you don't have them
curl -LsSf https://astral.sh/uv/install.sh | sh

# Clone and run from source
git clone https://github.com/kugamon/salesforce-mcp-auto-auth-chrome.git
cd salesforce-mcp-auto-auth-chrome
uv sync

# Smoke-test against an org you're logged into in Chrome
SALESFORCE_INSTANCE_URL=https://yourdomain.my.salesforce.com \
  uv run python -m salesforce_mcp_auto_auth_chrome
```

The process will start, print `[salesforce-mcp-auto-auth-chrome v0.1.1] Ready for ...`, and wait for MCP JSON-RPC on stdin.

To point your local Claude Desktop config at the working copy instead of the published repo, change the args to:

```jsonc
"args": ["run", "--directory", "/absolute/path/to/your/checkout", "python", "-m", "salesforce_mcp_auto_auth_chrome"]
```

---

## Behavior reference

### At Claude Desktop startup

| Logged into org in Chrome? | What happens |
| --- | --- |
| Yes | MCP server starts, all tools register, first call uses fresh `sid` |
| No | MCP server **still starts** — no "Server disconnected" warning. Errors surface only when you actually call a tool. |

### When you invoke a tool

| Logged into org in Chrome? | What happens |
| --- | --- |
| Yes | A fresh `sid` is read from Chrome and used for that call. Result returns normally. |
| No | Tool call returns: *"Not logged into Salesforce in Chrome for https://… . Open that org in Chrome, sign in, then retry."* |

### Token refresh cadence

Every API call. The `sid` from Chrome is read fresh each time — Salesforce sessions effectively never expire mid-conversation as long as your Chrome tab is still alive.

---

## Troubleshooting

**Claude shows "Server disconnected" at startup**: Almost always means the wrapper failed to import something — usually `pycookiecheat` or `mcp-salesforce-connector`. Look at `~/Library/Logs/Claude/mcp-server-<name>.log` for the actual Python traceback. Most fixes are a `uvx --reinstall` or running `uv sync` if you're working from a local clone.

**All Salesforce MCPs disconnect at once, log shows `AttributeError: 'Server' object has no attribute 'list_tools'`**: This means `uvx` resolved the `mcp` Python SDK to **2.0.0 or later**, which removed the low-level `Server.list_tools()` decorator that `mcp-salesforce-connector` (the upstream package we wrap) still uses. **Fixed in v0.1.1+** — this package now pins `mcp<2.0` at install time. If you're on an older version, either upgrade (`uvx --reinstall salesforce-mcp-auto-auth-chrome`) or add `--with "mcp<2"` to your `uvx` args as an immediate workaround:

```jsonc
"args": ["--with", "mcp<2", "salesforce-mcp-auto-auth-chrome"]
```

**`Not logged into Salesforce in Chrome for ...`** in chat: The wrapper couldn't find a `sid` cookie for the configured instance URL. Open that org in Chrome, sign in, then retry your Claude request. No Claude restart needed — the next tool call reads cookies fresh.

**Keychain prompt keeps appearing**: The first time the wrapper reads cookies, macOS asks permission to access "Chrome Safe Storage" via Keychain. Click **Always Allow** — not "Allow" — and the prompt won't return.

**Wrong instance URL?**: Look at the URL bar when logged into Salesforce — it'll be `https://yourdomain.my.salesforce.com/lightning/...`. Use the host (before `/lightning`) as `SALESFORCE_INSTANCE_URL`. Dev sandboxes often look like `https://yourname-dev-ed.develop.my.salesforce.com`.

**`HTTP 401` from Salesforce on every call**: The `sid` cookie you have is valid for the UI session but the org may have "Lock sessions to the IP address from which they originated" enabled with a strict policy. If so, switch that org to OAuth-based auth via the standard `mcp-salesforce-connector` config.

**Want to use Firefox/Safari/Brave/Arc instead of Chrome**: Not yet — today the wrapper only checks Chrome's cookie store. Adding other browsers is a small change to `auth.py` (pycookiecheat supports several); open an issue if you need one.

**Cookie reads work in Chrome standalone but the MCP can't read them**: The MCP process needs permission to access Keychain. If you ever click "Don't Allow" on the Keychain prompt by accident, open Keychain Access → search for "Chrome Safe Storage" → right-click → "Get Info" → "Access Control" → add the `uv` executable, or just delete the entry and let Chrome recreate it.

**Want to see what the wrapper is doing?**: Logs go to stderr, which Claude Desktop captures at `~/Library/Logs/Claude/mcp-server-<name>.log`. The wrapper prints `[salesforce-mcp-auto-auth-chrome v0.1.1] Ready for ...` at startup and `cookie read failed: ...` on individual failures.

---

## Version history

- **v0.1.1** — pin `mcp<2.0` at install time. The `mcp` Python SDK shipped 2.0.0 on 2026-07-28, removing the low-level `Server.list_tools()` decorator that `mcp-salesforce-connector` 0.1.15 relies on. Before this pin, every fresh `uvx` resolution crashed at import with `AttributeError: 'Server' object has no attribute 'list_tools'`. No functional changes — this is purely a dependency guard until the upstream connector is ported to the 2.x API.
- **v0.1.0** — initial release on PyPI. 14 tools (via mcp-salesforce-connector 0.1.15). Chrome-only, macOS-only, per-call `sid` refresh.

## Contributing

This project is maintained by Kugamon. Outside pull requests aren't accepted, but issues are welcome: if sign-in or a tool fails, open an [issue](https://github.com/kugamon/salesforce-mcp-auto-auth-chrome/issues) with your macOS and Chrome versions and the error from the log (never paste session cookies or tokens).

The MIT license lets you fork and adapt it for your own use.

## License

MIT — see [LICENSE](LICENSE).
