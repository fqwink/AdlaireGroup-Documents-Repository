# 基本コーディング標準 — 命名規則
## AFE-STD-100

---

| 項目 | 内容 |
|------|------|
| **ドキュメント種別** | 標準仕様書（Standard Specification） |
| **ドキュメントID** | AFE-STD-100 |
| **プロジェクト名** | AFE Standard |
| **バージョン** | Ver.1.0-1 |
| **作成日** | 2026-03-02 |
| **最終更新日** | 2026-03-03 |
| **ステータス** | ✅ 確定（初版） |
| **所有管理組織** | Adlaire Group |
| **機密レベル** | 社外秘 |
| **上位ドキュメント** | AFE-STD-CHARTER-001 |

---

## 目次

1. [概要・適用範囲](#1-概要適用範囲)
2. [PSRとの差別化ポイント（総括）](#2-psrとの差別化ポイント総括)
3. [PHPタグ・ファイル規則](#3-phpタグファイル規則)
4. [型宣言規則（AFE独自）](#4-型宣言規則afe独自)
5. [名前空間規則](#5-名前空間規則)
6. [クラス命名規則](#6-クラス命名規則)
7. [AFEコンポーネントパターン命名規則（AFE独自）](#7-afeコンポーネントパターン命名規則afe独自)
8. [インターフェース・抽象クラス・トレイト命名規則（AFE独自）](#8-インターフェース抽象クラストレイト命名規則afe独自)
9. [例外・列挙型命名規則（AFE独自）](#9-例外列挙型命名規則afe独自)
10. [メソッド・関数命名規則](#10-メソッド関数命名規則)
11. [テストメソッド命名規則（AFE独自）](#11-テストメソッド命名規則afe独自)
12. [プロパティ・変数命名規則](#12-プロパティ変数命名規則)
13. [定数命名規則（AFE独自）](#13-定数命名規則afe独自)
14. [ファイル・ディレクトリ命名規則](#14-ファイルディレクトリ命名規則)
15. [命名規則 早見表](#15-命名規則-早見表)

---

## 1. 概要・適用範囲

### 1.1 概要

AFE-STD-100 は、Adlaire Framework Ecosystem（AFE）およびその関連プロジェクトにおける**基本コーディング標準（命名規則）**を定義する標準仕様書である。

本標準は PHP-FIG PSR への依存を完全に排し、AFE の設計思想（モジュラーコンポーネント駆動・APIインターフェース駆動・DIコンテナ駆動）に最適化された命名規則を規定する。

### 1.2 適用範囲

本標準は以下のすべてに適用される。

- Adlaire Framework Ecosystem（AFE）のソースコード
- AFE を基盤として開発されるすべてのプロジェクト
- Adlaire Group の内部システム

### 1.3 準拠レベル定義

| キーワード | 意味 |
|-----------|------|
| **必須（MUST）** | 例外なく遵守すること ✅ |
| **禁止（MUST NOT）** | 例外なく行わないこと ❌ |
| **推奨（SHOULD）** | 特別な理由がない限り遵守すること |
| **非推奨（SHOULD NOT）** | 特別な理由がない限り行わないこと |

---

## 2. PSRとの差別化ポイント（総括）

PSR（PSR-1 / PSR-4 / PSR-12 相当）と AFE-STD-100 の主要な差別化ポイントを示す。

| 項目 | PSR（参考） | AFE-STD-100 | 差別化レベル |
|------|-----------|------------|------------|
| **インターフェース命名** | 未規定 | `{Name}Interface` サフィックス **必須** | 🔴 AFE独自 |
| **抽象クラス命名** | 未規定 | `Abstract{Name}` プレフィックス **必須** | 🔴 AFE独自 |
| **トレイト命名** | 未規定 | `{Name}Trait` サフィックス **必須** | 🔴 AFE独自 |
| **列挙型命名** | 未規定（PHP 8.1+未対応） | `{Name}Enum` サフィックス **必須** | 🔴 AFE独自 |
| **例外クラス命名** | 未規定 | `{Name}Exception` サフィックス **必須** | 🔴 AFE独自 |
| **AFEコンポーネントパターン** | 未規定 | 専用サフィックス **必須**（7種定義） | 🔴 AFE独自 |
| **strict_types=1 宣言** | 未規定 | 全PHPファイルで **必須** | 🔴 AFE独自 |
| **パラメータ型宣言** | 未規定 | 全パラメータで **必須** | 🔴 AFE独自 |
| **戻り値型宣言** | 未規定 | 全メソッド・関数で **必須** | 🔴 AFE独自 |
| **mixed 型使用** | 未規定 | **禁止**（void/never含む明示型を使用） | 🔴 AFE独自 |
| **define() 定数** | 未規定 | **禁止**（クラス定数・enum のみ許可） | 🔴 AFE独自 |
| **アンダースコアプレフィックス** | SHOULD NOT（推奨） | **禁止**（強制）| 🟡 強化 |
| **テストメソッド命名** | 未規定 | `test{Method}_{Condition}_{Expected}` **必須** | 🔴 AFE独自 |
| **クラス名（StudlyCaps）** | 必須 | 必須（同一） | 🟢 同一 |
| **メソッド名（camelCase）** | 必須 | 必須（同一） | 🟢 同一 |
| **定数（SCREAMING_SNAKE_CASE）** | 必須 | 必須（同一） | 🟢 同一 |

---

## 3. PHPタグ・ファイル規則

### 3.1 PHPタグ

| 規則 | 準拠レベル | 内容 |
|------|----------|------|
| `<?php` タグのみ使用する | **必須** | 短縮タグ `<?` の使用を禁止する |
| テンプレートファイルの `<?=` | **禁止** | テンプレートエンジンを使用し、PHPタグを直接記述しない |
| `?>` 閉じタグ（PHPのみファイル） | **禁止** | PHP のみを含むファイルは閉じタグを省略する |

```php
<?php
// ✅ 正しい: 開きタグのみ、閉じタグなし（PHP onlyファイル）
declare(strict_types=1);

namespace Adlaire\Framework\Core;
```

### 3.2 ファイルエンコーディング

| 規則 | 準拠レベル |
|------|----------|
| UTF-8（BOM なし）を使用する | **必須** |

### 3.3 1ファイル1定義の原則

| 規則 | 準拠レベル | 内容 |
|------|----------|------|
| 1ファイルに定義できるのはクラス・インターフェース・トレイト・列挙型のいずれか1つのみ | **必須** | 複数の定義を1ファイルに混在させてはならない |
| クラス定義ファイルには副作用（出力・設定変更等）を含めない | **必須** | 定義ファイルと副作用処理を分離する |

---

## 4. 型宣言規則（AFE独自）

> ⚠️ このセクションは PSR-1 / PSR-12 には存在しない AFE 独自の規則である。

### 4.1 strict_types 宣言（**PSRとの主要差別化点**）

すべての PHP ファイルの**先頭**（`<?php` タグの直後）に `declare(strict_types=1)` を宣言する。

| 規則 | 準拠レベル |
|------|----------|
| `declare(strict_types=1)` をすべての PHP ファイルに記述する | **必須** |
| `declare(strict_types=0)` の記述 | **禁止** |
| `declare` 文の省略 | **禁止** |

```php
<?php
// ✅ 正しい
declare(strict_types=1);

namespace Adlaire\Framework\Http;

// ❌ 誤り: strict_types 宣言なし
namespace Adlaire\Framework\Http;
```

### 4.2 型宣言（パラメータ・戻り値）（**PSRとの主要差別化点**）

| 規則 | 準拠レベル |
|------|----------|
| すべてのメソッド・関数のパラメータに型宣言を記述する | **必須** |
| すべてのメソッド・関数の戻り値に型宣言を記述する | **必須** |
| `mixed` 型の使用 | **禁止**（Union型・`never`・`void` で代替する） |
| プロパティの型宣言 | **必須** |

```php
<?php
declare(strict_types=1);

// ✅ 正しい: すべてのパラメータと戻り値に型宣言
public function find(int $id): UserInterface
{
    // ...
}

// ✅ 正しい: 複数型は Union 型で表現
public function resolve(string|int $key): ContainerInterface
{
    // ...
}

// ✅ 正しい: 戻り値なし
public function dispatch(EventInterface $event): void
{
    // ...
}

// ❌ 誤り: 型宣言なし
public function find($id)
{
    // ...
}

// ❌ 誤り: mixed 型の使用
public function get(string $key): mixed
{
    // ...
}
```

---

## 5. 名前空間規則

### 5.1 名前空間構造

| 規則 | 準拠レベル | 内容 |
|------|----------|------|
| すべての PHP ファイルに名前空間を宣言する | **必須** | グローバル名前空間のコードを禁止する |
| 名前空間のルートは `Adlaire\` とする | **必須** | AFE コアは `Adlaire\Framework\` を使用する |
| 名前空間はディレクトリ構造と完全に一致させる | **必須** | AFE-STD-101 に従う |

```php
<?php
declare(strict_types=1);

// ✅ 正しい: 名前空間あり
namespace Adlaire\Framework\Http\Message;

// ❌ 誤り: グローバル名前空間
class Request { }
```

### 5.2 名前空間の命名

| 対象 | 規則 | 例 |
|------|------|----|
| 名前空間セグメント | StudlyCaps | `Adlaire\Framework\Http\Message` |
| ルート名前空間 | `Adlaire` 固定 | — |
| AFE コア | `Adlaire\Framework\` | `Adlaire\Framework\Routing` |

---

## 6. クラス命名規則

### 6.1 通常クラス

| 規則 | 準拠レベル | 内容 |
|------|----------|------|
| **StudlyCaps**（PascalCase）を使用する | **必須** | 単語の先頭を大文字にした連結形式 |
| クラス名はその責務を明確に表す名詞または名詞句とする | **必須** | 動詞のみのクラス名は禁止 |
| クラス名とファイル名を完全に一致させる | **必須** | `UserRepository` → `UserRepository.php` |

```php
// ✅ 正しい
class UserRepository { }
class HttpRequest { }
class DatabaseConnection { }

// ❌ 誤り: 小文字・スネークケース
class user_repository { }
class httprequest { }
```

---

## 7. AFEコンポーネントパターン命名規則（AFE独自）

> ⚠️ このセクションは PSR に存在しない AFE 独自の規則である。

AFE のアーキテクチャを構成する主要コンポーネントは、以下の専用サフィックスを**必須**とする。
サフィックスにより、ファイルを見ただけでコンポーネントの役割が即座に判別できる。

| コンポーネント種別 | サフィックス | 例 | 禁止例 |
|----------------|------------|-----|-------|
| **サービスプロバイダー** | `ServiceProvider` | `AuthServiceProvider` | `AuthProvider`, `AuthService` |
| **ミドルウェア** | `Middleware` | `CsrfMiddleware` | `Csrf`, `CsrfCheck` |
| **コントローラー** | `Controller` | `UserController` | `User`, `UserHandler` |
| **リポジトリ** | `Repository` | `UserRepository` | `UserRepo`, `UserStorage` |
| **コマンド（CLI）** | `Command` | `MigrateCommand` | `Migrate`, `MigrateAction` |
| **イベント** | `Event` | `UserCreatedEvent` | `UserCreated`, `OnUserCreate` |
| **イベントリスナー** | `Listener` | `SendWelcomeMailListener` | `WelcomeMailHandler`, `OnUserCreated` |

```php
// ✅ 正しい
class AuthServiceProvider extends AbstractServiceProvider { }
class CsrfMiddleware implements MiddlewareInterface { }
class UserController extends AbstractController { }
class UserRepository implements UserRepositoryInterface { }
class MigrateCommand extends AbstractCommand { }
class UserCreatedEvent implements EventInterface { }
class SendWelcomeMailListener implements ListenerInterface { }

// ❌ 誤り: サフィックスなし・誤ったサフィックス
class Auth { }
class CsrfCheck { }
class UserControl { }
```

---

## 8. インターフェース・抽象クラス・トレイト命名規則（AFE独自）

> ⚠️ このセクションは PSR-1 に存在しない AFE 独自の規則である。

### 8.1 インターフェース（**PSRとの主要差別化点**）

| 規則 | 準拠レベル |
|------|----------|
| `{名詞/名詞句}Interface` のサフィックスを付ける | **必須** |
| `I{Name}` や `{Name}able` 形式 | **禁止** |

```php
// ✅ 正しい
interface ContainerInterface { }
interface LoggerInterface { }
interface UserRepositoryInterface { }
interface HttpRequestInterface { }

// ❌ 誤り: サフィックスなし・異なる命名
interface IContainer { }
interface Loggable { }
interface Container { }
```

### 8.2 抽象クラス（**PSRとの主要差別化点**）

| 規則 | 準拠レベル |
|------|----------|
| `Abstract{名詞/名詞句}` のプレフィックスを付ける | **必須** |
| `{Name}Abstract` や `Base{Name}` 形式 | **禁止** |

```php
// ✅ 正しい
abstract class AbstractController { }
abstract class AbstractRepository { }
abstract class AbstractCommand { }
abstract class AbstractServiceProvider { }

// ❌ 誤り: プレフィックスなし・異なる命名
abstract class Controller { }
abstract class BaseController { }
abstract class ControllerAbstract { }
```

### 8.3 トレイト（**PSRとの主要差別化点**）

| 規則 | 準拠レベル |
|------|----------|
| `{名詞/形容詞}Trait` のサフィックスを付ける | **必須** |
| `{Name}` のみ（サフィックスなし）・`Has{Name}` 形式 | **禁止** |

```php
// ✅ 正しい
trait TimestampableTrait { }
trait SoftDeletableTrait { }
trait SingletonTrait { }

// ❌ 誤り: サフィックスなし
trait Timestampable { }
trait HasTimestamp { }
```

---

## 9. 例外・列挙型命名規則（AFE独自）

### 9.1 例外クラス（**PSRとの主要差別化点**）

| 規則 | 準拠レベル |
|------|----------|
| `{名詞/動詞句}Exception` のサフィックスを付ける | **必須** |
| `{Name}Error` や `{Name}Fault` 形式 | **禁止** |
| すべての例外クラスは `\Exception` または AFE 基底例外クラスを継承する | **必須** |

```php
// ✅ 正しい
class NotFoundException extends AbstractAfeException { }
class InvalidArgumentException extends AbstractAfeException { }
class ContainerBindingException extends AbstractAfeException { }
class HttpRequestException extends AbstractAfeException { }

// ❌ 誤り
class NotFound extends \Exception { }
class InvalidArgumentError extends \Exception { }
```

### 9.2 列挙型（Enum）（**PSRとの主要差別化点** — PHP 8.1+）

PSR-1 は PHP 8.1 の Enum に関する規定を持たない。AFE-STD-100 は以下を定義する。

| 規則 | 準拠レベル |
|------|----------|
| `{名詞}Enum` のサフィックスを付ける | **必須** |
| Enum のケース名は `SCREAMING_SNAKE_CASE` を使用する | **必須** |
| Enum のケース名に `StudlyCaps` | **禁止** |

```php
// ✅ 正しい
enum HttpMethodEnum: string
{
    case GET = 'GET';
    case POST = 'POST';
    case PUT = 'PUT';
    case PATCH = 'PATCH';
    case DELETE = 'DELETE';
}

enum LogLevelEnum: int
{
    case DEBUG = 100;
    case INFO = 200;
    case WARNING = 300;
    case ERROR = 400;
    case CRITICAL = 500;
}

// ❌ 誤り: サフィックスなし・ケース名が StudlyCaps
enum HttpMethod: string
{
    case Get = 'GET';   // ❌ StudlyCaps 禁止
    case Post = 'POST'; // ❌ StudlyCaps 禁止
}
```

---

## 10. メソッド・関数命名規則

### 10.1 メソッド命名

| 規則 | 準拠レベル | 内容 |
|------|----------|------|
| `camelCase` を使用する | **必須** | 最初の単語は小文字、以降の単語の先頭を大文字 |
| メソッド名はその動作を明確に表す動詞または動詞句とする | **必須** | 名詞のみのメソッド名は禁止 |
| アンダースコアプレフィックス（`_methodName`） | **禁止** | アクセス修飾子（`private` / `protected`）で可視性を制御する |

```php
// ✅ 正しい
public function findById(int $id): UserInterface { }
public function createUser(array $data): UserInterface { }
private function hashPassword(string $plain): string { }
protected function validateInput(array $input): bool { }

// ❌ 誤り: アンダースコアプレフィックス
protected function _validate(array $input): bool { }
private function __hashPassword(string $plain): string { }
```

### 10.2 アクセサ・ミューテータ命名

| 種別 | 規則 | 例 |
|------|------|----|
| **Getter** | `get{PropertyName}()` | `getUserId()`, `getCreatedAt()` |
| **Setter** | `set{PropertyName}()` | `setUserId()`, `setStatus()` |
| **真偽チェック** | `is{Condition}()` または `has{Property}()` | `isActive()`, `hasPermission()` |

```php
// ✅ 正しい
public function getUserId(): int { }
public function setStatus(string $status): void { }
public function isActive(): bool { }
public function hasPermission(string $permission): bool { }

// ❌ 誤り
public function userId(): int { }    // get プレフィックスなし
public function active(): bool { }  // is/has プレフィックスなし
```

---

## 11. テストメソッド命名規則（AFE独自）

> ⚠️ このセクションは PSR に存在しない AFE 独自の規則である。

### 11.1 テストクラス命名

| 規則 | 準拠レベル | 例 |
|------|----------|----|
| `{テスト対象クラス名}Test` のサフィックスを付ける | **必須** | `UserRepositoryTest` |
| テストクラスのファイルは `{ClassName}Test.php` | **必須** | `UserRepositoryTest.php` |

### 11.2 テストメソッド命名（**PSRとの主要差別化点**）

| 規則 | 準拠レベル | 形式 |
|------|----------|------|
| `test{対象メソッド}_{条件}_{期待結果}` 形式を使用する | **必須** | スネークケースで連結 |
| `test` プレフィックスなしのメソッド | **禁止** | PHPUnit アノテーション `@test` による省略も禁止 |

```php
// ✅ 正しい
public function testFindById_WithValidId_ReturnsUser(): void { }
public function testFindById_WithNonExistentId_ThrowsNotFoundException(): void { }
public function testCreateUser_WithValidData_PersistsAndReturnsUser(): void { }
public function testCreateUser_WithDuplicateEmail_ThrowsValidationException(): void { }
public function testIsActive_WhenStatusIsActive_ReturnsTrue(): void { }
public function testIsActive_WhenStatusIsBanned_ReturnsFalse(): void { }

// ❌ 誤り: 形式不一致・プレフィックスなし
public function findById_returns_user(): void { }       // test プレフィックスなし
public function testUserCreation(): void { }            // 条件・期待結果なし
/** @test */
public function user_can_be_found_by_id(): void { }    // @test アノテーション禁止
```

---

## 12. プロパティ・変数命名規則

### 12.1 プロパティ命名

| 規則 | 準拠レベル | 内容 |
|------|----------|------|
| `camelCase` を使用する | **必須** | — |
| アンダースコアプレフィックス（`_property`） | **禁止** | アクセス修飾子で可視性を制御する |
| `readonly` プロパティの積極的活用 | **推奨** | 値オブジェクト・DTOには `readonly` を使用する |

```php
// ✅ 正しい
private int $userId;
protected string $userName;
public readonly string $email;

// ❌ 誤り: アンダースコアプレフィックス
private int $_userId;
protected string $_userName;
```

### 12.2 変数命名

| 規則 | 準拠レベル | 内容 |
|------|----------|------|
| `camelCase` を使用する | **必須** | — |
| 1文字変数名（ループ変数 `$i`, `$j`, `$k` を除く） | **非推奨** | 意味のある名前をつける |
| 略語・暗号的な変数名 | **禁止** | `$u` より `$user`、`$tmp` より `$temporaryValue` |

```php
// ✅ 正しい
$userId = 1;
$currentUser = $this->userRepository->findById($userId);
$isAuthenticated = $currentUser->isActive();

foreach ($items as $i => $item) { }  // ループ変数 $i は許可

// ❌ 誤り
$u = 1;
$cu = $this->userRepository->findById($u);
$tmp = $cu->isActive();
```

---

## 13. 定数命名規則（AFE独自）

### 13.1 クラス定数

| 規則 | 準拠レベル |
|------|----------|
| `SCREAMING_SNAKE_CASE`（全大文字・アンダースコア区切り）を使用する | **必須** |
| 定数名は意味のある名詞・形容詞で表す | **必須** |

```php
// ✅ 正しい
class HttpStatusCode
{
    public const int OK = 200;
    public const int NOT_FOUND = 404;
    public const int INTERNAL_SERVER_ERROR = 500;
    public const string DEFAULT_CHARSET = 'UTF-8';
}
```

### 13.2 グローバル定数（**PSRとの主要差別化点**）

| 規則 | 準拠レベル | 内容 |
|------|----------|------|
| `define()` によるグローバル定数の定義 | **禁止** | クラス定数または Enum を使用する |
| `const` によるグローバル定数（名前空間内） | **禁止** | クラス定数または Enum を使用する |

```php
// ✅ 正しい: クラス定数を使用
class AppConfig
{
    public const string VERSION = '1.0.0';
    public const int MAX_RETRY = 3;
}

// ✅ 正しい: Enum を使用（PHP 8.1+）
enum EnvironmentEnum: string
{
    case PRODUCTION = 'production';
    case STAGING = 'staging';
    case DEVELOPMENT = 'development';
}

// ❌ 禁止: define() によるグローバル定数
define('APP_VERSION', '1.0.0');
define('MAX_RETRY', 3);

// ❌ 禁止: グローバル const
const APP_VERSION = '1.0.0';
```

---

## 14. ファイル・ディレクトリ命名規則

### 14.1 PHP クラスファイル

| 規則 | 準拠レベル | 例 |
|------|----------|----|
| ファイル名はクラス名・インターフェース名・トレイト名・Enum名と**完全一致**させる | **必須** | `UserRepository.php` |
| 拡張子は `.php` | **必須** | — |
| 1ファイル1定義 | **必須** | セクション 3.3 参照 |

```
✅ 正しい
UserRepository.php       → class UserRepository
UserRepositoryInterface.php → interface UserRepositoryInterface
AbstractRepository.php   → abstract class AbstractRepository
TimestampableTrait.php   → trait TimestampableTrait
HttpMethodEnum.php       → enum HttpMethodEnum
UserController.php       → class UserController

❌ 誤り
user_repository.php      → ファイル名がスネークケース
userrepository.php       → ファイル名が全小文字
```

### 14.2 ディレクトリ命名

| 規則 | 準拠レベル | 内容 |
|------|----------|------|
| ディレクトリ名は **StudlyCaps** を使用する | **必須** | 名前空間セグメントと一致させる（AFE-STD-101 参照） |
| スネークケース・ケバブケースのディレクトリ名 | **禁止** | — |

```
✅ 正しい
src/
  Http/
    Message/
      Request.php          → namespace Adlaire\Framework\Http\Message
    Middleware/
  Database/
    Orm/
  Auth/
    Rbac/

❌ 誤り
src/
  http/                    → 小文字
  http_message/            → スネークケース
  http-message/            → ケバブケース
```

---

## 15. 命名規則 早見表

| 対象 | 命名形式 | 例 |
|------|---------|-----|
| **クラス** | StudlyCaps | `UserRepository` |
| **インターフェース** | `{Name}Interface` | `UserRepositoryInterface` |
| **抽象クラス** | `Abstract{Name}` | `AbstractRepository` |
| **トレイト** | `{Name}Trait` | `TimestampableTrait` |
| **例外クラス** | `{Name}Exception` | `NotFoundException` |
| **列挙型** | `{Name}Enum` | `HttpMethodEnum` |
| **Enumケース** | SCREAMING_SNAKE_CASE | `NOT_FOUND`, `HTTP_OK` |
| **ServiceProvider** | `{Name}ServiceProvider` | `AuthServiceProvider` |
| **Middleware** | `{Name}Middleware` | `CsrfMiddleware` |
| **Controller** | `{Name}Controller` | `UserController` |
| **Repository** | `{Name}Repository` | `UserRepository` |
| **Command** | `{Name}Command` | `MigrateCommand` |
| **Event** | `{Name}Event` | `UserCreatedEvent` |
| **Listener** | `{Name}Listener` | `SendWelcomeMailListener` |
| **メソッド** | camelCase | `findById`, `createUser` |
| **テストメソッド** | `test{Method}_{Condition}_{Expected}` | `testFindById_WithValidId_ReturnsUser` |
| **プロパティ** | camelCase | `userId`, `createdAt` |
| **変数** | camelCase | `currentUser`, `isActive` |
| **クラス定数** | SCREAMING_SNAKE_CASE | `NOT_FOUND`, `DEFAULT_CHARSET` |
| **ファイル名（クラス）** | クラス名と完全一致 | `UserRepository.php` |
| **ディレクトリ名** | StudlyCaps | `Http`, `Database` |

---

> 📖 変更履歴は [`../VERSION_HISTORY.md`](../VERSION_HISTORY.md) を参照すること。
> 📖 オートローディング規則は [`AFE-STD-101`](./AFE-STD-101.md) を参照すること。
> 📖 コーディングスタイル規則は [`AFE-STD-102`](./AFE-STD-102.md) を参照すること。

---

*本ドキュメントは Adlaire Group の機密情報です。Adlaire Group の書面による許可なく、外部への開示・複製・転用を禁止します。*

*© 2026 Adlaire Group / Adlaire Group DX Division. All Rights Reserved.*
