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

- 対象リポジトリ: [FreeRTOS-Kernel](https://github.com/FreeRTOS/FreeRTOS-Kernel)
- 検証スコープ: read-only の構造解釈性能のみ（コード編集機能は対象外）
- 想定エージェント: Claude Code（両ツールを同一エージェント経由で比較する）

---

# 第1部 一般的な紹介

## そもそも MCP サーバとは何か

MCP（Model Context Protocol）は、AI コーディングエージェント（Claude Code、GitHub Copilot、Cursor など）と外部ツールを接続するためのオープンな標準プロトコルです。

仕組みは単純で、以下の3者で構成されます。

```text
[LLM (クラウド)] ←→ [MCPクライアント = Coding Agent] ←→ [MCPサーバ (ローカルプロセス)]
                         (Claude Code 等)                  (Serena / codebase-memory-mcp)
```

- MCPサーバは「ツール」（例: `find_symbol`、`trace_call_path`）を提供するだけのプログラム。  
多くは stdio 上の JSON-RPC で動くローカルプロセスであり、それ自体に AI は入っていない。
- MCPクライアント（Coding Agent）が、ユーザーの自然言語の質問を解釈し、「どのツールをどの引数で呼ぶか」を決めて MCP サーバを呼び出す。
- ツールの実行結果はエージェントに返り、エージェントがそれを LLM のコンテキストに載せて回答を生成する。

今回比較する2つのツールはどちらも「エージェントに構造化されたコード情報を安く・正確に供給する索引サービス」であり、回答の言語化はエージェント側の LLM が担います。

## データ送信に関する整理

**ここで紹介する2つのMCPサーバはいずれも、自発的に外部へコードを送信することはありません。**  
**コードが社外（LLM API）へ出るのは、Coding Agent が通常の推論リクエストとして送信する経路のみです。**

- MCP サーバはローカルプロセスとして動作し、通信相手は MCP クライアント（エージェント）だけです。
- ツールの実行結果（関数一覧、呼び出しグラフ等）は、エージェントがコンテキストに含めた時点で初めて LLM API へ送られます。
これは MCP を使わずエージェントがファイルを直接 Read する場合と同じ経路であり、**MCP を挟むことで新しい送信経路が増えるわけではありません**。
- codebase-memory-mcp は README で「100%ローカル動作・テレメトリ収集なし」を明言しています。  
インデックス（SQLite DB）も `~/.cache/codebase-memory-mcp/` にローカル保存されます。
- Serena もローカル実行で、インデックスやメモリは `.serena/` 配下にローカル保存されます。

コード以外の外部通信として以下があります。

| 通信 | codebase-memory-mcp | Serena |
| --- | --- | --- |
| 初回インストール時 | GitHub Releases からバイナリ取得 | uvx が GitHub からソース取得・依存パッケージ取得 |
| 起動時 | 新バージョンの有無チェック（自動アップデート機構） | 言語サーバ（clangd 等）が未導入の場合の自動ダウンロード |
| 解析対象コード | 送信されない | 送信されない |

> 導入時はバージョン（コミットSHA / リリースタグ）を固定し、`.mcp.json` はプロジェクト単位で管理する。

## ツール紹介

### codebase-memory-mcp

tree-sitter による AST パースでコードベースを永続的な知識グラフ（SQLite）にインデックスする MCP サーバ。  
C を含む主要言語では Hybrid LSP と呼ばれる型解決を内蔵します。  
`search_graph` / `trace_call_path` / `query_graph`（Cypher）/ dead code 検出などのツールを提供します。  
単一の静的バイナリで、ランタイム依存ゼロ。

#### 強いところ

- セットアップがほぼゼロ。バイナリを置いて `.mcp.json` に1エントリ書くだけ。
- インデックスが高速・軽量で、クエリはミリ秒未満。トークン消費が file-by-file 探索より桁違いに少ない
- ビルド構成に依存しない「全ソースの網羅ビュー」。portable/ 以下の全ポートが見える
- read-only 設計なのでコードを壊すリスクが構造的にない

#### 弱いところ

- tree-sitter はプリプロセス前のソースを見るため、マクロで包装された API のエイリアス解決や、`#if` による可視性の判定は原理的に苦手な可能性が高い（今回の検証項目2・3）
- 関数ポインタ経由の間接呼び出しの解決には限界がある（これは Serena/clangd も同様）
- 編集機能はなく、構造クエリ専用

### Serena

LSP（Language Server Protocol）を土台にした「コーディングエージェント用ツールキット」。  
C/C++ では clangd を言語サーバとして使い、シンボルの定義・参照をセマンティックに解決します。  
`find_symbol` / `find_referencing_symbols` / `get_symbols_overview` などのツールを提供します。  

#### 強いところ

- clangd はプリプロセス後のコードを解釈するため、マクロ展開・typedef 連鎖・条件コンパイルを正しく追える（組込みCでは決定的な差になりうる）
- IDE の「定義へ移動」「参照検索」と同等の精度が期待できる
- 本検証の対象外だが、シンボル編集（リネーム・参照一括更新）も可能

#### 弱いところ

- C の精度は  `compile_commands.json` の有無と品質に完全に依存する。無いと参照解決が大きく劣化する
- Python + uv + clangd と依存が多く、セットアップコストが高い
- clangd は「1つのビルド構成」しか見ない。FreeRTOS のような多ポート構成では、compile_commands.json に含まれないポートは不可視になる
- 大規模リポジトリでは初回インデックスと言語サーバの起動が重い

### 役割の違い

| 観点            | codebase-memory-mcp                            | Serena                        |
| --------------- | ---------------------------------------------- | ----------------------------- |
| 解析基盤        | tree-sitter AST + Hybrid LSP（プリプロセス前） | clangd（LSP、プリプロセス後） |
| C解析の前提条件 | なし                                           | compile_commands.json 必須    |
| ビュー          | 全ソースの網羅ビュー                           | 単一ビルド構成の正確なビュー  |
| セットアップ    | ほぼゼロ                                       | 重い                          |
| 位置づけ        | 構造知識グラフのクエリエンジン                 | エージェントが操作するIDE     |

## 環境構築手順

### codebase-memory-mcp のセットアップ（`--skip-config` 前提）

方針:  
インストーラの自動設定（エージェント設定・スキル・フックの書き込み）は使わない。  
`--skip-config` でバイナリ配置のみ行い、`.mcp.json` は手書きで管理する。  
エージェント設定を勝手に書き換えられないので、何がどこに入ったかを把握できる。  
トレードオフとして、本来 `install` が仕込む指示ファイルと PreToolUse フック（Grep 時にグラフ検索結果を非ブロッキングで添える補助）が入らないため、実運用時のグラフツールへの誘導は copilot-instructions.md に自分で書く必要がある（→ 付録A）。

1. バイナリ入手と検証
   Releases から Windows 用 zip（`codebase-memory-mcp-windows-amd64.zip`）を取得し、`checksums.txt` の SHA-256 と Attestation を検証してから、同梱インストーラを `--skip-config` で実行する。

   ```powershell
   gh attestation verify .\codebase-memory-mcp.exe --repo DeusData/codebase-memory-mcp

   Expand-Archive codebase-memory-mcp-windows-amd64.zip -DestinationPath .
   # 一読してから実行（当社ルール）
   .\install.ps1 --skip-config   # バイナリ配置のみ。エージェント設定は書き換えない
   ```

2. **DB（インデックス）をワークスペース内に配置する**
   デフォルトでは SQLite DB が `~/.cache/codebase-memory-mcp/`（Windows ではユーザープロファイル配下）に作られ、リポジトリからは存在が見えません。環境変数 **`CBM_CACHE_DIR`** で格納先を上書きできるので、`.mcp.json` の `env` でリポジトリ内の `.cbm-cache/` を指定します。

   `.mcp.json`（リポジトリルートに作成）:

   ```json
   {
     "mcpServers": {
       "codebase-memory-mcp": {
         "type": "stdio",
         "command": "C:\\tools\\codebase-memory-mcp\\codebase-memory-mcp.exe",
         "env": {
           "CBM_CACHE_DIR": "C:\\work\\FreeRTOS-Kernel\\.cbm-cache"
         }
       }
     }
   }
   ```

   > パスは**絶対パスで書く**こと（相対パスは MCP サーバプロセスの起動ディレクトリ依存になり、意図しない場所に DB が生えるリスクがあります）。プロジェクトごとに `.mcp.json` を持つ運用なのでハードコードで問題ありません。

   `.gitignore` に追記します（インデックスは当面コミットしない方針）:

   ```gitignore
   .cbm-cache/
   .codebase-memory/
   ```

   > `.codebase-memory/` は Team-Shared Graph Artifact 用のディレクトリで、インデックス実行時に圧縮スナップショット `graph.db.zst` がリポジトリ直下に書かれることがあります。将来チームでインデックスを共有したくなったら、これを**あえてコミットする**選択肢があります（クローンした側は再インデックス不要になる公式機能）。今回は gitignore しておきます。

3. **Claude Code を再起動し、`/mcp` でツール（14個）が見えることを確認**

4. **インデックス作成**（エージェント経由でも CLI でも可。CLI の方が時間計測しやすい）
   **重要**: CLI から操作する場合も、MCP サーバと同じ DB を見るように**同じ `CBM_CACHE_DIR` をシェル側にも設定**してから実行します。これを忘れると CLI はデフォルトのユーザーディレクトリ側に別の DB を作ってしまい、「CLI ではインデックス済みなのにエージェントからは見えない」という事故になります。

   ```powershell
   $env:CBM_CACHE_DIR = "C:\work\FreeRTOS-Kernel\.cbm-cache"
   codebase-memory-mcp cli index_repository '{"repo_path": "C:\\work\\FreeRTOS-Kernel"}'
   codebase-memory-mcp cli list_projects   # ノード数・エッジ数・indexed_at を記録
   ```

5. **DB の更新忘れに気付ける仕組み**
   ソースを更新したのにインデックスが古いまま、という状態を検出できるようにします。二段構えにします。

   (a) **自動追従を有効にしておく**（第一の防御）。サーバには git ポーリングでファイル変更を検知して増分再インデックスする背景ウォッチャがあり、`auto_watch` はデフォルト有効です。設定を確認します:

   ```powershell
   codebase-memory-mcp config list   # auto_watch: true を確認
   ```

   (b) **鮮度チェックを習慣化・スクリプト化する**（第二の防御。ウォッチャが動いていなかった場合の検出）。`list_projects` の `indexed_at` と git の最終コミット時刻を突き合わせます。リポジトリに `check_index_freshness.ps1` として置いておきます:

   ```powershell
   $env:CBM_CACHE_DIR = "$PSScriptRoot\.cbm-cache"
   $indexedAt = (codebase-memory-mcp cli --raw list_projects | ConvertFrom-Json).projects |
       Where-Object { $_.name -like "*FreeRTOS*" } | Select-Object -ExpandProperty indexed_at
   $lastCommit = git log -1 --format=%cI
   if ([datetime]$lastCommit -gt [datetime]$indexedAt) {
       Write-Warning "インデックスが古い可能性: indexed_at=$indexedAt < last commit=$lastCommit → 再インデックス推奨"
   } else {
       Write-Host "インデックスは最新 (indexed_at=$indexedAt)"
   }
   ```

   > 注意点が2つ。ウォッチャは **git ベースの変更検知**なので、コミットされていない編集への追従タイミングは保証を当てにしないこと。また `--raw` 出力の JSON 構造はバージョンで変わりうるので、スクリプトが動かなくなったら `list_projects` の生出力を確認して調整してください。エージェント側からは `index_status` ツールでも状態確認できます。

   検証（第2部）では、**各クエリ実施前にこの鮮度チェックを通すこと**を手順に含めます。

6. **（実運用向け・比較検証では不要）CLAUDE.md にヒントを追記**
   `--skip-config` では誘導用の指示ファイルが入らないため、実運用時は「構造的な質問にはグラフツールを優先する」旨を CLAUDE.md に自分で書きます（→ 付録A）。今回の比較ではプロンプト側でツール使用を強制するので不要です（第2部参照）。

### 4.2 Serena のセットアップ

1. **uv の導入**

   ```powershell
   winget install astral-sh.uv
   ```

2. **clangd の導入**（LLVM 公式リリース、または `winget install LLVM.LLVM`。`clangd --version` が通ることを確認）

3. **compile_commands.json の生成（最重要）**
   FreeRTOS-Kernel は単体ではライブラリであり、`FreeRTOSConfig.h` とポート選択がないとビルドが成立しません。最小構成として POSIX ポート（または MSVC/MinGW ポート）でコンパイルDBだけ作ります。リポジトリ同梱の `examples/template_configuration/FreeRTOSConfig.h` を利用します。

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

   > 上記がリポジトリの CMake 構成の変更で通らない場合は、`FreeRTOS/FreeRTOS`（デモ付きリポジトリ）の Posix デモをビルドして compile_commands.json を得る方法が確実です。**どの手段で作ったかを記録に残すこと**（Serena の見えるビュー＝この構成、が検証結果の解釈に効くため）。

4. **Claude Code への登録（バージョン固定）**
   リポジトリルートで実行します。`@コミットSHA` または `@タグ` でピン止めします（uvx は指定がないと実行のたびに main の最新を取りに行くため）。

   ```powershell
   claude mcp add serena -s project -- uvx --from "git+https://github.com/oraios/serena@<コミットSHAまたはタグ>" serena start-mcp-server --context ide-assistant --project .
   ```

5. **プロジェクト設定**
   初回起動で生成される `.serena/project.yml` を編集します。

   ```yaml
   languages:
     - cpp        # C は cpp 言語キー（clangd）で扱う
   read_only: true  # 今回は read-only 比較なので編集ツールを無効化
   ```

6. **インデックスとオンボーディング**

   ```powershell
   uvx --from "git+https://github.com/oraios/serena@<同じピン>" serena project index
   ```

   Claude Code 起動後、初回はオンボーディング（プロジェクト理解メモの生成）が走ります。**オンボーディング完了後を計測開始点とする**（初回セッションを計測に含めない）。

7. **動作確認**
   `/mcp` で Serena のツール一覧（`find_symbol` 等）が見えること、ダッシュボード（<http://localhost:24282/dashboard/）でエラーが出ていないこと、`find_symbol`> で `xTaskCreate` の定義が `tasks.c` に解決されることを確認します。クロスファイル参照が0件しか返らない場合は clangd が compile_commands.json を読めていないサインです。

---

### 4.3 GitHub Copilot Chat（VS Code）から利用する場合

比較検証（第2部）は Claude Code で統一して実施します。本節は**実運用・チーム展開用**で、方針は 4.1/4.2 と同じ「グローバル登録せず、ワークスペース単位で閉じる」です。

#### 前提条件（Copilot Business 特有の注意）

1. **組織ポリシーの有効化が必須**。Copilot Business では「MCP servers in Copilot」ポリシーが**デフォルト無効**であり、組織管理者が有効化しない限りメンバーは MCP を使えません。チーム展開の際は最初にここを確認してください（技術的な設定より先に詰まるポイントです）。
2. **Agent モード限定**。MCP ツールは Copilot Chat の Agent モードでのみ使用でき、Ask / Edit モードからは見えません。
3. VS Code のバージョンは MCP 対応版（1.99 以降、推奨は最新安定版）。

#### 設定ファイル（`.vscode/mcp.json`）

ワークスペース単位の設定は、リポジトリ直下の **`.vscode/mcp.json`** に書きます。ユーザープロファイル側（`MCP: Open User Configuration`）に書くとグローバル登録になるため、当社方針では使いません。

**最重要の注意: ルートキーは `servers` です（Claude Code の `.mcp.json` は `mcpServers`）。** Claude Code 用の設定をコピペしてキーを直し忘れるのが定番の設定ミスなので、両ファイルを併置するこの運用では特に注意してください。

```json
{
  "servers": {
    "codebaseMemory": {
      "type": "stdio",
      "command": "C:\\tools\\codebase-memory-mcp\\codebase-memory-mcp.exe",
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

補足:

- VS Code の mcp.json は **`${workspaceFolder}` などの定義済み変数が使えます**。Claude Code の `.mcp.json` では絶対パスをハードコードしましたが、こちらは変数で書けるため、`.vscode/mcp.json` はそのままリポジトリにコミットしてチーム共有できます（DB 実体の `.cbm-cache/` は引き続き gitignore）。
- DB の格納先を `CBM_CACHE_DIR` でワークスペース内に向ける方針・鮮度チェック（4.1 の手順 2/5）は Copilot 利用時も共通です。CLI から操作する際にシェル側へ同じ `CBM_CACHE_DIR` を設定する注意も同様です。
- Serena の `--context ide-assistant` は VS Code 系 IDE アシスタント向けの推奨コンテキストです（Claude Code 用の設定と共通でよい）。
- 比較検証中は 4.1/4.2 と同様、`servers` から一方をコメントアウトして片方ずつ有効化します。

#### 起動と動作確認

1. `.vscode/mcp.json` を保存すると、ファイル内のサーバ一覧上部に **Start** ボタンが表示されるのでクリックして起動します（初回はサーバを信頼するかの確認ダイアログが出ます）。
2. Copilot Chat を開き、モードを **Agent** に切り替えます。
3. チャット入力欄の**ツールアイコン（工具マーク）**を開くと、MCP サーバとツールの一覧が表示されるので、両サーバのツールが見えることを確認します。ツール一覧では**使う予定のサーバ/ツールだけ有効化**してください。同時有効化できるツール数には上限があり、また有効ツールが多いほどツール定義分のコンテキストを毎ターン消費します（4.1 で述べたグローバル配置の問題と同根です）。
4. 初回のツール呼び出し時は実行許可の確認ダイアログが出ます。許可のスコープ（今回のみ / セッション / ワークスペース）を選べるので、検証時は「セッション」程度に留めるのが無難です。
5. トラブル時は コマンドパレット → `MCP: List Servers` → 対象サーバ → **Show Output** でサーバログを確認します。

#### グラフツールへの誘導（CLAUDE.md の Copilot 版）

Copilot には CLAUDE.md の代わりに **`.github/copilot-instructions.md`** を使います。付録A の雛形をそのまま転記すれば、リポジトリ単位で閉じた誘導になります（Claude Code と Copilot を併用するチームなら、両ファイルに同じ内容を置いてください。SkillSync の `daily.md` 二重管理と同じ構図です）。

---

## 付録A: 実運用での素の Grep と MCP ツールの使い分け

比較検証では公平性のため内蔵ツールを封じるが、**実運用で Grep を禁止してはいけない**。  
理由と使い分けの指針を示す。  

### なぜ禁止しないのか

1. 役割が異なる  
グラフ/LSP が得意なのは構造クエリ（呼び出し元・依存・影響範囲・dead code）。  
一方、文字列リテラル・コメント・ログメッセージ・設定キーの検索はテキスト検索の領域であり、グラフには入っていないか不完全。  
codebase-memory-mcp 自身のスキル定義も「テキスト検索は grep/Glob、単一ファイル読み取りは Read を使え」と明記している。  
封じると答えられない質問が生じ、エージェントが幻覚で埋めるリスクが上がる。  
2. トークン消費で高くつくのは Grep そのものではなく「探索ループ」  
構造的な質問を grep → ファイル全体を Read → また grep… と回すパターンが高コスト（構造クエリ5問でグラフ経由 約3,400トークン vs file-by-file 探索 約41.2万トークン、という報告値）。  
パターンが明確な単発 grep はマッチ行しか返らず数百トークンで済み、グラフクエリと大差ない。
3. フォールバック・クロスチェックとしての価値  
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

:::note
`--skip-config` を使わない標準インストールでは、Claude Code 向けに同趣旨の指示ファイル・スキルと、Grep/Glob 実行時にグラフ検索結果を additionalContext として添える非ブロッキング PreToolUse フックが自動で入る。`--skip-config` 運用ではこの雛形を copilot-instructions.md に書くことで代替する。
:::

### まとめ

| 質問の種類 | 使うべきツール | 理由 |
| --- | --- | --- |
| 呼び出し元・依存・影響範囲 | グラフ/LSP（MCP） | 1クエリで完結、トークン最小 |
| 文字列・コメント・設定値検索 | Grep | グラフに載っていない情報 |
| ファイル内容の精読 | Read | 構造化不要 |
| MCP の結果が0件・鮮度不明 | Grep で裏取り | 幻覚・古いインデックスの検出 |
