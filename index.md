---
html:
  embed_local_images: true
  embed_svg: true
  offline: true
  toc: true
export_on_save:
  html: true
---

# Serena / codebase-memory-mcp / CodeGraph 比較検証手順書

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

今回比較するツールはいずれも、構造化されたコード情報をエージェントへ安く正確に供給する索引サービスである。  
回答の言語化はエージェント側の LLM が担う。

## データ送信に関する整理

**ここで紹介するMCPサーバはいずれも、自発的に外部へコードを送信することはない。**  
**コードが社外（LLM API）へ出るのは、Coding Agent が通常の推論リクエストとして送信する経路のみである。**

- MCP サーバはローカルプロセスとして動作し、通信相手は MCP クライアント（エージェント）だけである。
- ツールの実行結果（関数一覧、呼び出しグラフ等）は、エージェントがコンテキストに含めた時点で初めて LLM API へ送られる。  
これは MCP を使わずエージェントがファイルを直接 Read する場合と同じ経路であり、MCP を挟むことで新しい送信経路が増えるわけではない。
- codebase-memory-mcp は README で「100%ローカル動作・テレメトリ収集なし」を明言している。  
インデックス（SQLite DB）もローカル保存される（格納先は環境構築の章を参照）。
- Serena もローカル実行で、インデックスやメモリは `.serena/` 配下にローカル保存される。
- CodeGraph もコードの処理は100%ローカルで、インデックスはリポジトリ直下 `.codegraph/` に保存される。  
ただし匿名の利用統計（使用コマンド種別・言語種別等）の送信がデフォルト有効である点が他2つと異なる。  
コード・パス・シンボル名は含まれないと TELEMETRY.md に明記されているが、無効化して利用する（環境構築の章を参照）。

コード以外の外部通信として以下がある。

| 通信 | codebase-memory-mcp | CodeGraph | Serena |
| --- | --- | --- | --- |
| 初回インストール時 | GitHub Releases からバイナリ取得 | npm / GitHub からパッケージ取得 | uvx が GitHub からソース取得・依存パッケージ取得 |
| 起動時 | 新バージョンの有無チェック（自動アップデート機構） | なし（`upgrade` は手動実行） | 言語サーバ（clangd 等）が未導入の場合の自動ダウンロード |
| 匿名利用統計 | なし | あり（既定ON、`telemetry off` で無効化） | なし |
| 解析対象コード | 送信されない | 送信されない | 送信されない |

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

### CodeGraph

tree-sitter でコードを解析し、リポジトリ直下 `.codegraph/` の SQLite に知識グラフとして永続化する MCP サーバ。  
TypeScript 実装で、Node ランタイム同梱のバンドル版または npm で導入する。  
MCP としては既定で `codegraph_explore` の1ツールのみを公開する設計。  
関連シンボルのソース・コールパス・影響範囲（blast radius）を1回の呼び出しで返す。

#### 強いところ

- インデックスが既定でリポジトリ直下 `.codegraph/` に置かれ、存在が見える（ワークスペース配置が標準）。
- 鮮度維持が3ツール中で最も強い。OSのファイルイベントで保存の都度自動同期（デバウンス既定2秒）する。  
未同期ファイルを参照する応答には警告バナーが付き、MCP接続時にも作業ツリーとの差分照合が走る。
- 公開ツールが1つだけなので、ツール定義によるコンテキスト消費が最小。
- C は `.c`/`.h` フル対応。実リポジトリ（redis）でのクロスファイル解決率 92.2% の実測値を公開している。

#### 弱いところ

- tree-sitter でプリプロセス前を見る点は codebase-memory-mcp と同じ。  
マクロエイリアスや `#if` による可視性の判定（検証項目2・3）は同様に苦手な可能性が高い。
- 匿名利用統計の送信がデフォルト有効（無効化は可能。データ送信の整理を参照）。
- npm / Node 系の配布であり、単一静的バイナリの codebase-memory-mcp より供給元の確認事項が多い。

### 役割の違い

| 観点 | codebase-memory-mcp | CodeGraph | Serena |
| --- | --- | --- | --- |
| 解析基盤 | tree-sitter AST + Hybrid LSP（プリプロセス前） | tree-sitter（プリプロセス前） | clangd（LSP、プリプロセス後） |
| C解析の前提条件 | なし | なし | compile_commands.json 必須 |
| ビュー | 全ソースの網羅ビュー | 全ソースの網羅ビュー | 単一ビルド構成の正確なビュー |
| インデックス格納先 | 環境変数で指定（既定はユーザー配下） | リポジトリ直下 `.codegraph/`（既定） | `.serena/` 配下 |
| 公開ツール数 | 14 | 1（explore に集約） | 多数 |
| テレメトリ | なし | 匿名利用統計（既定ON・無効化可） | なし |
| 位置づけ | 構造知識グラフのクエリエンジン | 同左（explore 特化） | エージェントが操作するIDE |

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
    "codegraph": {
      "type": "stdio",
      "command": "codegraph",
      "args": ["serve", "--mcp"]
    },
    "serena": {
      "type": "stdio",
      "command": "uvx",
      "args": [
        "--from", "git+https://github.com/oraios/serena@2449313",
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

CodeGraph を使う場合は、雛形中のグラフツール名を `codegraph_explore` に読み替える。

### まとめ

| 質問の種類 | 使うべきツール | 理由 |
| --- | --- | --- |
| 呼び出し元・依存・影響範囲 | グラフ/LSP（MCP） | 1クエリで完結、トークン最小 |
| 文字列・コメント・設定値検索 | Grep | グラフに載っていない情報 |
| ファイル内容の精読 | Read | 構造化不要 |
| MCP の結果が0件・鮮度不明 | Grep で裏取り | 幻覚・古いインデックスの検出 |

---

### Serena のセットアップ

#### uv の導入

以下で導入する。  
導入済みなら不要。  

```powershell
winget install --source winget --id astral-sh.uv -e
```

以下でバージョン確認できればOK

```powershell
uv -V
```

#### clangd の導入

以下で導入する。  
導入済みなら不要。  

```powershell
winget install --source winget --id LLVM.LLVM -e
```

以下パスをシステム環境変数PATHへ追加しておく。  

> C:\Program Files\LLVM\bin\clangd --version.exe

VSCode・ターミナルなど再起動後に以下通ればOK

```powershell
clangd --version
```

#### compile_commands.json の生成

このファイルを用意できないと話にならない。

FreeRTOS-Kernel は単体ではライブラリであり、`FreeRTOSConfig.h` とポート選択がないとビルドが成立しない。  
最小構成として POSIX ポート（または MSVC/MinGW ポート）でコンパイルDBだけ作る。  
リポジトリ同梱の `examples/template_configuration/FreeRTOSConfig.h` を利用する。

WSL2/Linux 側での例:

```bash
# リポジトリルートに検証用の最小 CMakeLists.txt を用意
cat > CMakeLists_bench.txt << 'EOF'
cmake_minimum_required(VERSION 3.15)
project(freertos_bench C)
set(FREERTOS_PORT GCC_POSIX CACHE STRING "")
add_library(freertos_config INTERFACE)
target_include_directories(freertos_config INTERFACE
    ${CMAKE_CURRENT_LIST_DIR}/examples/template_configuration)
add_subdirectory(. FreeRTOS-Kernel)
EOF

cmake -S . -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON \
      -DFREERTOS_PORT=GCC_POSIX \
      -C /dev/null -DCMAKE_PROJECT_INCLUDE=CMakeLists_bench.txt 2>/dev/null \
  || cmake -S . -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON -DFREERTOS_PORT=GCC_POSIX

# clangd はルート直下の compile_commands.json を探す
cp build/compile_commands.json .
```

> 上記がリポジトリの CMake 構成の変更で通らない場合の代替手段がある。  
> `FreeRTOS/FreeRTOS`（デモ付きリポジトリ）の Posix デモをビルドして compile_commands.json を得る方法が確実である。  
> どの手段で作ったかを必ず記録に残すこと（Serena の見えるビュー＝この構成、が検証結果の解釈に効くため）。

#### `.vscode/mcp.json` への登録

前出の設定例の serena エントリを使う。  
uvx は指定がないと実行のたびに main の最新を取りに行くため、`@コミットSHA` で必ずピン止めする。  
`--project ${workspaceFolder}` により、対象プロジェクトは開いているワークスペースに固定される。

#### プロジェクト設定  

初回起動で生成される `.serena/project.yml` を編集する。

```yaml
languages:
  - cpp        # C は cpp 言語キー（clangd）で扱う
read_only: true  # 生成AIによる実装変更させないため編集ツールを無効化
```

#### インデックスとオンボーディング

```powershell
uvx --from "git+https://github.com/oraios/serena@2449313" serena project index
```

Copilot Chat（Agent モード）での初回セッションではオンボーディング（プロジェクト理解メモの生成）が走る。  

### CodeGraph のセットアップ

方針:  
エージェント自動設定コマンド（`codegraph install`）は使わない（`--skip-config` と同方針）。  
なお `codegraph install` の自動構成対象に VS Code / Copilot は含まれておらず、いずれにせよ手動登録となる。  
使うのは CLI 導入・テレメトリ無効化・プロジェクト初期化・`.vscode/mcp.json` への手動登録の4つだけである。

1. CLI の導入（バージョン固定）。  
公式のワンライナーインストーラ（`irm ... | iex`）は使わず、npm でバージョンを固定して導入する。

   ```powershell
   npm i -g @colbymchenry/codegraph@<バージョン>
   codegraph version   # 起動確認
   ```

   > Node ランタイム同梱設計のためネイティブビルドは不要とされているが、`ignore-scripts=true` 環境で  
   > 起動に失敗する場合は、公式インストーラスクリプトを一読のうえ実行する方式に切り替える。

2. テレメトリの無効化。  
匿名利用統計の送信がデフォルト有効のため、当社方針に合わせて無効化する。

   ```powershell
   codegraph telemetry off   # 環境変数 CODEGRAPH_TELEMETRY=0 でも可
   codegraph telemetry       # off になっていることを確認
   ```

3. プロジェクト初期化（インデックス作成）。

   ```powershell
   cd C:\work\FreeRTOS-Kernel
   codegraph init      # .codegraph/ の作成とグラフ構築を一括で行う
   codegraph status    # シンボル数等を記録
   ```

   インデックスは既定でリポジトリ直下の `.codegraph/`（SQLite）に作られる。  
   ワークスペース内配置が標準動作のため、`CBM_CACHE_DIR` のような対策は不要である。  
   `.gitignore` に `.codegraph/` を追記する。

4. `.vscode/mcp.json` への登録。  
前掲の設定例の codegraph エントリを使う（`codegraph serve --mcp` を stdio で起動するだけ）。

5. 鮮度の考え方。  
CodeGraph は OS のファイルイベントで保存の都度自動同期し、未同期ファイルへの応答には警告バナーが付く。  
MCP 接続時にも作業ツリーとの差分照合が走るため、手動での再インデックスは原則不要である。  
比較検証では他ツールと条件を揃えるため、各クエリ実施前に `codegraph status` を実行し、  
`Pending sync:` が表示されないこと（＝未同期ファイルなし）を確認してから開始する。

6. 測定上の注意。  
MCP として公開されるツールは既定で `codegraph_explore` の1つだけである。  
このため採点表の「ツール呼び出し回数」は構造的に少なく出る。  
他2ツールとの回数比較は優劣ではなく設計思想の差として解釈し、備考欄にその旨を記録する。  
（`CODEGRAPH_MCP_TOOLS` 環境変数で callers / impact 等の個別ツールを公開することもできる。）

### 起動と動作確認

1. `.vscode/mcp.json` を保存すると、ファイル内のサーバ一覧上部に Start ボタンが表示されるのでクリックして起動する。  
初回はサーバを信頼するかの確認ダイアログが出る。
2. Copilot Chat を開き、モードを Agent に切り替える。
3. チャット入力欄のツールアイコンを開き、サーバとツールの一覧が見えることを確認する。  
ツール一覧では使う予定のサーバ/ツールだけ有効化する。  
同時有効化できるツール数には上限があり、有効ツールが多いほどツール定義分のコンテキストを毎ターン消費するためである。
4. 初回のツール呼び出し時は実行許可の確認ダイアログが出る。  
許可のスコープ（今回のみ / セッション / ワークスペース）を選べるので、検証時はセッション程度に留めるのが無難である。
5. 動作確認の目安は次のとおり。  
codebase-memory-mcp は `list_projects` でノード数・エッジ数・`indexed_at` が返ること。  
CodeGraph は `codegraph_explore` に `xTaskCreate` を尋ね、tasks.c のソースと呼び出し元が返ること。  
Serena は `find_symbol` で `xTaskCreate` の定義が `tasks.c` に解決されること。  
Serena でクロスファイル参照が0件しか返らない場合は、clangd が compile_commands.json を読めていないサインである。
6. トラブル時はコマンドパレット → `MCP: List Servers` → 対象サーバ → Show Output でサーバログを確認する。  
Serena はダッシュボード（<http://localhost:24282/dashboard/>）でも状態を確認できる。
