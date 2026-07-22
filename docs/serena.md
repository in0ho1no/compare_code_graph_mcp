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
Windows ネイティブで完結させるため、Windows シミュレータ用の `MSVC_MINGW` ポートを使ってコンパイルDBだけ作る。  
設定ヘッダはリポジトリ同梱の `examples/template_configuration/FreeRTOSConfig.h` を利用する。

前提ツールとして cmake / ninja / MinGW-w64 gcc が PATH に通っていること。

```powershell
cmake --version; ninja --version; gcc --version   # 3つとも通ることを確認
```

リポジトリルートで以下を実行する。実際にビルドする必要はなく、configure が完了した時点で compile_commands.json は生成される。

```powershell
cmake -S . -B build -G Ninja -DCMAKE_C_COMPILER=gcc `
      -DCMAKE_EXPORT_COMPILE_COMMANDS=ON `
      -DFREERTOS_PORT=MSVC_MINGW -DFREERTOS_HEAP=4 `
      "-DFREERTOS_CONFIG_FILE_DIRECTORY=$PWD/examples/template_configuration"

# clangd はルート直下の compile_commands.json を探す
Copy-Item build/compile_commands.json .
```

> - `FREERTOS_CONFIG_FILE_DIRECTORY` は deprecated 扱いだが V11.3.0 では動作し、freertos_config ターゲットを自作せずに済む。  
> configure 時に出る deprecated 警告と「No project() command is present」警告は無視してよい。
> - 生成される DB に入るのはカーネル本体 + `portable/MSVC-MingW/port.c` + `heap_4.c` の9ファイルである。  
> Serena/clangd から見えるのはこの構成だけになる（検証項目4の挙動に効く）。
> - configure に失敗した場合はキャッシュが壊れている可能性があるため、`build/` を削除してから再実行する。
> - 別のリポジトリに適用する場合はポートと設定ヘッダのパスを読み替え、どの構成で作ったかを必ず記録に残すこと（Serena の見えるビュー＝この構成、が検証結果の解釈に効くため）。

### clangd へのシステムインクルードパスの設定（`.clangd`）

compile_commands.json のコンパイラは MinGW gcc だが、clangd は gcc のシステムインクルードパス（`stdlib.h` 等の場所）を自動では解決できず、そのままでは `'stdlib.h' file not found` で解析精度が落ちる。  
リポジトリ直下に `.clangd` を生成してパスを教える。

```powershell
# gcc からシステムインクルードパスを取得して .clangd を生成する（リポジトリルートで実行）
$inc = & gcc -E "-Wp,-v" -xc nul 2>&1 | Select-String '^ ' | ForEach-Object { $_.Line.Trim() }
@("CompileFlags:", "  Add:", "    - `"--target=x86_64-w64-mingw32`"") +
    ($inc | ForEach-Object { "    - `"-isystem$_`"" }) | Set-Content .clangd
```

生成後、以下で確認する。

```powershell
clangd --check=tasks.c
```

システムヘッダの `file not found` エラーが出なければOK。  
出力に多数出る `tweak: SwapBinaryOperands ==> FAIL` は clangd 内部のリファクタリング自己テストのノイズであり、無視してよい。

### MCP クライアントへの登録

GitHub Copilot（VS Code）の場合は `.vscode/mcp.json` に以下のエントリを追加する（ファイル全体の書き方は mcp-basics\.md 参照）。  
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
ただし `${workspaceFolder}` は使えないため、`--project` は対象リポジトリの実パスで指定する（設定例は mcp-basics\.md）。

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

- 起動〜ツール有効化の共通手順は mcp-basics\.md の「起動と動作確認」を参照。
- 動作確認の目安: `find_symbol` で `xTaskCreate` の定義が `tasks.c` に解決されること。  
クロスファイル参照が0件しか返らない場合は、clangd が compile_commands.json を読めていないサインである。
- Serena はダッシュボード（<http://localhost:24282/dashboard/>）でも状態を確認できる。

## 参考リンク

- Serena: <https://github.com/oraios/serena> （C/C++ サポート: <https://oraios.github.io/serena/01-about/020_programming-languages.html> ）
