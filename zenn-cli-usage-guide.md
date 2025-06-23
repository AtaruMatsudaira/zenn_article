# Zenn CLI 使い方ガイド

## 概要

Zenn CLIは、Zennの記事や本をMarkdownファイルとGitHubリポジトリを使って管理するためのツールです。ローカル環境でコンテンツを作成・編集し、GitHubと連携して公開できます。

## 前提条件

- Node.js 14以上が必要
- GitHubアカウントとリポジトリの準備を推奨

## インストール

### 1. プロジェクトの初期化

```bash
# パッケージ管理ファイルの作成（デフォルト設定）
npm init --yes

# Zenn CLIのインストール
npm install zenn-cli
```

### 2. Zennプロジェクトの初期化

```bash
npx zenn init
```

このコマンドにより以下が作成されます：
- `README.md`
- `.gitignore`
- `articles/` ディレクトリ（記事用）
- `books/` ディレクトリ（本用）

## 主要コマンド

### 基本コマンド

| コマンド | 説明 |
|---------|------|
| `npx zenn init` | プロジェクト構造を初期化 |
| `npx zenn preview` | ローカルプレビューサーバーを起動 |
| `npx zenn new:article` | 新しい記事ファイルを作成 |
| `npx zenn new:book` | 新しい本のディレクトリを作成 |

### CLI更新

```bash
# 通常の更新
npm update zenn-cli

# 最新版への更新
npm install zenn-cli@latest
```

## 記事管理

### 記事の作成

```bash
npx zenn new:article
```

`articles/` ディレクトリに以下のような形式でファイルが作成されます：

```yaml
---
title: ""
emoji: "😸"
type: "tech" # "tech" または "idea"
topics: []
published: true
---

# ここに記事の内容を書く
```

### 記事のファイル構造

- **ファイル名**: `articles/記事のスラッグ.md`
- **Front Matter**: YAMLヘッダーでメタデータを定義
- **本文**: Markdownで記述

### 記事の設定項目

| 項目 | 説明 | 例 |
|------|------|-----|
| `title` | 記事のタイトル | `"Zenn CLIの使い方"` |
| `emoji` | 記事のアイコン | `"📝"` |
| `type` | 記事の種類 | `"tech"` または `"idea"` |
| `topics` | タグ | `["zenn", "cli", "markdown"]` |
| `published` | 公開状態 | `true` または `false` |

## 本管理

### 本の作成

```bash
npx zenn new:book
```

### 本のディレクトリ構造

```
books/
└── book-slug/
    ├── config.yaml      # 本の設定
    ├── cover.png        # カバー画像
    ├── chapter1.md      # 第1章
    ├── chapter2.md      # 第2章
    └── ...
```

### 本の設定ファイル (config.yaml)

```yaml
title: "本のタイトル"
summary: "本の概要"
topics: ["topic1", "topic2"]
published: true
price: 0 # 0=無料, 200以上=有料
chapters:
  - chapter1
  - chapter2
```

### 制限事項

- 1つの本につき最大100章まで
- 有料本の場合、最低価格は200円

## プレビュー機能

### ローカルプレビューの起動

```bash
npx zenn preview
```

デフォルトでは `http://localhost:8000` でプレビューサーバーが起動します。

### プレビューの特徴

- リアルタイムでの変更反映
- Zenn.devと同じ見た目で確認可能
- 記事と本の両方をプレビュー可能

## 公開ワークフロー

### GitHub連携による公開

1. **リポジトリの準備**
   - GitHubでリポジトリを作成
   - Zenn.devでGitHub連携を設定

2. **コンテンツの作成**
   ```bash
   npx zenn new:article
   # または
   npx zenn new:book
   ```

3. **ローカルプレビュー**
   ```bash
   npx zenn preview
   ```

4. **GitHubにプッシュ**
   ```bash
   git add .
   git commit -m "新しい記事を追加"
   git push origin main
   ```

5. **自動同期**
   - GitHubにプッシュすると自動的にZenn.devに反映

### 公開設定

- `published: true` で即座に公開
- `published: false` で下書き状態
- 公開日時の予約投稿も可能

## ベストプラクティス

### ファイル管理

- 記事のスラッグは分かりやすい名前に変更推奨
- 画像は `images/` ディレクトリで管理
- 定期的なバックアップとしてGitを活用

### 執筆環境

- お好みのMarkdownエディタで執筆
- プレビューサーバーで確認しながら執筆
- 段階的にコミットして履歴を管理

### SEO対策

- `topics` に適切なタグを設定
- タイトルと概要を充実させる
- 画像のalt属性を適切に設定

## トラブルシューティング

### よくある問題

1. **Node.jsのバージョンエラー**
   - Node.js 14以上にアップデート

2. **プレビューが表示されない**
   - ポート8000が使用中でないか確認
   - ファイアウォール設定を確認

3. **GitHubとの同期がされない**
   - Zenn.devでのGitHub連携設定を確認
   - リポジトリの公開設定を確認

4. **Markdownの表示がおかしい**
   - Zenn独自の記法を確認
   - プレビューで事前確認

## 参考リンク

- [Zenn CLI公式ガイド](https://zenn.dev/zenn/articles/zenn-cli-guide)
- [Zenn CLI NPMパッケージ](https://www.npmjs.com/package/zenn-cli)
- [Zenn開発者向けドキュメント](https://zenn-dev.github.io/zenn-docs-for-developers/)
- [ZennのMarkdown記法](https://zenn.dev/zenn/articles/markdown-guide)

## まとめ

Zenn CLIを使うことで、Git管理されたMarkdownファイルベースで効率的にコンテンツを作成・管理できます。ローカル環境での執筆とGitHubを通じた公開により、開発者にとって馴染みのあるワークフローでZennコンテンツを運用できます。