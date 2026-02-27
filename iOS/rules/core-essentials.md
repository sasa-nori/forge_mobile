# Core Essentials

常時読み込みされる最小限のルール。詳細は `~/.claude/reference/` を必要に応じて参照。

---

## エスカレーションポリシー

### 必須エスカレーション（`AskUserQuestion` で確認）

- **セキュリティ**: 認証・認可ロジックの設計・変更、Keychain設計、暗号化方式の選択、PII処理方法
- **データ**: CoreData / SwiftData スキーマ変更、マイグレーション戦略、データ整合性に影響する変更
- **アーキテクチャ**: 新モジュール・新ターゲット追加、破壊的API変更、レイヤー構成変更
- **本番環境**: リリースビルド設定、証明書・プロビジョニング変更、App Store 提出設定の変更

### 状況依存エスカレーション（自信がなければ確認）

- 仕様の曖昧性・複数解釈が可能な場合
- 技術的に同等な複数アプローチの選択（例: Combine vs Swift Concurrency）
- スコープ外の関連変更が必要になった場合

### 自律判断OK

- コードフォーマット、SwiftLint 自動修正、ローカル変数リネーム
- 明らかなバグ修正（Optional 未チェック、off-by-one）
- 自モジュール内のリファクタリング、テストコード追加

---

## セキュリティ必須事項

- ハードコードされたシークレット・APIキー・認証情報禁止
- 機密情報（パスワード・トークン・証明書）は Keychain（Security フレームワーク）を使用
- Secure Enclave / CryptoKit による暗号化（高セキュリティ要件時）
- ユーザー入力は必ず検証・サニタイズ（CoreData の NSPredicate インジェクション防止: パラメータ化クエリのみ）
- ATS（App Transport Security）でクリアテキスト通信を禁止（本番）
- `NSAllowsArbitraryLoads = false` を維持すること
- 証明書ピニングを実装（重要 API エンドポイント）
- `#if DEBUG` でデバッグ用コードをリリースビルドから除外
- Privacy Manifest（PrivacyInfo.xcprivacy）を作成・維持（App Store 提出要件）
- SPM 依存関係の脆弱性確認（定期的な `swift package update` と脆弱性チェック）

---

## Git コミット形式

- 形式: `<type>(<scope>): <日本語の説明>`
- type: `feat` / `fix` / `refactor` / `test` / `docs` / `chore` / `perf`
- 1コミット = 1つの論理的変更、動く状態でコミット

---

## オンデマンドルール参照先

作業対象に応じて `~/.claude/reference/` から必要なファイルを読み込むこと:

| ファイル | 読み込むタイミング |
|---|---|
| `reference/swift-rules.md` | Swift実装・型設計・慣用句確認時 |
| `reference/coding-standards.md` | コーディング規約の確認時 |
| `reference/core-rules.md` | フェーズ管理・検証ゲート確認時 |
| `reference/workflow-rules.md` | セッション管理・チェックポイント時 |
| `reference/common/coding-style.md` | ファイルサイズ・命名規約確認時 |
| `reference/common/testing.md` | テスト作成・TDD実践時 |
| `reference/common/performance.md` | パフォーマンス最適化時 |
| `reference/ios/conventions.md` | iOS全般・AppDelegate/SceneDelegate・Xcode作業時 |
| `reference/swiftui/conventions.md` | SwiftUI UI実装時 |
| `reference/core-data/conventions.md` | CoreData/SwiftDataスキーマ・DAO・マイグレーション時 |
| `reference/networking/conventions.md` | URLSession/Alamofire APIレイヤー実装時 |
