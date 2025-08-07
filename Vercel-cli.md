# Vercel CLI完全教科書

## 目次

1. [Vercel CLIの概要と特徴](#1-vercel-cliの概要と特徴)
2. [インストール方法](#2-インストール方法)
3. [初期設定とログイン](#3-初期設定とログイン)
4. [基本的なコマンドの使い方](#4-基本的なコマンドの使い方)
5. [プロジェクトのデプロイ方法](#5-プロジェクトのデプロイ方法)
6. [環境変数の管理](#6-環境変数の管理)
7. [ドメインとDNSの設定](#7-ドメインとdnsの設定)
8. [ログとモニタリング](#8-ログとモニタリング)
9. [チームとプロジェクト管理](#9-チームとプロジェクト管理)
10. [高度な使い方とTips](#10-高度な使い方とtips)
11. [トラブルシューティング](#11-トラブルシューティング)
12. [クイックリファレンス](#12-クイックリファレンス)

---

## 1. Vercel CLIの概要と特徴

### Vercel CLIとは

Vercel CLIは、Vercelプラットフォームをコマンドラインから操作するための公式ツールです。Webアプリケーションの開発、デプロイ、管理を効率的に行うことができます。

### 主な特徴

- **高速デプロイ**: プロジェクトを数秒でデプロイ可能
- **自動スケーリング**: トラフィックに応じて自動的にスケール
- **プレビューデプロイ**: ブランチごとのプレビューURL生成
- **Edge Network**: 世界中のCDNで高速配信
- **簡単な設定**: 設定ファイル不要で多くのフレームワークに対応

### 対応フレームワーク

- Next.js
- React
- Vue.js
- Nuxt.js
- Angular
- Svelte
- Static HTML
- Node.js API

---

## 2. インストール方法

### 2.1 npmを使用したインストール

```bash
# グローバルインストール
npm install -g vercel

# プロジェクトローカルにインストール
npm install --save-dev vercel
```

### 2.2 yarnを使用したインストール

```bash
# グローバルインストール
yarn global add vercel

# プロジェクトローカルにインストール
yarn add --dev vercel
```

### 2.3 pnpmを使用したインストール

```bash
# グローバルインストール
pnpm add -g vercel

# プロジェクトローカルにインストール
pnpm add -D vercel
```

### 2.4 OS別の直接インストール

#### macOS (Homebrew)

```bash
brew install vercel-cli
```

#### Windows (Chocolatey)

```bash
choco install vercel-cli
```

#### Linux (Snap)

```bash
sudo snap install vercel
```

### 2.5 インストール確認

```bash
vercel --version
# または
vc --version
```

---

## 3. 初期設定とログイン

### 3.1 初回ログイン

```bash
vercel login
```

このコマンドを実行すると、以下の選択肢が表示されます：

1. **Email**: メールアドレスでログイン
2. **GitHub**: GitHubアカウントでログイン
3. **GitLab**: GitLabアカウントでログイン
4. **Bitbucket**: Bitbucketアカウントでログイン

### 3.2 メールログインの例

```bash
$ vercel login
> Log in to Vercel
? How would you like to log in? 
❯ Continue with Email
  Continue with GitHub
  Continue with GitLab
  Continue with Bitbucket

? Enter your email address: user@example.com
> We sent an email to user@example.com. Please follow the steps provided inside it and make sure the security code matches Proud Unicorn.
```

### 3.3 認証の確認

```bash
vercel whoami
```

### 3.4 設定の確認

```bash
# 現在の設定を表示
vercel ls

# アカウント情報を表示
vercel teams
```

---

## 4. 基本的なコマンドの使い方

### 4.1 ヘルプの表示

```bash
# 全般的なヘルプ
vercel --help

# 特定のコマンドのヘルプ
vercel deploy --help
```

### 4.2 プロジェクトの初期化

```bash
# 新しいプロジェクトを作成
vercel init

# テンプレートから作成
vercel init my-app --example nextjs
```

### 4.3 プロジェクト情報の表示

```bash
# プロジェクト一覧
vercel ls

# 特定のプロジェクトの詳細
vercel inspect [deployment-url]
```

### 4.4 デプロイの管理

```bash
# 最新のデプロイを表示
vercel ls

# デプロイ履歴を表示
vercel ls --scope team-name

# デプロイを削除
vercel remove [deployment-url]
```

---

## 5. プロジェクトのデプロイ方法

### 5.1 基本のデプロイ

```bash
# プロジェクトディレクトリで実行
vercel

# または短縮形
vc
```

### 5.2 初回デプロイの詳細フロー

```bash
$ vercel
? Set up and deploy "~/my-project"? [Y/n] y
? Which scope do you want to deploy to? My Account
? Link to existing project? [y/N] n
? What's your project's name? my-project
? In which directory is your code located? ./
Auto-detected Project Settings (Next.js):
- Build Command: next build
- Output Directory: .next
- Development Command: next dev --port $PORT
? Want to override the settings? [y/N] n

🔗 Linked to myaccount/my-project (created .vercel folder)
🔍 Inspect: https://vercel.com/myaccount/my-project/xxx
✅ Preview: https://my-project-xxx.vercel.app
📝 To deploy to production (my-project.vercel.app), run `vercel --prod`
```

### 5.3 本番環境へのデプロイ

```bash
# 本番環境にデプロイ
vercel --prod

# または
vercel deploy --prod
```

### 5.4 特定のディレクトリからデプロイ

```bash
# 特定のディレクトリを指定
vercel ./dist --prod

# ビルド出力ディレクトリを直接デプロイ
vercel ./build
```

### 5.5 デプロイメントの設定

```bash
# 環境を指定してデプロイ
vercel --target production
vercel --target preview

# 特定のブランチからデプロイ
vercel --prod --force
```

---

## 6. 環境変数の管理

### 6.1 環境変数の追加

```bash
# 環境変数を追加
vercel env add

# 例：API_KEYを追加
$ vercel env add
? What's the name of the variable? API_KEY
? What's the value of API_KEY? secret-key-value
? Add API_KEY to which Environments? (Press <space> to select)
❯◉ Production
 ◉ Preview
 ◉ Development
```

### 6.2 環境変数の一覧表示

```bash
# 全ての環境変数を表示
vercel env ls

# 特定の環境の変数のみ表示
vercel env ls production
vercel env ls preview
vercel env ls development
```

### 6.3 環境変数の削除

```bash
# 環境変数を削除
vercel env rm API_KEY production
```

### 6.4 環境変数の更新

```bash
# 既存の環境変数を更新
vercel env add API_KEY  # 同じ名前で追加すると更新される
```

### 6.5 .envファイルからの一括追加

```bash
# .envファイルから環境変数を読み込んで追加
vercel env pull .env.local
```

### 6.6 環境変数の利用例

**.env.local**

```env
DATABASE_URL=postgresql://user:pass@localhost/mydb
API_KEY=your-secret-api-key
NEXT_PUBLIC_SITE_URL=https://example.com
```

**Next.jsでの使用例**

```javascript
// pages/api/data.js
export default function handler(req, res) {
  const apiKey = process.env.API_KEY;
  const dbUrl = process.env.DATABASE_URL;
  
  // APIロジック
}
```

---

## 7. ドメインとDNSの設定

### 7.1 カスタムドメインの追加

```bash
# ドメインを追加
vercel domains add example.com

# プロジェクトにドメインを割り当て
vercel domains add example.com --scope my-project
```

### 7.2 ドメインの一覧表示

```bash
# 全てのドメインを表示
vercel domains ls

# 特定のプロジェクトのドメインを表示
vercel domains ls --scope my-project
```

### 7.3 ドメインの削除

```bash
# ドメインを削除
vercel domains rm example.com
```

### 7.4 DNSレコードの管理

```bash
# DNSレコードを追加
vercel dns add example.com A 192.0.2.1

# CNAMEレコードを追加
vercel dns add www.example.com CNAME example.com

# MXレコードを追加
vercel dns add example.com MX "10 mail.example.com"
```

### 7.5 DNSレコードの一覧表示

```bash
# ドメインのDNSレコードを表示
vercel dns ls example.com
```

### 7.6 DNSレコードの削除

```bash
# DNSレコードを削除
vercel dns rm example.com A 192.0.2.1
```

### 7.7 SSL証明書の管理

```bash
# SSL証明書の状態を確認
vercel certs ls

# カスタム証明書をアップロード
vercel certs add example.com --cert cert.pem --key key.pem --ca ca.pem
```

---

## 8. ログとモニタリング

### 8.1 デプロイメントログの確認

```bash
# 最新のデプロイメントログを表示
vercel logs

# 特定のデプロイメントのログを表示
vercel logs https://my-app-xxx.vercel.app

# リアルタイムでログを追跡
vercel logs --follow
```

### 8.2 関数ログの確認

```bash
# API関数のログを表示
vercel logs --follow --output=raw
```

### 8.3 ビルドログの確認

```bash
# ビルドプロセスの詳細ログ
vercel inspect https://my-app-xxx.vercel.app
```

### 8.4 パフォーマンス情報

```bash
# デプロイメントの詳細情報を表示
vercel inspect [deployment-url] --wait
```

### 8.5 ログのフィルタリング

```bash
# エラーログのみ表示
vercel logs --since=1h --filter=error

# 特定の時間範囲のログ
vercel logs --since=2023-01-01 --until=2023-01-02
```

---

## 9. チームとプロジェクト管理

### 9.1 チームの作成と管理

```bash
# 新しいチームを作成
vercel teams add my-team

# チーム一覧を表示
vercel teams ls

# チームを切り替え
vercel teams switch my-team
```

### 9.2 チームメンバーの管理

```bash
# メンバーを招待
vercel teams invite user@example.com my-team

# メンバー一覧を表示
vercel teams members my-team

# メンバーを削除
vercel teams remove user@example.com my-team
```

### 9.3 プロジェクトの管理

```bash
# プロジェクト一覧を表示
vercel projects ls

# プロジェクトの詳細情報
vercel projects inspect my-project

# プロジェクトを削除
vercel projects rm my-project
```

### 9.4 プロジェクトの設定

```bash
# プロジェクト設定を表示
vercel project

# ビルド設定を変更
vercel project --build-command "npm run build:prod"
vercel project --output-directory "build"
```

### 9.5 権限管理

```bash
# プロジェクトの権限を設定
vercel project --public    # パブリックに設定
vercel project --private   # プライベートに設定
```

---

## 10. 高度な使い方とTips

### 10.1 vercel.jsonの設定

**vercel.json**の基本構成：

```json
{
  "version": 2,
  "builds": [
    {
      "src": "package.json",
      "use": "@vercel/static-build",
      "config": {
        "distDir": "build"
      }
    }
  ],
  "routes": [
    {
      "src": "/api/(.*)",
      "dest": "/api/$1"
    }
  ],
  "env": {
    "NODE_ENV": "production"
  },
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        {
          "key": "X-Frame-Options",
          "value": "DENY"
        }
      ]
    }
  ]
}
```

### 10.2 カスタムビルドコマンド

```json
{
  "scripts": {
    "build": "next build && next export"
  },
  "buildCommand": "npm run build",
  "outputDirectory": "out"
}
```

### 10.3 リダイレクトの設定

```json
{
  "redirects": [
    {
      "source": "/old-page",
      "destination": "/new-page",
      "permanent": true
    },
    {
      "source": "/blog/:slug",
      "destination": "/posts/:slug",
      "permanent": false
    }
  ]
}
```

### 10.4 プレビューデプロイの活用

```bash
# ブランチ名を指定してプレビューデプロイ
vercel --target preview --git-branch feature/new-ui

# プルリクエスト用デプロイ
vercel --target preview --meta githubCommitRef=refs/pull/123/merge
```

### 10.5 シークレット管理

```bash
# シークレットを追加
vercel secrets add my-secret "secret-value"

# シークレット一覧を表示
vercel secrets ls

# 環境変数でシークレットを使用
vercel env add API_KEY @my-secret production
```

### 10.6 デプロイフック

```bash
# デプロイフックを作成
vercel deploy --prod --force --with-hook

# webhookからデプロイをトリガー
curl -X POST https://api.vercel.com/v1/integrations/deploy/...
```

---

## 11. トラブルシューティング

### 11.1 よくある問題と解決方法

#### ビルドエラー

```bash
# 詳細なビルドログを確認
vercel logs --follow --output=raw

# ローカルでビルドテスト
npm run build

# 依存関係の確認
npm audit
npm update
```

#### 環境変数が反映されない

```bash
# 環境変数を確認
vercel env ls

# 正しい環境に設定されているか確認
vercel env ls production

# キャッシュをクリア
vercel --force
```

#### ドメインが正しく設定されない

```bash
# DNS設定を確認
vercel dns ls example.com

# ドメイン設定を確認
vercel domains ls

# DNSの伝播を確認
nslookup example.com
```

### 11.2 デバッグ方法

```bash
# デバッグモードで実行
DEBUG=* vercel

# 詳細なログを出力
vercel --debug

# 特定のコマンドをデバッグ
DEBUG=vercel:* vercel deploy
```

### 11.3 キャッシュの問題

```bash
# キャッシュをクリアしてデプロイ
vercel --force

# npm/yarnキャッシュをクリア
npm cache clean --force
yarn cache clean
```

### 11.4 認証の問題

```bash
# ログアウトして再ログイン
vercel logout
vercel login

# トークンを直接設定
vercel login --token your-token
```

### 11.5 パフォーマンスの問題

```bash
# バンドルサイズを確認
npm run build
npx bundlephobia

# 最適化の提案を確認
vercel inspect [deployment-url]
```

---

## 12. クイックリファレンス

### 12.1 基本コマンド

| コマンド                 | 説明                   |
| ------------------------ | ---------------------- |
| `vercel`               | プロジェクトをデプロイ |
| `vercel --prod`        | 本番環境にデプロイ     |
| `vercel login`         | ログイン               |
| `vercel logout`        | ログアウト             |
| `vercel whoami`        | 現在のユーザーを表示   |
| `vercel ls`            | デプロイ一覧を表示     |
| `vercel rm [url]`      | デプロイを削除         |
| `vercel inspect [url]` | デプロイ詳細を表示     |

### 12.2 プロジェクト管理

| コマンド                      | 説明                     |
| ----------------------------- | ------------------------ |
| `vercel init`               | プロジェクトを初期化     |
| `vercel link`               | 既存プロジェクトとリンク |
| `vercel projects ls`        | プロジェクト一覧         |
| `vercel projects rm [name]` | プロジェクト削除         |

### 12.3 環境変数

| コマンド                       | 説明                     |
| ------------------------------ | ------------------------ |
| `vercel env add`             | 環境変数を追加           |
| `vercel env ls`              | 環境変数一覧             |
| `vercel env rm [name] [env]` | 環境変数を削除           |
| `vercel env pull [file]`     | 環境変数をファイルに出力 |

### 12.4 ドメインとDNS

| コマンド                                   | 説明            |
| ------------------------------------------ | --------------- |
| `vercel domains add [domain]`            | ドメインを追加  |
| `vercel domains ls`                      | ドメイン一覧    |
| `vercel domains rm [domain]`             | ドメインを削除  |
| `vercel dns add [domain] [type] [value]` | DNSレコード追加 |
| `vercel dns ls [domain]`                 | DNSレコード一覧 |

### 12.5 チーム管理

| コマンド                               | 説明           |
| -------------------------------------- | -------------- |
| `vercel teams ls`                    | チーム一覧     |
| `vercel teams switch [team]`         | チーム切り替え |
| `vercel teams invite [email] [team]` | メンバー招待   |

### 12.6 ログとモニタリング

| コマンド                 | 説明               |
| ------------------------ | ------------------ |
| `vercel logs`          | ログを表示         |
| `vercel logs --follow` | リアルタイムログ   |
| `vercel logs [url]`    | 特定デプロイのログ |

### 12.7 よく使うオプション

| オプション         | 説明                       |
| ------------------ | -------------------------- |
| `--prod`         | 本番環境                   |
| `--force`        | 強制実行（キャッシュ無視） |
| `--debug`        | デバッグモード             |
| `--yes`          | 確認をスキップ             |
| `--scope [team]` | チーム指定                 |
| `--cwd [path]`   | 作業ディレクトリ指定       |

### 12.8 設定ファイルの例

**vercel.json**の基本設定：

```json
{
  "version": 2,
  "name": "my-project",
  "builds": [
    {
      "src": "package.json",
      "use": "@vercel/static-build"
    }
  ],
  "routes": [
    {
      "src": "/(.*)",
      "dest": "/$1"
    }
  ]
}
```

**package.json**のscripts設定：

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "deploy": "vercel --prod",
    "deploy:preview": "vercel"
  }
}
```

---

## まとめ

この教科書では、Vercel CLIの基本的な使い方から高度な機能まで、実践的な例とともに解説しました。Vercel CLIを使いこなすことで、効率的なWebアプリケーションの開発・デプロイワークフローを構築できます。

継続的な学習のために：

1. **公式ドキュメント**を定期的に確認
2. **コミュニティ**で最新情報をキャッチアップ
3. **実際のプロジェクト**で練習を重ねる
4. **チーム開発**でのベストプラクティスを学ぶ

Vercel CLIを活用して、素晴らしいWebアプリケーションを世界に届けましょう！
