# C言語で実装したBash互換シェル

開発リポジトリ: https://github.com/kenrio/Minishell

![ミニシェルデモ](/minishell-1.gif "ミニシェルデモ")

---

Bashの仕様に基づいたシェルをC言語で2人チームで再実装。

入力文字列の字句解析（レキサー）、抽象構文木（AST）への構文解析（パーサー）、コマンド実行という3段階のパイプライン構造で設計し、パイプやリダイレクション、環境変数展開、シグナルハンドリング、ヒアドキュメントなどの基本機能を実装した。Bashの挙動に準拠することを重視し、エラーコードや終了ステータスの扱いを含めた実際のシェルに近い動作を再現した。

機能：
- コマンド実行（絶対パス / `$PATH`環境変数による検索）
- ビルトインコマンド（echo / cd / pwd / export / unset / env / exit）
- パイプ（`|`）
- 入出力リダイレクション（`<` / `>` / `>>`）
- ヒアドキュメント（`<<`）
- シングルクォート / ダブルクォート
- 環境変数展開（`$VAR`、`$?`）
- シグナルハンドリング（`Ctrl-C` / `Ctrl-\` / `Ctrl-D`）

担当範囲：

シェルの「入力解析パイプライン（lexer / parser）」と「シグナル処理基盤」を主に担当した。

- メインループ：readlineによる入力、シグナル状態に応じた制御フロー
- レキサー：入力文字列のトークン化、クォート処理
- パーサー：再帰下降構文解析による抽象構文木（AST）の構築
- シグナルハンドリング：5つの状態に応じたシグナル制御（入力待機中、ヒアドキュメント親/子、コマンド実行親/子）
- ヒアドキュメント：子プロセスによる入力収集
- 一部ビルトインコマンドの実装：`env`, `export`, `unset`, `exit`

## 実行環境

- macOS / Linux
- C言語コンパイラ (clang / gcc)
- GNU Readline

macOSでは Homebrew 経由で readline をインストールする必要がある（Makefileで自動的に `/opt/homebrew/opt/readline` を参照する）。Linuxでは標準でインストール済みの readline を使用する。

## 実行方法

依存ライブラリのインストール (macOSの場合)：
```
$ brew install readline
```
ビルド：
```
$ make
```
実行：
```
$ ./minishell
```

起動後はBashと同様にコマンドを入力できる。

## 苦労した点

- パイプライン構造を正しく表現できるAST設計

	シェルが受け付けるコマンドは、単純なコマンドだけでなく `cmd1 | cmd2 | cmd3` のようなパイプライン、`cmd > file` のようなリダイレクションを含む複雑な構造を持つ。これを処理するにあたり、トークン列を直接処理する方式では、入れ子構造や優先順位を扱う際にコードが複雑化することが課題となった。

	対策として、トークン列を構文木として表現する方針を採用し、再帰下降パーサーによってASTを構築する設計とした。ASTのデータ構造は、コマンドノード（CMD）とパイプノード（PIPE）を `t_node_type` のタグと `union` で統一し、ノード種別を抽象化する形でC言語上で多態的に扱える構造とした。リダイレクトはCMDノードの属性として連結リストで保持する形とした。
	- `includes/minishell.h` の `t_ast` 構造体定義：`t_node_type type` と `union u_data { struct s_cmd cmd; struct s_pipe pipe; }`の組み合わせ
	- `src/parser/parser_pipe.c`, `src/parser/parse_cmd.c`：再帰下降によるAST構築

	加えて、パイプノード構築時に左側のサブツリーをディープコピーする設計を採用したことで、ノードの再利用とメモリ管理が両立できる構造になった一方、各ノードのフィールドを正しくコピー・解放する責任を明確化する必要があり、メモリ管理の設計に注意を要した。また、終了ステータス（`$?`）を子ノードから更新可能にするため、CMDノードに status pointer (`int *stp`) を埋め込み、AST全体で共有する設計とした。これにより、深い階層からでも `$?` の値を更新できる構造を実現した。
	- `src/parser/copy_ast.c` の `copy_ast()`：CMDノードの全フィールド（name, path, argv, envp, redirects）を再帰的にディープコピー
	- `src/parser/parser_utils.c` の `reset_cmd_node_fields()`：mainの`exit_status`のアドレスをCMDノードの`stp`に設定

- チーム開発における開発プロセス設計と運用

	初めてのチーム開発であったため、開発プロセスの構築から着手した。GitHub Issueによるタスク管理とfeatureブランチによる並行開発フローを設計し、担当範囲とファイル境界を明確にすることでコンフリクトを最小限に抑えた。

## 工夫・力をいれた点

- 再帰下降パーサーによるAST設計と構造化

	BNFで記述されたシェル文法に基づき、再帰下降構文解析を採用した。文法規則ごとに対応するパーサー関数を定義し、トークン列から抽象構文木を構築する設計とした。これにより、文法の構造とコード構造の対応関係が明確になり、新しい構文要素の追加や既存規則の変更が容易な実装となった。
	- `src/parser/parse_pipe.c` の `parse_pipe()`：パイプ規則のパース
	- `src/parser/parse_cmd.c` の `parse_cmd()`：コマンド規則のパース
	- `src/parser/make_pipe_node.c` の `make_pipe_node()`：パイプノード構築
	- `src/parser/make_cmd_node.c` の `make_cmd_node()`：コマンドノード構築（コマンド名、パス解決、argv構築）
	- `src/parser/add_args.c` の `add_args()`、`src/parser/add_redirect.c` の `add_redirect()`：引数とリダイレクトの追加処理

- タグ付きunionによる多態的なAST表現

	ASTのノードはCMD（コマンド）とPIPE（パイプ）の2種類があり、それぞれ持つべきデータが異なる。これを `t_node_type` のタグと `union` を組み合わせたタグ付き共用体（tagged union）として表現することで、両ノードを同一の`t_ast`型として扱える構造とした。実行処理側ではノード種別に応じてディスパッチする形式となり、構文木の走査と実行を分離して実装できた。
	- `includes/minishell.h` の `t_ast` 構造体定義：`t_node_type type` と `union u_data { struct s_cmd cmd; struct s_pipe pipe; }`の組み合わせ
	- `src/ast/exec_ast.c` の `execute_ast()`：`ast_node->type` でディスパッチ

- シグナル処理における状態管理

	シェルにおけるシグナルハンドリングは、プロンプト待機中・コマンド実行中・ヒアドキュメント入力中などの複数の状態で異なる挙動が求められ、状態管理が複雑になる課題があった。例えば `Ctrl-C` は、プロンプト待機中はプロンプトをリセット、コマンド実行中は子プロセスを終了、ヒアドキュメント入力中は入力を中断するなど、状態ごとに異なる処理が必要となる。

	対策として、グローバル変数 (`g_signal`) によるシグナル状態の伝搬と、readlineの制御（`rl_done` など）を組み合わせることで、状態ごとの適切なシグナル処理を実現した。具体的には、5つの状態（プロンプト待機中、ヒアドキュメント親プロセス / 子プロセス、コマンド実行中親プロセス / 子プロセス）それぞれに対応するハンドラ設定関数を定義し、コンテキスト切り替え時に適切なハンドラを設定する設計とした。
	- `src/main.c` の `get_input_line()`：`g_signal` と `rl_done`を使ったreadlineと連携
	- `src/signal/signal_handler.c` の `idle_handler()`、`exec_handler()`：シグナル受信時の処理
	- `src/signal/signal_set_handler.c`：5つの状態に対応するハンドラ設定関数

## 参考にしたソースファイル

- 低レイヤを知りたい人のためのCコンパイラ作成入門 (Rui Ueyama) (https://www.sigbus.info/compilerbook)
	- 参考箇所: 字句解析、再帰下降構文解析、抽象構文木の設計

- The Architecture of Open Source Applications - bash章 (https://m-takagi.github.io/aosa-ja/aosa.pdf)
	- 参考箇所: シェル全体のアーキテクチャ（Lexer / Parser / Executor の分離設計）

- シェル実装の課題を完走した (JUN's Blog) (https://jun-networks.hatenablog.com/entry/2021/07/02/034002)
	- 参考箇所: シェルのBNF文法、ヒアドキュメントの子プロセスによる入力収集、ビルトインコマンドの実行方式、環境変数の連結リストでの保持

- 環境変数の取り扱いについて (東京大学 ECCS講義資料) (https://lecture.ecc.u-tokyo.ac.jp/hideo-t/tutorials/set_environment_variables.html)
	- 参考箇所: 環境変数の取得・追加・削除の仕組み

- execveの使い方 (プログラミング雑記) (https://progzakki.sanachan.com/program-lang/c/how-to-use-execve/)
	- 参考箇所: execve系システムコールによる外部プログラムの実行

- The GNU Readline Library を使った (メモの日々) (http://ogawa.s18.xrea.com/tdiary/20080319p02.html)
	- 参考箇所: readlineを用いた入力処理、シグナル処理との連携（rl_done等）