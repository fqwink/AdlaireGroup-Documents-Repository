# AFE-STD-202: HTTP ハンドラー・ミドルウェアインターフェース標準

| 項目 | 内容 |
|------|------|
| **ドキュメントID** | AFE-STD-202 |
| **タイトル** | HTTP ハンドラー・ミドルウェアインターフェース標準 |
| **バージョン** | Ver.1.0-1 |
| **作成日** | 2026-03-02 |
| **最終更新日** | 2026-03-03 |
| **ステータス** | ✅ 確定（初版） |
| **PSR対応** | PSR-15（HTTP Handlers）代替 |
| **関連標準** | AFE-STD-200, AFE-STD-201, AFE-STD-300 |

---

## 1. 概要

AFE-STD-202 は AFE における **HTTP リクエストハンドラーとミドルウェアパイプラインの標準インターフェース仕様**を定義する。PSR-15 を代替し、型安全なパイプライン構築・プライオリティ制御・条件付きミドルウェアを追加する。

---

## 2. PSR-15 との互換性・差異

| 項目 | PSR-15 | AFE-STD-202 | 互換性 |
|------|--------|-------------|--------|
| `RequestHandlerInterface` | ✅ 定義 | ✅ **維持** | ✅ 互換 |
| `MiddlewareInterface` | ✅ 定義 | ✅ **拡張** | △ 拡張 |
| パイプライン管理 | 未定義 | ✅ **MiddlewarePipelineInterface** | AFE拡張 |
| ミドルウェア優先度 | 未定義 | ✅ **priority() メソッド追加** | AFE拡張 |
| 条件付きミドルウェア | 未定義 | ✅ **ConditionalMiddlewareInterface** | AFE拡張 |
| グループルーティング連携 | 未定義 | ✅ **ルートグループ適用** | AFE拡張 |
| TerminableMiddleware | 未定義 | ✅ **terminate() フック** | AFE拡張 |

---

## 3. インターフェース定義

### 3.1 リクエストハンドラー（PSR-15 互換）

```php
<?php

declare(strict_types=1);

namespace Adlaire\Http\Contracts;

/**
 * AFE-STD-202 準拠 リクエストハンドラーインターフェース
 * PSR-15 RequestHandlerInterface と同一セマンティクス
 */
interface RequestHandlerInterface
{
    public function handle(ServerRequestInterface $request): ResponseInterface;
}
```

### 3.2 ミドルウェアインターフェース（AFE拡張）

```php
<?php

declare(strict_types=1);

namespace Adlaire\Http\Contracts;

/**
 * AFE-STD-202 準拠 ミドルウェアインターフェース
 * PSR-15 MiddlewareInterface の拡張仕様
 */
interface MiddlewareInterface
{
    /**
     * リクエスト処理（PSR-15 互換）
     */
    public function process(
        ServerRequestInterface $request,
        RequestHandlerInterface $handler
    ): ResponseInterface;

    /**
     * ミドルウェアの実行優先度（AFE固有）
     * 数値が小さいほど先に実行される（デフォルト: 100）
     */
    public function priority(): int;
}
```

### 3.3 終了ミドルウェアインターフェース（AFE固有）

```php
<?php

declare(strict_types=1);

namespace Adlaire\Http\Contracts;

/**
 * レスポンス送信後に後処理を行うミドルウェア
 * FastCGI/RoadRunner 環境でのレスポンス送信後処理に使用
 */
interface TerminableMiddlewareInterface extends MiddlewareInterface
{
    /**
     * レスポンス送信後に実行される（非ブロッキング）
     */
    public function terminate(
        ServerRequestInterface $request,
        ResponseInterface $response
    ): void;
}
```

### 3.4 ミドルウェアパイプラインインターフェース（AFE固有）

```php
<?php

declare(strict_types=1);

namespace Adlaire\Http\Contracts;

/**
 * AFE-STD-202 準拠 ミドルウェアパイプラインインターフェース
 */
interface MiddlewarePipelineInterface extends RequestHandlerInterface
{
    /**
     * ミドルウェアをパイプラインに追加
     * priority() 値に基づいて自動ソートされる
     */
    public function pipe(MiddlewareInterface $middleware): static;

    /**
     * 特定ルートパターンにのみ適用するミドルウェアを追加
     */
    public function pipeWhen(string $routePattern, MiddlewareInterface $middleware): static;

    /**
     * パイプラインを確定（immutable化）
     */
    public function freeze(): static;
}
```

### 3.5 条件付きミドルウェア（AFE固有）

```php
<?php

declare(strict_types=1);

namespace Adlaire\Http\Contracts;

/**
 * リクエストに応じて適用の有無を動的に決定するミドルウェア
 */
interface ConditionalMiddlewareInterface extends MiddlewareInterface
{
    /**
     * このリクエストにミドルウェアを適用するかどうか
     */
    public function shouldProcess(ServerRequestInterface $request): bool;
}
```

---

## 4. ミドルウェアスタック規則

### 4.1 標準ミドルウェア優先度テーブル

| 優先度 | ミドルウェア種別 | 例 |
|--------|---------------|-----|
| 1〜10 | セキュリティ基盤 | HTTPS強制、CORS |
| 11〜20 | トレーシング・ログ | TraceIdMiddleware, LogContextMiddleware |
| 21〜50 | 認証・認可 | AuthMiddleware, AuthorizationMiddleware |
| 51〜80 | リクエスト変換 | BodyParserMiddleware, ContentNegotiation |
| 81〜100 | キャッシュ | ResponseCacheMiddleware |
| 101〜200 | アプリケーション固有 | RateLimitMiddleware |

### 4.2 ミドルウェア実装規則

```
RULE-202-01: ミドルウェアはステートレス設計とすること（リクエスト間で状態を保持しない）
RULE-202-02: ミドルウェア内でビジネスロジックを実装することを禁止する
RULE-202-03: ミドルウェアのコンストラクタでリクエスト固有のデータを注入することを禁止する
RULE-202-04: process() 内で必ず $handler->handle($request) を呼び出すか、レスポンスを返すこと（どちらも行わないことを禁止）
RULE-202-05: TerminableMiddleware の terminate() は非同期で実行可能な設計にすること
```

### 4.3 実装例

```php
<?php

declare(strict_types=1);

namespace Adlaire\Framework\Http\Middleware;

use Adlaire\Http\Contracts\MiddlewareInterface;
use Adlaire\Http\Contracts\ServerRequestInterface;
use Adlaire\Http\Contracts\ResponseInterface;
use Adlaire\Http\Contracts\RequestHandlerInterface;
use Adlaire\Logger\Contracts\LoggerInterface;
use Adlaire\Logger\ValueObjects\LogContext;

final class TraceIdMiddleware implements MiddlewareInterface
{
    public function __construct(
        private readonly LoggerInterface $logger,
    ) {}

    public function process(
        ServerRequestInterface $request,
        RequestHandlerInterface $handler
    ): ResponseInterface {
        $traceId = $request->getHeaderLine('X-Trace-Id')
            ?: bin2hex(random_bytes(16));

        $request = $request->withAttribute('trace_id', $traceId);

        $this->logger->info(
            message: 'Request started',
            context: LogContext::create()
                ->withTrace($traceId)
                ->with('method', $request->getMethod()->value)
                ->with('path', (string) $request->getUri())
        );

        $response = $handler->handle($request);

        return $response->withHeader('X-Trace-Id', $traceId);
    }

    public function priority(): int
    {
        return 15; // トレーシング層
    }
}
```

---

## 変更履歴

本ファイルの変更履歴は `../VERSION_HISTORY.md` の `AFE-STD-202` セクションを参照すること。

---

*© 2026 Adlaire Group. All Rights Reserved.*
