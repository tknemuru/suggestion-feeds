# Project: AI Autonomous Digest Agent

## Tech Stack & Architecture

- **Frontend:** Next.js 15 (App Router), Tailwind CSS, shadcn/ui
- **Backend/DB:** Supabase (PostgreSQL + pgvector), Prisma
- **Job Queue:** Trigger.dev v3
- **AI/LLM:** Vercel AI SDK (Core), Anthropic Claude 3.5 Sonnet
- **Email:** Resend, React Email

## Context Awareness Rules

- **ALWAYS** read `@SPEC.md` first to understand the architectural boundaries.
  - (常に `@SPEC.md` を読み、アーキテクチャの境界線を理解する)
- **Directory Roles:**
  - `src/lib/agent/`: Pure domain logic for search & summarization.
  - `src/trigger/`: Long-running background jobs (reliable execution).
  - `src/app/`: UI & Dashboard (Server Actions for mutations).
  - `src/emails/`: React Email templates.

## Coding Standards & UX Guidelines

- **Server Actions First:** Use Server Actions for all data mutations from the UI.
  - (UIからのデータ更新はすべてServer Actionsを使用する)
- **Type Safety:** Use Zod for all input validation (Forms & API responses). No `any`.
  - (入力検証には必ずZodを使用する。`any` は禁止)
- **Error Handling:**
  - In `src/trigger/`: Log explicitly. Failures usually mean retries, so ensure idempotency.
  - In `src/app/`: Show toast notifications for user feedback (Success/Error).
  - (Trigger.dev内では明示的にログを残し、再試行可能性（冪等性）を担保する。UIではToastでフィードバックを行う)

## Code Quality & Testing Guidelines

- Separate pure logic (Agent) from infrastructure (Trigger.dev/Next.js) where possible.
  - (可能な限り、純粋なロジック（Agent）とインフラ（Trigger.dev/Next.js）を分離する)
- **Pragmatic Testing Policy:**
  - Focus tests on the `src/lib/agent` logic (Input -> Output) to ensure AI prompt handling works.
  - Avoid mocking the entire database for simple CRUD; rely on manual verification for UI.
  - (AIプロンプト処理が機能することを確認するため、`src/lib/agent` のロジックテストに集中する。単純なCRUDのDBモックは避け、UIは手動検証に頼る)

## AI Implementation Workflow

### Planning Requirement

- For any non-trivial changes, **always create a plan first** and align with the user before implementation.
  - (軽微な変更以外は、必ず最初に計画を作成し、実装前にユーザーとすり合わせを行う)
- **Reference `SPEC.md`:** When planning, explicitly mention which phase or component of the SPEC is being addressed.
  - (計画時は `SPEC.md` のどのフェーズやコンポーネントに取り組むかを明示する)

### Test Execution Policy

- **Default to Implementation:** By default, focus on implementation only without running tests.
  - (デフォルトでは、テスト実行せず実装のみに集中する)
- **On Demand Testing:** Run tests only when the user explicitly requests it with phrases like "run tests" or "check logic."
  - (ユーザーが「テストを実行して」「ロジックを確認して」などと明示的に指示した場合のみテストを実行する)
- **Reasoning:** This keeps the "Vibe Coding" flow fast. We validate manually first, then solidify with tests if needed.
  - (これにより「バイブコーディング」のフローを高速に保つ。まずは手動で検証し、必要であれば後からテストで固める)
