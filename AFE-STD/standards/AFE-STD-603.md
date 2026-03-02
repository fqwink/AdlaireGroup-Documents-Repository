# AFE-STD-603: CLI コマンドインターフェース規約

| 項目 | 内容 |
|------|------|
| **ドキュメントID** | AFE-STD-603 |
| **タイトル** | CLI コマンドインターフェース規約 |
| **バージョン** | Ver.1.0.0 |
| **作成日** | 2026-03-02 |
| **最終更新日** | 2026-03-02 |
| **ステータス** | ✅ 初版発行 |
| **PSR対応** | 独自仕様（旧 AFE-STD-004 後継） |
| **関連標準** | AFE-STD-100, AFE-STD-300, AFE-STD-400 |

---

## 1. 概要

AFE-STD-603 は AFE における **CLI（Command Line Interface）コマンドの標準インターフェース仕様および命名規約**を定義する。PSR には対応する規格がないため、AFE 独自仕様（旧 AFE-STD-004 後継）として策定する。

### 設計方針

- **コマンドパターン必須**: すべての CLI 処理は CommandInterface を実装したクラスとして定義
- **コンソール入出力の抽象化**: IO 操作を ConsoleIOInterface で抽象化しテスト可能にする
- **冪等性の保証**: デプロイ・マイグレーション系コマンドは必ず冪等に実装する
- **終了コードの標準化**: 成功/失敗/警告の終了コードを統一する

---

## 2. インターフェース定義

### 2.1 コマンドインターフェース

```php
<?php

declare(strict_types=1);

namespace Adlaire\Console\Contracts;

/**
 * AFE-STD-603 準拠 CLI コマンドインターフェース
 */
interface CommandInterface
{
    /**
     * コマンドの識別名（例: user:create, cache:clear）
     * 形式: {コンテキスト}:{動詞} （必須）
     */
    public static function getName(): string;

    /**
     * コマンドの説明文
     */
    public static function getDescription(): string;

    /**
     * コマンド引数・オプションの定義
     *
     * @return array<ArgumentDefinitionInterface|OptionDefinitionInterface>
     */
    public static function getDefinition(): array;

    /**
     * コマンド実行
     *
     * @return ExitCode
     */
    public function execute(ConsoleIOInterface $io, ParsedInputInterface $input): ExitCode;
}
```

### 2.2 終了コード Enum

```php
<?php

declare(strict_types=1);

namespace Adlaire\Console\Enums;

enum ExitCode: int
{
    case Success = 0;
    case Failure = 1;
    case Warning = 2;
    case InvalidArgument = 64;  // BSD 標準
    case NoInput         = 66;  // BSD 標準
    case Unavailable     = 69;  // BSD 標準
}
```

### 2.3 コンソール IO インターフェース

```php
<?php

declare(strict_types=1);

namespace Adlaire\Console\Contracts;

/**
 * コンソール入出力抽象インターフェース（テスト可能設計）
 */
interface ConsoleIOInterface
{
    // --- 出力 ---
    public function writeln(string $message): void;
    public function write(string $message): void;
    public function info(string $message): void;
    public function success(string $message): void;
    public function warning(string $message): void;
    public function error(string $message): void;

    /**
     * 進捗バー
     *
     * @param int $total 総件数（0 = 不定）
     */
    public function progressBar(int $total = 0): ProgressBarInterface;

    /**
     * テーブル出力
     *
     * @param array<string> $headers
     * @param array<array<string>> $rows
     */
    public function table(array $headers, array $rows): void;

    // --- 入力 ---
    public function ask(string $question, ?string $default = null): string;
    public function askSecret(string $question): string;
    public function confirm(string $question, bool $default = false): bool;
    public function choice(string $question, array $choices, ?string $default = null): string;

    /**
     * 詳細出力モードかどうか（-v / --verbose）
     */
    public function isVerbose(): bool;
}
```

### 2.4 引数・オプション定義

```php
<?php

declare(strict_types=1);

namespace Adlaire\Console\Contracts;

enum ArgumentMode
{
    case Required;
    case Optional;
    case IsArray;
}

interface ArgumentDefinitionInterface
{
    public function getName(): string;
    public function getDescription(): string;
    public function getMode(): ArgumentMode;
    public function getDefault(): mixed;
}

interface OptionDefinitionInterface
{
    public function getName(): string;
    public function getShortcut(): ?string;
    public function getDescription(): string;
    public function getDefault(): mixed;
    public function isValueRequired(): bool;
    public function isFlag(): bool; // 値なしフラグオプション
}
```

---

## 3. コマンド命名規約

### 3.1 コマンド名形式

```
{コンテキスト}:{動詞}[-{対象}]

コンテキスト: user, order, cache, db, queue, schedule 等
動詞:        create, delete, clear, compile, run, list, show 等
対象:        (省略可能) tokens, table 等

例:
  user:create        ユーザー作成
  cache:clear        キャッシュクリア
  db:migrate         マイグレーション実行
  queue:work         ジョブワーカー起動
  schedule:run       スケジューラー実行
  container:compile  コンテナコンパイル（AFE-STD-300）
```

### 3.2 コマンドクラス命名

```
{動詞}{対象}Command

例: CreateUserCommand, ClearCacheCommand, RunMigrationsCommand
```

---

## 4. 実装規則

```
RULE-603-01: コマンドクラスは CommandInterface を実装すること
RULE-603-02: echo / print による直接出力を禁止する（ConsoleIOInterface 経由のみ）
RULE-603-03: exit() / die() の直接呼び出しを禁止する（ExitCode enum を返すこと）
RULE-603-04: コマンドの実行ロジックはサービス層に委譲し、コマンドクラスは入出力処理のみとすること
RULE-603-05: 本番環境で実行する破壊的コマンドは --force フラグを必須オプションとすること
RULE-603-06: 時間のかかるコマンドは ProgressBarInterface を必ず使用すること
RULE-603-07: デプロイ・マイグレーション系コマンドは冪等性（同一コマンドを複数回実行しても同じ結果）を保証すること
```

---

## 5. 実装コード例

```php
<?php

declare(strict_types=1);

namespace Adlaire\App\Console\Commands;

use Adlaire\Console\Contracts\CommandInterface;
use Adlaire\Console\Contracts\ConsoleIOInterface;
use Adlaire\Console\Contracts\ParsedInputInterface;
use Adlaire\Console\Enums\ExitCode;
use Adlaire\User\Contracts\UserServiceInterface;
use Adlaire\User\Commands\CreateUserCommand;

final class CreateUserConsoleCommand implements CommandInterface
{
    public function __construct(
        private readonly UserServiceInterface $userService,
    ) {}

    public static function getName(): string        { return 'user:create'; }
    public static function getDescription(): string { return 'Create a new user account'; }

    public static function getDefinition(): array
    {
        return [
            new Argument('name', ArgumentMode::Required, 'User name'),
            new Argument('email', ArgumentMode::Required, 'Email address'),
            new Option('--admin', '-a', 'Grant admin role', false),
        ];
    }

    public function execute(ConsoleIOInterface $io, ParsedInputInterface $input): ExitCode
    {
        $name  = $input->getArgument('name');
        $email = $input->getArgument('email');

        $io->info("Creating user: {$name} ({$email})");

        try {
            $user = $this->userService->create(
                new CreateUserCommand(name: $name, email: $email)
            );

            $io->success("User created successfully. ID: {$user->id->value()}");

            return ExitCode::Success;
        } catch (\Throwable $e) {
            $io->error("Failed to create user: {$e->getMessage()}");
            return ExitCode::Failure;
        }
    }
}
```

---

## 変更履歴

本ファイルの変更履歴は `../VERSION_HISTORY.md` の `AFE-STD-603` セクションを参照すること。

---

*© 2026 Adlaire Group. All Rights Reserved.*
