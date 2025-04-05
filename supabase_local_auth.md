# ✅ Supabaseローカル + OAuth（Googleログイン）のベストプラクティス

Supabaseをローカル開発環境で使っていると、「あれ？Googleログインってどうやって設定するの？」「本番とローカルで挙動が違って困る…」っていうこと、ありませんか？このドキュメントは、そんな悩みに寄り添って、**ローカルと本番の差異をすべて環境変数で吸収する構成**を提案します。

これを導入すれば、Authまわりの面倒な切り替えを `.env` だけでスッキリ管理できて、トラブルにも強くなれます。

以下、構成・具体例・CI対応・注意点をすべて含んだ完全版です。

---

## ✅ なぜこの運用が必要か？

Supabase の Auth 設定は以下の点でローカルと本番に違いがあります：

| 設定項目 | ローカル | 本番 |
|----------|----------|------|
| `site_url` | `http://localhost:3000` | `https://yourdomain.com` |
| `redirect_uri` | `http://localhost:54321/auth/v1/callback` | `https://<project>.supabase.co/auth/v1/callback` |
| `skip_nonce_check` | `true`（開発の都合） | `false`（セキュリティ重視） |
| Google OAuthのクレデンシャル | 開発用 | 本番用 |

この差分を `config.toml` に直書きすると、管理が破綻する危険があります。**すべての差分を環境変数で切り替え可能にすることが、最も安全で再現性の高い構成**です。

---

## ✅ ファイル構成例

```sh
supabase/
├── config.toml
├── .env.local        # ローカル用のOAuth設定
├── .env.production   # 本番用のOAuth設定（CI/CDや本番デプロイ用）
```

---

## ✅ `supabase/config.toml` の書き方（すべて環境変数で吸収）

```toml
[auth]
site_url = "env(SITE_URL)"
additional_redirect_urls = ["env(ADDITIONAL_REDIRECT_URLS)"]

[auth.external.google]
enabled = true
client_id = "env(GOOGLE_CLIENT_ID)"
secret = "env(GOOGLE_CLIENT_SECRET)"
redirect_uri = "env(GOOGLE_REDIRECT_URI)"
skip_nonce_check = "env(SKIP_NONCE_CHECK)"
```

- `env(...)` を使うことで Supabase CLI が `.env` ファイルを読み取り、値を自動で補完します。

---

## ✅ `.env.local`（ローカル開発用）

```env
SITE_URL=http://localhost:3000
ADDITIONAL_REDIRECT_URLS=http://localhost:3000
GOOGLE_REDIRECT_URI=http://localhost:54321/auth/v1/callback
GOOGLE_CLIENT_ID=your-local-google-client-id.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=your-local-secret
SKIP_NONCE_CHECK=true
```

---

## ✅ `.env.production`（本番用）

```env
SITE_URL=https://your-production-site.com
ADDITIONAL_REDIRECT_URLS=https://your-production-site.com
GOOGLE_REDIRECT_URI=https://yourproject.supabase.co/auth/v1/callback
GOOGLE_CLIENT_ID=your-prod-client-id.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=your-prod-secret
SKIP_NONCE_CHECK=false
```

---

## ✅ Google Cloud Console 側の設定

1. プロジェクト作成 → OAuth同意画面設定（テストユーザー可）
2. OAuthクライアントID作成（種類は「ウェブアプリ」）
3. **承認済みのリダイレクト URI** に次を登録：
   - ローカル: `http://localhost:54321/auth/v1/callback`
   - 本番: `https://yourproject.supabase.co/auth/v1/callback`
4. **承認済みのJavaScriptオリジン** に次を登録：
   - ローカル: `http://localhost:3000`
   - 本番: `https://your-production-site.com`
5. クライアントID・シークレットを `.env` に登録

---

## ✅ フロントエンドでの呼び出し例

```ts
const { data, error } = await supabase.auth.signInWithOAuth({
  provider: 'google',
  options: {
    redirectTo: `${window.location.origin}/auth/callback`,
  }
})
```

`redirectTo` を明示することで、ログイン後の遷移先をカスタマイズ可能。

---

## ✅ 開発中のメール確認：Inbucket

- `http://localhost:54324` にアクセスして、確認メールを開いてリンクをクリック
- `config.toml` で `enable_confirmations = true` にすると、メール確認が必要になります

---

## ✅ CI/CD・環境切り替え戦略

### ローカル:
```bash
cp .env.local .env
supabase start
```

### 本番CI/CD（例: GitHub Actions）:
```yaml
env:
  SITE_URL: ${{ secrets.SITE_URL }}
  ADDITIONAL_REDIRECT_URLS: ${{ secrets.ADDITIONAL_REDIRECT_URLS }}
  GOOGLE_CLIENT_ID: ${{ secrets.GOOGLE_CLIENT_ID }}
  GOOGLE_CLIENT_SECRET: ${{ secrets.GOOGLE_CLIENT_SECRET }}
  GOOGLE_REDIRECT_URI: ${{ secrets.GOOGLE_REDIRECT_URI }}
  SKIP_NONCE_CHECK: "false"
```

---

## ✅ まとめ

| 項目 | ローカル (.env.local) | 本番 (.env.production) |
|------|------------------------|--------------------------|
| `SITE_URL` | `http://localhost:3000` | `https://yourdomain.com` |
| `GOOGLE_REDIRECT_URI` | `http://localhost:54321/auth/v1/callback` | `https://<project>.supabase.co/auth/v1/callback` |
| `SKIP_NONCE_CHECK` | `true`（開発のため） | `false`（セキュリティ強化） |

このベストプラクティス構成により：

- `config.toml` は共通・一枚岩になる
- `.env` の切り替えだけでローカル〜本番対応が可能
- セキュリティとメンテナンス性を両立した運用が実現

---

## ✅ 補足と拡張性

- Firebase、GitHub、Apple など他のOAuthプロバイダも `[auth.external.X]` を追加して、同様に環境変数化できます
- `enable_confirmations` やメールテンプレートの切り替えも `.env` で吸収可能
- `skip_nonce_check` はローカルのみ。**本番では必ず false に戻すこと**

---

本番とローカルで挙動が違って困る……そんな悩みを `.env` で丸ごと解決できる、この構成。

**SupabaseでOAuthを運用するすべての開発者におすすめできる最適解です。**

