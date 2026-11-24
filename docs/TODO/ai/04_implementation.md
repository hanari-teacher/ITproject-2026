# AI質問機能 実装手順書・実装コード (`04_implementation.md`)

## 1. 文書概要・実装方針

### 1.1. 文書の位置づけ

本書は「AI質問機能 設計書 (`03_design.md`)」にもとづき、既存 Express / TypeScript / EJS アプリケーションへ AI質問機能（標準モデル: **`gemini-3.5-flash-lite`**）を組み込むための**環境構築手順・ファイル作成手順・完全版実装コード・既存コード統合手順**をまとめた実装手順書である。

### 1.2. 実装時の遵守事項

* **厳格な型安全性（NF-01）**: すべての TypeScript ファイルで `any` 型の使用を禁止し、DTO・リクエスト・レスポンス・エラー型を明示する。
* **機密情報の保護（NF-02）**: `AI_API_KEY` をコードやエラーログに出力しない。例外ログにはエラー種別コード（`AiErrorCode`）のみを記録する。
* **XSS 対策の徹底（NF-02）**: EJS では `<%= ... %>` のみを使用し、フロントエンド JS では `textContent` のみを使用して `innerHTML` を禁止する。
* **既存ルーティングの保護（NF-04）**: `index.ts` へのルーター登録は、必ず既存の 404 ハンドラーおよびグローバルエラーハンドラーよりも**前**に配置する。

---

## 2. 環境構築・事前準備手順

### Step 2-1: Google Gen AI SDK のインストール

プロジェクトルートで以下のコマンドを実行し、公式 SDK（`@google/genai`）を追加する。

```bash
npm install @google/genai

```

### Step 2-2: 環境変数の設定 (`.env` / `.env.example`)

既存の `.env` およびリポジトリ管理用の `.env.example` に以下の環境変数を追記する。

```env
# Google Gemini API 設定
AI_API_KEY=your_gemini_api_key_here
AI_MODEL=gemini-3.5-flash-lite

```

> ※ `AI_MODEL` を省略または空文字にした場合も、アプリケーション側で自動的に `gemini-3.5-flash-lite` が適用される。

### Step 2-3: ディレクトリの作成

新規作成するファイル用のディレクトリを作成する。

```bash
mkdir -p src/apps/ai views/ai public/ai

```

---

## 3. バックエンド実装コード

### Step 3-1: サービス層の実装 (`src/apps/ai/aiService.ts`)

定数・DTO型定義・カスタムエラークラス（`AiServiceError`）・Gemini API 呼び出し・30秒タイムアウト制御・例外マッピングを実装する。

```typescript
import { GoogleGenAI, ApiError } from '@google/genai';

/** AI質問機能 共通定数 */
export const AI_CONSTANTS = {
  MIN_QUESTION_LENGTH: 1,
  MAX_QUESTION_LENGTH: 1000,
  DEFAULT_MODEL: 'gemini-3.5-flash-lite',
  REQUEST_TIMEOUT_MS: 30_000,
} as const;

/** エラー種別コード */
export type AiErrorCode =
  | 'VALIDATION_ERROR'
  | 'CONFIG_ERROR'
  | 'RATE_LIMIT'
  | 'TIMEOUT'
  | 'EXTERNAL_API_ERROR';

/** POST /ai/ask リクエストボディ */
export interface AskRequestBody {
  question?: unknown;
}

/** サービス層からの生成結果 */
export interface GenerateAnswerResult {
  answer: string;
  model: string;
}

/** POST /ai/ask 正常レスポンス */
export interface AskSuccessResponse {
  success: true;
  data: GenerateAnswerResult;
}

/** POST /ai/ask エラーレスポンス */
export interface AskErrorResponse {
  success: false;
  error: {
    code: AiErrorCode;
    message: string;
  };
}

/** 業務エラー・外部APIエラーを表現するカスタムエラークラス */
export class AiServiceError extends Error {
  public readonly code: AiErrorCode;
  public readonly statusCode: number;

  constructor(code: AiErrorCode, message: string, statusCode: number) {
    super(message);
    this.name = 'AiServiceError';
    this.code = code;
    this.statusCode = statusCode;
    Object.setPrototypeOf(this, AiServiceError.prototype);
  }
}

const SYSTEM_INSTRUCTION =
  'あなたはWebアプリケーションに組み込まれた親切で優秀なAIアシスタントです。ユーザーの質問に対して、正確かつ簡潔で分かりやすい日本語で回答してください。質問が他言語で書かれている場合は、その言語に合わせて回答してください。';

/**
 * 外部SDKやネットワークエラーからHTTPステータスコードを安全に抽出する
 */
function extractErrorStatus(error: unknown): number | undefined {
  if (error instanceof ApiError) {
    return error.status;
  }
  if (typeof error === 'object' && error !== null && 'status' in error) {
    const status = (error as { status?: unknown }).status;
    if (typeof status === 'number') {
      return status;
    }
  }
  return undefined;
}

/**
 * 質問テキストを受け取り、Gemini API を呼び出して回答を生成する
 */
export async function generateAnswer(
  question: string
): Promise<GenerateAnswerResult> {
  const apiKey = process.env.AI_API_KEY?.trim();
  if (!apiKey) {
    console.error('[AI Service Error] code=CONFIG_ERROR (AI_API_KEY is missing)');
    throw new AiServiceError(
      'CONFIG_ERROR',
      'AIサービスの設定に問題があります。管理者にお問い合わせください。',
      503
    );
  }

  const model = process.env.AI_MODEL?.trim() || AI_CONSTANTS.DEFAULT_MODEL;
  const ai = new GoogleGenAI({ apiKey });

  const abortController = new AbortController();
  const timeoutId = setTimeout(() => {
    abortController.abort();
  }, AI_CONSTANTS.REQUEST_TIMEOUT_MS);

  try {
    const response = await ai.models.generateContent({
      model,
      contents: question,
      config: {
        systemInstruction: SYSTEM_INSTRUCTION,
        abortSignal: abortController.signal,
      },
    });

    const answerText = response.text?.trim();
    if (!answerText) {
      console.error('[AI Service Error] code=EXTERNAL_API_ERROR (Empty response)');
      throw new AiServiceError(
        'EXTERNAL_API_ERROR',
        'AIサービスとの通信に失敗しました。時間をおいて再度お試しください。',
        502
      );
    }

    return {
      answer: answerText,
      model,
    };
  } catch (error: unknown) {
    if (error instanceof AiServiceError) {
      throw error;
    }

    // タイムアウト判定
    const isAbortError =
      abortController.signal.aborted ||
      (error instanceof Error && error.name === 'AbortError');

    if (isAbortError) {
      console.error('[AI Service Error] code=TIMEOUT');
      throw new AiServiceError(
        'TIMEOUT',
        'AIからの応答がタイムアウトしました。質問を短くするか、時間をおいて再度お試しください。',
        504
      );
    }

    const status = extractErrorStatus(error);
    const rawMessage = error instanceof Error ? error.message : '';

    // レート制限判定 (429 / RESOURCE_EXHAUSTED)
    if (
      status === 429 ||
      rawMessage.includes('429') ||
      rawMessage.includes('RESOURCE_EXHAUSTED')
    ) {
      console.error('[AI Service Error] code=RATE_LIMIT');
      throw new AiServiceError(
        'RATE_LIMIT',
        '現在リクエストが集中しています。しばらく時間を置いて再度お試しください。',
        429
      );
    }

    // APIキー認証・権限不備判定 (401 / 403 / API_KEY_INVALID)
    if (
      status === 401 ||
      status === 403 ||
      rawMessage.includes('API_KEY_INVALID') ||
      rawMessage.includes('PERMISSION_DENIED')
    ) {
      console.error(`[AI Service Error] code=CONFIG_ERROR status=${status ?? 'unknown'}`);
      throw new AiServiceError(
        'CONFIG_ERROR',
        'AIサービスの設定に問題があります。管理者にお問い合わせください。',
        503
      );
    }

    // その他の外部API・通信障害（機密漏洩防止のため生エラーオブジェクトは出力しない）
    console.error(
      `[AI Service Error] code=EXTERNAL_API_ERROR status=${status ?? 'unknown'}`
    );
    throw new AiServiceError(
      'EXTERNAL_API_ERROR',
      'AIサービスとの通信に失敗しました。時間をおいて再度お試しください。',
      502
    );
  } finally {
    clearTimeout(timeoutId);
  }
}

```

---

### Step 3-2: コントローラー層の実装 (`src/apps/ai/aiController.ts`)

画面レンダリング・入力バリデーション・レスポンス整形・予期せぬ例外の `next(err)` 委譲を実装する。

```typescript
import { Request, Response, NextFunction } from 'express';
import {
  AI_CONSTANTS,
  AiServiceError,
  AskErrorResponse,
  AskRequestBody,
  AskSuccessResponse,
  generateAnswer,
} from './aiService';

/**
 * 質問テキストのバリデーションを実行し、トリム済み文字列を返却する
 */
function validateQuestion(rawQuestion: unknown): string {
  if (typeof rawQuestion !== 'string') {
    throw new AiServiceError(
      'VALIDATION_ERROR',
      '質問は文字列形式で指定してください。',
      400
    );
  }

  const trimmed = rawQuestion.trim();

  if (trimmed.length < AI_CONSTANTS.MIN_QUESTION_LENGTH) {
    throw new AiServiceError(
      'VALIDATION_ERROR',
      '質問内容を入力してください。',
      400
    );
  }

  if (trimmed.length > AI_CONSTANTS.MAX_QUESTION_LENGTH) {
    throw new AiServiceError(
      'VALIDATION_ERROR',
      `質問は${AI_CONSTANTS.MAX_QUESTION_LENGTH.toLocaleString()}文字以内で入力してください。`,
      400
    );
  }

  return trimmed;
}

/**
 * GET /ai
 * AI質問画面をレンダリングする
 */
export function renderAiPage(
  _req: Request,
  res: Response,
  next: NextFunction
): void {
  try {
    res.render('ai/index', {
      title: 'AI質問アシスタント',
      maxLength: AI_CONSTANTS.MAX_QUESTION_LENGTH,
    });
  } catch (error) {
    next(error);
  }
}

/**
 * POST /ai/ask
 * 質問文を受け取り、Gemini API から取得した回答を JSON で返却する
 */
export async function postAskQuestion(
  req: Request<
    Record<string, never>,
    AskSuccessResponse | AskErrorResponse,
    AskRequestBody
  >,
  res: Response<AskSuccessResponse | AskErrorResponse>,
  next: NextFunction
): Promise<void> {
  try {
    const rawQuestion = req.body?.question;
    const validatedQuestion = validateQuestion(rawQuestion);

    const result = await generateAnswer(validatedQuestion);

    res.status(200).json({
      success: true,
      data: result,
    });
  } catch (error) {
    if (error instanceof AiServiceError) {
      res.status(error.statusCode).json({
        success: false,
        error: {
          code: error.code,
          message: error.message,
        },
      });
      return;
    }

    // 予期せぬランタイム例外は既存のグローバルエラーハンドラーへ委譲
    next(error);
  }
}

```

---

### Step 3-3: ルーター層の実装 (`src/apps/ai/aiRouter.ts`)

`/ai` 配下のルーティング定義を行う。既存の `index.ts` で `express.json()` が未設定の場合にも安全に動作するよう、ルーター単位でも `express.json()` を適用しておく。

```typescript
import express, { Router } from 'express';
import { renderAiPage, postAskQuestion } from './aiController';

const aiRouter = Router();

aiRouter.use(express.json());

aiRouter.get('/', renderAiPage);
aiRouter.post('/ask', postAskQuestion);

export default aiRouter;

```

---

## 4. フロントエンド・ビュー実装コード

### Step 4-1: 画面テンプレートの実装 (`views/ai/index.ejs`)

設計書のワイヤーフレームおよび要素ID仕様に準拠した EJS テンプレートを作成する。すべてのテンプレート変数は `<%= ... %>` でHTMLエスケープして展開する。

```html
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title><%= title %></title>
  <style>
    :root {
      --bg-color: #f8fafc;
      --card-bg: #ffffff;
      --text-main: #1e293b;
      --text-muted: #64748b;
      --border-color: #cbd5e1;
      --primary: #2563eb;
      --primary-hover: #1d4ed8;
      --danger-bg: #fef2f2;
      --danger-border: #fca5a5;
      --danger-text: #b91c1c;
    }

    * {
      box-sizing: border-box;
    }

    body {
      margin: 0;
      padding: 24px 16px;
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Hiragino Sans", "Noto Sans JP", sans-serif;
      background-color: var(--bg-color);
      color: var(--text-main);
      line-height: 1.6;
    }

    .container {
      max-width: 800px;
      margin: 0 auto;
    }

    .header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-bottom: 24px;
      padding-bottom: 12px;
      border-bottom: 1px solid var(--border-color);
    }

    .header h1 {
      margin: 0;
      font-size: 1.5rem;
    }

    .back-link {
      color: var(--primary);
      text-decoration: none;
      font-size: 0.95rem;
      font-weight: 500;
    }

    .back-link:hover {
      text-decoration: underline;
    }

    .card {
      background: var(--card-bg);
      border: 1px solid var(--border-color);
      border-radius: 8px;
      padding: 20px;
      margin-bottom: 20px;
      box-shadow: 0 1px 2px rgba(0, 0, 0, 0.04);
    }

    .form-label {
      display: block;
      font-weight: 600;
      margin-bottom: 8px;
    }

    .question-textarea {
      width: 100%;
      padding: 12px;
      font-size: 1rem;
      font-family: inherit;
      border: 1px solid var(--border-color);
      border-radius: 6px;
      resize: vertical;
      line-height: 1.5;
    }

    .question-textarea:focus {
      outline: 2px solid var(--primary);
      border-color: transparent;
    }

    .question-textarea:disabled {
      background-color: #f1f5f9;
      cursor: not-allowed;
    }

    .form-footer {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-top: 12px;
      gap: 12px;
      flex-wrap: wrap;
    }

    .char-counter {
      font-size: 0.875rem;
      color: var(--text-muted);
    }

    .char-counter.is-limit {
      color: var(--danger-text);
      font-weight: 600;
    }

    .button-group {
      display: flex;
      gap: 10px;
    }

    .btn {
      padding: 10px 20px;
      font-size: 0.95rem;
      font-weight: 600;
      border-radius: 6px;
      cursor: pointer;
      border: 1px solid transparent;
      transition: background-color 0.15s ease;
    }

    .btn:disabled {
      opacity: 0.6;
      cursor: not-allowed;
    }

    .btn-primary {
      background-color: var(--primary);
      color: #ffffff;
    }

    .btn-primary:hover:not(:disabled) {
      background-color: var(--primary-hover);
    }

    .btn-secondary {
      background-color: #ffffff;
      color: var(--text-main);
      border-color: var(--border-color);
    }

    .btn-secondary:hover:not(:disabled) {
      background-color: #f1f5f9;
    }

    .loading-box {
      display: flex;
      align-items: center;
      gap: 10px;
      padding: 14px 16px;
      margin-bottom: 20px;
      background-color: #eff6ff;
      border: 1px solid #bfdbfe;
      border-radius: 6px;
      color: #1e40af;
      font-weight: 500;
    }

    .loading-box[hidden],
    .error-box[hidden] {
      display: none;
    }

    .spinner {
      width: 18px;
      height: 18px;
      border: 3px solid #93c5fd;
      border-top-color: #1d4ed8;
      border-radius: 50%;
      animation: spin 0.8s linear infinite;
    }

    @keyframes spin {
      to { transform: rotate(360deg); }
    }

    .error-box {
      padding: 14px 16px;
      margin-bottom: 20px;
      background-color: var(--danger-bg);
      border: 1px solid var(--danger-border);
      border-radius: 6px;
      color: var(--danger-text);
    }

    .error-box p {
      margin: 0;
      font-weight: 500;
    }

    .answer-title {
      margin: 0 0 12px 0;
      font-size: 1.1rem;
      font-weight: 600;
    }

    .answer-content {
      min-height: 120px;
      padding: 14px;
      background-color: #f8fafc;
      border: 1px solid #e2e8f0;
      border-radius: 6px;
      white-space: pre-wrap;
      word-break: break-word;
    }

    .answer-content.is-placeholder {
      color: var(--text-muted);
    }
  </style>
</head>
<body>
  <div class="container">
    <header class="header">
      <h1><%= title %></h1>
      <a href="/" class="back-link">&larr; トップ・TODO一覧へ戻る</a>
    </header>

    <section class="card">
      <form id="ai-question-form" novalidate>
        <label for="question-input" class="form-label">質問内容</label>
        <textarea
          id="question-input"
          class="question-textarea"
          rows="4"
          maxlength="<%= maxLength %>"
          placeholder="例: タスクの優先順位をつけるコツを教えてください"
          required
        ></textarea>

        <div class="form-footer">
          <span id="char-counter-wrapper" class="char-counter">
            文字数: <span id="char-count">0</span> / <span id="max-length"><%= maxLength %></span>
          </span>

          <div class="button-group">
            <button type="button" id="clear-btn" class="btn btn-secondary">クリア</button>
            <button type="submit" id="submit-btn" class="btn btn-primary">送信する</button>
          </div>
        </div>
      </form>
    </section>

    <div id="loading-indicator" class="loading-box" hidden aria-live="polite">
      <div class="spinner" aria-hidden="true"></div>
      <span>AIが回答を生成しています...</span>
    </div>

    <div id="error-banner" class="error-box" role="alert" hidden>
      <p id="error-message"></p>
    </div>

    <section id="answer-section" class="card">
      <h2 class="answer-title">AIの回答</h2>
      <div id="answer-output" class="answer-content is-placeholder">質問を入力して「送信する」を押すと、ここに回答が表示されます。</div>
    </section>
  </div>

  <script src="/ai/ai.js" defer></script>
</body>
</html>

```

> ※ 戻りリンクの遷移先（`href="/"`）は、既存アプリのルート構成（例: `/todos` の場合は `href="/todos"`）に合わせて調整すること。

---

### Step 4-2: フロントエンド制御スクリプトの実装 (`public/ai/ai.js`)

文字数カウント・二重送信防止・事前バリデーション・Fetch API による非同期通信・`textContent` を用いた安全な DOM 描画を実装する。

```javascript
document.addEventListener('DOMContentLoaded', () => {
  const form = document.getElementById('ai-question-form');
  const questionInput = document.getElementById('question-input');
  const charCount = document.getElementById('char-count');
  const charCounterWrapper = document.getElementById('char-counter-wrapper');
  const maxLengthEl = document.getElementById('max-length');
  const clearBtn = document.getElementById('clear-btn');
  const submitBtn = document.getElementById('submit-btn');
  const loadingIndicator = document.getElementById('loading-indicator');
  const errorBanner = document.getElementById('error-banner');
  const errorMessage = document.getElementById('error-message');
  const answerOutput = document.getElementById('answer-output');

  if (
    !form ||
    !questionInput ||
    !charCount ||
    !clearBtn ||
    !submitBtn ||
    !loadingIndicator ||
    !errorBanner ||
    !errorMessage ||
    !answerOutput
  ) {
    return;
  }

  const MAX_LENGTH = Number(maxLengthEl?.textContent) || 1000;
  const INITIAL_PLACEHOLDER =
    '質問を入力して「送信する」を押すと、ここに回答が表示されます。';

  let isSubmitting = false;

  /**
   * 送信中・待機中のUI状態を一括切り替え
   */
  function setLoadingState(loading) {
    isSubmitting = loading;
    submitBtn.disabled = loading;
    clearBtn.disabled = loading;
    questionInput.disabled = loading;
    submitBtn.textContent = loading ? '送信中...' : '送信する';
    loadingIndicator.hidden = !loading;
  }

  /**
   * エラーメッセージを表示する（XSS防止のため必ず textContent を使用）
   */
  function showError(message) {
    errorMessage.textContent = message;
    errorBanner.hidden = false;
  }

  /**
   * エラー表示を隠す
   */
  function hideError() {
    errorMessage.textContent = '';
    errorBanner.hidden = true;
  }

  /**
   * 文字数カウンターの更新
   */
  function updateCharCounter() {
    const currentLength = questionInput.value.length;
    charCount.textContent = String(currentLength);

    if (charCounterWrapper) {
      if (currentLength >= MAX_LENGTH) {
        charCounterWrapper.classList.add('is-limit');
      } else {
        charCounterWrapper.classList.remove('is-limit');
      }
    }
  }

  // 入力時のリアルタイム文字数更新
  questionInput.addEventListener('input', updateCharCounter);

  // クリアボタン押下時のリセット処理
  clearBtn.addEventListener('click', () => {
    if (isSubmitting) {
      return;
    }
    questionInput.value = '';
    updateCharCounter();
    hideError();
    answerOutput.textContent = INITIAL_PLACEHOLDER;
    answerOutput.classList.add('is-placeholder');
    questionInput.focus();
  });

  // フォーム送信時の非同期処理
  form.addEventListener('submit', async (event) => {
    event.preventDefault();

    if (isSubmitting) {
      return;
    }

    const trimmed = questionInput.value.trim();

    // フロントエンド事前バリデーション
    if (trimmed.length === 0) {
      showError('質問内容を入力してください。');
      questionInput.focus();
      return;
    }

    if (trimmed.length > MAX_LENGTH) {
      showError(`質問は${MAX_LENGTH.toLocaleString()}文字以内で入力してください。`);
      questionInput.focus();
      return;
    }

    hideError();
    setLoadingState(true);

    try {
      const response = await fetch('/ai/ask', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({ question: trimmed }),
      });

      let result = null;
      try {
        result = await response.json();
      } catch (_parseError) {
        result = null;
      }

      if (response.ok && result && result.success && result.data) {
        answerOutput.textContent = result.data.answer;
        answerOutput.classList.remove('is-placeholder');
      } else {
        const message =
          result?.error?.message ||
          'エラーが発生しました。時間をおいて再度お試しください。';
        showError(message);
      }
    } catch (_networkError) {
      showError('ネットワークエラーが発生しました。接続状況を確認してください。');
    } finally {
      setLoadingState(false);
    }
  });
});

```

---

## 5. 既存アプリケーションへの統合手順

### Step 5-1: エントリーポイントへのルーター登録 (`index.ts`)

既存の `index.ts` に `aiRouter` をインポートし、**404ハンドラーおよびエラーハンドラーよりも前**にマウントする。また、静的ファイル配信（`express.static('public')`）が有効であることを確認する。

```typescript
// --- 既存のインポート群 ---
import express from 'express';
import dotenv from 'dotenv';
// import todoRouter from './src/apps/todo/todoRouter';

// 【追加】AI質問ルーターのインポート
import aiRouter from './src/apps/ai/aiRouter';

dotenv.config();

const app = express();

// --- 既存の共通ミドルウェア ---
app.set('view engine', 'ejs');
app.use(express.static('public'));
app.use(express.urlencoded({ extended: true }));
app.use(express.json());

// --- 既存のルーター登録 ---
// app.use('/todos', todoRouter);

// 【追加】AI質問機能のルーター登録（必ず 404 / エラーハンドラーより前に配置）
app.use('/ai', aiRouter);

// --- 既存の 404 ハンドラー・グローバルエラーハンドラー（変更しない） ---

```

### Step 5-2: 既存画面からの遷移リンク追加

既存の TOP 画面または TODO 一覧画面の EJS テンプレート（例: `views/index.ejs` や `views/todo/index.ejs`）のナビゲーション領域に、AI質問画面へのリンクを追加する。

```html
<a href="/ai">AIに質問する</a>

```

---

## 6. ビルド・動作確認手順（スモークテスト）

実装完了後、テストフェーズへ進む前に以下のコマンドでコンパイルエラーがないことと基本動作を確認する。

1. **TypeScript 型チェック・ビルド確認**
```bash
npx tsc --noEmit

```


* エラーが 0 件であることを確認する。


2. **サーバー起動確認**
```bash
npm run dev

```


* 起動時に例外が発生せず、既存ポートで待ち受け状態になることを確認する。


3. **画面・API 疎通確認**
* ブラウザで `http://localhost:<PORT>/ai` を開き、画面および文字数カウンターが動作することを確認する。
* 質問を入力して送信し、ローディング表示後に `gemini-3.5-flash-lite` からの回答が描画されることを確認する。
* 既存の TODO 画面（`/` または `/todos`）および存在しないパス（`/not-found-test`）の 404 応答が従来通り動作することを確認する。