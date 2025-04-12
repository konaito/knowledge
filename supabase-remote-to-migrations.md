# Supabase Remote スキーマを Local Migrations に反映する方法

## 📖 背景

Supabase では、本来「**マイグレーションファイル（`migrations/`）を正とし、ローカルからスキーマを管理する**」という方針が推奨されています。  
しかし、現実には以下のようなケースが発生します：

- クラウド上（Supabase Studio や SQL Editor）でスキーマ変更を直接行ってしまった
- チームメンバーの中に Supabase CLI を使わず、Studio 上だけで開発している人がいる
- 本番環境のスキーマ変更をローカル環境にも適用したい

そのような状況で、**Remote の状態をローカルのマイグレーションファイルとして再構成**する方法を説明します。

---

## 🧭 手順概要

```bash
# ① Remote（本番）のスキーマを取得
supabase db pull

# ② ローカルDBをmigrationsから再構築
supabase db reset

# ③ Remoteとの差分をmigrationファイルとして保存
supabase db diff --file supabase/migrations/20250412_remote_changes.sql
```

---

## 🔍 各コマンドの意味と役割

### 1. `supabase db pull`
- **Remote のスキーマを取得**して、`supabase/schema.sql` に保存
- これはマイグレーションではなく **「現在のスナップショット」**

### 2. `supabase db reset`
- ローカル DB をリセットし、**`migrations/` にあるすべてのファイルを適用**
- これにより「**マイグレーションベースの理想の状態**」をローカルに再現

### 3. `supabase db diff`
- `schema.sql`（Remote の状態）と reset 済みローカル DB の差分を比較
- **差分だけを含む migration ファイル**を生成

---

## 💡 ユースケース別の活用方法

### ✅ ユースケース1: クラウドでうっかり直接変更してしまった

**問題点**：
- Studio で直接テーブルやカラムを追加・変更したが、それをローカルに反映していない  
- `migrations/` に変更履歴が存在しないため、チームでの再現性が失われる

**解決策**：
- `supabase db pull` で現在の状態を取得し、
- `supabase db reset` + `supabase db diff` により、**失われたマイグレーションを生成**

---

### ✅ ユースケース2: Supabase CLI を使っていない開発者との連携

**問題点**：
- 一部のメンバーは Supabase Studio のみを使って開発しており、`migrations/` を管理していない
- スキーマの一貫性や Git による追跡が困難

**解決策**：
- 共同作業の後に `supabase db pull` で最新のスキーマを取得し、
- チームリーダーなどが `db reset` → `db diff` を行い、**統一されたマイグレーションファイルを生成して Git に追加**
- 以降、**migrations ベースでの開発に統一**していくための橋渡しに

---

## 📁 参考ディレクトリ構成例

```
supabase/
├── migrations/
│   ├── 20240301_init.sql
│   ├── 20240401_add_users.sql
│   └── 20250412_remote_changes.sql  ←★ 今回生成される差分
├── schema.sql                       ← pull で取得されるRemoteのスナップショット
├── config.toml
```

---

## 🛑 注意事項

- `supabase db pull` では **マイグレーションファイルは生成されません**
- `schema.sql` を直接編集しないでください（破損の原因になります）
- 差分の内容はレビューしてから Git にコミットすることを推奨します
- `supabase db reset` はローカルDBを初期化します。実行前に必要に応じてバックアップを取りましょう

---

## ✨ まとめ

| 操作                        | 目的                                 |
|-----------------------------|--------------------------------------|
| `supabase db pull`         | Remote の状態を取得（schema.sql）     |
| `supabase db reset`        | ローカルDBを `migrations/` から再構築 |
| `supabase db diff`         | Remote との差分を新たな migration に変換 |
