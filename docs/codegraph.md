---
html:
  embed_local_images: true
  embed_svg: true
  offline: true
  toc: true
export_on_save:
  html: true
---

# CodeGraph セットアップ手順

MCP の概要・共通の前提条件・`.vscode/mcp.json` の基本は [mcp-basics.md](mcp-basics.md) を参照。

## ツール紹介

tree-sitter でコードを解析し、リポジトリ直下 `.codegraph/` の SQLite に知識グラフとして永続化する MCP サーバ。  
TypeScript 実装で、Node ランタイム同梱のバンドル版または npm で導入する。  
MCP としては既定で `codegraph_explore` の1ツールのみを公開する設計。  
関連シンボルのソース・コールパス・影響範囲（blast radius）を1回の呼び出しで返す。

### 強いところ

- インデックスが既定でリポジトリ直下 `.codegraph/` に置かれ、存在が見える（ワークスペース配置が標準）。
- 鮮度維持が3ツール中で最も強い。OSのファイルイベントで保存の都度自動同期（デバウンス既定2秒）する。  
未同期ファイルを参照する応答には警告バナーが付き、MCP接続時にも作業ツリーとの差分照合が走る。
- 公開ツールが1つだけなので、ツール定義によるコンテキスト消費が最小。
- C は `.c`/`.h` フル対応。実リポジトリ（redis）でのクロスファイル解決率 92.2% の実測値を公開している。

### 弱いところ

- tree-sitter でプリプロセス前を見る点は codebase-memory-mcp と同じ。  
マクロエイリアスや `#if` による可視性の判定（検証項目2・3）は同様に苦手な可能性が高い。
- 匿名利用統計の送信がデフォルト有効（無効化は可能。mcp-basics.md の「データ送信に関する整理」を参照）。
- npm / Node 系の配布であり、単一静的バイナリの codebase-memory-mcp より供給元の確認事項が多い。

## セットアップ

方針:  
エージェント自動設定コマンド（`codegraph install`）は使わない（codebase-memory-mcp の `--skip-config` と同方針）。  
なお `codegraph install` の自動構成対象に VS Code / Copilot は含まれておらず、いずれにせよ手動登録となる。  
使うのは CLI 導入・テレメトリ無効化・プロジェクト初期化・`.vscode/mcp.json` への手動登録の4つだけである。

### CLI の導入（バージョン固定）

公式のワンライナーインストーラ（`irm ... | iex`）は使わず、npm でバージョンを固定して導入する。

```powershell
npm i -g @colbymchenry/codegraph@<バージョン>
codegraph version   # 起動確認
```

> Node ランタイム同梱設計のためネイティブビルドは不要とされているが、`ignore-scripts=true` 環境で  
> 起動に失敗する場合は、公式インストーラスクリプトを一読のうえ実行する方式に切り替える。

### テレメトリの無効化

匿名利用統計の送信がデフォルト有効のため、当社方針に合わせて無効化する。

```powershell
codegraph telemetry off   # 環境変数 CODEGRAPH_TELEMETRY=0 でも可
codegraph telemetry       # off になっていることを確認
```

### プロジェクト初期化（インデックス作成）

```powershell
cd C:\work\FreeRTOS-Kernel
codegraph init      # .codegraph/ の作成とグラフ構築を一括で行う
codegraph status    # シンボル数等を記録
```

インデックスは既定でリポジトリ直下の `.codegraph/`（SQLite）に作られる。  
ワークスペース内配置が標準動作のため、`CBM_CACHE_DIR` のような対策は不要である。  
`.gitignore` に `.codegraph/` を追記する。

### MCP クライアントへの登録

GitHub Copilot（VS Code）の場合は `.vscode/mcp.json` に以下のエントリを追加する（ファイル全体の書き方は mcp-basics.md 参照）。  
`codegraph serve --mcp` を stdio で起動するだけである。

```json
{
  "servers": {
    "codegraph": {
      "type": "stdio",
      "command": "codegraph",
      "args": ["serve", "--mcp"]
    }
  }
}
```

Claude Code の場合はリポジトリ直下 `.mcp.json` の `mcpServers` に同じ内容を登録する（設定例は mcp-basics.md）。

### 鮮度の考え方

CodeGraph は OS のファイルイベントで保存の都度自動同期し、未同期ファイルへの応答には警告バナーが付く。  
MCP 接続時にも作業ツリーとの差分照合が走るため、手動での再インデックスは原則不要である。  
比較検証では他ツールと条件を揃えるため、各クエリ実施前に `codegraph status` を実行し、  
`Pending sync:` が表示されないこと（＝未同期ファイルなし）を確認してから開始する。

### 測定上の注意

MCP として公開されるツールは既定で `codegraph_explore` の1つだけである。  
このため採点表の「ツール呼び出し回数」は構造的に少なく出る。  
他2ツールとの回数比較は優劣ではなく設計思想の差として解釈し、備考欄にその旨を記録する。  
（`CODEGRAPH_MCP_TOOLS` 環境変数で callers / impact 等の個別ツールを公開することもできる。）

## 動作確認

- 起動〜ツール有効化の共通手順は mcp-basics.md の「起動と動作確認」を参照。
- 動作確認の目安: `codegraph_explore` に `xTaskCreate` を尋ね、tasks.c のソースと呼び出し元が返ること。
