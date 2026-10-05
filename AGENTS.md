# AGENTS.md — PrepNivo Parent App (OpenAI Codex)

Put this file in the repo root, with `SPEC.md` and `assets/` next to it. Codex reads AGENTS.md automatically.

## Source of truth
- `SPEC.md` is authoritative for every product rule, screen, design token and piece of seed data. Read all of it before starting any task.
- If SPEC.md doesn't cover something, pick the simplest option consistent with prepnivo.com. Log it in `DECISIONS.md` and continue.

## Task protocol
- Run **one phase per Codex task**. Prompt: "Implement Phase N from AGENTS.md. Follow SPEC.md."
- Each task produces **one PR**, titled `Phase N: <name>`. The PR description must include:
  - the routes added
  - a screenshot list (light and dark)
  - the checklist items from SPEC §9 that this phase completes
  - any new entries in DECISIONS.md
- Don't start the next phase in the same task.
- Before finishing a task, these must all pass:
  - `pnpm install`
  - `pnpm lint`
  - `pnpm typecheck`
  - `pnpm test`
  - `pnpm check:contrast`
  - `pnpm test:e2e` (from Phase 2 onward)

  If any of them fails, fix it. Never skip or disable a check.
- The Codex sandbox may have no network access. Do not call real Resend or Stripe. OTP uses the console fallback when `RESEND_API_KEY` is unset.

## Environment
- Node 20+, pnpm.
- Copy `.env.example` to `.env` and set `DATABASE_URL="file:./dev.db"` and `SESSION_SECRET` (32+ characters). `RESEND_API_KEY` is optional.
- Setup: `pnpm i && pnpm db:push && pnpm seed && pnpm dev`.

## Hard rules (never violate)
- This is the parent side only. Children get **no AI** anywhere.
- No PIN or picture sign-in for parents. Parents need email OTP on every login.
- "Topics improving" must not appear anywhere. JEE and "Build a custom test in Settings" are gone.
- No text smaller than 14px. No hard-coded colours; use tokens only.
- Nivo always renders on `--nivo-disc` and is never recoloured.
- Scores are shown as `correct / total`. Each metric comes from a single selector.
- The backend is limited to auth, sessions and OTPs. Everything else uses mock and seed data.

## Phases

### Phase 0 — Project setup, design system, assets
- Scaffold Next.js (App Router) + TypeScript + Tailwind + shadcn/ui + Prisma (SQLite) + Zustand + Recharts + next-themes. pnpm.
- Implement every colour token from SPEC §6 as CSS variables for light (`:root`) and dark (`.dark`). Map them in the Tailwind config. Never hard-code hex values in components.
- Implement the type scale from SPEC §6. Add a lint rule or a test that fails on any `text-[<14px]` or `text-xs` usage.
- Copy `assets/nivo.svg` and `assets/prepnivo-logo.svg` into `public/`.
  - Build `<Nivo size />`, which always renders on a `--nivo-disc` circle.
  - Build `<Logo />`, which uses CSS-variable colours.
- Write `scripts/contrast-check.ts`. It checks every text/background token pair in both themes against WCAG thresholds (body text ≥4.5:1 with a target of 7:1, muted text ≥4.5:1, UI ≥3:1) and exits non-zero on failure. Add it as `pnpm check:contrast`.
- Create typed seed data in `/data` exactly as in SPEC §8 (parent, Aarav, Anaya, curricula, sessions, weak topics, tests, buddy request, invoices).
- Done when: `pnpm dev` runs, `pnpm check:contrast` passes, and a `/styleguide` page shows the tokens, the type scale, Nivo and the logo in both themes.

### Phase 1 — Parent auth + real email 2FA
- Prisma models: Parent, Session, OtpCode (hashed code, expiresAt, attempts, purpose).
- Build `/welcome`, `/signup`, `/login`, `/verify`, `/forgot` and `/reset` per SPEC §5.1.
- OTP rules:
  - 6 digits, expires in 10 minutes, 5 attempts, then locked for 15 minutes.
  - Resend has a 60-second cooldown.
  - Rate-limit by email and IP.
  - OTP is required at sign-up and at every login.
- Email: send through Resend when `RESEND_API_KEY` is set. Otherwise, log the code to the console and show a "Dev mode: code in terminal" note.
- Session: signed httpOnly cookie. Middleware protects every app route.
- There is no PIN or picture login for parents.
- Vitest tests cover OTP generation, hashing, expiry, attempts and the cooldown.
- Done when: sign-up, then OTP, then logged in works, and logout followed by login requires OTP again.

### Phase 2 — Onboarding + Stripe mock checkout
- Build the 5-step stepper per SPEC §5.2: child details, class content breakdown, course details + price, child login setup, review.
- The class breakdown lists every subject and its chapters, from `/data/curricula`. It is read-only.
- Prices come from `config/pricing.ts`: INR ₹999/yr and ₹99/mo; EUR placeholders marked `// TODO confirm`. Yearly is the default and shows its saving.
- Child login setup:
  - Grades 1–4: choose at least 3 animals in order.
  - Grades 5–10: username + 4-digit PIN.
  - Add a printable login card that includes Nivo.
- The cart groups children by currency. A mixed cart shows 2 checkout blocks.
- Build `/checkout/redirect` (auto-continues after 1.5s), then `/checkout/stripe-mock` with a "Test mode" banner, pre-filled 4242 card, and Pay / Cancel / Simulate failure buttons. Then build the success, cancelled and failed pages per SPEC §5.3.
- Vitest tests cover cart grouping and pricing.
- Done when: adding Aarav (INR) and Anaya (EUR) produces two checkouts, and the success, cancel and fail paths all behave per spec.

### Phase 3 — App shell + Overview + drill-down
- Sidebar:
  - Logo, then the child switcher (avatar, name, class).
  - Nav: Overview · Weak topics · Tests · New questions · Settings.
  - The Settings nav item shows the buddy-request badge.
- Top bar:
  - Breadcrumb in the format `Child › Section › Sub`.
  - Bell with badge and a notifications panel.
  - Theme toggle and avatar menu.
  - The "Create more questions" button appears **only** on Tests and New questions.
- Overview per SPEC §5.4:
  - 4 cards (Mastery, Accuracy, Coverage, Subject drill-down card).
  - A 14-day strip; clicking a day updates the session details panel.
  - Subjects grid.
  - Alerts row (buddy request, grace period, test due).
- Drill-down routes: subject (detailed overview with an 8-week trend), then chapter (topics table), then topic (trend, attempts, recent wrong answers).
- Each metric comes from a single selector, so the numbers are identical everywhere.
- Scores always display as `correct / total`, with points shown separately.
- "Topics improving" must not exist anywhere.
- Done when: switching the child updates every view, and the drill-down plus breadcrumbs work at all levels.

### Phase 4 — Weak topics
- Build the list sorted by severity, with subject and chapter filters.
- Build the "Worth a look" card using Aarav's fractions example: the question, the wrong answer chosen ×4, the correct answer, and the misconception.
- Actions:
  - "Set a practice test on this" opens the test builder pre-filled.
  - "Generate similar questions" opens the AI generator pre-filled.
- Build the topic detail page.
- Done when: both actions arrive pre-filled.

### Phase 5 — Tests
- List view with tabs for Assigned, In progress and Completed.
- Builder with 4 steps, ported from "Build my drill" (SPEC §5.6):
  1. Setup
  2. Topics tree. Locked chapters are selectable and tagged "Locked for [child]".
  3. Questions, from My content / Platform / AI generated, with drag-to-reorder and "+ Add more questions".
  4. Review, showing points at the drill rate, then "Assign test".
- Test detail page with results and "Retest weak topics".
- The toast after assigning reads "Test assigned".
- Done when: a test can be built from all three sources and assigned, and it then appears under Assigned and on Overview.

### Phase 6 — New questions + AI (the most work)
- Tabs:
  - **My content:** the existing bank ported, with filters, add, edit, delete and bulk actions.
  - **Platform content:** read-only, with "Add to test".
  - **AI generated:** approved questions, plus a badge counting the review queue.
- Manual add supports: multiple choice, true/false, short answer, fill in the blank and number answer. Each question has an explanation, difficulty and a syllabus tag.
- `/questions/import` accepts PDF, photo (with a crop preview) and pasted text. A simulated "Reading your material…" state leads to drafts with suggested placement, then the review queue.
- `/questions/generate`:
  - Fields: child, subject, chapter, topic, difficulty, count, types, and optionally "match this weak topic".
  - Output: Questions or Challenge. A parent-made challenge uses **platform challenge points**.
- Review queue: Approve / Edit / Discard, plus bulk approve. Nothing reaches the child without approval.
- Mock AI with canned, curriculum-correct output and a 1.5–3s delay. Put it behind `lib/ai/generate.ts` so a real API can replace it later.
- There must be no AI entry point on any child-facing screen.
- Done when: import, then generate, then review, then approve, then use in a test all works end to end.

### Phase 7 — Settings
- Sub-nav: Account · Security · Children · Buddy requests · Billing · Appearance · Feedback. There is no "Build a custom test" and no JEE/segment.
- **Security:** change password (requires OTP); "2FA always on"; active sessions.
- **Children:** edit, reset child login, remove (confirm by typing the name), add child.
- **Change class:** a 4-step flow. Plan transfers, renewal date unchanged, old progress archived and viewable.
- **Buddy requests:** Pending (Approve / Decline), Approved (Remove), Declined. These feed the badges and the Overview alert.
- **Billing:**
  - Per-child subscription cards with status (Active / Renewing / Grace period / Paused) and an auto-renew toggle.
  - Switch yearly/monthly, effective from the next renewal.
  - Cancel.
  - Payment methods per currency, with a mock "Update card" form.
  - Invoices table with a mock PDF download.
  - Anaya's grace-period banner (payment failed 1 Oct, keep access until 1 Nov).
  - Build the `/access-paused` screen for after the grace period.
- **Appearance:** Light / Dark / System, persisted.
- Done when: every Settings action changes state and is reflected on Overview and in the badges.

### Phase 8 — States, dark-mode QA, tests, docs
- Every list has a loading skeleton that resolves, an empty state that tells the parent what to do (with Nivo on the empty states), and an error state that says how to fix it.
- Dark-mode pass on every screen: charts use the dark palette, status colours come with an icon and a label, there is visible keyboard focus, and Nivo looks identical in both modes.
- `pnpm check:contrast` passes, and there is no text under 14px.
- Playwright tests:
  - sign-up → OTP → add 2 children → 2 checkouts → overview
  - create test
  - AI generate → review → approve
  - change class
  - grace-period banner → update card
- README with one-command setup, plus `.env.example`.
- Write `DECISIONS.md` listing any assumption you made that is not covered by SPEC.md.
- Done when: every box in the SPEC §9 checklist is ticked.
