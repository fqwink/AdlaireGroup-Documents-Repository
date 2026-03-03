# AFE-STD-200: HTTP メッセージインターフェース標準

| 項目 | 内容 |
|------|------|
| **ドキュメントID** | AFE-STD-200 |
| **タイトル** | HTTP メッセージインターフェース標準 |
| **バージョン** | Ver.1.0-1 |
| **作成日** | 2026-03-02 |
| **最終更新日** | 2026-03-03 |
| **ステータス** | ✅ 確定（初版） |
| **オーナー組織** | Adlaire Group |
| **権限保持者** | 組織経営管理セグメント ゼネラルマネージャー兼 Adlaire Group DX事業セグメントグループ ゼネラルマネージャー 倉田 和宏 |
| **機密レベル** | 社内限定（internal-only） |
| **PSR対応** | PSR-7（HTTP Message Interface）代替 |
| **関連標準** | AFE-STD-100, AFE-STD-201, AFE-STD-202 |

---

## 目次

1. [概要](#1-概要)
2. [スコープ](#2-スコープ)
3. [PSR-7 との互換性・差異](#3-psr-7-との互換性差異)
4. [Adlaire 固有の拡張](#4-adlaire-固有の拡張)
5. [HTTPメッセージインターフェース定義](#5-httpメッセージインターフェース定義)
6. [リクエストインターフェース定義](#6-リクエストインターフェース定義)
7. [レスポンスインターフェース定義](#7-レスポンスインターフェース定義)
8. [イミュータブル設計規則](#8-イミュータブル設計規則)
9. [ストリーム規則](#9-ストリーム規則)
10. [実装コード例](#10-実装コード例)
11. [違反パターン](#11-違反パターン)

---

## 1. 概要

AFE-STD-200 は、AFE における **HTTP リクエスト・レスポンスメッセージの標準インターフェース仕様**を定義する。

PSR-7 を代替するが、以下の点で大幅に拡張する：
- **完全イミュータブル設計**: `with*()` メソッドでの変更はすべて新インスタンスを返す（PSR-7 と同じ哲学だが、より厳格）
- **型安全なヘッダー管理**: ヘッダー名・値の型を厳格化
- **ルーティング属性の統合**: ルートパラメータをリクエストオブジェクトに統合
- **入力バリデーション統合**: リクエストオブジェクトに入力検証の仕組みを組み込む

---

## 2. スコープ

### 2.1 適用範囲

- AFE フレームワーク本体の HTTP 層実装
- AFE ベースの全プロジェクトのコントローラー・ミドルウェア
- HTTP クライアント（AFE-STD-203）との整合

### 2.2 適用外

- WebSocket メッセージ（別途 AFE-STD-700 系で規定予定）
- CLI コマンドの入出力（AFE-STD-603 参照）

---

## 3. PSR-7 との互換性・差異

### 3.1 互換性マトリックス

| 項目 | PSR-7 | AFE-STD-200 | 互換性 |
|------|-------|-------------|--------|
| イミュータブル設計 | ✅ with*()パターン | ✅ **同一** | ✅ 互換 |
| `MessageInterface` | ✅ 定義 | ✅ **拡張** | △ 上位互換 |
| `RequestInterface` | ✅ 定義 | ✅ **拡張** | △ 上位互換 |
| `ResponseInterface` | ✅ 定義 | ✅ **拡張** | △ 上位互換 |
| `ServerRequestInterface` | ✅ 定義 | ✅ **拡張・統合** | △ 拡張 |
| `StreamInterface` | ✅ 定義 | ✅ **型安全版** | △ 拡張 |
| `UriInterface` | ✅ 定義 | ✅ **イミュータブル強化** | △ 拡張 |
| `UploadedFileInterface` | ✅ 定義 | ✅ **バリデーション統合** | △ 拡張 |
| ヘッダー名の大文字小文字非依存 | ✅ 必須 | ✅ **同一** | ✅ 互換 |
| ルーティング属性 | 実装依存 | ✅ **標準統合** | AFE拡張 |
| 入力バリデーション | 未定義 | ✅ **統合** | AFE拡張 |
| typed attributes | 未定義 | ✅ **型安全属性** | AFE拡張 |

---

## 4. Adlaire 固有の拡張

### 4.1 型安全な属性アクセス

```php
// PSR-7（廃止）
$userId = $request->getAttribute('user_id'); // mixed

// AFE-STD-200（必須）
$userId = $request->typedAttribute(int::class, 'user_id'); // int
```

### 4.2 ルーティングパラメータ統合

```php
// PSR-7 では実装依存
$id = $request->getAttribute('id');

// AFE-STD-200: ルートパラメータは専用メソッドで取得
$id = $request->routeParam('id'); // string（ルートパラメータ）
```

### 4.3 JSONリクエスト専用メソッド

```php
// PSR-7 では都度パース
$data = json_decode((string) $request->getBody(), true);

// AFE-STD-200
$data = $request->json(); // array<string, mixed>
$name = $request->jsonString('name'); // string（型安全）
$age  = $request->jsonInt('age');     // int（型安全）
```

---

## 5. HTTP メッセージインターフェース定義

### 5.1 ベースメッセージインターフェース

```php
<?php

declare(strict_types=1);

namespace Adlaire\Http\Contracts;

use Adlaire\Http\ValueObjects\HeaderCollection;
use Adlaire\Http\ValueObjects\ProtocolVersion;

/**
 * AFE-STD-200 準拠 HTTP メッセージ基底インターフェース
 * PSR-7 MessageInterface の代替仕様
 */
interface MessageInterface
{
    public function getProtocolVersion(): ProtocolVersion;
    public function withProtocolVersion(ProtocolVersion $version): static;

    /** @return array<string, array<string>> */
    public function getHeaders(): array;
    public function hasHeader(string $name): bool;

    /** @return array<string> */
    public function getHeader(string $name): array;
    public function getHeaderLine(string $name): string;

    public function withHeader(string $name, string ...$values): static;
    public function withAddedHeader(string $name, string ...$values): static;
    public function withoutHeader(string $name): static;

    public function getBody(): StreamInterface;
    public function withBody(StreamInterface $body): static;
}
```

---

## 6. リクエストインターフェース定義

### 6.1 サーバーリクエストインターフェース

```php
<?php

declare(strict_types=1);

namespace Adlaire\Http\Contracts;

use Adlaire\Http\ValueObjects\HttpMethod;
use Adlaire\Http\ValueObjects\Uri;

/**
 * AFE-STD-200 準拠 サーバーリクエストインターフェース
 * PSR-7 ServerRequestInterface の代替仕様（拡張）
 */
interface ServerRequestInterface extends MessageInterface
{
    public function getMethod(): HttpMethod;
    public function withMethod(HttpMethod $method): static;

    public function getUri(): Uri;
    public function withUri(Uri $uri, bool $preserveHost = false): static;

    public function getRequestTarget(): string;
    public function withRequestTarget(string $requestTarget): static;

    /** @return array<string, string> */
    public function getServerParams(): array;

    /** @return array<string, \Adlaire\Http\ValueObjects\CookieValue> */
    public function getCookieParams(): array;
    public function withCookieParams(array $cookies): static;

    /** @return array<string, string> */
    public function getQueryParams(): array;
    public function withQueryParams(array $query): static;

    /** @return array<string, \Adlaire\Http\Contracts\UploadedFileInterface> */
    public function getUploadedFiles(): array;

    /** @return array<string, mixed>|object|null */
    public function getParsedBody(): array|object|null;
    public function withParsedBody(array|object|null $data): static;

    // --- AFE 固有拡張 ---

    /**
     * 型安全な属性取得
     * @template T
     * @param class-string<T>|'int'|'string'|'float'|'bool' $type
     * @return T
     */
    public function typedAttribute(string $type, string $key): mixed;

    public function withAttribute(string $name, mixed $value): static;
    public function withoutAttribute(string $name): static;

    /**
     * ルーティングパラメータ取得
     */
    public function routeParam(string $name): string;

    /** @return array<string, string> */
    public function routeParams(): array;

    /**
     * JSON ボディのパース（Content-Type: application/json のみ）
     * @return array<string, mixed>
     */
    public function json(): array;

    public function jsonString(string $key): string;
    public function jsonInt(string $key): int;
    public function jsonFloat(string $key): float;
    public function jsonBool(string $key): bool;

    /**
     * クライアント IP（プロキシを考慮）
     */
    public function clientIp(): string;

    /**
     * HTTPS リクエストかどうか
     */
    public function isSecure(): bool;
}
```

---

## 7. レスポンスインターフェース定義

### 7.1 レスポンスインターフェース

```php
<?php

declare(strict_types=1);

namespace Adlaire\Http\Contracts;

use Adlaire\Http\ValueObjects\HttpStatusCode;

/**
 * AFE-STD-200 準拠 HTTP レスポンスインターフェース
 * PSR-7 ResponseInterface の代替仕様（拡張）
 */
interface ResponseInterface extends MessageInterface
{
    public function getStatusCode(): HttpStatusCode;
    public function getReasonPhrase(): string;
    public function withStatus(HttpStatusCode $code, string $reasonPhrase = ''): static;

    // --- AFE 固有拡張 ---

    /**
     * JSON レスポンス生成
     * @param array<string, mixed> $data
     */
    public static function json(array $data, HttpStatusCode $status = HttpStatusCode::Ok): static;

    /**
     * リダイレクトレスポンス生成
     */
    public static function redirect(string $url, HttpStatusCode $status = HttpStatusCode::Found): static;

    /**
     * 空レスポンス（204 No Content）
     */
    public static function noContent(): static;

    /**
     * Cookie 追加
     */
    public function withCookie(\Adlaire\Http\ValueObjects\Cookie $cookie): static;
}
```

---

## 8. イミュータブル設計規則

```
RULE-200-01: HTTP メッセージオブジェクトはすべてイミュータブルとして実装すること
RULE-200-02: with*() メソッドは必ず clone を返すこと（$this の変更禁止）
RULE-200-03: コンストラクタでのみ初期化を行い、setterは禁止
RULE-200-04: MessageInterface 実装クラスに public な mutable プロパティを持たせないこと
```

```php
// ✅ 正しいイミュータブル実装
public function withHeader(string $name, string ...$values): static
{
    $clone = clone $this;
    $clone->headers[$name] = $values;
    return $clone;
}

// ❌ 禁止: $this を変更して返す
public function withHeader(string $name, string ...$values): static
{
    $this->headers[$name] = $values; // 禁止
    return $this;
}
```

---

## 9. ストリーム規則

```php
<?php

declare(strict_types=1);

namespace Adlaire\Http\Contracts;

interface StreamInterface
{
    public function __toString(): string;
    public function close(): void;
    public function detach(): mixed;
    public function getSize(): ?int;
    public function tell(): int;
    public function eof(): bool;
    public function isSeekable(): bool;
    public function seek(int $offset, int $whence = SEEK_SET): void;
    public function rewind(): void;
    public function isWritable(): bool;
    public function write(string $string): int;
    public function isReadable(): bool;
    public function read(int $length): string;
    public function getContents(): string;
    public function getMetadata(?string $key = null): mixed;
}
```

---

## 10. 実装コード例

### 10.1 コントローラーでの使用

```php
<?php

declare(strict_types=1);

namespace Adlaire\App\Http\Controllers;

use Adlaire\Http\Contracts\ServerRequestInterface;
use Adlaire\Http\Contracts\ResponseInterface;
use Adlaire\Http\ValueObjects\HttpStatusCode;
use Adlaire\User\Contracts\UserServiceInterface;

final class UserController
{
    public function __construct(
        private readonly UserServiceInterface $userService,
    ) {}

    public function show(ServerRequestInterface $request): ResponseInterface
    {
        $userId = (int) $request->routeParam('id');
        $user   = $this->userService->findById($userId);

        return ResponseInterface::json([
            'id'    => $user->id->value(),
            'name'  => $user->name->value(),
            'email' => $user->email->value(),
        ], HttpStatusCode::Ok);
    }

    public function store(ServerRequestInterface $request): ResponseInterface
    {
        $command = new CreateUserCommand(
            name:  $request->jsonString('name'),
            email: $request->jsonString('email'),
        );

        $user = $this->userService->create($command);

        return ResponseInterface::json(
            ['id' => $user->id->value()],
            HttpStatusCode::Created
        );
    }
}
```

---

## 11. 違反パターン

```php
// ❌ VIOLATION-200-01: ミュータブルな変更
$request->headers['Content-Type'] = 'application/json'; // 禁止

// ❌ VIOLATION-200-02: PSR-7 の get() 互換メソッドで mixed 取得
$userId = $request->getAttribute('user_id'); // mixed → typedAttribute() 使用

// ❌ VIOLATION-200-03: ボディを直接文字列変換してパース
$data = json_decode((string) $request->getBody(), true); // json() メソッドを使用

// ❌ VIOLATION-200-04: レスポンスで直接 echo/print
echo json_encode($data); // ResponseInterface を返すこと
```

---

## 変更履歴

本ファイルの変更履歴は `../VERSION_HISTORY.md` の `AFE-STD-200` セクションを参照すること。

---

*© 2026 Adlaire Group. All Rights Reserved. 本ドキュメントは Adlaire Group の機密情報です。*
