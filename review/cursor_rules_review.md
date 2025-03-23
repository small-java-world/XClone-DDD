# Cursorルールパターン適用のレビュー

## 想定ファイルパス一覧と適用ルール確認

| ファイルパス | 適用されるルール |
|------------|----------------|
| `src/main/kotlin/com/example/domain/model/user/User.kt` | entity-factory-patterns, git-workflow |
| `src/main/kotlin/com/example/domain/model/value/Email.kt` | value-object-patterns, git-workflow |
| `src/main/kotlin/com/example/domain/model/UserId.kt` | value-object-patterns, git-workflow |
| `src/main/kotlin/com/example/domain/repository/UserRepository.kt` | backend-repository-patterns, git-workflow |
| `src/main/kotlin/com/example/infrastructure/repository/UserRepositoryImpl.kt` | backend-repository-patterns, git-workflow |
| `src/main/kotlin/com/example/application/usecase/user/CreateUserUseCase.kt` | usecase-command-pattern, git-workflow |
| `src/main/kotlin/com/example/application/usecase/user/GetUserUseCase.kt` | usecase-query-pattern, git-workflow |
| `src/main/kotlin/com/example/application/usecase/user/FindUserQueryUseCase.kt` | usecase-query-pattern, git-workflow |

## ルール適用の妥当性とグロブパターンの重複レビュー

### 重複パターンの確認

1. **問題点**: `entity-factory-patterns`と`value-object-patterns`のglobsパターンで競合の可能性
   - entity-factory-patterns: `"**/domain/model/**/*.kt"`
   - value-object-patterns: `"**/*Id.kt"`, `"**/*Name.kt"`, `"**/*Status.kt"`
   - **解決策**: entity-factory-patternsでは除外パターン`"!**/domain/model/value/*.kt"`を使用して明確に分離

2. **問題点**: IDファイルに複数のルールが適用される可能性
   - `UserId.kt`に`value-object-patterns`と`uuid-id-type`(参照されている)の両方が適用される
   - **解決策**: 関連ルールセクションで明確に関係性が説明されている

### ルール内容の整合性

1. **問題点**: 一部のルールで「関連ルール」として参照されるルールが不完全
   - 例: `uuid-id-type`が参照されているが、全ての関連ルールで言及されているわけではない
   - **解決策**: 各ルールの「関連ルール」セクションを統一的に整備

2. **問題点**: `git-workflow`は全ファイルに適用されるため、他のルールと競合する可能性
   - **解決策**: このルールはコード実装に関するルールではないことを明記

## AIエージェントの自走性向上の観点からのレビュー

### 改善されている点

1. **クイックスタートセクション**
   - 各ルールの冒頭に簡潔な要約があり、AIが迅速に理解できる

2. **実装判断フロー**
   - 決定木形式のガイダンスにより、AIが判断プロセスを理解しやすい

3. **最小実装テンプレート**
   - 簡潔なコード例により、AIが基本パターンを迅速に把握できる

4. **複数ルール適用時の優先順位**
   - ルール間の重複適用時の指針が明記されている

5. **適用シナリオの明確化**
   - どのような状況でルールを適用すべきかが明確

### さらなる改善点

1. **相互依存関係の視覚化**
   - ルール間の依存関係をグラフィカルに表現する図の追加

2. **実装順序の明確化**
   - 「実装を始める際は、まず値オブジェクト→エンティティ→リポジトリの順に取り組む」という指針の追加

3. **ルール間のナビゲーション改善**
   - ルール内で関連ルールへの言及を明確なフォーマットで統一（例: `「@see value-object-patterns」`）

4. **共通セクション構造の統一**
   - 全てのルールで同じセクション構造と順序を保つこと
   - 例: 「適用シナリオ」→「関連ルール」→「クイックスタート」→「実装判断フロー」

5. **チェックリストの活用**
   - 実装チェックリストは維持しつつ、より明確な形式に（例: `- [ ] リポジトリインターフェースをドメイン層に作成`）

## 総合評価

現状のルールファイルは、AIエージェントが自走しやすいように以下の点で十分に最適化されています：

- 階層化された情報（重要→詳細）
- 実装判断の明確なガイダンス
- 関連ルールの明示
- 最小実装テンプレートの提供
- グロブパターンの重複への対処

ただし、異なるルール間でセクション構造や表現を統一し、関連ルールへの言及を標準化することでさらに改善できます。また、ルール間の依存関係を視覚的に表現することで、AIエージェントがより効率的に作業を進められるでしょう。 