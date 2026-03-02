# AFE-STD-600: ORM・データベース抽象化標準

| 項目 | 内容 |
|------|------|
| **ドキュメントID** | AFE-STD-600 |
| **タイトル** | ORM・データベース抽象化標準 |
| **バージョン** | Ver.1.0.0 |
| **作成日** | 2026-03-02 |
| **最終更新日** | 2026-03-02 |
| **ステータス** | ✅ 初版発行 |
| **PSR対応** | 独自仕様（旧 AFE-STD-001 後継） |
| **関連標準** | AFE-STD-100, AFE-STD-300, AFE-STD-402, AFE-STD-500 |

---

## 目次

1. [概要](#1-概要)
2. [スコープ](#2-スコープ)
3. [外部標準との互換性](#3-外部標準との互換性)
4. [Adlaire 固有の拡張](#4-adlaire-固有の拡張)
5. [リポジトリインターフェース定義](#5-リポジトリインターフェース定義)
6. [クエリビルダー規則](#6-クエリビルダー規則)
7. [エンティティ・集約規則](#7-エンティティ集約規則)
8. [トランザクション規則](#8-トランザクション規則)
9. [マイグレーション規則](#9-マイグレーション規則)
10. [実装コード例](#10-実装コード例)
11. [違反パターン](#11-違反パターン)

---

## 1. 概要

AFE-STD-600 は AFE における **ORM（Object-Relational Mapping）およびデータベース抽象化層の標準仕様**を定義する。PSR には対応する規格がないため、AFE 独自の仕様（旧 AFE-STD-001 の後継）として策定する。

### 設計方針

- **Repository パターン必須**: すべての DB アクセスは RepositoryInterface 経由
- **Aggregate Root 中心設計**: DDD の集約境界に沿ったデータアクセス
- **クエリの型安全化**: 生 SQL 文字列の直接使用を禁止し、クエリビルダーを必須化
- **トランザクション境界の明確化**: UnitOfWork パターンによるトランザクション管理
- **読み書き分離**: ReadRepositoryInterface / WriteRepositoryInterface の分離

---

## 2. スコープ

### 2.1 適用範囲

- AFE フレームワーク本体のデータベース層
- AFE ベースプロジェクトのすべての DB アクセスコード
- ORM エンティティとドメインエンティティのマッピング

### 2.2 適用外

- 生 SQL が必要な極少数のケース（ `RawQueryInterface` を経由した場合のみ例外許可、要 GMレビュー）
- マイグレーションファイル内の DDL

---

## 3. 外部標準との互換性

| 項目 | 外部標準 | AFE-STD-600 | 互換性 |
|------|---------|-------------|--------|
| Eloquent ORM | Laravel 独自 | ❌ **廃止**（ActiveRecord パターン禁止） | 非互換 |
| Doctrine ORM | Doctrine 独自 | △ **DataMapper パターンの概念は採用** | 参考 |
| PDO | PHP 標準 | ✅ **低レベル実装で使用可** | 内部使用 |
| DBAL | Doctrine DBAL | △ **クエリビルダー参考** | 参考 |

### 3.1 ActiveRecord パターン禁止の理由

```
ActiveRecord（Eloquent等）の問題点:
  1. ドメインモデルと永続化層が混在する
  2. エンティティが DB スキーマに依存してしまう
  3. テストが DB に依存する
  4. DDD の Aggregate Root 設計が困難

AFE-STD-600 の採用パターン:
  DataMapper + Repository パターン
  → ドメインエンティティと永続化を完全分離
```

---

## 4. Adlaire 固有の拡張

### 4.1 読み書き分離

```
ReadRepositoryInterface:  SELECT 系クエリのみ
WriteRepositoryInterface: INSERT/UPDATE/DELETE のみ
RepositoryInterface:      両方を継承（デフォルト）
```

### 4.2 仕様パターン（Specification Pattern）

```php
// クエリの条件をオブジェクトとして表現
$spec = new ActiveUserSpecification()
    ->and(new UserAgeRangeSpecification(min: 18, max: 65))
    ->or(new AdminUserSpecification());

$users = $repository->matching($spec);
```

### 4.3 カーソルページネーション標準化

```php
// Offset ページネーション禁止（パフォーマンス問題）
// カーソルベースページネーション必須（大量データ対応）
$page = $repository->cursorPaginate(cursor: $lastId, limit: 20);
```

---

## 5. リポジトリインターフェース定義

### 5.1 読み取りリポジトリ

```php
<?php

declare(strict_types=1);

namespace Adlaire\Database\Contracts;

/**
 * AFE-STD-600 準拠 読み取りリポジトリインターフェース
 *
 * @template TEntity of EntityInterface
 * @template TId of EntityIdInterface
 */
interface ReadRepositoryInterface
{
    /**
     * ID による単一エンティティ取得
     *
     * @param TId $id
     * @return TEntity
     * @throws EntityNotFoundException
     */
    public function findById(EntityIdInterface $id): EntityInterface;

    /**
     * ID による単一エンティティ取得（存在しない場合 null）
     *
     * @param TId $id
     * @return TEntity|null
     */
    public function findByIdOrNull(EntityIdInterface $id): ?EntityInterface;

    /**
     * 仕様に合致するエンティティ取得
     *
     * @param SpecificationInterface $spec
     * @return array<TEntity>
     */
    public function matching(SpecificationInterface $spec): array;

    /**
     * カーソルページネーション
     *
     * @param TId|null $cursor
     * @return CursorPaginatorInterface<TEntity>
     */
    public function cursorPaginate(?EntityIdInterface $cursor = null, int $limit = 20): CursorPaginatorInterface;

    /**
     * 件数カウント
     */
    public function count(SpecificationInterface $spec): int;

    /**
     * 存在確認
     */
    public function exists(SpecificationInterface $spec): bool;
}
```

### 5.2 書き込みリポジトリ

```php
<?php

declare(strict_types=1);

namespace Adlaire\Database\Contracts;

/**
 * AFE-STD-600 準拠 書き込みリポジトリインターフェース
 *
 * @template TEntity of EntityInterface
 * @template TId of EntityIdInterface
 */
interface WriteRepositoryInterface
{
    /**
     * エンティティ保存（INSERT or UPDATE）
     *
     * @param TEntity $entity
     * @return TEntity
     */
    public function save(EntityInterface $entity): EntityInterface;

    /**
     * エンティティ削除（論理削除・物理削除はドメイン設計に依存）
     *
     * @param TId $id
     */
    public function delete(EntityIdInterface $id): void;

    /**
     * 複数エンティティの一括保存
     *
     * @param array<TEntity> $entities
     */
    public function saveAll(array $entities): void;
}
```

### 5.3 エンティティインターフェース

```php
<?php

declare(strict_types=1);

namespace Adlaire\Database\Contracts;

use Adlaire\Event\Contracts\DomainEventInterface;

interface EntityInterface
{
    public function getId(): EntityIdInterface;

    /**
     * ドメインイベントを記録（AFE-STD-401 EventRecorderInterface 統合）
     *
     * @return array<DomainEventInterface>
     */
    public function releaseEvents(): array;
}
```

### 5.4 仕様インターフェース（Specification Pattern）

```php
<?php

declare(strict_types=1);

namespace Adlaire\Database\Contracts;

interface SpecificationInterface
{
    public function isSatisfiedBy(EntityInterface $entity): bool;
    public function and(SpecificationInterface $other): static;
    public function or(SpecificationInterface $other): static;
    public function not(): static;

    /**
     * クエリビルダーへの変換（永続化層での最適化）
     */
    public function toQueryConstraint(): QueryConstraintInterface;
}
```

---

## 6. クエリビルダー規則

```
RULE-600-01: 生 SQL 文字列の直接使用を禁止する（RawQueryInterface のみ例外、要レビュー）
RULE-600-02: N+1 クエリを禁止する（Eager Loading を必須とする）
RULE-600-03: SELECT * を禁止する（必要カラムを明示すること）
RULE-600-04: WHERE 句の条件は SpecificationInterface を使用すること
RULE-600-05: OFFSET ページネーション禁止（cursorPaginate() を使用すること）
RULE-600-06: 外部キー制約が必要な場合は必ずスキーマに定義すること（アプリケーション層での代替禁止）
```

---

## 7. エンティティ・集約規則

```
RULE-600-07: エンティティの識別子は必ず EntityIdInterface を実装したバリューオブジェクトとすること
RULE-600-08: 集約ルート（Aggregate Root）のみリポジトリを持つこと（子エンティティに直接リポジトリを作成しない）
RULE-600-09: エンティティの生成は static ファクトリーメソッド（create()）を通じて行うこと
RULE-600-10: エンティティプロパティのセッターを public に公開することを禁止する（ドメインメソッドで変更）
```

---

## 8. トランザクション規則

```php
interface TransactionManagerInterface
{
    /**
     * トランザクション実行
     *
     * @template T
     * @param callable(UnitOfWorkInterface): T $callback
     * @return T
     */
    public function run(callable $callback): mixed;

    /**
     * 明示的なコミット（非推奨：run() を使用すること）
     */
    public function commit(): void;

    /**
     * ロールバック（run() では例外時に自動実行）
     */
    public function rollback(): void;
}
```

```
RULE-600-11: DB トランザクションは必ず TransactionManagerInterface::run() を使用すること
RULE-600-12: コントローラー・サービス層でのトランザクション開始を禁止する（UseCaseクラス内のみ許可）
```

---

## 9. マイグレーション規則

```
RULE-600-13: マイグレーションファイルのクラス名は Migration_{YYYYMMDDHHMMSS}_{説明} 形式とすること
RULE-600-14: down() メソッドを必ず実装すること（ロールバック可能性の確保）
RULE-600-15: 本番マイグレーション適用前にステージング環境での検証を必須とする
RULE-600-16: NOT NULL 制約はデフォルト値とともに設定すること（既存データへの影響を最小化）
```

---

## 10. 実装コード例

```php
<?php

declare(strict_types=1);

namespace Adlaire\User\Infrastructure;

use Adlaire\Database\Contracts\ReadRepositoryInterface;
use Adlaire\Database\Contracts\WriteRepositoryInterface;
use Adlaire\Database\Contracts\SpecificationInterface;
use Adlaire\User\Domain\Entities\User;
use Adlaire\User\Domain\ValueObjects\UserId;
use Adlaire\User\Domain\Exceptions\UserNotFoundException;

final class EloquentUserRepository implements ReadRepositoryInterface, WriteRepositoryInterface
{
    public function __construct(
        private readonly UserMapper $mapper,
        private readonly UserQueryBuilder $queryBuilder,
    ) {}

    public function findById(EntityIdInterface $id): User
    {
        $record = $this->queryBuilder->findById($id->value());

        if ($record === null) {
            throw new UserNotFoundException("User not found: {$id->value()}");
        }

        return $this->mapper->toDomain($record);
    }

    public function save(EntityInterface $entity): User
    {
        assert($entity instanceof User);

        $record = $this->mapper->toRecord($entity);
        $saved  = $this->queryBuilder->upsert($record);

        return $this->mapper->toDomain($saved);
    }

    public function matching(SpecificationInterface $spec): array
    {
        $constraints = $spec->toQueryConstraint();
        $records     = $this->queryBuilder->applyConstraints($constraints)->get();

        return array_map(fn($r) => $this->mapper->toDomain($r), $records);
    }
}
```

---

## 11. 違反パターン

```php
// ❌ VIOLATION-600-01: ActiveRecord パターン（Eloquent直接使用）
User::where('email', $email)->first();

// ❌ VIOLATION-600-02: 生 SQL
DB::select('SELECT * FROM users WHERE email = ?', [$email]);

// ❌ VIOLATION-600-03: コントローラーでの DB 直接アクセス
class UserController {
    public function show(int $id): ResponseInterface {
        $user = User::find($id); // 禁止
    }
}

// ❌ VIOLATION-600-04: OFFSET ページネーション
$users = $repository->paginate(page: 100, perPage: 20); // 禁止 → cursorPaginate() を使用

// ❌ VIOLATION-600-05: SELECT *
$this->queryBuilder->select('*')->get(); // 禁止
```

---

## 変更履歴

本ファイルの変更履歴は `../VERSION_HISTORY.md` の `AFE-STD-600` セクションを参照すること。

---

*© 2026 Adlaire Group. All Rights Reserved.*
