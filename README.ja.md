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

`/plugin install` でスコープを聞かれます。**User** を選ぶと開く全プロジェクトで shireito が使えるようになります（個人開発の推奨）。チームで共有したい（`.claude/` を git で配りたい）場合は **Project** を選択。**Local** はほぼ選ぶ場面なし。

プロジェクトごとに 1 回：

```
/shireito:setup
```

setup スキルは、プロジェクトのルートを検出し、worktree の置き場所と permission 戦略をユーザーに確認した上で、`.claude/settings.json` にマージで書き込みます。**既存の他の設定は触りません**。

## アップデート

```
/plugin marketplace update shireito   # marketplace カタログを再取得
/plugin update shireito               # 取り込んだ最新版に入れ替える
```

両方が必要です。`marketplace update` だけだと index 更新のみで本体は古いまま、`plugin update` だけだと index 側が古くて最新版を認識できません。

このプラグインをローカルで開発している場合（`--plugin-dir` で読み込み中）は `/reload-plugins` を使ってください。

## クイックスタート

インストール＋セットアップが済んだら、プロジェクト内で Claude に：

> explorer サブエージェントで認証モジュールの構造を把握して、code-analyst で設計上の問題を分析して。

司令塔（メイン Claude）が両方のサブエージェントを並列発火（独立な読み取りタスク同士なので）、結果を集約して報告します。

並列で実装が必要な作業：

> feature A と feature B を並列で実装して。implementer サブエージェントを別の worktree で動かして。

司令塔が worktree を作成し、1 worktree につき 1 つの `implementer` を割り当てます。**両方完了後、各 worktree の差分確認と main への merge は人間の手作業として残ります。**

### `/shireito:orchestrate` を自分で invoke するタイミング

`orchestrate` skill は auto-invocable — 司令塔が「multi-subagent の planning」と認識すれば自動で読み込まれることがあります。ただし **auto-invocation は保証されません**。モデルが description マッチで判断するため、見落とすこともあります。以下のケースでは自分で明示的に呼んでください：

- `implementer` サブエージェントを worktree で並列実行する直前
- permission prompt が出て並列 subagent が止まっているとき（permission 継承の罠の可能性）
- 方針を決める前に、判断ルール全体をコンテキストに先入れしておきたいとき
- 司令塔パターンに不慣れで、運用ルールを一通り読みたいとき

```
/shireito:orchestrate
```

明示呼び出しならルールロードは確実。自動発火に頼ると外れることがあります。

## 使い方の例

shireito をインストールしてプロジェクトで `/shireito:setup` を走らせた後、以下のプロンプトで使えます。ファイルパスは各自のコードベースに合わせて調整してください。

### コードベースのマッピング（explorer）

> explorer サブエージェントで、このコードベースを map して。各トップレベルディレクトリを1行で要約して。

haiku で高速、メインコンテキストをファイル内容で汚さずに全体像を掴む。

### 設計レビュー（code-analyst）

> code-analyst で `src/auth.py` の設計を評価して。責務はきれいに分かれているか、変えるべき点はあるか。

sonnet で深い分析、読み取り専用（編集はしない）。

### 隔離 worktree での並列実装（implementer × N）— **真骨頂**

> 2つの feature を `isolation: "worktree"` で並列実装して：
> - Feature A: `cli.py` に `--format=json` を追加
> - Feature B: `cli.py` に `--filter=<glob>` を追加

司令塔が `implementer` サブエージェントを同時に 2 体起動。各々が自動作成された worktree で動くので、同じファイルを同時編集しても衝突しない。両方完了後、worktree diff をレビューして統合。**仕様駆動開発と真の並列性が出会う場所**。

### 変更後のコードレビュー（code-reviewer）

> code-reviewer で直近2 commit の差分をレビューして。セキュリティ、コード品質、エンジニアリング原則の観点で。

設計上 read-only：問題を指摘するだけ、編集はしない。

### 根本原因デバッグ（debugger）

> `tests/test_foo.py::test_edge_case` が失敗している。debugger サブエージェントで根本原因を突き止めて最小修正を当てて。

`debugger` は Edit 権限あり：根本原因、エビデンス、当てた修正、再発防止策の推奨をセットで返す。

## 司令塔パターンの全体像

```
[メイン Claude Code セッション = 司令塔]
   ├─ Agent("explorer", "...")     ← 並列可（読み取りのみ）
   ├─ Agent("code-analyst", "...") ← 並列可（読み取りのみ）
   └─ Agent("implementer", isolation: "worktree", "...")  ← 順次 or 隔離して並列
       ↓
   司令塔が結果を集約 → 人間にレポート
```

司令塔がタスクの分解・並列/順次の判断・subagent からのテキスト結果の集約を担当します。**worktree レベルの変更（並列 implementer など）は、各 worktree を人間が確認・統合します。**

## なぜこの組み合わせが速いのか（仕様駆動開発 × worktree × subagent）

仕様駆動開発（Kiro / cc-sdd、あるいは任意の `.kiro/specs/<feature>/{requirements,design,tasks}.md` フレーバー）は、実装前に**機能を独立した作業単位に分解**します。shireito と git worktree を組み合わせると、司令塔がそれらの作業単位を**複数の `implementer` subagent に並列で振り分け**、それぞれが独立した worktree で同時実装します。

得られるもの：

- **N 個の独立した spec → N 並列 implementer**。スループットは「一人がどれだけ速くタイプできるか」ではなく「実装準備が整った独立 spec が何本あるか」で決まる
- **ファイル衝突によるシリアル化なし**。別 worktree の subagent は同じファイルを同時編集しても衝突しない。衝突は実装中ではなく統合時に解消する
- **レビューもデバッグも並列化**。worktree ごとに `code-reviewer` を 1 体、あるいは特定の worktree で失敗したケースに `debugger` 1 体、と並列で回せる
- **spec はセッションを超えて生き残る**。数ヶ月後に新しい subagent が同じ spec を読めば、実装文脈が即座に復元される。要件の再説明不要

この組み合わせにより、仕様駆動開発は「spec を書くのは overhead」から「**spec を書くと並列化が解放される**」に変わります。司令塔は、N 個の implementer worktree を稼働させ続けるのに十分な独立 spec が用意されていれば、それを次々と捌いていけます。

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

## なぜ permission を直接同梱せず setup スキルにしたか

- サブエージェントは親セッションの `permissions.allow` を**継承しません**
- worktree はリポジトリ外のパスに作られるため、`Edit(/abs/path/to/repo/**)` のような path-scoped ルールでは**カバーされません**
- 結果として、書き込み権限を持つサブエージェント（`implementer` / `debugger`）を並列で動かすと、各々が permission prompt を出して**実質シリアル化**してしまう

正しい設定はプロジェクト絶対パスを含むため、プラグインファイルとして配ることができません。setup スキルがプロジェクトごとに 1 回それを書きます。

## リリース手順

install ユーザーに届けるべき変更 — skill の追加・修正、agent の追加・修正、orchestration ルールの挙動を変える編集 — を加えるときは、**同じ commit（または push 前）で `.claude-plugin/plugin.json` の `version` を bump** してください。bump し忘れると、`/plugin marketplace update` でカタログが最新になっても `/plugin update shireito` が「already at the latest version」で短絡してしまい、install 済みクライアントは古い版を黙って使い続けます。

README のみ・cosmetic・typo の変更には bump 不要。

バージョン体系：semver。fix や軽微な追加は patch（`0.1.x`）、新しい skill/agent や大きな挙動変更は minor（`0.x.0`）。

## 出典・参照

`agents/` の subagent セットは、当初 Anthropic 公式の Claude Code ドキュメント（https://docs.claude.com/en/docs/claude-code/sub-agents）の example subagents、特に `code-reviewer` と `debugger` のパターンに着想を得ました。本リポジトリの定義はそこから司令塔パターン配布用に大幅に再構成・拡張したものです。`orchestrate` / `setup` スキル、5 subagent のキュレーション、プラグイン全体構成は本プラグイン側で独自に組み立てています。

## ライセンス

MIT

## 作者

[ijust (辻 良繁 / Yoshishige Tsuji)](https://github.com/ijust)
