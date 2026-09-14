# haldata-skills

HALDATA が公開する Claude 用スキル集。
[mcp-dir（流通の仕事で使えるMCPディレクトリ）](https://haldata.net/mcp-dir/) の「組み合わせレシピ」を、Claude が読んでそのまま実行できる **SKILL.md** に落としたものです。

レシピ（何をどの順で使うか、人が読む説明）と、スキル（Claude が実行する手順・閾値・出力の型）の二層になっています。レシピで「できること」を選び、スキルを入れて「一言で実行」してください。

## 対応レシピ

| スキル | レシピ | 使うMCP | 所要 |
|---|---|---|---|
| [fix-high-bounce-product-pages](skills/fix-high-bounce-product-pages/) | [売れ筋なのに離脱される商品ページを直す](https://haldata.net/mcp-dir/#rcp=0) | GA4 → Clarity → TrendViewer → Shopify | 約15分 |

順次追加します（次：広告の訴求軸をレビュー統計で決める／新規参入カテゴリの下調べ）。

## 導入手順

### 1. 必要なMCPをつなぐ

各スキルの「前提」に書いてあります。共通で要るのは次の2つです。

- **TrendViewer MCP** — [接続手順](https://haldata.net/trendviewer-mcp/)（接続用URLを登録してログインするだけ。APIキーの発行は不要）
- **GA4 MCP** — Google 公式 [`analytics-mcp`](https://github.com/googleanalytics/google-analytics-mcp)（`pipx run analytics-mcp`、読み取り専用）

任意：[Microsoft Clarity MCP](https://github.com/microsoft/clarity-mcp-server)、Shopify（[Claude 用 Shopify コネクタ](https://help.shopify.com/en/manual/ai-powered-tools/connecting-ai-tools)）

### 2. スキルを入れる

**Claude デスクトップ／Claude.ai**
`skills/<スキル名>/` フォルダを zip にして、設定 → スキル からアップロードします。または SKILL.md の内容をプロジェクトの指示に貼っても動きます。

**Claude Code**
```bash
git clone https://github.com/haldata-net/haldata-skills.git
cp -r haldata-skills/skills/fix-high-bounce-product-pages ~/.claude/skills/
```

### 3. 一言で呼ぶ

> 離脱率の高い商品ページを直したい。GA4 のプロパティは 123456789、商品ページは /products/ 配下。

スキルが必要な情報を確認してから、手順どおりに進めます。書き込み（Shopify の商品更新など）は文案を見せてから、あなたの OK があるまで実行しません。

## 方針

- **レビュー原文は出力しない。** TrendViewer の集計値（件数・率）だけを根拠に使います
- **書き込みは確認後。** 下書きまでで止め、公開状態は変えません
- **閾値は既定値。** 直帰率70%、セッション30件などは SKILL.md に書いてあり、依頼時に変えられます

## 検証

各スキルの `examples/` に、HALDATA の自社データで最後まで実行した記録があります（商品名・ブランド名は伏せてあります）。

## 運営

HALDATA株式会社 — https://haldata.net/
掲載・改善の相談は https://haldata.net/contact/?from=skills
