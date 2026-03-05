# AFE-STD-105: Package Manager Interface

---

## ドキュメント情報

| 項目 | 内容 |
|------|------|
| **ドキュメントID** | AFE-STD-105 |
| **タイトル** | Package Manager Interface |
| **バージョン** | Ver.1.0-1 |
| **ステータス** | ✅ 確定（初版） |
| **作成日** | 2026-03-05 |
| **最終更新日** | 2026-03-05 |
| **所有者** | Adlaire Group DX事業セグメント |
| **機密区分** | 🔒 社外秘 |

---

## 概要

本標準は、AFE（Adlaire Framework Ecosystem）における**パッケージマネージャのインターフェース仕様**を定義する。

Composer の代替として、Adlaire Group 内製パッケージ専用のパッケージ管理システムを提供する。

---

## 関連標準

| 標準ID | 名称 | 関係 |
|--------|------|------|
| AFE-STD-101 | オートローディング標準 | オートロード生成機能で連携 |
| AFE-STD-106 | Package Repository Interface | リポジトリ管理で連携 |
| AFE-STD-107 | Package Configuration Standard | 設定ファイル形式を定義 |

---

## 設計原則

### 1. Adlaire 内製パッケージ専用

- **サードパーティパッケージ統合を禁止**
- Adlaire Group 製パッケージのみを管理
- 完全な品質管理とセキュリティ保証

### 2. 常に最新版を使用

- **バージョン制約処理なし**（`^1.0`, `~1.0` 等は未対応）
- 固定バージョン指定 または `"latest"` キーワード
- 自動更新推奨（常に最新の安定版を使用）

### 3. 型安全性重視

- PHP 8.2+ 型システムを最大活用
- インターフェース駆動設計
- 厳密な型チェック

### 4. AFE-STD 準拠の徹底

- インストールするパッケージの AFE-STD 準拠を検証
- 非準拠パッケージはインストール拒否

---

## 適用範囲

### 対象

- AFE パッケージマネージャ実装
- AFE パッケージのインストール・更新・削除
- オートローダ生成（AFE-STD-101 準拠）
- 設定ファイル管理（`afe.json`, `afe.lock`）

### 対象外

- サードパーティパッケージの統合
- Composer との互換性
- プラットフォーム要件管理（PHP バージョン、拡張チェック等）
- プラグイン機構

---

## 外部標準との互換性

### Composer との差異

| 項目 | Composer | AFE-STD-105 | 互換性 |
|------|----------|-------------|--------|
| **バージョン制約** | `^1.0`, `~1.0` 対応 | 固定バージョンまたは `"latest"` のみ | ❌ 非互換 |
| **依存関係解決** | 複雑なアルゴリズム | 単純なバージョン指定 | ❌ 非互換 |
| **サードパーティ** | Packagist 統合 | Adlaire 製のみ | ❌ 非互換 |
| **オートロード** | PSR-4/PSR-0 対応 | PSR-4 のみ（AFE-STD-101） | ⚠️ 部分互換 |
| **スクリプト** | フック・カスタム対応 | フック・カスタム対応 | ✅ 互換 |
| **リポジトリ** | 多様なタイプ対応 | Git/Path のみ | ⚠️ 部分互換 |
| **設定ファイル** | composer.json | afe.json | ❌ 非互換 |
| **ロックファイル** | composer.lock | afe.lock | ❌ 非互換 |

---

## インターフェース定義

### PackageManagerInterface

パッケージマネージャのコアインターフェース。

```php
<?php

declare(strict_types=1);

namespace Adlaire\PackageManager\Contracts;

/**
 * パッケージマネージャインターフェース
 * 
 * Adlaire 内製パッケージの管理を提供する。
 * サードパーティパッケージの統合は禁止。
 */
interface PackageManagerInterface
{
    /**
     * パッケージをインストールする
     * 
     * @param string $configPath 設定ファイルパス（afe.json）
     * @return InstallResultInterface インストール結果
     * @throws PackageNotFoundException パッケージが見つからない
     * @throws SecurityViolationException セキュリティポリシー違反
     * @throws StandardsViolationException AFE-STD 非準拠
     */
    public function install(string $configPath): InstallResultInterface;
    
    /**
     * パッケージを更新する
     * 
     * @param array<string> $packages 更新対象パッケージ名（空の場合は全て）
     * @return UpdateResultInterface 更新結果
     */
    public function update(array $packages = []): UpdateResultInterface;
    
    /**
     * パッケージを追加する
     * 
     * @param string $package パッケージ名（例: "adlaire/core"）
     * @param string $version バージョン（"latest" または "1.2.5"）
     * @param bool $dev 開発依存として追加するか
     * @return void
     */
    public function require(string $package, string $version = 'latest', bool $dev = false): void;
    
    /**
     * パッケージを削除する
     * 
     * @param string $package パッケージ名
     * @return void
     */
    public function remove(string $package): void;
    
    /**
     * インストール済みパッケージ一覧を取得
     * 
     * @return array<PackageInterface>
     */
    public function list(): array;
    
    /**
     * 特定パッケージの情報を取得
     * 
     * @param string $package パッケージ名
     * @return PackageInterface|null
     */
    public function show(string $package): ?PackageInterface;
    
    /**
     * 設定ファイルを検証する
     * 
     * @param string $configPath 設定ファイルパス
     * @return ValidationResultInterface 検証結果
     */
    public function validate(string $configPath): ValidationResultInterface;
    
    /**
     * オートローダを生成する（AFE-STD-101 準拠）
     * 
     * @param bool $optimize 最適化するか
     * @param bool $classmapAuthoritative クラスマップ優先モードか
     * @return void
     */
    public function dumpAutoload(
        bool $optimize = false,
        bool $classmapAuthoritative = false
    ): void;
    
    /**
     * 古いパッケージを検出する
     * 
     * @param int $thresholdDays 閾値（日数）
     * @return array<OutdatedPackageInterface>
     */
    public function detectOutdated(int $thresholdDays = 7): array;
    
    /**
     * サードパーティパッケージを検出する（禁止されているパッケージ）
     * 
     * @return array<string> サードパーティパッケージ名のリスト
     */
    public function detectThirdParty(): array;
}
```

---

### InstallResultInterface

インストール結果を表すインターフェース。

```php
<?php

declare(strict_types=1);

namespace Adlaire\PackageManager\Contracts;

interface InstallResultInterface
{
    /**
     * インストールが成功したか
     */
    public function isSuccess(): bool;
    
    /**
     * インストールされたパッケージ一覧
     * 
     * @return array<PackageInterface>
     */
    public function getInstalledPackages(): array;
    
    /**
     * エラーメッセージ（失敗時）
     */
    public function getErrors(): array;
    
    /**
     * 実行時間（秒）
     */
    public function getExecutionTime(): float;
}
```

---

### UpdateResultInterface

更新結果を表すインターフェース。

```php
<?php

declare(strict_types=1);

namespace Adlaire\PackageManager\Contracts;

interface UpdateResultInterface
{
    /**
     * 更新が成功したか
     */
    public function isSuccess(): bool;
    
    /**
     * 更新されたパッケージ一覧
     * 
     * @return array<UpdatedPackageInterface>
     */
    public function getUpdatedPackages(): array;
    
    /**
     * 更新前→更新後のバージョンマップ
     * 
     * @return array<string, array{from: string, to: string}>
     */
    public function getVersionChanges(): array;
    
    /**
     * エラーメッセージ（失敗時）
     */
    public function getErrors(): array;
}
```

---

### PackageInterface

パッケージを表すインターフェース。

```php
<?php

declare(strict_types=1);

namespace Adlaire\PackageManager\Contracts;

interface PackageInterface
{
    /**
     * パッケージ名（例: "adlaire/core"）
     */
    public function getName(): string;
    
    /**
     * バージョン（例: "1.2.5"）
     */
    public function getVersion(): string;
    
    /**
     * パッケージタイプ（library, application）
     */
    public function getType(): string;
    
    /**
     * 説明
     */
    public function getDescription(): string;
    
    /**
     * ライセンス
     */
    public function getLicense(): string;
    
    /**
     * 依存パッケージ
     * 
     * @return array<string, string>
     */
    public function getDependencies(): array;
    
    /**
     * オートロード設定
     * 
     * @return array<string, mixed>
     */
    public function getAutoload(): array;
    
    /**
     * AFE-STD 準拠情報
     * 
     * @return AfeStandardsComplianceInterface
     */
    public function getStandardsCompliance(): AfeStandardsComplianceInterface;
    
    /**
     * インストールパス
     */
    public function getInstallPath(): string;
    
    /**
     * インストール日時
     */
    public function getInstalledAt(): \DateTimeImmutable;
}
```

---

### ValidationResultInterface

検証結果を表すインターフェース。

```php
<?php

declare(strict_types=1);

namespace Adlaire\PackageManager\Contracts;

interface ValidationResultInterface
{
    /**
     * 検証が成功したか
     */
    public function isValid(): bool;
    
    /**
     * エラー一覧
     * 
     * @return array<ValidationErrorInterface>
     */
    public function getErrors(): array;
    
    /**
     * 警告一覧
     * 
     * @return array<ValidationWarningInterface>
     */
    public function getWarnings(): array;
}
```

---

### AfeStandardsComplianceInterface

AFE-STD 準拠情報を表すインターフェース。

```php
<?php

declare(strict_types=1);

namespace Adlaire\PackageManager\Contracts;

interface AfeStandardsComplianceInterface
{
    /**
     * AFE-STD に準拠しているか
     */
    public function isCompliant(): bool;
    
    /**
     * 準拠している標準のリスト
     * 
     * @return array<string> 例: ["AFE-STD-100", "AFE-STD-101"]
     */
    public function getCompliantStandards(): array;
    
    /**
     * 非準拠の標準のリスト
     * 
     * @return array<string>
     */
    public function getNonCompliantStandards(): array;
    
    /**
     * 準拠レベル（strict, standard, loose）
     */
    public function getComplianceLevel(): string;
}
```

---

## 例外クラス

### PackageNotFoundException

```php
<?php

declare(strict_types=1);

namespace Adlaire\PackageManager\Exceptions;

class PackageNotFoundException extends \RuntimeException
{
    public function __construct(string $packageName)
    {
        parent::__construct("Package not found: {$packageName}");
    }
}
```

---

### SecurityViolationException

```php
<?php

declare(strict_types=1);

namespace Adlaire\PackageManager\Exceptions;

class SecurityViolationException extends \RuntimeException
{
    public function __construct(string $message)
    {
        parent::__construct("Security policy violation: {$message}");
    }
}
```

---

### StandardsViolationException

```php
<?php

declare(strict_types=1);

namespace Adlaire\PackageManager\Exceptions;

class StandardsViolationException extends \RuntimeException
{
    public function __construct(string $packageName, array $violations)
    {
        $violationsList = implode(', ', $violations);
        parent::__construct(
            "Package {$packageName} does not comply with AFE standards: {$violationsList}"
        );
    }
}
```

---

## 実装例

### パッケージのインストール

```php
<?php

use Adlaire\PackageManager\PackageManager;

$manager = new PackageManager();

// afe.json に基づいてインストール
$result = $manager->install('./afe.json');

if ($result->isSuccess()) {
    echo "Installed packages:\n";
    foreach ($result->getInstalledPackages() as $package) {
        echo "  - {$package->getName()} {$package->getVersion()}\n";
    }
} else {
    echo "Installation failed:\n";
    foreach ($result->getErrors() as $error) {
        echo "  - {$error}\n";
    }
}
```

---

### パッケージの追加

```php
<?php

use Adlaire\PackageManager\PackageManager;

$manager = new PackageManager();

// 最新版を追加
$manager->require('adlaire/http');

// 固定バージョンを追加
$manager->require('adlaire/core', '1.2.5');

// 開発依存として追加
$manager->require('adlaire/testing', 'latest', dev: true);
```

---

### パッケージの更新

```php
<?php

use Adlaire\PackageManager\PackageManager;

$manager = new PackageManager();

// 全パッケージを最新版に更新
$result = $manager->update();

if ($result->isSuccess()) {
    foreach ($result->getVersionChanges() as $package => $change) {
        echo "{$package}: {$change['from']} → {$change['to']}\n";
    }
}
```

---

### 古いパッケージの検出

```php
<?php

use Adlaire\PackageManager\PackageManager;

$manager = new PackageManager();

// 7日以上古いパッケージを検出
$outdated = $manager->detectOutdated(thresholdDays: 7);

if (!empty($outdated)) {
    echo "Outdated packages (update required):\n";
    foreach ($outdated as $package) {
        echo "  - {$package->getName()} {$package->getVersion()}\n";
        echo "    Latest: {$package->getLatestVersion()}\n";
    }
}
```

---

### サードパーティパッケージの検出

```php
<?php

use Adlaire\PackageManager\PackageManager;

$manager = new PackageManager();

// Adlaire 製以外のパッケージを検出
$thirdParty = $manager->detectThirdParty();

if (!empty($thirdParty)) {
    echo "Third-party packages detected (not allowed):\n";
    foreach ($thirdParty as $package) {
        echo "  - {$package}\n";
    }
    echo "\nAFE only allows Adlaire Group packages.\n";
}
```

---

## アンチパターン

### ❌ サードパーティパッケージの使用

```json
{
    "dependencies": {
        "runtime": {
            "symfony/http-foundation": "^6.0"
        }
    }
}
```

**理由**: AFE はサードパーティパッケージの統合を禁止。

---

### ❌ バージョン制約の使用

```json
{
    "dependencies": {
        "runtime": {
            "adlaire/core": "^1.0.0"
        }
    }
}
```

**理由**: バージョン制約（`^`, `~`）は未対応。`"latest"` または固定バージョンを使用。

---

### ❌ 古いバージョンの放置

```php
// 7日以上古いバージョンを放置
$manager->install('./afe.json'); // adlaire/core 1.0.0

// 最新版は 1.5.0 だが更新していない
```

**理由**: AFE は常に最新版を使用することを推奨。

---

## 改版履歴

| バージョン | 改訂日 | 改訂者 | 改訂内容 |
|-----------|--------|--------|---------|
| Ver.1.0-1 | 2026-03-05 | Adlaire Group DX事業セグメント ゼネラルマネージャー | 初版作成 |

---

*本ドキュメントは Adlaire Group の機密情報です。Adlaire Group の書面による許可なく、外部への開示・複製・転用を禁止します。*

*© 2026 Adlaire Group / Adlaire Group DX Division. All Rights Reserved.*
