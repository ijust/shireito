# Shireito (司令塔)

Claude Code 用の**司令塔パターン**プラグイン。専門化された 5 種類のサブエージェントと、運用ルール、対話的な permission 設定をひとまとめにして配布します。

> English version: [README.md](./README.md)

## 同梱されるもの

- **5 つのサブエージェント** (`agents/`)
  - `explorer` — 高速なコードベース探索（haiku）
  - `code-analyst` — 設計・アーキテクチャの深層分析（sonnet）
  - `code-reviewer` — 品質・セキュリティレビュー、構造的に読み取り専用（sonnet）
  - `implementer` — 実装・修正・リファクタ（sonnet、Edit/Write 権限あり）
  - `debugger` — 障害・テスト失敗の根本原因追究と最小修正（sonnet、Edit 権限あり）
- **`/shireito:orchestrate`** スキル — 司令塔の運用ルール（並列／順次の判断、worktree 隔離、permission の落とし穴）を、複数サブエージェントを使う作業を計画するときに司令塔のコンテキストに読み込ませる
- **`/shireito:setup`** スキル — 現在のプロジェクトの `.claude/settings.json` に必要な permission と worktree パスを、対話しながら整える

## インストール

Claude Code 内で：

```
/plugin marketplace add ijust/shireito
/plugin install shireito
```

プロジェクトごとに 1 回：

```
/shireito:setup
```

setup スキルは、プロジェクトのルートを検出し、worktree の置き場所と permission 戦略をユーザーに確認した上で、`.claude/settings.json` にマージで書き込みます。**既存の他の設定は触りません**。

## なぜ permission を直接同梱せず setup スキルにしたか

- サブエージェントは親セッションの `permissions.allow` を**継承しません**
- worktree はリポジトリ外のパスに作られるため、`Edit(/abs/path/to/repo/**)` のような path-scoped ルールでは**カバーされません**
- 結果として、書き込み権限を持つサブエージェント（`implementer` / `debugger`）を並列で動かすと、各々が permission prompt を出して**実質シリアル化**してしまう

正しい設定はプロジェクト絶対パスを含むため、プラグインファイルとして配ることができません。setup スキルがプロジェクトごとに 1 回それを書きます。

## クイックスタート

インストール＋セットアップが済んだら、プロジェクト内で Claude に：

> explorer サブエージェントで認証モジュールの構造を把握して、code-analyst で設計上の問題を分析して。

司令塔（メイン Claude）が両方のサブエージェントを並列発火（独立な読み取りタスク同士なので）、結果を集約して報告します。

並列で実装が必要な作業：

> feature A と feature B を並列で実装して。implementer サブエージェントを別の worktree で動かして。

司令塔が worktree を作成し、1 worktree につき 1 つの `implementer` を割り当て、後で統合します。

どのサブエージェントを呼ぶか迷うときは、`/shireito:orchestrate` で判断ルールをコンテキストに読み込ませてください。

## 司令塔パターンの全体像

```
[メイン Claude Code セッション = 司令塔]
   ├─ Agent("explorer", "...")     ← 並列可（読み取りのみ）
   ├─ Agent("code-analyst", "...") ← 並列可（読み取りのみ）
   └─ Agent("implementer", ".worktrees/feat-a", "...")  ← 順次 or 隔離して並列
       ↓
   司令塔が結果を集約 → 人間にレポート
```

人間が話しかける相手は**司令塔だけ**。何を委譲するか、いつ並列にするか、どう統合するかは司令塔が判断します。

## ファイル構成

```
shireito/
├── .claude-plugin/plugin.json     # プラグイン マニフェスト
├── agents/                        # 5 つのサブエージェント定義
│   ├── explorer.md
│   ├── code-analyst.md
│   ├── code-reviewer.md
│   ├── implementer.md
│   └── debugger.md
├── skills/
│   ├── setup/SKILL.md             # 対話的 permission セットアップ
│   └── orchestrate/SKILL.md       # 運用ルール
└── README.md / README.ja.md
```

## ライセンス

MIT

## 作者

[ijust (辻 良繁 / Yoshishige Tsuji)](https://github.com/ijust)
