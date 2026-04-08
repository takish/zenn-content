---
title: "GCP複数アカウントをdirenvで自動切替 — gcloud authとADCの違いも解説"
emoji: "🔀"
type: "tech"
topics: ["gcp", "gcloud", "direnv", "multicloud", "devops"]
published: false
---

## 切り替えたつもりでデプロイしたらエラーになった

複数の GCP アカウントを使い分けていると、こんな場面に遭遇します。

- アカウントを切り替えたつもりで `gcloud run deploy` したら、別プロジェクトへのデプロイでエラー
- 作業のたびに `gcloud auth list` → `gcloud config set account ...` → `gcloud config set project ...` の確認が煩雑
- 今どのアカウント・プロジェクトにいるのか、毎回不安になる

さらに厄介なのが、gcloud CLI のアカウントを正しく切り替えても、BigQuery や Terraform などの SDK が参照する **ADC（Application Default Credentials）は切り替わらない** という問題です。

```bash
$ gcloud config get account
your-email@company.com  # ← 正しい

$ python3 -c "from google.cloud import bigquery; print(bigquery.Client().project)"
# → 403 Forbidden（なぜか個人アカウントの認証情報が使われている）
```

この記事では、`gcloud auth` と ADC の違いを整理した上で、direnv による自動切り替えの設定手順をまとめます。

---

## この記事で得られること

- `gcloud auth login` と ADC の違い — なぜ SDK で認証エラーが出るのか
- direnv + gcloud configurations による自動切り替えの設定手順（アカウント・プロジェクト・リージョンを丸ごと切替）
- ADC も含めた `cd` だけで完結する構成
- BigQuery・Terraform など SDK 経由のアクセスで認証が通らない問題の解決

---

## 前提

### 想定するシナリオ

| アカウント | 用途 | プロジェクト |
|-----------|------|-------------|
| `your-email@company.com` | 業務用 | company-prod, company-dev |
| `your-personal@gmail.com` | 個人開発 | my-side-project |

この2つ（以上）を日常的に切り替える必要がある状況を想定します。

### GCP の認証は 2 系統ある

GCP の認証には**2つの独立した仕組み**があり、これが混乱の元です。

| 認証の種類 | コマンド | 用途 | 保存先 |
|-----------|---------|------|--------|
| **gcloud CLI 認証** | `gcloud auth login` | gcloud コマンド | gcloud CLI の内部ストレージ |
| **ADC** | `gcloud auth application-default login` | SDK・Terraform・BigQuery | `~/.config/gcloud/application_default_credentials.json` |

**この2つは完全に別管理です。** gcloud のアカウントを切り替えても、ADC は切り替わりません。BigQuery クライアントや Terraform が「認証エラー」を返す原因の多くがこれです。

---

## アカウント切り替えの手法比較

GCP アカウント切り替えの主な方法を整理します。

### 1. `gcloud config configurations`（組み込み機能）

```bash
# プロファイル作成
gcloud config configurations create work
gcloud auth login your-email@company.com  # ブラウザ認証
gcloud config set project company-dev

gcloud config configurations create personal
gcloud auth login your-personal@gmail.com  # ブラウザ認証
gcloud config set project my-side-project

# 切り替え
gcloud config configurations activate work
```

- **メリット**: gcloud標準機能、追加インストール不要
- **デメリット**: 手動切り替え、忘れると事故

### 2. `CLOUDSDK_ACTIVE_CONFIG_NAME` 環境変数

```bash
# ターミナル単位で切り替え
export CLOUDSDK_ACTIVE_CONFIG_NAME=work

# 別のターミナルでは
export CLOUDSDK_ACTIVE_CONFIG_NAME=personal
```

- **メリット**: ターミナルごとに独立、グローバル設定を汚さない
- **デメリット**: 毎回exportが必要

### 3. コマンドフラグ（`--account`, `--project`）

```bash
gcloud compute instances list --account=your-email@company.com --project=company-dev
```

- **メリット**: 明示的、状態を持たない
- **デメリット**: 冗長、毎回書くのは現実的でない

### 4. direnv + .envrc（自動切り替え）

```bash
# プロジェクトディレクトリに入ると自動で環境変数が設定される
cd ~/work/company-project
# → CLOUDSDK_ACTIVE_CONFIG_NAME=work が自動適用
```

- **メリット**: 完全自動、事故防止、ディレクトリを出ると解除される
- **デメリット**: direnv のインストールが必要、IDE の統合ターミナルではシェルフック設定が必要

### 5. サードパーティツール（gcloud-ctx 等）

kubectx にインスパイアされたRust製ツール。

```bash
gcloud-ctx        # 一覧表示
gcloud-ctx work   # 切り替え
```

- **メリット**: kubectxユーザーに馴染みやすい
- **デメリット**: 外部バイナリ依存

### 比較表

| 手法 | 自動化 | ポータブル | 事故防止 | 導入コスト |
|------|--------|-----------|---------|-----------|
| configurations activate | 手動 | ◎ | △ | なし |
| CLOUDSDK_ACTIVE_CONFIG_NAME | ターミナル単位 | ◎ | ○ | なし |
| コマンドフラグ | なし | ◎ | ◎ | なし |
| **direnv + .envrc** | **cd で自動** | **○** | **◎** | **低** |
| gcloud-ctx | 手動 | △ | △ | 低 |

---

## セットアップ: direnv + gcloud configurations

direnv はディレクトリごとに環境変数を自動設定・解除するツールです。GCP に限らず、AWS・Terraform・Node.js 等の環境管理にも使われています。ここでは gcloud configurations と組み合わせた設定手順をまとめます。

#### Step 1: direnv のインストール

```bash
brew install direnv
```

シェルフックを追加します（これが唯一のシェル設定変更です）:

```bash
# zsh
echo 'eval "$(direnv hook zsh)"' >> ~/.zshrc

# bash
echo 'eval "$(direnv hook bash)"' >> ~/.bashrc

# fish
echo 'direnv hook fish | source' >> ~/.config/fish/config.fish
```

#### Step 2: gcloud configurations の作成

```bash
# 業務用
gcloud config configurations create work
gcloud config configurations activate work
gcloud auth login your-email@company.com  # ブラウザが開き、認証を求められる
gcloud config set project company-dev
gcloud config set compute/region asia-northeast1

# 個人用
gcloud config configurations create personal
gcloud config configurations activate personal
gcloud auth login your-personal@gmail.com  # 同様にブラウザ認証
gcloud config set project my-side-project
gcloud config set compute/region us-central1
```

configuration にはアカウント・プロジェクト・リージョンがまとめて保存されます。切り替え時にはこれらが丸ごと変わるため、アカウントとプロジェクトを別々に `set` し直す必要はありません。

設定を確認します:

```bash
gcloud config configurations list
# NAME      IS_ACTIVE  ACCOUNT                    PROJECT
# default   False      -                          -
# personal  True       your-personal@gmail.com    my-side-project
# work      False      your-email@company.com     company-dev
```

#### Step 3: プロジェクトごとに `.envrc` を配置

```bash
# 業務プロジェクト
cd ~/work/company-project
echo 'export CLOUDSDK_ACTIVE_CONFIG_NAME=work' > .envrc
direnv allow

# 個人プロジェクト
cd ~/personal/side-project
echo 'export CLOUDSDK_ACTIVE_CONFIG_NAME=personal' > .envrc
direnv allow
```

`direnv allow` は `.envrc` の内容をホワイトリストに登録するセキュリティ機構です。`.envrc` が変更されると自動的にブロックされ、再度 `allow` が必要になります。信頼できないリポジトリの `.envrc` を安易に `allow` しないよう注意してください。

#### 結果: cd するだけで切り替わる

```bash
$ cd ~/work/company-project
direnv: loading .envrc
direnv: export +CLOUDSDK_ACTIVE_CONFIG_NAME

$ gcloud config get account
your-email@company.com

$ cd ~/personal/side-project
direnv: loading .envrc
direnv: export ~CLOUDSDK_ACTIVE_CONFIG_NAME

$ gcloud config get account
your-personal@gmail.com

$ cd ~
direnv: unloading  # ← ディレクトリを出ると解除される
```

---

## ADC の罠と対処法

冒頭で触れた「gcloud は正しいのに SDK でエラーが出る」問題の原因と対処法を整理します。

### 原因: gcloud CLI 認証と ADC は独立している

```
gcloud CLI                    SDK / Terraform / BigQuery
    │                              │
    ▼                              ▼
gcloud auth login             gcloud auth application-default login
    │                              │
    ▼                              ▼
gcloud 内部ストレージ          application_default_credentials.json
（configurations で切替可能）   （グローバルに1つ。GOOGLE_APPLICATION_CREDENTIALS で上書き可能）
```

**gcloud configurations を切り替えても ADC は切り替わりません。** これが SDK で 403 エラーが出る主な原因です。

### 対処法 1: ADC を都度ログインし直す（シンプル）

```bash
gcloud config configurations activate work
gcloud auth application-default login
# → ブラウザでログイン → ADCが上書きされる
```

手動ですが、頻度が低ければ十分です。

### 対処法 2: GOOGLE_APPLICATION_CREDENTIALS で切り替え（自動化）

ADC のクレデンシャルファイルを configuration ごとに保存し、direnv で切り替えます。

```bash
# 業務用ADCでログイン
gcloud config configurations activate work
gcloud auth application-default login
cp ~/.config/gcloud/application_default_credentials.json \
   ~/.config/gcloud/adc-work.json
chmod 600 ~/.config/gcloud/adc-work.json

# 個人用ADCでログイン
gcloud config configurations activate personal
gcloud auth application-default login
cp ~/.config/gcloud/application_default_credentials.json \
   ~/.config/gcloud/adc-personal.json
chmod 600 ~/.config/gcloud/adc-personal.json
```

ADC ファイルにはリフレッシュトークンが含まれているため、ファイル権限は 600 に設定します。また、`gcloud auth application-default login` を再実行すると元の `application_default_credentials.json` が上書きされるため、その場合はコピーファイルも更新が必要です。

`.envrc` に追記:

```bash
# ~/work/company-project/.envrc
export CLOUDSDK_ACTIVE_CONFIG_NAME=work
export GOOGLE_APPLICATION_CREDENTIALS="${HOME}/.config/gcloud/adc-work.json"
```

```bash
# ~/personal/side-project/.envrc
export CLOUDSDK_ACTIVE_CONFIG_NAME=personal
export GOOGLE_APPLICATION_CREDENTIALS="${HOME}/.config/gcloud/adc-personal.json"
```

`GOOGLE_APPLICATION_CREDENTIALS` のパスには `~` ではなく `${HOME}` を使います。チルダ展開はシェルの機能であり、一部の SDK やライブラリは環境変数の値に含まれるチルダを展開しません。

BigQuery 等で quota project のエラーが出る場合は、ADC に quota project を設定します:

```bash
gcloud auth application-default set-quota-project company-dev
```

これで gcloud CLI と SDK の両方が `cd` だけで切り替わる構成になります。

---

## よくある質問

### Q: JSON キーファイルをダウンロードする必要がある？

**個人アカウントでは不要です。** `gcloud auth login` も `gcloud auth application-default login` もブラウザ認証で完結します。

JSON キーファイルが必要なのは **サービスアカウント** を使う場合（CI/CD、サーバー上のバッチ処理など）です。ローカル開発では個人認証が推奨されています。

### Q: トークンの有効期限は？

アクセストークンの有効期限は約1時間ですが、リフレッシュトークンにより自動更新されます。通常の利用では期限切れを意識する必要はありません。ただし、パスワード変更や長期間未使用の場合にリフレッシュトークンが失効することがあり、その際は `gcloud auth login`（および ADC を使っている場合は `gcloud auth application-default login` とファイルの再コピー）が必要です。configurations の設定自体は消えません。

### Q: .envrc を .gitignore に入れるべき？

**はい。** `.envrc` にはアカウント名やプロジェクトIDが含まれるため、チームリポジトリでは `.gitignore` に追加しましょう。

```gitignore
.envrc
```

代わりに `.envrc.example` をコミットしておくと親切です:

```bash
# .envrc.example
export CLOUDSDK_ACTIVE_CONFIG_NAME=your-config-name
# export GOOGLE_APPLICATION_CREDENTIALS="${HOME}/.config/gcloud/adc-your-config.json"
```

### Q: AWS や Azure も同時に管理できる？

`.envrc` に複数のクラウドの設定をまとめられます:

```bash
# .envrc（マルチクラウドプロジェクト）
export CLOUDSDK_ACTIVE_CONFIG_NAME=work
export AWS_PROFILE=company-dev
export GOOGLE_APPLICATION_CREDENTIALS="${HOME}/.config/gcloud/adc-work.json"
```

---

## direnv の制約

direnv は万能ではありません。以下の点は把握しておく必要があります。

- **IDE の統合ターミナル**: VS Code や JetBrains の統合ターミナルではシェルフックの設定次第で direnv が動作しない場合がある。各 IDE の direnv 拡張やプラグインの導入を検討してください
- **サブシェル・スクリプト**: `bash -c "gcloud ..."` のようにサブシェルを起動した場合、direnv のフックは効きません。環境変数自体は子プロセスに引き継がれるため、同一シェルセッション内で設定済みであれば問題ありませんが、新しいシェルを起動する場合は注意が必要です
- **CI/CD 環境**: direnv はローカル開発向けのツールです。CI/CD ではサービスアカウントキーや Workload Identity Federation など、別の認証方式を使います

---

## まとめ

| 何を | どう管理するか |
|------|--------------|
| gcloud CLI の認証 | `gcloud config configurations` で切り替え |
| SDK/Terraform の認証（ADC） | `GOOGLE_APPLICATION_CREDENTIALS` で切り替え |
| 切り替えの自動化 | `direnv` + `.envrc` で `cd` するだけ |

ポイントを整理すると:

1. **gcloud auth と ADC は別物** — gcloud のアカウントを切り替えても ADC は切り替わらない
2. **`GOOGLE_APPLICATION_CREDENTIALS` で ADC の切り替えも自動化できる** — パスには `${HOME}` を使う
3. **JSON キーファイルは個人開発では不要** — ブラウザ認証で十分
4. **direnv には制約がある** — IDE ターミナルや CI/CD では別途対応が必要

---

## 参考リンク

- [gcloud CLI - Managing configurations](https://cloud.google.com/sdk/docs/configurations)
- [Application Default Credentials](https://cloud.google.com/docs/authentication/application-default-credentials)
- [direnv 公式ドキュメント](https://direnv.net/)
