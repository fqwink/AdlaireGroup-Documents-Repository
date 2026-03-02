# AFE-STD-203: HTTP クライアントインターフェース標準

| 項目 | 内容 |
|------|------|
| **ドキュメントID** | AFE-STD-203 |
| **タイトル** | HTTP クライアントインターフェース標準 |
| **バージョン** | Ver.1.0.0 |
| **作成日** | 2026-03-02 |
| **最終更新日** | 2026-03-02 |
| **ステータス** | ✅ 初版発行 |
| **PSR対応** | PSR-18（HTTP Client）代替 |
| **関連標準** | AFE-STD-200, AFE-STD-201, AFE-STD-400 |

---

## 1. 概要

AFE-STD-203 は AFE における **外部 HTTP 通信クライアントの標準インターフェース仕様**を定義する。PSR-18 を代替し、リトライ戦略・タイムアウト設定・認証インターセプター・非同期通信を追加する。

---

## 2. PSR-18 との互換性・差異

| 項目 | PSR-18 | AFE-STD-203 | 互換性 |
|------|--------|-------------|--------|
| `ClientInterface::sendRequest()` | ✅ 定義 | ✅ **維持** | ✅ 互換 |
| `ClientExceptionInterface` | ✅ 定義 | ✅ **独自例外に置換** | △ |
| `NetworkExceptionInterface` | ✅ 定義 | ✅ **独自例外に置換** | △ |
| `RequestExceptionInterface` | ✅ 定義 | ✅ **独自例外に置換** | △ |
| リトライ戦略 | 未定義 | ✅ **RetryStrategyInterface** | AFE拡張 |
| タイムアウト設定 | 未定義 | ✅ **ClientOptions 統合** | AFE拡張 |
| 認証インターセプター | 未定義 | ✅ **AuthInterceptorInterface** | AFE拡張 |
| 非同期通信 | 未定義 | ✅ **AsyncHttpClientInterface** | AFE拡張 |
| リクエストロギング | 未定義 | ✅ **自動ロギングインターセプター** | AFE拡張 |

---

## 3. インターフェース定義

### 3.1 メイン HTTP クライアント

```php
<?php

declare(strict_types=1);

namespace Adlaire\Http\Client\Contracts;

use Adlaire\Http\Contracts\ServerRequestInterface;
use Adlaire\Http\Contracts\ResponseInterface;
use Adlaire\Http\Client\ValueObjects\ClientOptions;
use Adlaire\Http\Client\Exceptions\HttpClientException;

/**
 * AFE-STD-203 準拠 HTTP クライアントインターフェース
 * PSR-18 ClientInterface の代替仕様（拡張）
 */
interface HttpClientInterface
{
    /**
     * HTTP リクエスト送信（PSR-18 互換）
     *
     * @throws HttpClientException
     */
    public function sendRequest(ServerRequestInterface $request): ResponseInterface;

    /**
     * オプション付きリクエスト送信（AFE拡張）
     *
     * @throws HttpClientException
     */
    public function send(
        ServerRequestInterface $request,
        ClientOptions $options = new ClientOptions()
    ): ResponseInterface;

    /**
     * インターセプター追加
     */
    public function withInterceptor(InterceptorInterface $interceptor): static;

    /**
     * タイムアウト設定付きクライアント生成
     */
    public function withTimeout(int $connectTimeoutMs, int $readTimeoutMs): static;

    /**
     * リトライ設定付きクライアント生成
     */
    public function withRetry(RetryStrategyInterface $strategy): static;
}
```

### 3.2 非同期 HTTP クライアント（AFE固有）

```php
<?php

declare(strict_types=1);

namespace Adlaire\Http\Client\Contracts;

use Adlaire\Http\Contracts\ServerRequestInterface;

interface AsyncHttpClientInterface extends HttpClientInterface
{
    /**
     * 非同期リクエスト送信（Promise パターン）
     *
     * @return PromiseInterface<ResponseInterface>
     */
    public function sendAsync(ServerRequestInterface $request): PromiseInterface;

    /**
     * 複数リクエストの並列送信
     *
     * @param array<string, ServerRequestInterface> $requests
     * @return array<string, ResponseInterface>
     */
    public function sendConcurrent(array $requests, int $concurrency = 5): array;
}
```

### 3.3 リトライ戦略

```php
<?php

declare(strict_types=1);

namespace Adlaire\Http\Client\Contracts;

use Adlaire\Http\Contracts\ResponseInterface;
use Adlaire\Http\Client\Exceptions\HttpClientException;

interface RetryStrategyInterface
{
    /**
     * リトライすべきかどうか判定
     */
    public function shouldRetry(
        int $attempt,
        ?ResponseInterface $response,
        ?HttpClientException $exception
    ): bool;

    /**
     * 次のリトライまでの待機時間（ミリ秒）
     */
    public function delayMs(int $attempt): int;

    /**
     * 最大リトライ回数
     */
    public function maxAttempts(): int;
}
```

### 3.4 ClientOptions バリューオブジェクト

```php
<?php

declare(strict_types=1);

namespace Adlaire\Http\Client\ValueObjects;

final class ClientOptions
{
    public function __construct(
        public readonly int $connectTimeoutMs = 5000,
        public readonly int $readTimeoutMs    = 30000,
        public readonly bool $followRedirects  = true,
        public readonly int $maxRedirects      = 5,
        public readonly bool $verifySSL        = true,
        public readonly ?string $proxy         = null,
    ) {}

    public function withConnectTimeout(int $ms): static
    {
        return new static(
            connectTimeoutMs: $ms,
            readTimeoutMs: $this->readTimeoutMs,
            followRedirects: $this->followRedirects,
            maxRedirects: $this->maxRedirects,
            verifySSL: $this->verifySSL,
            proxy: $this->proxy,
        );
    }
}
```

---

## 4. 実装規則

```
RULE-203-01: SSL 検証（verifySSL）はデフォルト true とし、本番環境で false にすることを禁止する
RULE-203-02: タイムアウト未設定での通信を禁止する（接続タイムアウト最大 10秒、読み取りタイムアウト最大 60秒）
RULE-203-03: 外部 HTTP 通信はすべて LoggingInterceptor 経由でログを記録すること
RULE-203-04: レスポンスボディを全読み込みする前に Content-Length を確認し、10MB 超の場合はストリーミング処理を行うこと
RULE-203-05: 503/429 レスポンスに対しては自動リトライ戦略（指数バックオフ）を適用すること
```

---

## 変更履歴

本ファイルの変更履歴は `../VERSION_HISTORY.md` の `AFE-STD-203` セクションを参照すること。

---

*© 2026 Adlaire Group. All Rights Reserved.*
