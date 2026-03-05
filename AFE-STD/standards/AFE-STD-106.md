# AFE-STD-106: Package Repository Interface

---

## ドキュメント情報

| 項目 | 内容 |
|------|------|
| **ドキュメントID** | AFE-STD-106 |
| **タイトル** | Package Repository Interface |
| **バージョン** | Ver.1.0-1 |
| **ステータス** | ✅ 確定（初版） |
| **作成日** | 2026-03-05 |
| **最終更新日** | 2026-03-05 |
| **所有者** | Adlaire Group DX事業セグメント |
| **機密区分** | 🔒 社外秘 |

---

## 概要

本標準は、AFE（Adlaire Framework Ecosystem）における**パッケージリポジトリのインターフェース仕様**を定義する。

パッケージの取得元（Git リポジトリ、ローカルパス等）を抽象化し、統一的なインターフェースで管理する。

---

## 関連標準

| 標準ID | 名称 | 関係 |
|--------|------|------|
| AFE-STD-105 | Package Manager Interface | リポジトリを利用してパッケージ管理 |
| AFE-STD-107 | Package Configuration Standard | リポジトリ設定を定義 |

---

## 設計原則

### 1. リポジトリタイプの制限

- **Git リポジトリ**: GitHub, GitLab 等
- **ローカルパス**: モノレポ、ローカル開発用
- **HTTP リポジトリ**: ❌ 将来検討（現時点では未対応）
- **Packagist**: ❌ 未対応（サードパーティ統合禁止）

### 2. 認証対応

- Git リポジトリの認証（Personal Access Token）
- 環境変数からトークン取得
- セキュアな認証情報管理

### 3. 優先度管理

- 複数リポジトリの優先度設定
- ローカルパスを優先（開発効率）

---

## 適用範囲

### 対象

- パッケージリポジトリの抽象化
- Git リポジトリからのパッケージ取得
- ローカルパスからのパッケージ取得
- リポジトリ認証管理

### 対象外

- HTTP リポジトリ（将来対応予定）
- Packagist 統合
- プライベート Composer リポジトリ

---

## インターフェース定義

### RepositoryInterface

リポジトリの抽象インターフェース。

```php
<?php

declare(strict_types=1);

namespace Adlaire\PackageRepository\Contracts;

/**
 * パッケージリポジトリインターフェース
 * 
 * パッケージの取得元を抽象化する。
 */
interface RepositoryInterface
{
    /**
     * リポジトリタイプを取得
     * 
     * @return string "git" | "path"
     */
    public function getType(): string;
    
    /**
     * リポジトリ名を取得
     * 
     * @return string 例: "official", "local"
     */
    public function getName(): string;
    
    /**
     * リポジトリ優先度を取得
     * 
     * @return int 数値が大きいほど優先度が高い
     */
    public function getPriority(): int;
    
    /**
     * パッケージを検索する
     * 
     * @param string $packageName パッケージ名（例: "adlaire/core"）
     * @return PackageMetadataInterface|null パッケージメタデータ
     */
    public function find(string $packageName): ?PackageMetadataInterface;
    
    /**
     * パッケージの全バージョンを取得
     * 
     * @param string $packageName パッケージ名
     * @return array<string> バージョンリスト（例: ["1.0.0", "1.1.0", "1.2.0"]）
     */
    public function getVersions(string $packageName): array;
    
    /**
     * パッケージの最新バージョンを取得
     * 
     * @param string $packageName パッケージ名
     * @return string|null 最新バージョン（例: "1.2.5"）
     */
    public function getLatestVersion(string $packageName): ?string;
    
    /**
     * パッケージをダウンロードする
     * 
     * @param string $packageName パッケージ名
     * @param string $version バージョン
     * @param string $targetPath ダウンロード先パス
     * @return DownloadResultInterface ダウンロード結果
     */
    public function download(
        string $packageName,
        string $version,
        string $targetPath
    ): DownloadResultInterface;
    
    /**
     * パッケージメタデータを取得
     * 
     * @param string $packageName パッケージ名
     * @param string $version バージョン
     * @return PackageMetadataInterface
     */
    public function getMetadata(
        string $packageName,
        string $version
    ): PackageMetadataInterface;
    
    /**
     * リポジトリが利用可能か確認
     * 
     * @return bool
     */
    public function isAvailable(): bool;
}
```

---

### GitRepositoryInterface

Git リポジトリ専用インターフェース。

```php
<?php

declare(strict_types=1);

namespace Adlaire\PackageRepository\Contracts;

interface GitRepositoryInterface extends RepositoryInterface
{
    /**
     * Git リポジトリ URL を取得
     * 
     * @return string 例: "https://github.com/adlaire-group/afe-packages.git"
     */
    public function getUrl(): string;
    
    /**
     * 認証情報を取得
     * 
     * @return AuthenticationInterface|null
     */
    public function getAuthentication(): ?AuthenticationInterface;
    
    /**
     * 特定コミットのパッケージを取得
     * 
     * @param string $packageName パッケージ名
     * @param string $commit コミットハッシュ
     * @param string $targetPath ダウンロード先
     * @return DownloadResultInterface
     */
    public function downloadByCommit(
        string $packageName,
        string $commit,
        string $targetPath
    ): DownloadResultInterface;
    
    /**
     * タグ一覧を取得
     * 
     * @param string $packageName パッケージ名
     * @return array<string> タグリスト
     */
    public function getTags(string $packageName): array;
    
    /**
     * ブランチ一覧を取得
     * 
     * @param string $packageName パッケージ名
     * @return array<string> ブランチリスト
     */
    public function getBranches(string $packageName): array;
}
```

---

### PathRepositoryInterface

ローカルパスリポジトリ専用インターフェース。

```php
<?php

declare(strict_types=1);

namespace Adlaire\PackageRepository\Contracts;

interface PathRepositoryInterface extends RepositoryInterface
{
    /**
     * ローカルパスを取得
     * 
     * @return string 例: "./packages/*"
     */
    public function getPath(): string;
    
    /**
     * シンボリックリンクを使用するか
     * 
     * @return bool
     */
    public function useSymlink(): bool;
    
    /**
     * パスパターンに一致するパッケージを検索
     * 
     * @return array<PackageMetadataInterface>
     */
    public function findAllPackages(): array;
}
```

---

### AuthenticationInterface

認証情報を表すインターフェース。

```php
<?php

declare(strict_types=1);

namespace Adlaire\PackageRepository\Contracts;

interface AuthenticationInterface
{
    /**
     * 認証タイプを取得
     * 
     * @return string "token" | "ssh" | "basic"
     */
    public function getType(): string;
    
    /**
     * 認証トークンを取得（token タイプの場合）
     * 
     * @return string|null
     */
    public function getToken(): ?string;
    
    /**
     * トークンを環境変数から取得するか
     * 
     * @return string|null 環境変数名（例: "ADLAIRE_REPO_TOKEN"）
     */
    public function getTokenEnv(): ?string;
    
    /**
     * ユーザー名を取得（basic タイプの場合）
     * 
     * @return string|null
     */
    public function getUsername(): ?string;
    
    /**
     * パスワードを取得（basic タイプの場合）
     * 
     * @return string|null
     */
    public function getPassword(): ?string;
}
```

---

### PackageMetadataInterface

パッケージメタデータを表すインターフェース。

```php
<?php

declare(strict_types=1);

namespace Adlaire\PackageRepository\Contracts;

interface PackageMetadataInterface
{
    /**
     * パッケージ名
     */
    public function getName(): string;
    
    /**
     * バージョン
     */
    public function getVersion(): string;
    
    /**
     * パッケージタイプ
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
     * リポジトリ情報
     * 
     * @return array{type: string, url: string, reference: string}
     */
    public function getSource(): array;
    
    /**
     * 配布ファイル情報
     * 
     * @return array{type: string, url: string, checksum: string}|null
     */
    public function getDist(): ?array;
    
    /**
     * AFE-STD 準拠情報
     * 
     * @return array{compliant: bool, standards: array<string>}
     */
    public function getAfeStandards(): array;
}
```

---

### DownloadResultInterface

ダウンロード結果を表すインターフェース。

```php
<?php

declare(strict_types=1);

namespace Adlaire\PackageRepository\Contracts;

interface DownloadResultInterface
{
    /**
     * ダウンロードが成功したか
     */
    public function isSuccess(): bool;
    
    /**
     * ダウンロード先パス
     */
    public function getTargetPath(): string;
    
    /**
     * ダウンロードサイズ（バイト）
     */
    public function getSize(): int;
    
    /**
     * ダウンロード時間（秒）
     */
    public function getDownloadTime(): float;
    
    /**
     * チェックサム
     */
    public function getChecksum(): string;
    
    /**
     * エラーメッセージ（失敗時）
     */
    public function getError(): ?string;
}
```

---

## 実装例

### Git リポジトリからパッケージを取得

```php
<?php

use Adlaire\PackageRepository\GitRepository;
use Adlaire\PackageRepository\Authentication;

// リポジトリ作成
$repo = new GitRepository(
    name: 'official',
    url: 'https://github.com/adlaire-group/afe-packages.git',
    priority: 100,
    authentication: new Authentication(
        type: 'token',
        tokenEnv: 'ADLAIRE_REPO_TOKEN'
    )
);

// パッケージ検索
$metadata = $repo->find('adlaire/core');

if ($metadata !== null) {
    echo "Found: {$metadata->getName()} {$metadata->getVersion()}\n";
    
    // パッケージダウンロード
    $result = $repo->download(
        packageName: 'adlaire/core',
        version: '1.2.5',
        targetPath: './vendor/adlaire/core'
    );
    
    if ($result->isSuccess()) {
        echo "Downloaded to {$result->getTargetPath()}\n";
        echo "Size: {$result->getSize()} bytes\n";
    }
}
```

---

### ローカルパスからパッケージを取得

```php
<?php

use Adlaire\PackageRepository\PathRepository;

// ローカルリポジトリ作成
$repo = new PathRepository(
    name: 'local',
    path: './packages/*',
    priority: 200,
    symlink: true
);

// 全パッケージを検索
$packages = $repo->findAllPackages();

foreach ($packages as $metadata) {
    echo "Local package: {$metadata->getName()}\n";
}

// 特定パッケージを取得
$metadata = $repo->find('adlaire/http');

if ($metadata !== null) {
    echo "Found local package: {$metadata->getName()}\n";
}
```

---

### 複数リポジトリの優先度管理

```php
<?php

use Adlaire\PackageRepository\RepositoryManager;
use Adlaire\PackageRepository\GitRepository;
use Adlaire\PackageRepository\PathRepository;

$manager = new RepositoryManager();

// ローカルパス（優先度: 200）
$manager->addRepository(new PathRepository(
    name: 'local',
    path: './packages/*',
    priority: 200
));

// Git リポジトリ（優先度: 100）
$manager->addRepository(new GitRepository(
    name: 'official',
    url: 'https://github.com/adlaire-group/afe-packages.git',
    priority: 100
));

// パッケージ検索（優先度の高いリポジトリから検索）
$metadata = $manager->find('adlaire/core');

// ローカルに存在すればローカルから、なければ Git から取得
echo "Found in: {$metadata->getSource()['type']}\n";
```

---

### 最新バージョンの取得

```php
<?php

use Adlaire\PackageRepository\GitRepository;

$repo = new GitRepository(
    name: 'official',
    url: 'https://github.com/adlaire-group/afe-packages.git',
    priority: 100
);

// 全バージョンを取得
$versions = $repo->getVersions('adlaire/core');
echo "Available versions: " . implode(', ', $versions) . "\n";

// 最新バージョンを取得
$latest = $repo->getLatestVersion('adlaire/core');
echo "Latest version: {$latest}\n";

// 最新版をダウンロード
$result = $repo->download(
    packageName: 'adlaire/core',
    version: $latest,
    targetPath: './vendor/adlaire/core'
);
```

---

## アンチパターン

### ❌ HTTP リポジトリの使用

```php
<?php

// 現時点では未対応
$repo = new HttpRepository(
    name: 'http-repo',
    url: 'https://packages.adlaire.group'
);
```

**理由**: HTTP リポジトリは将来対応予定だが、現時点では未実装。

---

### ❌ Packagist の統合

```php
<?php

// 禁止
$repo = new PackagistRepository();
```

**理由**: サードパーティパッケージ統合は禁止。

---

### ❌ 認証情報のハードコード

```php
<?php

// ❌ 悪い例
$repo = new GitRepository(
    name: 'official',
    url: 'https://github.com/adlaire-group/afe-packages.git',
    priority: 100,
    authentication: new Authentication(
        type: 'token',
        token: 'ghp_abc123...'  // ハードコード禁止
    )
);

// ✅ 良い例
$repo = new GitRepository(
    name: 'official',
    url: 'https://github.com/adlaire-group/afe-packages.git',
    priority: 100,
    authentication: new Authentication(
        type: 'token',
        tokenEnv: 'ADLAIRE_REPO_TOKEN'  // 環境変数から取得
    )
);
```

**理由**: セキュリティリスク。認証情報は環境変数から取得すべき。

---

## 改版履歴

| バージョン | 改訂日 | 改訂者 | 改訂内容 |
|-----------|--------|--------|---------|
| Ver.1.0-1 | 2026-03-05 | Adlaire Group DX事業セグメント ゼネラルマネージャー | 初版作成 |

---

*本ドキュメントは Adlaire Group の機密情報です。Adlaire Group の書面による許可なく、外部への開示・複製・転用を禁止します。*

*© 2026 Adlaire Group / Adlaire Group DX Division. All Rights Reserved.*
