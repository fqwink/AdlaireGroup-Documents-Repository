# AFE-STD-500: キャッシュインターフェース標準

| 項目 | 内容 |
|------|------|
| **ドキュメントID** | AFE-STD-500 |
| **タイトル** | キャッシュインターフェース標準 |
| **バージョン** | Ver.1.0.0 |
| **作成日** | 2026-03-02 |
| **最終更新日** | 2026-03-02 |
| **ステータス** | ✅ 初版発行 |
| **PSR対応** | PSR-6（Caching Interface）代替 |
| **関連標準** | AFE-STD-100, AFE-STD-300, AFE-STD-402, AFE-STD-501 |

---

## 1. 概要

AFE-STD-500 は AFE における **キャッシュプールとキャッシュアイテムの標準インターフェース仕様**を定義する。PSR-6 を代替し、型安全なキャッシュ操作・タグベースの無効化・統計情報・デコレーターパターンを提供する。

---

## 2. PSR-6 との互換性・差異

| 項目 | PSR-6 | AFE-STD-500 | 互換性 |
|------|-------|-------------|--------|
| `CacheItemPoolInterface` | ✅ 定義 | ✅ **拡張** | △ 拡張 |
| `CacheItemInterface` | ✅ 定義 | ✅ **型安全化** | △ 変更 |
| `getItem()` 戻り値 | `CacheItemInterface` | ✅ **同一** | ✅ 互換 |
| `set()` 戻り値 | `bool` | ✅ **同一** | ✅ 互換 |
| TTL の型 | `int|\DateInterval|null` | ✅ **同一** | ✅ 互換 |
| タグ付きキャッシュ | 未定義 | ✅ **TaggableCacheInterface** | AFE拡張 |
| 型安全 getter | 未定義 | ✅ **getString/Int/Float/Bool** | AFE拡張 |
| キャッシュ統計 | 未定義 | ✅ **CacheStatsInterface** | AFE拡張 |
| Remember パターン | 未定義 | ✅ **remember() メソッド** | AFE拡張 |
| ClockInterface 統合 | 未定義 | ✅ **TTL 計算に ClockInterface 使用** | AFE拡張 |

---

## 3. インターフェース定義

### 3.1 キャッシュプールインターフェース

```php
<?php

declare(strict_types=1);

namespace Adlaire\Cache\Contracts;

use DateInterval;
use DateTimeImmutable;

/**
 * AFE-STD-500 準拠 キャッシュプールインターフェース
 * PSR-6 CacheItemPoolInterface の代替仕様（拡張）
 */
interface CachePoolInterface
{
    /**
     * キャッシュアイテム取得（PSR-6 互換）
     */
    public function getItem(string $key): CacheItemInterface;

    /**
     * 複数キャッシュアイテム取得
     *
     * @param array<string> $keys
     * @return array<string, CacheItemInterface>
     */
    public function getItems(array $keys): array;

    /**
     * キャッシュ存在確認
     */
    public function hasItem(string $key): bool;

    /**
     * キャッシュ全クリア
     */
    public function clear(): bool;

    /**
     * キャッシュアイテム削除
     */
    public function deleteItem(string $key): bool;

    /**
     * 複数キャッシュアイテム削除
     *
     * @param array<string> $keys
     */
    public function deleteItems(array $keys): bool;

    /**
     * キャッシュアイテム保存
     */
    public function save(CacheItemInterface $item): bool;

    /**
     * キャッシュアイテムの遅延保存
     */
    public function saveDeferred(CacheItemInterface $item): bool;

    /**
     * 遅延保存のコミット
     */
    public function commit(): bool;

    // --- AFE 固有拡張 ---

    /**
     * Remember パターン: キャッシュがあれば返し、なければコールバックで生成して保存
     *
     * @template T
     * @param callable(): T $callback
     * @return T
     */
    public function remember(
        string $key,
        int|DateInterval|null $ttl,
        callable $callback
    ): mixed;

    /**
     * 型安全な値取得（キャッシュミスの場合はデフォルト値を返す）
     */
    public function getString(string $key, string $default = ''): string;
    public function getInt(string $key, int $default = 0): int;
    public function getFloat(string $key, float $default = 0.0): float;
    public function getBool(string $key, bool $default = false): bool;

    /**
     * キャッシュ統計情報取得
     */
    public function stats(): CacheStatsInterface;
}
```

### 3.2 キャッシュアイテムインターフェース

```php
<?php

declare(strict_types=1);

namespace Adlaire\Cache\Contracts;

use DateInterval;
use DateTimeImmutable;

/**
 * AFE-STD-500 準拠 キャッシュアイテムインターフェース
 * PSR-6 CacheItemInterface の代替仕様（型安全化）
 */
interface CacheItemInterface
{
    public function getKey(): string;

    /**
     * キャッシュ値取得（PSR-6 互換）
     */
    public function get(): mixed;

    public function isHit(): bool;

    /**
     * 型安全な値取得（AFE固有）
     */
    public function getString(): string;
    public function getInt(): int;
    public function getFloat(): float;
    public function getBool(): bool;

    /**
     * @template T of object
     * @param class-string<T> $class
     * @return T
     */
    public function getObject(string $class): object;

    public function set(mixed $value): static;

    public function expiresAt(?DateTimeImmutable $expiration): static;
    public function expiresAfter(int|DateInterval|null $time): static;

    /**
     * タグ付け（AFE固有）
     */
    public function withTags(string ...$tags): static;

    /**
     * 有効期限取得（AFE固有）
     */
    public function getExpiry(): ?DateTimeImmutable;
}
```

### 3.3 タグ付きキャッシュインターフェース（AFE固有）

```php
<?php

declare(strict_types=1);

namespace Adlaire\Cache\Contracts;

/**
 * タグベースのキャッシュ無効化インターフェース
 */
interface TaggableCacheInterface extends CachePoolInterface
{
    /**
     * 指定タグを持つキャッシュをすべて無効化
     *
     * @param array<string> $tags
     */
    public function invalidateTags(array $tags): bool;

    /**
     * 指定タグを持つキャッシュをすべて取得
     *
     * @param string $tag
     * @return array<CacheItemInterface>
     */
    public function getItemsByTag(string $tag): array;
}
```

### 3.4 キャッシュ統計インターフェース（AFE固有）

```php
<?php

declare(strict_types=1);

namespace Adlaire\Cache\Contracts;

interface CacheStatsInterface
{
    public function hits(): int;
    public function misses(): int;
    public function hitRatio(): float;
    public function totalItems(): int;
    public function memoryUsageBytes(): int;
}
```

---

## 4. 実装規則

```
RULE-500-01: キャッシュキーには英数字・アンダースコア・ハイフンのみ使用（PSR-6 準拠）
RULE-500-02: TTL 未指定（null）はキャッシュドライバーのデフォルト TTL を使用すること
RULE-500-03: キャッシュに個人情報・認証トークン等の機密情報を保存することを禁止する
RULE-500-04: remember() パターンを活用し、手動の get/set パターンを最小限にすること
RULE-500-05: タグ付きキャッシュを使用する場合は TaggableCacheInterface を使用すること
RULE-500-06: ユーザー固有のキャッシュキーには必ずユーザー ID をプレフィックスとして付与すること
```

---

## 5. 実装コード例

```php
<?php

declare(strict_types=1);

namespace Adlaire\User\Services;

use Adlaire\Cache\Contracts\TaggableCacheInterface;
use Adlaire\User\Contracts\UserServiceInterface;

final class CachedUserService implements UserServiceInterface
{
    private const CACHE_TTL = 3600; // 1 hour

    public function __construct(
        private readonly UserServiceInterface $inner,
        private readonly TaggableCacheInterface $cache,
    ) {}

    public function findById(int $userId): User
    {
        return $this->cache->remember(
            key: "user.{$userId}",
            ttl: self::CACHE_TTL,
            callback: fn() => $this->inner->findById($userId),
        );
    }

    public function update(UpdateUserCommand $command): User
    {
        $user = $this->inner->update($command);
        // ユーザー更新時にキャッシュを無効化
        $this->cache->invalidateTags(["user.{$command->userId}"]);
        return $user;
    }
}
```

---

## 変更履歴

本ファイルの変更履歴は `../VERSION_HISTORY.md` の `AFE-STD-500` セクションを参照すること。

---

*© 2026 Adlaire Group. All Rights Reserved.*
