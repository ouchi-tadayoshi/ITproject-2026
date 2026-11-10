TODOアプリ V1 設計書1. 作るものとファイルの役割1.1 システム概要既存の Express + TypeScript + EJS アプリケーションに TODO 管理機能（CRUD 4操作）を追加する。
データベースには Neon PostgreSQL を利用し、Vercel 上の Serverless Functions 環境で稼働させる。1.2 ファイル構成と各役割（一般例）※実際のディレクトリ構造やファイル配置は既存リポジトリに合わせる必要があります（要確認）。.
├── src/
│   ├── index.ts               # エントリーポイント（追記はルーティング登録等の最低限に留める）
│   ├── routes/                # ルーティング定義
│   │   ├── viewRouter.ts      # 画面（/todo）用ルーター（要確認）
│   │   └── todoApiRouter.ts   # REST API（/api/todo/v1/todos）用ルーター（要確認）
│   ├── controllers/           # リクエスト制御・バリデーション
│   │   └── todoController.ts  # リクエスト受け取り、レスポンス成形（要確認）
│   ├── services/              # ビジネスロジック
│   │   └── todoService.ts     # CRUDロジック・DB操作呼び出し（要確認）
│   └── db/                    # DB接続
│       └── client.ts          # Neon DB接続設定（DATABASE_URL を利用）（要確認）
├── views/                     # EJS テンプレート
│   └── todo.ejs               # TODO画面のビュー（要確認）
└── public/                    # 静的ファイル
    └── js/
        ├── todoApi.js         # クライアント側 Fetch API カプセル化クラス
        └── todoApp.js         # DOM操作および UI イベントハンドラー（要確認）
2. リクエストはどこを通るか2.1 画面表示処理 (GET /todo)Client (Browser): URL /todo にアクセス。Express (index.ts -> viewRouter): /todo リクエストを受容。Controller / View: views/todo.ejs をレンダリングし HTML を返却。Client (Browser): 読み込まれた todoApp.js が API (GET /api/todo/v1/todos) を叩き、初期データを取得して描画。2.2 REST API 処理 (/api/todo/v1/todos)Client (todoApp.js): ユーザー操作（作成/完了切替/削除）を検知。Client (TodoApi クラス): Fetch API を実行。Express (index.ts -> todoApiRouter): 各 HTTP メソッドに応じたルートへ振り分け。todoController: 入力パラメータ（title, id, is_completed）のバリデーションを実施。todoService / DB Client: Neon PostgreSQL に対して SQL を実行。todoController: 成功/エラーの共通 JSON 形式でレスポンスを返却。Client (todoApp.js): レスポンス結果を受け取り、DOM を動的更新。3. DBの設計3.1 テーブル定義 (todos)Neon PostgreSQL 上に以下のテーブルを構築する（テーブル名・型は要確認）。CREATE TABLE IF NOT EXISTS todos (
    id SERIAL PRIMARY KEY,
    title VARCHAR(100) NOT NULL,
    is_completed BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
3.2 カラム詳細カラム名型制約説明idSERIAL (または BIGINT)PRIMARY KEYTODOの識別子（1以上の整数）titleVARCHAR(100)NOT NULLTODOのタイトル（前後の空白トリム後1〜100文字）is_completedBOOLEANNOT NULL, DEFAULT false完了状態（true: 完了, false: 未完了）created_atTIMESTAMPTZDEFAULT CURRENT_TIMESTAMP作成日時4. APIの共通ルール4.1 ベースURL/api/todo/v1/todos4.2 レスポンスフォーマット成功時 (ステータス 200, 201):{
  "data": { ... }
}
※ GETでの一覧取得時は data: [...] とし、0件の場合は { "data": [] } を返却する。※ DELETE成功時 (ステータス 204) はレスポンス本文なし。エラー時 (ステータス 400, 422, 404, 500, 503):{
  "error": {
    "code": "ERROR_CODE_NAME",
    "message": "エラーメッセージの詳細"
  }
}
5. 各機能の詳細設計 (REQ-01 〜 REQ-04)5.1 REQ-01 TODO一覧を取得するメソッド・エンドポイント: GET /api/todo/v1/todos処理内容: 登録済みの全 TODO を取得。成功レスポンス (200 OK):{
  "data": [
    { "id": 1, "title": "プログラミング課題", "is_completed": false },
    { "id": 2, "title": "環境構築", "is_completed": true }
  ]
}
※データが0件の場合は "data": [] を返却する。5.2 REQ-02 TODOを登録するメソッド・エンドポイント: POST /api/todo/v1/todosリクエストボディ:{ "title": "新しいTODO" }
入力検証:title が必須。title.trim() 後の長さが 1〜100 文字であること。成功レスポンス (201 Created):{
  "data": { "id": 3, "title": "新しいTODO", "is_completed": false }
}
5.3 REQ-03 完了状態を変えるメソッド・エンドポイント: PATCH /api/todo/v1/todos/:idパスパラメータ: id（1以上の整数）リクエストボディ:{ "is_completed": true }
入力検証:id が 1 以上の整数であること。is_completed がブール値（true または false）であること。成功レスポンス (200 OK):{
  "data": { "id": 1, "title": "プログラミング課題", "is_completed": true }
}
5.4 REQ-04 TODOを削除するメソッド・エンドポイント: DELETE /api/todo/v1/todos/:idパスパラメータ: id（1以上の整数）入力検証:id が 1 以上の整数であること。成功レスポンス (204 No Content):レスポンスボディなし（空）6. 画面とTodoApiの設計6.1 クライアント側API通信クラス (TodoApi)Fetch API の処理とエラー共通ハンドリングをモジュール化する。// public/js/todoApi.js（配置場所は要確認）
class TodoApi {
  constructor(baseUrl = '/api/todo/v1/todos') {
    this.baseUrl = baseUrl;
  }

  async getAll() {
    const res = await fetch(this.baseUrl);
    return this._handleResponse(res);
  }

  async create(title) {
    const res = await fetch(this.baseUrl, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ title })
    });
    return this._handleResponse(res);
  }

  async updateStatus(id, isCompleted) {
    const res = await fetch(`${this.baseUrl}/${id}`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ is_completed: isCompleted })
    });
    return this._handleResponse(res);
  }

  async delete(id) {
    const res = await fetch(`${this.baseUrl}/${id}`, { method: 'DELETE' });
    if (res.status === 204) return true;
    return this._handleResponse(res);
  }

  async _handleResponse(res) {
    const json = await res.json().catch(() => ({}));
    if (!res.ok) {
      const error = new Error(json.error?.message || 'エラーが発生しました');
      error.status = res.status;
      error.code = json.error?.code;
      throw error;
    }
    return json.data;
  }
}
6.2 画面イベント制御 (todoApp.js)TodoApi を呼び出し、UI（フォーム送信、チェックボックス操作、削除ボタンクリック）と同期させる。7. エラーはどこで処理するか7.1 階層ごとの処理責務コントローラー層 (入力バリデーション):不正なパラメータ（ID範囲外、タイトル文字数オーバーなど）を検知し、400 または 422 を即座に返却。サービス/データアクセス層 (リソース不在・DBエラー):対象の ID が存在しない場合は 404 を返却。DB 接続エラーや SQL 実行例外は捕捉して 500 または 503 を返却。クライアント層 (UI表示):TodoApi からスローされたエラーを受け取り、画面上の通知エリアやダイアログにエラーメッセージを表示。8. 実装への注意事項index.ts への追記の最小化:ルーターのバインド（app.use(...)）のみに留め、ロジックはコントローラーやサービスにカプセル化する。SQL インジェクション対策:生の文字列結合を行わず、プレースホルダー（$1, $2）を利用したパラメータ化クエリを実行する。Serverless 環境での DB コネクション:Vercel Serverless Functions での過剰接続を防ぐため、接続プールの適切な使用または Neon 用クライアントの単一化を行う。二重バリデーション:クライアント側（JavaScript）での事前チェックに加え、サーバー側（Express Controller）でも同等以上の検証を行う。