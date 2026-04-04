---
title: "Ghosttyのタブ × tmuxで開発画面を瞬時に切り替える — Cmd+数字のワークスペース設計"
emoji: "👻"
type: "tech"
topics: ["ghostty", "tmux", "terminal", "macos"]
published: false
---

# ターミナル6画面を Cmd+数字 で切り替える ── Ghostty Tab × tmux 3層構成

ターミナルで複数プロジェクトを同時に開いていると、「この画面群をまとめて切り替えたい」と感じる瞬間があります。IDEならワークスペースやプロジェクト切り替えで実現できますが、ターミナル環境ではどうすればいいのか。

本記事では Ghostty + tmux の組み合わせで、**画面の「塊」単位で切り替え可能なワークスペース**を構築する方法を解説します。結論としては、Ghostty Tab をワークスペースとして使い、各Tab内でtmuxセッションを並列管理する3層構造に落ち着きました。

---

## この記事で得られること

- Ghostty と tmux の分割機能の使い分け
- 複数プロジェクトの画面群を「塊」で切り替える3つの方法と比較
- tmux の `join-pane` / `break-pane` を活用した柔軟なレイアウト管理
- 実用的なキーバインド設定と Ghostty config テンプレート

## 前提環境

- **ターミナル**: Ghostty 1.2.0+（macOS）— `Cmd+数字` によるタブ切り替えがデフォルトで使える
- **マルチプレクサ**: tmux（prefix: `C-t`）
- **用途**: 複数プロジェクトでCLI（Claude Code等）を並行して運用

---

## tmux の階層構造 ── まず用語を整理する

この記事ではSession / Window / Pane という用語が頻出するため、先に整理しておきます。

```
Server
└─ Session（セッション）── プロジェクト単位で作る大きな箱
     └─ Window（ウィンドウ）── タブに相当。セッション内で複数持てる
          └─ Pane（ペイン）── 画面分割の最小単位
```

| 概念 | 比喩 | 切り替え |
|------|------|---------|
| Session | プロジェクト | `C-t s` で一覧選択 |
| Window | タブ | `C-t n/p` または `C-t 数字` |
| Pane | 画面分割 | `C-t 矢印` |

ポイントは、**Sessionがtmuxの最上位の管理単位**であることです。Session間はウィンドウ構成やレイアウトが独立しており、あるSessionでのWindow操作やPane分割が別Sessionに影響しません。この性質がプロジェクト管理に効きます。

---

## 現状の構成 ── Ghostty Split × tmux セッション

筆者の環境では、Ghosttyの画面分割（`Cmd+D` / `Cmd+Shift+D`）で6つのペインを作り、それぞれの中で独立したtmuxセッションを起動しています。

![改善前: Ghostty Split 1枚に全プロジェクトが並んだ状態](/images/ghostty-tmux-before.jpeg)
*Ghostty Split で6つの tmux セッションを1画面に並べた状態。常に全プロジェクトが視界に入る*

```
Ghostty ウィンドウ（1枚に全部入り）
├─ Ghostty Split 1 → tmux session: web-app
├─ Ghostty Split 2 → tmux session: content-hub
├─ Ghostty Split 3 → tmux session: mobile-app
├─ Ghostty Split 4 → tmux session: blog-admin
├─ Ghostty Split 5 → tmux session: dotfiles
└─ Ghostty Split 6 → tmux session: asset-tools
```

各tmuxセッション内では `C-t c` でWindowを複数作り、`C-t p` / `C-t n` で切り替えています。

### この構成の良い点

- **プロジェクト完全分離**: 各tmuxセッションが独立しているため、誤操作の影響範囲が限定される
- **セッション復元**: tmux-resurrect/continuum による自動保存・復元がセッション単位で効く
- **ステータスバー**: セッション名ごとに色分けされ、どのプロジェクトか一目でわかる

### 課題

**「画面の塊」を切り替えられない。**

例えば web-app / content-hub / mobile-app の「開発系3つ」と、blog-admin / dotfiles の「コンテンツ系2つ」をグループ化して、ワンアクションで切り替えたいところです。今の構成ではGhostty Splitが1枚のウィンドウに全部並んでいるため、常に6プロジェクトが視界に入ります。

画面が多すぎて「今どこを見ているのか」を認知する負荷が地味にかかっていました。

---

## 解決策の比較 ── 3つのアプローチ

### 方法1: Ghostty Tab を使う（採用）

Ghosttyには **Tab** 機能があります。各Tabが独立したSplit構成を持てるため、「タブ = ワークスペース」として使えます。

```
Tab 1 "開発" (Cmd+1):
├─ Split: web-app (tmux)
├─ Split: content-hub (tmux)
└─ Split: mobile-app (tmux)

Tab 2 "コンテンツ" (Cmd+2):
├─ Split: blog-admin (tmux)
└─ Split: dotfiles (tmux)

Tab 3 "インフラ" (Cmd+3):
└─ Split: asset-tools (tmux)
```

**メリット:**
- 既存のtmuxセッション構成を一切変えなくていい
- `Cmd+数字` で瞬時にワークスペース切り替え
- 学習コストほぼゼロ

**デメリット:**
- Ghostty依存（他のターミナルに移行する場合、Tab構成は持っていけない）

### 方法2: Ghostty Window を使う

`Cmd+N` で新しいGhosttyウィンドウを作り、macOSの Mission Control や `Cmd+`` で切り替える方法です。

**メリット:**
- ウィンドウごとにサイズ・配置を変えられる
- 外部ディスプレイとの組み合わせに強い

**デメリット:**
- ウィンドウ切り替えが macOS 依存で、Tab (`Cmd+数字`) より遅い
- ウィンドウが増えると Mission Control が煩雑になる

### 方法3: tmux側に寄せる（構成変更）

Ghostty分割をやめて、1つのtmuxセッション内でWindowをワークスペースとして使う方法です。

```
tmux session: work
├─ Window 1 "開発":     [web-app pane | content-hub pane | mobile-app pane]
├─ Window 2 "コンテンツ": [blog-admin pane | dotfiles pane]
└─ Window 3 "インフラ":   [asset-tools pane]
```

`C-t n` / `C-t p` や `C-t 数字` でWindow単位に切り替えます。ターミナルエミュレータに依存しない純tmux構成です。

**メリット:**
- ターミナルエミュレータに依存しない（SSH先でも同じ構成が使える）
- `C-t w` でツリー表示から選択できる

**デメリット:**
- 各pane内で別のtmuxセッションを起動するにはネスト対応（`TMUX` 環境変数の解除や `-L` でのソケット分離）が必要で、設定が煩雑になる
- ステータスバーが1本になり、セッション別の色分けが失われる
- セッション単位のresurrect復元粒度が変わる（全Windowが1セッションに束ねられる）

### 比較表

| 観点 | 方法1: Ghostty Tab | 方法2: Ghostty Window | 方法3: tmux Window |
|------|-------------------|---------------------|-------------------|
| 切り替え速度 | `Cmd+数字` で瞬時 | Mission Control経由 | `C-t 数字` で瞬時 |
| 既存構成の変更 | 不要 | 不要 | 大幅に必要 |
| ターミナル依存 | Ghostty依存 | Ghostty依存 | なし |
| セッション独立性 | 完全に維持 | 完全に維持 | 失われる |
| resurrect復元 | セッション単位で維持 | セッション単位で維持 | Window単位に変更 |
| SSH先での再現 | 不可 | 不可 | 可能 |

**結論: 方法1（Ghostty Tab）を採用しました。** 既存のtmuxセッション構成を維持したまま、上位のレイヤーでグルーピングできるのが決め手でした。方法3も魅力的ですが、tmuxセッションの独立性（resurrect復元、ステータスバー色分け）を捨てるコストが大きいと判断しました。

---

## なぜ「レイヤーの使い分け」がうまくいくのか

今回の構成は、Ghosttyとtmuxの**得意なことをそれぞれ任せる**設計になっています。

| レイヤー | 担当 | 得意なこと |
|---------|------|-----------|
| Ghostty Tab | ワークスペース切り替え | GUI統合（`Cmd+数字`）、視覚的なタブバー |
| Ghostty Split | プロジェクト並列表示 | フォント描画、GPU描画の管理 |
| tmux Session/Window/Pane | プロジェクト内の多層管理 | セッション永続化、復元、リモート接続 |

逆パターン、つまり全部をtmuxに押し込めようとすると破綻しやすくなります。tmuxはGUIのタブバーを持たないため、どのWindowが「開発グループ」でどれが「コンテンツグループ」なのかが視覚的にわかりにくくなります。ステータスバーにWindow一覧は出ますが、6〜8個並ぶと認知負荷が増えます。

一方、全部をGhostty側に寄せる（tmuxを使わない）と、セッション永続化や復元を失います。ターミナルを閉じたら全部消えてしまいます。

**各レイヤーが自分の得意分野だけ担当する**という分離の原則は、ターミナル環境の設計でも効きます。

---

## Ghostty の設定テンプレート

Tab機能とSplit機能を使うための `~/.config/ghostty/config` の設定例を紹介します。macOS ではタブが2つ以上あればネイティブのタブバーが自動表示されるため、タブバー表示の設定は不要です。

Ghostty 1.2.0 以降では、`Cmd+1〜9` による `goto_tab` や `Cmd+D` による `new_split` はデフォルトでバインドされています。以下は筆者の設定ですが、1.2.0+ を使っていればほぼそのまま動きます。デフォルトキーバインドの一覧は `ghostty +list-keybinds` で確認できます。

```ini
# ── Tab ──
# 新しいタブを開く（デフォルト）
keybind = super+t=new_tab

# タブ間移動（デフォルト）
keybind = super+shift+right_bracket=next_tab
keybind = super+shift+left_bracket=previous_tab

# タブ番号でジャンプ（1.2.0+ デフォルト）
keybind = super+one=goto_tab:1
keybind = super+two=goto_tab:2
keybind = super+three=goto_tab:3
keybind = super+four=goto_tab:4
keybind = super+five=goto_tab:5

# ── Split ──
# 水平分割 / 垂直分割（デフォルト）
keybind = super+d=new_split:right
keybind = super+shift+d=new_split:down

# Split 間移動（デフォルト）
keybind = super+alt+left=goto_split:left
keybind = super+alt+right=goto_split:right
keybind = super+alt+up=goto_split:top
keybind = super+alt+down=goto_split:bottom
```

上記はすべて Ghostty 1.2.0+ のデフォルトと同じです。明示的に書いておくと設定の見通しがよくなるため記載していますが、設定ファイルがなくても動作します。

### キーバインド一覧

| キー | 動作 |
|------|------|
| `Cmd+T` | 新しいタブ作成 |
| `Cmd+Shift+]` | 次のタブ |
| `Cmd+Shift+[` | 前のタブ |
| `Cmd+1〜5` | タブ番号でジャンプ |
| `Cmd+D` | 右に分割 |
| `Cmd+Shift+D` | 下に分割 |
| `Cmd+Alt+矢印` | Split間移動 |

---

## 補足テクニック: tmux の Window ⇔ Pane 変換

tmuxのWindowは「1つずつ全画面表示」が前提ですが、`join-pane` コマンドで**別Windowの内容を今の画面にpaneとして結合**できます。

### 基本コマンド

```bash
# Window 2 のpaneを今のWindowに結合
# prefix + : でコマンドモードに入って実行
join-pane -s :2

# 今のpaneを独立Windowに分離（デフォルトバインド）
# prefix + !
break-pane
```

### 便利なキーバインド

`~/.tmux.conf` に以下を追加すると、ツリーUIから直感的に操作できます。

```bash
# prefix + J: ツリーから選択して今のWindowにpaneとして結合
bind J choose-tree "join-pane -s '%%'"
```

操作フロー:
1. `C-t J` を押す
2. セッション/Windowのツリーが表示される
3. 持ってきたいWindowを選択（※現在のWindowは選択できない）
4. 今の画面にpaneとして結合される

現在のWindowを選択すると「Can't join a pane to its own window」エラーになります。必ず別のWindowを選んでください。

戻したい時は `C-t !` で独立Windowに分離できます。

### ユースケース

- 普段は別Windowにしている「ログ監視」を、デバッグ中だけ横に並べたい
- 一時的に2つのプロジェクトのターミナルを見比べたい

Windowを「非表示のpane退避先」として扱い、必要な時だけ `join-pane` で引っ張ってくる運用が実用的でした。

---

## 筆者の最終構成

実際に運用している構成を公開します。

```
Ghostty Tab 1 "開発" (Cmd+1)
├─ Ghostty Split → tmux session: web-app
│    ├─ Window 1: claude-code
│    └─ Window 2: dev server + shell
├─ Ghostty Split → tmux session: content-hub
│    └─ Window 1: claude-code
└─ Ghostty Split → tmux session: mobile-app
     ├─ Window 1: claude-code
     └─ Window 2: dev server

Ghostty Tab 2 "コンテンツ" (Cmd+2)
├─ Ghostty Split → tmux session: blog-admin
│    └─ Window 1: claude-code
└─ Ghostty Split → tmux session: dotfiles
     └─ Window 1: shell

Ghostty Tab 3 "インフラ" (Cmd+3)
└─ Ghostty Split → tmux session: asset-tools
     └─ Window 1: shell

tmux.conf 追加設定:
  bind J choose-tree "join-pane -s '%%'"
```

Ghostty Tab でワークスペース切り替え、各Tab内はGhostty Splitで並列表示、各Split内はtmuxで多層管理。3つのレイヤーがそれぞれ異なる粒度を担当する構成になっています。

### Ghostty 再起動時の挙動

tmuxセッションはターミナルとは独立したサーバープロセスとして動作するため、Ghostty を閉じてもセッションは生き続けます。再起動後に `tmux attach -t web-app` で再接続すれば、作業状態がそのまま復元されます。

ただし、**Ghostty の Tab 構成やSplit レイアウトは永続化されません**。再起動するとTab 1枚のウィンドウに戻ります。Tab/Splitの再構成は手動で行う必要がある点は留意しておいてください。tmux側の状態が保全されるため、実運用では「Ghostty を開く → Tab/Splitを作る → 各Splitで `tmux attach` する」という手順になります。

---

## まとめ

| やりたいこと | 解決策 |
|------------|--------|
| 画面の「塊」をワンアクションで切り替えたい | Ghostty Tab (`Cmd+数字`) |
| tmux Window の中身を一時的に並べて見たい | `join-pane` / `break-pane` |
| プロジェクトごとのセッション独立性を維持したい | 現状維持（Ghostty Split × tmuxセッション） |

ターミナル環境の「ワークスペース」は、ターミナルエミュレータとtmuxの**レイヤーの使い分け**で実現できます。各レイヤーに得意な粒度を任せて、役割を重複させないのがこの構成のポイントでした。
