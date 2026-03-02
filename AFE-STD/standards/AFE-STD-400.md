# AFE-STD-400: ロガーインターフェース標準

| 項目 | 内容 |
|------|------|
| **ドキュメントID** | AFE-STD-400 |
| **タイトル** | ロガーインターフェース標準 |
| **バージョン** | Ver.1.0.0 |
| **作成日** | 2026-03-02 |
| **最終更新日** | 2026-03-02 |
| **ステータス** | ✅ 初版発行 |
| **オーナー組織** | Adlaire Group |
| **権限保持者** | 組織経営管理セグメント ゼネラルマネージャー兼 Adlaire Group DX事業セグメントグループ ゼネラルマネージャー 倉田 和宏 |
| **機密レベル** | 社内限定（internal-only） |
| **PSR対応** | PSR-3（Logger Interface）代替 |
| **関連標準** | AFE-STD-100, AFE-STD-300, AFE-STD-401 |

---

## 目次

1. [概要](#1-概要)
2. [スコープ](#2-スコープ)
3. [PSR-3 との互換性・差異](#3-psr-3-との互換性差異)
4. [Adlaire 固有の拡張](#4-adlaire-固有の拡張)
5. [ロガーインターフェース定義](#5-ロガーインターフェース定義)
6. [ログレベル規則](#6-ログレベル規則)
7. [構造化ログ規則](#7-構造化ログ規則)
8. [ログハンドラー規則](#8-ログハンドラー規則)
9. [コンテキスト伝播規則](#9-コンテキスト伝播規則)
10. [実装コード例](#10-実装コード例)
11. [違反パターン](#11-違反パターン)

---

## 1. 概要

AFE-STD-400 は、Adlaire Framework Ecosystem（AFE）における**ロガーの標準インターフェース仕様**を定義する。

本標準は PSR-3 LoggerInterface を**代替**するものであり、構造化ログ（Structured Logging）の強制・型安全なコンテキスト管理・ログレベルの厳格化・分散トレーシング対応を追加することで、PSR-3 を大幅に拡張する。

### 設計方針

- **構造化ログ必須**: すべてのログはキー・バリュー形式の構造化コンテキストを持つ
- **型安全なコンテキスト**: コンテキストの型安全性を保証する LogContext オブジェクトを導入
- **トレース対応**: Trace ID / Span ID を標準フィールドとして組み込む
- **ログレベルの厳格化**: PSR-3 の 8 レベルを維持しつつ、使用基準を厳格に定義
- **ハンドラー分離**: ログの書き込み先（Handler）を独立したインターフェースで管理

---

## 2. スコープ

### 2.1 適用範囲

- AFE フレームワーク本体のロガー実装すべて
- AFE ベースの全プロジェクトにおけるログ出力
- サービス・リポジトリ・ミドルウェア等すべてのコンポーネント

### 2.2 適用外

- アプリケーションが内部で使用するデバッグ専用 dump/dd 系の出力（開発環境のみ）
- 外部 SaaS ロギングサービスの SDK（AFE-STD-400 準拠のアダプター経由で使用すること）

---

## 3. PSR-3 との互換性・差異

### 3.1 互換性マトリックス

| 項目 | PSR-3 | AFE-STD-400 | 互換性 |
|------|-------|-------------|--------|
| ログレベル（8段階） | ✅ emergency〜debug | ✅ **同一レベル維持** | ✅ セマンティクス互換 |
| `log(level, message, context)` | ✅ 定義 | ✅ **LogLevel Enum に変更** | △ 型変更 |
| `emergency/alert/.../debug()` | ✅ 8メソッド | ✅ **維持** | ✅ 互換 |
| コンテキスト型 | `array<string,mixed>` | ✅ **LogContext オブジェクト** | ❌ 非互換 |
| `{placeholder}` 補間 | ✅ 定義 | ❌ **廃止**（構造化ログで代替） | ❌ 非互換 |
| ハンドラー管理 | 未定義 | ✅ **LogHandlerInterface 追加** | AFE拡張 |
| トレース ID | 未定義 | ✅ **標準フィールド** | AFE拡張 |
| ログ集約・フィルタ | 未定義 | ✅ **LogPipelineInterface** | AFE拡張 |
| 非同期ログ出力 | 未定義 | ✅ **AsyncLoggerInterface** | AFE拡張 |

### 3.2 PSR-3 が廃止された理由

```
PSR-3 の問題点:
  1. コンテキストが array<string,mixed> → 型安全性なし
  2. {placeholder} 補間は XSS・インジェクションリスクを含む
  3. ハンドラー管理の標準がない → 実装依存
  4. 分散トレーシング未対応 → マイクロサービス/分散環境で機能不足

AFE-STD-400 の解決策:
  1. LogContext オブジェクトで型安全なコンテキスト管理
  2. {placeholder} 補間を廃止し構造化ログで代替
  3. LogHandlerInterface による標準ハンドラー管理
  4. TraceId/SpanId フィールドを標準化
```

---

## 4. Adlaire 固有の拡張

### 4.1 LogContext オブジェクト

```php
// PSR-3（廃止）
$logger->info('User created', ['user_id' => 123, 'email' => 'foo@example.com']);

// AFE-STD-400（必須）
$logger->info(
    message: 'User created',
    context: LogContext::create()
        ->with('user_id', $user->id)
        ->with('email', $user->email->value())
        ->withTrace($request->traceId())
);
```

### 4.2 ログレベル Enum

```php
enum LogLevel: string
{
    case Emergency = 'emergency';
    case Alert     = 'alert';
    case Critical  = 'critical';
    case Error     = 'error';
    case Warning   = 'warning';
    case Notice    = 'notice';
    case Info      = 'info';
    case Debug     = 'debug';
}
```

### 4.3 構造化ログ出力例（JSON）

```json
{
  "timestamp": "2026-03-02T12:34:56.789Z",
  "level": "info",
  "message": "User created",
  "trace_id": "abc123def456",
  "span_id": "span789",
  "context": {
    "user_id": 42,
    "email": "user@adlaire.dev"
  },
  "service": "adlaire-framework",
  "environment": "production"
}
```

---

## 5. ロガーインターフェース定義

### 5.1 メインロガーインターフェース

```php
<?php

declare(strict_types=1);

namespace Adlaire\Logger\Contracts;

use Adlaire\Logger\Enums\LogLevel;
use Adlaire\Logger\ValueObjects\LogContext;

/**
 * AFE-STD-400 準拠 ロガーインターフェース
 * PSR-3 LoggerInterface の代替仕様（非互換）
 */
interface LoggerInterface
{
    /**
     * システム使用不可（データベース停止、ディスクフル等）
     */
    public function emergency(string $message, LogContext $context = new LogContext()): void;

    /**
     * 即時対応必要（SMS・ページャー通知相当）
     */
    public function alert(string $message, LogContext $context = new LogContext()): void;

    /**
     * 重大な状態（アプリケーションコンポーネントが利用不可）
     */
    public function critical(string $message, LogContext $context = new LogContext()): void;

    /**
     * ランタイムエラー（即時対応不要だが要記録）
     */
    public function error(string $message, LogContext $context = new LogContext()): void;

    /**
     * 警告（エラーではないが注意が必要な状態）
     */
    public function warning(string $message, LogContext $context = new LogContext()): void;

    /**
     * 通常だが重要なイベント
     */
    public function notice(string $message, LogContext $context = new LogContext()): void;

    /**
     * 通常の運用ログ（重要なビジネスイベント）
     */
    public function info(string $message, LogContext $context = new LogContext()): void;

    /**
     * 詳細なデバッグ情報（本番環境では出力禁止）
     */
    public function debug(string $message, LogContext $context = new LogContext()): void;

    /**
     * 任意のレベルでログ出力
     */
    public function log(LogLevel $level, string $message, LogContext $context = new LogContext()): void;

    /**
     * コンテキストプロセッサーの追加
     * リクエスト全体に共通コンテキストを自動付与する
     */
    public function withContext(LogContext $sharedContext): static;

    /**
     * チャンネル（名前付きロガー）を取得
     */
    public function channel(string $channel): static;
}
```

### 5.2 LogContext バリューオブジェクト

```php
<?php

declare(strict_types=1);

namespace Adlaire\Logger\ValueObjects;

use Adlaire\Logger\Exceptions\LogContextException;

final class LogContext
{
    /** @var array<string, scalar|null> */
    private array $data = [];

    private ?string $traceId = null;
    private ?string $spanId  = null;
    private ?string $channel = null;

    public static function create(): self
    {
        return new self();
    }

    /**
     * @param scalar|null $value mixed は禁止、型安全な値のみ受け付ける
     */
    public function with(string $key, int|float|string|bool|null $value): static
    {
        $clone = clone $this;
        $clone->data[$key] = $value;
        return $clone;
    }

    public function withTrace(string $traceId, ?string $spanId = null): static
    {
        $clone = clone $this;
        $clone->traceId = $traceId;
        $clone->spanId  = $spanId;
        return $clone;
    }

    public function withException(\Throwable $e): static
    {
        return $this
            ->with('exception_class', $e::class)
            ->with('exception_message', $e->getMessage())
            ->with('exception_file', $e->getFile())
            ->with('exception_line', $e->getLine());
    }

    /** @return array<string, scalar|null> */
    public function toArray(): array
    {
        $result = $this->data;
        if ($this->traceId !== null) {
            $result['trace_id'] = $this->traceId;
        }
        if ($this->spanId !== null) {
            $result['span_id'] = $this->spanId;
        }
        return $result;
    }
}
```

### 5.3 ログハンドラーインターフェース

```php
<?php

declare(strict_types=1);

namespace Adlaire\Logger\Contracts;

use Adlaire\Logger\ValueObjects\LogRecord;

interface LogHandlerInterface
{
    /**
     * ログレコードの書き込み
     */
    public function handle(LogRecord $record): void;

    /**
     * このハンドラーが処理すべきレベルかどうか
     */
    public function isHandling(LogLevel $level): bool;
}
```

---

## 6. ログレベル規則

### 6.1 ログレベル使用基準（AFE必須定義）

| レベル | 使用基準 | 例 | 通知要否 |
|--------|---------|-----|---------|
| `emergency` | システム全体が機能不能 | DB停止、ディスクフル | 即時（PagerDuty等） |
| `alert` | 即時対応が必要な重大障害 | APIゲートウェイ全停止 | 即時（SMS） |
| `critical` | コンポーネント機能喪失 | 決済サービス障害 | 15分以内 |
| `error` | 処理失敗（ユーザー影響あり） | 未処理例外、バリデーション以外のエラー | 1時間以内 |
| `warning` | 警告（処理は継続） | 非推奨API使用、遅延閾値超過 | 日次確認 |
| `notice` | 重要な正常イベント | ユーザーログイン、設定変更 | なし |
| `info` | 通常の業務イベント | ユーザー登録、注文完了 | なし |
| `debug` | デバッグ情報 | SQL クエリ、メソッド呼び出し詳細 | **本番禁止** |

### 6.2 ログレベル禁止事項

```
RULE-400-01: debug レベルを本番環境で出力することを禁止する
RULE-400-02: 例外のキャッチ&ログのみで再スローしない場合は必ず error 以上を使用すること
RULE-400-03: ビジネスロジックの正常系処理を error/critical で記録することを禁止する
RULE-400-04: ログメッセージに個人情報（メールアドレス、パスワード、クレカ番号等）を含めることを禁止する
```

---

## 7. 構造化ログ規則

### 7.1 必須フィールド

すべてのログレコードに以下のフィールドを**自動付与**すること：

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `timestamp` | ISO8601（ミリ秒精度） | ログ発生日時（UTC） |
| `level` | string | ログレベル名（小文字） |
| `message` | string | ログメッセージ |
| `service` | string | サービス名（`adlaire-framework` 等） |
| `environment` | string | 実行環境（`production`/`staging`/`local`） |
| `channel` | string | ログチャンネル（デフォルト: `app`） |

### 7.2 推奨フィールド

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `trace_id` | string | 分散トレーシング用トレースID |
| `span_id` | string | スパンID |
| `user_id` | int\|null | 認証ユーザーID（個人情報は含めない） |
| `request_id` | string | HTTPリクエストID |
| `duration_ms` | float | 処理時間（ミリ秒） |

### 7.3 {placeholder} 補間の廃止

```php
// ❌ 禁止: PSR-3 の {placeholder} 補間
$logger->info('User {user_id} was created', ['user_id' => 42]);

// ✅ 必須: 構造化コンテキスト
$logger->info(
    message: 'User was created',
    context: LogContext::create()->with('user_id', $user->id)
);
```

---

## 8. ログハンドラー規則

### 8.1 標準ハンドラー

AFE は以下の標準ハンドラーを提供する：

| ハンドラー | 説明 | 本番推奨 |
|-----------|------|---------|
| `StreamLogHandler` | ファイル・STDOUT への書き込み | ✅ |
| `DatabaseLogHandler` | DBへのログ保存 | △（低頻度のみ） |
| `AsyncLogHandler` | キュー経由の非同期出力 | ✅（高負荷環境） |
| `NullLogHandler` | ログ破棄（テスト用） | テストのみ |

### 8.2 ハンドラーチェーン（LogPipeline）

```php
$logger = LoggerFactory::create()
    ->addHandler(new StreamLogHandler(stream: 'stderr', minLevel: LogLevel::Warning))
    ->addHandler(new AsyncLogHandler(queue: 'logs', minLevel: LogLevel::Info))
    ->build();
```

---

## 9. コンテキスト伝播規則

### 9.1 リクエストスコープのコンテキスト自動付与

```php
// ミドルウェアでリクエストコンテキストを設定
final class LogContextMiddleware implements MiddlewareInterface
{
    public function process(
        ServerRequestInterface $request,
        RequestHandlerInterface $handler
    ): ResponseInterface {
        $sharedContext = LogContext::create()
            ->withTrace($request->getAttribute('trace_id'))
            ->with('request_id', $request->getAttribute('request_id'))
            ->with('user_id', $request->getAttribute('user_id'));

        $logger = $this->logger->withContext($sharedContext);
        // コンテナに登録して以降のサービスが共有コンテキスト付きロガーを使用
        $this->container->instance(LoggerInterface::class, $logger);

        return $handler->handle($request);
    }
}
```

---

## 10. 実装コード例

### 10.1 サービスでのログ使用

```php
<?php

declare(strict_types=1);

namespace Adlaire\User\Services;

use Adlaire\Logger\Contracts\LoggerInterface;
use Adlaire\Logger\ValueObjects\LogContext;
use Adlaire\User\Contracts\UserServiceInterface;
use Adlaire\User\Contracts\UserRepositoryInterface;
use Adlaire\User\Commands\CreateUserCommand;
use Adlaire\User\Entities\User;

final class UserService implements UserServiceInterface
{
    public function __construct(
        private readonly UserRepositoryInterface $repository,
        private readonly LoggerInterface $logger,
    ) {}

    public function create(CreateUserCommand $command): User
    {
        $this->logger->info(
            message: 'Creating new user',
            context: LogContext::create()
                ->with('email_domain', $command->email->domain())
        );

        try {
            $user = $this->repository->save(User::create($command));

            $this->logger->info(
                message: 'User created successfully',
                context: LogContext::create()->with('user_id', $user->id->value())
            );

            return $user;
        } catch (\Throwable $e) {
            $this->logger->error(
                message: 'Failed to create user',
                context: LogContext::create()
                    ->withException($e)
                    ->with('email_domain', $command->email->domain())
            );
            throw $e;
        }
    }
}
```

---

## 11. 違反パターン

```php
// ❌ VIOLATION-400-01: debug を本番で出力
$logger->debug('SQL query', LogContext::create()->with('query', $sql)); // 本番禁止

// ❌ VIOLATION-400-02: 個人情報をコンテキストに含める
$logger->info('User login', LogContext::create()->with('password', $password)); // 絶対禁止

// ❌ VIOLATION-400-03: PSR-3 の {placeholder} 補間の使用
$logger->info('User {id} logged in', ['id' => $userId]); // 禁止

// ❌ VIOLATION-400-04: error でビジネス正常系を記録
$logger->error('User found', LogContext::create()->with('user_id', 1)); // 禁止（info を使用）

// ❌ VIOLATION-400-05: array でコンテキストを渡す（PSR-3 形式）
$logger->info('message', ['key' => 'value']); // LogContext オブジェクト必須
```

---

## 変更履歴

本ファイルの変更履歴は `../VERSION_HISTORY.md` の `AFE-STD-400` セクションを参照すること。

---

*© 2026 Adlaire Group. All Rights Reserved. 本ドキュメントは Adlaire Group の機密情報です。*
