# AFE パッケージマネージャ方針

**ファイル名**: `PACKAGE_MANAGER.md`
**バージョン**: Ver.1.0-1
**最終更新**: 2026-03-06
**作成部門**: Adlaire Group DX 事業セグメント
**機密レベル**: 社外秘

---

## 概要

本ドキュメントは、Adlaire Framework Ecosystem（AFE）における**パッケージ管理機能の方針**を定めるものである。
AFE-STD-105〜107の標準仕様に基づき、AFEプロジェクトに独自パッケージマネージャを含める方針を策定する。

| 項目 | 内容 |
|------|------|
| **対象標準** | AFE-STD-105（Package Manager Interface）<br>AFE-STD-106（Package Repository Interface）<br>AFE-STD-107（Package Configuration Standard） |
| **実装時期** | Phase 1 以降（Phase 0 では方針策定のみ） |
| **実装状況** | 🔜 未実装（方針のみ確定） |
| **実装制約** | コード実装は Phase 1 以降に限定。Phase 0 では仕様・方針の策定のみを行う |

---

## 方針の背景

### 1. Composer 内製化の必要性

| 理由 | 説明 |
|------|------|
| **完全独立性** | 外部ツール（Composer）への依存を排除し、AFE エコシステムの完全独立を実現 |
| **AFE-STD 完全準拠** | AFE-STD-101（オートローディング）等の標準に完全準拠したパッケージ管理を実現 |
| **カスタマイズ性** | Adlaire Group 独自の要件（セキュリティ・ライセンス監査等）に最適化 |
| **セキュリティ強化** | 社内専用パッケージのみを扱い、外部依存による脆弱性リスクを排除 |
| **技術的理解** | パッケージ管理の内部構造を完全把握し、保守性・拡張性を向上 |

### 2. サードパーティ統合の禁止

AFE パッケージマネージャは、**Adlaire Group 社内パッケージのみ**を対象とする。

| 項目 | 方針 |
|------|------|
| **サードパーティ統合** | ❌ **禁止** — Packagist 等の外部リポジトリとの統合は行わない |
| **許可されるパッケージ** | Adlaire Group が開発・管理するパッケージのみ |
| **外部パッケージ** | 使用不可。必要な場合は社内で再実装またはフォーク管理 |
| **Composer 互換性** | 非互換。`afe.json` / `afe.lock` 独自形式を採用 |

### 3. 常に最新版使用ルール

| 項目 | 方針 |
|------|------|
| **バージョン制約** | ❌ **廃止** — `^1.2.3`, `~1.2.3`, `>=1.0` 等の制約表記は使用しない |
| **許可される指定** | 固定バージョン（`1.2.5`）または `"latest"` のみ |
| **自動更新** | 推奨。daily での自動更新を標準とする |
| **古いバージョン** | 7日以上古いバージョンはエラー扱い（警告または拒否） |
| **安定性保証** | `stable` のみを対象。`dev`, `alpha`, `beta` は明示的指定時のみ |

---

## AFE パッケージマネージャの特徴

### 1. 独自仕様

| 特徴 | 内容 |
|------|------|
| **設定ファイル** | `afe.json`（プロジェクト設定）<br>`afe.lock`（ロックファイル） |
| **コマンド** | `afe install`, `afe update`, `afe require`, `afe remove` 等 |
| **標準準拠** | AFE-STD-105（Package Manager Interface）<br>AFE-STD-106（Package Repository Interface）<br>AFE-STD-107（Package Configuration Standard） |
| **Composer との違い** | Composer 非互換。独自形式・独自コマンド・独自設定 |

### 2. afe.json の構造（概要）

```json
{
  "afe": {
    "version": "1.0",
    "schema": "https://standards.adlaire.group/afe-package-schema/v1.0.json"
  },
  "package": {
    "name": "adlaire/my-project",
    "version": "1.0.0",
    "type": "application"
  },
  "dependencies": {
    "runtime": {
      "adlaire/core": "latest",
      "adlaire/http": "1.2.5"
    },
    "development": {
      "adlaire/testing": "latest",
      "adlaire/debug": "latest"
    }
  },
  "repositories": [
    {
      "name": "official",
      "type": "git",
      "url": "https://github.com/adlaire-group/afe-packages.git",
      "authentication": {
        "type": "token",
        "token": "${ADLAIRE_REPO_TOKEN}"
      }
    }
  ],
  "autoload": {
    "psr-4": {
      "App\\": "src/"
    }
  },
  "standards": {
    "enforce": true,
    "required": ["AFE-STD-100", "AFE-STD-101", "AFE-STD-102"]
  },
  "security": {
    "scan": {
      "enabled": true,
      "on": ["install", "update"]
    }
  }
}
```

### 3. afe.lock の構造（概要）

```json
{
  "afe": {
    "version": "1.0"
  },
  "locked-at": "2026-03-06T10:00:00+09:00",
  "content-hash": "abc123def456...",
  "packages": {
    "runtime": [
      {
        "name": "adlaire/core",
        "version": "1.2.5",
        "source": {
          "type": "git",
          "url": "https://github.com/adlaire-group/afe-core.git",
          "reference": "a1b2c3d4e5f6..."
        },
        "checksum": {
          "type": "sha256",
          "value": "abc123..."
        }
      }
    ]
  }
}
```

---

## 主要機能（Phase 1 以降実装予定）

### 1. コア機能

| 機能 | 説明 | 実装時期 |
|------|------|----------|
| **パッケージインストール** | `afe install` — afe.json に基づいてパッケージをインストール | Phase 1 |
| **パッケージ更新** | `afe update` — 最新版への更新 | Phase 1 |
| **パッケージ追加** | `afe require adlaire/package` — パッケージを追加 | Phase 1 |
| **パッケージ削除** | `afe remove adlaire/package` — パッケージを削除 | Phase 1 |
| **オートローダ生成** | `afe dump-autoload` — PSR-4/Classmap オートローダを生成 | Phase 1 |
| **ロックファイル管理** | `afe.lock` の自動生成・検証 | Phase 1 |

### 2. リポジトリ管理

| 機能 | 説明 | 実装時期 |
|------|------|----------|
| **Git リポジトリ** | Git プロトコルでのパッケージ取得 | Phase 1 |
| **HTTP リポジトリ** | HTTP/HTTPS でのパッケージ取得 | Phase 1 |
| **ローカルリポジトリ** | `./packages/*` 等のローカルパス対応 | Phase 1 |
| **トークン認証** | GitHub/GitLab 等のプライベートリポジトリ認証 | Phase 1 |

### 3. AFE-STD 準拠検証（独自機能）

| 機能 | 説明 | 実装時期 |
|------|------|----------|
| **標準準拠チェック** | AFE-STD-100/101/102 への準拠を自動検証 | Phase 2 |
| **型安全性検証** | strict-mode での型チェック | Phase 2 |
| **インターフェース検証** | 必須インターフェースの実装チェック | Phase 2 |

### 4. セキュリティ機能（独自機能）

| 機能 | 説明 | 実装時期 |
|------|------|----------|
| **ビルトインスキャン** | インストール時の自動脆弱性スキャン | Phase 2 |
| **チェックサム検証** | SHA-256 によるパッケージ整合性検証 | Phase 1 |
| **署名検証** | RSA-SHA256 による署名検証 | Phase 2 |
| **ライセンス監査** | 許可/禁止ライセンスの自動チェック | Phase 2 |

### 5. 環境別プロファイル（独自機能）

| 機能 | 説明 | 実装時期 |
|------|------|----------|
| **プロファイル管理** | development / staging / production 別の設定 | Phase 2 |
| **プロファイル切替** | `afe install --profile=production` | Phase 2 |
| **差分管理** | プロファイル間の依存関係・設定差分を管理 | Phase 2 |

### 6. その他機能

| 機能 | 説明 | 実装時期 |
|------|------|----------|
| **依存関係可視化** | `afe deps:graph` — 依存ツリーを SVG/JSON 出力 | Phase 2 |
| **スナップショット** | `afe snapshot:create` — 更新前の状態保存 | Phase 2 |
| **ロールバック** | `afe snapshot:restore <id>` — 以前の状態に復元 | Phase 2 |
| **モノレポ対応** | workspace 機能によるモノレポ統合管理 | Phase 3 |

---

## コマンド一覧（Phase 1 以降実装予定）

### パッケージ管理

| コマンド | 説明 |
|----------|------|
| `afe install` | afe.json に基づいてパッケージをインストール |
| `afe update` | すべてのパッケージを最新版に更新 |
| `afe update adlaire/core` | 特定パッケージのみ更新 |
| `afe require adlaire/http` | パッケージを追加 |
| `afe require adlaire/http:1.2.5` | 固定バージョンで追加 |
| `afe remove adlaire/http` | パッケージを削除 |
| `afe show` | インストール済みパッケージ一覧 |
| `afe show adlaire/core` | 特定パッケージの詳細情報 |
| `afe outdated` | 古いパッケージの一覧（7日以上） |

### オートローダ

| コマンド | 説明 |
|----------|------|
| `afe dump-autoload` | オートローダファイル生成 |
| `afe dump-autoload --optimize` | 最適化されたオートローダ生成 |

### 検証・診断

| コマンド | 説明 |
|----------|------|
| `afe validate` | afe.json の検証 |
| `afe validate:standards` | AFE-STD 準拠検証 |
| `afe diagnose` | 環境診断 |

### セキュリティ

| コマンド | 説明 |
|----------|------|
| `afe security:audit` | 脆弱性スキャン |
| `afe security:report` | セキュリティレポート生成 |
| `afe license:audit` | ライセンス監査 |

### 依存関係

| コマンド | 説明 |
|----------|------|
| `afe deps:graph` | 依存ツリーを可視化（SVG/JSON） |
| `afe deps:analyze` | 依存関係分析（循環・未使用・重複検出） |
| `afe deps:why adlaire/http` | なぜこのパッケージが必要か |

### スナップショット

| コマンド | 説明 |
|----------|------|
| `afe snapshot:create` | 現在の状態をスナップショット保存 |
| `afe snapshot:list` | スナップショット一覧 |
| `afe snapshot:restore <id>` | スナップショットから復元 |

### その他

| コマンド | 説明 |
|----------|------|
| `afe init` | afe.json を対話式で作成 |
| `afe clear-cache` | キャッシュをクリア |
| `afe --version` | バージョン情報表示 |

---

## ディレクトリ構造（Phase 1 以降実装予定）

Phase 1 以降に実装する際の推奨ディレクトリ構造：

```
AFE/
├── packages/
│   ├── package-manager/          # AFE-STD-105 実装
│   │   ├── src/
│   │   │   ├── Contracts/
│   │   │   │   ├── PackageManagerInterface.php
│   │   │   │   ├── InstallerInterface.php
│   │   │   │   ├── AutoloaderInterface.php
│   │   │   │   └── LockFileInterface.php
│   │   │   └── (実装クラス)
│   │   └── afe.json
│   │
│   ├── package-repository/        # AFE-STD-106 実装
│   │   ├── src/
│   │   │   ├── Contracts/
│   │   │   │   ├── RepositoryInterface.php
│   │   │   │   ├── PackageInterface.php
│   │   │   │   └── AuthenticatorInterface.php
│   │   │   └── (実装クラス)
│   │   └── afe.json
│   │
│   └── package-config/            # AFE-STD-107 実装
│       ├── src/
│       │   ├── Contracts/
│       │   │   ├── ConfigInterface.php
│       │   │   ├── ValidatorInterface.php
│       │   │   └── SchemaInterface.php
│       │   └── (実装クラス)
│       └── afe.json
│
├── bin/
│   └── afe                        # CLIエントリーポイント
│
└── afe.json                       # ルート設定ファイル
```

---

## 実装フェーズ計画

### Phase 1（2026 Q2 予定）

| 項目 | 内容 |
|------|------|
| **対象** | コア機能実装 |
| **実装内容** | - インターフェース定義（Contracts）<br>- AfeConfig / AfeLock 実装<br>- Autoloader 実装（PSR-4/Classmap）<br>- 基本的なリポジトリ管理（Git/Local）<br>- Installer 実装（install/update/require/remove）<br>- CLI（afe コマンド）基本実装 |
| **成果物** | `afe install`, `afe update`, `afe require`, `afe remove`, `afe dump-autoload` |

### Phase 2（2026 Q3 予定）

| 項目 | 内容 |
|------|------|
| **対象** | 独自機能実装 |
| **実装内容** | - AFE-STD 準拠検証<br>- セキュリティスキャン<br>- ライセンス監査<br>- 依存関係可視化<br>- 環境別プロファイル<br>- スナップショット/ロールバック |
| **成果物** | `afe validate:standards`, `afe security:audit`, `afe deps:graph`, `afe snapshot:*` |

### Phase 3（2026 Q4 予定）

| 項目 | 内容 |
|------|------|
| **対象** | 高度な機能実装 |
| **実装内容** | - モノレポ統合管理<br>- プラグイン機構<br>- パフォーマンス最適化<br>- 並列ダウンロード |
| **成果物** | workspace 機能、プラグインシステム |

---

## 制約事項

| 項目 | 内容 |
|------|------|
| **Phase 0 の制約** | ❌ **実装作業禁止** — Phase 0 では方針策定・ドキュメント化のみを行う |
| **実装開始時期** | Phase 1 以降（2026 Q2 以降） |
| **対象パッケージ** | Adlaire Group 社内パッケージのみ。サードパーティ統合は禁止 |
| **バージョン制約** | 固定バージョンまたは "latest" のみ。セマンティックバージョニング制約（^, ~, >= 等）は使用しない |
| **Composer 互換性** | 非互換。Composer との相互運用は行わない |

---

## 参照標準

| 標準ID | 名称 | 概要 |
|--------|------|------|
| **AFE-STD-105** | Package Manager Interface | パッケージマネージャの抽象化インターフェース定義 |
| **AFE-STD-106** | Package Repository Interface | パッケージリポジトリの抽象化インターフェース定義 |
| **AFE-STD-107** | Package Configuration Standard | afe.json / afe.lock の設定ファイル仕様 |

詳細は [`/AFE-STD/standards/`](../AFE-STD/standards/) を参照すること。

---

## まとめ

- ✅ **Phase 0 完了**: パッケージマネージャ方針策定完了
- 🔜 **Phase 1 以降**: コア機能実装（2026 Q2 予定）
- ❌ **サードパーティ統合**: 禁止（社内パッケージのみ）
- ✅ **常に最新版**: 固定バージョンまたは "latest" のみ
- ✅ **AFE-STD 準拠**: AFE-STD-105/106/107 に完全準拠

---

> 📖 変更履歴は [`VERSION_HISTORY.md`](./VERSION_HISTORY.md) を参照すること。

---

*本ドキュメントは Adlaire Group の機密情報です。Adlaire Group の書面による許可なく、外部への開示・複製・転用を禁止します。*

*© 2026 Adlaire Group / Adlaire Group DX Division. All Rights Reserved.*
