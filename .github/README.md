# SFTi DevBridge

**Live iOS PWA ↔ AI Agent Diagnostic Runtime**

A self-contained diagnostic plugin that gives an AI agent real-time observability into a PWA running on an iPhone over LAN. Console intercept, network tracing, storage inspection, screenshot capture, and remote JS evaluation — all through a card-based task queue.

## Quick Start

```bash
# Activate the self-contained venv
source DevBridge/.venv/bin/activate

# Install dependencies (first time only)
pip install -r DevBridge/ai.server/requirements.txt

# Start the bridge server (logs write to DevBridge/server.log automatically)
python3 DevBridge/ai.server/server.py
# → 📡 LAN URL: http://192.168.x.x:8765

# Open on iPhone: visit the LAN URL in Safari
# Tap Share → Add to Home Screen → Open from Home Screen
```

## Integrating DevBridge into a Host PWA

DevBridge has two roles: the **dashboard** (served from port 8765) and the **bridge client** (`bridge.js`, injected into the host app). When integrating into a host app running on a **different origin** (e.g., port 8080), the bridge client must know the DevBridge server URL.

### How bridge.js detects the server URL

`bridge.js` uses the following priority chain:

1. **`window.__DEVBRIDGE_URL`** — Explicit global set before the script loads
2. **`data-server` attribute** — On the `<script>` tag itself
3. **Script `src` URL** — Parses the origin from wherever `bridge.js` was loaded
4. **`window.location.origin`** — Last resort fallback (same-origin only)

> ⚠️ **Important:** `document.currentScript` is `null` for dynamically-created `<script>` elements. When injecting `bridge.js` dynamically, the script finds itself by scanning for `script[src*="bridge.js"]` in the DOM. This works reliably, but methods 1 and 2 are more explicit.

### Method 1 — Dynamic injection with auto-detection (Recommended)

This is the recommended pattern for host apps. The bridge URL is auto-detected from the page's hostname, assuming DevBridge runs on port 8765 on the same machine:

```html
<!-- Add before </body> in your host app's HTML -->
<script>
  (function() {
    // Auto-detect: use the same host that served this page, port 8765
    const autoUrl = `${window.location.protocol}//${window.location.hostname}:8765`;
    const url = new URLSearchParams(window.location.search).get('devbridge') || autoUrl;
    const script = document.createElement('script');
    script.src = `${url}/bridge.js`;
    script.onerror = function() { console.warn('[DevBridge] Failed to load bridge.js from', url); };
    document.body.appendChild(script);
  })();
</script>
```

This works for iOS homescreen PWAs, which strip query parameters. The `?devbridge=` param is an optional override.

### Method 2 — Static script tag with `data-server`

```html
<script src="http://<LAN_IP>:8765/bridge.js" data-server="http://<LAN_IP>:8765"></script>
```

### Method 3 — Global override

```html
<script>
  window.__DEVBRIDGE_URL = 'http://192.168.1.163:8765';
</script>
<script src="http://192.168.1.163:8765/bridge.js"></script>
```

### Same-origin (dashboard.html)

When bridge.js is served from the same origin as the DevBridge server (e.g. inside `dashboard.html`), no configuration is needed:

```html
<script src="bridge.js"></script>
```

The script detects the server URL from its own `src` attribute automatically.

### Adding a DevBridge link in the host app

In the app's settings menu or navigation dock, add an entry that opens the DevBridge dashboard:

```html
<!-- Option A: open in browser (fullscreen diagnostic view) -->
<a href="http://<LAN_IP>:8765/" target="_blank" rel="noopener">
  DevBridge Diagnostics
</a>

<!-- Option B: auto-detect URL from the same hostname -->
<script>
  window.openDevBridge = function() {
    const url = `${window.location.protocol}//${window.location.hostname}:8765`;
    window.open(url, '_blank');
  };
</script>
<button onclick="openDevBridge()">Diagnostics</button>
```

When DevBridge is not needed, remove the `<script>` tag. Zero runtime overhead when disabled.

## Architecture

```
DevBridge/
├── .venv/                          Self-contained Python environment (portable)
├── .pyre_configuration             IDE type-check config
├── server.log                      Server output (written here automatically)
├── ai.server/
│   ├── server.py                   API endpoints + SSE + static file serving
│   └── libs/                       Vendored Python dependencies
├── client/
│   ├── dashboard.html              Main entry point (Liquid Glass UI)
│   ├── bridge.js                   Telemetry capture + card polling + card execution
│   ├── manifest.json               PWA manifest
│   ├── sw.js                       Service worker (pass-through, no caching)
│   └── tabs/                       Diagnostic views
│       ├── telemetry.html          Device/runtime stats
│       ├── cards.html              Card execution history
│       ├── storage.html            localStorage/sessionStorage inspector
│       └── network.html            Real-time latency + resource timing
├── hu.ui/                          UI layer (neural mesh canvas, effects config, CSS)
└── ico/                            SVG icon assets
```

## API

| Endpoint | Method | Purpose |
|---|---|---|
| `/health` | GET | Server status + LAN IP |
| `/log` | POST | Receive telemetry from device |
| `/logs` | GET | Latest N logs |
| `/poll` | GET | Device polls for next card |
| `/card` | POST | Agent pushes a task card |
| `/cards` | GET | Full queue state |
| `/stream` | GET | SSE live feed |

## Card Types

| Type | What It Does |
|---|---|
| `eval` | Execute JS in browser context |
| `screenshot` | Capture viewport as PNG |
| `test` | Run JS assertions, return pass/fail |
| `reload` | Force `location.reload()` |
| `fetch` | Make a network request from device |
| `storage_read` / `storage_write` | Inspect/modify localStorage |
| `click_element` | Click a DOM element by CSS selector |
| `query_selector` | Inspect a DOM element by CSS selector |
| `csa_state` | Dump the CSA global state object |
| `webrtc_check` | Check WebRTC/media device capabilities |

## Full Documentation

See [`.github/instructions/anithravity.build.md`](.github/instructions/anithravity.build.md) for the complete technical reference including safety mechanisms, iOS constraints, and agent workflow.
