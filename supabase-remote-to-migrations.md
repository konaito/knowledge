# Supabase Remote スキーマを Local Migrations に反映する方法（2025年 CLI 対応）

## 📖 背景

Supabase では「マイグレーションファイルを正」としてローカルからスキーマを管理するのが基本方針です。  
しかし現実には以下のようなケースが発生します：

- Studio 上で直接テーブル・カラムを変更してしまった
- Supabase CLI を使っていない開発者と連携したい

これらの場合、**Remote の状態をマイグレーションとして復元する方法**として `supabase db pull` が有効です。

---

## 🚀 やることはこれだけ！

```bash
supabase db pull
```

このコマンドを実行することで：

- Remote の現在のスキーマが `supabase/migrations/` 以下に `yyyymmdd_timestamp_remote_schema.sql` 形式で出力されます。
- `schema_migrations` テーブルの履歴も自動で修復されます（CLI により `Repaired migration history:` が表示されます）。

---

## 🧠 どういうときに使う？

### ✅ ユースケース1: 本番をうっかり直接いじってしまった！

- Studio 上で直接テーブルやカラムを追加
- だがローカル `migrations/` には記録されていない

→ `supabase db pull` を実行すれば、**差分がマイグレーションとして復元され、git 管理可能に**

---

### ✅ ユースケース2: Supabase CLI を使っていない人との連携

- チームの一部が Studio だけで開発している
- CLI 派と Studio 派でスキーマの一貫性がない

→ リーダーが `supabase db pull` を使って remote を取り込み、**統一されたマイグレーション履歴を生成**

---

## 📁 ファイル構成の例

```bash
supabase/
├── migrations/
│   ├── 20250412023324_remote_schema.sql  # ← supabase db pull により生成
├── config.toml
```

> 補足：`auth` や `storage` スキーマは除外されています。必要に応じて以下を実行：
>
> ```bash
> supabase db pull --schema auth,storage
> ```

---

## ⚠️ 注意点

- `supabase db pull` は **Remote の現在の状態をそのまま migrations として取り込みます**  
  → Studio 上でのすべての変更が反映されます
- CLI のバージョンによって動作が異なるため、**最新の CLI（v1.0 以上）を推奨**
- `supabase db reset` や `diff` は、特別な理由がない限り不要です（例：差分のみ抽出したい場合など）

---

## ✅ 結論

| 操作 | 意味 |
|------|------|
| `supabase db pull` | Remote の状態をマイグレーションとして復元 |
| → git commit | 以後の開発・本番反映が再現可能に |

Studio 操作も安全にバージョン管理できるようになりました ✨
