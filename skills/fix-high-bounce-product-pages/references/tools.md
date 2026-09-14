# 使うMCPツールの正確な呼び方

2026-09-15 時点で、各MCPの実際のツール定義を確認して書いた。仕様が変わることがあるので、呼ぶ前にクライアントのツール一覧で名前と引数を確かめること。

## GA4

### Google 公式 `analytics-mcp`（`pipx run analytics-mcp`、ADC 認証）

ツール名：`run_report`。引数は **snake_case**。

```json
{
  "property_id": 123456789,
  "date_ranges": [{"start_date": "28daysAgo", "end_date": "2daysAgo"}],
  "dimensions": ["pagePath"],
  "metrics": ["sessions", "bounceRate", "engagementRate", "averageSessionDuration", "keyEvents"],
  "dimension_filter": {"filter": {"field_name": "pagePath", "string_filter": {"match_type": "CONTAINS", "value": "/products/"}}},
  "metric_filter": {"filter": {"field_name": "sessions", "numeric_filter": {"operation": "GREATER_THAN_OR_EQUAL", "value": {"int64_value": "30"}}}},
  "order_bys": [{"metric": {"metric_name": "bounceRate"}, "desc": true}],
  "limit": 20
}
```

他に `get_account_summaries`（プロパティIDが分からないとき）、`get_custom_dimensions_and_metrics`、`run_realtime_report` がある。

### 自作版（HALDATA が使っている `run_report`）

引数はフラットな文字列。フィルタ類は **camelCase の JSON 文字列**。

```json
{
  "property_id": "123456789",
  "start_date": "28daysAgo",
  "end_date": "2daysAgo",
  "dimensions": "pagePath",
  "metrics": "sessions,bounceRate,engagementRate,averageSessionDuration,keyEvents",
  "dimension_filter": "{\"filter\":{\"fieldName\":\"pagePath\",\"stringFilter\":{\"matchType\":\"CONTAINS\",\"value\":\"/products/\"}}}",
  "metric_filter": "{\"filter\":{\"fieldName\":\"sessions\",\"numericFilter\":{\"operation\":\"GREATER_THAN_OR_EQUAL\",\"value\":{\"int64Value\":\"30\"}}}}",
  "order_bys": "[{\"metric\":{\"metricName\":\"bounceRate\"},\"desc\":true}]",
  "limit": 20
}
```

### 返り値の読み方（両方共通）

`rows[]` の `dimensionValues[0].value` がページ、`metricValues[]` が指標の順。`bounceRate` と `engagementRate` は 0〜1 の小数（0.7457 = 74.6%）。`averageSessionDuration` は秒。

EC計測があるプロパティで使える指標：`addToCarts`, `ecommercePurchases`, `itemsViewed`（ディメンション `itemName` / `itemId` と組む）。

## Microsoft Clarity（公式 `@microsoft/clarity-mcp-server`）

- `query-analytics-dashboard({ query })`：自然言語1問。期間を必ず含める
  - 例：`"Average scroll depth and session count for https://example.com/products/abc between 2026/08/18 and 2026/09/13"`
  - 一覧なら：`"Top 20 visited URLs by session count with average scroll depth between 2026/08/18 and 2026/09/13"`
  - 「last 28 days」と書くと当日が終端になり GA4（2日前終端）とずれる。日付を書く
  - 返り値：`data[]` に `VisitedUrl`, `AvgScrollDepthPercent`, `SessionCount`
- `list-session-recordings({ filters, count, sortBy })`
  - `filters.date` は必須。UTC ISO 8601 ミリ秒付き（`2026-09-08T00:00:00.000Z`）
  - `filters.visitedUrls: [{ url, operator: "contains" }]`、`filters.scrollDepth: { min: 0, max: 30 }`
  - `count: 3`、`sortBy: "SessionDuration_DESC"`
  - 返り値：`link`（録画URL）、`totalDuration`、`timeline[]`（ページごとのクリック・デッドクリック）

## TrendViewer MCP（`https://haldata.net/trendviewer-mcp/` の接続手順で追加）

| ツール | 主な引数 | 返り値で使う項目 |
|---|---|---|
| `list_analyses` | `state="complete"`, `sort="latest_dataset_at"`, `order="desc"`, `name`（完全一致） | `analyses[].name, wtg_slug, wldh_slug` |
| `submit_analysis` | `name`, `category={name, detail}`, `products=[{code, mall}]` または `search_queries=[{keyword, mall}]`, `mode="manual"` | `wtg_slug`, `triggered`（true なら新規実行、`wldh_slug` は null） |
| `get_analysis_status` | `wtg_slug` | `state`（`complete` で次へ）, `wldh_slug` |
| `get_analysis_data` | `wldh_slug`, `include_per_sku=true`, `include_sub_viewpoints=false` | `viewpoints[]{id, viewpoint_ordinal_number, name, mention_count, positive_rate, negative_rate}`, `total_mentions`, `per_sku[]{code, product_id, mall, title, description, review_count, rating, price, product_url}`。返りは大きい（30 SKU で 70KB 超）。必要な行だけ抜く |
| `get_review_insights` | `dataset_slug`, `dimension="motivation"`, `group_by="viewpoint"` または `"product"`, `min_count=30` | `data[]{name, total, pm_count, pm_rate}`, `meta.baseline_pm_rate` |
| `get_review_insights` | `dataset_slug`, `dimension="context_3w"`, `mall_product_id`, `limit=8` | `data.who / where / when []{text, count, share}`, `meta.coverage_rate` |
| `get_review_sentences` | `dataset_slug`, `mall_product_id`（`per_sku[].product_id`。JAN ではない）, `viewpoint_id`（`viewpoints[].id`）, `purchase_motivation="only"`, `per_page=20` | `data[].content`（読むだけ。転載しない） |

注意点：
- `code` は楽天・Yahoo が JAN、Amazon が ASIN。`mall` は `rakuten` / `amazon` / `yahoo`
- 率は 0〜1 の小数
- `get_review_sentences` の `mall_product_id` は内部ID（`per_sku[].product_id`）。JAN を渡すと取れない
- 同じ名前で `submit_analysis` すると既存が再利用される（重複は作られない）
- データセットは契約枠に上限がある。「空き枠がありません」と返ったら、何を消すかはユーザーの判断

## Shopify

Shopify 公式に「Shopify connector for Claude」（Claude から自分の店舗の商品・コレクション・在庫・注文・顧客・割引・分析を扱える）がある。設定は Shopify ヘルプセンター「Connecting your Shopify store to AI tools」。
公式資料に個々のツール名は書かれていないので、接続後にツール一覧を見て **商品を読むツール** と **商品を更新するツール** を特定する。Admin API をラップしたコミュニティ製 MCP でも同じ。

このスキルで使うのは2つだけ：

1. 商品の読み取り：`handle` または `id` で商品を取り、`descriptionHtml`（または `body_html`）と `variants[].barcode`（JAN）を読む
2. 商品説明の更新：文案をユーザーに見せて OK をもらった後だけ。`status` / 公開状態は触らない

## Webページ取得

Shopify で商品説明が読めないときは、クライアントの fetch 系ツールで商品ページを取り、本文を読む。取れないページ（ログイン必須など）は「本文未確認」としてレポートに書く。
