---
description: "M25 の Linear Issue を並行執筆するオーケストレーター。依存関係を分析し、サブエージェントにドキュメントセクションの執筆を委任"
user-invokable: true
---

# /orchestrate

M25 (Documentation Site) の Issue を依存関係に基づいて並行でサブエージェントに執筆させるオーケストレーションスキル。

## 引数

- `/orchestrate` — M25 の未完了 Issue を分析し、次に執筆すべきセクションを提案
- `/orchestrate <Issue ID> [Issue ID...]` — 指定 Issue を依存関係を考慮して執筆
- `/orchestrate status` — 進捗状況を表示

---

## 手順

### Phase 0: 現状把握

1. `mcp__linear-server__list_milestones(project=yorishiro-proxy)` で M25 の progress を確認
2. 未完了 Issue を取得:
   - `mcp__linear-server__list_issues(team=Usk6666, project=yorishiro-proxy, state=backlog)`
   - `mcp__linear-server__list_issues(team=Usk6666, project=yorishiro-proxy, state=unstarted)`
   - `mcp__linear-server__list_issues(team=Usk6666, project=yorishiro-proxy, state=started)`
3. M25 の Issue のみをフィルタリング
4. `mkdocs.yml` と `docs/` の現状を確認し、既に執筆済みのセクションを把握

### Phase 1: 依存分析と実行計画

#### 依存関係

```
USK-334 (MkDocs 基盤) ─── 全 Issue の前提
  ├── USK-335 (ホームページ)
  ├── USK-336 (Getting Started)
  ├── USK-337 (MCP ツールリファレンス)
  ├── USK-338 (CLI フラグ・設定リファレンス)
  │     └── USK-341 (Configuration セクション)
  ├── USK-339 (Features)
  ├── USK-340 (Protocols)
  ├── USK-342 (Concepts)
  ├── USK-343 (WebUI)
  ├── USK-344 (Plugins)
  └── USK-345 (Guides)
```

#### 並行実行の判定基準

ドキュメントセクションは異なるディレクトリに書くため、ほぼ全て並行実行可能。
ただし以下の制約がある:

- **USK-334 (基盤)** は最初に完了する必要がある (`mkdocs.yml`, ディレクトリ構造)
- **USK-341** は USK-338 に依存 (CLI リファレンスの上に詳細設定を追加)
- **最大同時実行数: 3** (リソース制約)

#### バッチ構成 (優先度順)

各バッチ内の Issue は `get_issue` で詳細を確認してから、優先度・依存関係で並行グループを構成する。

ユーザーに実行計画を提示し、承認を得てから Phase 2 に進む。

### Phase 2: サブエージェントの起動と管理

#### 2-1. バッチ実行

各 Issue に対して Agent ツールを起動する:

```
Agent(
  description="Write docs USK-XXX",
  subagent_type="general-purpose",
  isolation="worktree",
  prompt=<doc-writer プロンプト>
)
```

**プロンプト構築:**
1. `.claude/agents/doc-writer.md` を Read ツールで読み込む
2. `## プロンプト本文` セクション内のコードブロックを抽出
3. プレースホルダーを実際の値に置換:
   - `{{ISSUE_ID}}` → Issue ID
   - `{{ISSUE_TITLE}}` → Issue タイトル
   - `{{ISSUE_DESCRIPTION}}` → Issue 説明 (Markdown)
   - `{{BRANCH_NAME}}` → ブランチ名
   - `{{EXISTING_NAV}}` → 現在の `mkdocs.yml` の nav セクション

**バッチ内並行起動:**
同一メッセージ内で複数の Agent ツールを呼び出す。

#### 2-2. バッチ完了待機

バッチ内の全サブエージェントが完了したら:

1. 各サブエージェントの成否を確認
2. 成功した PR をユーザーに報告し、マージ判断を仰ぐ
3. マージ後、main を最新化してから次バッチへ
4. 失敗した Issue は原因を報告し、次のアクションを提案

#### 2-3. レビュー

ドキュメントの場合、コード品質や Security Review の必要性は低い。
代わりに、以下を確認する:

- `mkdocs build` がパスすること (サブエージェント内で検証済み)
- PR の差分を確認し、明らかな問題がないかチェック

重大な問題がなければレビューゲートはスキップし、ユーザーにマージ判断を委ねる。

### Phase 3: 結果集約

全バッチ完了後、以下を報告:

```markdown
## 実装結果サマリー

### M25: Documentation Site

| Issue | タイトル | ステータス | PR | ページ数 |
|-------|---------|----------|-----|---------|
| USK-XXX | ... | OK | #N | 5 |
| ...

### 残り Issue
- ...

### 次ステップ
- ...
```

#### Worktree クリーンアップ

全バッチ完了後、起動した全サブエージェントの worktree を削除:

```bash
git worktree remove .claude/worktrees/agent-<agentId> --force 2>/dev/null || true
git worktree prune
```

---

## 注意事項

- ソースリポジトリ `/home/user/repositories/yorishiro-proxy` は参照のみ (変更しない)
- 最大同時実行数: 3
- 各サブエージェントは独立して動作し、相互参照しない
- `mkdocs.yml` の nav セクションは複数エージェントが同時に変更するとコンフリクトしやすいため、各エージェントは自分のセクションのみ追加する。マージ時にオーケストレーターが調整する
- Linear Issue のステータス更新はオーケストレーター側の責務
