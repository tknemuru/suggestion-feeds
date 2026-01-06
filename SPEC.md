# AI Autonomous Digest Agent - Product Specification

## 1. Project Overview

自然言語で指定されたトピック（例: "Claude 3.5の効果的な使い方"）に基づき、Web上から定期的に情報を収集・要約し、ユーザーにインサイトを提供する自律型エージェントシステム。
ユーザーからのフィードバック（Good/Bad）を学習し、次回の情報収集・要約の精度と「好み」を継続的に改善する(Human-in-the-loop)。

## 2. Tech Stack & Constraints

このプロジェクトは以下の最新スタックに厳格に従うこと。

- **Framework:** Next.js 15 (App Router)
- **Language:** TypeScript
- **Styling:** Tailwind CSS, shadcn/ui
- **Database / ORM:** Supabase (PostgreSQL), Prisma
- **Vector Search:** pgvector (via Prisma `Unsupported` type or raw SQL for now)
- **Job Scheduling:** Trigger.dev v3 (Must use v3 SDK)
- **AI Orchestration:** Vercel AI SDK (Core)
- **Search Provider:** Tavily API
- **LLM:** Anthropic Claude 3.5 Sonnet
- **Email Service:** Resend
- **Email Template:** React Email

## 3. Data Model (Prisma Schema)

`prisma/schema.prisma` は以下の構造を正とする。

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
  query       String   // e.g. "AI agent trends"
  sources     String[] // e.g. ["twitter", "web"]
  schedule    String   // Cron syntax
  isActive    Boolean  @default(true)
  maxItems    Int      @default(3)
  
  user        User     @relation(fields: [userId], references: [id])
  results     Result[]
  
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
}

model Result {
  id          String   @id @default(uuid())
  jobId       String
  summary     String
  rawUrl      String
  publishedAt DateTime
  
  job         Job      @relation(fields: [jobId], references: [id])
  feedback    Feedback?

  createdAt   DateTime @default(now())
}

model Feedback {
  id        String   @id @default(uuid())
  userId    String
  resultId  String   @unique
  isPositive Boolean
  comment   String?
  
  // Note: Handle vector embeddings via raw query or specific pgvector setup
  // embedding vector(1536)

  user      User     @relation(fields: [userId], references: [id])
  result    Result   @relation(fields: [resultId], references: [id])
}

申し訳ありません。トークン制限により途切れてしまったようです。
安全確実にコピー＆ペーストできるよう、**2つのコードブロック（Part 1 / Part 2）** に分割して出力します。

この2つを結合して、`SPEC.md` として保存してください。内容は先ほどの「メール通知仕様（Resend）」も統合した完全版です。

### SPEC.md - Part 1/2

（プロジェクト概要、技術スタック、データモデルまで）

```markdown
# AI Autonomous Digest Agent - Product Specification

## 1. Project Overview
自然言語で指定されたトピック（例: "Claude 3.5の効果的な使い方"）に基づき、Web上から定期的に情報を収集・要約し、ユーザーにインサイトを提供する自律型エージェントシステム。
ユーザーからのフィードバック（Good/Bad）を学習し、次回の情報収集・要約の精度と「好み」を継続的に改善する(Human-in-the-loop)。

## 2. Tech Stack & Constraints
このプロジェクトは以下の最新スタックに厳格に従うこと。

- **Framework:** Next.js 15 (App Router)
- **Language:** TypeScript
- **Styling:** Tailwind CSS, shadcn/ui
- **Database / ORM:** Supabase (PostgreSQL), Prisma
- **Vector Search:** pgvector (via Prisma `Unsupported` type or raw SQL for now)
- **Job Scheduling:** Trigger.dev v3 (Must use v3 SDK)
- **AI Orchestration:** Vercel AI SDK (Core)
- **Search Provider:** Tavily API
- **LLM:** Anthropic Claude 3.5 Sonnet
- **Email Service:** Resend
- **Email Template:** React Email

## 3. Data Model (Prisma Schema)
`prisma/schema.prisma` は以下の構造を正とする。

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
  query       String   // e.g. "AI agent trends"
  sources     String[] // e.g. ["twitter", "web"]
  schedule    String   // Cron syntax
  isActive    Boolean  @default(true)
  maxItems    Int      @default(3)
  
  user        User     @relation(fields: [userId], references: [id])
  results     Result[]
  
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
}

model Result {
  id          String   @id @default(uuid())
  jobId       String
  summary     String
  rawUrl      String
  publishedAt DateTime
  
  job         Job      @relation(fields: [jobId], references: [id])
  feedback    Feedback?

  createdAt   DateTime @default(now())
}

model Feedback {
  id        String   @id @default(uuid())
  userId    String
  resultId  String   @unique
  isPositive Boolean
  comment   String?
  
  // Note: Handle vector embeddings via raw query or specific pgvector setup
  // embedding vector(1536)

  user      User     @relation(fields: [userId], references: [id])
  result    Result   @relation(fields: [resultId], references: [id])
}

```

---

### SPEC.md - Part 2/2

（アーキテクチャ詳細、実装ステップ、コーディング規約）

```markdown
## 4. Core Architecture Components

### A. Agent Logic (`src/lib/agent/`)
**責務:** 情報収集と要約生成の純粋なロジック。
1. **Context Retrieval:** `Job` に紐づく過去の `Feedback` を取得し、ユーザーの「好き(Positive)」「嫌い(Negative)」の傾向リストを作成する。
2. **Search:** `Tavily API` を使用する。
   - `Job.sources` に "twitter" が含まれる場合は `include_domains` で `x.com`, `twitter.com` を指定。
   - 一般Webの場合は除外ドメインなし。
3. **Synthesis (LLM):** Vercel AI SDK を使用し、Claude 3.5 Sonnet に以下を指示する。
   - 検索結果を分析する。
   - 「嫌い」なトピックを除外し、「好き」な傾向に近い情報を優先する。
   - 単なる事実の羅列ではなく、インサイト（見解）を含めた要約を作成する。

### B. Scheduling System (`src/trigger/`)
**責務:** 定期実行の管理と長時間プロセスの保証。
- **Trigger.dev v3** を使用。
- Cronスケジュールで起動するタスク (`digestTask`) を定義。
- タスク内で `src/lib/agent` のロジックを呼び出し、DBへの保存を行う。
- **Notification:** `Resend` SDKを使用し、ユーザーのEmailアドレスへ通知を送る。メール本文は `src/emails/` 以下の React Email コンポーネントを使用し、HTMLメールとしてレンダリングすること。

### C. Frontend Dashboard (`src/app/`)
**責務:** 設定管理とフィードバック収集。
- **Dashboard:** ユーザーがJobを作成・編集・削除できるCRUD画面。
- **Feed:** 生成された `Result` をタイムライン形式で表示。
- **Feedback UI:** 各Resultに対して「👍 / 👎」ボタンを配置。押下時に即座に `Feedback` テーブルへ保存するServer Actionを叩く。

## 5. Implementation Steps for Cursor

### Phase 1: Foundation
1. Setup Next.js 15, Prisma, and Supabase connection.
2. Apply the Prisma schema and generate client.
3. Install Trigger.dev v3 SDK and configure the project.

### Phase 2: The Agent
1. Implement `src/lib/agent/digest-generator.ts`.
2. Integrate Tavily API for search.
3. Integrate Vercel AI SDK for summarization with "Context/Preference" injection.

### Phase 3: The Scheduler
1. Create `src/trigger/digest-job.ts`.
2. Connect the scheduler to fetch active jobs from Prisma and execute the Agent logic.

### Phase 3.5: Email Notification
1. Install `resend` and `@react-email/components`.
2. Create a standardized email template at `src/emails/DigestTemplate.tsx`.
3. Integrate `resend.emails.send` within the `digestTask` in `src/trigger/digest-job.ts`.

### Phase 4: The UI
1. Build `JobForm` component using `react-hook-form` and `zod`.
2. Build `ResultFeed` component with optimistic UI for Feedback actions.

## 6. Coding Standards
- **Server Actions:** Use Server Actions for all data mutations.
- **Type Safety:** No `any`. Zod schemas for all inputs.
- **Error Handling:** Use `try/catch` in all async operations and log errors clearly (especially inside Trigger.dev tasks).
- **Environment:** Access API keys strictly via `process.env`.

```
