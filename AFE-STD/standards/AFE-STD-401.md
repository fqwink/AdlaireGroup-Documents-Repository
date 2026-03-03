# AFE-STD-401: イベントディスパッチャーインターフェース標準

| 項目 | 内容 |
|------|------|
| **ドキュメントID** | AFE-STD-401 |
| **タイトル** | イベントディスパッチャーインターフェース標準 |
| **バージョン** | Ver.1.0-1 |
| **作成日** | 2026-03-02 |
| **最終更新日** | 2026-03-03 |
| **ステータス** | ✅ 確定（初版） |
| **PSR対応** | PSR-14（Event Dispatcher）代替 |
| **関連標準** | AFE-STD-100, AFE-STD-300, AFE-STD-400 |

---

## 1. 概要

AFE-STD-401 は AFE における **ドメインイベントおよびアプリケーションイベントのディスパッチャー標準インターフェース仕様**を定義する。PSR-14 を代替し、型安全なイベント定義・非同期ディスパッチ・イベントソーシング対応・優先度制御を追加する。

---

## 2. PSR-14 との互換性・差異

| 項目 | PSR-14 | AFE-STD-401 | 互換性 |
|------|--------|-------------|--------|
| `EventDispatcherInterface::dispatch()` | ✅ 定義 | ✅ **型安全化** | △ 拡張 |
| `ListenerProviderInterface` | ✅ 定義 | ✅ **拡張** | △ 拡張 |
| `StoppableEventInterface` | ✅ 定義 | ✅ **維持** | ✅ 互換 |
| イベント型の制約 | 任意のオブジェクト | ✅ **EventInterface 必須** | ❌ 非互換 |
| 非同期ディスパッチ | 未定義 | ✅ **AsyncEventDispatcherInterface** | AFE拡張 |
| リスナー優先度 | 未定義 | ✅ **priority() 標準化** | AFE拡張 |
| ドメインイベント分離 | 未定義 | ✅ **DomainEventInterface** | AFE拡張 |
| イベントソーシング | 未定義 | ✅ **EventRecorderInterface** | AFE拡張 |
| イベントログ | 未定義 | ✅ **自動ログ記録** | AFE拡張 |

---

## 3. イベント基底インターフェース（AFE固有）

```php
<?php

declare(strict_types=1);

namespace Adlaire\Event\Contracts;

use DateTimeImmutable;

/**
 * AFE-STD-401 全イベントの基底インターフェース
 * PSR-14 では任意オブジェクトを許可しているが AFE では本インターフェース必須
 */
interface EventInterface
{
    /**
     * イベント発生日時（UTC）
     */
    public function occurredAt(): DateTimeImmutable;

    /**
     * イベント識別子（UUID v7 推奨）
     */
    public function eventId(): string;

    /**
     * イベント名（例: user.created, order.completed）
     */
    public function eventName(): string;
}
```

---

## 4. ドメインイベントインターフェース（AFE固有）

```php
<?php

declare(strict_types=1);

namespace Adlaire\Event\Contracts;

/**
 * ドメインイベントインターフェース（イベントソーシング対応）
 * Entity / Aggregate Root が発行するイベント
 */
interface DomainEventInterface extends EventInterface
{
    /**
     * 集約ルートの ID
     */
    public function aggregateId(): string;

    /**
     * 集約のバージョン（楽観的ロック用）
     */
    public function aggregateVersion(): int;
}
```

---

## 5. イベントディスパッチャーインターフェース定義

### 5.1 同期ディスパッチャー

```php
<?php

declare(strict_types=1);

namespace Adlaire\Event\Contracts;

/**
 * AFE-STD-401 準拠 イベントディスパッチャーインターフェース
 * PSR-14 EventDispatcherInterface の代替仕様
 */
interface EventDispatcherInterface
{
    /**
     * イベントをディスパッチ（PSR-14 互換だが引数型を強化）
     *
     * @template T of EventInterface
     * @param T $event
     * @return T
     */
    public function dispatch(EventInterface $event): EventInterface;

    /**
     * 複数イベントを一括ディスパッチ
     *
     * @param array<EventInterface> $events
     */
    public function dispatchAll(array $events): void;
}
```

### 5.2 非同期ディスパッチャー（AFE固有）

```php
<?php

declare(strict_types=1);

namespace Adlaire\Event\Contracts;

interface AsyncEventDispatcherInterface extends EventDispatcherInterface
{
    /**
     * イベントをキューに積んで非同期ディスパッチ
     * リスナーの処理はワーカープロセスで実行される
     */
    public function dispatchAsync(EventInterface $event, string $queue = 'default'): void;

    /**
     * 遅延ディスパッチ（指定時間後に処理）
     */
    public function dispatchDelayed(EventInterface $event, int $delaySeconds): void;
}
```

### 5.3 リスナープロバイダー（AFE拡張）

```php
<?php

declare(strict_types=1);

namespace Adlaire\Event\Contracts;

interface ListenerProviderInterface
{
    /**
     * イベントに対応するリスナーを取得
     * PSR-14 互換だが戻り値を型付きに強化
     *
     * @template T of EventInterface
     * @param T $event
     * @return iterable<EventListenerInterface<T>>
     */
    public function getListenersForEvent(EventInterface $event): iterable;

    /**
     * リスナーを登録
     *
     * @param class-string<EventInterface> $eventClass
     */
    public function listen(string $eventClass, EventListenerInterface $listener): void;
}
```

### 5.4 イベントリスナーインターフェース（AFE固有）

```php
<?php

declare(strict_types=1);

namespace Adlaire\Event\Contracts;

/**
 * @template T of EventInterface
 */
interface EventListenerInterface
{
    /**
     * イベント処理
     *
     * @param T $event
     */
    public function handle(EventInterface $event): void;

    /**
     * リスナーの優先度（数値が低いほど先に実行、デフォルト: 0）
     */
    public function priority(): int;

    /**
     * 非同期で実行可能かどうか
     */
    public function isAsync(): bool;
}
```

### 5.5 イベントレコーダー（イベントソーシング用）

```php
<?php

declare(strict_types=1);

namespace Adlaire\Event\Contracts;

/**
 * Aggregate Root に実装するドメインイベント記録インターフェース
 */
interface EventRecorderInterface
{
    /**
     * ドメインイベントを記録（蓄積）
     */
    public function recordEvent(DomainEventInterface $event): void;

    /**
     * 記録済みイベントを全取得してクリア
     *
     * @return array<DomainEventInterface>
     */
    public function releaseEvents(): array;
}
```

---

## 6. イベント実装規則

```
RULE-401-01: イベントクラスは必ず EventInterface を実装すること
RULE-401-02: イベントクラスはイミュータブルとして実装すること（プロパティはすべて readonly）
RULE-401-03: ドメインイベントクラスは DomainEventInterface を実装し、Aggregate Root から発行すること
RULE-401-04: リスナークラスは EventListenerInterface を実装し、1リスナー = 1イベント対応を原則とすること
RULE-401-05: イベント名は「{コンテキスト}.{エンティティ}.{動詞過去形}」形式とすること（例: user.account.created）
RULE-401-06: リスナー内でビジネスロジックを直接実装することを禁止する（サービス呼び出しのみ）
RULE-401-07: 外部 API 呼び出しを行うリスナーは必ず isAsync(): true として非同期化すること
```

---

## 7. 実装コード例

### 7.1 ドメインイベントの定義と発行

```php
<?php

declare(strict_types=1);

namespace Adlaire\User\Domain\Events;

use Adlaire\Event\Contracts\DomainEventInterface;
use DateTimeImmutable;

final class UserCreatedEvent implements DomainEventInterface
{
    public function __construct(
        private readonly string $eventId,
        private readonly string $userId,
        private readonly string $email,
        private readonly DateTimeImmutable $occurredAt,
        private readonly int $aggregateVersion,
    ) {}

    public function occurredAt(): DateTimeImmutable  { return $this->occurredAt; }
    public function eventId(): string                { return $this->eventId; }
    public function eventName(): string              { return 'user.account.created'; }
    public function aggregateId(): string            { return $this->userId; }
    public function aggregateVersion(): int          { return $this->aggregateVersion; }
    public function email(): string                  { return $this->email; }
}
```

### 7.2 リスナーの実装

```php
<?php

declare(strict_types=1);

namespace Adlaire\User\Listeners;

use Adlaire\Event\Contracts\EventListenerInterface;
use Adlaire\Event\Contracts\EventInterface;
use Adlaire\User\Domain\Events\UserCreatedEvent;
use Adlaire\Mail\Contracts\MailerInterface;

final class SendWelcomeEmailListener implements EventListenerInterface
{
    public function __construct(
        private readonly MailerInterface $mailer,
    ) {}

    public function handle(EventInterface $event): void
    {
        assert($event instanceof UserCreatedEvent);
        $this->mailer->sendWelcomeEmail($event->email());
    }

    public function priority(): int { return 10; }
    public function isAsync(): bool { return true; } // メール送信は非同期
}
```

---

## 変更履歴

本ファイルの変更履歴は `../VERSION_HISTORY.md` の `AFE-STD-401` セクションを参照すること。

---

*© 2026 Adlaire Group. All Rights Reserved.*
