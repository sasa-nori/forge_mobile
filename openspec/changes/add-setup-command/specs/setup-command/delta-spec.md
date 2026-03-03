# setup-command デルタスペック

## ADDED Requirements

### Requirement: REQ-001 技術スタック自動検出

`/setup` コマンドは、プロジェクトルートのファイルパターンから技術スタックを自動検出しなければならない (SHALL)。検出結果はユーザーに提示し、確認を得てからスキル検索に進む。ただし `$ARGUMENTS` でキーワードが直接指定された場合は自動検出をスキップする。

#### Happy Path Scenarios

- **GIVEN** プロジェクトルートに `package.json` が存在し、dependencies に `next` が含まれる **WHEN** `/setup` を実行する **THEN** 技術スタックとして「Next.js」を検出し、package.json から Next.js のメジャーバージョンも特定して表示する
- **GIVEN** プロジェクトルートに `prisma/schema.prisma` が存在する **WHEN** `/setup` を実行する **THEN** 技術スタックとして「Prisma」を検出する
- **GIVEN** プロジェクトルートに `*.tf` ファイルが存在する **WHEN** `/setup` を実行する **THEN** 技術スタックとして「Terraform」を検出する
- **GIVEN** プロジェクトルートに `tsconfig.json` が存在する **WHEN** `/setup` を実行する **THEN** 技術スタックとして「TypeScript」を検出する
- **GIVEN** プロジェクトルートに `go.mod` が存在する **WHEN** `/setup` を実行する **THEN** 技術スタックとして「Go」を検出する
- **GIVEN** プロジェクトルートに `requirements.txt` または `pyproject.toml` が存在する **WHEN** `/setup` を実行する **THEN** 技術スタックとして「Python」を検出する
- **GIVEN** プロジェクトルートに `Cargo.toml` が存在する **WHEN** `/setup` を実行する **THEN** 技術スタックとして「Rust」を検出する
- **GIVEN** プロジェクトルートに `pom.xml` または `build.gradle` が存在する **WHEN** `/setup` を実行する **THEN** 技術スタックとして「Java」を検出する
- **GIVEN** 複数の技術スタックが検出される **WHEN** 検出結果を表示する **THEN** 検出した全ての技術スタックをリストとして表示し、ユーザーに確認する
- **GIVEN** `/setup nextjs` のようにキーワード引数（`$ARGUMENTS`）が指定されている **WHEN** `/setup` を実行する **THEN** 技術スタック自動検出をスキップし、指定されたキーワード「nextjs」で直接 REQ-002 のスキル検索に進む

#### Error Scenarios

- **GIVEN** プロジェクトルートにどのファイルパターンにも一致するファイルが存在しない **WHEN** `/setup` を実行する **THEN** 「技術スタックを自動検出できませんでした。検索したいキーワードを入力してください」と表示し、手動キーワード入力に遷移する
- **GIVEN** プロジェクトルートに `package.json` が存在するがパースエラーが発生する（不正な JSON 等） **WHEN** `/setup` を実行する **THEN** 警告「package.json のパースに失敗しました。依存関係の詳細検出をスキップします」を表示し、ファイル存在のみをもって「Node.js プロジェクト」として検出する
- **GIVEN** `package.json` の dependencies に `next` が含まれるがバージョンが `workspace:*` や `*` など解析不能な形式である **WHEN** バージョン検出を試みる **THEN** 「Next.js（バージョン不明）」として検出し、処理を続行する

#### Boundary Scenarios

- **GIVEN** モノレポ構成で `packages/app/package.json` と `packages/api/prisma/schema.prisma` が存在する **WHEN** `/setup` を実行する **THEN** ルート直下 + 1階層下（`*/` 配下の全サブディレクトリ。例: `packages/*/`, `apps/*/`, `services/*/`）のファイルを検出対象とし、検出された全技術スタックを統合表示する

---

### Requirement: REQ-002 スキル検索（find-skills + GitHub API 並行）

`/setup` コマンドは、検出した技術スタックをキーワードとして skills.sh API と GitHub API を並行で検索し、結果をソース別グループとしてランキング付きで表示しなければならない (SHALL)。

#### Happy Path Scenarios

- **GIVEN** 技術スタックとして「Next.js」が検出されている **WHEN** スキル検索を実行する **THEN** skills.sh API（`npx skills find nextjs`）と GitHub API（awesome-claude-skills, everything-claude-code, トピック検索）を並行で実行し、結果をマージする
- **GIVEN** skills.sh と GitHub API の両方から結果が返る **WHEN** 結果を表示する **THEN** 同名スキルが複数ソースから見つかった場合は重複排除せず両方表示する。ソース別にグループ化して表示する。skills.sh グループを上位に install 数降順で表示し、GitHub グループを下位に star 数降順で表示する。各スキルにソース（skills.sh / GitHub リポジトリ名）・install 数または star 数を明記する
- **GIVEN** GitHub API 検索を実行する **WHEN** `gh` コマンドが利用可能 **THEN** 認証済み `gh api` を使用して Search API レート制限を 30req/min に拡大する

#### Error Scenarios

- **GIVEN** skills.sh API がタイムアウトまたはエラーを返す **WHEN** スキル検索を実行する **THEN** 「skills.sh からの検索に失敗しました。GitHub からの検索結果のみ表示します」と警告を表示し、GitHub API の結果のみで続行する
- **GIVEN** GitHub API がレート制限（403）を返す **WHEN** スキル検索を実行する **THEN** 「GitHub API のレート制限に達しました。skills.sh からの検索結果のみ表示します」と警告を表示し、skills.sh の結果のみで続行する
- **GIVEN** skills.sh API と GitHub API の両方が失敗する **WHEN** スキル検索を実行する **THEN** 「外部検索に失敗しました。手動でスキル名を入力するか、後で再試行してください」と表示する

#### Boundary Scenarios

- **GIVEN** `npx` コマンドがインストールされていない **WHEN** skills.sh 検索を実行しようとする **THEN** 警告「npx が利用できません。GitHub からの検索のみ実行します」を表示し、GitHub API 検索のみで続行する（エラーではなく warn レベル）
- **GIVEN** `gh` コマンドが未認証状態である **WHEN** GitHub API 検索を実行する **THEN** 未認証レート制限（10req/min）で動作し、「gh auth login を実行すると検索レート制限が緩和されます」と案内を表示する
- **GIVEN** 検索結果が0件である **WHEN** 結果表示を試みる **THEN** 「'{keyword}' に一致するスキルは見つかりませんでした」と表示し、REQ-005 の追加検索フローに遷移する
- **GIVEN** 検索結果が50件を超える **WHEN** 結果を表示する **THEN** skills.sh グループは上位10件、GitHub グループは上位10件のみ表示し、「他にも {N} 件の結果があります。キーワードを絞り込むことで詳細な結果を取得できます」と案内する

#### Non-Functional Requirements

- **RELIABILITY**: skills.sh または GitHub API のいずれかが障害でも、他方の結果のみで続行する graceful degradation を実装する
- **PERFORMANCE**: skills.sh と GitHub API の検索は並行実行し、全体のレスポンス時間を最小化する

---

### Requirement: REQ-003 対話的スキル選択・インストール

`/setup` コマンドは、検索結果をランキング付きで提示し、ユーザーが対話的にスキルを選択・インストール先を指定できなければならない (SHALL)。インストール先のデフォルトはプロジェクトローカルとする。`/setup` でインストールしたスキルは skills-lock.json にソース URL と SHA-256 ハッシュを記録する。

#### Happy Path Scenarios

- **GIVEN** スキル検索結果が表示されている **WHEN** ユーザーがインストールするスキルを番号で選択する **THEN** 選択されたスキルについてインストール先（プロジェクト `<project>/.claude/skills/` / グローバル `~/.claude/skills/`）を確認する。デフォルトはプロジェクト
- **GIVEN** ユーザーがプロジェクトローカルを選択する **WHEN** スキルをインストールする **THEN** `<project>/.claude/skills/<skill-name>/SKILL.md` にインストールする
- **GIVEN** ユーザーがグローバルを選択する **WHEN** スキルをインストールする **THEN** `~/.claude/skills/<skill-name>/SKILL.md` にインストールする
- **GIVEN** skills.sh 経由のスキルを選択する **WHEN** インストールを実行する **THEN** `npx skills add {owner}/{repo} --skill {name}` を使用してインストールする
- **GIVEN** GitHub リポジトリ経由のスキルを選択する **WHEN** インストールを実行する **THEN** `gh api` でリポジトリから SKILL.md を取得し、対象ディレクトリにコピーする。検索ソースごとの取得パスは以下の通り:
  - awesome-claude-skills: `{skill-name}/SKILL.md` またはルート直下のサブディレクトリ
  - everything-claude-code: `skills/{skill-name}/SKILL.md`
  - 個別リポジトリ: ルート直下の `SKILL.md` または `.claude/skills/{name}/SKILL.md`
- **GIVEN** スキルのインストールが完了する **WHEN** skills-lock.json を更新する **THEN** インストールしたスキルのエントリ（ソース URL + SHA-256 ハッシュ）を skills-lock.json に追記する

#### Error Scenarios

- **GIVEN** `npx skills add` の実行が失敗する **WHEN** スキルインストールを実行する **THEN** エラーメッセージを表示し、「手動でインストールしますか？リポジトリ URL: {url}」と代替手段を提示する
- **GIVEN** GitHub リポジトリから SKILL.md の取得が失敗する **WHEN** スキルインストールを実行する **THEN** エラーメッセージを表示し、リポジトリ URL を提示して手動クローンを案内する
- **GIVEN** ユーザーが表示された番号の範囲外の値を入力する **WHEN** スキル選択を行う **THEN** 「無効な番号です。1-{max} の範囲で入力してください」と表示し、再入力を促す
- **GIVEN** スキルのインストールは成功したが skills-lock.json の書き込みに失敗する **WHEN** skills-lock.json を更新する **THEN** 警告「skills-lock.json の更新に失敗しました。スキルは正常にインストールされています」を表示し、インストール自体はロールバックしない

#### Boundary Scenarios

- **GIVEN** ネットワーク接続がない状態で GitHub リポジトリからの SKILL.md 取得を試みる **WHEN** タイムアウトが発生する **THEN** 「ネットワーク接続を確認してください。接続回復後に再試行するか、手動でインストールしてください」と案内する
- **GIVEN** インストールしようとするスキルと同名のスキルが既に異なるソースからインストール済みである **WHEN** インストールを実行する **THEN** 「'{skill-name}' は既に {existing-source} からインストールされています。上書きしますか？ (y/N)」と確認する

---

### Requirement: REQ-004 セキュリティ検証（インストール前の SKILL.md 要約表示 + ユーザー確認）

`/setup` コマンドは、外部スキルのインストール前に SKILL.md の内容を要約表示し、ユーザーの明示的な確認を得なければならない (SHALL)。

#### Happy Path Scenarios

- **GIVEN** ユーザーがインストールするスキルを選択した **WHEN** インストール前の確認を行う **THEN** そのスキルの SKILL.md の description と主要セクション見出しを要約表示し、ソース URL、star 数（または install 数）、最終更新日を明示表示した上で「このスキルをインストールしますか？ (y/N)」と確認する
- **GIVEN** ユーザーが確認に「y」と回答する **WHEN** インストールを実行する **THEN** スキルのインストールを続行する
- **GIVEN** ユーザーが確認に「N」と回答する **WHEN** インストールをスキップする **THEN** そのスキルをスキップし、次のスキル選択に戻る

#### Error Scenarios

- **GIVEN** SKILL.md の内容を事前に取得できない **WHEN** 要約表示を試みる **THEN** 「SKILL.md の内容を事前に確認できません。ソース: {url}。インストールしますか？ (y/N)」とリスクを明示して確認する

#### Non-Functional Requirements

- **SECURITY**: 外部スキルのインストールは必ずユーザー確認を挟む。自動インストールは行わない。これは YAGNI の例外（セキュリティ防御策には YAGNI を適用しない）
- **SECURITY**: 要約表示時にソース URL、star 数（または install 数）、最終更新日を明示表示し、ユーザーが信頼性を判断できる情報を提供する

---

### Requirement: REQ-005 追加キーワード検索

`/setup` コマンドは、初回の技術スタックベース検索後に追加キーワードによる検索を対話的に繰り返せなければならない (SHALL)。

#### Happy Path Scenarios

- **GIVEN** 初回のスキル検索・インストールが完了した **WHEN** 「他に探したいスキルはありますか？」と質問する **THEN** ユーザーがキーワードを入力した場合、そのキーワードで REQ-002 と同じ検索フローを再実行する
- **GIVEN** 追加検索の結果が表示された **WHEN** ユーザーがスキルを選択する **THEN** REQ-003 および REQ-004 と同じ選択・確認・インストールフローを実行する
- **GIVEN** ユーザーが「いいえ」または空入力で回答する **WHEN** 追加検索の質問に対して **THEN** 追加検索ループを終了し、次のステップ（スキル作成提案）に進む

#### Error Scenarios

- **GIVEN** 追加キーワードで検索した **WHEN** 検索結果が0件 **THEN** 「'{keyword}' に一致するスキルは見つかりませんでした。別のキーワードを試すか、スキップしてください」と表示する

---

### Requirement: REQ-006 スキル作成提案（skill-creator 連携）

`/setup` コマンドは、検索結果が0件、または全ての結果が以下のいずれかに該当する場合、skill-creator を使ったスキル生成を対話的にガイドしなければならない (SHALL): (a) skills.sh ソースのスキルで install 数 1,000 未満、(b) GitHub ソースのスキルで star 数 100 未満。

#### Happy Path Scenarios

- **GIVEN** 検出した技術スタックの一部に対してスキル検索結果が0件、または全ての結果が以下のいずれかに該当する場合: (a) skills.sh ソースのスキルで install 数 1,000 未満、(b) GitHub ソースのスキルで star 数 100 未満 **WHEN** スキル作成提案を行う **THEN** 「以下の技術スタックにはマッチするスキルが見つかりませんでした: {list}。skill-creator で新しいスキルを作成しますか？」と提案する
- **GIVEN** ユーザーが skill-creator の使用を承認する **WHEN** スキル作成をガイドする **THEN** skill-creator に渡すおすすめのプロンプト（技術スタック名 + description 3部構成テンプレート）を提示し、ユーザーがカスタマイズ後に skill-creator を呼び出す
- **GIVEN** ユーザーがスキル作成を辞退する **WHEN** 提案に「いいえ」と回答する **THEN** スキル作成をスキップし、次のステップ（設定ファイル生成）に進む

#### Error Scenarios

- **GIVEN** skill-creator がインストールされていない **WHEN** スキル作成を試みる **THEN** 「skill-creator がインストールされていません。`./install.sh` を再実行して Forge を更新してください」と案内する

---

### Requirement: REQ-007 設定ファイル生成（setup.md + CLAUDE.md）

`/setup` コマンドは、インストール完了後に `<project>/.claude/setup.md` と `<project>/CLAUDE.md` を生成・更新しなければならない (SHALL)。

#### Happy Path Scenarios

- **GIVEN** スキルのインストールが完了した **WHEN** setup.md を生成する **THEN** `<project>/.claude/setup.md` にインストールしたスキルのガイダンステーブル（スキル名 + 簡易説明）を生成する。SKILL.md の内容をインライン展開しない（コンテキスト保護）
- **GIVEN** `<project>/CLAUDE.md` が存在しない **WHEN** CLAUDE.md を生成する **THEN** 以下の構造化テンプレートを新規作成する:

```
# <project-name>

## プロジェクト概要
<!-- プロジェクトの説明を記載 -->

## 技術スタック
<!-- /setup で検出された技術スタック -->

## Available Skills
See: .claude/setup.md

## Available Agents
<!-- 使用するエージェントを記載 -->
```

- **GIVEN** `<project>/CLAUDE.md` が既に存在し `## Available Skills` セクションがある **WHEN** 参照を追記する **THEN** `## Available Skills` セクション内に `See: .claude/setup.md` を追記する。既存内容は変更しない
- **GIVEN** `<project>/CLAUDE.md` が既に存在し `## Available Skills` セクションがない **WHEN** 参照を追記する **THEN** ファイル末尾に `## Available Skills\nSee: .claude/setup.md` を追記する。既存内容は変更しない
- **GIVEN** setup.md を生成する **WHEN** ガイダンステーブルの内容を構成する **THEN** 各スキルについてスキル名・インストール先（プロジェクト/グローバル）・description の1行要約のみを含める

#### Error Scenarios

- **GIVEN** `<project>/.claude/` ディレクトリが存在しない **WHEN** setup.md を生成する **THEN** ディレクトリを作成してから setup.md を生成する

#### Non-Functional Requirements

- **PERFORMANCE**: setup.md はスキル名と簡易説明のみとし、SKILL.md のインライン展開を禁止する。Claude Code のコンテキスト効率を保護する
- **COMPATIBILITY**: 既存の CLAUDE.md の内容を破壊しない。追記のみ許可する

---

### Requirement: REQ-008 冪等性（既存スキルスキップ、差分提案）

`/setup` コマンドは冪等に動作し、再実行時は既存スキルをスキップして新しく検出された技術スタックに対してのみ追加提案しなければならない (SHALL)。

#### Happy Path Scenarios

- **GIVEN** 過去に `/setup` を実行してスキルがインストール済み **WHEN** `/setup` を再実行する **THEN** `<project>/.claude/skills/` と `~/.claude/skills/` をスキャンし、インストール済みスキルを検出して検索結果から除外する
- **GIVEN** 前回実行時になかった新しい技術スタック（例: prisma/schema.prisma を追加した）がある **WHEN** `/setup` を再実行する **THEN** 新しい技術スタックに対してのみスキル検索・提案を行う
- **GIVEN** setup.md が既に存在する **WHEN** `/setup` を再実行する **THEN** setup.md を全件再生成する（マネージドファイル）。新たにインストールしたスキルを含む最新状態にする
- **GIVEN** CLAUDE.md に setup.md への参照が既に含まれている **WHEN** `/setup` を再実行する **THEN** 参照の重複追記を行わない

#### Error Scenarios

- **GIVEN** 前回インストールしたスキルのディレクトリが手動削除されている **WHEN** `/setup` を再実行する **THEN** 削除されたスキルを未インストールとして扱い、再度検索結果に含める

---

### Requirement: REQ-009 skill-creator の Forge リポへの同梱

skill-creator（`anthropics/skills/skill-creator`）を Forge リポジトリの `skills/skill-creator/` ディレクトリに同梱し、install.sh の既存ロジックでグローバルにインストールされるようにしなければならない (SHALL)。skills-lock.json にもエントリを追加する。

#### Happy Path Scenarios

- **GIVEN** Forge リポの `skills/skill-creator/SKILL.md` が存在する **WHEN** `./install.sh` を実行する **THEN** `~/.claude/skills/skill-creator/` にシンボリックリンクが作成される（install.sh の既存ロジック）
- **GIVEN** skills-lock.json を検証する **WHEN** skill-creator のエントリを確認する **THEN** `"skill-creator": { "source": "anthropics/skills", "sourceType": "github", "computedHash": "..." }` のエントリが存在する
- **GIVEN** CLAUDE.md の Available Skills テーブルを検証する **WHEN** skill-creator の記載を確認する **THEN** skill-creator が Available Skills テーブルに記載されている

#### Error Scenarios

- **GIVEN** `anthropics/skills` リポジトリから skill-creator を取得できない **WHEN** Forge リポに同梱する **THEN** 取得失敗をログに記録し、手動での配置手順をドキュメントに記載する

#### Non-Functional Requirements

- **COMPATIBILITY**: install.sh への変更は不要。既存の `FORGE_DIRS` ロジック（`skills` ディレクトリの個別要素リンク）で自動的に処理される

---

## MODIFIED Requirements

### Requirement: domain-skills REQ-005 CLAUDE.md の更新

**変更理由**: `/setup` コマンドで skill-creator を Forge に同梱するため、domain-skills REQ-005 の Available Skills テーブルに `skill-creator` を追加する必要がある。

#### Happy Path Scenarios

- **GIVEN** skill-creator が `skills/skill-creator/SKILL.md` として Forge リポに同梱されている **WHEN** CLAUDE.md の Available Skills テーブルを更新する **THEN** 既存の14 Domain Skills に加えて `skill-creator` が Available Skills テーブルに記載されている

#### Error Scenarios

- **GIVEN** skill-creator の SKILL.md が Forge リポに存在しない **WHEN** CLAUDE.md を更新する **THEN** skill-creator を Available Skills テーブルに記載せず、REQ-009 の完了を前提条件とする旨をログに記録する

---

## REMOVED Requirements

なし
