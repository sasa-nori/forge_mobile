# add-setup-command 提案書

## 意図（Intent）

Forge を技術スタック非依存のワークフローシステムとして汎用化するにあたり、プロジェクトごとのドメインスキル設定を対話的に支援する `/setup` コマンドが必要。現状ではスキルの発見・インストール・設定をすべて手動で行う必要があり、Forge を新規プロジェクトに導入する際のハードルが高い。`/brainstorm` → `/spec` → `/implement` の前段として `/setup` を位置づけ、Forge のワークフロー思想（対話的に段階を踏む）をプロジェクト初期設定にも適用する。

**背景**: PR #12（project-specific-config）で議論された結果、`~/.claude/projects/<hash>/` を使った3層アーキテクチャではなく、Claude Code の標準機能のみで実現する方針に合意。`/setup` コマンドはその合意に基づく実装。

## スコープ（Scope）

### ユーザーストーリー

- 開発者として、新規プロジェクトで `/setup` を実行して技術スタックに合ったスキルを自動検索・インストールしたい。なぜなら、手動でスキルを探してインストールする手間を省きたいから。
- 開発者として、インストール済みスキルがある状態で `/setup` を再実行しても安全に差分だけ追加提案してほしい。なぜなら、プロジェクトの技術スタック追加時にも同じフローで対応したいから。
- 開発者として、検出した技術スタックに対してスキル検索でヒットしない、またはstar/install数が低くマイナーな場合に、skill-creator を使ったスキル生成を対話的にガイドしてほしい。なぜなら、カバーされない領域を自作スキルで補完したいから。
- 開発者として、スキルのインストール先をプロジェクトローカルかグローバルか選びたい。なぜなら、プロジェクト固有のスキルと全プロジェクト共通のスキルを使い分けたいから。

### 対象領域

- `/setup` コマンド定義（`commands/setup.md`）
- スキルカタログデータ（技術スタック → 検索キーワードのマッピング）
- `<project>/.claude/setup.md` 生成ロジック
- `<project>/CLAUDE.md` テンプレート生成
- skill-creator のグローバルインストール（Forge の install.sh に追加）

### `/setup` コマンドのフロー

1. **技術スタック自動検出**: `package.json`, `prisma/schema.prisma`, `*.tf`, `go.mod`, `requirements.txt` 等のファイルパターンから技術スタックを推論
2. **既存スキルスキャン**: `<project>/.claude/skills/` と `~/.claude/skills/` をスキャンし、インストール済みスキルを把握
3. **スキル検索**: 検出した技術スタックをキーワードに、以下のソースから検索
   - skills.sh API（`/api/search?q={query}`）-- install数でランキング
   - GitHub API（awesome-claude-skills, everything-claude-code 等の有名リポジトリ）
   - GitHub トピック検索（`claude-code-skill` 等）
4. **対話的選択**: 検索結果をランキング付きで提示し、ユーザーがインストールするスキルを選択。各スキルについてインストール先（プロジェクト/グローバル）を確認（デフォルト: プロジェクト）
5. **インストール**: `npx skills add` または git clone + コピーでスキルをインストール
6. **追加検索**: 「他に探したいスキルはありますか？」でキーワード追加検索
7. **スキル作成提案**: カバーされない技術スタック（検索ヒットなし or star/install数が低くマイナー）がある場合、skill-creator を使ったスキル生成を対話的にガイド
8. **設定ファイル生成**:
   - `<project>/.claude/setup.md`: インストールしたスキル/エージェントのガイダンステーブル
   - `<project>/CLAUDE.md`: 存在しない場合はテンプレートを生成（setup.md への参照を含む）。既に存在する場合は setup.md への参照のみ追加

### 冪等性

- 再実行時は既存スキルをスキップし、新しく検出された技術スタックに対してのみ追加提案
- `setup.md` は再生成（マネージドファイル）
- `CLAUDE.md` への参照追加は重複チェック

### 検索ソース

| ソース | 検索方法 | ランキング情報 | インストール方法 |
|---|---|---|---|
| skills.sh | JSON API `/api/search?q={query}` | 週間install数 | `npx skills add` |
| awesome-claude-skills | GitHub API `git/trees` + README パース | リポジトリstar数 (39K) | git clone + コピー |
| everything-claude-code | GitHub API `contents/skills` | リポジトリstar数 (56K) | git clone + コピー |
| GitHub トピック検索 | `claude-code-skill` 等のトピックで検索 | リポジトリ個別star数 | リポジトリごと |

### skill-creator 連携

- skill-creator（Anthropic公式: `anthropics/skills/skill-creator`）を Forge の install.sh でグローバルにインストール
- `/setup` 内でカバーされない領域を検出した場合、skill-creator に渡すプロンプトのおすすめを対話的に提示
- ユーザーが承認すれば skill-creator を呼び出してスキルを生成

## スコープ外（Out of Scope）

- Forge リポジトリからのドメインスキル除去: 手動で別途実施 -- 前提条件ではあるが `/setup` のスコープ外
- グローバル CLAUDE.md の整理: 手動で別途実施
- `permissions.deny` の自動設定: ドメインスキルが Forge から除去されるため不要
- スキルの自動更新機能: 初回セットアップと追加に集中 -- YAGNI
- MCP サーバーの設定: スキルのみに集中 -- YAGNI
- エージェントのインストール: エージェントはワークフロー汎用のため Forge に残る -- スコープ外

## 未解決の疑問点（Open Questions）

- skills.sh API のレート制限やサービス安定性（公開APIだが SLA 不明）
- awesome-claude-skills / everything-claude-code のスキルインストール方式の統一（リポジトリごとにディレクトリ構造が異なる可能性）
- skill-creator のスキル生成品質をどこまで `/setup` 内で検証するか（eval まで回すか、SKILL.md 生成のみか）
