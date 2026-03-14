---
description: "Linear Issue に基づいてドキュメントセクションを執筆し、PR を作成する"
user-invokable: true
---

# /implement

Linear Issue に基づいてドキュメントセクションを執筆し、ブランチ作成から PR 作成までを一気通貫で実行する。

## 引数

- `/implement <Issue ID>` — 指定 Issue のドキュメントセクションを執筆する (例: `/implement USK-336`)

## 手順

1. **Issue 読み込み**: `mcp__linear-server__get_issue` で Issue の詳細を取得
2. **Issue ステータス更新**: ステータスを "In Progress" に更新
3. **ブランチ作成**: `feat/<id>-<short-desc>` 形式でブランチを作成
4. **ソース調査**: ソースリポジトリ (`src-ref/`, git submodule) から正確な情報を収集
   - Issue の説明にある「ソース」セクションを参照
   - MCP ツール定義: `src-ref/internal/mcp/tool_*.go`
   - CLI フラグ: `src-ref/cmd/yorishiro-proxy/main.go`
   - config 構造: `src-ref/internal/config/config.go`
   - プロトコル実装: `src-ref/internal/protocol/*/`
   - プラグイン: `src-ref/internal/plugin/` + `src-ref/docs/plugins.md`
   - 既存ドキュメント: `src-ref/docs/getting-started.md`, `src-ref/docs/plugins.md`, `src-ref/README.md`
   - WebUI: `src-ref/web/src/pages/`
   - MCP ヘルプリソース: `src-ref/internal/mcp/resources/`
5. **執筆**: Issue の「ページ構成」に従ってドキュメントを執筆
   - CLAUDE.md の執筆規約に従う
   - コード例は実際のソースコードから正確に記述
   - 内部リンクは相対パスで記述
6. **ナビゲーション更新**: `mkdocs.yml` の `nav` セクションに新ページを追加
7. **検証**: `mkdocs build` でビルドが通ることを確認
8. **コミット**: Conventional Commits 形式でコミット
   - `docs(<scope>): add <section-name> section`
   - フッターに `Refs: <Issue ID>`
9. **プッシュ**: `git push -u origin <branch-name>`
10. **PR 作成**: `gh pr create` で PR を作成
11. **Issue ステータス更新**: ステータスを "In Review" に更新
12. **結果報告**: 作成したページ一覧 + PR URL

## PR 本文テンプレート

```markdown
## Summary
- <追加したページの箇条書き>

## Checklist
- [ ] `mkdocs build` passes
- [ ] All internal links are valid
- [ ] Code examples are accurate (verified against source)
- [ ] Navigation updated in mkdocs.yml

Resolves <Issue ID>
Linear: https://linear.app/usk6666/issue/<Issue ID>
```

## 注意事項

- ソースコードから情報を取得する際は Read ツールで直接ファイルを読む
- 推測でパラメータや動作を記述しない。不明な点はソースコードで確認する
- `mkdocs build` が失敗した場合は修正してから再実行
- 画像やスクリーンショットが必要なページは、プレースホルダー (`<!-- TODO: screenshot -->`) を残す
- 大量のページ (5+) を含むセクションでは、まず index.md を書き、次に個別ページを順に書く
