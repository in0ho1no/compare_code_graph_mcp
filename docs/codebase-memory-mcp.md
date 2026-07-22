---
html:
  embed_local_images: true
  embed_svg: true
  offline: true
  toc: true
export_on_save:
  html: true
---

# codebase-memory-mcp セットアップ手順

MCP の概要・共通の前提条件・`.vscode/mcp.json` の基本は [mcp-basics.md](mcp-basics.md) を参照。

## ツール紹介

tree-sitter による AST パースでコードベースを永続的な知識グラフ（SQLite）にインデックスする MCP サーバ。  
C を含む主要言語では Hybrid LSP と呼ばれる型解決を内蔵する。  
`search_graph` / `trace_call_path` / `query_graph`（Cypher）/ dead code 検出などのツールを提供する。  
単一の静的バイナリで、ランタイム依存ゼロ。

### 強いところ

- セットアップがほぼゼロ。バイナリを置いて `.vscode/mcp.json` に1エントリ書くだけで動く。
- インデックスが高速・軽量で、クエリはミリ秒未満。トークン消費が file-by-file 探索より桁違いに少ない。
- ビルド構成に依存しない「全ソースの網羅ビュー」。portable/ 以下の全ポートが見える。
- read-only 設計なのでコードを壊すリスクが構造的にない。

### 弱いところ

- tree-sitter はプリプロセス前のソースを見るため、マクロで包装された API のエイリアス解決や、`#if` による可視性の判定は原理的に苦手な可能性が高い（検証項目2・3で確認する）。
- 関数ポインタ経由の間接呼び出しの解決には限界がある（これは Serena/clangd も同様）。
- 編集機能はなく、構造クエリ専用である。

## セットアップ（`--skip-config` 前提）

方針:  
インストーラの自動設定（エージェント設定・スキル・フックの書き込み）は使わない。  
`--skip-config` でバイナリ配置のみ行い、`.vscode/mcp.json` は手書きで管理する。  
エージェント設定を勝手に書き換えられないので、何がどこに入ったかを把握できる。  
トレードオフとして、本来 `install` が仕込む指示ファイルとフック（Grep 時にグラフ検索結果を非ブロッキングで添える補助）が入らない。  
そのため、実運用時のグラフツールへの誘導は copilot-instructions.md に自分で書く必要がある（mcp-basics.md の付録A）。

### バイナリ入手と検証

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

インストーラは PATH への追加まで行うため、MCP 設定の `command` はコマンド名だけでよい（後述の設定例）。  
ただしユーザー PATH の変更は既存プロセスに反映されない。  
インストール直後は新しいターミナルで `where.exe codebase-memory-mcp` が通ることを確認し、VS Code は完全に再起動（終了→起動）、Claude Code も新しいターミナルから起動し直す。

バージョンは以下で確認できる。

```powershell
codebase-memory-mcp --version
```

### MCP クライアントへの登録

GitHub Copilot（VS Code）の場合は `.vscode/mcp.json` に以下のエントリを追加する（ファイル全体の書き方は mcp-basics.md 参照）。

```json
{
  "servers": {
    "codebaseMemory": {
      "type": "stdio",
      "command": "codebase-memory-mcp",
      "env": {
        "CBM_CACHE_DIR": "${workspaceFolder}\\.cbm-cache"
      }
    }
  }
}
```

Claude Code の場合はリポジトリ直下 `.mcp.json` の `mcpServers` に同じ内容を登録する。  
ただし `${workspaceFolder}` は使えないため、`CBM_CACHE_DIR` は実パスで指定する（設定例は mcp-basics.md）。

### DB（インデックス）をワークスペース内に配置する

デフォルトでは SQLite DB がユーザープロファイル配下の既定キャッシュディレクトリに作られ、リポジトリから存在が見えない。  
環境変数 `CBM_CACHE_DIR` で格納先を上書きできるので、MCP 設定の `env` で `${workspaceFolder}\.cbm-cache`（Claude Code は実パス）を指定する（前出の設定例）。  
あわせて `.gitignore` に追記する（今回はインデックスをコミットしない方針）。

```gitignore
.cbm-cache/
.codebase-memory/
```

> `.codebase-memory/` は Team-Shared Graph Artifact 用のディレクトリで、インデックス実行時に圧縮スナップショット `graph.db.zst` がリポジトリ直下に書かれることがある。  
> 将来チームでインデックスを共有したくなったら、これをあえてコミットする選択肢がある（クローンした側は再インデックス不要になる公式機能）。今回は gitignore しておく。

### インデックス作成

エージェント経由でも CLI でも実行できるが、無駄にトークン消費しないよう CLI から実行する。  
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

### DB の更新忘れに気付ける仕組み

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

検証（[比較検証.md](比較検証.md)）では、各クエリ実施前にこのチェックを通すことを手順に含める。

### グラフツールへの誘導記述（実運用向け・比較検証では不要）

`--skip-config` では誘導用の指示ファイルが入らないため、実運用時は mcp-basics.md 付録Aの雛形を `.github/copilot-instructions.md`（Claude Code の場合は `CLAUDE.md`）に書く。  
比較検証ではプロンプト側でツール使用を強制するため不要である（比較検証.md 参照）。

## 動作確認

- 起動〜ツール有効化の共通手順は mcp-basics.md の「起動と動作確認」を参照。
- 動作確認の目安: `list_projects` でノード数・エッジ数・`indexed_at` が返ること。

## 参考リンク

- codebase-memory-mcp: <https://github.com/DeusData/codebase-memory-mcp> （論文: arXiv:2603.27277）
