# AFE-STD-604: Go 連携ブリッジ標準

| 項目 | 内容 |
|------|------|
| **ドキュメントID** | AFE-STD-604 |
| **タイトル** | Go 連携ブリッジ標準 |
| **バージョン** | Ver.1.0-1 |
| **作成日** | 2026-03-02 |
| **最終更新日** | 2026-03-03 |
| **ステータス** | ✅ 確定（初版） |
| **PSR対応** | 独自仕様（旧 AFE-STD-005 後継） |
| **関連標準** | AFE-STD-100, AFE-STD-200, AFE-STD-300, AFE-STD-400 |

---

## 目次

1. [概要](#1-概要)
2. [スコープ](#2-スコープ)
3. [Go 連携アーキテクチャ](#3-go-連携アーキテクチャ)
4. [PHP↔Go 通信プロトコル規則](#4-phpgo-通信プロトコル規則)
5. [ブリッジインターフェース定義](#5-ブリッジインターフェース定義)
6. [データシリアライゼーション規則](#6-データシリアライゼーション規則)
7. [FrankenPHP・RoadRunner 連携規則](#7-frankenphroadrunner-連携規則)
8. [エラーハンドリング規則](#8-エラーハンドリング規則)
9. [パフォーマンス規則](#9-パフォーマンス規則)
10. [実装コード例](#10-実装コード例)
11. [違反パターン](#11-違反パターン)

---

## 1. 概要

AFE-STD-604 は AFE における **PHP と Go 言語間の相互運用ブリッジの標準インターフェース仕様**を定義する。PSR には対応する規格がないため、AFE 独自仕様（旧 AFE-STD-005 後継）として策定する。

本標準は以下のユースケースを対象とする：

1. **FrankenPHP**: Go 製 PHP サーバーとの統合（ワーカーモード）
2. **RoadRunner**: Go 製ワーカーサーバーとの連携（ジョブキュー・gRPC）
3. **カスタム Go ゲートウェイ**: AFE 専用 Go ゲートウェイとの通信
4. **Go マイクロサービス呼び出し**: PHPから Go サービスへの gRPC 呼び出し

### 設計方針

- **通信プロトコルの統一**: PHP↔Go 間の通信は Protocol Buffers（protobuf）+ gRPC を標準とする
- **型安全な相互運用**: protobuf 定義から生成した型安全なクライアントを使用する
- **ワーカーモードの抽象化**: FrankenPHP/RoadRunner のワーカーモードを統一インターフェースで管理
- **シリアライゼーションの標準化**: JSON と protobuf の使い分け基準を明確化

---

## 2. スコープ

### 2.1 適用範囲

- PHP → Go サービス呼び出しのすべて（gRPC/HTTP）
- FrankenPHP ワーカーモードの実装
- RoadRunner ジョブワーカーの実装
- PHP↔Go 間のデータ交換フォーマット

### 2.2 適用外

- Go 側のコード実装（AFE Go 標準で別途規定予定）
- PHP と Go を直接 FFI で連携すること（安全性上禁止）

---

## 3. Go 連携アーキテクチャ

### 3.1 推奨連携パターン

```
パターン A: FrankenPHP ワーカーモード（推奨 2026 Q4〜）
┌──────────────────────────────────────────┐
│  FrankenPHP (Go)                         │
│  ├── HTTP/2・HTTP/3 サーバー              │
│  ├── PHP ワーカープール管理               │
│  └── 静的ファイル配信                    │
│         ↓ FastCGI / Embedded PHP         │
│  PHP AFE アプリケーション                │
└──────────────────────────────────────────┘

パターン B: RoadRunner + Go（検討中 2027〜）
┌──────────────────────────────────────────┐
│  RoadRunner (Go)                         │
│  ├── HTTP サーバー                       │
│  ├── ジョブキュー (AMQP/Redis)           │
│  ├── gRPC サーバー                       │
│  └── PHP ワーカー管理                    │
│         ↓ Goridge プロトコル             │
│  PHP AFE ワーカー                        │
└──────────────────────────────────────────┘

パターン C: カスタム Go ゲートウェイ（2027 以降）
┌──────────────────────────────────────────┐
│  Go AFE Gateway                          │
│  ├── 認証・認可                          │
│  ├── レートリミット                      │
│  ├── ルーティング                        │
│  └── gRPC ↔ PHP 変換                   │
│         ↓ gRPC / HTTP/2                  │
│  PHP AFE アプリケーション                │
└──────────────────────────────────────────┘
```

---

## 4. PHP↔Go 通信プロトコル規則

### 4.1 プロトコル選択基準

| 用途 | プロトコル | 理由 |
|------|---------|------|
| サービス間 API 呼び出し | gRPC + protobuf | 高速・型安全・スキーマ定義 |
| イベント通知 | gRPC Streaming | 双方向ストリーミング対応 |
| 外部クライアント向け | REST + JSON | 互換性・デバッグ容易性 |
| ジョブキュー | RoadRunner Goridge | バイナリ効率的 |
| FrankenPHP 連携 | Embedded PHP | ネットワーク不要 |

### 4.2 プロトコル禁止事項

```
RULE-604-01: PHP と Go 間で PHP シリアライズ（serialize()）を使用することを禁止する
RULE-604-02: PHP と Go 間で XML を通信フォーマットとして使用することを禁止する
RULE-604-03: PHP FFI（外部関数インターフェース）による直接 Go 呼び出しを禁止する
RULE-604-04: TCP ソケットへの直接書き込みを禁止する（BridgeInterface 経由のみ）
```

---

## 5. ブリッジインターフェース定義

### 5.1 Go ブリッジインターフェース

```php
<?php

declare(strict_types=1);

namespace Adlaire\Bridge\Go\Contracts;

/**
 * AFE-STD-604 準拠 Go ブリッジインターフェース
 * PHP から Go サービスを呼び出す統一インターフェース
 */
interface GoBridgeInterface
{
    /**
     * gRPC ユナリー呼び出し
     *
     * @template TRequest of \Google\Protobuf\Internal\Message
     * @template TResponse of \Google\Protobuf\Internal\Message
     * @param class-string<TResponse> $responseClass
     * @return TResponse
     * @throws GoBridgeException
     */
    public function call(
        string $service,
        string $method,
        \Google\Protobuf\Internal\Message $request,
        string $responseClass
    ): \Google\Protobuf\Internal\Message;

    /**
     * gRPC サーバーストリーミング
     *
     * @template TResponse of \Google\Protobuf\Internal\Message
     * @return iterable<TResponse>
     */
    public function stream(
        string $service,
        string $method,
        \Google\Protobuf\Internal\Message $request,
        string $responseClass
    ): iterable;

    /**
     * 接続状態確認
     */
    public function isHealthy(): bool;

    /**
     * タイムアウト設定
     */
    public function withTimeout(int $timeoutMs): static;
}
```

### 5.2 FrankenPHP ワーカーインターフェース

```php
<?php

declare(strict_types=1);

namespace Adlaire\Bridge\FrankenPHP\Contracts;

use Adlaire\Http\Contracts\ServerRequestInterface;
use Adlaire\Http\Contracts\ResponseInterface;

/**
 * FrankenPHP ワーカーモード用インターフェース
 * ワーカーモードではリクエスト間でアプリケーション状態を保持する
 */
interface WorkerInterface
{
    /**
     * ワーカー起動時の初期化（一度だけ実行）
     * コンテナのビルド・サービスの起動等
     */
    public function boot(): void;

    /**
     * リクエスト処理（リクエストごとに実行）
     */
    public function handle(ServerRequestInterface $request): ResponseInterface;

    /**
     * ワーカー終了時のクリーンアップ
     */
    public function terminate(): void;

    /**
     * リクエスト間でリセットすべき状態の定義
     * Scoped サービスのリセット等
     *
     * @return array<class-string> リセット対象サービスの class-string リスト
     */
    public function resetableServices(): array;
}
```

### 5.3 RoadRunner ジョブワーカーインターフェース

```php
<?php

declare(strict_types=1);

namespace Adlaire\Bridge\RoadRunner\Contracts;

/**
 * RoadRunner ジョブキューワーカーインターフェース
 */
interface JobWorkerInterface
{
    /**
     * ジョブ処理
     */
    public function handle(JobPayloadInterface $payload): JobResultInterface;

    /**
     * このワーカーが処理するジョブキュー名
     *
     * @return array<string>
     */
    public function queues(): array;

    /**
     * 同時並行数
     */
    public function concurrency(): int;
}
```

---

## 6. データシリアライゼーション規則

### 6.1 protobuf メッセージ定義規則

```protobuf
// AFE 標準 protobuf スキーマ規則

syntax = "proto3";

// パッケージ名: adlaire.{コンテキスト}.v{バージョン}
package adlaire.user.v1;

// オプション設定
option php_namespace = "Adlaire\\User\\Proto\\V1";
option go_package = "github.com/adlaire/framework/proto/user/v1";

// メッセージ名: StudlyCaps（AFE-STD-100 準拠）
message CreateUserRequest {
    string name  = 1;
    string email = 2;
}

message CreateUserResponse {
    int64  user_id    = 1;
    string created_at = 2; // ISO 8601 UTC
}

// サービス名: {コンテキスト}Service
service UserService {
    rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
    rpc GetUser(GetUserRequest) returns (GetUserResponse);
}
```

### 6.2 データ型マッピング

| PHP 型 | protobuf 型 | Go 型 |
|--------|-------------|-------|
| `int` | `int64` | `int64` |
| `float` | `double` | `float64` |
| `string` | `string` | `string` |
| `bool` | `bool` | `bool` |
| `DateTimeImmutable` | `string`（ISO8601） | `time.Time` |
| `array<T>` | `repeated T` | `[]T` |
| バリューオブジェクト | `message` | `struct` |

---

## 7. FrankenPHP・RoadRunner 連携規則

### 7.1 FrankenPHP ワーカーモード規則

```
RULE-604-05: ワーカーモードでは boot() 時にコンテナをコンパイルし、以降のリクエストで再ビルドを禁止する
RULE-604-06: Scoped サービスはリクエストごとに必ずリセットすること（メモリリーク防止）
RULE-604-07: ワーカー間でグローバル変数・静的プロパティに状態を保持することを禁止する
RULE-604-08: FrankenPHP ワーカーモードでの ob_start()/ob_end_flush() 直接使用を禁止する
```

### 7.2 RoadRunner 規則

```
RULE-604-09: RoadRunner ジョブのペイロードは JSON エンコードのみ使用すること
RULE-604-10: ジョブ処理は必ず冪等性を保証すること
RULE-604-11: ジョブの最大リトライ回数を必ず設定すること（デフォルト: 3回）
RULE-604-12: デッドレタートピックを必ず設定し、失敗ジョブの追跡を可能にすること
```

---

## 8. エラーハンドリング規則

```php
namespace Adlaire\Bridge\Go\Exceptions;

/** Go サービス接続エラー */
final class GoBridgeConnectionException extends GoBridgeException {}

/** gRPC ステータスエラー */
final class GrpcStatusException extends GoBridgeException
{
    public function __construct(
        public readonly int $statusCode,
        string $message,
    ) {
        parent::__construct($message);
    }
}

/** タイムアウトエラー */
final class GoBridgeTimeoutException extends GoBridgeException {}
```

```
RULE-604-13: gRPC エラーは必ず GrpcStatusException として変換し、gRPC ステータスコードを保持すること
RULE-604-14: Go サービスへの呼び出しはすべてタイムアウトを設定し、無制限待機を禁止する
RULE-604-15: サーキットブレーカーパターンを Go サービス呼び出しに適用すること
```

---

## 9. パフォーマンス規則

```
RULE-604-16: gRPC コネクションはアプリケーション起動時に確立し、リクエストごとの接続確立を禁止する（接続プーリング必須）
RULE-604-17: protobuf メッセージは必要最小限のフィールドのみ送信すること
RULE-604-18: gRPC ストリーミングは 1000 件以上のデータ転送時のみ使用すること（小規模呼び出しは Unary を使用）
```

---

## 10. 実装コード例

### 10.1 PHP から Go の UserService を gRPC で呼び出す

```php
<?php

declare(strict_types=1);

namespace Adlaire\User\Infrastructure;

use Adlaire\Bridge\Go\Contracts\GoBridgeInterface;
use Adlaire\User\Contracts\UserServiceClientInterface;
use Adlaire\User\Proto\V1\CreateUserRequest;
use Adlaire\User\Proto\V1\CreateUserResponse;

final class GrpcUserServiceClient implements UserServiceClientInterface
{
    public function __construct(
        private readonly GoBridgeInterface $bridge,
    ) {}

    public function createUser(string $name, string $email): int
    {
        $request = new CreateUserRequest();
        $request->setName($name);
        $request->setEmail($email);

        /** @var CreateUserResponse $response */
        $response = $this->bridge
            ->withTimeout(5000) // 5秒タイムアウト
            ->call(
                service: 'adlaire.user.v1.UserService',
                method:  'CreateUser',
                request: $request,
                responseClass: CreateUserResponse::class,
            );

        return (int) $response->getUserId();
    }
}
```

### 10.2 FrankenPHP ワーカーモード実装

```php
<?php

declare(strict_types=1);

namespace Adlaire\Framework\Worker;

use Adlaire\Bridge\FrankenPHP\Contracts\WorkerInterface;
use Adlaire\Container\Contracts\ContainerInterface;
use Adlaire\Http\Contracts\ServerRequestInterface;
use Adlaire\Http\Contracts\ResponseInterface;

final class AfeWorker implements WorkerInterface
{
    private ContainerInterface $container;

    public function boot(): void
    {
        // 1度だけ: コンテナビルド・コンパイル
        $this->container = (new ApplicationBootstrapper())->build();
        $this->container->compile(storagePath: '/storage/framework/container.cache');
    }

    public function handle(ServerRequestInterface $request): ResponseInterface
    {
        // リクエストごとに実行
        $kernel = $this->container->resolve(HttpKernelInterface::class);
        return $kernel->handle($request);
    }

    public function terminate(): void
    {
        // クリーンアップ
    }

    public function resetableServices(): array
    {
        return [
            AuthenticatedUserInterface::class, // Scoped: リクエストごとにリセット
            DatabaseTransactionInterface::class,
        ];
    }
}
```

---

## 11. 違反パターン

```php
// ❌ VIOLATION-604-01: PHP シリアライズで Go へデータ送信
$data = serialize($object);
socket_write($goSocket, $data); // 禁止

// ❌ VIOLATION-604-02: FFI による直接 Go 呼び出し
$ffi = FFI::load('go_library.h'); // 禁止

// ❌ VIOLATION-604-03: タイムアウト未設定の gRPC 呼び出し
$bridge->call('UserService', 'GetUser', $req, UserResponse::class); // withTimeout() 必須

// ❌ VIOLATION-604-04: ワーカーモードでのグローバル変数使用
$GLOBALS['current_user'] = $user; // 禁止（ワーカー間で汚染）

// ❌ VIOLATION-604-05: リクエストごとの gRPC 接続確立
$connection = new GrpcConnection($host); // boot() 時に1度だけ確立すること
```

---

## 変更履歴

本ファイルの変更履歴は `../VERSION_HISTORY.md` の `AFE-STD-604` セクションを参照すること。

---

*© 2026 Adlaire Group. All Rights Reserved. 本ドキュメントは Adlaire Group の機密情報です。*
