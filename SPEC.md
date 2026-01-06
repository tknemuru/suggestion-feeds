# AI Autonomous Digest Agent - Product Specification

## 1. Project Overview

自然言語で指定された興味のあるトピックについて、Web上から情報を定期的に収集・分析し、ユーザーにとって価値のある形（デフォルメ・見解付き）で通知する自律型エージェントシステム。
ユーザーからのフィードバック（Good/Bad）を学習ループに組み込み、情報の選別精度と要約のスタイルを継続的に「自分好み」にパーソナライズする。

### Key Requirements (要件定義)

1. **Interest-based Collection:** ユーザーは自然言語で収集したいトピックを指定できる（例：「Claudeの効率的な使い方」）。
2. **Flexible Scheduling:** 「定期」の設定はJobごとに個別に行える。
   - 高頻度: 「とても関心がある」→ 5分おき、上限10件。
   - 低頻度: 「少し興味がある」→ 1日1回、上限3件。
   - 設定変更: 頻度や件数の変更、Jobの削除（停止）が随時可能。
3. **Source Control:** 収集元を「X（Twitter）のみ」「全Webサイト」など指定可能。
4. **Insight & Deformation:** 単なる情報の羅列ではなく、情報をわかりやすくデフォルメ（噛み砕き）し、要点と「AIとしての見解」を付与して通知する。
5. **Feedback Loop:** 通知された情報に対し「よかった（Good）」「悪かった（Bad）」のフィードバックが可能。
6. **Adaptive Personalization:** フィードバックに基づき、次回の収集・要約時に「好みの情報はより多く」「嫌いな情報は除外」するよう自動調整される。

## 2. Tech Stack & Constraints

このプロジェクトは以下の最新スタックに厳格に従うこと。

- **Framework:** Next.js 15 (App Router)
- **Language:** TypeScript
- **Styling:** Tailwind CSS, shadcn/ui
- **Database / ORM:** Supabase (PostgreSQL), Prisma
- **Vector Search:** pgvector (via Prisma `Unsupported` type or raw SQL)
- **Job Scheduling:** Trigger.dev v3 (Must use v3 SDK)
- **AI Orchestration:** Vercel AI SDK (Core)
- **Search Provider:** Tavily API
- **LLM:** Anthropic Claude 3.5 Sonnet
- **Email Service:** Resend
- **Email Template:** React Email

## 3. Data Model (Prisma Schema)

`prisma/schema.prisma` は以下の構造を正とする。Jobモデルには要件を満たすためのスケジュールと件数制限を含める。

```prisma
model User {
  id        String     @id @default(uuid())
  email     String     @unique
  jobs      Job[]
  feedbacks Feedback[]
}

model Job {
  id          String   @id @default(uuid())
  userId      String
  query       String   // e.g. "Claude AI effective usage"
  sources     String[] // e.g. ["twitter", "web"] ("web" implies all sources excluding twitter if needed, or combined)
  
  // Scheduling & Limits
  schedule    String   // Cron syntax (e.g. "*/5 * * * *" or "0 9 * * *")
  isActive    Boolean  @default(true) // User can pause/resume jobs
  maxItems    Int      @default(3)    // Max items per digest (e.g. 3 or 10)
  
  user        User     @relation(fields: [userId], references: [id])
  results     Result[]
  
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
}

model Result {
  id          String   @id @default(uuid())
  jobId       String
  summary     String   // Deformed and insight-rich summary
  rawUrl      String   // Primary source URL
  publishedAt DateTime
  
  job         Job      @relation(fields: [jobId], references: [id])
  feedback    Feedback?

  createdAt   DateTime @default(now())
}

model Feedback {
  id        String   @id @default(uuid())
  userId    String
  resultId  String   @unique
  isPositive Boolean // Good (True) or Bad (False)
  comment   String?  // Optional detailed feedback
  
  // Embedding for future personalization (summary text + feedback context)
  // embedding vector(1536)

  user      User     @relation(fields: [userId], references: [id])
  result    Result   @relation(fields: [resultId], references: [id])
}
```

## 4. Core Architecture Components

### A. Agent Logic (`src/lib/agent/`)

**責務:** 情報収集と「わかりやすい要約・見解」の生成。

1. **Context Retrieval (Personalization):** - Job実行時、過去の `Feedback` を参照する。
   - Positiveなフィードバックが付いたトピック/傾向を「強化要素」、Negativeなものを「除外要素」として抽出する。
2. **Search Strategy (Tavily):** - `Job.query` を検索クエリとする。
   - `Job.sources` に基づきドメインフィルタリングを行う（例: Xのみ、Web全体）。
   - `Job.maxItems` の2倍程度の候補を取得し、フィルタリングの余地を持たせる。
3. **Synthesis & Deformation (LLM):** - Claude 3.5 Sonnet に以下のシステムプロンプトを与える：
     - **Role:** 優秀なアナリスト兼エディター。
     - **Task:** 検索結果を分析し、ユーザーの「好き/嫌い」の傾向に合わせて情報を選別する。
     - **Output Requirement:** - 専門用語を避け、初心者にもわかるように**デフォルメ（噛み砕く）**すること。
       - 単なる要約にとどまらず、**「なぜこれが重要か」「どう活用できるか」という独自の見解（Insight）**を含めること。
       - 件数は `Job.maxItems` 以内に収めること。

### B. Scheduling System (`src/trigger/`)

**責務:** 個別の定期設定に基づく確実な実行と通知。

- **Trigger.dev v3** を使用。
- **Master Cron Job:** - 最短粒度（例: 5分ごと `*/5 * * * *`）で起動するタスク (`digestTask`) を定義。
  - タスク内で `Job` テーブルをスキャンし、`schedule` (Cron) と `isActive` をチェックして「今実行すべきJob」のみをフィルタリングして実行する。
- **Notification:** - `Resend` SDKを使用。
  - `src/emails/` の React Email テンプレートを使用し、デフォルメされた要約と見解を美しくフォーマットして送信する。

### C. Frontend Dashboard (`src/app/`)

**責務:** 柔軟な設定管理とフィードバックループのUI。

- **Job Management:** - 新規作成・編集フォームにて以下の設定を提供：
  - **Query:** 自然言語入力。
  - **Sources:** チェックボックス（X, Web, etc）。
  - **Frequency:** プリセット選択肢を提供し、裏でCron式に変換して保存。
    - "High Interest (Every 5 mins)" -> `*/5 * * * *`
    - "Daily Digest (Every morning)" -> `0 9 * * *`
    - "Weekly" -> etc.
  - **Max Items:** 数値入力（例: 3, 10）。
  - Jobの削除または一時停止（`isActive: false`）が可能。
- **Feedback UI:** - メール内のリンク、またはダッシュボードのタイムラインから、各Resultに対して「👍 / 👎」をワンタップで送信可能。

## 5. Implementation Steps for Cursor

### Phase 1: Foundation

1. Setup Next.js 15, Prisma, and Supabase connection.
2. Apply the updated Prisma schema (including `schedule`, `maxItems`).
3. Install Trigger.dev v3, Tavily, AI SDK, Resend.

### Phase 2: The Agent (With Deformation Logic)

1. Implement `src/lib/agent/digest-generator.ts`.
2. **Crucial:** Design the LLM prompt to strictly follow the "Deformation & Insight" requirement defined in Section 4-A.
3. Implement Source filtering logic.

### Phase 3: The Scheduler (Flexible Cron)

1. Create `src/trigger/digest-job.ts`.
2. Implement the logic to check which jobs match their Cron schedule at the current runtime (Polling pattern).
3. Integrate `Resend` for email delivery.

### Phase 4: The UI (Job Management & Feedback)

1. Build `JobForm` with "Frequency" presets mapping to Cron strings.
2. Build `ResultFeed` with Feedback buttons (Server Actions).

## 6. Coding Standards

- **Server Actions:** Use Server Actions for all mutations.
- **Type Safety:** Zod schemas for all inputs (especially Cron validation).
- **Environment:** API keys via `process.env`.
- **Maintainability:** Keep Agent logic pure and testable.
