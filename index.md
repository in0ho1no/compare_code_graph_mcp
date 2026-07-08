---
html:
  embed_local_images: true
  embed_svg: true
  offline: true
  toc: true
export_on_save:
  html: true
---

# Serena / codebase-memory-mcp 比較検証手順書

- 検証スコープ: read-only の構造解釈性能のみ（コード編集機能は対象外）
- 想定エージェント: GitHub Copilot Chat（VS Code、Agent モード）
- 機能比較の実施手順（検証クエリ・正解データ・採点表）は別紙とする
- サンプルリポジトリ: [FreeRTOS-Kernel](https://github.com/FreeRTOS/FreeRTOS-Kernel)  
本書のパス・コマンド例は、このリポジトリを `C:\work\FreeRTOS-Kernel` に取得した前提で記載する。  
別のリポジトリに適用する場合は、パスと clangd 用ビルド構成（Serena の手順3）を読み替える。

---

# 第1部 一般的な紹介

## そもそも MCP サーバとは何か

MCP（Model Context Protocol）は、AI コーディングエージェントと外部ツールを接続するためのオープンな標準プロトコルである。  
GitHub Copilot や Cursor など、主要なエージェントが対応している。

仕組みは単純で、以下の3者で構成される。

```text
[LLM (クラウド)] ←→ [MCPクライアント = Coding Agent] ←→ [MCPサーバ (ローカルプロセス)]
                        (GitHub Copilot 等)             (Serena / codebase-memory-mcp)
```

- MCPサーバは「ツール」（例: `find_symbol`、`trace_call_path`）を提供するだけのプログラムである。  
多くは stdio 上の JSON-RPC で動くローカルプロセスであり、それ自体に AI は入っていない。
- MCPクライアント（Coding Agent）がユーザーの自然言語の質問を解釈し、どのツールをどの引数で呼ぶかを決めて MCP サーバを呼び出す。
- ツールの実行結果はエージェントに返り、エージェントがそれを LLM のコンテキストに載せて回答を生成する。

今回比較する2つのツールはどちらも、構造化されたコード情報をエージェントへ安く正確に供給する索引サービスである。  
回答の言語化はエージェント側の LLM が担う。

## データ送信に関する整理

**ここで紹介する2つのMCPサーバはいずれも、自発的に外部へコードを送信することはない。**  
**コードが社外（LLM API）へ出るのは、Coding Agent が通常の推論リクエストとして送信する経路のみである。**

- MCP サーバはローカルプロセスとして動作し、通信相手は MCP クライアント（エージェント）だけである。
- ツールの実行結果（関数一覧、呼び出しグラフ等）は、エージェントがコンテキストに含めた時点で初めて LLM API へ送られる。  
これは MCP を使わずエージェントがファイルを直接 Read する場合と同じ経路であり、MCP を挟むことで新しい送信経路が増えるわけではない。
- codebase-memory-mcp は README で「100%ローカル動作・テレメトリ収集なし」を明言している。  
インデックス（SQLite DB）もローカル保存される（格納先は環境構築の章を参照）。
- Serena もローカル実行で、インデックスやメモリは `.serena/` 配下にローカル保存される。

コード以外の外部通信として以下がある。

| 通信 | codebase-memory-mcp | Serena |
| --- | --- | --- |
| 初回インストール時 | GitHub Releases からバイナリ取得 | uvx が GitHub からソース取得・依存パッケージ取得 |
| 起動時 | 新バージョンの有無チェック（自動アップデート機構） | 言語サーバ（clangd 等）が未導入の場合の自動ダウンロード |
| 解析対象コード | 送信されない | 送信されない |

> 導入時はバージョン（コミットSHA / リリースタグ）を固定し、設定は `.vscode/mcp.json` でワークスペース単位に閉じる。

## ツール紹介

### codebase-memory-mcp

tree-sitter による AST パースでコードベースを永続的な知識グラフ（SQLite）にインデックスする MCP サーバ。  
C を含む主要言語では Hybrid LSP と呼ばれる型解決を内蔵する。  
`search_graph` / `trace_call_path` / `query_graph`（Cypher）/ dead code 検出などのツールを提供する。  
単一の静的バイナリで、ランタイム依存ゼロ。

#### 強いところ

- セットアップがほぼゼロ。バイナリを置いて `.vscode/mcp.json` に1エントリ書くだけで動く。
- インデックスが高速・軽量で、クエリはミリ秒未満。トークン消費が file-by-file 探索より桁違いに少ない。
- ビルド構成に依存しない「全ソースの網羅ビュー」。portable/ 以下の全ポートが見える。
- read-only 設計なのでコードを壊すリスクが構造的にない。

#### 弱いところ

- tree-sitter はプリプロセス前のソースを見るため、マクロで包装された API のエイリアス解決や、`#if` による可視性の判定は原理的に苦手な可能性が高い（検証項目2・3で確認する）。
- 関数ポインタ経由の間接呼び出しの解決には限界がある（これは Serena/clangd も同様）。
- 編集機能はなく、構造クエリ専用である。

### Serena

LSP（Language Server Protocol）を土台にした「コーディングエージェント用ツールキット」。  
C/C++ では clangd を言語サーバとして使い、シンボルの定義・参照をセマンティックに解決する。  
`find_symbol` / `find_referencing_symbols` / `get_symbols_overview` などのツールを提供する。

#### 強いところ

- clangd はプリプロセス後のコードを解釈するため、マクロ展開・typedef 連鎖・条件コンパイルを正しく追える（組込みCでは決定的な差になりうる）。
- IDE の「定義へ移動」「参照検索」と同等の精度が期待できる。
- 本検証の対象外だが、シンボル編集（リネーム・参照一括更新）も可能である。

#### 弱いところ

- C の精度は `compile_commands.json` の有無と品質に完全に依存する。無いと参照解決が大きく劣化する。
- Python + uv + clangd と依存が多く、セットアップコストが高い。
- clangd は「1つのビルド構成」しか見ない。FreeRTOS のような多ポート構成では、compile_commands.json に含まれないポートは不可視になる。
- 大規模リポジトリでは初回インデックスと言語サーバの起動が重い。

### 役割の違い

| 観点            | codebase-memory-mcp                            | Serena                        |
| --------------- | ---------------------------------------------- | ----------------------------- |
| 解析基盤        | tree-sitter AST + Hybrid LSP（プリプロセス前） | clangd（LSP、プリプロセス後） |
| C解析の前提条件 | なし                                           | compile_commands.json 必須    |
| ビュー          | 全ソースの網羅ビュー                           | 単一ビルド構成の正確なビュー  |
| セットアップ    | ほぼゼロ                                       | 重い                          |
| 位置づけ        | 構造知識グラフのクエリエンジン                 | エージェントが操作するIDE     |

## 環境構築手順

### 共通準備（サンプルリポジトリの取得）

比較の再現性のため、タグを固定して取得する。  
両ツールに同一コミットをインデックスさせることが比較の大前提となる。

```powershell
cd C:\work
git clone --branch V11.3.0 --depth 1 https://github.com/FreeRTOS/FreeRTOS-Kernel.git
cd FreeRTOS-Kernel
git rev-parse HEAD   # コミットSHAを記録に残しておく
```

### 前提条件（GitHub Copilot / VS Code）

1. 組織ポリシーの有効化。  
Copilot Business では「MCP servers in Copilot」ポリシーがデフォルト無効であり、組織管理者が有効化しない限り MCP を使えない。  
技術的な設定より先に詰まりやすいポイントなので、最初に確認する。
2. Agent モード限定。  
MCP ツールは Copilot Chat の Agent モードでのみ使用でき、Ask / Edit モードからは見えない。
3. VS Code は MCP 対応版（1.99 以降、推奨は最新安定版）を使う。

### 設定ファイルの基本（`.vscode/mcp.json`）

ワークスペース単位の設定は、リポジトリ直下の `.vscode/mcp.json` に書く。  
ユーザープロファイル側（`MCP: Open User Configuration`）はグローバル登録になるため使わない。  
グローバル登録すると全プロジェクトの全セッションにツール定義が載り、無関係な作業でもコンテキストを消費するためである。

**ルートキーは `servers` である（`mcpServers` ではない）。**  
他エージェント向けの設定例をコピーしてキーを直し忘れるのが定番のミスなので注意する。

`${workspaceFolder}` などの定義済み変数が使えるため、パスをハードコードせずに書ける。  
DB 実体（後述の `.cbm-cache/`）を gitignore しておけば、このファイル自体はコミットしてチーム共有できる。

両サーバを設定した例を示す。

```json
{
  "servers": {
    "codebaseMemory": {
      "type": "stdio",
      "command": "codebase-memory-mcp",
      "env": {
        "CBM_CACHE_DIR": "${workspaceFolder}\\.cbm-cache"
      }
    },
    "serena": {
      "type": "stdio",
      "command": "uvx",
      "args": [
        "--from", "git+https://github.com/oraios/serena@<コミットSHAまたはタグ>",
        "serena", "start-mcp-server",
        "--context", "ide-assistant",
        "--project", "${workspaceFolder}"
      ]
    }
  }
}
```

比較検証中は一方をコメントアウトし、片方ずつ有効化して実施する（別紙参照）。

### codebase-memory-mcp のセットアップ（`--skip-config` 前提）

方針:  
インストーラの自動設定（エージェント設定・スキル・フックの書き込み）は使わない。  
`--skip-config` でバイナリ配置のみ行い、`.vscode/mcp.json` は手書きで管理する。  
エージェント設定を勝手に書き換えられないので、何がどこに入ったかを把握できる。  
トレードオフとして、本来 `install` が仕込む指示ファイルとフック（Grep 時にグラフ検索結果を非ブロッキングで添える補助）が入らない。  
そのため、実運用時のグラフツールへの誘導は copilot-instructions.md に自分で書く必要がある（付録A）。

#### バイナリ入手と検証

Releases から Windows 用 zip（`codebase-memory-mcp-windows-amd64.zip`）を取得する。  
`checksums.txt` の SHA-256 と Attestation を検証しておく。  

```powershell
gh attestation verify .\codebase-memory-mcp-windows-amd64.zip --repo DeusData/codebase-memory-mcp
```

問題ないことが確認できたら圧縮ファイルを展開する。

```powershell
Expand-Archive codebase-memory-mcp-windows-amd64.zip -DestinationPath ./codebase-memory-mcp
```

エージェント設定は書き換えず、バイナリ配置とPATH追加のみ行うように`--skip-config`を指定して実行する。

```powershell
.\codebase-memory-mcp\install.ps1 --skip-config
```

インストーラは PATH への追加まで行うため、`.vscode/mcp.json` の `command` はコマンド名だけでよい。（前出の設定例）。  
ただしユーザー PATH の変更は既存プロセスに反映されない。  
インストール直後は新しいターミナルで `where.exe codebase-memory-mcp` が通ることを確認し、VS Code は完全に再起動（終了→起動）させる。

バージョンは以下で確認できる。

```powershell
codebase-memory-mcp --version
```

#### DB（インデックス）をワークスペース内に配置する

デフォルトでは SQLite DB がユーザープロファイル配下（`~/.cache/codebase-memory-mcp/`）に作られ、リポジトリから存在が見えない。  
環境変数 `CBM_CACHE_DIR` で格納先を上書きできるので、`.vscode/mcp.json` の `env` で `${workspaceFolder}\.cbm-cache` を指定する（前出の設定例）。  
あわせて `.gitignore` に追記する（今回はインデックスをコミットしない方針）。

```gitignore
.cbm-cache/
.codebase-memory/
```

> `.codebase-memory/` は Team-Shared Graph Artifact 用のディレクトリで、インデックス実行時に圧縮スナップショット `graph.db.zst` がリポジトリ直下に書かれることがある。  
> 将来チームでインデックスを共有したくなったら、これをあえてコミットする選択肢がある（クローンした側は再インデックス不要になる公式機能）。今回は gitignore しておく。

#### インデックス作成

エージェント経由でも CLI でも実行できるが、無駄にトークン消費しいないよう CLI から実行する。  
**CLI から操作する場合も、MCP サーバと同じ DB を見るように同じ `CBM_CACHE_DIR` をシェル側に設定してから実行する。**  
これを忘れると CLI はデフォルトのユーザーディレクトリ側に別の DB を作ってしまう。  
「CLI ではインデックス済みなのにエージェントからは見えない」という事故になるため注意する。  
なお `${workspaceFolder}` は VS Code の変数であり、シェルでは展開されないので実パスで指定する。  

```powershell
$env:CBM_CACHE_DIR = "C:\work\FreeRTOS-Kernel\.cbm-cache"
codebase-memory-mcp cli index_repository --repo-path "C:\\work\\FreeRTOS-Kernel" | ConvertFrom-Json | ConvertTo-Json -Depth 10
codebase-memory-mcp cli list_projects | ConvertFrom-Json | ConvertTo-Json -Depth 10
```

> Windows PowerShell 5.1 の `ConvertTo-Json` は非ASCII文字を `\uXXXX` にエスケープするため、日本語を含む出力は PowerShell 7（`pwsh`）の方が読みやすい。  

#### DB の更新忘れに気付ける仕組み

ソースを更新したのにインデックスが古いまま、という状態を検出できるようにしておく。  

codebase-memory-mcpには git ポーリングでファイル変更を検知して増分再インデックスする設定が存在する。  
設定の確認・変更は `config list` / `config set` で行う。  
なお設定はインデックスと同じ格納ディレクトリに保存されるため、前項と同様に `CBM_CACHE_DIR` をシェルに設定してから実行する。

```powershell
$env:CBM_CACHE_DIR = "C:\work\FreeRTOS-Kernel\.cbm-cache"
codebase-memory-mcp config list                  # auto_watch: true を確認
codebase-memory-mcp config set auto_watch true   # false だった場合に有効化する
```

> ウォッチャは git ベースの変更検知なので、コミットされていない編集への追従タイミングは保証を当てにしないこと。  

検証（別紙）では、各クエリ実施前にこのチェックを通すことを手順に含める。

#### グラフツールへの誘導記述（実運用向け・比較検証では不要）

`--skip-config` では誘導用の指示ファイルが入らないため、実運用時は付録Aの雛形を `.github/copilot-instructions.md` に書く。  
比較検証ではプロンプト側でツール使用を強制するため不要である（別紙参照）。

---

## 付録A: 実運用での素の Grep と MCP ツールの使い分け

比較検証では公平性のため内蔵ツールを封じるが、実運用で Grep を禁止してはいけない。  
理由と使い分けの指針を示す。

### なぜ禁止しないのか

1. 役割が異なる。  
グラフ/LSP が得意なのは構造クエリ（呼び出し元・依存・影響範囲・dead code）である。  
一方、文字列リテラル・コメント・ログメッセージ・設定キーの検索はテキスト検索の領域であり、グラフには入っていないか不完全である。  
codebase-memory-mcp 自身のスキル定義も「テキスト検索は grep/Glob、単一ファイル読み取りは Read を使え」と明記している。  
封じると答えられない質問が生じ、エージェントが幻覚で埋めるリスクが上がる。
2. トークン消費で高くつくのは Grep そのものではなく「探索ループ」である。  
構造的な質問を grep → ファイル全体を Read → また grep… と回すパターンが高コストになる。  
（構造クエリ5問でグラフ経由 約3,400トークン vs file-by-file 探索 約41.2万トークン、という報告値がある。）  
パターンが明確な単発 grep はマッチ行しか返らず数百トークンで済み、グラフクエリと大差ない。
3. フォールバック・クロスチェックとしての価値がある。  
インデックスが古い/パース漏れがあるとき、grep での裏取りが誤答を防ぐ。

### 使い分けの指針（copilot-instructions.md に書く内容の雛形）

```markdown
## コード調査ツールの使い分け
- 構造的な質問（この関数を呼ぶのは誰か / 何に依存するか / 変更の影響範囲 /
  dead code）は、grep で探索せず codebase-memory-mcp のグラフツール
  （trace_call_path, search_graph, detect_changes 等）を最初に使うこと。
  1回のグラフクエリで済む質問を grep + Read の繰り返しで調べないこと。
- 文字列リテラル・コメント・ログメッセージ・設定値の検索は Grep を使うこと。
- 単一ファイルの内容確認は Read を使うこと。
- グラフの結果が疑わしい場合（0件・鮮度警告あり）は、インデックスの
  再作成を提案した上で grep で裏取りしてよい。
```

> `--skip-config` を使わない標準インストールでは、検出したエージェントに対して同趣旨の指示ファイルやスキル、Grep 時にグラフ検索結果を添える非ブロッキングのフックが自動で入る。  
> `--skip-config` 運用では、この雛形を copilot-instructions.md に書くことで代替する。

### まとめ

| 質問の種類 | 使うべきツール | 理由 |
| --- | --- | --- |
| 呼び出し元・依存・影響範囲 | グラフ/LSP（MCP） | 1クエリで完結、トークン最小 |
| 文字列・コメント・設定値検索 | Grep | グラフに載っていない情報 |
| ファイル内容の精読 | Read | 構造化不要 |
| MCP の結果が0件・鮮度不明 | Grep で裏取り | 幻覚・古いインデックスの検出 |
