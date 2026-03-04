# add-setup-command タスクリスト

## テスト戦略

- ユニットテスト: 対象外（コマンド定義は Markdown ファイルであり、プログラムコードではない）
- 統合テスト: 対象外（同上）
- E2Eテスト: 手動シナリオベース検証。各タスク完了後に検証チェックリストに基づいて確認する
- 構造検証: ファイルの存在確認、frontmatter の形式確認、既存パターンとの整合性確認を `grep`/`cat` コマンドで実施

## タスク

### Task 1: skill-creator の SKILL.md を Forge リポに配置（推定: 3分）

- **対象ファイル**: `skills/skill-creator/SKILL.md`（新規）
- **やること**: `anthropics/skills` リポジトリから skill-creator の SKILL.md を取得し、`skills/skill-creator/SKILL.md` に配置する。`gh api` を使用してリポジトリ内容を取得する
- **検証方法**: `cat skills/skill-creator/SKILL.md` で SKILL.md が存在し、frontmatter に `name: skill-creator` と `description` が含まれていることを確認する
- **関連要件**: REQ-009
- **関連スペック**: `specs/setup-command/delta-spec.md#skill-creator の Forge リポへの同梱`
- **依存**: なし

### Task 2: skills-lock.json に skill-creator エントリを追加（推定: 2分）

- **対象ファイル**: `skills-lock.json`（既存）
- **やること**: skills-lock.json に `"skill-creator": { "source": "anthropics/skills", "sourceType": "github", "computedHash": "..." }` エントリを追加する。computedHash は SKILL.md の SHA-256 ハッシュから算出する
- **検証方法**: `cat skills-lock.json | grep skill-creator` で エントリが存在することを確認する。`python3 -c "import json; d=json.load(open('skills-lock.json')); print(d['skills']['skill-creator'])"` で正しい形式を確認する
- **関連要件**: REQ-009
- **関連スペック**: `specs/setup-command/delta-spec.md#skill-creator の Forge リポへの同梱`
- **依存**: Task 1

### Task 3: commands/setup.md の frontmatter とヘッダーを作成（推定: 2分）

- **対象ファイル**: `commands/setup.md`（新規）
- **やること**: 既存コマンド（brainstorm.md）のパターンに従い、frontmatter（`description`, `disable-model-invocation: true`, `argument-hint: "[keyword]"`）と `# /setup コマンド` ヘッダー、`REQUIRED SKILLS:` セクション、`## 目的` セクションを作成する
- **検証方法**: `head -20 commands/setup.md` で frontmatter の形式を確認する。brainstorm.md と同じパターン（`description`, `disable-model-invocation`, `argument-hint`）であることを目視確認する
- **関連要件**: REQ-001, REQ-002, REQ-003, REQ-004, REQ-005, REQ-006, REQ-007, REQ-008
- **関連スペック**: `specs/setup-command/delta-spec.md`（全要件の基盤）
- **依存**: なし

### Task 4: commands/setup.md にステップ1（技術スタック自動検出）を記述（推定: 5分）

- **対象ファイル**: `commands/setup.md`（既存 - Task 3で作成）
- **やること**: ワークフローのステップ1として技術スタック自動検出ロジックを記述する。ファイルパターン → 技術スタックのマッピングテーブル、バージョン検出方法（package.json の dependencies からメジャーバージョン取得）、検出結果のユーザー確認フローを含む。`$ARGUMENTS` が指定されている場合は自動検出をスキップしてキーワードとして使用する旨を記述する。モノレポ対応としてルート直下 + 1階層下（`*/` 配下の全サブディレクトリ。例: `packages/*/`, `apps/*/`, `services/*/`）の検出範囲を明記する。package.json パースエラー時のフォールバック（ファイル存在のみで「Node.js プロジェクト」として検出）、バージョン検出失敗時のフォールバック（「バージョン不明」として検出）を含む
- **検証方法**: `grep -c "package.json\|prisma\|tsconfig\|\.tf\|go\.mod\|requirements\.txt\|Cargo\.toml\|pom\.xml" commands/setup.md` でファイルパターンが網羅されていることを確認する。`grep -c "ARGUMENTS\|モノレポ\|1階層\|パースエラー\|バージョン不明" commands/setup.md` で追加要件の記述を確認する
- **関連要件**: REQ-001（Happy Path 全件、Error Scenarios 全件、Boundary Scenario、$ARGUMENTS シナリオ）
- **関連スペック**: `specs/setup-command/delta-spec.md#技術スタック自動検出`
- **依存**: Task 3

### Task 5: commands/setup.md にステップ2（既存スキルスキャン）を記述（推定: 2分）

- **対象ファイル**: `commands/setup.md`（既存 - Task 3で作成）
- **やること**: ワークフローのステップ2として、`<project>/.claude/skills/` と `~/.claude/skills/` の両方をスキャンし、インストール済みスキルのリストを構築するロジックを記述する。冪等性の基盤となるステップ
- **検証方法**: `grep -c "\.claude/skills" commands/setup.md` でプロジェクトローカルとグローバル両方の参照があることを確認する
- **関連要件**: REQ-008
- **関連スペック**: `specs/setup-command/delta-spec.md#冪等性`
- **依存**: Task 4

### Task 6: commands/setup.md にステップ3（スキル検索）を記述（推定: 5分）

- **対象ファイル**: `commands/setup.md`（既存 - Task 3で作成）
- **やること**: ワークフローのステップ3としてスキル検索ロジックを記述する。skills.sh（`npx skills find`）と GitHub API（`gh api`）の並行検索、結果のマージ（同名スキルは重複排除せず両方表示、各スキルにソース明記）、ソース別グループ表示（skills.sh グループを上位に install 数降順、GitHub グループを下位に star 数降順）を含む。各検索ソース（skills.sh, awesome-claude-skills, everything-claude-code, GitHub トピック検索）のパース方法を個別に記述する。フォールバック戦略（片方障害時の graceful degradation、npx 未インストール時の warn + GitHub のみ続行、gh 未認証時の未認証レート制限動作 + 案内表示）を含む。検索結果0件時の REQ-005 遷移、50件超時の上位10件表示を含む
- **検証方法**: `grep -c "skills find\|gh api\|awesome-claude-skills\|everything-claude-code\|install 数\|star 数\|グループ\|npx.*利用できません\|上位10件" commands/setup.md` で検索ソース、ソート基準、グループ表示、フォールバック、件数制限の記述を確認する
- **関連要件**: REQ-002（Happy Path 全件、Error Scenarios 全件、Boundary Scenarios 全件）
- **関連スペック**: `specs/setup-command/delta-spec.md#スキル検索`
- **依存**: Task 5

### Task 7: commands/setup.md にステップ4（対話的選択・インストール）を記述（推定: 5分）

- **対象ファイル**: `commands/setup.md`（既存 - Task 3で作成）
- **やること**: ワークフローのステップ4として対話的スキル選択・インストールフローを記述する。ランキング表示形式、インストール先選択（プロジェクト/グローバル、デフォルト: プロジェクト）、インストール方法（npx skills add / gh api + コピー）の分岐を含む。検索ソースごとの SKILL.md 取得パス（awesome-claude-skills: `{skill-name}/SKILL.md`、everything-claude-code: `skills/{skill-name}/SKILL.md`、個別リポジトリ: ルート直下 `SKILL.md` or `.claude/skills/{name}/SKILL.md`）を明記する。skills-lock.json へのエントリ追記を含む。不正な選択番号入力時の再入力促し、ネットワーク接続なし時の案内、同名スキル上書き確認、skills-lock.json 書き込み失敗時の警告（インストールはロールバックしない）を含む
- **検証方法**: `grep -c "プロジェクト\|グローバル\|npx skills add\|gh api\|skills-lock\|取得パス\|無効な番号\|上書き" commands/setup.md` でインストール先、インストール方法、lock 記録、取得パス、エラーハンドリングの記述を確認する
- **関連要件**: REQ-003（Happy Path 全件、Error Scenarios 全件、Boundary Scenarios 全件）
- **関連スペック**: `specs/setup-command/delta-spec.md#対話的スキル選択・インストール`
- **依存**: Task 6

### Task 8: commands/setup.md にステップ4.5（セキュリティ検証）を記述（推定: 3分）

- **対象ファイル**: `commands/setup.md`（既存 - Task 3で作成）
- **やること**: ステップ4のインストール実行前に挟むセキュリティ検証フローを記述する。SKILL.md の内容要約表示（description + 主要セクション見出し）とソース URL、star 数（または install 数）、最終更新日の明示表示、ユーザー確認（y/N）、SKILL.md 取得不可時のリスク明示フローを含む
- **検証方法**: `grep -c "SKILL.md.*要約\|確認.*y/N\|セキュリティ\|ソース URL\|最終更新日" commands/setup.md` で確認フローの記述を確認する
- **関連要件**: REQ-004（Happy Path 全件、Error Scenarios 全件）
- **関連スペック**: `specs/setup-command/delta-spec.md#セキュリティ検証`
- **依存**: Task 7

### Task 9a: commands/setup.md にステップ5（追加キーワード検索ループ）を記述（推定: 2分）

- **対象ファイル**: `commands/setup.md`（既存 - Task 3で作成）
- **やること**: ステップ5として追加キーワード検索ループ（「他に探したいスキルはありますか？」→ キーワード入力 → 検索 → 選択・インストール → ループ or 終了）を記述する。検索結果0件時のメッセージ表示を含む
- **検証方法**: `grep -c "他に探したい\|追加.*検索\|ループ" commands/setup.md` でステップ5の主要要素が記述されていることを確認する
- **関連要件**: REQ-005（Happy Path 全件、Error Scenarios 全件）
- **関連スペック**: `specs/setup-command/delta-spec.md#追加キーワード検索`
- **依存**: Task 8

### Task 9b: commands/setup.md にステップ6（スキル作成提案）を記述（推定: 3分）

- **対象ファイル**: `commands/setup.md`（既存 - Task 3で作成）
- **やること**: ステップ6として、カバーされない技術スタックの検出と skill-creator 連携を記述する。スキル作成提案の閾値（検索結果0件、または全結果が以下のいずれかに該当: (a) skills.sh ソースで install 数 1,000 未満、(b) GitHub ソースで star 数 100 未満）、おすすめプロンプトの提示、description 3部構成テンプレートの提供、ユーザーカスタマイズ後の skill-creator 呼び出しを含む。skill-creator 未インストール時の案内を含む
- **検証方法**: `grep -c "skill-creator\|3部構成\|1,000\|100" commands/setup.md` でステップ6の主要要素と閾値が記述されていることを確認する
- **関連要件**: REQ-006（Happy Path 全件、Error Scenarios 全件）
- **関連スペック**: `specs/setup-command/delta-spec.md#スキル作成提案`
- **依存**: Task 9a

### Task 10: commands/setup.md にステップ7（設定ファイル生成）を記述（推定: 4分）

- **対象ファイル**: `commands/setup.md`（既存 - Task 3で作成）
- **やること**: ステップ7として設定ファイル生成ロジックを記述する。setup.md のガイダンステーブル形式（スキル名・インストール先・description 1行要約）、CLAUDE.md の新規作成時の構造化テンプレート（プロジェクト概要・技術スタック・Available Skills・Available Agents の4セクション）と既存ファイルへの参照追記ロジック（重複チェック付き）を含む。setup.md には SKILL.md のインライン展開を禁止する旨を明記する
- **検証方法**: `grep -c "setup\.md\|CLAUDE\.md\|ガイダンステーブル\|インライン展開禁止\|プロジェクト概要\|技術スタック\|Available Skills\|Available Agents" commands/setup.md` で設定ファイル生成の主要要素と構造化テンプレートの記述を確認する
- **関連要件**: REQ-007（Happy Path 全件、Error Scenarios 全件）
- **関連スペック**: `specs/setup-command/delta-spec.md#設定ファイル生成`
- **依存**: Task 9b

### Task 11: commands/setup.md に冪等性ルールを記述（推定: 2分）

- **対象ファイル**: `commands/setup.md`（既存 - Task 3で作成）
- **やること**: コマンド定義の「重要なルール」セクションまたは各ステップ内に冪等性に関するルールを記述する。再実行時の既存スキルスキップ、差分のみ提案、setup.md 再生成、CLAUDE.md 参照の重複チェックを含む
- **検証方法**: `grep -c "冪等\|再実行\|スキップ\|重複" commands/setup.md` で冪等性関連の記述を確認する
- **関連要件**: REQ-008（Happy Path 全件、Error Scenarios 全件）
- **関連スペック**: `specs/setup-command/delta-spec.md#冪等性`
- **依存**: Task 10

### Task 12: CLAUDE.md に /setup コマンドと skill-creator の記載を追加（推定: 3分）

- **対象ファイル**: `CLAUDE.md`（既存）
- **やること**: CLAUDE.md の Forge ワークフローセクションに `/setup` の位置づけを追記する。Available Skills テーブルに `skill-creator` を追加する。ワークフロー図の `/brainstorm` の前に `/setup`（初回のみ）を追記する
- **検証方法**: `grep -c "setup\|skill-creator" CLAUDE.md` で /setup と skill-creator の記載を確認する
- **関連要件**: REQ-007, REQ-009, domain-skills REQ-005（MODIFIED）
- **関連スペック**: `specs/setup-command/delta-spec.md#設定ファイル生成`, `specs/setup-command/delta-spec.md#skill-creator の Forge リポへの同梱`
- **依存**: Task 3

### Task 13: 最終検証（推定: 5分）

- **対象ファイル**: 全対象ファイル
- **やること**: 以下の検証チェックリストを全て実行する:
  1. `commands/setup.md` が存在し、frontmatter に `description`, `disable-model-invocation: true`, `argument-hint` が含まれている
  2. `commands/setup.md` の本文構造が brainstorm.md パターン（`# /setup コマンド` → `REQUIRED SKILLS:` → `## 目的` → `## ワークフロー`）に従っている
  3. `skills/skill-creator/SKILL.md` が存在し、frontmatter に `name` と `description` が含まれている
  4. `skills-lock.json` に `skill-creator` エントリが存在する
  5. `CLAUDE.md` に `/setup` と `skill-creator` の記載がある
  6. `commands/setup.md` にステップ1-7の全ステップが記述されている
  7. セキュリティ検証（SKILL.md 要約表示 + ソース URL + star 数 + 最終更新日 + ユーザー確認）が記述されている
  8. フォールバック戦略（skills.sh/GitHub API 障害時、npx 未インストール時、gh 未認証時）が記述されている
  9. 冪等性ルール（既存スキルスキップ、差分提案）が記述されている
  10. ソース別グループ表示（skills.sh 上位 install 数降順、GitHub 下位 star 数降順）が記述されている
  11. モノレポ対応（ルート + 1階層下）が記述されている
  12. $ARGUMENTS によるキーワード直接指定が記述されている
  13. CLAUDE.md 構造化テンプレートが記述されている
  14. skills-lock.json へのエントリ追記が記述されている
  15. スキル作成提案の閾値（skills.sh: install 数 1,000 未満 / GitHub: star 数 100 未満）が記述されている
  16. REQ-001 の Error Scenarios（パースエラー、バージョン不明）が記述されている
  17. REQ-002 の Boundary Scenarios（npx 未インストール、gh 未認証、0件、50件超）が記述されている
  18. REQ-003 の Error/Boundary Scenarios（不正入力、ネットワーク断、同名スキル上書き、skills-lock.json 失敗）が記述されている
- **検証方法**: 上記チェックリストを `grep` と `cat` コマンドで一つずつ確認する
- **関連要件**: REQ-001 - REQ-009（全要件）+ domain-skills REQ-005（MODIFIED）
- **関連スペック**: `specs/setup-command/delta-spec.md`（全体）
- **依存**: Task 1, Task 2, Task 3, Task 4, Task 5, Task 6, Task 7, Task 8, Task 9a, Task 9b, Task 10, Task 11, Task 12
