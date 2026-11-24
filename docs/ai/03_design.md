# AI質問機能 設計書 (`03_design.md`)

## 1. 全体アーキテクチャ・処理フロー

### 1.1. コンポーネント構成

既存の `src/apps/todo/` のレイヤー構造を踏襲し、責務を以下のように分離する。

```text
[Browser: views/ai/index.ejs + public/ai/ai.js]
   │
   ├─ (1) GET /ai ──────────────────► [aiRouter] ─► [aiController.renderIndex] ─► EJSレンダリング
   │
   └─ (2) POST /ai/ask (JSON) ──────► [aiRouter] ─► [aiController.askQuestion]
                                                          │
                                                          ├─ バリデーション (空文字・型・2,000文字上限)
                                                          ▼
                                                    [aiService.generateAnswer]
                                                          │
                                                          ▼
                                                    [Gemini API (@google/genai)]

```

### 1.2. ディレクトリ・ファイル配置

| ファイルパス | 区分 | 主な責務 |
| --- | --- | --- |
| `src/apps/ai/aiRouter.ts` | ルーティング | `/ai` 配下のパスと HTTP メソッドのマッピング

 |
| `src/apps/ai/aiController.ts` | コントローラー | 画面表示制御、入力バリデーション、JSONレスポンス返却、例外の `next(err)` 連携

 |
| `src/apps/ai/aiService.ts` | サービス | `@google/genai` クライアントの初期化と `gemini-2.5-flash` 呼び出しロジック

 |
| `views/ai/index.ejs` | ビュー | AI質問画面のHTMLテンプレート

 |
| `public/ai/ai.js` | クライアントJS | フォーム送信イベント制御、`fetch` 通信、ローディング・回答・エラーのDOM描画

 |
| `index.ts` | エントリーポイント | `app.use('/ai', aiRouter)` のマウント追加（既存ルート・404・エラーハンドラーの手前に配置）

 |

---

## 2. API・エンドポイント設計

### 2.1. エンドポイント一覧

| メソッド | パス | 概要 | レスポンス形式 |
| --- | --- | --- | --- |
| `GET` | `/ai` | AI質問画面の表示 | `text/html` (EJS) |
| `POST` | `/ai/ask` | 質問テキストを受け取り、Gemini API からの回答を返却 | `application/json` |

### 2.2. `POST /ai/ask` 詳細仕様

#### リクエスト

* **Headers**: `Content-Type: application/json`
* **Body**:
```json
{
  "question": "TypeScriptの型ガードとは何ですか？"
}

```



#### レスポンス（正常系: `200 OK`）

```json
{
  "success": true,
  "answer": "TypeScriptの型ガード（Type Guard）とは、特定のスコープ内で変数の型を絞り込むための仕組みです..."
}

```

#### レスポンス（異常系）

| HTTPステータス | 発生条件 | レスポンスボディ例 |
| --- | --- | --- |
| `400 Bad Request` | `question` が未指定、文字列以外、空白のみ、または 2,000文字超過 | `{"success": false, "error": "質問を入力してください。"}`<br>

<br>`{"success": false, "error": "質問は2,000文字以内で入力してください。"}` |
| `500 Internal Server Error` | `AI_API_KEY` 未設定、または Gemini API 呼び出し時の通信・サーバーエラー | `{"success": false, "error": "現在AIからの回答を取得できません。しばらくしてから再度お試しください。"}` |

---

## 3. モジュール詳細設計

### 3.1. `src/apps/ai/aiService.ts`

* **依存パッケージ**: `@google/genai`


* **環境変数**: `process.env.AI_API_KEY`


* **主要関数**: `generateAnswer(question: string): Promise<string>`
1. `process.env.AI_API_KEY` の存在を確認する（未設定の場合は `Error('AI_API_KEY is not configured')` をスロー）。
2. `new GoogleGenAI({ apiKey })` でクライアントを初期化する。
3. `ai.models.generateContent` を以下のパラメータで呼び出す。
* `model`: `'gemini-2.5-flash'`
* `contents`: `question`
* `config.systemInstruction`: `'あなたは親切で正確なアシスタントです。ユーザーの質問に対して、分かりやすく簡潔に日本語で回答してください。'`


4. `response.text` を検証し、空の場合はエラーをスロー、正常であれば文字列を返却する。



### 3.2. `src/apps/ai/aiController.ts`

* **`renderIndex(req: Request, res: Response, next: NextFunction)`**
* `res.render('ai/index', { title: 'AI質問アシスタント' })` を実行する。例外発生時は `next(error)` を呼び出す。


* **`askQuestion(req: Request, res: Response, next: NextFunction)`**
1. `req.body?.question` を取得する。
2. バリデーションを実施する（違反時は `res.status(400).json({ success: false, error: '...' })` を返して終了）。
* 型チェック: `typeof question !== 'string'`
* 必須チェック: `question.trim().length === 0`
* 文字数上限チェック: `question.trim().length > 2000`


3. `try` ブロック内で `await aiService.generateAnswer(question.trim())` を呼び出し、`res.status(200).json({ success: true, answer })` を返却する。
4. `catch (error)` ブロック:
* 外部APIエラー等の詳細をサーバーコンソール（`console.error`）へ出力する。
* APIリクエスト（`/ai/ask`）はJSONレスポンスを期待するため、エラーログ記録後に `res.status(500).json({ success: false, error: '現在AIからの回答を取得できません。しばらくしてから再度お試しください。' })` を返却するか、既存のグローバルエラーハンドラーが `/ai/ask` のJSON判定に対応していない場合に備えてコントローラー側で確実にJSONエラーを返し、サーバークラッシュを防止する（さらに予期せぬフレームワーク例外は `next(error)` へ連携）。







### 3.3. `src/apps/ai/aiRouter.ts` & `index.ts`

* **`aiRouter.ts`**:
* `router.get('/', aiController.renderIndex)`
* `router.post('/ask', aiController.askQuestion)`


* **`index.ts` への統合位置**:
* `express.json()` ミドルウェアが有効になっていることを確認する（未設定の場合は追加）。
* 既存の `app.use('/todo', ...)` 等の直後、かつ 404 ハンドラー・グローバルエラーハンドラーの**手前**に `app.use('/ai', aiRouter)` を配置する。





---

## 4. 画面・フロントエンド設計 (`views/ai/index.ejs` / `public/ai/ai.js`)

### 4.1. 画面レイアウト構成 (`views/ai/index.ejs`)

* **ヘッダー**: ページタイトル（「AI質問アシスタント」）および既存画面（TOP / TODO一覧）への戻りリンク
* **入力フォーム (`#ai-form`)**:
* `<textarea id="question-input" rows="5" maxlength="2000" placeholder="AIに質問したい内容を入力してください..."></textarea>`
* 文字数カウンター表示（`0 / 2000`）
* 送信ボタン（`<button type="submit" id="submit-btn">質問する</button>`）


* **ステータス・エラー領域**:
* ローディング表示（`#loading-indicator`：初期状態 `hidden`、「回答を生成中...」）
* エラー表示枠（`#error-box`：初期状態 `hidden`、赤枠・エラーテキスト表示）


* **回答表示領域 (`#answer-section`)**:
* 初期状態 `hidden`
* 回答本文表示コンテナ（`#answer-content`：`white-space: pre-wrap;` を指定し、改行を保持して表示）



### 4.2. 状態遷移・クライアント処理 (`public/ai/ai.js`)

1. **フォーム送信 (`submit` イベント)**:
* `e.preventDefault()` でデフォルトのページ遷移を抑止する。
* フロントエンド側でも `trim()` 後の空文字・2,000文字超過チェックを行い、不正なら `#error-box` にメッセージを表示して中断する。


2. **リクエスト送信中（Loading状態）**:
* `#submit-btn` を `disabled = true` に設定し、二重送信を防止する。
* `#error-box` と `#answer-section` を非表示にし、`#loading-indicator` を表示する。


3. **非同期通信 (`fetch('/ai/ask', ...)`)**:
* `method: 'POST'`, `headers: { 'Content-Type': 'application/json' }`, `body: JSON.stringify({ question })` で送信する。


4. **レスポンス反映**:
* **成功時 (`response.ok && data.success`)**: `#answer-content.textContent = data.answer` でXSS対策を行いつつテキストをセットし、`#answer-section` を表示する。
* **エラー時（HTTP 400 / 500 またはネットワーク切断）**: `data.error` または通信エラーメッセージを `#error-box.textContent` にセットして表示する。
* **`finally` 処理**: `#loading-indicator` を非表示にし、`#submit-btn` の `disabled` を解除する。



---

## 5. 例外・セキュリティ対策まとめ

* **環境変数漏洩防止**: `.env` に `AI_API_KEY=...` を記載し、`.gitignore` に `.env` が含まれていることを前提とする。クライアント側 JS には一切キーを渡さない。


* **XSS対策**: AIからの返答およびエラーメッセージのDOM挿入には `innerHTML` ではなく必ず `textContent` を使用する。
* **既存機能保護**: `src/apps/todo/` 配下のファイルには一切変更を加えない。



