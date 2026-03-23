# zenn-content

Zenn tech blog の記事リポジトリ。GitHub連携で自動デプロイ。

## セットアップ（初回のみ・手動）

### 1. Zenn アカウントと GitHub 連携

1. [zenn.dev](https://zenn.dev) にログイン
2. ダッシュボード → [デプロイ管理](https://zenn.dev/dashboard/deploys) を開く
3. 「リポジトリを連携する」→ `takish/zenn-content` を選択
4. 連携完了を確認（mainブランチへのpushで自動デプロイされる）

### 2. ローカル環境

```bash
cd ~/Development/zenn-content

# プレビュー（localhost:8000）
npx zenn preview

# 記事一覧
npx zenn list:articles
```

## 記事の管理フロー

記事の執筆・検証は [content-hub](https://github.com/takish/content-hub)（private）で行う。

```
content-hub (private)                    zenn-content (public)
articles/{slug}/draft.md                 articles/{slug}.md
  ↓ /publish {slug}                        ↓ push to main
  セキュリティスキャン                       Zenn に自動反映
  → コピー (published: false)              → 下書きとして表示
  ↓ /publish {slug} --live
  published: true に変更 → push            → Zenn で公開
```

## 注意事項

- **このリポジトリは public** — `published: false` でも GitHub 上ではコードが見える
- 秘密情報（APIキー、ローカルパス等）を含む記事は絶対にコミットしない
- 記事の素材・調査メモ・verifyレポートは content-hub に残し、ここには完成品のみ配置する
