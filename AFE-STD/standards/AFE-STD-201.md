# AFE-STD-201: HTTP ファクトリーインターフェース標準

| 項目 | 内容 |
|------|------|
| **ドキュメントID** | AFE-STD-201 |
| **タイトル** | HTTP ファクトリーインターフェース標準 |
| **バージョン** | Ver.1.0.0 |
| **作成日** | 2026-03-02 |
| **最終更新日** | 2026-03-02 |
| **ステータス** | ✅ 初版発行 |
| **オーナー組織** | Adlaire Group |
| **権限保持者** | 組織経営管理セグメント ゼネラルマネージャー兼 Adlaire Group DX事業セグメントグループ ゼネラルマネージャー 倉田 和宏 |
| **機密レベル** | 社内限定（internal-only） |
| **PSR対応** | PSR-17（HTTP Factories）代替 |
| **関連標準** | AFE-STD-200, AFE-STD-202 |

---

## 1. 概要

AFE-STD-201 は、AFE における **HTTP メッセージオブジェクトのファクトリーインターフェース標準**を定義する。PSR-17 を代替し、型安全な生成メソッドと Adlaire 固有のファクトリーパターンを提供する。

---

## 2. PSR-17 との互換性・差異

| 項目 | PSR-17 | AFE-STD-201 | 互換性 |
|------|--------|-------------|--------|
| `RequestFactoryInterface` | ✅ 定義 | ✅ **拡張** | △ 拡張 |
| `ResponseFactoryInterface` | ✅ 定義 | ✅ **拡張** | △ 拡張 |
| `ServerRequestFactoryInterface` | ✅ 定義 | ✅ **拡張** | △ 拡張 |
| `StreamFactoryInterface` | ✅ 定義 | ✅ **拡張** | △ 拡張 |
| `UriFactoryInterface` | ✅ 定義 | ✅ **維持** | ✅ 互換 |
| `UploadedFileFactoryInterface` | ✅ 定義 | ✅ **バリデーション統合** | △ 拡張 |
| 統合ファクトリー | 未定義 | ✅ **HttpMessageFactoryInterface** | AFE拡張 |
| PHP globals からの生成 | 実装依存 | ✅ **fromGlobals() 標準化** | AFE拡張 |

### 2.1 PSR-17 廃止理由

```
PSR-17 の問題点:
  1. ファクトリーが6つの独立したインターフェースに分散 → DI が複雑
  2. PHP superglobals からの生成方法が未標準化 → 実装依存
  3. string $uri 引数が型安全でない

AFE-STD-201 の解決策:
  1. 統合ファクトリー HttpMessageFactoryInterface で一元管理
  2. fromGlobals() を標準化
  3. Uri バリューオブジェクトで型安全化
```

---

## 3. Adlaire 固有の拡張

### 3.1 統合ファクトリーインターフェース

```php
<?php

declare(strict_types=1);

namespace Adlaire\Http\Contracts;

use Adlaire\Http\ValueObjects\HttpMethod;
use Adlaire\Http\ValueObjects\HttpStatusCode;
use Adlaire\Http\ValueObjects\Uri;

/**
 * AFE-STD-201 準拠 HTTP メッセージ統合ファクトリーインターフェース
 * PSR-17 の6インターフェースを統合した AFE 独自仕様
 */
interface HttpMessageFactoryInterface
{
    // --- リクエスト生成 ---

    public function createRequest(HttpMethod $method, Uri $uri): ServerRequestInterface;

    /**
     * PHP superglobals からサーバーリクエストを生成（標準化）
     */
    public function createServerRequestFromGlobals(): ServerRequestInterface;

    public function createServerRequest(
        HttpMethod $method,
        Uri $uri,
        array $serverParams = []
    ): ServerRequestInterface;

    // --- レスポンス生成 ---

    public function createResponse(
        HttpStatusCode $code = HttpStatusCode::Ok,
        string $reasonPhrase = ''
    ): ResponseInterface;

    public function createJsonResponse(
        array $data,
        HttpStatusCode $code = HttpStatusCode::Ok
    ): ResponseInterface;

    public function createRedirectResponse(
        string $url,
        HttpStatusCode $code = HttpStatusCode::Found
    ): ResponseInterface;

    // --- ストリーム生成 ---

    public function createStream(string $content = ''): StreamInterface;
    public function createStreamFromFile(string $filename, string $mode = 'r'): StreamInterface;
    public function createStreamFromResource(mixed $resource): StreamInterface;

    // --- URI 生成 ---

    public function createUri(string $uri = ''): Uri;

    // --- アップロードファイル生成 ---

    public function createUploadedFile(
        StreamInterface $stream,
        ?int $size = null,
        int $error = UPLOAD_ERR_OK,
        ?string $clientFilename = null,
        ?string $clientMediaType = null
    ): UploadedFileInterface;
}
```

### 3.2 HttpMethod Enum

```php
<?php

declare(strict_types=1);

namespace Adlaire\Http\ValueObjects;

enum HttpMethod: string
{
    case Get     = 'GET';
    case Post    = 'POST';
    case Put     = 'PUT';
    case Patch   = 'PATCH';
    case Delete  = 'DELETE';
    case Head    = 'HEAD';
    case Options = 'OPTIONS';
    case Connect = 'CONNECT';
    case Trace   = 'TRACE';
}
```

### 3.3 HttpStatusCode Enum（主要コード）

```php
<?php

declare(strict_types=1);

namespace Adlaire\Http\ValueObjects;

enum HttpStatusCode: int
{
    // 2xx Success
    case Ok                  = 200;
    case Created             = 201;
    case Accepted            = 202;
    case NoContent           = 204;

    // 3xx Redirection
    case MovedPermanently    = 301;
    case Found               = 302;
    case SeeOther            = 303;
    case NotModified         = 304;

    // 4xx Client Error
    case BadRequest          = 400;
    case Unauthorized        = 401;
    case Forbidden           = 403;
    case NotFound            = 404;
    case MethodNotAllowed    = 405;
    case Conflict            = 409;
    case UnprocessableEntity = 422;
    case TooManyRequests     = 429;

    // 5xx Server Error
    case InternalServerError = 500;
    case NotImplemented      = 501;
    case ServiceUnavailable  = 503;

    public function isSuccessful(): bool
    {
        return $this->value >= 200 && $this->value < 300;
    }

    public function isClientError(): bool
    {
        return $this->value >= 400 && $this->value < 500;
    }

    public function isServerError(): bool
    {
        return $this->value >= 500;
    }
}
```

---

## 4. 実装規則

```
RULE-201-01: HTTP メッセージオブジェクトの生成は必ず HttpMessageFactoryInterface 経由で行うこと
RULE-201-02: new Request() / new Response() 等のコンストラクタ直接呼び出しは禁止
RULE-201-03: PHP superglobals（$_GET, $_POST, $_SERVER等）への直接アクセス禁止（createServerRequestFromGlobals() のみ許可）
RULE-201-04: ファクトリーは必ずコンテナ経由で注入すること（静的呼び出し禁止）
```

---

## 変更履歴

本ファイルの変更履歴は `../VERSION_HISTORY.md` の `AFE-STD-201` セクションを参照すること。

---

*© 2026 Adlaire Group. All Rights Reserved.*
