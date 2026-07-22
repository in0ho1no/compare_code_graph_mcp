---
html:
  embed_local_images: true
  embed_svg: true
  offline: true
  toc: true
export_on_save:
  html: true
---

# Serena セットアップ手順

MCP の概要・共通の前提条件・`.vscode/mcp.json` の基本は [mcp-basics.md](mcp-basics.md) を参照。

## ツール紹介

LSP（Language Server Protocol）を土台にした「コーディングエージェント用ツールキット」。  
C/C++ では clangd を言語サーバとして使い、シンボルの定義・参照をセマンティックに解決する。  
`find_symbol` / `find_referencing_symbols` / `get_symbols_overview` などのツールを提供する。

### 強いところ

- clangd はプリプロセス後のコードを解釈するため、マクロ展開・typedef 連鎖・条件コンパイルを正しく追える（組込みCでは決定的な差になりうる）。
- IDE の「定義へ移動」「参照検索」と同等の精度が期待できる。
- 本検証の対象外だが、シンボル編集（リネーム・参照一括更新）も可能である。

### 弱いところ

- C の精度は `compile_commands.json` の有無と品質に完全に依存する。無いと参照解決が大きく劣化する。
- Python + uv + clangd と依存が多く、セットアップコストが高い。
- clangd は「1つのビルド構成」しか見ない。FreeRTOS のような多ポート構成では、compile_commands.json に含まれないポートは不可視になる。
- 大規模リポジトリでは初回インデックスと言語サーバの起動が重い。

## セットアップ

### uv の導入

以下で導入する。  
導入済みなら不要。  

```powershell
winget install --source winget --id astral-sh.uv -e
```

以下でバージョン確認できればOK

```powershell
uv -V
```

### clangd の導入

以下で導入する。  
導入済みなら不要。  

```powershell
winget install --source winget --id LLVM.LLVM -e
```

以下パスをシステム環境変数PATHへ追加しておく。  

> C:\Program Files\LLVM\bin

VSCode・ターミナルなど再起動後に以下通ればOK

```powershell
clangd --version
```

### compile_commands.json の生成

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

### MCP クライアントへの登録

GitHub Copilot（VS Code）の場合は `.vscode/mcp.json` に以下のエントリを追加する（ファイル全体の書き方は mcp-basics.md 参照）。  
uvx は指定がないと実行のたびに main の最新を取りに行くため、`@コミットSHA` で必ずピン止めする。  
`--project ${workspaceFolder}` により、対象プロジェクトは開いているワークスペースに固定される。

```json
{
  "servers": {
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

Claude Code の場合はリポジトリ直下 `.mcp.json` の `mcpServers` に同じ内容を登録する。  
ただし `${workspaceFolder}` は使えないため、`--project` は対象リポジトリの実パスで指定する（設定例は mcp-basics.md）。

### プロジェクト設定

初回起動で生成される `.serena/project.yml` を編集する。

```yaml
languages:
  - cpp        # C は cpp 言語キー（clangd）で扱う
read_only: true  # 生成AIによる実装変更させないため編集ツールを無効化
```

### インデックスとオンボーディング

```powershell
uvx --from "git+https://github.com/oraios/serena@2449313" serena project index
```

エージェント（Claude Code / Copilot Chat Agent モード）での初回セッションではオンボーディング（プロジェクト理解メモの生成）が走る。  

## 動作確認

- 起動〜ツール有効化の共通手順は mcp-basics.md の「起動と動作確認」を参照。
- 動作確認の目安: `find_symbol` で `xTaskCreate` の定義が `tasks.c` に解決されること。  
クロスファイル参照が0件しか返らない場合は、clangd が compile_commands.json を読めていないサインである。
- Serena はダッシュボード（<http://localhost:24282/dashboard/>）でも状態を確認できる。

## 参考リンク

- Serena: <https://github.com/oraios/serena> （C/C++ サポート: <https://oraios.github.io/serena/01-about/020_programming-languages.html> ）
