## ✅ Supabaseローカル + OAuth（Googleログイン）のベストプラクティス

### 1. **Google OAuthを正しく設定（Google Cloud側）**
- **OAuthクライアントID & シークレットを作成**（種類は「ウェブアプリ」）
- **承認済みリダイレクトURI** に以下を必ず追加：
  ```
  http://localhost:54321/auth/v1/callback
  ```
- **承認済みJavaScriptオリジン** にフロントエンドのURL（例: `http://localhost:3000`）を追加

### 2. **`.env` に機密情報を記述（Gitに含めない！）**
```env
GOOGLE_CLIENT_ID=your-client-id.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=your-secret
```

### 3. **`supabase/config.toml` を正しく設定**
```toml
[auth]
site_url = "http://localhost:3000"
additional_redirect_urls = ["http://localhost:3000"]

[auth.external.google]
enabled = true
client_id = "env(GOOGLE_CLIENT_ID)"
secret = "env(GOOGLE_CLIENT_SECRET)"
redirect_uri = "http://localhost:54321/auth/v1/callback"
skip_nonce_check = true  # ローカル環境では true にしておく
```

> 🔐 **skip_nonce_check は本番では false に戻すのを忘れずに！**

### 4. **Supabaseを再起動して設定反映**
```bash
supabase stop
supabase start
```

### 5. **フロントエンドではリダイレクトURLを明示**
```js
await supabase.auth.signInWithOAuth({
  provider: 'google',
  options: {
    redirectTo: 'http://localhost:3000/auth/callback', // 必要に応じて設定
  }
})
```

### 6. **メール確認が必要なら Inbucket を使う**
- ブラウザで `http://localhost:54324` にアクセスしてメール内容確認
- `[auth.email] enable_confirmations = true` を設定すれば本番と同様のメール確認フローも再現可能

---

## 🔁 本番移行時のチェックポイント

| 項目 | ローカル | 本番（Supabase Cloud） |
|------|----------|-------------------------|
| `site_url`       | `http://localhost:3000`       | `https://yourdomain.com`または `.supabase.co` |
| `redirect_uri`   | `http://localhost:54321/auth/v1/callback` | `https://<project>.supabase.co/auth/v1/callback` |
| `skip_nonce_check` | `true`（開発用）              | `false`（セキュリティ重視）               |
| SMTPメール        | Inbucket                    | SendGridやMailgun（SMTP設定必須）       |

---

## 🌟 まとめ：ローカルでのベスト運用ルール

| 🔧 項目 | ✅ ベストプラクティス |
|--------|---------------------|
| OAuth設定 | Google Cloud + `.env` + `config.toml` |
| URL整合性 | `localhost` vs `127.0.0.1` は揃える！ |
| セキュリティ | `.env` で秘密管理 / `skip_nonce_check` の戻し忘れ注意 |
| メール確認 | Inbucket を活用。開発時は十分。 |
| 移植性 | `config.toml`ベースの運用で本番移行もスムーズに |
