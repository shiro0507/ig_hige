# セットアップ状況

ig_sleep/ig_akiyaと同じ構成でスキャフォールド(2026-08-20)。今回は静止画投稿にも対応(REELSと画像を自動判別)。

- Instagramアカウント: `@medikaratsuyoihigeotoko` (`INSTAGRAM_ACCOUNT_ID` = `17841461989825682`)
- Facebookページ: `Medikaratsuyoihigeotoko`（2026-08-20新規作成、カテゴリ「個人ブログ」）。IG側のアカウントセンターからページにリンク済み。
- `INSTAGRAM_ACCESS_TOKEN` / `INSTAGRAM_ACCOUNT_ID` GitHub Secrets: **設定済み**(2026-08-20)。`insight.yml`手動実行で疎通確認済み(フォロワー数4)。

## 1. アクセストークン取得 — 完了(2026-08-20)

Meta App `akiya.app` (APP_ID: `27566135536361698`) をig_akiya/ig_sleepと共用。

**過去にハマったポイント（毎回同じ罠にかかるので必ず参照）:**
- Graph API Explorerの「Generate Access Token」ボタンを素直に押すと、新しいページ/IGアカウントの組み合わせでは「Instagram Login」フローに誘導され、`IGAA`プレフィックスのトークンが発行されてしまう(`graph.facebook.com`では使えない)。
- 回避策: Graph API Explorerを介さず、直接OAuthダイアログURLを叩く。
  ```
  https://www.facebook.com/v22.0/dialog/oauth?client_id=27566135536361698&redirect_uri=https%3A%2F%2Fdevelopers.facebook.com%2Ftools%2Fexplorer%2Fcallback&response_type=token&scope=instagram_basic,instagram_content_publish,pages_show_list,pages_read_engagement
  ```
  「設定を編集」からページ・IGアカウントを明示的に選択すると、`EAA`/`EAG`プレフィックスの正しいトークンが発行される。
  - `pages_manage_metadata`をscopeに含めると"Invalid Scopes"エラーになるので外すこと。
- **新規発見(ig_hige, 2026-08-20): akiya.appが既に「ビジネス統合」化している場合、素の`設定を編集`はページ選択画面を出さず、過去の許可(古いページのみ)をそのまま素通りしてしまう。** この場合はOAuth URLに`&auth_type=rerequest`を追加して再認可を強制すると、「akiya.appがアクセスするページを選択」→「akiya.appがアクセスするInstagramアカウントを選択」の画面が改めて出るので、新しいページ/IGアカウントに明示的にチェックを入れて進める。
- 新しく許可したページは`me/accounts`一覧にすぐ反映されないことがある。その場合は該当ページIDまたはIGアカウントIDを直接指定して確認する: `GET /{page-id}?fields=name,instagram_business_account` または `GET /{ig-account-id}?fields=id,username`
- Facebookページ↔Instagram連携自体は、ページ単体の「ページ設定」画面ではなく、**ページのプロフェッショナルダッシュボード → その他 → リンク済みのアカウント → Instagram → アカウントをリンク**から行う（IG側のアカウントセンター/プロアカウント設定には見当たらない）。IGアカウント側は事前にブラウザでログインしておく必要がある。

取得した短期トークンは以下で長期化:
```
python scripts/exchange_token.py <短期トークン>
gh secret set INSTAGRAM_ACCESS_TOKEN --repo shiro0507/ig_hige --body "<長期トークン>"
gh secret set INSTAGRAM_ACCOUNT_ID --repo shiro0507/ig_hige --body "<IGアカウントID>"
```
長期トークンは2026-08-20発行、約60日後(2026年10月中旬頃)に手動更新が必要。

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
