# セットアップ状況

ig_sleep/ig_akiyaと同じ構成でスキャフォールド(2026-08-20)。今回は静止画投稿にも対応(REELSと画像を自動判別)。

- Instagramアカウント: `@medikaratsuyoihigeotoko`
- Facebookページ: (未確認 — 連携先ページ名・ID要確認)
- `INSTAGRAM_ACCESS_TOKEN` / `INSTAGRAM_ACCOUNT_ID` GitHub Secrets: **未設定**

## 1. アクセストークン取得 — 未完了

Meta App `akiya.app` (APP_ID: `27566135536361698`) をig_akiya/ig_sleepと共用。

**過去にハマったポイント（毎回同じ罠にかかるので必ず参照）:**
- Graph API Explorerの「Generate Access Token」ボタンを素直に押すと、新しいページ/IGアカウントの組み合わせでは「Instagram Login」フローに誘導され、`IGAA`プレフィックスのトークンが発行されてしまう(`graph.facebook.com`では使えない)。
- 回避策: Graph API Explorerを介さず、直接OAuthダイアログURLを叩く。
  ```
  https://www.facebook.com/v22.0/dialog/oauth?client_id=27566135536361698&redirect_uri=https%3A%2F%2Fdevelopers.facebook.com%2Ftools%2Fexplorer%2Fcallback&response_type=token&scope=instagram_basic,instagram_content_publish,pages_show_list,pages_read_engagement
  ```
  「設定を編集」からページ・IGアカウントを明示的に選択すると、`EAA`/`EAG`プレフィックスの正しいトークンが発行される。
  - `pages_manage_metadata`をscopeに含めると"Invalid Scopes"エラーになるので外すこと。
- 新しく許可したページは`me/accounts`一覧にすぐ反映されないことがある。その場合は該当ページIDを直接指定して確認する: `GET /{page-id}?fields=name,instagram_business_account`

取得した短期トークンは以下で長期化:
```
python scripts/exchange_token.py <短期トークン>
gh secret set INSTAGRAM_ACCESS_TOKEN --repo shiro0507/ig_hige --body "<長期トークン>"
gh secret set INSTAGRAM_ACCOUNT_ID --repo shiro0507/ig_hige --body "<IGアカウントID>"
```

## 2. コンテンツ

`content/YYYY-MM-DD/` 配下に置くと、その日の `post.yml` 実行で投稿される:
- `video.mp4` があればReelsとして投稿
- なければ `image.jpg` / `image.jpeg` / `image.png` があれば画像として投稿(単一画像、カルーセル非対応)
- `caption.txt`（任意、空でも可）
- `thumb_offset.txt`（任意、動画のみ）

2026-08-21〜2026-08-30の10日分、`/Users/makoto/Downloads/higeotoko`から画像を取り込み済み。**キャプションは全て空**なので、投稿前に各`content/YYYY-MM-DD/caption.txt`を埋める必要あり。

## 3. トークンの更新（手動運用）

長期トークンは発行から約60日で失効。自動更新ワークフローは組み込んでいない(他アカウントと同じ方針)ので、期限前に手動で再実行:

```
python scripts/exchange_token.py <新しい短期トークン>
gh secret set INSTAGRAM_ACCESS_TOKEN --repo shiro0507/ig_hige --body "<新しい長期トークン>"
```

## 4. insight.py の既知の制約

`insight.py`は`media_product_type == "REELS"`の投稿のみ集計する。画像投稿(IMAGE)のインサイトは現状フォロー未対応 — 必要になったら拡張する。
