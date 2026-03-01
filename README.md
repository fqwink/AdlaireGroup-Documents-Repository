# AdlaireGroup Documents Repository

> **AdlaireGroupに関する全プロジェクトのドキュメントを一元管理するリポジトリ**

---

| 項目 | 内容 |
|------|------|
| **リポジトリ名** | AdlaireGroup-Documents-Repository |
| **所有管理組織** | Adlaire Group |
| **管理部門** | 組織経営管理セグメント / Adlaire Group DX Division |
| **機密レベル** | 社外秘 |
| **最終更新日** | 2026-03-01 |

---

## 📖 リポジトリ概要

本リポジトリは、**Adlaire Group** が管理・運営するすべてのプロジェクトに関するドキュメントを一元的に管理するための中央リポジトリです。

### 目的

- プロジェクト横断的なドキュメント管理の効率化
- グループ全体の知識・情報の資産化
- プロジェクト間の標準化と品質向上
- ドキュメントへのアクセス性と可視性の向上

---

## 📁 リポジトリ構造

```
AdlaireGroup-Documents-Repository/
├── README.md              # 本ファイル（リポジトリ全体概要）
├── LICENSE                # AdlaireGroup共通ライセンス
├── .gitignore             # Git除外設定
│
└── AFE/                   # Adlaire Framework Ecosystem
    ├── README.md
    ├── PROJECT_CHARTER.md
    ├── FRAMEWORK_SPECIFICATION.md
    ├── VERSION_HISTORY.md
    ├── VERSIONING.md
    └── status-guideline.md
```

### 今後の拡張

新規プロジェクトは、プロジェクト略称をフォルダ名としてルート直下に追加されます。

```
AdlaireGroup-Documents-Repository/
├── AFE/                   # Adlaire Framework Ecosystem
├── XXX/                   # 新規プロジェクト1（略称）
└── YYY/                   # 新規プロジェクト2（略称）
```

---

## 🚀 管理対象プロジェクト一覧

### 稼働中プロジェクト

| 略称 | プロジェクト名 | ステータス | フェーズ | ドキュメントフォルダ |
|------|--------------|----------|---------|------------------|
| **AFE** | Adlaire Framework Ecosystem | ✅ 稼働中 | Phase 0 完了 | [`/AFE/`](./AFE/) |

### 今後追加予定

今後、新規プロジェクトが追加される予定です。

---

## 📚 プロジェクト詳細

### [AFE - Adlaire Framework Ecosystem](./AFE/)

**PHP/Go言語ベースの独自フレームワーク開発プロジェクト**

- **目的**: Adlaire Group全体で利用可能な共通基盤フレームワークの開発
- **技術スタック**: PHP 8.2+、Go（将来）
- **現フェーズ**: Phase 0（ドキュメント整備フェーズ）✅ 完了
- **次フェーズ**: Phase 1（コアアーキテクチャ設計）- 2026 Q2予定

#### 主要ドキュメント

| ドキュメント | ファイル | 説明 |
|------------|---------|------|
| プロジェクト概要 | [`AFE/README.md`](./AFE/README.md) | AFEプロジェクト全体概要 |
| プロジェクト憲章 | [`AFE/PROJECT_CHARTER.md`](./AFE/PROJECT_CHARTER.md) | プロジェクトの権限・目的・背景 |
| フレームワーク仕様書 | [`AFE/FRAMEWORK_SPECIFICATION.md`](./AFE/FRAMEWORK_SPECIFICATION.md) | 技術仕様・設計思想 |
| バージョン履歴 | [`AFE/VERSION_HISTORY.md`](./AFE/VERSION_HISTORY.md) | 変更履歴管理 |
| バージョニング規則 | [`AFE/VERSIONING.md`](./AFE/VERSIONING.md) | バージョン表記ルール |
| ステータスガイドライン | [`AFE/status-guideline.md`](./AFE/status-guideline.md) | ステータス記号定義 |

---

## 🔒 アクセス権限・機密管理

### 機密レベル

本リポジトリおよび配下のすべてのドキュメントは **社外秘** です。

### アクセス権限

| 権限区分 | 対象者 |
|---------|-------|
| **閲覧権限** | Adlaire Group 全従業員 |
| **編集権限** | 各プロジェクトの開発・管理担当者 |
| **承認権限** | プロジェクトオーナー、経営管理セグメント |
| **所有権** | 組織経営管理セグメント ゼネラルマネージャー兼 Adlaire Group DX事業セグメントグループ ゼネラルマネージャー 倉田 和宏 |

### 機密保持

- Adlaire Groupの書面による許可なく、外部への開示・複製・転用を禁止します
- 本リポジトリの内容を外部に公開する場合は、必ず事前承認を得ること

---

## 📝 ドキュメント作成・更新ガイドライン

### 新規プロジェクトの追加

1. **プロジェクト略称でフォルダを作成**
   - 例: `DXP/`, `IMS/` など
   - ルート直下に配置

2. **最低限必要なドキュメントを作成**
   - `README.md` - プロジェクト概要
   - `PROJECT_CHARTER.md` - プロジェクト憲章（推奨）

3. **本README.mdを更新**
   - プロジェクト一覧テーブルに追加
   - プロジェクト詳細セクションに説明追加

### ドキュメント更新時の注意事項

- **バージョン管理**: 重要な変更は必ず `VERSION_HISTORY.md` に記録
- **相互参照**: 関連ドキュメント間のリンクを適切に設定
- **ステータス表記**: AFEプロジェクトの `status-guideline.md` を参考に適切なステータス記号を使用
- **コミットメッセージ**: 変更内容を明確に記述

---

## 🛠️ 技術情報

### リポジトリ管理

| 項目 | 内容 |
|------|------|
| **バージョン管理** | Git |
| **ホスティング** | GitHub |
| **リポジトリURL** | https://github.com/fqwink/AdlaireGroup-Documents-Repository |
| **デフォルトブランチ** | `main` |

### ブランチ戦略

- **main**: 本番ブランチ（承認済みドキュメント）
- **genspark_ai_developer**: AI開発支援用ブランチ
- **feature/***: 各種開発・更新作業用ブランチ

---

## 📞 お問い合わせ

ドキュメントに関する質問・提案・問題報告は、以下にお問い合わせください：

- **管理部門**: Adlaire Group DX Division
- **責任者**: 組織経営管理セグメント ゼネラルマネージャー兼 Adlaire Group DX事業セグメントグループ ゼネラルマネージャー 倉田 和宏

---

## 📄 ライセンス

本リポジトリのライセンス情報は [`LICENSE`](./LICENSE) を参照してください。

---

*本リポジトリおよび配下のすべてのドキュメントは Adlaire Group の機密情報です。Adlaire Group の書面による許可なく、外部への開示・複製・転用を禁止します。*

*© 2026 Adlaire Group. All Rights Reserved.*
