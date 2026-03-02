# AFE-STD-601: 認証・認可（RBAC）標準

| 項目 | 内容 |
|------|------|
| **ドキュメントID** | AFE-STD-601 |
| **タイトル** | 認証・認可（RBAC）標準 |
| **バージョン** | Ver.1.0.0 |
| **作成日** | 2026-03-02 |
| **最終更新日** | 2026-03-02 |
| **ステータス** | ✅ 初版発行 |
| **PSR対応** | 独自仕様（旧 AFE-STD-002 後継） |
| **関連標準** | AFE-STD-100, AFE-STD-200, AFE-STD-300, AFE-STD-400 |

---

## 1. 概要

AFE-STD-601 は AFE における **認証（Authentication）および認可（Authorization / RBAC）の標準インターフェース仕様**を定義する。PSR には対応する規格がないため、AFE 独自仕様（旧 AFE-STD-002 後継）として策定する。

### 設計方針

- **認証・認可の明確な分離**: 認証（誰か）と認可（何ができるか）を完全に分離
- **RBAC（ロールベースアクセス制御）**: ロール・パーミッション・ポリシーの3層構造
- **認証ドライバー抽象化**: Session/JWT/API Key/OAuth2 を統一インターフェースで管理
- **ゲートウェイパターン**: PolicyInterface による宣言的アクセス制御

---

## 2. インターフェース定義

### 2.1 認証マネージャー

```php
<?php

declare(strict_types=1);

namespace Adlaire\Auth\Contracts;

/**
 * AFE-STD-601 準拠 認証マネージャーインターフェース
 */
interface AuthManagerInterface
{
    /**
     * 認証情報から認証済みユーザーを取得
     *
     * @throws AuthenticationException 認証失敗時
     */
    public function authenticate(CredentialsInterface $credentials): AuthenticatedUserInterface;

    /**
     * 認証済みユーザーの取得
     *
     * @throws UnauthenticatedException 未認証時
     */
    public function user(): AuthenticatedUserInterface;

    /**
     * 認証済みかどうか
     */
    public function check(): bool;

    /**
     * ゲスト（未認証）かどうか
     */
    public function guest(): bool;

    /**
     * ログアウト
     */
    public function logout(): void;

    /**
     * 認証ドライバーの切り替え
     */
    public function guard(string $driver): static;
}
```

### 2.2 認証済みユーザーインターフェース

```php
<?php

declare(strict_types=1);

namespace Adlaire\Auth\Contracts;

/**
 * 認証済みユーザーを表すインターフェース（イミュータブル）
 */
interface AuthenticatedUserInterface
{
    public function getId(): int|string;
    public function getIdentifier(): string; // メール・ユーザー名等

    /**
     * このユーザーに付与されたロール一覧
     *
     * @return array<RoleInterface>
     */
    public function getRoles(): array;

    /**
     * ロールを持つか確認
     */
    public function hasRole(string $roleName): bool;

    /**
     * パーミッションを持つか確認（ロール経由のパーミッション含む）
     */
    public function hasPermission(string $permission): bool;

    /**
     * 複数ロールのいずれかを持つか確認
     */
    public function hasAnyRole(string ...$roleNames): bool;

    /**
     * すべてのロールを持つか確認
     */
    public function hasAllRoles(string ...$roleNames): bool;
}
```

### 2.3 認可ゲートインターフェース（ポリシー管理）

```php
<?php

declare(strict_types=1);

namespace Adlaire\Auth\Contracts;

/**
 * AFE-STD-601 準拠 認可ゲートインターフェース
 * PolicyInterface との連携で宣言的アクセス制御を実現
 */
interface AuthorizationGateInterface
{
    /**
     * アクションの実行可否確認
     *
     * @param string $ability 例: 'view', 'update', 'delete'
     * @param object|null $resource 対象リソース（ポリシー対象）
     */
    public function allows(string $ability, ?object $resource = null): bool;

    /**
     * アクションの実行可否確認（不可の場合は例外）
     *
     * @throws AuthorizationException
     */
    public function authorize(string $ability, ?object $resource = null): void;

    /**
     * ポリシーの登録
     *
     * @param class-string $resourceClass
     * @param class-string<PolicyInterface> $policyClass
     */
    public function policy(string $resourceClass, string $policyClass): void;

    /**
     * クロージャによるアビリティの直接定義
     */
    public function define(string $ability, callable $callback): void;
}
```

### 2.4 ポリシーインターフェース

```php
<?php

declare(strict_types=1);

namespace Adlaire\Auth\Contracts;

/**
 * リソース別のアクセス制御ポリシー
 */
interface PolicyInterface
{
    /**
     * 認可前フック（全アビリティより前に実行）
     * null を返した場合は通常の認可フローに進む
     */
    public function before(AuthenticatedUserInterface $user, string $ability): ?bool;
}
```

### 2.5 RBAC ロールインターフェース

```php
<?php

declare(strict_types=1);

namespace Adlaire\Auth\Contracts;

interface RoleInterface
{
    public function getName(): string;
    public function getDisplayName(): string;

    /**
     * このロールが持つパーミッション一覧
     *
     * @return array<PermissionInterface>
     */
    public function getPermissions(): array;

    public function hasPermission(string $permission): bool;

    /**
     * ロールの継承（親ロールのパーミッションを引き継ぐ）
     *
     * @return array<RoleInterface>
     */
    public function getParentRoles(): array;
}
```

### 2.6 認証ドライバーインターフェース

```php
<?php

declare(strict_types=1);

namespace Adlaire\Auth\Contracts;

use Adlaire\Http\Contracts\ServerRequestInterface;

/**
 * 認証ドライバー抽象インターフェース
 * Session / JWT / API Key / OAuth2 を統一管理
 */
interface AuthDriverInterface
{
    /**
     * リクエストから認証情報を抽出・検証
     */
    public function authenticate(ServerRequestInterface $request): ?AuthenticatedUserInterface;

    /**
     * 認証トークン生成
     */
    public function generateToken(AuthenticatedUserInterface $user): TokenInterface;

    /**
     * トークン失効
     */
    public function revokeToken(TokenInterface $token): void;
}
```

---

## 3. 実装規則

```
RULE-601-01: 認証チェックはすべて AuthManagerInterface::check() 経由で行うこと
RULE-601-02: $_SESSION への直接アクセスを禁止する（セッションドライバー経由のみ許可）
RULE-601-03: パスワードは bcrypt または Argon2id でハッシュ化し、平文での保存・ログ出力を絶対禁止する
RULE-601-04: JWT の秘密鍵は環境変数で管理し、ソースコードへの埋め込みを絶対禁止する
RULE-601-05: 認可は必ず PolicyInterface または AuthorizationGateInterface を通じて行うこと
RULE-601-06: コントローラー内で直接 $user->role === 'admin' のような文字列比較を行うことを禁止する
RULE-601-07: RBAC の権限変更はログに記録すること（AFE-STD-400 経由）
RULE-601-08: セッション固定攻撃対策として認証成功時に必ずセッション ID を再生成すること
```

---

## 4. 実装コード例

### 4.1 ポリシーの定義

```php
<?php

declare(strict_types=1);

namespace Adlaire\App\Policies;

use Adlaire\Auth\Contracts\AuthenticatedUserInterface;
use Adlaire\Auth\Contracts\PolicyInterface;
use Adlaire\Order\Domain\Entities\Order;

final class OrderPolicy implements PolicyInterface
{
    public function before(AuthenticatedUserInterface $user, string $ability): ?bool
    {
        // 管理者はすべてを許可
        if ($user->hasRole('admin')) {
            return true;
        }
        return null; // 通常フローへ
    }

    public function view(AuthenticatedUserInterface $user, Order $order): bool
    {
        return $user->getId() === $order->getUserId();
    }

    public function update(AuthenticatedUserInterface $user, Order $order): bool
    {
        return $user->getId() === $order->getUserId()
            && $order->isEditable();
    }

    public function delete(AuthenticatedUserInterface $user, Order $order): bool
    {
        return $user->hasPermission('order.delete');
    }
}
```

### 4.2 コントローラーでの使用

```php
<?php

declare(strict_types=1);

final class OrderController
{
    public function __construct(
        private readonly AuthorizationGateInterface $gate,
        private readonly OrderServiceInterface $orderService,
    ) {}

    public function update(ServerRequestInterface $request, int $orderId): ResponseInterface
    {
        $order = $this->orderService->findById($orderId);

        // 認可チェック（不可の場合は AuthorizationException）
        $this->gate->authorize('update', $order);

        // 処理続行...
    }
}
```

---

## 変更履歴

本ファイルの変更履歴は `../VERSION_HISTORY.md` の `AFE-STD-601` セクションを参照すること。

---

*© 2026 Adlaire Group. All Rights Reserved.*
