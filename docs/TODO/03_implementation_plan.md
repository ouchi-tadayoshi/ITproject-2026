TODOアプリ V1 実装計画書 (docs/todo/03_implementation_plan.md)

本ドキュメントは 01_requirements.md および 02_design.md を根拠とし、既存の Express + TypeScript + EJS アプリケーションに対して、既存ルート（/ および /api/db-check）を破壊することなく安全かつ段階的に TODO 機能を実装するための作業手順を定義する。

全体方針・フォルダ構成

既存環境への影響を最小限に抑えるため、TODO機能に関連するコードは専用のディレクトリに集約する。

サーバーサイド処理: src/apps/todo/

todoRouter.ts: /todo 画面および /api/todo/v1/todos REST API のルーティング定義

todoController.ts: 入力バリデーションおよびレスポンス生成

todoService.ts: DB操作クエリの実行

画面テンプレート: views/todo/index.ejs

クライアント側JS: public/todo/

todoApi.js: Fetch API カプセル化クラス (TodoApi)

todoApp.js: DOM操作およびイベントハンドリング

DBマイグレーションSQL: docs/todo/sql/01_create_todos_table.sql

エントリーポイントへの追加: src/index.ts（最小限のルーティング追加のみ）

実装工程一覧

STEP 1: DBテーブル作成用SQLの作成とGit保存

対象ファイル: docs/todo/sql/01_create_todos_table.sql (新規作成)

作業内容:

docs/todo/sql/ ディレクトリを作成する。

設計書に基づき todos テーブル作成用SQLを記述したファイルを作成しコミットする。

確認コマンド:

cat docs/todo/sql/01_create_todos_table.sql


期待結果: id (SERIAL), title (VARCHAR(100)), is_completed (BOOLEAN), created_at (TIMESTAMPTZ) を持つテーブル作成SQLが正しく出力されること。

失敗時に最初に見る場所: ファイルパスおよびディレクトリ作成ログ。

STEP 2: Neon SQL Editor でのテーブル作成 (手動実行)

対象環境: Neon PostgreSQL コンソール (SQL Editor)

作業内容:

Neon の管理画面を開き、対象プロジェクトの SQL Editor を選択する。

STEP 1 で作成した SQL 文を貼り付けて実行する。

確認用SQL (実行して確認):

SELECT column_name, data_type, character_maximum_length, is_nullable
FROM information_schema.columns
WHERE table_name = 'todos';


期待結果: todos テーブルが存在し、設計通りのカラム（id, title, is_completed, created_at）が取得できること。

失敗時に最初に見る場所: Neon SQL Editor のエラーメッセージ、DATABASE_URL が参照しているデータベース環境（Preview/Production共有DB）。

STEP 3: クライアント側 Fetch 通信クラス (TodoApi) の実装

対象ファイル: public/todo/todoApi.js (新規作成)

作業内容:

REST API (/api/todo/v1/todos) と非同期通信を行う TodoApi クラスを実装する。

getAll(), create(title), updateStatus(id, isCompleted), delete(id) の4メソッドを定義する。

204 No Content のハンドリングおよびエラーレスポンス時の例外スローを実装する。

確認コマンド:

npx tsc --noEmit # 静的チェックまたはビルド確認


期待結果: 構文エラーがなく、TodoApi クラスが正常にエクスポート可能であること。

失敗時に最初に見る場所: public/todo/todoApi.js 内の fetch 呼び出しと JSON パースの条件分岐処理。

STEP 4: ビュー（EJS）と画面動的操作スクリプトの実装

対象ファイル:

views/todo/index.ejs (新規作成)

public/todo/todoApp.js (新規作成)

作業内容:

views/todo/index.ejs にフォーム（タイトル入力、追加ボタン）、TODO一覧表示エリア、エラー表示エリアを記述する。todoApi.js および todoApp.js を <script> で読み込む。

public/todo/todoApp.js に TodoApi を呼び出して画面の初期取得・登録・完了切り替え・削除を行うイベントハンドラーを実装する。

確認コマンド:

ls public/todo/ views/todo/


期待結果: views/todo/index.ejs, public/todo/todoApi.js, public/todo/todoApp.js が規定の位置に存在すること。

失敗時に最初に見る場所: EJS タグの閉じ忘れ、public 静的ファイルの相対パス指定。

STEP 5: サーバーサイド サービス層 (todoService.ts) の実装

対象ファイル: src/apps/todo/todoService.ts (新規作成)

作業内容:

既存の src/db.ts (または同等の DB 接続モジュール) から DB クライアント/プールをインポートする。

パラメータ化クエリ ($1, $2 等) を使用した CRUD4操作（findAll, create, updateStatus, deleteById）の関数を実装する。

確認コマンド:

npx tsc --noEmit


期待結果: 型エラーが発生せず、TypeScript コンパイルが通過すること。

失敗時に最初に見る場所: src/db.ts のエクスポート形式と todoService.ts 側のインポート名の一致。

STEP 6: サーバーサイド コントローラー層 (todoController.ts) の実装

対象ファイル: src/apps/todo/todoController.ts (新規作成)

作業内容:

リクエストパラメータの入力バリデーション（title のトリム後 1〜100 文字チェック、id の 1 以上の整数チェック）を実装する。

バリデーション失敗時は 400 または 422 エラーJSONを返却する。

todoService を呼び出し、該当 ID 不在時は 404、成功時は 200 / 201 / 204 を返却する。例外発生時は 500 / 503 をハンドリングする。

確認コマンド:

npx tsc --noEmit


期待結果: コントローラー関数群がエラーなく定義され、コンパイルが成功すること。

失敗時に最初に見る場所: res.status().json() の呼び出し型定義、バリデーションロジックの条件分岐。

STEP 7: ルーターモジュール (todoRouter.ts) の作成

対象ファイル: src/apps/todo/todoRouter.ts (新規作成)

作業内容:

Express Router() のインスタンスを作成する。

画面表示ルート GET /（views/todo/index.ejs をレンダリング）を定義する。

REST API ルート GET /api/v1/todos, POST /api/v1/todos, PATCH /api/v1/todos/:id, DELETE /api/v1/todos/:id を todoController の各関数へマウントする。

確認コマンド:

npx tsc --noEmit


期待結果: todoRouter が正しくエクスポートできること。

失敗時に最初に見る場所: HTTP メソッドとパスパラメータ :id のルーティング定義。

STEP 8: src/index.ts への最小限のルーティング追加

対象ファイル: src/index.ts (変更)

作業内容:

既存のルーターや /, /api/db-check の定義を維持したまま、todoRouter をインポートする。

静的ファイル提供ミドルウェアに public/todo (または public) が含まれていることを確認する。

最低限の変更として app.use('/todo', todoRouter) および app.use('/api/todo/v1/todos', todoApiRouter)（または統合した app.use('/', todoRouter)）をマウントする。

確認コマンド:

npm run dev  # または npm start


期待結果: サーバーが正常に起動し、既存の / および /api/db-check が従来通り応答すること。

失敗時に最初に見る場所: src/index.ts のミドルウェア追加位置（既存ルートより前に壊す変更を入れていないか）。

STEP 9: 動作確認と受け入れ条件の検証 (ローカル環境)

対象環境: ローカル開発環境 (http://localhost:3000)

作業内容:

ブラウザで /todo にアクセスし、一覧表示・画面描画を確認する。

【REQ-01】DBデータ0件時に GET /api/todo/v1/todos が 200 かつ { "data": [] } を返すことを確認する。

【REQ-02】0文字、空白のみ、1文字、100文字、101文字のタイトルを入力し、400/422 および 201 の挙動を確認する。

【REQ-03】チェックボックス操作で PATCH が発行され、DBおよび画面更新されることを確認する。

【REQ-04】削除ボタン押下で DELETE が発行され、204（本文なし）が返り削除されることを確認する。

確認コマンド:

curl -i http://localhost:3000/api/todo/v1/todos
curl -i -X POST http://localhost:3000/api/todo/v1/todos -H "Content-Type: application/json" -d '{"title":"テストTODO"}'


期待結果: すべての要件定義（REQ-01 〜 REQ-04）の受け入れ条件を満たすこと。

失敗時に最初に見る場所: ブラウザの開発者ツール (Network / Console タブ) および サーバーターミナルのログ。

STEP 10: Vercel Preview / Production 環境デプロイ確認

対象環境: Vercel (Preview & Production)

作業内容:

変更を Git にコミット・プッシュし、Vercel Preview 環境のビルドログを確認する。

Preview URL および Production URL から /todo にアクセスし、共通 Neon DB へのCRUD操作が正常に動作することを確認する。

確認コマンド:

git status
git push origin main  # または PR 作成


期待結果: Vercel 上で Serverless Functions のエラーが発生せず、動作確認が完了すること。

失敗時に最初に見る場所: Vercel Dashboard の Function Logs、環境変数 DATABASE_URL の設定状態。