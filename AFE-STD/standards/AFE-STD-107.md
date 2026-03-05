# AFE-STD-107: Package Configuration Standard

---

## ドキュメント情報

| 項目 | 内容 |
|------|------|
| **ドキュメントID** | AFE-STD-107 |
| **タイトル** | Package Configuration Standard |
| **バージョン** | Ver.1.0-1 |
| **ステータス** | ✅ 確定（初版） |
| **作成日** | 2026-03-05 |
| **最終更新日** | 2026-03-05 |
| **所有者** | Adlaire Group DX事業セグメント |
| **機密区分** | 🔒 社外秘 |

---

## 概要

本標準は、AFE（Adlaire Framework Ecosystem）における**パッケージ設定ファイルの形式**を定義する。

`afe.json`（設定ファイル）と `afe.lock`（ロックファイル）の完全な仕様を規定する。

---

## 関連標準

| 標準ID | 名称 | 関係 |
|--------|------|------|
| AFE-STD-101 | オートローディング標準 | オートロード設定形式を定義 |
| AFE-STD-105 | Package Manager Interface | 設定ファイルを読み込んで処理 |
| AFE-STD-106 | Package Repository Interface | リポジトリ設定を定義 |

---

## 設計原則

### 1. JSON 形式の採用

- **人間が読みやすい**構造化データ
- 標準的なフォーマット（YAML, PHP配列は不採用）
- スキーマ検証可能

### 2. Composer 非互換

- Composer の `composer.json` とは**完全に独立**
- AFE 独自の設計思想を反映
- サードパーティパッケージ統合を禁止

### 3. バージョン制約なし

- `^1.0`, `~1.0` 等の制約記法は未対応
- 固定バージョン または `"latest"` キーワードのみ

### 4. プロファイル機能

- 環境別設定（development, production 等）
- プロファイル切り替えで異なるバージョン・設定を適用

---

## 適用範囲

### 対象

- `afe.json` の完全な仕様定義
- `afe.lock` の完全な仕様定義
- 設定ファイルのバリデーション
- スキーマ定義

### 対象外

- Composer 互換性
- プラットフォーム要件管理
- プラグイン設定

---

## `afe.json` 完全仕様

### ファイル構造

```json
{
    "afe": { },
    "package": { },
    "dependencies": { },
    "repositories": { },
    "autoload": { },
    "autoload-dev": { },
    "scripts": { },
    "config": { },
    "update-policy": { },
    "security": { },
    "standards": { },
    "type-safety": { },
    "profiles": { },
    "active-profile": "development"
}
```

---

### セクション詳細

#### 1. `afe` セクション（必須）

AFE パッケージマネージャのメタ情報。

```json
{
    "afe": {
        "version": "1.0",
        "schema": "https://standards.adlaire.group/afe-package-schema/v1.0.json"
    }
}
```

| フィールド | 型 | 必須 | 説明 |
|-----------|-----|-----|------|
| `version` | string | ✅ | AFE 設定ファイル形式のバージョン |
| `schema` | string | ⭕ | JSON スキーマ URL |

---

#### 2. `package` セクション（必須）

パッケージの基本情報。

```json
{
    "package": {
        "name": "adlaire/my-project",
        "version": "1.0.0",
        "type": "application",
        "description": "Project description",
        "keywords": ["framework", "adlaire"],
        "homepage": "https://adlaire.group",
        "license": "proprietary",
        "authors": [
            {
                "name": "倉田 和宏",
                "email": "kurata@adlaire.group",
                "role": "owner",
                "homepage": "https://adlaire.group"
            }
        ],
        "support": {
            "email": "support@adlaire.group",
            "issues": "https://github.com/adlaire-group/issues",
            "docs": "https://docs.adlaire.group",
            "source": "https://github.com/adlaire-group/afe"
        }
    }
}
```

| フィールド | 型 | 必須 | 説明 |
|-----------|-----|-----|------|
| `name` | string | ✅ | パッケージ名（`vendor/package` 形式） |
| `version` | string | ✅ | パッケージバージョン（セマンティックバージョニング） |
| `type` | string | ✅ | パッケージタイプ（`application`, `library`） |
| `description` | string | ⭕ | 説明 |
| `keywords` | array | ⭕ | キーワード |
| `homepage` | string | ⭕ | ホームページ URL |
| `license` | string | ✅ | ライセンス |
| `authors` | array | ⭕ | 作者情報 |
| `support` | object | ⭕ | サポート情報 |

---

#### 3. `dependencies` セクション（必須）

依存パッケージの定義。

```json
{
    "dependencies": {
        "runtime": {
            "adlaire/core": "latest",
            "adlaire/http": "1.0.3",
            "adlaire/container": "latest"
        },
        "development": {
            "adlaire/testing": "latest",
            "adlaire/debug": "latest"
        },
        "suggest": {
            "adlaire/cache": "For caching support",
            "adlaire/database": "For database access"
        }
    }
}
```

| フィールド | 型 | 必須 | 説明 |
|-----------|-----|-----|------|
| `runtime` | object | ✅ | 本番環境依存パッケージ |
| `development` | object | ⭕ | 開発環境依存パッケージ |
| `suggest` | object | ⭕ | 推奨パッケージ |

**バージョン指定方式**:
- `"latest"`: 常に最新の安定版を使用
- `"1.2.5"`: 固定バージョン指定
- `"^1.0"`, `"~1.0"`: ❌ 未対応

---

#### 4. `repositories` セクション（必須）

パッケージリポジトリの定義。

```json
{
    "repositories": {
        "official": {
            "type": "git",
            "url": "https://github.com/adlaire-group/afe-packages.git",
            "priority": 100,
            "authentication": {
                "type": "token",
                "token-env": "ADLAIRE_REPO_TOKEN"
            }
        },
        "local": {
            "type": "path",
            "path": "./packages/*",
            "priority": 200,
            "symlink": true
        }
    }
}
```

| フィールド | 型 | 必須 | 説明 |
|-----------|-----|-----|------|
| リポジトリ名 | object | ✅ | リポジトリ設定 |
| `type` | string | ✅ | `"git"` または `"path"` |
| `url` | string | ✅ (git) | Git リポジトリ URL |
| `path` | string | ✅ (path) | ローカルパス |
| `priority` | int | ✅ | 優先度（数値が大きいほど優先） |
| `authentication` | object | ⭕ | 認証情報 |
| `symlink` | bool | ⭕ (path) | シンボリックリンク使用 |

---

#### 5. `autoload` セクション（必須）

オートロード設定（AFE-STD-101 準拠）。

```json
{
    "autoload": {
        "psr-4": {
            "App\\": "src/",
            "App\\Controllers\\": "src/Controllers/"
        },
        "classmap": [
            "legacy/OldClass.php"
        ],
        "files": [
            "src/helpers.php"
        ],
        "exclude-from-classmap": [
            "/Tests/",
            "/test/"
        ]
    }
}
```

| フィールド | 型 | 必須 | 説明 |
|-----------|-----|-----|------|
| `psr-4` | object | ✅ | PSR-4 オートロード設定 |
| `classmap` | array | ⭕ | クラスマップ |
| `files` | array | ⭕ | 直接読み込みファイル |
| `exclude-from-classmap` | array | ⭕ | クラスマップ除外パターン |

---

#### 6. `autoload-dev` セクション

開発環境専用オートロード設定。

```json
{
    "autoload-dev": {
        "psr-4": {
            "App\\Tests\\": "tests/"
        }
    }
}
```

---

#### 7. `scripts` セクション

スクリプト・フック定義。

```json
{
    "scripts": {
        "hooks": {
            "pre-install": {
                "commands": ["echo 'Pre-install hook'"],
                "timeout": 60,
                "allow-failure": false
            },
            "post-install": {
                "commands": [
                    "@php artisan cache:clear",
                    "@php artisan config:cache"
                ],
                "parallel": false,
                "timeout": 300
            },
            "post-update": {
                "commands": ["@php artisan optimize"]
            }
        },
        "custom": {
            "test": {
                "description": "Run tests",
                "commands": ["phpunit"],
                "timeout": 600
            },
            "deploy": {
                "description": "Deploy to production",
                "commands": [
                    "afe install --profile=production",
                    "@php artisan migrate --force"
                ],
                "confirmation": true
            }
        }
    }
}
```

**フックイベント**:
- `pre-install`: インストール前
- `post-install`: インストール後
- `pre-update`: 更新前
- `post-update`: 更新後
- `pre-autoload-dump`: オートロード生成前
- `post-autoload-dump`: オートロード生成後

---

#### 8. `config` セクション

設定管理。

```json
{
    "config": {
        "directories": {
            "vendor": "vendor",
            "bin": "vendor/bin",
            "cache": ".afe-cache"
        },
        "performance": {
            "optimize-autoloader": true,
            "classmap-authoritative": false,
            "apcu-autoloader": false,
            "parallel-downloads": 4
        },
        "network": {
            "timeout": 300,
            "retry": 3,
            "verify-ssl": true
        },
        "behavior": {
            "sort-packages": true,
            "lock": true
        }
    }
}
```

---

#### 9. `update-policy` セクション

更新ポリシー（AFE 独自機能）。

```json
{
    "update-policy": {
        "strategy": "always-latest",
        "auto-update": {
            "enabled": true,
            "schedule": "daily",
            "stability": "stable",
            "notification": true
        },
        "outdated-warning": {
            "enabled": true,
            "threshold-days": 7,
            "severity": "error"
        }
    }
}
```

| フィールド | 説明 |
|-----------|------|
| `strategy` | `"always-latest"`: 常に最新版を推奨 |
| `auto-update.enabled` | 自動更新の有効化 |
| `outdated-warning.threshold-days` | 古いバージョン警告の閾値（日数） |

---

#### 10. `security` セクション

セキュリティポリシー（AFE 独自機能）。

```json
{
    "security": {
        "policies": {
            "allow-third-party": false,
            "allowed-vendors": ["adlaire"],
            "block-external-repositories": true,
            "require-signature": true,
            "require-checksum-verification": true
        },
        "scan": {
            "enabled": true,
            "on-install": true,
            "on-update": true,
            "severity-threshold": "medium",
            "auto-reject": ["critical", "high"]
        },
        "vulnerability-database": {
            "sources": [
                "https://security.adlaire.group/vulnerabilities"
            ],
            "update-interval": 3600
        }
    }
}
```

---

#### 11. `standards` セクション

AFE-STD 準拠検証（AFE 独自機能）。

```json
{
    "standards": {
        "enforce": true,
        "required": [
            "AFE-STD-100",
            "AFE-STD-101",
            "AFE-STD-102"
        ],
        "validation": {
            "naming-convention": true,
            "autoloading": true,
            "attributes": true,
            "interfaces": true
        },
        "compliance-level": "strict"
    }
}
```

---

#### 12. `type-safety` セクション

型安全性検証（AFE 独自機能）。

```json
{
    "type-safety": {
        "enabled": true,
        "strict-mode": true,
        "validation": {
            "check-interface-compatibility": true,
            "check-return-types": true,
            "check-parameter-types": true
        },
        "rules": {
            "forbid-mixed-types": true,
            "require-typed-properties": true,
            "require-return-types": true
        }
    }
}
```

---

#### 13. `profiles` セクション

環境別プロファイル（AFE 独自機能）。

```json
{
    "profiles": {
        "development": {
            "dependencies": {
                "runtime": {
                    "adlaire/core": "latest"
                },
                "development": {
                    "adlaire/debug": "latest",
                    "adlaire/testing": "latest"
                }
            },
            "config": {
                "performance": {
                    "optimize-autoloader": false
                }
            }
        },
        "production": {
            "dependencies": {
                "runtime": {
                    "adlaire/core": "1.2.5"
                }
            },
            "config": {
                "performance": {
                    "optimize-autoloader": true,
                    "classmap-authoritative": true
                }
            },
            "security": {
                "policies": {
                    "require-signature": true
                }
            }
        }
    },
    "active-profile": "development"
}
```

---

## `afe.lock` 完全仕様

### ファイル構造

```json
{
    "afe": { },
    "content-hash": "...",
    "packages": { },
    "profile": "development"
}
```

---

### セクション詳細

#### 1. `afe` セクション

```json
{
    "afe": {
        "version": "1.0",
        "locked-at": "2026-03-05T12:00:00+09:00",
        "generator": "afe-package-manager/1.0.0"
    }
}
```

---

#### 2. `content-hash` セクション

`afe.json` の内容ハッシュ。変更検出に使用。

```json
{
    "content-hash": "abc123def456ghi789jkl012mno345pqr678"
}
```

---

#### 3. `packages` セクション

インストール済みパッケージの詳細情報。

```json
{
    "packages": {
        "runtime": [
            {
                "name": "adlaire/core",
                "version": "1.2.5",
                "version-normalized": "1.2.5.0",
                "source": {
                    "type": "git",
                    "url": "https://github.com/adlaire-group/afe-core.git",
                    "reference": "a1b2c3d4e5f6",
                    "commit": "a1b2c3d4e5f6789012345678901234567890abcd",
                    "branch": "main"
                },
                "dist": {
                    "type": "zip",
                    "url": "https://github.com/adlaire-group/afe-core/archive/1.2.5.zip",
                    "checksum": {
                        "sha256": "abc123..."
                    },
                    "signature": {
                        "algorithm": "rsa-sha256",
                        "value": "def456..."
                    }
                },
                "dependencies": {
                    "adlaire/container": "^1.0",
                    "adlaire/http": "^1.0"
                },
                "autoload": {
                    "psr-4": {
                        "Adlaire\\Core\\": "src/"
                    }
                },
                "type": "library",
                "license": ["MIT"],
                "authors": [
                    {
                        "name": "Adlaire Group",
                        "email": "dev@adlaire.group"
                    }
                ],
                "description": "Adlaire Core Package",
                "keywords": ["adlaire", "core"],
                "time": "2026-03-01T10:00:00+09:00",
                "afe-standards": {
                    "compliant": true,
                    "standards": ["AFE-STD-100", "AFE-STD-101", "AFE-STD-102"]
                },
                "security": {
                    "vulnerabilities": [],
                    "last-scanned": "2026-03-05T12:00:00+09:00"
                }
            }
        ],
        "development": [
            {
                "name": "adlaire/testing",
                "version": "1.0.2",
                "source": { },
                "autoload": { },
                "time": "2026-02-25T09:00:00+09:00"
            }
        ]
    }
}
```

---

#### 4. `profile` セクション

使用中のプロファイル名。

```json
{
    "profile": "development"
}
```

---

## バリデーション

### `afe.json` の検証

```bash
afe validate
afe validate --strict
```

**検証項目**:
1. ✅ JSON 形式が正しいか
2. ✅ 必須フィールドが存在するか
3. ✅ バージョン形式が正しいか
4. ✅ リポジトリ設定が正しいか
5. ✅ サードパーティパッケージが含まれていないか
6. ✅ プロファイル設定が正しいか

---

## JSON スキーマ

`afe.json` の JSON スキーマ定義（抜粋）。

```json
{
    "$schema": "http://json-schema.org/draft-07/schema#",
    "title": "AFE Package Configuration",
    "type": "object",
    "required": ["afe", "package", "dependencies", "repositories", "autoload"],
    "properties": {
        "afe": {
            "type": "object",
            "required": ["version"],
            "properties": {
                "version": {
                    "type": "string",
                    "pattern": "^[0-9]+\\.[0-9]+$"
                },
                "schema": {
                    "type": "string",
                    "format": "uri"
                }
            }
        },
        "package": {
            "type": "object",
            "required": ["name", "version", "type", "license"],
            "properties": {
                "name": {
                    "type": "string",
                    "pattern": "^[a-z0-9-]+/[a-z0-9-]+$"
                },
                "version": {
                    "type": "string",
                    "pattern": "^[0-9]+\\.[0-9]+\\.[0-9]+$"
                },
                "type": {
                    "type": "string",
                    "enum": ["application", "library"]
                },
                "license": {
                    "type": "string"
                }
            }
        },
        "dependencies": {
            "type": "object",
            "required": ["runtime"],
            "properties": {
                "runtime": {
                    "type": "object",
                    "patternProperties": {
                        "^adlaire/[a-z0-9-]+$": {
                            "type": "string",
                            "pattern": "^(latest|[0-9]+\\.[0-9]+\\.[0-9]+)$"
                        }
                    }
                }
            }
        }
    }
}
```

---

## 実装例

### `afe.json` 最小構成

```json
{
    "afe": {
        "version": "1.0"
    },
    "package": {
        "name": "adlaire/my-app",
        "version": "1.0.0",
        "type": "application",
        "license": "proprietary"
    },
    "dependencies": {
        "runtime": {
            "adlaire/core": "latest"
        }
    },
    "repositories": {
        "official": {
            "type": "git",
            "url": "https://github.com/adlaire-group/afe-packages.git",
            "priority": 100
        }
    },
    "autoload": {
        "psr-4": {
            "App\\": "src/"
        }
    }
}
```

---

### `afe.json` フル構成

前述の各セクション詳細を参照。

---

## アンチパターン

### ❌ バージョン制約記法の使用

```json
{
    "dependencies": {
        "runtime": {
            "adlaire/core": "^1.0.0"
        }
    }
}
```

**理由**: `^`, `~` 等の制約記法は未対応。

---

### ❌ サードパーティパッケージの指定

```json
{
    "dependencies": {
        "runtime": {
            "symfony/http-foundation": "^6.0"
        }
    }
}
```

**理由**: Adlaire 製パッケージのみ許可。

---

### ❌ Packagist の使用

```json
{
    "repositories": {
        "packagist": {
            "type": "composer",
            "url": "https://packagist.org"
        }
    }
}
```

**理由**: Packagist 統合は禁止。

---

## 改版履歴

| バージョン | 改訂日 | 改訂者 | 改訂内容 |
|-----------|--------|--------|---------|
| Ver.1.0-1 | 2026-03-05 | Adlaire Group DX事業セグメント ゼネラルマネージャー | 初版作成 |

---

*本ドキュメントは Adlaire Group の機密情報です。Adlaire Group の書面による許可なく、外部への開示・複製・転用を禁止します。*

*© 2026 Adlaire Group / Adlaire Group DX Division. All Rights Reserved.*
