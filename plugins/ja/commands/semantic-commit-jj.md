## Semantic Commit

大きな変更を意味のある最小単位に分割して、セマンティックなコミットメッセージと共に順次コミットします。jujutsu (jj) 標準コマンドのみを使用します。

### 使い方

```bash
/semantic-commit-jj [オプション]
```

### オプション

- `--dry-run` : 実際のコミットは行わず、提案されるコミット分割のみを表示
- `--lang <言語>` : コミットメッセージの言語を強制指定 (en, ja)
- `--max-commits <数>` : 最大コミット数を指定 (デフォルト: 10)

### 基本例

```bash
# 現在の変更を分析して、論理的な単位でコミット
/semantic-commit-jj

# 分割案のみを確認 (実際のコミットなし)
/semantic-commit-jj --dry-run

# 英語でコミットメッセージを生成
/semantic-commit-jj --lang en

# 日本語でコミットメッセージを生成
/semantic-commit-jj --lang ja

# 最大 5 個のコミットに分割
/semantic-commit-jj --max-commits 5
```

### 動作フロー

1. **変更分析**: `jj diff` で全変更を取得
2. **ファイル分類**: 変更されたファイルを論理的にグループ化
3. **コミット提案**: 各グループに対してセマンティックなコミットメッセージを生成
4. **順次実行**: ユーザー確認後、各グループを順次コミット

### 変更分割の核心機能

#### 「大きな変更」の検出

以下の条件で大きな変更として検出：

1. **変更ファイル数**: 5 ファイル以上の変更
2. **変更行数**: 100 行以上の変更
3. **複数機能**: 2 つ以上の機能領域にまたがる変更
4. **混在パターン**: feat + fix + docs が混在

```bash
# 変更規模の分析
CHANGED_FILES=$(jj diff --summary | wc -l)
CHANGED_LINES=$(jj diff --stat | tail -1 | grep -o '[0-9]\+ insertions\|[0-9]\+ deletions' | awk '{sum+=$1} END {print sum}')

if [ $CHANGED_FILES -ge 5 ] || [ $CHANGED_LINES -ge 100 ]; then
  echo "大きな変更を検出: 分割を推奨"
fi
```

#### 「意味のある最小単位」への分割戦略

##### 1. 機能境界による分割

```bash
# ディレクトリ構造から機能単位を特定
jj diff --summary | cut -d' ' -f2- | cut -d'/' -f1-2 | sort | uniq
# → src/auth, src/api, components/ui など
```

##### 2. 変更種別による分離

```bash
# 新規ファイル vs 既存ファイル修正
jj diff --summary | grep '^A' # 新規ファイル
jj diff --summary | grep '^M' # 修正ファイル
jj diff --summary | grep '^D' # 削除ファイル
```

##### 3. 依存関係の分析

```bash
# インポート関係の変更を検出
jj diff | grep -E '^[+-].*import|^[+-].*require' | \
cut -d' ' -f2- | sort | uniq
```

#### ファイル単位の詳細分析

```bash
# 変更されたファイル一覧を取得
jj diff --summary

# 各ファイルの変更内容を個別に分析
jj diff <file>

# ファイルの変更タイプを判定
jj diff --summary | while read status file; do
  case $status in
    A) echo "$file: 新規作成" ;;
    M) echo "$file: 修正" ;;
    D) echo "$file: 削除" ;;
    R*) echo "$file: リネーム" ;;
  esac
done
```

#### 論理的グループ化の基準

1. **機能単位**: 同一機能に関連するファイル
   - `src/auth/` 配下のファイル → 認証機能
   - `components/` 配下のファイル → UI コンポーネント

2. **変更種別**: 同じ種類の変更
   - テストファイルのみ → `test:`
   - ドキュメントのみ → `docs:`
   - 設定ファイルのみ → `chore:`

3. **依存関係**: 相互に関連するファイル
   - モデル + マイグレーション
   - コンポーネント + スタイル

4. **変更規模**: 適切なコミットサイズの維持
   - 1 コミットあたり 10 ファイル以下
   - 関連性の高いファイルをグループ化

### 出力例

```bash
$ /semantic-commit-jj

変更分析中...
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

検出された変更:
• src/auth/login.ts (修正)
• src/auth/register.ts (新規)
• src/auth/types.ts (修正)
• tests/auth.test.ts (新規)
• docs/authentication.md (新規)

提案されるコミット分割:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
コミット 1/3
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
メッセージ: feat: implement user registration and login system
含まれるファイル:
  • src/auth/login.ts
  • src/auth/register.ts
  • src/auth/types.ts

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
コミット 2/3
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
メッセージ: test: add comprehensive tests for authentication system
含まれるファイル:
  • tests/auth.test.ts

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
コミット 3/3
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
メッセージ: docs: add authentication system documentation
含まれるファイル:
  • docs/authentication.md

この分割案でコミットを実行しますか？ (y/n/edit):
```

### 実行時の選択肢

- `y` : 提案されたコミット分割で実行
- `n` : キャンセル
- `edit` : コミットメッセージを個別に編集
- `merge <番号 1> <番号 2>` : 指定したコミットをマージ
- `split <番号>` : 指定したコミットをさらに分割

### Dry Run モード

```bash
$ /semantic-commit-jj --dry-run

変更分析中... (DRY RUN)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[コミット分割提案の表示]

ℹ️  DRY RUN モード: 実際のコミットは実行されません
💡 実行する場合は --dry-run オプションを除いて再実行してください
```

### スマート分析機能

#### 1. プロジェクト構造の理解

- `package.json`, `Cargo.toml`, `pom.xml` などからプロジェクト種別を判定
- フォルダ構造から機能単位を推測

#### 2. 変更パターンの認識

```bash
# バグ修正パターンの検出
- "fix", "bug", "error" などのキーワード
- 例外処理の追加
- 条件分岐の修正

# 新機能パターンの検出
- 新ファイル作成
- 新メソッド追加
- API エンドポイント追加
```

#### 3. 依存関係の分析

- インポート文の変更
- 型定義の追加/修正
- 設定ファイルとの関連性

### 技術的実装

#### jj 標準コマンドによる順次コミット実装

##### 1. グループ別の順次コミット実行

```bash
# 該当ファイルのみをsplit
jj split -m "コミットメッセージ" <該当ファイル>
```

##### 2. 完了後の検証

```bash
# 全変更がコミットされたかの確認
remaining_changes=$(jj diff --summary | grep -E '^[AMD]' | wc -l)
if [ $remaining_changes -eq 0 ]; then
  echo "✅ すべての変更がコミットされました"
else
  echo "⚠️  未コミットの変更が残っています:"
  jj diff --summary
fi

# コミット履歴の表示
echo "作成されたコミット:"
jj log -r 'all()' --limit 10
```

##### 3. 自動プッシュの抑制

```bash
# 注意: 自動プッシュは行わない
echo "📝 注意: 自動プッシュは実行されません"
echo "必要に応じて以下のコマンドでプッシュしてください:"
echo "  jj git push"
```

#### 分割アルゴリズムの詳細

##### ステップ 1: 初期分析

```bash
# 全変更ファイルの取得と分類
jj diff --summary

# 機能ディレクトリ別の変更統計
jj diff --summary | grep -E '^[AMD]' | awk '{print $2}' | cut -d'/' -f1-2 | sort | uniq -c
```

##### ステップ 2: 機能境界による初期グループ化

```bash
# ディレクトリベースのグループ化
GROUPS=$(jj diff --summary | grep -E '^[AMD]' | awk '{print $2}' | cut -d'/' -f1-2 | sort | uniq)
for group in $GROUPS; do
  echo "=== グループ: $group ==="
  jj diff --summary | grep -E '^[AMD]' | awk '{print $2}' | grep "^$group" | head -10
done
```

##### ステップ 3: 変更内容の類似性分析

```bash
# 各ファイルの変更タイプを分析
jj diff --summary | grep -E '^[AMD]' | awk '{print $2}' | while read file; do
  # 新規関数/クラス追加の検出
  NEW_FUNCTIONS=$(jj diff "$file" | grep -c '^+.*function\|^+.*class\|^+.*def ')

  # バグ修正パターンの検出
  BUG_FIXES=$(jj diff "$file" | grep -c '^+.*fix\|^+.*bug\|^-.*error')

  # テストファイルかの判定
  if [[ "$file" =~ test|spec ]]; then
    echo "$file: TEST"
  elif [ $NEW_FUNCTIONS -gt 0 ]; then
    echo "$file: FEAT"
  elif [ $BUG_FIXES -gt 0 ]; then
    echo "$file: FIX"
  else
    echo "$file: REFACTOR"
  fi
done
```

##### ステップ 4: 依存関係による調整

```bash
# インポート関係の分析
jj diff | grep -E '^[+-].*import|^[+-].*from.*import' | \
while read line; do
  echo "$line" | sed 's/^[+-]//' | awk '{print $2}'
done | sort | uniq

# 関連ファイルのグループ化
jj diff --summary | grep -E '^[AMD]' | awk '{print $2}' | while read file; do
  basename=$(basename "$file" .js .ts .py)
  related=$(jj diff --summary | grep -E '^[AMD]' | awk '{print $2}' | grep "$basename" | grep -v "^$file$")
  if [ -n "$related" ]; then
    echo "関連ファイル群: $file <-> $related"
  fi
done
```

##### ステップ 5: コミットサイズの最適化

```bash
# グループサイズの調整
MAX_FILES_PER_COMMIT=8
current_group=1
file_count=0

jj diff --summary | grep -E '^[AMD]' | awk '{print $2}' | while read file; do
  if [ $file_count -ge $MAX_FILES_PER_COMMIT ]; then
    current_group=$((current_group + 1))
    file_count=0
  fi
  echo "コミット $current_group: $file"
  file_count=$((file_count + 1))
done > /tmp/commit_plan.txt
```

##### ステップ 6: 最終グループ決定

```bash
# 分割結果の検証
for group in $(seq 1 $current_group); do
  files=$(grep "コミット $group:" /tmp/commit_plan.txt | cut -d':' -f2-)
  lines=$(echo "$files" | xargs jj diff -- | wc -l)
  echo "コミット $group: $(echo "$files" | wc -w) ファイル, $lines 行変更"
done
```

### Conventional Commits 準拠

#### 基本形式

```text
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

#### 標準タイプ

**必須タイプ**:

- `feat`: 新機能 (ユーザーに見える機能追加)
- `fix`: バグ修正

**任意タイプ**:

- `build`: ビルドシステムや外部依存関係の変更
- `chore`: その他の変更 (リリースに影響しない)
- `ci`: CI 設定ファイルやスクリプトの変更
- `docs`: ドキュメントのみの変更
- `style`: コードの意味に影響しない変更 (空白、フォーマット、セミコロンなど)
- `refactor`: バグ修正や機能追加を伴わないコード変更
- `perf`: パフォーマンス改善
- `test`: テストの追加や修正

#### スコープ (任意)

変更の影響範囲を示す：

```text
feat(api): add user authentication endpoint
fix(ui): resolve button alignment issue
docs(readme): update installation instructions
```

#### Breaking Change

API の破壊的変更がある場合：

```text
feat!: change user API response format

BREAKING CHANGE: user response now includes additional metadata
```

または

```text
feat(api)!: change authentication flow
```

#### プロジェクト規約の自動検出

**重要**: プロジェクト独自の規約が存在する場合は、それを優先します。

##### 1. CommitLint 設定の確認

以下のファイルから設定を自動検出：

- `commitlint.config.js`
- `commitlint.config.mjs`
- `commitlint.config.cjs`
- `commitlint.config.ts`
- `.commitlintrc.js`
- `.commitlintrc.json`
- `.commitlintrc.yml`
- `.commitlintrc.yaml`
- `package.json` の `commitlint` セクション

```bash
# 設定ファイル例の確認
cat commitlint.config.mjs
cat .commitlintrc.json
grep -A 10 '"commitlint"' package.json
```

##### 2. カスタムタイプの検出

プロジェクト独自のタイプ例：

```javascript
// commitlint.config.mjs
export default {
  extends: ["@commitlint/config-conventional"],
  rules: {
    "type-enum": [
      2,
      "always",
      [
        "feat",
        "fix",
        "docs",
        "style",
        "refactor",
        "test",
        "chore",
        "wip", // 作業中
        "hotfix", // 緊急修正
        "release", // リリース
        "deps", // 依存関係更新
        "config", // 設定変更
      ],
    ],
  },
};
```

##### 3. 言語設定の検出

```javascript
// プロジェクトが日本語メッセージを使用する場合
export default {
  rules: {
    "subject-case": [0], // 日本語対応のため無効化
    "subject-max-length": [2, "always", 72], // 日本語は文字数制限を調整
  },
};
```

#### 自動分析の流れ

1. **設定ファイル検索**

   ```bash
   find . -name "commitlint.config.*" -o -name ".commitlintrc.*" | head -1
   ```

2. **既存コミット分析**

   ```bash
   jj log -r 'all()' --limit 50 -T 'description' -G
   ```

3. **使用タイプ統計**

   ```bash
   jj log -r 'all()' --limit 100 -T 'description' -G | \
   grep -oE '^[a-z]+(\([^)]+\))?' | \
   sort | uniq -c | sort -nr
   ```

#### プロジェクト規約の例

##### Angular スタイル

```text
feat(scope): add new feature
fix(scope): fix bug
docs(scope): update documentation
```

##### Gitmoji 併用スタイル

```text
✨ feat: add user registration
🐛 fix: resolve login issue
📚 docs: update API docs
```

##### 日本語プロジェクト

```text
feat: ユーザー登録機能を追加
fix: ログイン処理のバグを修正
docs: API ドキュメントを更新
```

### 言語判定

このコマンドで完結する言語判定ロジック：

1. **CommitLint 設定**から言語設定を確認

   ```bash
   # subject-case ルールが無効化されている場合は日本語と判定
   grep -E '"subject-case".*\[0\]|subject-case.*0' commitlint.config.*
   ```

2. **jj log 分析**による自動判定

   ```bash
   # 最近 20 コミットの言語を分析
   jj log -r 'all()' --limit 20 -T 'description' -G | \
   grep -E '^[あ-ん]|[ア-ン]|[一-龯]' | wc -l
   # 50% 以上が日本語なら日本語モード
   ```

3. **プロジェクトファイル**の言語設定

   ```bash
   # README.md の言語確認
   head -10 README.md | grep -E '^[あ-ん]|[ア-ン]|[一-龯]' | wc -l

   # package.json の description 確認
   grep -E '"description".*[あ-ん]|[ア-ン]|[一-龯]' package.json
   ```

4. **変更ファイル内**のコメント・文字列分析

   ```bash
   # 変更されたファイルのコメント言語を確認
   jj diff | grep -E '^[+-].*//.*[あ-ん]|[ア-ン]|[一-龯]' | wc -l
   ```

#### 判定アルゴリズム

```bash
# 言語判定スコア計算
JAPANESE_SCORE=0

# 1. CommitLint 設定 (+3 点)
if grep -q '"subject-case".*\[0\]' commitlint.config.* 2>/dev/null; then
  JAPANESE_SCORE=$((JAPANESE_SCORE + 3))
fi

# 2. jj log 分析 (最大+2 点)
JAPANESE_COMMITS=$(jj log -r 'all()' --limit 20 -T 'description' | \
  grep -cE '[あ-ん]|[ア-ン]|[一-龯]' 2>/dev/null || echo 0)
if [ $JAPANESE_COMMITS -gt 10 ]; then
  JAPANESE_SCORE=$((JAPANESE_SCORE + 2))
elif [ $JAPANESE_COMMITS -gt 5 ]; then
  JAPANESE_SCORE=$((JAPANESE_SCORE + 1))
fi

# 3. README.md 確認 (+1 点)
if head -5 README.md 2>/dev/null | grep -qE '[あ-ん]|[ア-ン]|[一-龯]'; then
  JAPANESE_SCORE=$((JAPANESE_SCORE + 1))
fi

# 4. 変更ファイル内容確認 (+1 点)
if jj diff 2>/dev/null | grep -qE '^[+-].*[あ-ん]|[ア-ン]|[一-龯]'; then
  JAPANESE_SCORE=$((JAPANESE_SCORE + 1))
fi

# 判定: 3 点以上で日本語モード
if [ $JAPANESE_SCORE -ge 3 ]; then
  LANGUAGE="ja"
else
  LANGUAGE="en"
fi
```

### 設定ファイル自動読み込み

#### 実行時の動作

コマンド実行時に以下の順序で設定を確認：

1. **CommitLint 設定ファイルの検索**

   ```bash
   # 以下の順序で検索し、最初に見つかったファイルを使用
   commitlint.config.mjs
   commitlint.config.js
   commitlint.config.cjs
   commitlint.config.ts
   .commitlintrc.js
   .commitlintrc.json
   .commitlintrc.yml
   .commitlintrc.yaml
   package.json (commitlint セクション)
   ```

2. **設定内容の解析**
   - 使用可能なタイプの一覧を抽出
   - スコープの制限があるかを確認
   - メッセージ長制限の取得
   - 言語設定の確認

3. **既存コミット履歴の分析**

   ```bash
   # 最近のコミットから使用パターンを学習
   jj log -r 'all()' --limit 100 -T 'description' | head -20
   ```

#### 設定例の分析

**標準的な commitlint.config.mjs**:

```javascript
export default {
  extends: ["@commitlint/config-conventional"],
  rules: {
    "type-enum": [
      2,
      "always",
      ["feat", "fix", "docs", "style", "refactor", "perf", "test", "chore"],
    ],
    "scope-enum": [2, "always", ["api", "ui", "core", "auth", "db"]],
  },
};
```

**日本語対応の設定**:

```javascript
export default {
  extends: ["@commitlint/config-conventional"],
  rules: {
    "subject-case": [0], // 日本語のため無効化
    "subject-max-length": [2, "always", 72],
    "type-enum": [
      2,
      "always",
      ["feat", "fix", "docs", "style", "refactor", "test", "chore"],
    ],
  },
};
```

**カスタムタイプを含む設定**:

```javascript
export default {
  extends: ["@commitlint/config-conventional"],
  rules: {
    "type-enum": [
      2,
      "always",
      [
        "feat",
        "fix",
        "docs",
        "style",
        "refactor",
        "test",
        "chore",
        "wip", // Work in Progress
        "hotfix", // 緊急修正
        "release", // リリース準備
        "deps", // 依存関係更新
        "config", // 設定変更
      ],
    ],
  },
};
```

#### フォールバック動作

設定ファイルが見つからない場合：

1. **jj log 分析**による自動推測

   ```bash
   # 最近 100 コミットからタイプを抽出
   jj log -r 'all()' --limit 100 -T 'description' -G | \
   grep -oE '^[a-z]+(\([^)]+\))?' | \
   sort | uniq -c | sort -nr
   ```

2. **Conventional Commits 標準**をデフォルト使用

   ```
   feat, fix, docs, style, refactor, perf, test, chore, build, ci
   ```

3. **言語判定**
   - 日本語コミットが 50% 以上 → 日本語モード
   - その他 → 英語モード

### 前提条件

- jj リポジトリ内で実行
- 未コミットの変更が存在すること

### 注意事項

- **自動プッシュなし**: コミット後の `jj git push` は手動実行
- **ブックマーク作成なし**: 現在のブックマークでコミット
- **バックアップ推奨**: 重要な変更前には `jj new` で新しいリビジョンを作成してバックアップ

### プロジェクト規約の優先度

コミットメッセージ生成時の優先度：

1. **CommitLint 設定** (最優先)
   - `commitlint.config.*` ファイルの設定
   - カスタムタイプやスコープの制限
   - メッセージ長やケースの制限

2. **既存コミット履歴** (第 2 優先)
   - 実際に使用されているタイプの統計
   - メッセージの言語 (日本語/英語)
   - スコープの使用パターン

3. **プロジェクト種別** (第 3 優先)
   - `package.json` → Node.js プロジェクト
   - `Cargo.toml` → Rust プロジェクト
   - `pom.xml` → Java プロジェクト

4. **Conventional Commits 標準** (フォールバック)
   - 設定が見つからない場合の標準動作

#### 規約検出の実例

**Monorepo での scope 自動検出**:

```bash
# packages/ フォルダから scope を推測
ls packages/ | head -10
# → api, ui, core, auth などを scope として提案
```

**フレームワーク固有の規約**:

```javascript
// Angular プロジェクトの場合
{
  'scope-enum': [2, 'always', [
    'animations', 'common', 'core', 'forms', 'http', 'platform-browser',
    'platform-server', 'router', 'service-worker', 'upgrade'
  ]]
}

// React プロジェクトの場合
{
  'scope-enum': [2, 'always', [
    'components', 'hooks', 'utils', 'types', 'styles', 'api'
  ]]
}
```

**企業・チーム固有の規約**:

```javascript
// 日本の企業でよく見られるパターン
{
  'type-enum': [2, 'always', [
    'feat', 'fix', 'docs', 'style', 'refactor', 'test', 'chore',
    'wip',      // 作業中 (プルリクエスト用)
    'hotfix',   // 緊急修正
    'release'   // リリース準備
  ]],
  'subject-case': [0],  // 日本語対応
  'subject-max-length': [2, 'always', 72]  // 日本語は長めに設定
}
```

### ベストプラクティス

1. **プロジェクト規約の尊重**: 既存の設定やパターンに従う
2. **小さな変更単位**: 1 つのコミットは 1 つの論理的変更
3. **明確なメッセージ**: 何を変更したかが明確
4. **関連性の重視**: 機能的に関連するファイルをグループ化
5. **テストの分離**: テストファイルは別コミットに
6. **設定ファイルの活用**: CommitLint を導入してチーム全体で規約を統一

### jj 特有の機能の活用

#### jj split による対話的な分割

```bash
# 現在の変更を分割
jj split -m "コミットメッセージ" <該当ファイル>

# 各パートに適切な説明を設定
jj describe -m "feat: add user authentication"
jj edit @-  # 親リビジョンに移動
jj describe -m "test: add authentication tests"
```

#### jj squash による変更の統合

```bash
# 複数の小さな変更を統合
jj squash --into @-

# 統合後のメッセージを編集
jj describe -m "feat: implement complete authentication system"
```

#### jj new による作業の分岐

```bash
# 新しい変更を開始
jj new

# 特定のファイルのみ復元
jj restore --from @- src/auth/login.ts
```

### 実際の分割例 (Before/After)

#### 例 1: 大規模な認証システム追加

**Before(1 つの巨大なリビジョン):**

```bash
# 変更されたファイル (15 ファイル、850 行変更)
src/auth/login.js          # 新規作成
src/auth/register.js       # 新規作成
src/auth/password.js       # 新規作成
src/auth/types.js          # 新規作成
src/api/auth-routes.js     # 新規作成
src/middleware/auth.js     # 新規作成
src/database/migrations/001_users.sql  # 新規作成
src/database/models/user.js            # 新規作成
tests/auth/login.test.js   # 新規作成
tests/auth/register.test.js # 新規作成
tests/api/auth-routes.test.js # 新規作成
docs/authentication.md    # 新規作成
package.json              # 依存関係追加
README.md                 # 使用方法追加
.env.example             # 環境変数例追加

# 従来の問題のあるリビジョン
feat: implement complete user authentication system with login, registration, password reset, API routes, database models, tests and documentation
```

**After(意味のある 5 つのリビジョンに分割):**

```bash
# リビジョン 1: データベース基盤
feat(db): add user model and authentication schema

含まれるファイル:
- src/database/migrations/001_users.sql
- src/database/models/user.js
- src/auth/types.js

理由: データベース構造は他の機能の基盤となるため最初にコミット

# リビジョン 2: 認証ロジック
feat(auth): implement core authentication functionality

含まれるファイル:
- src/auth/login.js
- src/auth/register.js
- src/auth/password.js
- src/middleware/auth.js

理由: 認証の核となるビジネスロジックを一括でコミット

# リビジョン 3: API エンドポイント
feat(api): add authentication API routes

含まれるファイル:
- src/api/auth-routes.js

理由: API レイヤーは認証ロジックに依存するため後でコミット

# リビジョン 4: 包括的なテスト
test(auth): add comprehensive authentication tests

含まれるファイル:
- tests/auth/login.test.js
- tests/auth/register.test.js
- tests/api/auth-routes.test.js

理由: 実装完了後にテストを一括追加

# リビジョン 5: 設定とドキュメント
docs(auth): add authentication documentation and configuration

含まれるファイル:
- docs/authentication.md
- package.json
- README.md
- .env.example

理由: ドキュメントと設定は最後にまとめてコミット
```

#### 例 2: バグ修正とリファクタリングの混在

**Before(混在した問題のあるリビジョン):**

```bash
# 変更されたファイル (8 ファイル、320 行変更)
src/user/service.js       # バグ修正 + リファクタリング
src/user/validator.js     # 新規作成 (リファクタリング)
src/auth/middleware.js    # バグ修正
src/api/user-routes.js    # バグ修正 + エラーハンドリング改善
tests/user.test.js        # テスト追加
tests/auth.test.js        # バグ修正テスト追加
docs/user-api.md          # ドキュメント更新
package.json              # 依存関係更新

# 問題のあるリビジョン
fix: resolve user validation bugs and refactor validation logic with improved error handling
```

**After(種別別に 3 つのリビジョンに分割):**

```bash
# リビジョン 1: 緊急バグ修正
fix: resolve user validation and authentication bugs

含まれるファイル:
- src/user/service.js(バグ修正部分のみ)
- src/auth/middleware.js
- tests/auth.test.js(バグ修正テストのみ)

理由: 本番環境に影響するバグは最優先で修正

# リビジョン 2: バリデーションロジックのリファクタリング
refactor: extract and improve user validation logic

含まれるファイル:
- src/user/service.js(リファクタリング部分)
- src/user/validator.js
- src/api/user-routes.js
- tests/user.test.js

理由: 構造改善は機能単位でまとめてコミット

# リビジョン 3: ドキュメントと依存関係更新
chore: update documentation and dependencies

含まれるファイル:
- docs/user-api.md
- package.json

理由: 開発環境の整備は最後にまとめてコミット
```

#### 例 3: 複数機能の同時開発

**Before(機能横断の巨大リビジョン):**

```bash
# 変更されたファイル (12 ファイル、600 行変更)
src/user/profile.js       # 新機能 A
src/user/avatar.js        # 新機能 A
src/notification/email.js # 新機能 B
src/notification/sms.js   # 新機能 B
src/api/profile-routes.js # 新機能 A 用 API
src/api/notification-routes.js # 新機能 B 用 API
src/dashboard/widgets.js  # 新機能 C
src/dashboard/charts.js   # 新機能 C
tests/profile.test.js     # 新機能 A 用テスト
tests/notification.test.js # 新機能 B 用テスト
tests/dashboard.test.js   # 新機能 C 用テスト
package.json              # 全機能の依存関係

# 問題のあるリビジョン
feat: add user profile management, notification system and dashboard widgets
```

**After(機能別に 4 つのリビジョンに分割):**

```bash
# リビジョン 1: ユーザープロフィール機能
feat(profile): add user profile management

含まれるファイル:
- src/user/profile.js
- src/user/avatar.js
- src/api/profile-routes.js
- tests/profile.test.js

理由: プロフィール機能は独立した機能単位

# リビジョン 2: 通知システム
feat(notification): implement email and SMS notifications

含まれるファイル:
- src/notification/email.js
- src/notification/sms.js
- src/api/notification-routes.js
- tests/notification.test.js

理由: 通知機能は独立した機能単位

# リビジョン 3: ダッシュボードウィジェット
feat(dashboard): add interactive widgets and charts

含まれるファイル:
- src/dashboard/widgets.js
- src/dashboard/charts.js
- tests/dashboard.test.js

理由: ダッシュボード機能は独立した機能単位

# リビジョン 4: 依存関係とインフラ更新
chore: update dependencies for new features

含まれるファイル:
- package.json

理由: 共通の依存関係更新は最後にまとめて
```

### 分割効果の比較

| 項目           | Before(巨大リビジョン)        | After(適切な分割)                       |
| -------------- | ----------------------------- | --------------------------------------- |
| **レビュー性** | ❌ 非常に困難                 | ✅ 各リビジョンが小さくレビュー可能     |
| **バグ追跡**   | ❌ 問題箇所の特定が困難       | ✅ 問題のあるリビジョンを即座に特定     |
| **リバート**   | ❌ 全体をリバートする必要     | ✅ 問題部分のみをピンポイントでリバート |
| **並行開発**   | ❌ コンフリクトが発生しやすい | ✅ 機能別でマージが容易                 |
| **デプロイ**   | ❌ 全機能を一括デプロイ       | ✅ 段階的なデプロイが可能               |

### トラブルシューティング

#### リビジョン作成失敗時

- 変更の確認 (`jj diff --summary`)
- 依存関係の解決
- 個別ファイルでの再試行

#### 分割が適切でない場合

- `--max-commits` オプションで調整
- 手動での `edit` モード使用
- より細かい単位での再実行
