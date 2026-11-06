# Yahoo US Search MCP

米Yahoo（Yahoo US）の検索をAIエージェントから使えるようにするMCPサーバー。**APIキー不要・アカウント不要・設定不要**。

[English](README.en.md)

[![CI](https://github.com/mouseos/Yahoo_US_MCP/actions/workflows/ci.yml/badge.svg)](https://github.com/mouseos/Yahoo_US_MCP/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

> これは米国の Yahoo Search（search.yahoo.com）用です。Yahoo! JAPAN（日本版）とは別サービスで、日本版は [Yahoo-Japan-Search-MCP](https://github.com/mouseos/Yahoo-Japan-Search-MCP) が担当します。

---

## 対応する検索

| ツール | 検索 | 1ページあたり |
|---|---|---|
| `yahoo_us_web_search` | ウェブ検索 | 7 |
| `yahoo_us_video_search` | 動画検索 | 60 |
| `yahoo_us_news_search` | ニュース検索 | 10 |
| `yahoo_us_suggest` | サジェスト（キーワード補完） | 最大20 |
| `yahoo_us_build_search_url` | 検索URLの生成（通信なし） | — |
| `yahoo_us_list_verticals` | 上記の一覧を返す | — |

画像検索は未対応です。Yahoo の画像検索ページ（images.search.yahoo.com）は、UAやCookieを変えても通常のHTTPでは HTTP 500 しか返らず、取得方法を確認できなかったため入れていません（検証の詳細は後述）。

## インストール

### 1. リリースから

1. [Releases](https://github.com/mouseos/Yahoo_US_MCP/releases) から自分のOS向けZIPをダウンロードして展開
2. `yahoo-us-mcp-launcher` を起動し、使いたいエージェントの **インストール** をクリック
3. エージェントを再起動

ランチャーは Claude Desktop / Claude Code / Google Antigravity / Codex CLI を自動検出して登録します。

### 2. コマンドラインから

```bash
# 設定用のJSONとコマンドを表示
yahoo-us-mcp setup

# Claude Code
claude mcp add Yahoo_US_MCP -- /path/to/yahoo-us-mcp
```

### 3. 手動設定

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

`env` は空のままで構いません。設定する項目はありません。

### 4. ソースからビルド

```bash
git clone https://github.com/mouseos/Yahoo_US_MCP.git
cd Yahoo_US_MCP
cargo build --release
# → target/release/yahoo-us-mcp
```

Rust 1.80以上が必要です（`std::sync::LazyLock` のため）。Linuxではランチャーのビルドに
`libfontconfig1-dev libxkbcommon-dev libgtk-3-dev` が追加で必要です。サーバー本体に
システム依存はありません。

## Rustライブラリとして使う

検索ロジックは provider crate（package名 `yahoo-us-mcp-provider`、import名は
lib名の `yahoo_us_search`）にあり、MCPサーバーを介さず自分のRustプログラムから
そのまま使えます。APIキー・ログイン・設定は不要で、MCP版と同じく匿名で動きます。

```toml
[dependencies]
# GitHubから
yahoo_us_search = { git = "https://github.com/mouseos/Yahoo_US_MCP", package = "yahoo-us-mcp-provider" }
# ローカルチェックアウトから
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

公開API（すべて crate ルートから import できます）:

| 種類 | 名前 |
|---|---|
| クライアント | `YahooUsSearchClient`（`web_search` / `video_search` / `news_search` / `suggest`、戻り値は `anyhow::Result`） |
| 共通の結果型 | `SearchResult<T>`、`PROVIDER`、`verticals()` |
| 各検索の結果型 | `WebItem`、`VideoItem`、`NewsItem`、`SuggestResult` / `Suggestion` |
| URL生成 | `build_search_url`（通信なし） |
| オフライン用パーサー | `parse_web_results` / `web_has_next_page`、`parse_video_results` / `parse_video_total` / `parse_video_next_offset`、`parse_news_results` / `news_has_next_page`、`parse_suggest_response`（保存したページHTML・JSONから抽出）、`clean_text`、`decode_entities`、`decode_redirect_url`、`urlencode` / `urldecode` |

結果型は serde の `Serialize` / `Deserialize` に対応し、JSONへの保存と読み戻しが
できます。ネットワーク不要の利用例が
`crates/providers/yahoo_us/examples/`（`offline_basics` と `parse_saved_pages`）に、
外部利用の形そのままのテストが `crates/providers/yahoo_us/tests/library_api.rs` にあります。

なお、MCPサーバー経由で検索が失敗した場合は、ツール結果として
`{provider, ok:false, error:{kind, message}}` の形で返ります（Baidu_MCP / Yahoo-Japan-Search-MCP と共通の形式）。
引数の誤り（クエリ未指定など）は従来どおり MCP の `isError` で返ります。

## 使用例

エージェントへの指示例:

- 「米国のリンゴ農園の収穫量を調べて」 → `yahoo_us_web_search`
- 「Apple のイベント動画を探して」 → `yahoo_us_video_search`
- 「米国の経済ニュースを5件」 → `yahoo_us_news_search`
- 「"app" で始まる検索候補を出して」 → `yahoo_us_suggest`

ツール呼び出し:

```json
{
  "name": "yahoo_us_web_search",
  "arguments": { "query": "apple pie recipe", "limit": 3 }
}
```

レスポンス（抜粋）:

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

ウェブ検索の結果URLは、Yahoo のリダイレクトURL（r.search.yahoo.com）ではなく
実際の遷移先URLに変換して返します。総件数はこのSERPの版には表示自体がないため、
ウェブ検索では `total_hits` を返しません。

## 設計

```
crates/
├── providers/yahoo_us/       # 検索ロジック
│   ├── common.rs             #   HTML整形・URLデコード・リダイレクト解決
│   ├── web.rs video.rs news.rs suggest.rs urlbuild.rs
│   └── lib.rs                #   HTTPクライアントと共通レスポンス型
├── server/                   # MCPサーバー本体（stdio JSON-RPC 2.0）
│   ├── main.rs               #   プロトコル処理
│   └── tools.rs              #   ツール定義とディスパッチ
└── launcher/                 # 登録用GUI（egui）
```

各検索は、Yahoo 公式アプリ（Yahoo Scout）が WebView で開く公開検索ページをそのまま
読み取ります。米Yahoo の SERP は JSON ペイロードを持たないため、サーバーレンダリング済み
HTML の DOM から抽出します。方式の確定は、公式Androidアプリ
（`com.yahoo.mobile.client.android.search` 10.6.0）の静的解析と実通信検証に基づきます。

### ページング

| 検索 | 形式 |
|---|---|
| ウェブ | `b=1, 8, 15, …` ＋ `pz=7`（7件/ページ） |
| 動画 | `b=1, 61, 121, …`（60件/ページ、`resultsMeta` で総件数も取得） |
| ニュース | `b=1, 11, 21, …`（10件/ページ） |

### サジェストについて

アプリ内蔵の SearchAssist サジェストは crumb（CSRFトークン相当）を必要とし、
匿名では `{"crumb":null}` しか返らないため使えません。代わりに SERP 系の古典的
サジェストAPI（`search.yahoo.com/sugg/gossip/gossip-us-ura/`）を使います。
こちらは匿名で JSON が返ることを実測しています。

### 画像検索が未対応の理由

`images.search.yahoo.com` は、デスクトップUA・モバイルUA・Cookie付きの
いずれでも HTTP 500（本文なし）を返し続けました。403/429/CAPTCHA のような
ブロック応答ではないため、この環境の出口に固有の可能性は残りますが、
通常のHTTPで取得できることを確認できなかった以上、実装していません。

## 開発

```bash
cargo test --workspace                  # 全テスト（実際にYahooへ通信するE2E含む）
cargo test --workspace -- --skip e2e_   # ネットワーク不要のテストのみ
cargo clippy --workspace --all-targets -- -D warnings
cargo fmt --all
```

`e2e_*` テストは実際の米Yahoo に接続し、各検索が正しく解析できているかを検証します。
パーサーのオフラインテストには、解析時に記録した実SERPのフィクスチャ
（`crates/providers/yahoo_us/tests/fixtures/`）を使います。

stdio越しの手動確認:

```bash
printf '%s\n' \
  '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"x","version":"1"}}}' \
  '{"jsonrpc":"2.0","id":2,"method":"tools/list"}' \
  '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"yahoo_us_web_search","arguments":{"query":"apple","limit":2}}}' \
  | ./target/debug/yahoo-us-mcp
```

## 注意

- 公開されている検索結果ページを読み取ります。常識的な頻度で利用してください。
- Yahoo がページ構造を変更した場合、該当する検索が一時的に失敗することがあります。エラーメッセージにその旨が示されます。
- 本プロジェクトは Yahoo Inc. とは無関係です。Yahoo! JAPAN（Yahoo Japan Corporation）とも別物です。

## ライセンス

MIT — [LICENSE](LICENSE) を参照。
