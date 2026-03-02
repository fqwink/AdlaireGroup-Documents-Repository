# AFE-STD-300: コンテナインターフェース標準

| 項目 | 内容 |
|------|------|
| **ドキュメントID** | AFE-STD-300 |
| **タイトル** | コンテナインターフェース標準 |
| **バージョン** | Ver.1.0.0 |
| **作成日** | 2026-03-02 |
| **最終更新日** | 2026-03-02 |
| **ステータス** | ✅ 初版発行 |
| **オーナー組織** | Adlaire Group |
| **権限保持者** | 組織経営管理セグメント ゼネラルマネージャー兼 Adlaire Group DX事業セグメントグループ ゼネラルマネージャー 倉田 和宏 |
| **機密レベル** | 社内限定（internal-only） |
| **PSR対応** | PSR-11（ContainerInterface）代替 |
| **関連標準** | AFE-STD-100, AFE-STD-101, AFE-STD-102 |

---

## 目次

1. [概要](#1-概要)
2. [スコープ](#2-スコープ)
3. [PSR-11 との互換性・差異](#3-psr-11-との互換性差異)
4. [Adlaire 固有の拡張](#4-adlaire-固有の拡張)
5. [コンテナインターフェース定義](#5-コンテナインターフェース定義)
6. [バインディング規則](#6-バインディング規則)
7. [スコープ管理規則](#7-スコープ管理規則)
8. [サービスプロバイダー規則](#8-サービスプロバイダー規則)
9. [コンテナコンパイル・最適化](#9-コンテナコンパイルー最適化)
10. [実装コード例](#10-実装コード例)
11. [違反パターン](#11-違反パターン)

---

## 1. 概要

AFE-STD-300 は、Adlaire Framework Ecosystem（AFE）における **DIコンテナ（依存性注入コンテナ）の標準インターフェース仕様**を定義する。

本標準は PSR-11 ContainerInterface を**代替**するものであり、PSR-11 との後方互換性は**意図的に破棄**している。AFE の設計思想である「型安全性の最大化」「明示的な依存宣言」「コンパイル済みコンテナによるパフォーマンス最適化」を実現するために、PSR-11 を超える厳格な仕様を規定する。

### 設計方針

- **型安全コンテナ**: すべての解決はジェネリクス相当の型安全性を保証する
- **明示的バインディング**: 暗黙的な自動バインディングは禁止（明示的登録のみ）
- **シングルトン・トランジェント・スコープ**: 3種のライフタイム管理を標準サポート
- **コンパイル済みコンテナ**: 本番環境では必ずコンテナをコンパイルして使用する

---

## 2. スコープ

### 2.1 適用範囲

本標準は以下に適用される：

- AFE フレームワーク本体のDIコンテナ実装
- AFE ベースの全プロジェクトのサービスバインディング
- ServiceProvider の実装すべて
- コンテナを経由したすべての依存解決

### 2.2 適用外

- テストコード内での Mock 差し替え（AFE-TEST-001 で別途規定）
- 外部パッケージが提供するコンテナ（使用自体を禁止）

---

## 3. PSR-11 との互換性・差異

### 3.1 互換性マトリックス

| 項目 | PSR-11 | AFE-STD-300 | 互換性 |
|------|--------|-------------|--------|
| `get(string $id): mixed` | ✅ 定義 | ❌ **廃止**（型安全版に置換） | ❌ 非互換 |
| `has(string $id): bool` | ✅ 定義 | ✅ **維持**（セマンティクス同一） | ✅ 互換 |
| `NotFoundExceptionInterface` | ✅ 定義 | ✅ **独自例外に置換**（同セマンティクス） | △ 参考 |
| `ContainerExceptionInterface` | ✅ 定義 | ✅ **独自例外に置換** | △ 参考 |
| 自動バインディング | 実装依存 | ❌ **禁止** | ❌ 非互換 |
| スコープ管理 | 未定義 | ✅ **Singleton/Transient/Scoped 必須** | AFE拡張 |
| コンパイル | 未定義 | ✅ **本番環境必須** | AFE拡張 |
| ServiceProvider | 未定義 | ✅ **必須パターン** | AFE拡張 |
| タグ付きバインディング | 未定義 | ✅ **サポート** | AFE拡張 |
| 条件付きバインディング | 未定義 | ✅ **サポート** | AFE拡張 |

### 3.2 PSR-11 が廃止された理由

```
PSR-11 の問題点:
  1. get() の戻り値が mixed → 型安全性ゼロ
  2. 自動バインディングの可否が実装依存 → 移植性なし
  3. スコープ管理の標準化なし → ライフタイムバグが発生しやすい
  4. コンパイルの概念なし → 本番パフォーマンスが実装依存

AFE-STD-300 の解決策:
  1. resolve<T>() による型付き解決
  2. 自動バインディング完全禁止・明示登録必須
  3. Singleton/Transient/Scoped を標準化
  4. compile() によるコンテナコンパイルを義務化
```

---

## 4. Adlaire 固有の拡張

### 4.1 型安全な解決メソッド

```php
// PSR-11（廃止）
$service = $container->get('App\Services\UserService'); // mixed 戻り値

// AFE-STD-300（必須）
$service = $container->resolve(UserServiceInterface::class); // UserServiceInterface 戻り値
```

### 4.2 スコープ管理

| スコープ | 説明 | 使用場面 |
|---------|------|---------|
| `Singleton` | アプリケーション全体で1インスタンス | サービス、リポジトリ |
| `Transient` | 解決のたびに新インスタンス生成 | バリューオブジェクト、軽量DTO |
| `Scoped` | リクエスト単位で1インスタンス | ユーザーコンテキスト、リクエストスコープ |

### 4.3 タグ付きバインディング

複数の実装を同一タグで束ねてコレクション解決できる。

```php
$container->bind(LogHandlerInterface::class, FileLogHandler::class)
          ->tag('log.handler');
$container->bind(LogHandlerInterface::class, SlackLogHandler::class)
          ->tag('log.handler');

// タグ付きの全実装を解決
$handlers = $container->resolveTagged('log.handler'); // array<LogHandlerInterface>
```

### 4.4 条件付きバインディング

```php
$container->when(UserController::class)
          ->needs(RepositoryInterface::class)
          ->give(CachedUserRepository::class);
```

### 4.5 コンテナコンパイル（本番必須）

```php
// 本番環境でのみ必須
if (app()->isProduction()) {
    $container->compile(storagePath: '/storage/framework/container.cache');
}
```

---

## 5. コンテナインターフェース定義

### 5.1 メインインターフェース

```php
<?php

declare(strict_types=1);

namespace Adlaire\Container\Contracts;

use Adlaire\Container\Exceptions\ContainerException;
use Adlaire\Container\Exceptions\NotFoundException;
use Adlaire\Container\Enums\BindingScope;

/**
 * AFE-STD-300 準拠 DIコンテナインターフェース
 * PSR-11 の代替仕様（非互換）
 */
interface ContainerInterface
{
    /**
     * 型安全なサービス解決
     * PSR-11 の get(string $id): mixed を代替する
     *
     * @template T of object
     * @param class-string<T> $abstract
     * @param array<string, mixed> $parameters 追加パラメータ（Transient スコープのみ有効）
     * @return T
     * @throws NotFoundException 未登録の場合
     * @throws ContainerException 解決失敗の場合
     */
    public function resolve(string $abstract, array $parameters = []): object;

    /**
     * バインディング存在確認（PSR-11 has() と同セマンティクス）
     *
     * @param class-string $abstract
     */
    public function has(string $abstract): bool;

    /**
     * バインディング登録
     *
     * @template T of object
     * @param class-string<T> $abstract
     * @param class-string<T>|callable $concrete
     * @param BindingScope $scope デフォルト: Singleton
     */
    public function bind(
        string $abstract,
        string|callable $concrete,
        BindingScope $scope = BindingScope::Singleton
    ): BindingBuilderInterface;

    /**
     * シングルトンバインディング（bind + Singleton の省略形）
     *
     * @template T of object
     * @param class-string<T> $abstract
     * @param class-string<T>|callable $concrete
     */
    public function singleton(string $abstract, string|callable $concrete): BindingBuilderInterface;

    /**
     * トランジェントバインディング（bind + Transient の省略形）
     *
     * @template T of object
     * @param class-string<T> $abstract
     * @param class-string<T>|callable $concrete
     */
    public function transient(string $abstract, string|callable $concrete): BindingBuilderInterface;

    /**
     * インスタンス直接登録（既存インスタンスをコンテナに登録）
     *
     * @template T of object
     * @param class-string<T> $abstract
     * @param T $instance
     */
    public function instance(string $abstract, object $instance): void;

    /**
     * タグ付きバインディングの一括解決
     *
     * @param string $tag
     * @return array<object>
     */
    public function resolveTagged(string $tag): array;

    /**
     * 条件付きバインディングビルダー
     *
     * @param class-string $when 適用するクラス
     */
    public function when(string $when): ConditionalBindingBuilderInterface;

    /**
     * コンテナのコンパイル（本番環境必須）
     *
     * @param string $storagePath コンパイル済みキャッシュの保存先
     */
    public function compile(string $storagePath): void;

    /**
     * コンパイル済みキャッシュの破棄
     */
    public function flush(): void;
}
```

### 5.2 バインディングビルダーインターフェース

```php
<?php

declare(strict_types=1);

namespace Adlaire\Container\Contracts;

interface BindingBuilderInterface
{
    /**
     * タグ付け
     */
    public function tag(string ...$tags): static;

    /**
     * バインディングを確定する
     */
    public function commit(): ContainerInterface;
}
```

### 5.3 条件付きバインディングビルダー

```php
<?php

declare(strict_types=1);

namespace Adlaire\Container\Contracts;

interface ConditionalBindingBuilderInterface
{
    /**
     * 必要とする抽象型を指定
     *
     * @param class-string $abstract
     */
    public function needs(string $abstract): static;

    /**
     * 提供する具体型を指定
     *
     * @param class-string|callable $concrete
     */
    public function give(string|callable $concrete): ContainerInterface;
}
```

### 5.4 スコープ列挙型

```php
<?php

declare(strict_types=1);

namespace Adlaire\Container\Enums;

enum BindingScope
{
    /** アプリケーション全体で1インスタンス */
    case Singleton;

    /** 解決ごとに新インスタンス */
    case Transient;

    /** リクエスト単位で1インスタンス */
    case Scoped;
}
```

### 5.5 例外クラス

```php
<?php

declare(strict_types=1);

namespace Adlaire\Container\Exceptions;

use RuntimeException;

/** バインディング未登録例外（PSR-11 NotFoundExceptionInterface 相当） */
final class NotFoundException extends RuntimeException {}

/** コンテナ一般例外（PSR-11 ContainerExceptionInterface 相当） */
final class ContainerException extends RuntimeException {}

/** 循環依存例外 */
final class CircularDependencyException extends ContainerException {}

/** スコープ違反例外（Scoped サービスをシングルトンに注入しようとした場合） */
final class ScopeMismatchException extends ContainerException {}
```

---

## 6. バインディング規則

### 6.1 必須規則

```
RULE-300-01: すべてのバインディングはインターフェース → 実装の形式で登録すること
RULE-300-02: 具体クラスを直接バインドすること（ConcreteClass → ConcreteClass）は禁止
RULE-300-03: 文字列識別子によるバインディング禁止（class-string のみ許可）
RULE-300-04: グローバル関数 app() / resolve() によるコンテナアクセスはルートレベルのみ許可
RULE-300-05: コンストラクタ外でのコンテナ直接アクセスは ServiceLocator アンチパターンとして禁止
```

### 6.2 バインディング登録場所

```
✅ 許可: ServiceProvider::register() メソッド内
✅ 許可: BootstrapProvider::boot() メソッド内（初期化後処理のみ）
❌ 禁止: Controller, Service, Repository 等のビジネスロジッククラス内
❌ 禁止: グローバルスコープ（ファイルのトップレベル）
```

### 6.3 自動バインディング（オートワイヤリング）禁止

```php
// ❌ 禁止: 自動バインディング（PSR-11 実装でよく見られるパターン）
$container->get(UserService::class); // 登録なしに自動解決 → AFE-STD-300 では例外

// ✅ 必須: 明示的バインディング後に解決
$container->singleton(UserServiceInterface::class, UserService::class);
$service = $container->resolve(UserServiceInterface::class);
```

---

## 7. スコープ管理規則

### 7.1 スコープ選択ガイドライン

```
Singleton（デフォルト推奨）:
  - サービスクラス全般（UserService, OrderService 等）
  - リポジトリ全般（UserRepository 等）
  - インフラ層（Mailer, Logger 等）
  - 状態を持たない（ステートレス）クラス

Transient:
  - バリューオブジェクト（Money, Address 等）
  - コマンドオブジェクト（CreateUserCommand 等）
  - 軽量な生成コストが低いオブジェクト

Scoped:
  - 認証ユーザー情報（AuthenticatedUser 等）
  - リクエストスコープのログコンテキスト
  - DB トランザクションオブジェクト
```

### 7.2 スコープ違反の検出

```php
// ❌ 禁止: Singleton が Scoped を注入（スコープ違反）
// → ScopeMismatchException がスローされる
$container->singleton(UserService::class, function (ContainerInterface $c) {
    return new UserService(
        $c->resolve(AuthenticatedUser::class) // Scoped → ScopeMismatchException
    );
});
```

---

## 8. サービスプロバイダー規則

### 8.1 ServiceProvider インターフェース

```php
<?php

declare(strict_types=1);

namespace Adlaire\Container\Contracts;

interface ServiceProviderInterface
{
    /**
     * バインディング登録フェーズ
     * コンテナへの登録のみを行う（他サービスの解決禁止）
     */
    public function register(ContainerInterface $container): void;

    /**
     * ブートフェーズ（登録完了後に実行）
     * 他サービスの解決可（イベントリスナー登録等）
     */
    public function boot(ContainerInterface $container): void;

    /**
     * このプロバイダーが提供するバインディングの宣言（最適化用）
     *
     * @return array<class-string>
     */
    public function provides(): array;
}
```

### 8.2 ServiceProvider 実装例

```php
<?php

declare(strict_types=1);

namespace Adlaire\Framework\Providers;

use Adlaire\Container\Contracts\ContainerInterface;
use Adlaire\Container\Contracts\ServiceProviderInterface;
use Adlaire\User\Contracts\UserRepositoryInterface;
use Adlaire\User\Infrastructure\EloquentUserRepository;
use Adlaire\User\Contracts\UserServiceInterface;
use Adlaire\User\Services\UserService;

final class UserServiceProvider implements ServiceProviderInterface
{
    public function register(ContainerInterface $container): void
    {
        $container->singleton(UserRepositoryInterface::class, EloquentUserRepository::class);
        $container->singleton(UserServiceInterface::class, UserService::class);
    }

    public function boot(ContainerInterface $container): void
    {
        // ブートフェーズ処理（必要な場合のみ）
    }

    public function provides(): array
    {
        return [
            UserRepositoryInterface::class,
            UserServiceInterface::class,
        ];
    }
}
```

---

## 9. コンテナコンパイル・最適化

### 9.1 コンパイル要件

| 環境 | コンパイル | 備考 |
|------|---------|------|
| 本番（production） | **必須** | デプロイ時に `php artisan container:compile` 実行 |
| ステージング（staging） | **推奨** | 本番相当の検証に必要 |
| 開発（local） | オプション | 開発速度優先のため任意 |
| CI/CD パイプライン | **必須** | コンパイル成功をデプロイ条件とする |

### 9.2 コンパイルコマンド（AFE CLI）

```bash
# コンテナコンパイル
php artisan container:compile

# キャッシュクリア
php artisan container:clear

# バインディング一覧表示（デバッグ用）
php artisan container:list
```

---

## 10. 実装コード例

### 10.1 標準的なバインディング・解決

```php
<?php

declare(strict_types=1);

// --- ServiceProvider での登録 ---
$container->singleton(
    UserRepositoryInterface::class,
    EloquentUserRepository::class
);

$container->singleton(
    UserServiceInterface::class,
    fn(ContainerInterface $c) => new UserService(
        repository: $c->resolve(UserRepositoryInterface::class),
        logger:     $c->resolve(LoggerInterface::class),
    )
);

// --- Controller での利用（コンストラクタ注入のみ許可） ---
final class UserController
{
    public function __construct(
        private readonly UserServiceInterface $userService,
    ) {}

    public function show(int $userId): ResponseInterface
    {
        $user = $this->userService->findById($userId);
        // ...
    }
}
```

### 10.2 タグ付きバインディングの活用

```php
<?php

declare(strict_types=1);

// 登録
$container->singleton(FileLogHandler::class, FileLogHandler::class)->tag('log.handler');
$container->singleton(DatabaseLogHandler::class, DatabaseLogHandler::class)->tag('log.handler');

// 解決
$handlers = $container->resolveTagged('log.handler');
// → array<LogHandlerInterface> [FileLogHandler, DatabaseLogHandler]
```

---

## 11. 違反パターン

### 11.1 禁止パターン一覧

```php
// ❌ VIOLATION-300-01: 具体クラスの直接バインド
$container->singleton(UserService::class, UserService::class);

// ❌ VIOLATION-300-02: 文字列 ID によるバインド
$container->bind('user.service', UserService::class);

// ❌ VIOLATION-300-03: コントローラー内でのコンテナ直接アクセス（Service Locator）
class UserController {
    public function show(): ResponseInterface {
        $service = app()->resolve(UserServiceInterface::class); // 禁止
    }
}

// ❌ VIOLATION-300-04: 未登録の自動解決
$container->resolve(UserService::class); // バインディングなしで解決 → 例外

// ❌ VIOLATION-300-05: PSR-11 互換 get() の使用
$container->get(UserServiceInterface::class); // mixed 戻り値 → 禁止
```

---

## 変更履歴

本ファイルの変更履歴は `../VERSION_HISTORY.md` の `AFE-STD-300` セクションを参照すること。

---

*© 2026 Adlaire Group. All Rights Reserved. 本ドキュメントは Adlaire Group の機密情報です。*
