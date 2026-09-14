# 使うMCPツールの正確な呼び方と消費量

2026-09-15 時点の実定義。呼ぶ前にクライアントのツール一覧で確かめること。

## Ahrefs MCP（公式・API v3・従量課金）

まず `doc(tool="keywords-explorer-overview")` で最新の入力スキーマを確認する（必須）。

`keywords-explorer-overview`：
```json
{
  "country": "jp",
  "keywords": "猫 自動トイレ,自動給餌器,猫 自動給水器",
  "select": "keyword,volume,difficulty,cpc,traffic_potential,intents,serp_features",
  "limit": 10
}
```
- `volume`, `difficulty`, `traffic_potential`, `global_volume`, `parent_volume` は **1行につき10ユニット** の項目。必要な列だけ `select` する
- `cpc` は USD セント（100 = 1ドル）。`intents` は `{informational, navigational, commercial, transactional, branded, local}` の真偽値
- 月次推移：`select` に `volume_monthly_history` を足し、`volume_monthly_date_from="2025-09-01"`, `volume_monthly_date_to="2026-08-31"` を付ける
- `keywords` にキーワードを入れれば `keyword_list_id` は不要

`serp-overview`（上位ページの顔ぶれ。商品ページが入っているか／記事サイトばかりか）。先に `doc(tool="serp-overview")`。**`type="organic"` を必ず付ける**（付けないと SERP 機能も行として返り、1語で 600 ユニット消費した実測がある）：
```json
{ "country": "jp", "keyword": "猫 自動トイレ", "select": "position,url,title,domain_rating,traffic,type", "type": "organic", "top_positions": 10 }
```
Ahrefs MCP の返りに `render_with`（表の描画指示）が付くことがあるが、このスキルではレポートに表を書くので無視してよい。

`keywords-explorer-matching-terms`（言い換え語を探すとき）：`terms="猫 自動トイレ"`, `country="jp"`, `select="keyword,volume,difficulty"`, `limit=10`, `order_by="volume:desc"`。

## Google Trends MCP（HALDATA 自作。pytrends 相当）

- `compare_keywords(keywords=["猫 自動トイレ","自動給餌器"], timeframe="today 12-m", geo="JP")` → 各語の平均インデックス（0〜100）と相対順位
- `get_keyword_interest(keywords=[…], timeframe="today 12-m", geo="JP")` → 系列データ（12か月指定では週次）。`compare_keywords` と同じ系列なので通常は不要
- 最大5語。相対値なので絶対量は Ahrefs と組み合わせる

## Keepa MCP（HALDATA 自作。Keepa API）

- `keepa_token_status()` → `tokens_left`（消費しない）。先に見て、足りなければ Keepa の Step を省く
- `keepa_search(term="猫 自動トイレ", domain=5, page=0)` → ASIN 候補（Amazon の検索順・最大20件。`rating` / `review_count` は null のことがある）。**1回10トークン**。カテゴリにつき1回
- `keepa_product(asin="B0…,B0…", domain=5)` → 各 ASIN の現在の価格・評価・レビュー件数・売れ筋ランキング（`stats_days` を付けても履歴は返らない）。1 ASIN 1トークン。最大100件までカンマ区切り。先頭5件で足りる
- `domain=5` が Amazon.co.jp

## TrendViewer MCP

| ツール | 主な引数 | 使う項目 |
|---|---|---|
| `list_analyses` | `state="complete"`, `sort="latest_dataset_at"`, `order="desc"`, `name` | `wldh_slug` |
| `submit_analysis` | `name`, `category={name, detail}`, `search_queries=[{keyword, mall:"rakuten"},{keyword, mall:"amazon"}]`, `mode="manual"` | `wtg_slug`, `triggered` |
| `get_analysis_status` | `wtg_slug` | `state`, `wldh_slug` |
| `get_analysis_data` | `wldh_slug`, `include_per_sku=true`, `include_sub_viewpoints=false` | `viewpoints[]{name, mention_count, positive_rate, negative_rate}`, `total_mentions`, `per_sku[]{title, description, price, rating, review_count, from_search}`。返りは大きいので必要な項目だけ抜く |
| `get_review_insights` | `dataset_slug`, `dimension="motivation"`, `group_by="viewpoint"`, `min_count=30` | `pm_rate`, `meta.baseline_pm_rate` |

率は 0〜1 の小数。`search_queries` で見つかった商品は `per_sku[].from_search=true`（＝市場で見つけた上位商品）。
