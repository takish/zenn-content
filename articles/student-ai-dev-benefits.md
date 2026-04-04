---
title: "【学生無料】GitHub Copilotの始め方 — Student Pack申請からVS Codeセットアップまで"
emoji: "🎓"
type: tech
topics:
  - github
  - githubcopilot
  - vscode
  - 学割
  - AI
  - mac
published: false
---

プログラミングを始めたばかりの頃、開発ツールの選択肢が多すぎて「結局どれを入れればいいの？」と迷った経験はないでしょうか。AIコーディングツール、クラウドサービス、デザインツール——どれも便利そうだけど、有料のものが多くて手が出しにくい。

学生なら、まずやるべきことは1つです。**GitHub Student Developer Pack**に申請してください。これが通れば、GitHub Copilotを含む100以上のツール・サービスが無料で使えるようになります。

この記事では、GitHub Student Developer Packの申請からGitHub Copilotを使い始めるまでを、ステップバイステップで案内します。

:::message
この記事の情報は2026年4月時点のものです。各サービスの料金・特典内容は変更される可能性があるため、申請前に公式サイトで最新情報を確認してください。
:::

## GitHub Student Developer Packで何が手に入るか

GitHub Student Developer Packは、学生認証を行うだけで開発ツール・サービスが無料または大幅割引で利用できるプログラムです。

**特に価値が大きいもの**を抜粋します。

| サービス | 通常料金 | 学生特典 |
|---------|---------|---------|
| GitHub Copilot | $10/月 | **Copilot Student（無料）** |
| GitHub Pro | $4/月 | **無料** |
| GitHub Codespaces | 従量課金 | **月180コア時間無料** |
| JetBrains全製品 | $289/年（All Products Pack初年度） | **無料（非商用）** |
| 1Password | $2.99/月 | **1年間無料** |
| DigitalOcean | 従量課金 | **$200クレジット** |
| Microsoft Azure | 従量課金 | **$100クレジット** |
| Notion | $12/月 | **Plus Plan無料**（Notion独自の教育プログラム） |

通常料金を合算すると、年間20万円以上に相当します。この中で最も日常的に使うことになるのがGitHub Copilotです。Copilot StudentプランではAIによるコード補完が無制限で使えます。

:::message
Copilot Studentは2026年3月に新設されたプランで、Copilot Proとは一部異なります。主な違いは、GPT-5.4やClaude Opus等のプレミアムモデルを手動で選択できない点です（Auto modeでCopilotが最適なモデルを自動選択します）。日常的なコード補完やチャットには影響しません。
:::

## Step 1: GitHubアカウントを作成する

GitHubアカウントを大学メール（.ac.jp）で作成します。すでにアカウントがある場合はStep 2に進んでください。

1. ブラウザでGitHubのサインアップページを開く
2. **メールアドレス**を入力する
   - **大学のメールアドレス（.ac.jp）を使うことを強く推奨**します。後のStudent Pack申請がスムーズになります
   - 個人メールで登録済みの場合は、Settings → Emails から大学メールを追加できます
3. **パスワード**を設定する
4. **ユーザー名**を決める
   - 本名に近いものが推奨です（就活やポートフォリオでそのまま使えるため）
   - 後から変更可能ですが、URLも変わるため最初に決めておく方が楽です
5. メール認証を済ませる

:::message
大学メールを持っていない場合（入学前など）は、個人メールで登録してから後で大学メールを追加する手順でも問題ありません。
:::

## Step 2: GitHub Student Developer Packに申請する

GitHub Educationのページから学生証画像をアップロードして申請します。.ac.jpメールなら多くの場合1日以内に承認されます。

### 申請に必要なもの

| 項目 | 内容 |
|------|------|
| GitHubアカウント | 大学メールが紐づいていること |
| 学生証の画像 | スマホで撮影したもので可 |
| 在学証明 | 学生証の画像で兼用できる |

### 申請手順

1. GitHub Educationのページを開く
2. **「Join Global Campus」**をクリック
3. **「I'm a student」**を選択
4. 学校名を入力する
   - 日本の大学・専門学校・高専も登録されています
   - 英語表記で入力すると候補が出やすいです（例: `University of Tokyo`, `Tokyo Institute of Technology`）
   - 見つからない場合は正式名称を手動入力します
5. **利用目的**を記入する
   - 英語で簡潔に書けばOKです
   - 例: `I'm studying computer science and want to use GitHub for coursework and personal projects.`
6. **学生証の画像をアップロード**する
   - 名前、学校名、有効期限が読み取れる状態であること
   - スマホのカメラで撮影したもので通ります
   - 暗い写真や文字がぼやけているものは再提出を求められることがあります
7. **Submit**をクリックして送信

### 審査の目安

| 条件 | 審査期間 |
|------|---------|
| .ac.jpメールで申請 | 数時間〜1日 |
| 個人メール + 学生証画像 | 数日〜1週間 |
| 情報不足で再提出 | さらに数日 |

.ac.jpメールを使っている場合は、多くのケースで1日以内に承認されます。承認されるとGitHubからメールが届きます。

Student Packの有効期限は通常2年間（または卒業まで）です。期限が近づくと更新手続きの案内が届きます。JetBrainsなど一部のサービスは年次更新が必要なため、メールの確認を忘れないようにしてください。

:::message alert
**よくある審査落ちの原因**
- 学生証の画像がぼやけていて文字が読めない
- 学生証に有効期限が記載されていない（裏面に記載されている場合は裏面もアップロード）
- GitHubプロフィールに情報がほとんどない（簡単なプロフィールを埋めておくと通りやすい）
:::

## Step 3: GitHub Copilotを有効化する

Student Packの承認後、GitHubの設定画面からCopilotを有効化します。追加の申請は不要です。

1. GitHubのSettings → Copilot を開く
2. **「Enable GitHub Copilot」**をクリック
   - Student Packが承認されていれば、料金は表示されません（$0/月）
3. **Suggestions matching public code**の設定を確認する
   - 「Allowed」にすると、公開コードと一致するコード補完も表示されます
   - 学習目的で個人プロジェクトに使う範囲であれば「Allowed」で問題ありません。商用プロジェクトではライセンス確認が必要になる場合があります
4. 設定を保存する

これでGitHub側の設定は完了です。

## Step 4: VS Codeに GitHub Copilot をインストールする

VS Code（Visual Studio Code）にCopilot拡張機能をインストールし、コード補完が動作することを確認します。

### VS Codeのインストール

VS Codeが未インストールの場合は、まずインストールします。

macOSの場合、Homebrewがインストール済みであればターミナルで以下を実行するのが最も簡単です。

```bash
brew install --cask visual-studio-code
```

Homebrewがない場合は、Visual Studio Codeの公式サイトから.dmgをダウンロードしてインストールします。

### Copilot拡張機能のインストール

1. VS Codeを起動する
2. 左サイドバーの**拡張機能アイコン**（四角が4つ並んだアイコン）をクリック
3. 検索窓に `GitHub Copilot` と入力
4. **「GitHub Copilot」**をインストールする
   - 「GitHub Copilot Chat」も一緒にインストールされます
5. VS Code右下に表示される**「Sign in to GitHub」**をクリック
6. ブラウザが開くのでGitHubアカウントでログインする
7. VS Codeに戻ると、Copilotが有効化されている

### 動作確認

新しいファイルを作成して、Copilotが動作することを確認します。

```python
# test.py を作成して以下を入力
def hello():
    # ここまで入力すると、Copilotがグレーの文字で候補を提案する
    # 例: print("Hello, World!") のような補完が表示される
```

グレーの文字で補完候補が表示されたら、**Tabキー**で受け入れます。これがCopilotのコード補完です。

補完が表示されない場合は、VS Code右下のCopilotアイコンを確認してください。アイコンに斜線が入っている場合は、クリックしてCopilotを有効にします。

## Step 5: Copilot ChatとAgent Modeを試す

コード補完に加えて、Copilot ChatとAgent Modeも試してみます。どちらもCopilot Studentプランに含まれています。

### Copilot Chat

1. VS Codeで**Ctrl + Cmd + I**（macOS）を押す
2. Chat viewが開く
3. 質問を入力する

```
Pythonでフィボナッチ数列を返す関数を書いて
```

Copilot Chatは、コードの説明、バグの特定、リファクタリングの提案など、開発中のさまざまな場面で使えます。

### Agent Mode

Copilotには**Agent Mode**があります。Agent Modeでは、Copilotが複数のステップを自律的に実行します。

1. **Cmd + Shift + I**（macOS）を押すとAgent Modeに切り替わる
   - または、Chat viewを開いた状態でモードピッカーから「Agent」を選択する
2. プロンプトを入力する
   - 例: `このプロジェクトにREADME.mdを作成して`

Agent Modeでは、ファイルの作成・編集、ターミナルコマンドの実行までCopilotが行います。ただし、Agent Modeの操作はプレミアムリクエストを消費します（Copilot Studentプランは月300回）。通常のコード補完は無制限でプレミアムリクエストを消費しないため、日常的な開発には影響しません。

## Student Packが通ったら追加で申請できるサービス

GitHub Student Packが起点になると、以下のサービスも連鎖的に利用可能になります。余裕があるときに追加で申請しておくと便利です。

| 優先度 | サービス | 申請方法 | 得られるもの |
|--------|---------|---------|-------------|
| **高** | JetBrains | 大学メール or Student Pack連携 | IntelliJ, PyCharm等の全IDE |
| **高** | Cursor | .eduメール or 学生証アップロード（SheerID） | AIエディタ Pro 1年無料 |
| 中 | Figma Education | 学校メール | デザインツール無料 |
| 中 | Notion Education | 学校メール | Plus Plan無料 |
| 中 | 1Password | Student Pack経由 | パスワード管理1年無料 |
| 低 | DigitalOcean | Student Pack経由 | $200クラウドクレジット |
| 低 | Azure for Students | 学校メール | $100クラウドクレジット |

CursorはVS Codeベースの独立AIエディタです。GitHub Copilotとは異なるアプローチでAIコーディング支援を行うため、両方試してみて自分に合う方をメインにする判断ができます。どちらも学生は無料です。

:::message
Cursorの学生プランは.eduメールが前提です。日本の.ac.jpメールでSheerID認証が通るかはケースによります。通らない場合は学生証画像のアップロードで対応できます。
:::

## 学割がないAIツールとのコスト感

GitHub CopilotとCursorは学生無料ですが、学割のないAIツールもあります。

| ツール | 料金 | 位置づけ |
|--------|------|---------|
| Claude Code | Pro $20/月〜 | ターミナルベースのAI開発ツール。大規模リファクタリング向き |
| ChatGPT | Go $8/月 | 汎用チャットAI。大学がChatGPT Eduを導入していれば無料 |

まずはCopilot（無料）で開発を進めて、「もっと強力なAI支援がほしい」と感じたタイミングで有料ツールを検討する流れが合理的です。

## まとめ

- **GitHub Student Developer Packに申請する**のが最初の一歩です。これが通れば100以上のツール・サービスが無料で使えます
- 申請には**大学メール（.ac.jp）**を使うとスムーズです。審査は多くの場合1日以内に完了します
- Student Packが承認されたら、**VS Code + GitHub Copilot**をセットアップします。コード補完が無制限で使えるようになります
- 余裕があるときに**Cursor、JetBrains、Figma**などを追加で申請しておくと、開発環境がさらに充実します
- 学割のないAIツール（Claude Code、ChatGPT）は、必要になったタイミングで検討すれば十分です
