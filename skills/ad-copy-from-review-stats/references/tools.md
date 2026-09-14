# 使うMCPツールの正確な呼び方

2026-09-15 時点の実定義。呼ぶ前にクライアントのツール一覧で確かめること。

## Google Ads

### Google 公式 `google-ads-mcp`（読み取り専用・GAQL）

インストール：`pipx run --spec "google-ads-mcp==X.Y.Z" google-ads-mcp`。環境変数 `GOOGLE_PROJECT_ID`, `GOOGLE_ADS_DEVELOPER_TOKEN`, `GOOGLE_APPLICATION_CREDENTIALS`（マネージャー経由なら `GOOGLE_ADS_LOGIN_CUSTOMER_ID`）。スコープ `https://www.googleapis.com/auth/adwords`。

ツール：`list_accessible_customers`（アカウント一覧）、`search`（GAQL 実行）、`get_resource_metadata`（リソースの項目一覧）。`search` の引数名はクライアントのツール定義で確認する（customer_id と GAQL 文字列）。

広告グループ単位の成果（直近30日）：

```sql
SELECT ad_group.id, ad_group.name, campaign.name,
       metrics.impressions, metrics.clicks, metrics.ctr,
       metrics.conversions, metrics.cost_micros
FROM ad_group
WHERE campaign.id = {campaign_id}
  AND ad_group.status = 'ENABLED'
  AND segments.date DURING LAST_30_DAYS
ORDER BY metrics.impressions DESC
```

検索語（どの語で表示・クリックされたか）：

```sql
SELECT ad_group.name, search_term_view.search_term,
       metrics.impressions, metrics.clicks, metrics.ctr, metrics.conversions
FROM search_term_view
WHERE campaign.id = {campaign_id}
  AND segments.date DURING LAST_30_DAYS
  AND metrics.impressions >= 20
ORDER BY metrics.impressions DESC
LIMIT 50
```

既存の見出し（レスポンシブ検索広告）：

```sql
SELECT ad_group.name, ad_group_ad.ad.id,
       ad_group_ad.ad.responsive_search_ad.headlines,
       ad_group_ad.ad.responsive_search_ad.descriptions,
       ad_group_ad.status
FROM ad_group_ad
WHERE campaign.id = {campaign_id}
  AND ad_group_ad.status = 'ENABLED'
```

`metrics.ctr` は 0〜1 の小数。`cost_micros` は円の 100万倍（9,449,000,000 = 9,449円）。

### HALDATA 自作版（読み書き可）

| ツール | 引数 | 返り値 |
|---|---|---|
| `list_campaigns` | `customer_id` | `campaign_id, name, status, budget_micros, channel_type` |
| `list_ad_groups` | `campaign_id`, `customer_id` | `ad_group_id, name, status, type` |
| `get_campaign_performance` | `campaign_id`, `customer_id`, `start_date`, `end_date`（YYYY-MM-DD。`date_range` という引数は無いので渡さない） | `impressions, clicks, cost_micros, ctr, average_cpc, conversions`（キャンペーン単位のみ） |
| `list_keywords` | `ad_group_id`, `customer_id` | `criterion_id, keyword, match_type, status, cpc_bid_micros, quality_score`（成果は返らない。除外キーワードも混ざるので `quality_score > 0` を目安に見る） |
| `create_responsive_search_ad` | 書き込み。案を見せて OK をもらった後だけ。一時停止で作成 | |

自作版では **広告グループ単位の CTR と検索語と既存見出しは取れない**。キャンペーン単位の CTR で絞り、既存見出しはユーザーに貼ってもらう。

## TrendViewer MCP

| ツール | 主な引数 | 使う項目 |
|---|---|---|
| `list_analyses` | `state="complete"`, `sort="latest_dataset_at"`, `order="desc"`, `name` | `wldh_slug` |
| `submit_analysis` | `name`, `category={name, detail}`, `products=[{code, mall}]` または `search_queries=[{keyword, mall}]`, `mode="manual"` | `wtg_slug`, `triggered` |
| `get_analysis_status` | `wtg_slug` | `state`, `wldh_slug` |
| `get_analysis_data` | `wldh_slug`, `include_per_sku=false`, `include_sub_viewpoints=false` | `viewpoints[]{id, name, mention_count, positive_rate, negative_rate}`, `total_mentions` |
| `get_review_insights` | `dataset_slug`, `dimension="motivation"`, `group_by="viewpoint"`, `min_count=30` | `data[]{name, total, pm_count, pm_rate}`, `meta.baseline_pm_rate`（`total` は文の数。言及数には使わない） |
| `get_review_insights` | `dataset_slug`, `dimension="context_3w"`, `limit=8` | `data.who/where/when[]{text, count}`, `meta.coverage_rate` |
| `get_review_sentences` | `dataset_slug`, `mall_product_id`（`per_sku[].product_id`）, `viewpoint_id`（`viewpoints[].id`）, `purchase_motivation="only"`, `per_page=20` | 読むだけ。転載しない |

率は 0〜1 の小数。`code` は楽天・Yahoo が JAN、Amazon が ASIN。
