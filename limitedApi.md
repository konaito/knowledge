### API設計書 — *認証・sort パラメータなし版*

> **目的**
> 100 件を超えるデータコレクションについて、クライアントが `page` と `limit` で任意ページを取得できる RESTful API を設計する。シンプルな **Offset-based Pagination** を採用し、後々 **Cursor-based Pagination** へ差し替え可能な拡張性も確保する。

---

## 1. 仕様概要

| 項目       | 内容                                                         |
| -------- | ---------------------------------------------------------- |
| 基本 URL   | `https://api.example.com/v1/items`                         |
| HTTPメソッド | `GET`                                                      |
| 認証       | **なし（公開 API）**                                             |
| レスポンス形式  | `application/json; charset=utf-8`                          |
| 同期保証     | リクエスト時点のスナップショットを返す（DB Read Replica +  Read-consistent tx） |

---

## 2. リクエスト

### 2.1 クエリパラメータ

| パラメータ   | 型       | 必須 | デフォルト | 制約          | 説明         |
| ------- | ------- | -- | ----- | ----------- | ---------- |
| `page`  | integer | ✔️ | 1     | **≥ 1**     | 取得したいページ番号 |
| `limit` | integer | ❌  | 20    | **1 – 100** | 1 ページあたり件数 |

> **バリデーション失敗時:** HTTP 400 を返却。

### 2.2 例

```
GET /v1/items?page=3&limit=50 HTTP/1.1
```

---

## 3. レスポンス

```jsonc
HTTP/1.1 200 OK
Link: <https://api.example.com/v1/items?page=4&limit=50>; rel="next",
      <https://api.example.com/v1/items?page=2&limit=50>; rel="prev"
Cache-Control: private, max-age=0
ETag: "5d8c72e2-3a9"

{
  "data": [
    /* item objects (≤ limit 件) */
  ],
  "meta": {
    "total_count": 1823,
    "page": 3,
    "limit": 50,
    "total_pages": 37,
    "has_next": true,
    "has_prev": true
  }
}
```

* **`data`** — リソース配列（ビジネス固有）。
* **`meta`** — ページング情報。
* **HTTP Link ヘッダ** — RFC 5988 準拠で `next` / `prev`（必要に応じ `first` / `last`）を返却。

---

## 4. エラーハンドリング

| HTTP Status | エラーコード           | 意味                             |
| ----------- | ---------------- | ------------------------------ |
| `400`       | `INVALID_PARAM`  | page または limit が範囲外／非数         |
| `404`       | `NOT_FOUND`      | ページが存在しない（page > total\_pages） |
| `429`       | `RATE_LIMITED`   | レート上限超過                        |
| `500`       | `INTERNAL_ERROR` | サーバ内部エラー                       |

エンベロープ形式:

```jsonc
{
  "error": {
    "code": "INVALID_PARAM",
    "message": "limit must be between 1 and 100"
  }
}
```

---

## 5. 内部実装指針

### 5.1 DB クエリ例（PostgreSQL）

```sql
SELECT *
  FROM items
 WHERE deleted_at IS NULL
 ORDER BY created_at DESC, id
 LIMIT :limit
OFFSET (:page - 1) * :limit;
```

* **安定ソートキー**として `created_at, id` を固定。
* 総件数取得は `COUNT(*)` を同トランザクションで実行。

### 5.2 パフォーマンス対策

1. `limit` 上限 100 で OFFSET 負荷を抑制。
2. 1 M 件超のテーブルでは cursor 方式に移行しやすいよう、レスポンス `meta.next_cursor` を将来的に追加予定。
3. **ETag** と `If-None-Match` で同一リクエストに 304 応答。

---

## 6. 公開 API 向けガバナンス

| 項目    | ポイント                     |
| ----- | ------------------------ |
| レート制限 | 100 req/min/IP（429 応答）   |
| ログ    | page/limit を記録、PII を除外   |
| キャッシュ | エッジ CDN 可。動的パラメータに Vary。 |

---

## 7. 将来拡張

| 要件              | 変更点                                             |
| --------------- | ----------------------------------------------- |
| **Realtime 更新** | `/v1/items/stream` SSE/WS 追加                    |
| **Cursor 化**    | クエリ `cursor` & `limit`、レスポンス `meta.next_cursor` |
| **GraphQL**     | `/graphql` を追加し複数呼び出し削減                         |

---

## 8. テスト計画（抜粋）

1. **正常系**

   * `page=1&limit=20` 総件数 100 → 20件返却 `total_pages=5`
2. **境界値**

   * `limit=1` / `limit=100`
   * `page=total_pages` → 末尾データ取得
3. **エラー系**

   * `page=0` → 400
   * `limit=101` → 400
   * `page>total_pages` → 404
4. **並行更新**

   * ページング中の追加削除でもスナップショット一貫性保持

---

## 9. まとめ

* 最小限のパラメータ (`page`, `limit`) で実装が簡潔。
* **Link ヘッダ** と **meta** ブロックを持たせ、UI 側が総ページ数を把握しやすい。
* 後々の **Cursor-based Pagination** へのリプレイスを想定した拡張性を確保した。
