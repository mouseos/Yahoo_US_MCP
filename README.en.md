# Yahoo US Search MCP

An MCP server that gives AI agents access to Yahoo (US) search. **No API key, no login, no configuration.**

[日本語](README.md)

[![CI](https://github.com/mouseos/Yahoo_US_MCP/actions/workflows/ci.yml/badge.svg)](https://github.com/mouseos/Yahoo_US_MCP/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

> This is for Yahoo Search US (search.yahoo.com). Yahoo! JAPAN is a different service, covered by [Yahoo-Japan-Search-MCP](https://github.com/mouseos/Yahoo-Japan-Search-MCP).

---

## Verticals

| Tool | Vertical | Per page |
|---|---|---|
| `yahoo_us_web_search` | Web search | 7 |
| `yahoo_us_video_search` | Video search | 60 |
| `yahoo_us_news_search` | News search | 10 |
| `yahoo_us_suggest` | Autocomplete suggestions | up to 20 |
| `yahoo_us_build_search_url` | Builds a search URL (no network) | — |
| `yahoo_us_list_verticals` | Lists the above | — |

Image search is not supported. Yahoo's image SERP (images.search.yahoo.com)
returned HTTP 500 over plain HTTP under every condition tested (desktop and
mobile user agents, with and without cookies), so there is no verified way
to read it — see "Why image search is missing" below.

## Install

### 1. From a release

1. Download the ZIP for your platform from [Releases](https://github.com/mouseos/Yahoo_US_MCP/releases) and extract it
2. Run `yahoo-us-mcp-launcher` and click **Install** for your agent
3. Restart the agent

The launcher detects and registers Claude Desktop, Claude Code, Google Antigravity and Codex CLI.

### 2. From the command line

```bash
# print copy-paste config
yahoo-us-mcp setup

# Claude Code
claude mcp add Yahoo_US_MCP -- /path/to/yahoo-us-mcp
```

### 3. Manual config

```json
{
  "mcpServers": {
    "Yahoo_US_MCP": {
      "command": "/path/to/yahoo-us-mcp",
      "args": []
    }
  }
}
```

Leave `env` empty — there is nothing to configure.

### 4. Build from source

```bash
git clone https://github.com/mouseos/Yahoo_US_MCP.git
cd Yahoo_US_MCP
cargo build --release
# → target/release/yahoo-us-mcp
```

Requires Rust 1.80+ (for `std::sync::LazyLock`). On Linux the launcher additionally
needs `libfontconfig1-dev libxkbcommon-dev libgtk-3-dev`; the server itself has no
system dependencies.

## Use as a Rust library

The search logic lives in the provider crate (package `yahoo-us-mcp-provider`,
imported under its lib name `yahoo_us_search`), so you can call it directly from
your own Rust program without the MCP server. No API key, login or
configuration — anonymous, exactly like the MCP server.

```toml
[dependencies]
# From GitHub
yahoo_us_search = { git = "https://github.com/mouseos/Yahoo_US_MCP", package = "yahoo-us-mcp-provider" }
# From a local checkout
# yahoo_us_search = { path = "../Yahoo_US_MCP/crates/providers/yahoo_us", package = "yahoo-us-mcp-provider" }
tokio = { version = "1", features = ["full"] }
anyhow = "1"
```

```rust
use yahoo_us_search::YahooUsSearchClient;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let client = YahooUsSearchClient::new();
    let result = client.web_search("apple pie recipe", 1, 5).await?;
    for item in &result.items {
        println!("{} — {}", item.title, item.url);
    }
    let suggest = client.suggest("app", 10).await?;
    for s in &suggest.suggestions {
        println!("{}", s.keyword);
    }
    Ok(())
}
```

Public API (everything is importable from the crate root):

| Kind | Names |
|---|---|
| Client | `YahooUsSearchClient` (`web_search` / `video_search` / `news_search` / `suggest`, all returning `anyhow::Result`) |
| Shared result types | `SearchResult<T>`, `PROVIDER`, `verticals()` |
| Per-vertical item types | `WebItem`, `VideoItem`, `NewsItem`, `SuggestResult` / `Suggestion` |
| URL building | `build_search_url` (no network) |
| Offline parsers | `parse_web_results` / `web_has_next_page`, `parse_video_results` / `parse_video_total` / `parse_video_next_offset`, `parse_news_results` / `news_has_next_page`, `parse_suggest_response` (from saved page HTML / JSON), `clean_text`, `decode_entities`, `decode_redirect_url`, `urlencode` / `urldecode` |

Result types derive serde `Serialize` / `Deserialize`, so results can be
stored as JSON and read back. Network-free examples live in
`crates/providers/yahoo_us/examples/` (`offline_basics` and
`parse_saved_pages`), and `crates/providers/yahoo_us/tests/library_api.rs`
exercises the API exactly the way a downstream user would.

When a search fails through the MCP server, the tool returns
`{provider, ok:false, error:{kind, message}}` (the form shared with Baidu_MCP
and Yahoo-Japan-Search-MCP). Argument errors (such as a missing query) still
surface as MCP `isError` results.

## Usage examples

Prompts for an agent:

- "Look up US apple orchard harvest figures" → `yahoo_us_web_search`
- "Find videos of the Apple event" → `yahoo_us_video_search`
- "Give me 5 US economy news articles" → `yahoo_us_news_search`
- "What completions does 'app' have?" → `yahoo_us_suggest`

Tool call:

```json
{
  "name": "yahoo_us_web_search",
  "arguments": { "query": "apple pie recipe", "limit": 3 }
}
```

Response (excerpt):

```json
{
  "provider": "yahoo_us",
  "vertical": "web",
  "ok": true,
  "query": "apple pie recipe",
  "page": 1,
  "count": 3,
  "next_page": 2,
  "items": [
    {
      "rank": 1,
      "title": "Apple Pie Recipe",
      "url": "https://example.com/apple-pie",
      "description": "A classic double-crust apple pie…",
      "site": "example.com"
    }
  ]
}
```

Web result URLs are unwrapped from Yahoo's redirect URLs
(r.search.yahoo.com) into real destination URLs. This SERP variant shows no
total hit count, so web search does not return `total_hits`.

## Design

```
crates/
├── providers/yahoo_us/       # search logic
│   ├── common.rs             #   HTML cleanup, URL codecs, redirect unwrapping
│   ├── web.rs video.rs news.rs suggest.rs urlbuild.rs
│   └── lib.rs                #   HTTP client and shared response types
├── server/                   # MCP server (stdio JSON-RPC 2.0)
│   ├── main.rs               #   protocol handling
│   └── tools.rs              #   tool definitions and dispatch
└── launcher/                 # registration GUI (egui)
```

Each vertical reads the same public search pages the official Yahoo app
(Yahoo Scout) opens in its WebView. The US SERP ships no JSON payload, so
results are extracted from the server-rendered HTML DOM. The approach was
pinned down by static analysis of the official Android app
(`com.yahoo.mobile.client.android.search` 10.6.0) plus live verification.

### Paging

| Vertical | Form |
|---|---|
| Web | `b=1, 8, 15, …` with `pz=7` (7 per page) |
| Video | `b=1, 61, 121, …` (60 per page; totals from `resultsMeta`) |
| News | `b=1, 11, 21, …` (10 per page) |

### About suggestions

The app's built-in SearchAssist suggestions require a crumb (a CSRF-style
token) that is only issued to logged-in sessions — anonymously the endpoint
returns `{"crumb":null}`. This server uses the SERP's classic gossip
suggest API (`search.yahoo.com/sugg/gossip/gossip-us-ura/`) instead, which
answers JSON anonymously (verified live).

### Why image search is missing

`images.search.yahoo.com` kept returning HTTP 500 with an empty body —
with desktop and mobile user agents, and with Yahoo-issued cookies. That is
not a block response (no 403/429/CAPTCHA), so an egress-region cause cannot
be ruled out; but since retrieval over plain HTTP could not be verified,
it is not implemented.

## Development

```bash
cargo test --workspace                  # all tests, including live e2e against Yahoo
cargo test --workspace -- --skip e2e_   # offline tests only
cargo clippy --workspace --all-targets -- -D warnings
cargo fmt --all
```

The `e2e_*` tests connect to the real Yahoo US and verify each vertical
still parses. The offline parser tests run against real SERP fixtures
recorded during the analysis (`crates/providers/yahoo_us/tests/fixtures/`).

Manual stdio check:

```bash
printf '%s\n' \
  '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"x","version":"1"}}}' \
  '{"jsonrpc":"2.0","id":2,"method":"tools/list"}' \
  '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"yahoo_us_web_search","arguments":{"query":"apple","limit":2}}}' \
  | ./target/debug/yahoo-us-mcp
```

## Notes

- This reads publicly available search result pages. Use it at a reasonable rate.
- If Yahoo changes its page structure, the affected vertical may fail temporarily; the error message says so.
- This project is not affiliated with Yahoo Inc., and is a separate thing from Yahoo! JAPAN (Yahoo Japan Corporation).

## License

MIT — see [LICENSE](LICENSE).
