# yorishiro-proxy-docs

yorishiro-proxy の公式ドキュメントサイト。MkDocs Material + GitHub Pages で構築。

**ソースリポジトリ**: `src-ref/` (git submodule) — ドキュメントの正確性検証に使用。
`git submodule update --init` で初期化すること。

## サイト構成

```
docs/
  index.md                    # ホームページ
  getting-started/             # 初回セットアップガイド (5p)
  concepts/                    # 設計思想・アーキテクチャ (4p)
  tools/                       # MCP ツールリファレンス (12p)
  features/                    # 機能詳細ガイド (11p)
  protocols/                   # プロトコル解説 (7p)
  webui/                       # WebUI ガイド (9p)
  plugins/                     # プラグイン開発ガイド (6p)
  configuration/               # 設定リファレンス (5p)
  guides/                      # 実践チュートリアル (5p)
mkdocs.yml                     # サイト設定
```

## ビルド・プレビュー

```bash
pip install mkdocs-material    # 初回のみ
mkdocs build                   # サイトをビルド (site/ に出力)
mkdocs serve                   # ローカルプレビュー (http://127.0.0.1:8000)
```

## 執筆規約

### スタイル

- **言語**: 英語
- **人称**: 二人称 ("you") を使用
- **トーン**: 技術的だが親しみやすい。簡潔で直接的
- **コードブロック**: 言語指定必須 (```bash, ```json, ```python 等)
- **MCP ツール呼び出し例**: JSON 形式、`// tool_name` コメント付き
- **見出し**: Title Case は使わない、Sentence case を使う

### MCP ツール呼び出し例のフォーマット

```markdown
```json
// proxy_start
{
  "listen_addr": "127.0.0.1:8080"
}
```
```

### 参照ソース

ドキュメントの内容はソースリポジトリから正確に取得する:

| 情報 | ソース |
|------|--------|
| MCP ツールパラメータ | `src-ref/internal/mcp/tool_*.go` の ToolDefinition |
| CLI フラグ | `src-ref/cmd/yorishiro-proxy/main.go` の flag 定義 |
| config 構造 | `src-ref/internal/config/config.go` の Config struct |
| プロトコル動作 | `src-ref/internal/protocol/*/` の実装 |
| プラグイン API | `src-ref/internal/plugin/` + `src-ref/docs/plugins.md` |
| 既存ドキュメント | `src-ref/docs/getting-started.md`, `src-ref/docs/plugins.md` |
| README | `src-ref/README.md`, `src-ref/README-ja.md` |
| WebUI ページ | `src-ref/web/src/pages/` の React コンポーネント |
| MCP ヘルプリソース | `src-ref/internal/mcp/resources/` の埋め込みテキスト |

### ページ構成パターン

各ページは以下の構成を基本とする:

```markdown
# ページタイトル

概要 (1-2 文)

## セクション 1

本文...

### サブセクション

コード例...

## 関連ページ

- [リンク](../path/to/page.md) — 説明
```

### 注意事項

- ソースリポジトリは `src-ref/` (git submodule) を参照
- 画像は `docs/images/` に配置
- 内部リンクは相対パスを使用
- `mkdocs build` でエラーが出ないことを確認してからコミット
- ナビゲーション構造は `mkdocs.yml` の `nav` セクションで管理

## ブランチ戦略

- `main` — 常にビルド可能な状態を維持
- 機能ブランチ: `feat/<issue-id>-<short-desc>`
- PR はすべてビルド通過後にマージ

## コミット規約

Conventional Commits 形式:

```
docs(<scope>): <description>

[optional body]

Refs: <Issue ID>
```

scope: セクション名 (例: `getting-started`, `tools`, `features`, `plugins`)

## Linear

- チーム: Usk6666
- プロジェクト: yorishiro-proxy
- マイルストーン: M25: Documentation Site
