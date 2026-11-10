# TODOアプリ V1 実装計画書

## 1. 全体方針と注意事項

* `01_requirements.md` および `02_design.md` に基づき、小規模なステップに分割して実装を進めます。
* **既存コードの保護**: `index.ts` の既存ルーティング（`/` および `/api/db-check`）および `db.ts` を一切変更・破損させず、TODO用ルーティングのみを追加します。


* **`index.ts` への追記は最小限**: JSONのパース用ミドルウェア `app.use(express.json())` と `todoRouter` の読み込み・マウントの最小限の差分にとどめます。


* **SQL作業の分離**: SQLファイルの作成（Git管理）と Neon SQL Editor でのテーブル作成（手動実行）を独立した工程とします。

---

## 2. 実装ステップ一覧

| Step | 工程名 | 概要 | 主な対象ファイル |
| --- | --- | --- | --- |
| **Step 1** | テーブル作成SQLの保存 | DDLのGit管理化 | `sql/todo/020_create_todo_tasks.sql` |
| **Step 2** | Neon DBでのテーブル作成 | Neon SQL Editorでテーブル作成手動実行 | Neon PostgreSQL Dashboard |
| **Step 3** | DBアクセスモデルの実装 | SQLクエリ実行関数群の追加 | `src/apps/todo/todoModel.ts` |
| **Step 4** | コントローラーの実装 | 入力チェック・HTTPレスポンス制御 | `src/apps/todo/todoController.ts` |
| **Step 5** | ルーターの実装 | パスとコントローラーの結びつけ | `src/apps/todo/todoRouter.ts` |
| **Step 6** | EJS画面テンプレートの実装 | UIの構造化とレイアウト作成 | `views/todo/index.ejs` |
| **Step 7** | フロントエンドJavaScriptの実装 | `TodoApi` クラスとCRUD操作ハンドラ | `public/todo/todo.js` |
| **Step 8** | `index.ts` へのルーター登録 | TODOルーティングとJSONミドルウェア追加 | `index.ts` |
| **Step 9** | 全体結合テスト・動作確認 | 全機能の疎通とエラーハンドリング確認 | アプリケーション全体 |

---

## 3. 各ステップの詳細計画

### Step 1: テーブル作成SQLの保存

* **対象ファイル**: `sql/todo/020_create_todo_tasks.sql`
* **作業内容**:
* `sql/todo/` ディレクトリを作成し、`todo_tasks` テーブル作成用の DDL を記述した SQL ファイルを新規作成する。


```sql
CREATE TABLE IF NOT EXISTS todo_tasks (
    id SERIAL PRIMARY KEY,
    title VARCHAR(100) NOT NULL,
    completed BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

```


* **確認コマンド**: `ls -la sql/todo/020_create_todo_tasks.sql`
* **期待結果**: ファイルが正常に作成され、リポジトリに保存されること。
* **失敗時に最初に見る場所**: パス名（`sql/todo/` ディレクトリが存在するか）および文字コード (UTF-8)。

---

### Step 2: Neon DBでのテーブル作成（手動実行）

* **対象環境**: Neon PostgreSQL Dashboard (SQL Editor)
* **作業内容**:
* Step 1 で作成した SQL の内容を Neon の SQL Editor に貼り付けて実行し、データベース上に `todo_tasks` テーブルを作成する。


* **確認手順（SQL Editor上）**: `SELECT table_name FROM information_schema.tables WHERE table_name = 'todo_tasks';` を実行。
* **期待結果**: 検索結果に `todo_tasks` テーブルが返却されること。
* **失敗時に最初に見る場所**: `DATABASE_URL` で指定された Neon データベース・プロジェクトと同じプロジェクトの SQL Editor を開いているか。

---

### Step 3: DBアクセスモデル（`todoModel.ts`）の実装

* **対象ファイル**: `src/apps/todo/todoModel.ts`
* **作業内容**:
* `db.ts` から `pool` をインポートする。


* `TodoTask` インターフェースを定義する。
* 4操作に対応する `findAll`, `create`, `updateStatus`, `delete` の 4つの非同期関数を実装する。


* **確認コマンド**: `npx tsc --noEmit`
* **期待結果**: TypeScript のコンパイルエラーが 0 件であること。
* **失敗時に最初に見る場所**: `db.ts` からのインポートパス（`../../../db` 等）および型定義ミス。



---

### Step 4: コントローラー（`todoController.ts`）の実装

* **対象ファイル**: `src/apps/todo/todoController.ts`
* **作業内容**:
* `getTodoList`, `createTodo`, `updateTodoStatus`, `deleteTodo` ハンドラ関数を実装する。
* 各ハンドラ内で入力バリデーション（`title` トリム後1〜100文字チェック、`id` 1以上整数チェック、`completed` booleanチェック）を実装。
* 要件定義に指定されたエラー時ステータス（400, 404, 422, 500）と JSON フォーマットを返却する処理を記述。


* **確認コマンド**: `npx tsc --noEmit`
* **期待結果**: 型エラーが発生せず、型が適切に付与されていること。
* **失敗時に最初に見る場所**: Express の `Request`, `Response` の型付け、およびバリデーション条件分岐。

---

### Step 5: ルーター（`todoRouter.ts`）の実装

* **対象ファイル**: `src/apps/todo/todoRouter.ts`
* **作業内容**:
* `express.Router()` インスタンスを生成する。
* `GET /todo` （EJS `views/todo/index.ejs` のレンダリングルート）を定義する。
* REST API 用エンドポイント (`GET`, `POST`, `PATCH`, `DELETE`) と `todoController` をバインドする。
* `export const todoRouter` または `export default` としてエクスポートする。


* **確認コマンド**: `npx tsc --noEmit`
* **期待結果**: エラーなく TypeScript のトランスパイルチェックを通過すること。
* **失敗時に最初に見る場所**: `todoController` からの関数インポート名とルーティングのパスの指定ミス。

---

### Step 6: EJS画面テンプレート（`index.ejs`）の実装

* **対象ファイル**: `views/todo/index.ejs`
* **作業内容**:
* TODO管理画面のHTML構造を実装する。
* タスク追加フォーム、エラーメッセージ表示エリア、タスク一覧コンテナ領域（`<ul>` または `<div>`）を作成。
* `<script src="/todo/todo.js"></script>` を読み込むタグを記述。


* **確認コマンド**: `ls -la views/todo/index.ejs`
* **期待結果**: ファイルが存在し、HTML構造および `<script>` タグが正しく配置されていること。
* **失敗時に最初に見る場所**: script タグのパス（`public` 配下を指しているか）。

---

### Step 7: フロントエンドJavaScript（`todo.js`）の実装

* **対象ファイル**: `public/todo/todo.js`
* **作業内容**:
* APIアクセスを担当する `TodoApi` クラス（`fetchAll`, `create`, `updateStatus`, `delete`）を実装する。
* 各 CRUD に対になる UI 操作・イベントハンドラ関数（`loadAndRenderTodos`, `handleCreateTodo`, `handleToggleComplete`, `handleDeleteTodo`）を記述する。
* DOMContentLoaded イベントで `loadAndRenderTodos()` を呼び出す。
* `DELETE` 成功時は 204 返却のため `res.json()` をパースしない処理を含める。


* **確認コマンド**: ブラウザの開発者ツール Console に構文エラー（SyntaxError）が出ないこと（Step 8 実施後に確認）。
* **期待結果**: Fetch 呼出と DOM 操作が正しく紐づいていること。
* **失敗時に最初に見る場所**: `fetch` リクエストのレスポンスステータス判定および JSON パースの処理。

---

### Step 8: `index.ts` へのルーター登録（最小追記）

* **対象ファイル**: `index.ts`
* **作業内容**:
* 既存の処理・ルーティングを壊さないよう、以下の **最小限の追記（2箇所・合計3行程度）** のみを行う。





```typescript
// インポート文の追加（上部）
import { todoRouter } from './src/apps/todo/todoRouter';

// ... (既存の基本設定)
app.use(express.json()); // POST/PATCH の JSONボディを受け取るために追加[cite: 2]
app.use(express.urlencoded({ extended: true })); // 既存[cite: 2]

// ... (既存ルーティング)

// TODOルーティングの追加（最下部・app.listenの手前に最小追記）
app.use('/', todoRouter);

```

* **確認コマンド**:
1. `npm run dev` または `npx ts-node index.ts` でサーバー起動。
2. `curl http://localhost:3000/` （既存ルート確認）


3. `curl http://localhost:3000/api/db-check` （既存DBチェックルート確認）




* **期待結果**:
* 既存ルート `/` で `20261110 test UP` が返却されること。


* `/api/db-check` で `{"status":"ok","database":"connected"}` が返却されること。


* サーバースタート時にコンソールエラーが発生しないこと。


* **失敗時に最初に見る場所**: `index.ts` の差分位置、`app.use(express.json())` の挿入位置（ルーティング前にあるか）。



---

### Step 9: 全体結合テスト・動作確認

* **対象環境**: ローカル開発環境（`http://localhost:3000/todo`）
* **作業内容**:
* ブラウザで `/todo` にアクセスし、一覧取得、新規作成、完了切り替え、削除の 4 機能を実行する。
* 空文字送信（422）、101文字以上送信（422）時のエラー表示を確認する。


* **確認コマンド / 手順**:
* ブラウザ開発者ツールの Network タブを開き、各 Fetch 操作のレスポンスコード（200, 201, 204, 422 等）を確認。


* **期待結果**: 4機能がすべて正常動作し、画面が自動更新されること。
* **失敗時に最初に見る場所**: ブラウザ Console のエラーログ、および Express サーバーのコンソール出力ログ。
