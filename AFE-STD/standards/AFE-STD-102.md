# AFE-STD-102 — コーディングスタイルガイドライン

| 項目 | 内容 |
|------|------|
| **文書 ID** | AFE-STD-102 |
| **バージョン** | Ver.1.0-1 |
| **ステータス** | ✅ 確定（初版） |
| **作成日** | 2026-03-02 |
| **最終更新** | 2026-03-03 |
| **担当部門** | Adlaire Framework Engineering |
| **承認者** | (未定) |

---

## 目次

1. [目的・適用範囲](#1-目的適用範囲)
2. [用語定義](#2-用語定義)
3. [一般原則](#3-一般原則)
4. [命名規則](#4-命名規則)
5. [ファイル・ディレクトリ構造](#5-ファイルディレクトリ構造)
6. [書式・インデント](#6-書式インデント)
7. [コメント・ドキュメンテーション](#7-コメントドキュメンテーション)
8. [TypeScript / JavaScript](#8-typescript--javascript)
9. [Python](#9-python)
10. [Go](#10-go)
11. [Rust](#11-rust)
12. [SQL・データベース](#12-sqlデータベース)
13. [シェルスクリプト](#13-シェルスクリプト)
14. [設定ファイル (YAML / TOML / JSON)](#14-設定ファイル-yaml--toml--json)
15. [エラー処理](#15-エラー処理)
16. [テスト](#16-テスト)
17. [セキュリティ配慮](#17-セキュリティ配慮)
18. [ツールチェーン・自動整形](#18-ツールチェーン自動整形)
19. [レビューチェックリスト](#19-レビューチェックリスト)

---

## 1. 目的・適用範囲

### 1.1 目的

本規格は、Adlaire Framework Engineering (AFE) が開発・維持管理するすべてのソフトウェアコードについて、統一されたスタイルとフォーマットを定めることで以下を達成することを目的とする。

- **可読性の向上**: コードを書いた人以外が容易に理解できる状態を維持する。
- **保守性の向上**: 変更・拡張・デバッグのコストを低減する。
- **レビュー効率の向上**: スタイル上の議論を排除し、ロジック・設計に集中できる環境を整える。
- **自動化との親和性**: Linter / Formatter との整合性を保ち、CI/CD パイプラインでの自動チェックを可能にする。

### 1.2 適用範囲

本規格は以下に適用される。

| 対象 | 詳細 |
|------|------|
| プロダクションコード | 本番環境に展開されるすべてのソースコード |
| テストコード | ユニット・統合・E2E テストのすべて |
| インフラコード | IaC (Terraform, Kubernetes マニフェスト等) |
| スクリプト類 | CI/CD パイプライン、自動化スクリプト |
| ドキュメント埋め込みコード | 技術文書内のコードブロック |

### 1.3 適用除外

以下は本規格の適用除外とするが、可能な限り準拠することを推奨する。

- 外部ライブラリ・フレームワーク本体のコード
- 自動生成されたコード (protobuf 出力, ORM マイグレーション等) ※ただし手動編集部分は適用
- レガシーコードのうち、リファクタリング計画が未定のもの

---

## 2. 用語定義

| 用語 | 定義 |
|------|------|
| **SHALL / しなければならない** | 強制要件。違反はコードレビュー不承認事由となる |
| **SHOULD / すべきである** | 推奨事項。合理的な理由があれば逸脱可能 |
| **MAY / してもよい** | 任意。チームの裁量による |
| **Linter** | 静的解析ツール (ESLint, pylint, golangci-lint 等) |
| **Formatter** | 自動整形ツール (Prettier, black, gofmt 等) |
| **Magic number** | コード中に直接記述された意味不明な数値リテラル |
| **Dead code** | 到達・実行されないコード |

---

## 3. 一般原則

### 3.1 明瞭性優先

コードは書くことより読まれることのほうが多い。パフォーマンス上の明確な必要性がない限り、**明瞭性を複雑な最適化より優先する (SHALL)**。

```
// ❌ 悪い例: 何をしているか不明
const r = arr.reduce((a, c) => a + (c.s === 1 ? c.v : 0), 0);

// ✅ 良い例: 意図が明確
const totalActiveValue = arr.reduce((sum, item) => {
  if (item.status === ITEM_STATUS.ACTIVE) {
    return sum + item.value;
  }
  return sum;
}, 0);
```

### 3.2 DRY 原則

同じロジックを複数箇所に書かない (Don't Repeat Yourself)。  
3 回以上同じパターンが出現したら関数・モジュールへ切り出す **(SHOULD)**。

### 3.3 単一責任

関数・クラス・モジュールは単一の責任を持つよう設計する **(SHOULD)**。  
関数の行数目安: **50 行以内 (SHOULD)**、絶対上限: **150 行 (SHALL)**。

### 3.4 早期リターン

ネストを深くしないため、ガード節 (早期 return / throw) を積極的に使う **(SHOULD)**。

```typescript
// ❌ 悪い例
function processUser(user: User | null): string {
  if (user !== null) {
    if (user.isActive) {
      if (user.hasPermission('read')) {
        return user.name;
      }
    }
  }
  return '';
}

// ✅ 良い例
function processUser(user: User | null): string {
  if (user === null) return '';
  if (!user.isActive) return '';
  if (!user.hasPermission('read')) return '';
  return user.name;
}
```

### 3.5 副作用の局所化

副作用 (I/O, 状態変更, ネットワーク通信等) は関数シグネチャや名前から明らかになるよう命名・設計する **(SHOULD)**。

---

## 4. 命名規則

### 4.1 共通ルール

| ルール | 詳細 |
|--------|------|
| 英語で命名 | 日本語識別子は使用不可 **(SHALL)** |
| 略語は最小限 | `usr` より `user`、`btn` より `button` **(SHOULD)** |
| 意味のある名前 | `data`, `tmp`, `val` 等の汎用名は原則禁止 **(SHALL)** |
| 否定形を避ける | `isNotActive` より `isInactive` **(SHOULD)** |
| 接頭辞で型を表さない | ハンガリアン記法禁止 (`strName`, `intCount` 等) **(SHALL)** |

### 4.2 ケース規則

| 対象 | 規則 | 例 |
|------|------|----|
| 変数・関数 | camelCase | `getUserById`, `totalCount` |
| クラス・型・インターフェース | PascalCase | `UserService`, `ApiResponse` |
| 定数 (不変値) | SCREAMING_SNAKE_CASE | `MAX_RETRY_COUNT`, `DEFAULT_TIMEOUT_MS` |
| ファイル名 (TS/JS) | kebab-case | `user-service.ts`, `api-client.ts` |
| ファイル名 (Python) | snake_case | `user_service.py`, `api_client.py` |
| ファイル名 (Go) | snake_case | `user_service.go` |
| 環境変数 | SCREAMING_SNAKE_CASE | `DATABASE_URL`, `APP_PORT` |
| CSS クラス | kebab-case | `btn-primary`, `modal-overlay` |

### 4.3 ブール値の命名

ブール型変数・プロパティは `is`, `has`, `can`, `should`, `was` 等の助動詞で始める **(SHALL)**。

```typescript
// ✅ 良い例
const isLoading: boolean = false;
const hasPermission: boolean = true;
const canDelete: boolean = false;

// ❌ 悪い例
const loading: boolean = false;
const permission: boolean = true;
```

### 4.4 コレクションの命名

配列・リスト等のコレクションは複数形を使用する **(SHALL)**。

```typescript
// ✅
const users: User[] = [];
const fileNames: string[] = [];

// ❌
const userList: User[] = [];   // 冗長
const user: User[] = [];       // 単数形
```

### 4.5 非同期関数の命名

非同期処理を伴う関数には `async` を明示するか、または戻り値型で明示する。  
`fetch`, `load`, `save`, `create`, `update`, `delete`, `get`, `send` 等の動詞は非同期を暗示するため許容 **(MAY)**。

---

## 5. ファイル・ディレクトリ構造

### 5.1 1ファイル1責任

1 つのファイルには原則として 1 つのクラス・コンポーネント・モジュールのみを定義する **(SHOULD)**。  
ただし、密結合した小さなヘルパー型・関数は同一ファイルに含めてもよい **(MAY)**。

### 5.2 ファイルサイズ上限

| レベル | 目安行数 |
|--------|---------|
| 推奨 | 300 行以内 |
| 警告 | 300〜500 行 |
| 必須リファクタリング | 500 行超 (SHALL) |

### 5.3 インポート順序

インポート文は以下の順序でグループ化し、グループ間に空行を入れる **(SHALL)**。

1. 標準ライブラリ / ビルトインモジュール
2. 外部サードパーティライブラリ
3. 内部モジュール (絶対パス)
4. 相対インポート

```typescript
// 1. Node.js 標準
import { readFile } from 'fs/promises';
import path from 'path';

// 2. 外部ライブラリ
import express from 'express';
import { z } from 'zod';

// 3. 内部モジュール (絶対)
import { UserRepository } from '@/repositories/user-repository';
import { Logger } from '@/utils/logger';

// 4. 相対インポート
import { validateInput } from './helpers';
import type { CreateUserDto } from './types';
```

---

## 6. 書式・インデント

### 6.1 インデント

| 言語 | インデント |
|------|-----------|
| TypeScript / JavaScript | スペース 2 文字 |
| Python | スペース 4 文字 (PEP 8) |
| Go | タブ (gofmt 準拠) |
| Rust | スペース 4 文字 (rustfmt 準拠) |
| YAML | スペース 2 文字 |
| SQL | スペース 2 文字 |
| Shell | スペース 2 文字 |

タブ文字とスペースを混在させてはならない **(SHALL)**。

### 6.2 行長

| 言語 | 上限 |
|------|------|
| TypeScript / JavaScript | 100 文字 |
| Python | 88 文字 (black デフォルト) |
| Go | 制限なし (gofmt 準拠) |
| Rust | 100 文字 |
| SQL | 120 文字 |
| Markdown | 制限なし (段落単位) |

### 6.3 末尾空白・改行

- 行末の空白文字は除去する **(SHALL)**
- ファイル末尾は改行 1 つで終わる **(SHALL)**
- Windows 改行コード (CRLF) は使用しない **(SHALL)** — Git の `.gitattributes` で強制する

### 6.4 中括弧スタイル

K&R スタイルを採用する。開き中括弧は同一行に記述する **(SHALL)**。

```typescript
// ✅ K&R (良い)
if (condition) {
  doSomething();
}

// ❌ Allman (禁止)
if (condition)
{
  doSomething();
}
```

### 6.5 セミコロン

TypeScript / JavaScript においてセミコロンは **必須 (SHALL)**。  
Prettier の設定で強制すること。

---

## 7. コメント・ドキュメンテーション

### 7.1 コメントの目的

コメントは「**何をするか**」ではなく「**なぜそうするか**」を説明する **(SHALL)**。

```typescript
// ❌ 悪い例: コードを繰り返しているだけ
// ユーザーを ID で取得する
const user = await getUserById(userId);

// ✅ 良い例: 意図・背景を説明
// キャッシュ期限切れ後でも古いデータを返すことで UX を向上させる
// 参照: ADR-0023 (Stale-While-Revalidate 方針)
const user = await getUserById(userId, { allowStale: true });
```

### 7.2 JSDoc / docstring

公開 API (エクスポートされた関数・クラス・型) には必ず JSDoc / docstring を記述する **(SHALL)**。

```typescript
/**
 * ユーザーを ID で取得する。
 *
 * @param userId - 対象ユーザーの UUID
 * @param options - 取得オプション
 * @param options.allowStale - true の場合キャッシュ期限切れデータを許容する (デフォルト: false)
 * @returns ユーザーオブジェクト。存在しない場合は null
 * @throws {DatabaseError} DB 接続エラー時
 */
async function getUserById(
  userId: string,
  options: { allowStale?: boolean } = {},
): Promise<User | null> {
  // ...
}
```

### 7.3 TODO / FIXME / HACK コメント

| タグ | 用途 |
|------|------|
| `TODO` | 将来実装予定の機能・改善 |
| `FIXME` | 既知のバグ・問題 |
| `HACK` | 暫定的な回避策 |
| `NOTE` | 重要な補足説明 |

必ず担当者名または Issue 番号を付記する **(SHALL)**。

```typescript
// TODO(tanaka): ページネーション実装後に削除 — AFE-ISSUE-234
// FIXME(#567): タイムゾーン変換が夏時間で誤動作する
// HACK: ライブラリバグ回避。v3.2.0 リリース後に削除 — https://github.com/xxx/yyy/issues/123
```

### 7.4 コメントアウトされたコード

コメントアウトされたコードをコミットしてはならない **(SHALL)**。  
不要なコードは削除し、Git 履歴で管理する。

---

## 8. TypeScript / JavaScript

### 8.1 TypeScript 設定

`tsconfig.json` には以下を必須とする **(SHALL)**。

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true
  }
}
```

### 8.2 型定義

- `any` 型の使用は禁止 **(SHALL)**。やむを得ない場合は `unknown` を使用し、型ガードで絞り込む。
- 型アサーション (`as`) は可能な限り避ける **(SHOULD)**。使用する場合はコメントで理由を明記する。
- `@ts-ignore` / `@ts-expect-error` の使用は禁止 **(SHALL)**。ただし、外部ライブラリの型不備に対応する場合に限り `@ts-expect-error` を使用可能とし、コメントで理由を明記する。

```typescript
// ✅ 良い例: unknown + 型ガード
function processResponse(data: unknown): string {
  if (typeof data !== 'object' || data === null) {
    throw new TypeError('Invalid response data');
  }
  if (!('message' in data) || typeof (data as { message: unknown }).message !== 'string') {
    throw new TypeError('Missing message field');
  }
  return (data as { message: string }).message;
}
```

### 8.3 関数・アロー関数

- 純粋関数 (副作用なし) にはアロー関数を優先する **(SHOULD)**
- クラスメソッドには通常の `function` キーワードまたはメソッド構文を使用する **(SHOULD)**
- 引数が 1 つのアロー関数でも括弧を省略しない **(SHALL)**

```typescript
// ✅
const double = (n: number): number => n * 2;
const users = rawData.map((item) => parseUser(item));

// ❌ 括弧省略
const double = n => n * 2;
```

### 8.4 非同期処理

- コールバックベースの非同期は使用しない **(SHALL)**。`Promise` または `async/await` を使用する。
- `Promise.all` / `Promise.allSettled` を活用して並列処理を最適化する **(SHOULD)**
- `async/await` 内の例外は必ず `try/catch` で処理する **(SHALL)**

```typescript
// ✅ 並列処理
const [users, products] = await Promise.all([
  fetchUsers(),
  fetchProducts(),
]);

// ❌ 直列処理 (不必要に遅い)
const users = await fetchUsers();
const products = await fetchProducts();
```

### 8.5 イミュータビリティ

- 変数は可能な限り `const` を使用する **(SHALL)**
- `let` は値の再代入が必要な場合のみ使用する **(SHALL)**
- `var` は使用禁止 **(SHALL)**
- オブジェクト・配列の変更には spread 演算子 / `Array.from` / `structuredClone` を活用する **(SHOULD)**

### 8.6 エラーハンドリング

- `Error` を継承したカスタムエラークラスを使用する **(SHOULD)**
- エラーメッセージにはコンテキスト情報を含める **(SHALL)**

```typescript
class UserNotFoundError extends Error {
  constructor(userId: string) {
    super(`User not found: userId=${userId}`);
    this.name = 'UserNotFoundError';
  }
}
```

### 8.7 禁止事項

| 禁止 | 理由 |
|------|------|
| `eval()` | セキュリティリスク |
| `with` 文 | スコープ汚染 |
| `==` / `!=` | 型強制による予期しない比較 |
| `Object.assign` (ミュータブル操作) | 副作用リスク |
| `console.log` (プロダクションコード) | デバッグ用途限定 |

---

## 9. Python

### 9.1 スタイル準拠

Python コードは **PEP 8** および **PEP 257** に準拠する **(SHALL)**。  
自動整形は `black` を使用し、Lint は `ruff` または `flake8` + `pylint` を使用する **(SHALL)**。

### 9.2 型ヒント

Python 3.9 以降の型ヒント構文を使用する **(SHALL)**。

```python
# ✅ 良い例
def get_user(user_id: str) -> User | None:
    ...

def process_items(items: list[str]) -> dict[str, int]:
    ...

# ❌ 古い構文 (Python 3.8 以前)
from typing import Optional, List, Dict
def get_user(user_id: str) -> Optional[User]:
    ...
```

### 9.3 Docstring

公開関数・クラスには Google スタイルの docstring を記述する **(SHALL)**。

```python
def calculate_discount(price: float, rate: float) -> float:
    """割引後の価格を計算する。

    Args:
        price: 元の価格 (0 以上)
        rate: 割引率 (0.0〜1.0)

    Returns:
        割引後の価格

    Raises:
        ValueError: price が負の値、または rate が範囲外の場合
    """
    if price < 0:
        raise ValueError(f"price must be non-negative, got {price}")
    if not 0.0 <= rate <= 1.0:
        raise ValueError(f"rate must be in [0.0, 1.0], got {rate}")
    return price * (1.0 - rate)
```

### 9.4 クラス定義

- データクラスには `@dataclass` または Pydantic `BaseModel` を使用する **(SHOULD)**
- `__init__` のみで属性を定義する素のクラスは避ける **(SHOULD)**

```python
# ✅ Pydantic (バリデーション付き)
from pydantic import BaseModel, Field

class CreateUserRequest(BaseModel):
    name: str = Field(..., min_length=1, max_length=100)
    email: str = Field(..., pattern=r'^[^@]+@[^@]+\.[^@]+$')
    age: int = Field(..., ge=0, le=150)
```

### 9.5 例外処理

- 例外は具体的な型でキャッチする。`except Exception:` の裸のキャッチは禁止 **(SHALL)**
- 例外を再発生させる場合は `raise ... from e` を使用する **(SHALL)**

```python
try:
    result = await db.find_user(user_id)
except DatabaseConnectionError as e:
    raise UserServiceError(f"Failed to fetch user: {user_id}") from e
```

---

## 10. Go

### 10.1 フォーマット

`gofmt` / `goimports` による自動整形を必須とする **(SHALL)**。  
Lint は `golangci-lint` を使用し、設定ファイル `.golangci.yml` をリポジトリに含める **(SHALL)**。

### 10.2 エラーハンドリング

- エラーは必ず呼び出し元で処理する。`_` での無視は禁止 **(SHALL)**
- エラーには `fmt.Errorf` + `%w` でコンテキストを付加する **(SHALL)**

```go
// ✅
user, err := repo.FindByID(ctx, userID)
if err != nil {
    return nil, fmt.Errorf("finding user %s: %w", userID, err)
}

// ❌ エラー無視
user, _ := repo.FindByID(ctx, userID)
```

### 10.3 パッケージ命名

- パッケージ名は小文字・単語1つ **(SHALL)**
- `util`, `common`, `helper` 等の汎用名は禁止 **(SHALL)**

### 10.4 コメント

`godoc` に準拠し、エクスポートされた識別子にはすべてコメントを付与する **(SHALL)**。

---

## 11. Rust

### 11.1 フォーマット・Lint

`rustfmt` による自動整形および `clippy` による Lint を必須とする **(SHALL)**。  
`clippy::pedantic` を有効にし、警告をエラーとして扱う **(SHOULD)**。

```toml
# Cargo.toml
[workspace.lints.clippy]
pedantic = "warn"
```

### 11.2 エラー処理

- `unwrap()` / `expect()` はテストコードのみ許容 **(SHALL)**
- プロダクションコードでは `?` 演算子または `match` を使用する **(SHALL)**
- カスタムエラー型には `thiserror` クレートを使用する **(SHOULD)**

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum UserError {
    #[error("user not found: id={0}")]
    NotFound(String),
    #[error("database error: {0}")]
    Database(#[from] sqlx::Error),
}
```

### 11.3 ライフタイム注釈

ライフタイム注釈が必要な場合は `'a`, `'b` 等の短縮名でなく、意味のある名前を使用する **(SHOULD)**。

---

## 12. SQL・データベース

### 12.1 命名規則

| 対象 | 規則 | 例 |
|------|------|----|
| テーブル名 | snake_case・複数形 | `users`, `order_items` |
| カラム名 | snake_case | `created_at`, `user_id` |
| インデックス名 | `idx_{table}_{column}` | `idx_users_email` |
| 外部キー名 | `fk_{table}_{ref_table}` | `fk_orders_users` |

### 12.2 クエリスタイル

- SQL キーワードは大文字 **(SHALL)**
- 1 クエリが長い場合は適切に改行・インデントする **(SHALL)**

```sql
-- ✅
SELECT
  u.id,
  u.name,
  u.email,
  COUNT(o.id) AS order_count
FROM users AS u
LEFT JOIN orders AS o ON o.user_id = u.id
WHERE u.is_active = TRUE
  AND u.created_at >= '2026-01-01'
GROUP BY u.id, u.name, u.email
ORDER BY order_count DESC
LIMIT 100;
```

### 12.3 パラメータバインディング

SQL クエリへの値の埋め込みには必ずパラメータバインディングを使用する **(SHALL)**。  
文字列結合による SQL 構築は **絶対禁止 (SHALL)**。

```typescript
// ✅ パラメータバインディング
const result = await db.query(
  'SELECT * FROM users WHERE email = $1',
  [email],
);

// ❌ 文字列結合 (SQL インジェクション脆弱性)
const result = await db.query(`SELECT * FROM users WHERE email = '${email}'`);
```

---

## 13. シェルスクリプト

### 13.1 Shebang・オプション

すべての Shell スクリプトは以下を先頭に記述する **(SHALL)**。

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'
```

| オプション | 効果 |
|-----------|------|
| `-e` | エラー時即時終了 |
| `-u` | 未定義変数参照時エラー |
| `-o pipefail` | パイプ内のエラーを伝播 |

### 13.2 変数

- ローカル変数は `local` で宣言する **(SHALL)**
- 変数展開は常にクォートで囲む **(SHALL)**

```bash
# ✅
local user_name="${1}"
echo "Hello, ${user_name}"

# ❌
user_name=$1
echo "Hello, $user_name"
```

### 13.3 Shellcheck

`shellcheck` による静的解析を CI/CD に組み込む **(SHALL)**。

---

## 14. 設定ファイル (YAML / TOML / JSON)

### 14.1 YAML

- インデントはスペース 2 文字 **(SHALL)**
- タブ使用禁止 **(SHALL)**
- シークレット値は直接記述禁止 **(SHALL)** — 環境変数参照または Secret Manager を使用する
- Anchor (`&`) / Alias (`*`) は使用を最小限に留める **(SHOULD)**

### 14.2 JSON

- 手動編集する JSON 設定ファイルはコメント不可のため、複雑な設定は YAML / TOML へ移行を検討する **(SHOULD)**
- JSON スキーマ (`$schema`) を明示する **(SHOULD)**

### 14.3 環境変数

- 環境変数名は `SCREAMING_SNAKE_CASE` **(SHALL)**
- `.env` ファイルはリポジトリにコミットしない **(SHALL)** — `.env.example` のみコミット
- 必須環境変数は起動時にバリデーションする **(SHALL)**

---

## 15. エラー処理

### 15.1 エラーの伝播

エラーは適切なレイヤーまで伝播させ、末端の呼び出し元で一括処理する設計とする **(SHOULD)**。

```
Repository Layer → Service Layer → Controller Layer → Error Handler Middleware
```

### 15.2 エラーメッセージの国際化

エラーメッセージは英語で記述し、UI 表示用メッセージは i18n 化する **(SHOULD)**。  
内部ログには技術的詳細を含め、ユーザー向けメッセージには含めない **(SHALL)**。

### 15.3 エラーコード

API エラーには一意なエラーコードを付与する **(SHALL)**。

```json
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "The requested user does not exist.",
    "details": {
      "userId": "usr_abc123"
    }
  }
}
```

---

## 16. テスト

### 16.1 テスト命名

テスト関数・テストケースの命名は `[対象]_[条件]_[期待結果]` 形式とする **(SHOULD)**。

```typescript
// ✅
describe('getUserById', () => {
  it('valid_userId_returnsUser', async () => { ... });
  it('nonexistent_userId_returnsNull', async () => { ... });
  it('invalid_userId_format_throwsValidationError', async () => { ... });
});
```

### 16.2 テストカバレッジ

| テスト種別 | 最低カバレッジ |
|-----------|--------------|
| ユニットテスト | 80% |
| 統合テスト | 主要ユースケース 100% |

### 16.3 テストデータ

- テストデータにプロダクションデータを使用しない **(SHALL)**
- ファクトリ関数 / Fixture でテストデータを生成する **(SHOULD)**
- マジックナンバーではなく定数または説明的な変数名でテストデータを定義する **(SHALL)**

### 16.4 モック

- 外部 I/O (DB, API, ファイルシステム) はモック化する **(SHALL)**
- モックは過剰に設定せず、テスト対象の振る舞いに最小限のモックを使用する **(SHOULD)**

---

## 17. セキュリティ配慮

### 17.1 シークレット管理

- API キー・パスワード・証明書等をコードにハードコードしない **(SHALL)**
- シークレットは環境変数または Secret Manager (AWS Secrets Manager, HashiCorp Vault 等) から取得する **(SHALL)**
- `git-secrets` または `detect-secrets` を pre-commit フックに設定する **(SHALL)**

### 17.2 依存関係

- 依存ライブラリのバージョンを固定する (`package-lock.json`, `poetry.lock` 等) **(SHALL)**
- 既知の脆弱性を含むライブラリを使用しない **(SHALL)** — Dependabot / Renovate で自動更新
- 依存ライブラリのライセンスを確認する **(SHOULD)**

### 17.3 入力バリデーション

- 外部入力はすべて信頼せず、受け取り時点でバリデーションする **(SHALL)**
- バリデーションライブラリ (Zod, Pydantic 等) を使用する **(SHOULD)**

---

## 18. ツールチェーン・自動整形

### 18.1 必須ツール

| 言語 | Formatter | Linter |
|------|-----------|--------|
| TypeScript / JS | Prettier | ESLint |
| Python | black | ruff / pylint |
| Go | gofmt / goimports | golangci-lint |
| Rust | rustfmt | clippy |
| Shell | — | shellcheck |
| YAML | prettier | yamllint |
| SQL | — | sqlfluff |

### 18.2 Pre-commit フック

`.pre-commit-config.yaml` を使用し、コミット前に以下を自動実行する **(SHALL)**。

1. Formatter による自動整形
2. Linter によるエラーチェック
3. シークレットスキャン
4. 単体テスト (変更ファイル関連のみ)

### 18.3 CI/CD 統合

CI パイプラインに以下のチェックを組み込む **(SHALL)**。

1. Lint チェック (エラーでビルド失敗)
2. テスト実行・カバレッジレポート
3. セキュリティスキャン (SAST, 依存関係脆弱性)
4. ビルド成功確認

---

## 19. レビューチェックリスト

プルリクエストのレビュー時に以下を確認する。

### 19.1 自己レビュー (作成者)

- [ ] 命名規則に従っているか
- [ ] コメントは「なぜ」を説明しているか
- [ ] Dead code / コメントアウトコードがないか
- [ ] Magic number を定数に置き換えているか
- [ ] エラーハンドリングが適切か
- [ ] テストが追加・更新されているか
- [ ] シークレットがコードに含まれていないか
- [ ] Lint / Formatter が通っているか

### 19.2 レビュアーチェック

- [ ] ロジックの正確性
- [ ] エッジケース・境界値の考慮
- [ ] パフォーマンスへの影響
- [ ] セキュリティへの影響
- [ ] 下位互換性の維持
- [ ] ドキュメントの更新


*本文書は AFE-STD シリーズの一部です。関連規格: AFE-STD-101 (バージョン管理), AFE-STD-103 (コードレビュープロセス)*
