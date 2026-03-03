# AFE-STD-501: シンプルキャッシュインターフェース標準

| 項目 | 内容 |
|------|------|
| **ドキュメントID** | AFE-STD-501 |
| **タイトル** | シンプルキャッシュインターフェース標準 |
| **バージョン** | Ver.1.0-1 |
| **作成日** | 2026-03-02 |
| **最終更新日** | 2026-03-03 |
| **ステータス** | ✅ 確定（初版） |
| **PSR対応** | PSR-16（Simple Cache）代替 |
| **関連標準** | AFE-STD-500, AFE-STD-402 |

---

## 1. 概要

AFE-STD-501 は AFE における **シンプルなキャッシュ操作のインターフェース標準**を定義する。PSR-16 を代替し、AFE-STD-500 との整合性を保ちながら、軽量なキャッシュ操作 API を提供する。PSR-16 の `SimpleCacheInterface` を置き換え、型安全な操作・TTL の厳格化・ClockInterface 連携を実現する。

---

## 2. PSR-16 との互換性・差異

| 項目 | PSR-16 | AFE-STD-501 | 互換性 |
|------|--------|-------------|--------|
| `get(key, default)` | ✅ 定義 | ✅ **型安全化** | △ 変更 |
| `set(key, value, ttl)` | ✅ 定義 | ✅ **維持** | ✅ 互換 |
| `delete(key)` | ✅ 定義 | ✅ **維持** | ✅ 互換 |
| `clear()` | ✅ 定義 | ✅ **維持** | ✅ 互換 |
| `getMultiple()` | ✅ 定義 | ✅ **型安全化** | △ 変更 |
| `setMultiple()` | ✅ 定義 | ✅ **維持** | ✅ 互換 |
| `deleteMultiple()` | ✅ 定義 | ✅ **維持** | ✅ 互換 |
| `has(key)` | ✅ 定義 | ✅ **維持** | ✅ 互換 |
| 型安全 getter | 未定義 | ✅ **getString/Int/Bool** | AFE拡張 |
| Remember パターン | 未定義 | ✅ **remember()** | AFE拡張 |
| ClockInterface 統合 | 未定義 | ✅ **TTL 計算に ClockInterface** | AFE拡張 |

---

## 3. インターフェース定義

```php
<?php

declare(strict_types=1);

namespace Adlaire\Cache\Contracts;

use DateInterval;

/**
 * AFE-STD-501 準拠 シンプルキャッシュインターフェース
 * PSR-16 SimpleCacheInterface の代替仕様（拡張）
 */
interface SimpleCacheInterface
{
    /**
     * キャッシュ値取得（PSR-16 互換）
     *
     * @param mixed $default キャッシュミス時のデフォルト値
     */
    public function get(string $key, mixed $default = null): mixed;

    /**
     * キャッシュ値保存（PSR-16 互換）
     *
     * @param int|DateInterval|null $ttl null = ドライバーデフォルト
     */
    public function set(string $key, mixed $value, int|DateInterval|null $ttl = null): bool;

    public function delete(string $key): bool;
    public function clear(): bool;

    /**
     * 複数値取得（PSR-16 互換）
     *
     * @param iterable<string> $keys
     * @param mixed $default
     * @return iterable<string, mixed>
     */
    public function getMultiple(iterable $keys, mixed $default = null): iterable;

    /**
     * 複数値保存（PSR-16 互換）
     *
     * @param iterable<string, mixed> $values
     */
    public function setMultiple(iterable $values, int|DateInterval|null $ttl = null): bool;

    /**
     * 複数値削除（PSR-16 互換）
     *
     * @param iterable<string> $keys
     */
    public function deleteMultiple(iterable $keys): bool;

    public function has(string $key): bool;

    // --- AFE 固有拡張 ---

    /**
     * 型安全な値取得
     */
    public function getString(string $key, string $default = ''): string;
    public function getInt(string $key, int $default = 0): int;
    public function getFloat(string $key, float $default = 0.0): float;
    public function getBool(string $key, bool $default = false): bool;

    /**
     * Remember パターン
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
     * アトミックなインクリメント（AFE固有）
     */
    public function increment(string $key, int $step = 1): int;

    /**
     * アトミックなデクリメント（AFE固有）
     */
    public function decrement(string $key, int $step = 1): int;
}
```

---

## 4. AFE-STD-500 との使い分け指針

| 用途 | 推奨インターフェース |
|------|-------------------|
| 単純な GET/SET/DELETE 操作 | `SimpleCacheInterface`（AFE-STD-501） |
| タグ付きキャッシュ・一括操作 | `CachePoolInterface`（AFE-STD-500） |
| キャッシュドライバーの切り替え | `CachePoolInterface` |
| 軽量コンポーネント・ライブラリ内部 | `SimpleCacheInterface` |

---

## 5. 実装規則

```
RULE-501-01: PSR-16 SimpleCacheInterface との混用禁止（AFE-STD-501 のみ使用）
RULE-501-02: get() の戻り値が mixed の場合は型安全メソッド（getString等）を優先使用すること
RULE-501-03: TTL は必ず明示的に指定すること（null をデフォルトとして依存することを禁止）
RULE-501-04: increment/decrement はアトミック操作を保証するドライバー（Redis等）使用時のみ利用すること
```

---

## 変更履歴

本ファイルの変更履歴は `../VERSION_HISTORY.md` の `AFE-STD-501` セクションを参照すること。

---

*© 2026 Adlaire Group. All Rights Reserved.*
