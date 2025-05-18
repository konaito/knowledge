# JobPosting 構造化データ 教科書

---

## 序章：なぜ JobPosting 構造化データを使うのか

Google 検索に求人情報を最適化するためには、`JobPosting` 構造化データを用いて求人ページを機械可読にすることが不可欠です。適切にマークアップすると、Google 求人検索（Google for Jobs）でロゴや給与情報などを強調したリッチリザルトが表示され、応募意欲の高いユーザーを効率良く呼び込めます。citeturn5view0

---

## 第1章 JobPosting 構造化データの全体像

### 1‑1 期待できる効果

* **インタラクティブな検索結果** – ロゴ、レビュー、詳細情報を表示し視認性を向上。citeturn5view0
* **精度の高いフィルタリング** – 地域や職種で絞り込めるため、適格な候補者のクリック率が上がる。
* **コンバージョン機会の増加** – 求人一覧と詳細ページ間の導線を Google 側が補完。

### 1‑2 実装フロー（7 ステップ）

1. Googlebot がクロール可能か確認
2. ページ複製がある場合は正規 URL を設定
3. **必須 & 推奨プロパティ** を追加
4. 技術・コンテンツ ガイドライン遵守
5. リッチリザルト テストで検証
6. 少数ページで公開 → URL 検査ツールで確認
7. Indexing API & サイトマップで最新情報を送信citeturn5view0

---

## 第2章 プロパティ解説

### 2‑1 必須プロパティ

| プロパティ                | 型              | 概要                                    | 例                             |                   |
| -------------------- | -------------- | ------------------------------------- | ----------------------------- | ----------------- |
| `datePosted`         | `Date`         | 掲載開始日（ISO8601）                        | `"2025-05-01"`                |                   |
| `description`        | `Text`         | HTML 可。職務・資格・勤務条件など詳細説明。`title` と同一不可 | `<p>飲食店でのホール業務…</p>`          |                   |
| `hiringOrganization` | `Organization` | 会社名、`sameAs`、`logo` など                | `"name":"Example Inc"`        |                   |
| `jobLocation`        | `Place`        | 物理勤務地。`addressCountry` 必須             | `"addressRegion":"Tokyo"`     |                   |
| `title`              | `Text`         | 職務名のみを簡潔に記述                           | `"title":"Software Engineer"` | citeturn7view0 |

> **Tips**: `title` に給与や住所を含めない／特殊文字を多用しない。

### 2‑2 条件付き必須プロパティ

| シナリオ      | 必須プロパティ                                                             | ポイント                             |
| --------- | ------------------------------------------------------------------- | -------------------------------- |
| 有効期限のある求人 | `validThrough` (`DateTime`)                                         | 期限切れ後に削除必須 citeturn8view0     |
| 完全リモート勤務  | `jobLocationType`=`"TELECOMMUTE"` & `applicantLocationRequirements` | 応募可能国を 1 つ以上指定 citeturn5view0 |

### 2‑3 推奨・強化プロパティ（抜粋）

| プロパティ                           | 型                                      | 主な効果 / 注意                                                                                                     |
| ------------------------------- | -------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| `baseSalary`                    | `MonetaryAmount` → `QuantitativeValue` | `unitText` は `HOUR` `DAY` `WEEK` `MONTH` `YEAR` いずれか。範囲は `minValue`/`maxValue` で指定。雇用主のみ設定可。citeturn8view0 |
| `employmentType`                | `Text`                                 | `FULL_TIME` `PART_TIME` 等複数可。citeturn8view0                                                                |
| `identifier`                    | `PropertyValue`                        | 企業側で使う一意 ID。citeturn8view0                                                                                 |
| `directApply`                   | `Boolean`                              | 中間ページなしで応募完了できる場合は `true`。β機能。citeturn8view0                                                               |
| `applicantLocationRequirements` | `AdministrativeArea`                   | 在宅勤務で応募者の居住国／地域を制限する時に使用。                                                                                     |

---

## 第3章 実装サンプル

### 3‑1 標準的な求人

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org/",
  "@type": "JobPosting",
  "title": "Software Engineer",
  "description": "<p>多様な視点が優れた…</p>",
  "identifier": {
    "@type": "PropertyValue",
    "name": "Example Inc",
    "value": "1234567"
  },
  "datePosted": "2025-05-01",
  "validThrough": "2025-07-01T00:00",
  "employmentType": "FULL_TIME",
  "hiringOrganization": {
    "@type": "Organization",
    "name": "Example Inc",
    "sameAs": "https://example.com",
    "logo": "https://example.com/logo.png"
  },
  "jobLocation": {
    "@type": "Place",
    "address": {
      "@type": "PostalAddress",
      "streetAddress": "123 Main St",
      "addressLocality": "Shibuya",
      "addressRegion": "Tokyo",
      "postalCode": "150-0002",
      "addressCountry": "JP"
    }
  },
  "baseSalary": {
    "@type": "MonetaryAmount",
    "currency": "JPY",
    "value": {
      "@type": "QuantitativeValue",
      "value": 2800,
      "unitText": "HOUR"
    }
  }
}
</script>
```

JSON-LD を `<head>` 内に置くのが推奨。citeturn5view0

### 3‑2 完全リモート勤務求人

ポイント: `jobLocationType` と `applicantLocationRequirements` を併用し、`jobLocation` は任意（オフィス併用の場合は追加）。citeturn5view0

---

## 第4章 コンテンツ & 技術ガイドライン

1. **求人 1 件につき 1 URL** – 検索結果や一覧ページではなく詳細ページにのみ `JobPosting` を配置。citeturn3view0
2. **実際のページ内容と一致** – タイトル・給与などページ上に表示されていない情報を構造化データで追加しない。citeturn3view0
3. **期限切れ対応** – `validThrough` 超過・採用完了後は `JobPosting` を削除 or ページを 404/410。Indexing API で速やかに通知。citeturn5view0
4. **スパム & 可読性** – 過度な装飾文字や文法ミスはポリシー違反。citeturn7view0

---

## 第5章 テスト & 配信

* **リッチリザルト テスト** – マークアップの妥当性とプレビュー確認
* **Search Console URL 検査** – レンダリング確認 & クロール依頼
* **Indexing API** – 求人 URL の新規・更新・削除を即時通知
* **サイトマップ** – `lastmod` を更新して Google に変更を知らせる

---

## 第6章 トラブルシューティング

| 症状             | よくある原因                      | 解決策                      |
| -------------- | --------------------------- | ------------------------ |
| 構造化データが拾われない   | 必須プロパティ不足／構文エラー             | リッチリザルト テストで詳細を確認        |
| リスティングページに警告   | 検索結果ページに `JobPosting` を配置   | 一覧から削除し詳細ページみに限定         |
| 期限切れ求人が表示され続ける | `validThrough` 未設定またはページ未削除 | API で削除通知 & ページを 404/410 |

---

## 第7章 まとめ & 次のステップ

* `JobPosting` は求職者体験を向上させ、採用効率を高める最重要マークアップ
* **必須 + 推奨プロパティ** を網羅し、ガイドラインに沿った運用を継続
* Search Console & Indexing API を活用し、鮮度を保つ

---

### 章末問題（理解度チェック）

1. `baseSalary` を雇用主以外がマークアップしてはいけない理由を説明しなさい。
2. 在宅勤務求人で `applicantLocationRequirements` が無い場合、Google はどのように扱いますか。

---

*最終更新: 2025‑05‑18*
