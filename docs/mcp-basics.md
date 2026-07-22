---
html:
  embed_local_images: true
  embed_svg: true
  offline: true
  toc: true
export_on_save:
  html: true
---

# MCP 導入の基本（Serena / codebase-memory-mcp / CodeGraph 共通）

- 検証スコープ: read-only の構造解釈性能のみ（コード編集機能は対象外）
- 想定エージェント: Claude Code および GitHub Copilot Chat（VS Code、Agent モード）
- サンプルリポジトリ: [FreeRTOS-Kernel](https://github.com/FreeRTOS/FreeRTOS-Kernel)  
本書のパス・コマンド例は、このリポジトリを `C:\work\FreeRTOS-Kernel` に取得した前提で記載する。  
別のリポジトリに適用する場合は、パスと clangd 用ビルド構成（serena\.md の手順）を読み替える。

## ドキュメント構成

本書群は役割ごとに以下へ分割している。

| ファイル | 内容 |
| --- | --- |
| mcp-basics\.md（本書） | MCP の概要、データ送信の整理、3ツール共通の前提・設定・有効化手順 |
| [codebase-memory-mcp.md](codebase-memory-mcp.md) | codebase-memory-mcp の紹介とセットアップ |
| [serena.md](serena.md) | Serena の紹介とセットアップ |
| [codegraph.md](codegraph.md) | CodeGraph の紹介とセットアップ |
| [比較検証.md](比較検証.md) | FreeRTOS を用いた機能比較の実施手順（検証クエリ・正解データの作成手順・判定の目安） |
| [answers/](answers/) | Q1〜Q6 の正解データ（コミット `9b777ae5` 時点で作成済み） |
| [results/](results/) | 採点シート（試行記録・エージェント別集計サマリ） |

---

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
**コードが外部（LLM API）へ出るのは、Coding Agent が通常の推論リクエストとして送信する経路のみである。**

- MCP サーバはローカルプロセスとして動作し、通信相手は MCP クライアント（エージェント）だけである。
- ツールの実行結果（関数一覧、呼び出しグラフ等）は、エージェントがコンテキストに含めた時点で初めて LLM API へ送られる。  
これは MCP を使わずエージェントがファイルを直接 Read する場合と同じ経路であり、MCP を挟むことで新しい送信経路が増えるわけではない。
- codebase-memory-mcp は README で「100%ローカル動作・テレメトリ収集なし」を明言している。  
インデックス（SQLite DB）もローカル保存される（格納先は codebase-memory-mcp\.md を参照）。
- Serena もローカル実行で、インデックスやメモリは `.serena/` 配下にローカル保存される。
- CodeGraph もコードの処理は100%ローカルで、インデックスはリポジトリ直下 `.codegraph/` に保存される。  
ただし匿名の利用統計（使用コマンド種別・言語種別等）の送信がデフォルト有効である点が他2つと異なる。  
コード・パス・シンボル名は含まれないと TELEMETRY\.md に明記されているが、無効化して利用する（codegraph\.md を参照）。

コード以外の外部通信として以下がある。

| 通信 | codebase-memory-mcp | CodeGraph | Serena |
| --- | --- | --- | --- |
| 初回インストール時 | GitHub Releases からバイナリ取得 | npm / GitHub からパッケージ取得 | uvx が GitHub からソース取得・依存パッケージ取得 |
| 起動時 | 新バージョンの有無チェック（自動アップデート機構） | なし（`upgrade` は手動実行） | 言語サーバ（clangd 等）が未導入の場合の自動ダウンロード |
| 匿名利用統計 | なし | あり（既定ON、`telemetry off` で無効化） | なし |
| 解析対象コード | 送信されない | 送信されない | 送信されない |

> 導入時はバージョン（コミットSHA / リリースタグ）を固定し、設定は `.vscode/mcp.json` でワークスペース単位に閉じる。

## 3ツールの位置づけ（役割の違い）

各ツールの詳しい紹介（強み・弱み）はそれぞれのファイルに記載する。ここでは俯瞰のみ示す。

| 観点 | codebase-memory-mcp | CodeGraph | Serena |
| --- | --- | --- | --- |
| 解析基盤 | tree-sitter AST + Hybrid LSP（プリプロセス前） | tree-sitter（プリプロセス前） | clangd（LSP、プリプロセス後） |
| C解析の前提条件 | なし | なし | compile_commands.json 必須 |
| ビュー | 全ソースの網羅ビュー | 全ソースの網羅ビュー | 単一ビルド構成の正確なビュー |
| インデックス格納先 | 環境変数で指定（既定はユーザー配下） | リポジトリ直下 `.codegraph/`（既定） | `.serena/` 配下 |
| 公開ツール数 | 14 | 1（explore に集約） | 多数 |
| テレメトリ | なし | 匿名利用統計（既定ON・無効化可） | なし |
| 位置づけ | 構造知識グラフのクエリエンジン | 同左（explore 特化） | エージェントが操作するIDE |

## 共通準備（サンプルリポジトリの取得）

比較の再現性のため、タグを固定して取得する。  
全ツールに同一コミットをインデックスさせることが比較の大前提となる。

```powershell
cd C:\work
git clone --branch V11.3.0 --depth 1 https://github.com/FreeRTOS/FreeRTOS-Kernel.git
cd FreeRTOS-Kernel
git rev-parse HEAD   # コミットSHAを記録に残しておく
```

## 前提条件

### GitHub Copilot（VS Code）の場合

1. 組織ポリシーの有効化。  
Copilot Business では「MCP servers in Copilot」ポリシーがデフォルト無効であり、組織管理者が有効化しない限り MCP を使えない。  
技術的な設定より先に詰まりやすいポイントなので、最初に確認する。
2. Agent モード限定。  
MCP ツールは Copilot Chat の Agent モードでのみ使用でき、Ask / Edit モードからは見えない。
3. VS Code は MCP 対応版（1.99 以降、推奨は最新安定版）を使う。

### Claude Code の場合

1. Claude Code（CLI または VS Code 拡張）が導入済みであること。MCP は標準サポートである。
2. リポジトリ直下の `.mcp.json`（プロジェクトスコープ）に定義したサーバは、初回に使用許可の確認が出る。  
承認状態をやり直したい場合は `claude mcp reset-project-choices` を実行する。

## 設定ファイルの基本

設定ファイルはクライアントごとに異なるが、方針は共通である。

- ワークスペース（リポジトリ）単位の設定ファイルに書き、グローバル登録は使わない。  
グローバル登録すると全プロジェクトの全セッションにツール定義が載り、無関係な作業でもコンテキストを消費するためである。
- サーバのバージョン（コミットSHA / リリースタグ）を固定する。
- DB 実体（`.cbm-cache/` 等）を gitignore しておけば、設定ファイル自体はコミットしてチーム共有できる。

| クライアント | 設定ファイル | ルートキー |
| --- | --- | --- |
| GitHub Copilot（VS Code） | `.vscode/mcp.json` | `servers` |
| Claude Code | リポジトリ直下の `.mcp.json` | `mcpServers` |

**ルートキーはクライアントごとに異なる（VS Code は `servers`、Claude Code は `mcpServers`）。**  
他クライアント向けの設定例をコピーしてキーを直し忘れるのが定番のミスなので注意する。

### GitHub Copilot（VS Code）: `.vscode/mcp.json`

ユーザープロファイル側（`MCP: Open User Configuration`）はグローバル登録になるため使わない。  
`${workspaceFolder}` などの定義済み変数が使えるため、パスをハードコードせずに書ける。

3サーバをすべて設定した例を示す。各エントリの意味・前提はそれぞれのファイルを参照。

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

### Claude Code: `.mcp.json`

リポジトリ直下の `.mcp.json`（プロジェクトスコープ）に書く。  
ユーザースコープ（`claude mcp add --scope user`）はグローバル登録になるため使わない。  
`${workspaceFolder}` は VS Code の変数であり Claude Code では使えないため、パスは実パスで記載する（環境変数の `${VAR}` 展開には対応している）。

```json
{
  "mcpServers": {
    "codebaseMemory": {
      "type": "stdio",
      "command": "codebase-memory-mcp",
      "env": {
        "CBM_CACHE_DIR": "C:\\work\\FreeRTOS-Kernel\\.cbm-cache"
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
        "--project", "C:\\work\\FreeRTOS-Kernel"
      ]
    }
  }
}
```

### 検証時の切り替え

比較検証中は対象の1サーバだけを有効化して実施する（[比較検証.md](比較検証.md) 参照）。

- `.vscode/mcp.json` はコメントが書けるため、使わないサーバをコメントアウトすればよい。
- `.mcp.json` は素の JSON でコメントが書けないため、使わないサーバのエントリごと外す。

## 起動と動作確認（共通手順）

### GitHub Copilot（VS Code）

1. `.vscode/mcp.json` を保存すると、ファイル内のサーバ一覧上部に Start ボタンが表示されるのでクリックして起動する。  
初回はサーバを信頼するかの確認ダイアログが出る。
2. Copilot Chat を開き、モードを Agent に切り替える。
3. チャット入力欄のツールアイコンを開き、サーバとツールの一覧が見えることを確認する。  
ツール一覧では使う予定のサーバ/ツールだけ有効化する。  
同時有効化できるツール数には上限があり、有効ツールが多いほどツール定義分のコンテキストを毎ターン消費するためである。
4. 初回のツール呼び出し時は実行許可の確認ダイアログが出る。  
許可のスコープ（今回のみ / セッション / ワークスペース）を選べるので、検証時はセッション程度に留めるのが無難である。
5. トラブル時はコマンドパレット → `MCP: List Servers` → 対象サーバ → Show Output でサーバログを確認する。

### Claude Code

1. リポジトリ直下で Claude Code を起動すると `.mcp.json` が検出され、初回はプロジェクトスコープのサーバを使用してよいかの承認確認が出る。
2. `/mcp` でサーバの接続状態とツール一覧を確認できる。接続に失敗している場合もここにエラーとして表示される。
3. 初回のツール呼び出し時に実行許可の確認が出る。検証時は都度確認か、セッション単位の許可に留めるのが無難である。

### 動作確認の目安

各サーバの動作確認の目安は、それぞれのファイルの「動作確認」を参照する。

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

### 使い分けの指針（copilot-instructions\.md に書く内容の雛形）

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

> codebase-memory-mcp を `--skip-config` を使わず標準インストールした場合は、検出したエージェントに対して同趣旨の指示ファイルやスキル、Grep 時にグラフ検索結果を添える非ブロッキングのフックが自動で入る。  
> `--skip-config` 運用では、この雛形を copilot-instructions\.md に書くことで代替する（codebase-memory-mcp\.md 参照）。

CodeGraph を使う場合は、雛形中のグラフツール名を `codegraph_explore` に読み替える。  
Claude Code で運用する場合は、この雛形を copilot-instructions\.md ではなくプロジェクトの `CLAUDE.md` に書く。

### まとめ

| 質問の種類 | 使うべきツール | 理由 |
| --- | --- | --- |
| 呼び出し元・依存・影響範囲 | グラフ/LSP（MCP） | 1クエリで完結、トークン最小 |
| 文字列・コメント・設定値検索 | Grep | グラフに載っていない情報 |
| ファイル内容の精読 | Read | 構造化不要 |
| MCP の結果が0件・鮮度不明 | Grep で裏取り | 幻覚・古いインデックスの検出 |
