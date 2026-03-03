# AFE-STD-602: バリデーション標準

| 項目 | 内容 |
|------|------|
| **ドキュメントID** | AFE-STD-602 |
| **タイトル** | バリデーション標準 |
| **バージョン** | Ver.1.0-1 |
| **作成日** | 2026-03-02 |
| **最終更新日** | 2026-03-03 |
| **ステータス** | ✅ 確定（初版） |
| **PSR対応** | 独自仕様（旧 AFE-STD-003 後継） |
| **関連標準** | AFE-STD-100, AFE-STD-200, AFE-STD-300 |

---

## 1. 概要

AFE-STD-602 は AFE における **入力バリデーションの標準インターフェース仕様**を定義する。PSR には対応する規格がないため、AFE 独自仕様（旧 AFE-STD-003 後継）として策定する。

### 設計方針

- **型付きリクエストオブジェクト（FormRequest）**: バリデーション済み入力を型安全に扱う
- **宣言的バリデーションルール**: ルールをオブジェクトとして定義し再利用可能にする
- **ドメインバリデーション分離**: 入力バリデーション（フォーム層）とドメインバリデーション（ビジネスルール）を明確に分離
- **エラーメッセージの i18n 対応**: 多言語対応を標準化

---

## 2. インターフェース定義

### 2.1 バリデーターインターフェース

```php
<?php

declare(strict_types=1);

namespace Adlaire\Validation\Contracts;

/**
 * AFE-STD-602 準拠 バリデーターインターフェース
 */
interface ValidatorInterface
{
    /**
     * バリデーション実行
     *
     * @param array<string, mixed> $data
     * @param array<string, array<RuleInterface>|RuleInterface> $rules
     * @return ValidationResultInterface
     */
    public function validate(array $data, array $rules): ValidationResultInterface;

    /**
     * バリデーション実行（失敗時に例外）
     *
     * @throws ValidationException
     */
    public function validateOrFail(array $data, array $rules): ValidatedDataInterface;
}
```

### 2.2 バリデーション結果インターフェース

```php
<?php

declare(strict_types=1);

namespace Adlaire\Validation\Contracts;

interface ValidationResultInterface
{
    public function passes(): bool;
    public function fails(): bool;

    /**
     * @return array<string, array<string>>
     */
    public function errors(): array;

    /**
     * @return array<string>
     */
    public function errorsFor(string $field): array;

    public function firstError(string $field): ?string;

    /**
     * バリデーション済みデータ（パスした場合のみ）
     *
     * @throws ValidationException fails() の場合
     */
    public function validated(): ValidatedDataInterface;
}
```

### 2.3 バリデーション済みデータインターフェース

```php
<?php

declare(strict_types=1);

namespace Adlaire\Validation\Contracts;

/**
 * バリデーション通過済みの型安全データコンテナ
 */
interface ValidatedDataInterface
{
    public function getString(string $key): string;
    public function getInt(string $key): int;
    public function getFloat(string $key): float;
    public function getBool(string $key): bool;

    /**
     * @return array<mixed>
     */
    public function getArray(string $key): array;

    public function has(string $key): bool;
    public function only(string ...$keys): static;
    public function except(string ...$keys): static;

    /** @return array<string, mixed> */
    public function toArray(): array;
}
```

### 2.4 バリデーションルールインターフェース

```php
<?php

declare(strict_types=1);

namespace Adlaire\Validation\Contracts;

/**
 * バリデーションルールの基底インターフェース
 * 各ルールはオブジェクトとして定義し再利用する
 */
interface RuleInterface
{
    /**
     * バリデーション実行
     *
     * @param string $field フィールド名
     * @param mixed $value バリデーション対象の値
     * @param array<string, mixed> $allData 全フィールドデータ（相互依存ルール用）
     */
    public function passes(string $field, mixed $value, array $allData): bool;

    /**
     * エラーメッセージ（i18n キー形式推奨: validation.required 等）
     */
    public function message(): string;
}
```

### 2.5 型付きリクエストオブジェクト（FormRequest）

```php
<?php

declare(strict_types=1);

namespace Adlaire\Validation\Contracts;

use Adlaire\Auth\Contracts\AuthenticatedUserInterface;
use Adlaire\Http\Contracts\ServerRequestInterface;

/**
 * バリデーション済み・型安全な入力を持つリクエストオブジェクト
 * Controller に注入され、バリデーション済み入力のみを提供する
 */
abstract class FormRequest
{
    private ValidatedDataInterface $validated;

    final public function __construct(
        protected readonly ServerRequestInterface $request,
        private readonly ValidatorInterface $validator,
        protected readonly ?AuthenticatedUserInterface $user = null,
    ) {
        $result = $this->validator->validate(
            $this->resolveInput(),
            $this->rules()
        );

        if ($result->fails()) {
            throw new ValidationException($result->errors());
        }

        $this->validated = $result->validated();
    }

    /**
     * バリデーションルールの定義（サブクラスで必須実装）
     *
     * @return array<string, array<RuleInterface>|RuleInterface>
     */
    abstract public function rules(): array;

    /**
     * 認可チェック（デフォルト: 認証済みユーザーに許可）
     */
    public function authorize(): bool
    {
        return $this->user !== null;
    }

    /**
     * バリデーション済みデータ取得
     */
    final public function validated(): ValidatedDataInterface
    {
        return $this->validated;
    }

    /** @return array<string, mixed> */
    protected function resolveInput(): array
    {
        return array_merge(
            $this->request->getQueryParams(),
            (array) $this->request->getParsedBody()
        );
    }
}
```

---

## 3. 標準バリデーションルール

AFE は以下の標準ルールを提供する（名前空間: `Adlaire\Validation\Rules\`）：

| ルールクラス | 説明 | PSR 類似 |
|------------|------|---------|
| `RequiredRule` | 必須フィールド | なし |
| `StringRule` | 文字列型 | なし |
| `IntegerRule` | 整数型 | なし |
| `FloatRule` | 浮動小数点型 | なし |
| `BooleanRule` | 真偽値型 | なし |
| `EmailRule` | メールアドレス形式 | なし |
| `UrlRule` | URL 形式 | なし |
| `MinLengthRule` | 最小文字数 | なし |
| `MaxLengthRule` | 最大文字数 | なし |
| `MinValueRule` | 最小値 | なし |
| `MaxValueRule` | 最大値 | なし |
| `RegexRule` | 正規表現 | なし |
| `InRule` | 許可値リスト | なし |
| `NotInRule` | 禁止値リスト | なし |
| `UniqueRule` | DB ユニーク確認 | なし |
| `ExistsRule` | DB 存在確認 | なし |
| `DateRule` | 日付形式 | なし |
| `AfterDateRule` | 指定日以降 | なし |
| `BeforeDateRule` | 指定日以前 | なし |
| `ConfirmedRule` | 確認フィールド一致 | なし |
| `NullableRule` | null 許容 | なし |

---

## 4. 実装規則

```
RULE-602-01: リクエスト入力のバリデーションは必ず FormRequest を通じて行うこと
RULE-602-02: Controller でのバリデーション手動実行を禁止する（FormRequest 注入を使用）
RULE-602-03: ドメインバリデーション（ビジネスルール違反）は例外を使用すること（ValidatorInterface ではなく DomainException 系）
RULE-602-04: バリデーションエラーメッセージは i18n キー形式で定義すること（ハードコード禁止）
RULE-602-05: パスワード入力フィールドはバリデーション後に即座にハッシュ化し、平文を保持しないこと
RULE-602-06: カスタムルールは RuleInterface を実装し、再利用可能なクラスとして定義すること
```

---

## 5. 実装コード例

```php
<?php

declare(strict_types=1);

namespace Adlaire\App\Http\Requests;

use Adlaire\Validation\Contracts\FormRequest;
use Adlaire\Validation\Rules\RequiredRule;
use Adlaire\Validation\Rules\EmailRule;
use Adlaire\Validation\Rules\MinLengthRule;
use Adlaire\Validation\Rules\MaxLengthRule;
use Adlaire\Validation\Rules\UniqueRule;

final class CreateUserRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'name'  => [new RequiredRule(), new MinLengthRule(2), new MaxLengthRule(50)],
            'email' => [new RequiredRule(), new EmailRule(), new UniqueRule('users', 'email')],
            'password' => [new RequiredRule(), new MinLengthRule(8)],
        ];
    }

    public function authorize(): bool
    {
        return true; // 登録は認証不要
    }
}

// Controller での使用
final class UserController
{
    public function store(CreateUserRequest $request): ResponseInterface
    {
        // バリデーション済みの型安全データ
        $command = new CreateUserCommand(
            name:     $request->validated()->getString('name'),
            email:    $request->validated()->getString('email'),
            password: $request->validated()->getString('password'),
        );

        $user = $this->userService->create($command);

        return ResponseInterface::json(['id' => $user->id->value()], HttpStatusCode::Created);
    }
}
```

---

## 変更履歴

本ファイルの変更履歴は `../VERSION_HISTORY.md` の `AFE-STD-602` セクションを参照すること。

---

*© 2026 Adlaire Group. All Rights Reserved.*
