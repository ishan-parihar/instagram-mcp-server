# Instagram MCP Server

<p align="left">
  <a href="https://pypi.org/project/instagram-scraper-mcp/" target="_blank"><img src="https://img.shields.io/pypi/v/instagram-scraper-mcp?color=blue" alt="PyPI Version"></a>
  <a href="https://github.com/stickerdaniel/instagram-mcp-server/actions/workflows/ci.yml" target="_blank"><img src="https://github.com/stickerdaniel/instagram-mcp-server/actions/workflows/ci.yml/badge.svg?branch=main" alt="CI Status"></a>
  <a href="https://github.com/stickerdaniel/instagram-mcp-server/actions/workflows/release.yml" target="_blank"><img src="https://github.com/stickerdaniel/instagram-mcp-server/actions/workflows/release.yml/badge.svg?branch=main" alt="Release"></a>
  <a href="https://github.com/stickerdaniel/instagram-mcp-server/blob/main/LICENSE" target="_blank"><img src="https://img.shields.io/badge/License-Apache%202.0-%233fb950?labelColor=32383f" alt="License"></a>
  <img src="https://img.shields.io/badge/Python-3.12+-blue" alt="Python Version">
</p>

**The Problem**: Instagram has no official public API for profile scraping, insights, or messaging. AI assistants (`Claude`, `Cursor`, `Windsurf`) need structured Instagram data — profiles, posts, reels, DMs, business insights — but Instagram's front end is a moving target. Hashed CSS classes change weekly. Rate limits and auth barriers are aggressive. A naive `selenium` script breaks within days.

**What this is**: A purpose-built MCP server that treats Instagram's web client as an *adversarial data source*. It uses a custom browser orchestration engine that survives DOM churn, manages browser sessions across three runtime modes (legacy, CDP, Docker), and serializes tool execution to prevent Instagram's session state from corrupting.

**Engineering highlights**: innerText-based extraction (zero DOM selector dependence), state-machine driven bootstrap with background retries, sequential tool middleware to prevent concurrent navigation conflicts, and a rich error diagnostic system that maps 15+ Instagram failure modes to actionable messages.

---

## The Problem

Instagram's web app is engineered *against* automation:

- **No public API** for profiles, reels, DMs, or business insights. Everything must go through the web client.
- **Hashed CSS classes** (`x1n2onr6`, `x1lliihq`, etc.) change with every deploy. Any scraper relying on DOM selectors breaks within weeks.
- **Concurrent page states** interfere. If two tools navigate Instagram simultaneously (e.g., "fetch profile" and "send DM"), the session enters an inconsistent state and Instagram forces re-authentication.
- **Rate limits and auth barriers** are invisible — no HTTP status code. They appear as in-page "blocked" overlays, requiring DOM-level detection.
- **Session persistence** across restarts requires cookie extraction from browser-native SQLite stores, which differ by browser (Chrome, Firefox, Brave, Edge, etc.).

Existing solutions (Puppeteer wrappers, static scrapers) fail because they couple extraction logic to ephemeral layout details, or lack the state management needed for multi-tool AI workflows.

---

## Engineering Highlights

### 1. DOM-Agnostic Extraction Engine

**Anti-pattern**: Most scrapers use CSS selectors like `div.x1n2onr6 > span._aacl`. When Instagram rotates these classes, the scraper silently returns empty data.

**Solution**: The extractor (`scraping/extractor.py`) relies on **`innerText` and URL navigation**, not DOM selectors. Each "section" (`posts`, `reels`, `followers`) maps to exactly one page navigation (`USER_SECTIONS` dict). Extraction reads visible text, then strips Instagram's chrome (footer links, sidebar noise) using regex markers:

```python
# Noise markers strip Instagram chrome instead of brittle CSS selectors
_NOISE_MARKERS = [
    re.compile(r"^About\n+(?:Help|Press|API|Jobs|Terms|Privacy)", re.MULTILINE),
    re.compile(r"^© \d{4} Instagram from Meta$", re.MULTILINE),
    re.compile(r"^Suggested for you$", re.MULTILINE),
]
```

This approach survives layout changes because Instagram cannot hide text content from the user without breaking its own UX — and `innerText` reads exactly what a human sees.

**Key constraint**: "One section = one navigation." Each section triggers exactly one `page.goto()`. Combining multiple data sources into a single navigation creates coupling that breaks when Instagram reorganized pages.

### 2. Three-Mode Browser Architecture

The server supports three distinct runtime policies, each solving a different operational constraint:

| Mode | Entry | Cookie Source | Use Case |
|------|-------|--------------|----------|
| **Legacy** | Default | SQLite cookie store detection | Desktop AI clients |
| **CDP** | `--cdp` flag | Live Brave browser session | Users who don't want cookie extraction |
| **Docker** | `Dockerfile` | Pre-exported portable auth (`tar` archive) | Server/headless deployments |

The **CDP bridge** (`drivers/browser._bridge_runtime_profile()`) is the most architecturally interesting: it connects to a running Brave browser via Chrome DevTools Protocol, imports cookies from the persistent source session into an ephemeral runtime profile, and isolates scraping in a separate browser context — zero interference with the user's actual browsing.

The **Docker runtime** solves the "headless auth" problem: Instagram's anti-bot checks detect missing GPU/display. The solution creates an authenticated profile on the host via `--login`, then `tar` archives it for injection into the container. The container never handles login — just cookie replay.

### 3. Sequential Tool Execution Middleware

Instagram's web app is a single-page application with mutable global state. If two MCP tool calls navigate Instagram simultaneously, the following happens:

1. Tool A navigates to `instagram.com/natgeo/posts/`
2. Tool B navigates to `instagram.com/direct/inbox/`
3. Instagram's SPA state corrupts — neither tool gets valid data
4. Instagram detects the "impossible" navigation pattern and forces re-login

**Solution**: An `asyncio.Lock`-based middleware (`SequentialToolExecutionMiddleware`) serializes all MCP tool calls within the same server process. Each tool waits in a queue, acquires the lock, executes, and releases:

```python
class SequentialToolExecutionMiddleware(Middleware):
    async def on_call_tool(self, context, call_next):
        async with self._lock:  # Only one navigation at a time
            return await call_next(context)
```

This is a middleware registered at server creation, not per-tool — zero tool implementation changes.

### 4. Bootstrap & Authentication State Machine

The bootstrap system (`bootstrap.py`) manages a complex initialization lifecycle across process restarts:

```
IDLE → SETUP_IN_PROGRESS → READY
           ↓ (failed)
         FAILED → (background retry) → SETUP_IN_PROGRESS
```

The auth state machine runs independently:

```
UNKNOWN → CHECKING → READY
   ↓ (expired)          ↓ (expired)
INVALID → RELOGIN_IN_PROGRESS → READY
```

Key design decisions:
- **Background-first browser setup**: `patchright` Chromium downloads in a background task, not at startup. Tools become available immediately; if the browser isn't ready, they raise `BrowserSetupInProgressError` with a "retry in a few minutes" message.
- **Cookie bridge, not credential store**: No Instagram passwords are stored. Auth state is purely cookie-based, extracted from the host browser's SQLite store.
- **Auto-relogin on expiry**: When Instagram invalidates a session mid-flight, the system detects it via `detect_auth_barrier()` and triggers a fresh login flow — no manual intervention.

### 5. Rich Error Diagnostics

A naive scraper returns HTTP 200 with a "login required" page, and the AI client has no way to understand what happened.

**Solution**: A centralized `raise_tool_error()` function maps 15+ Instagram-specific exception types to user-friendly `ToolError` messages with **auto-generated diagnostics**:

```python
except AuthenticationError:
    raise ToolError(
        "Authentication failed. Run with --login to re-authenticate."
    ) from exception
except RateLimitError:
    raise ToolError(
        f"Rate limit detected. Wait {exception.suggested_wait_time}s before retrying."
    ) from exception
```

Each diagnostic includes an **issue template path** — a markdown file in `docs/` that provides context-specific troubleshooting. This turns opaque scraping failures into actionable guidance.

---

## Architecture Overview

```
MCP Client (Claude, Cursor, etc.)
        │
  ┌─────▼──────┐
  │  FastMCP   │  ← MCP protocol (stdio or streamable-http)
  │  Server    │
  └─────┬──────┘
        │
  ┌─────▼──────────────┐
  │ SequentialToolExec │  ← asyncio.Lock middleware
  │ Middleware          │     serializes all navigations
  └─────┬──────────────┘
        │
  ┌─────▼──────────────────┐
  │    Tool Registry        │  ← 28+ tools across 7 categories
  │ (user / posts / search  │
  │  / insights / messaging │
  │  / actions / gemini)    │
  └─────┬──────────────────┘
        │
  ┌─────▼────────────────────────┐
  │   Bootstrap & Auth State      │
  │   Machine                     │  ← manages browser lifecycle
  └─────┬────────────────────────┘
        │
  ┌─────▼──────────────────────┐
  │  Browser Orchestrator       │
  │  (Legacy / CDP / Docker)    │  ← three runtime modes
  └─────┬──────────────────────┘
        │
  ┌─────▼──────────────┐
  │  innerText          │
  │  Extraction Engine  │  ← zero DOM selector dependence
  └────────────────────┘
```

---

## Tech Stack

| Technology | Purpose |
|------------|---------|
| **Python 3.12+** | Type-safe async runtime with structural pattern matching |
| **FastMCP 3.x** | MCP protocol server — enables stdio and streamable-http transports |
| **Patchright** | Anti-detection Playwright fork — bypasses Instagram's webdriver checks |
| **asyncio.Lock + Middleware** | Serialized tool execution — prevents concurrent navigation corruption |
| **Gemini 2.0 Flash** | Multimodal reel analysis — video-to-text without local Whisper |
| **Docker** | Headless deployment — portable auth via cookie archives |
| **Ruff / Ty** | Strict linting (Ruff) and type checking (Ty, not mypy) |

---

## Quick Start

```bash
# Install via uvx (no local install needed)
uvx instagram-scraper-mcp
```

On first tool call, a login window opens. Log in once; cookies persist across restarts.

See [MCP Client Configuration](#mcp-client-configuration) for IDE setup.

### Advanced Configurations

| Mode | Command |
|------|---------|
| **CDP (Brave)** | `uvx instagram-scraper-mcp --cdp` |
| **Docker** | See [docs/docker-hub.md](docs/docker-hub.md) |
| **Gemini Analysis** | `GEMINI_API_KEY=xxx uvx instagram-scraper-mcp` |
| **Debug Mode** | `uvx instagram-scraper-mcp --log-level DEBUG --no-headless` |

---

## Tool Suite

| Category | Tools | Capabilities |
|----------|-------|-------------|
| **Profile & Content** | `get_user_profile`, `get_user_posts`, `get_user_reels`, `get_user_stories`, `get_user_highlights`, `get_post_details` | Posts, reels, stories, highlights, followers/following lists |
| **Search & Discovery** | `search_users`, `search_hashtags`, `search_locations`, `get_hashtag_posts`, `get_location_posts` | User, hashtag, and location search |
| **Messaging & Actions** | `get_direct_inbox`, `get_dm_conversation`, `send_dm`, `follow_user`, `unfollow_user`, `like_post`, `unlike_post`, `save_post`, `comment_on_post` | Full DM and engagement suite |
| **Business Insights** | `get_business_insights`, `get_audience_insights`, `get_content_insights`, `get_activity_insights` | Reach, impressions, demographics (Business/Creator accounts only) |
| **AI Analysis** | `analyze_reel_with_gemini`, `bulk_analyze_reels_with_gemini` | Multimodal reel transcription and analysis via Gemini 2.0 Flash |
| **Transcription** | `transcribe_user_reels`, `transcribe_reel` | Local Whisper-based SRT subtitle generation |

---

## Potentialities

- **Headless auth recovery**: Currently Docker runtime requires host-side `--login`. A self-service web portal for one-time auth token generation would eliminate this friction.
- **Multi-account session management**: The sequential middleware prevents intra-session conflicts, but switching between Instagram accounts requires a separate profile. Native account switching would enable parallel multi-account extraction.
- **Webhook-based rate limit mitigation**: Rate limits are currently synchronous (wait N seconds). An async queue with webhook callbacks would allow batch processing without tool timeouts.
- **GraphQL API fallback**: Instagram's internal GraphQL API occasionally surfaces in responses. A hybrid strategy (extraction + API probes) could reduce page navigations for known-stable endpoints.

---

## License & Acknowledgements

Licensed under the [Apache 2.0 License](LICENSE).

Built with [FastMCP](https://gofastmcp.com/) and [Patchright](https://github.com/Kaliiiiiiiiii-Vinyzu/patchright-python).

Use in accordance with [Instagram's Terms of Use](https://help.instagram.com/581066165581870). Web scraping may violate Instagram's terms. This tool is for personal use only.
---
Developed by [Ishan Parihar](https://github.com/ishan-parihar) — If you find this useful, [consider supporting](https://rzp.io/rzp/ishan-parihar)
