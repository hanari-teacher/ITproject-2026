# TODOアプリ V1 設計書

## 1. 作るものとファイルの役割

既存の Express + TypeScript + EJS アプリケーションに対して、TODO 管理機能（V1）を追加します。画面側は Fetch API を使用して非同期通信を行い、サーバー側は RESTful API として実装します。

### 1.1 ファイル構成

授業テキストの標準構成に合わせて以下のように配置します。

```text
プロジェクト直下/
├── index.ts                     共通：TODO用ルーター（/api/todo/v1/todos, /todo）の登録（追記は最低限）
├── db.ts                        共通：PostgreSQL (pg Pool) 接続インスタンスのエクスポート
├── package.json、tsconfig.json    共通
├── public/todo/todo.js          ブラウザ用JavaScript（TodoApiクラスとCRUD非同期通信・画面描画）
├── views/todo/index.ejs         TODO画面（EJSテンプレート）
├── src/apps/todo/
│   ├── todoRouter.ts            ルーティング定義（エンドポイントとControllerの接続）
│   ├── todoController.ts        リクエスト/レスポンス制御、入力バリデーション、HTTPステータス管理
│   └── todoModel.ts             DBアクセスロジック（SQLクエリの実行と型変換）
├── docs/todo/
│   ├── 01_requirements.md
│   ├── 02_design.md (本ファイル)
│   ├── 03_implementation_plan.md
│   ├── 04_test_results.md
│   └── /todo/v2/
└── sql/
    └── todo/020_create_todo_tasks.sql TODO用テーブル作成DDL

```

---

## 2. リクエストはどこを通るか

クライアントからのリクエストは以下のフローで処理されます。

```text
[ブラウザ /todo] ──(Fetch)──> [index.ts] 
                              │ (ルーター委譲)
                              ▼
                      [todoRouter.ts]
                              │ (ルーティング)
                              ▼
                    [todoController.ts] ──(Validation/HTTP制御)
                              │
                              ▼
                       [todoModel.ts]  ──(SQLクエリ実行)
                              │
                              ▼
                          [db.ts]      ──(pg Pool)──> [Neon PostgreSQL]

```

1. **画面ルーティング (`GET /todo`)**: `index.ts` または `todoRouter.ts` から `views/todo/index.ejs` をレンダリング。
2. **APIルーティング (`/api/todo/v1/todos`)**:
* ブラウザの `todo.js` から Fetch リクエストを送信。
* `index.ts` を経由して `todoRouter.ts` がハンドリング。
* `todoController.ts` がパラメータバリデーションを行い、`todoModel.ts` を呼び出す。
* `todoModel.ts` は `db.ts` からエクスポートされた `pool` を使用して DB 操作を実施。





---

## 3. DBの設計

### 3.1 テーブル定義（`sql/todo/020_create_todo_tasks.sql`）

```sql
CREATE TABLE IF NOT EXISTS todo_tasks (
    id SERIAL PRIMARY KEY,
    title VARCHAR(100) NOT NULL,
    completed BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

```

### 3.2 カラム定義

| カラム名 | 型 | 制約 | 説明 |
| --- | --- | --- | --- |
| `id` | `SERIAL` | PRIMARY KEY | タスク識別ID（1以上の自動連番整数） |
| `title` | `VARCHAR(100)` | NOT NULL | タスク内容（トリム後 1〜100 文字） |
| `completed` | `BOOLEAN` | NOT NULL DEFAULT FALSE | 完了状態（`true`: 完了, `false`: 未完了） |
| `created_at` | `TIMESTAMPTZ` | DEFAULT CURRENT_TIMESTAMP | 作成日時 |
| `updated_at` | `TIMESTAMPTZ` | DEFAULT CURRENT_TIMESTAMP | 更新日時 |

---

## 4. APIの共通ルール

### 4.1 ベースURL

`/api/todo/v1/todos`

### 4.2 データ構造（TypeScript 型定義）

```typescript
export interface TodoTask {
  id: number;
  title: string;
  completed: boolean;
  created_at: string;
  updated_at: string;
}

```

### 4.3 レスポンス形式

* **成功時**: JSONオブジェクト、配列、または 204 No Content（ボディなし）。
* **エラー時共通フォーマット (TODO専用)**:
```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "エラーの理由説明"
  }
}

```



---

## 5. 各機能の詳細設計

### 5.1 REQ-01 TODO一覧を取得する

* **エンドポイント**: `GET /api/todo/v1/todos`
* **Controller**: `todoController.getTodoList`
* **Model**: `todoModel.findAll()`
* SQL: `SELECT id, title, completed, created_at, updated_at FROM todo_tasks ORDER BY id ASC;`


* **レスポンス**:
* 成功 (200 OK): `TodoTask[]` (0件の場合は `[]`)



### 5.2 REQ-02 TODOを登録する

* **エンドポイント**: `POST /api/todo/v1/todos`
* **Controller**: `todoController.createTodo`
* バリデーション: `req.body.title` の文字列チェック。トリム後 `length < 1` または `length > 100` の場合 422 を返却。


* **Model**: `todoModel.create(title: string)`
* SQL: `INSERT INTO todo_tasks (title) VALUES ($1) RETURNING id, title, completed, created_at, updated_at;`


* **レスポンス**:
* 成功 (201 Created): 作成された `TodoTask` オブジェクト
* エラー: 400 (JSON形式不正), 422 (文字数違反)



### 5.3 REQ-03 完了状態を変える

* **エンドポイント**: `PATCH /api/todo/v1/todos/:id`
* **Controller**: `todoController.updateTodoStatus`
* バリデーション: パスパラメータ `id` が1以上の整数か確認（失敗時400）。`req.body.completed` が boolean か確認（失敗時400）。


* **Model**: `todoModel.updateStatus(id: number, completed: boolean)`
* SQL: `UPDATE todo_tasks SET completed = $1, updated_at = CURRENT_TIMESTAMP WHERE id = $2 RETURNING id, title, completed, created_at, updated_at;`


* **レスポンス**:
* 成功 (200 OK): 更新された `TodoTask` オブジェクト
* エラー: 400 (パラメータ不正), 404 (指定IDが存在しない)



### 5.4 REQ-04 TODOを削除する

* **エンドポイント**: `DELETE /api/todo/v1/todos/:id`
* **Controller**: `todoController.deleteTodo`
* バリデーション: パスパラメータ `id` が1以上の整数か確認（失敗時400）。


* **Model**: `todoModel.delete(id: number)`
* SQL: `DELETE FROM todo_tasks WHERE id = $1 RETURNING id;`


* **レスポンス**:
* 成功 (204 No Content): レスポンス本文なし
* エラー: 400 (パラメータ不正), 404 (指定IDが存在しない)



---

## 6. 画面とTodoApiの設計

フロントエンド (`public/todo/todo.js`) は、API呼び出しを担当する `TodoApi` クラスと、CRUDそれぞれの操作に対になるイベント・UI更新関数を設ける構造とします。

### 6.1 `TodoApi` クラス設計

```javascript
class TodoApi {
  static async fetchAll() {
    const res = await fetch('/api/todo/v1/todos');
    if (!res.ok) throw await res.json();
    return await res.json();
  }

  static async create(title) {
    const res = await fetch('/api/todo/v1/todos', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ title }),
    });
    if (!res.ok) throw await res.json();
    return await res.json();
  }

  static async updateStatus(id, completed) {
    const res = await fetch(`/api/todo/v1/todos/${id}`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ completed }),
    });
    if (!res.ok) throw await res.json();
    return await res.json();
  }

  static async delete(id) {
    const res = await fetch(`/api/todo/v1/todos/${id}`, {
      method: 'DELETE',
    });
    if (!res.ok) throw await res.json();
    return; // 204 No Content
  }
}

```

### 6.2 CRUDごとの対になる画面処理関数

CRUD操作に対応する API 呼出関数と UI 描画/ハンドラ関数を 1:1 で対応させて定義します。

| CRUD | API呼び出し (TodoApi) | 対になるUI操作ハンドラ | 処理概要 |
| --- | --- | --- | --- |
| **Read** | `TodoApi.fetchAll()` | `loadAndRenderTodos()` | 初期表示・再取得時、APIから全件取得してDOM描画 |
| **Create** | `TodoApi.create(title)` | `handleCreateTodo(event)` | フォーム送信時、登録APIを呼んで一覧を更新 |
| **Update** | `TodoApi.updateStatus(id, flag)` | `handleToggleComplete(id, flag)` | チェックボックス切替時、更新APIを呼んでUI反映 |
| **Delete** | `TodoApi.delete(id)` | `handleDeleteTodo(id)` | 削除ボタン押下時、削除APIを呼んでDOMから除外 |

---

## 7. エラーはどこで処理するか

### 7.1 バリデーション・ビジネスエラー (`todoController.ts`)

* リクエスト形式の不整合（400）、入力文字列の長さ違反（422）、対象データ非存在（404）は `todoController.ts` にて判定し、即座に該当ステータスコードと JSON エラーオブジェクトをレスポンスします。

### 7.2 DB・例外エラー (`todoModel.ts` ＆ Express Catch)

* DB接続不可やSQL実行時の例外は `todoModel.ts` からスローされ、`todoController.ts` 内の `try-catch` ブロックで受けて `500 Internal Server Error` または `503 Service Unavailable` としてレスポンスします。

### 7.3 フロントエンドのエラー表示 (`todo.js`)

* `TodoApi` からスローされたエラーオブジェクト（`res.json()` の中身）を受け取り、画面上の共通エラーメッセージ表示領域に `message` を出力します。

---

## 8. 実装への注意事項

1. **`index.ts` の追記は最低限に抑える**
* `index.ts` への追加ルーティング設定は以下の2行程度に留め、ロジックはすべて `todoRouter.ts` に委譲します。
```typescript
import { todoRouter } from './src/apps/todo/todoRouter';
// ...
app.use('/', todoRouter); // /todo 画面および /api/todo/v1/todos API をルーティング

```




2. **`db.ts` の共通利用**
* `todoModel.ts` では `import { pool } from '../../../db';` として既存の接続プールを再利用し、新規の接続プール作成や接続情報の再読み込みを行わないでください。




3. **204 No Content のレスポンス処理**
* Express 側の DELETE 成功時は `res.status(204).send()` または `res.status(204).end()` を使用し、レスポンスボディを送信しないでください。
* フロントエンド側（`todo.js`）でも `204` の場合は `res.json()` をパースしようとせずリターンしてください。



---

## 9. 要確認事項

本設計書において、実装前に確認・最終決定が必要な項目は以下の通りです。

1. **`index.ts` でのルーティング定義形式**
* `app.use('/api/todo/v1/todos', apiRouter)` と `app.use('/todo', viewRouter)` に分けてマウントするか、`todoRouter.ts` 内部でパスを完結させるかについて要確認。

