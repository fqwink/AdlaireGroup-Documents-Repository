# AFE-STD-402: クロックインターフェース標準

| 項目 | 内容 |
|------|------|
| **ドキュメントID** | AFE-STD-402 |
| **タイトル** | クロックインターフェース標準 |
| **バージョン** | Ver.1.0.0 |
| **作成日** | 2026-03-02 |
| **最終更新日** | 2026-03-02 |
| **ステータス** | ✅ 初版発行 |
| **PSR対応** | PSR-20（Clock）代替 |
| **関連標準** | AFE-STD-100, AFE-STD-300, AFE-STD-401 |

---

## 1. 概要

AFE-STD-402 は AFE における **時刻取得インターフェースの標準仕様**を定義する。PSR-20 を代替し、タイムゾーン強制・テスト用可変クロック・高精度タイムスタンプ・UTC 強制を追加する。

`new DateTimeImmutable()` や `time()` の直接使用を禁止し、すべての時刻取得を `ClockInterface` 経由に統一することで、テスト可能性とタイムゾーン安全性を保証する。

---

## 2. PSR-20 との互換性・差異

| 項目 | PSR-20 | AFE-STD-402 | 互換性 |
|------|--------|-------------|--------|
| `ClockInterface::now()` | ✅ 定義 | ✅ **維持** | ✅ 互換 |
| 戻り値型 | `DateTimeImmutable` | ✅ **同一** | ✅ 互換 |
| タイムゾーン規定 | 未定義 | ✅ **UTC 強制** | AFE拡張 |
| テスト用可変クロック | 未定義 | ✅ **FrozenClock / TravelingClock** | AFE拡張 |
| Unix タイムスタンプ | 未定義 | ✅ **unixTimestamp() 追加** | AFE拡張 |
| 高精度タイムスタンプ | 未定義 | ✅ **microtime 対応** | AFE拡張 |
| `new DateTimeImmutable()` 直接使用 | 許可 | ❌ **禁止**（ClockInterface 必須） | AFE拡張 |

---

## 3. インターフェース定義

### 3.1 メインクロックインターフェース

```php
<?php

declare(strict_types=1);

namespace Adlaire\Clock\Contracts;

use DateTimeImmutable;

/**
 * AFE-STD-402 準拠 クロックインターフェース
 * PSR-20 ClockInterface の代替仕様（拡張）
 */
interface ClockInterface
{
    /**
     * 現在の UTC 日時を取得（PSR-20 互換）
     * 必ず UTC タイムゾーンで返すこと
     */
    public function now(): DateTimeImmutable;

    /**
     * 現在の Unix タイムスタンプ（秒）
     */
    public function unixTimestamp(): int;

    /**
     * 現在の Unix タイムスタンプ（マイクロ秒精度）
     */
    public function microtime(): float;

    /**
     * ISO 8601 形式の UTC 文字列
     * 例: "2026-03-02T12:34:56.789Z"
     */
    public function toIso8601(): string;
}
```

### 3.2 テスト用クロック（AFE固有）

```php
<?php

declare(strict_types=1);

namespace Adlaire\Clock;

use Adlaire\Clock\Contracts\ClockInterface;
use DateTimeImmutable;
use DateTimeZone;
use DateInterval;

/**
 * テスト用の固定クロック（時間を止める）
 */
final class FrozenClock implements ClockInterface
{
    private DateTimeImmutable $frozenAt;

    public function __construct(DateTimeImmutable $frozenAt)
    {
        $this->frozenAt = $frozenAt->setTimezone(new DateTimeZone('UTC'));
    }

    public static function at(string $datetime): self
    {
        return new self(new DateTimeImmutable($datetime, new DateTimeZone('UTC')));
    }

    public function now(): DateTimeImmutable  { return $this->frozenAt; }
    public function unixTimestamp(): int      { return $this->frozenAt->getTimestamp(); }
    public function microtime(): float        { return (float) $this->frozenAt->format('U.u'); }
    public function toIso8601(): string       { return $this->frozenAt->format('Y-m-d\TH:i:s.v\Z'); }

    /**
     * 時間を進める（テスト用）
     */
    public function advance(DateInterval $interval): void
    {
        $this->frozenAt = $this->frozenAt->add($interval);
    }

    /**
     * 指定秒数進める（テスト用）
     */
    public function advanceSeconds(int $seconds): void
    {
        $this->frozenAt = $this->frozenAt->modify("+{$seconds} seconds");
    }
}

/**
 * テスト用の時間旅行クロック（一定速度で進む）
 */
final class TravelingClock implements ClockInterface
{
    public function __construct(
        private readonly DateTimeImmutable $startedAt,
        private readonly float $speedMultiplier = 1.0,
        private readonly float $startedRealTime = 0.0,
    ) {}

    public static function startingAt(string $datetime, float $speed = 1.0): self
    {
        return new self(
            startedAt: new DateTimeImmutable($datetime, new \DateTimeZone('UTC')),
            speedMultiplier: $speed,
            startedRealTime: microtime(true),
        );
    }

    public function now(): DateTimeImmutable
    {
        $elapsed = (microtime(true) - $this->startedRealTime) * $this->speedMultiplier;
        return $this->startedAt->modify("+{$elapsed} seconds");
    }

    public function unixTimestamp(): int  { return $this->now()->getTimestamp(); }
    public function microtime(): float    { return (float) $this->now()->format('U.u'); }
    public function toIso8601(): string   { return $this->now()->format('Y-m-d\TH:i:s.v\Z'); }
}
```

### 3.3 システムクロック（本番実装）

```php
<?php

declare(strict_types=1);

namespace Adlaire\Clock;

use Adlaire\Clock\Contracts\ClockInterface;
use DateTimeImmutable;
use DateTimeZone;

final class SystemClock implements ClockInterface
{
    private readonly DateTimeZone $utc;

    public function __construct()
    {
        $this->utc = new DateTimeZone('UTC');
    }

    public function now(): DateTimeImmutable
    {
        return new DateTimeImmutable('now', $this->utc);
    }

    public function unixTimestamp(): int  { return time(); }
    public function microtime(): float    { return microtime(true); }
    public function toIso8601(): string   { return $this->now()->format('Y-m-d\TH:i:s.v\Z'); }
}
```

---

## 4. 実装規則

```
RULE-402-01: new DateTimeImmutable() の直接使用を禁止する（ClockInterface 経由のみ許可）
RULE-402-02: time() / microtime() 関数の直接呼び出しを禁止する（ClockInterface 経由のみ）
RULE-402-03: date() / strtotime() 関数の使用を禁止する（DateTimeImmutable メソッドを使用）
RULE-402-04: ClockInterface は必ずコンストラクタ注入で使用すること
RULE-402-05: 本番環境では SystemClock、テスト環境では FrozenClock を使用すること
RULE-402-06: ClockInterface から取得した DateTimeImmutable は必ず UTC であることを前提とすること
RULE-402-07: 表示目的のみタイムゾーン変換を行い、保存・計算は必ず UTC で行うこと
```

---

## 5. テストでの使用例

```php
<?php

declare(strict_types=1);

namespace Adlaire\Tests\User;

use Adlaire\Clock\FrozenClock;
use Adlaire\User\Services\UserService;
use PHPUnit\Framework\TestCase;

final class UserServiceTest extends TestCase
{
    public function test_create_user_sets_created_at_to_current_utc_time(): void
    {
        $frozenClock = FrozenClock::at('2026-03-02T00:00:00Z');
        $service = new UserService(
            repository: $this->createMock(UserRepositoryInterface::class),
            clock: $frozenClock,
        );

        $user = $service->create(new CreateUserCommand('Test User', 'test@adlaire.dev'));

        $this->assertEquals('2026-03-02T00:00:00.000Z', $user->createdAt()->toIso8601());
    }

    public function test_subscription_expires_after_30_days(): void
    {
        $clock = FrozenClock::at('2026-01-01T00:00:00Z');
        $subscription = new Subscription(createdAt: $clock->now(), durationDays: 30);

        $clock->advanceSeconds(31 * 86400);

        $this->assertTrue($subscription->isExpired($clock->now()));
    }
}
```

---

## 変更履歴

本ファイルの変更履歴は `../VERSION_HISTORY.md` の `AFE-STD-402` セクションを参照すること。

---

*© 2026 Adlaire Group. All Rights Reserved.*
