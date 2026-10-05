# PrepNivo Parent App — Build Spec

Single source of truth for Claude Code (CLAUDE.md) and Codex (AGENTS.md). Both agents build the same thing from this file. If a task file and this spec disagree, this spec wins.

Last updated: 5 Oct 2026, from team meeting notes and Q&A with Aayushmaan.

---

## 1. What we are building

A **standalone, runnable desktop web app** for the **parent side** of PrepNivo, based on the live structure at prepnivo.com/dashboard. The child side is out of scope, apart from the child-login setup that parents do during onboarding.

| Area | Real or mock |
|---|---|
| Parent sign-up, login, **email 2FA (OTP)** | **Real.** Minimal backend: API routes, SQLite, email service. |
| Stripe checkout | **Frontend mock.** "Redirecting to Stripe…" then a Stripe-style mock checkout page, then success, cancel or fail. |
| Payment management | **Custom in-app screen** with mock data. Not the Stripe Customer Portal. |
| Dashboard, progress, tests, questions, AI generation | **Frontend with seeded realistic data.** AI generation is mocked with canned output and a simulated delay. |

Target: **desktop 1440×900** first. It must not break at 1280 wide. Tablet and mobile are not in scope.

---

## 2. Existing PrepNivo structure (as of 5 Oct 2026)

Keep this visual language: mint-green background, white cards, green primary, rounded cards, left sidebar, breadcrumb bar, avatar at top right.

- **Sidebar:** Klasse switcher. Learn: Dashboard, Drills, Challenges, My content. Track: Progress. Then Children, Settings.
- **Dashboard:** welcome banner with "Start a drill" and "Start a challenge"; 4 stat cards (Mastery, Accuracy, Questions, Day streak); Your drills; Subject mastery; Last mock exam.
- **Progress** (`/dashboard/analytics`): 5 cards (Overall mastery, Accuracy, Coverage, Questions attempted, Study streak). Below them, subject cards (mastery, accuracy, coverage, attempts, "View chapters"), then a chapter-level statistics table, then topic-level statistics. **This card style and drill-down are the reference for the parent Overview.**
- **Drills:** "Practice zone" with a "Build my drill" builder (Topics, Difficulty, Timing, Question count) and a recent drills table.
- **Challenges:** stat chips and a "Create test" button.
- **My content:** a Question bank tab (filters for Subject, Chapter, Topic and text search; Add question; Enrich; Delete; table) and a Personal AI tab (step 1: import sources such as topic notes, PDF or bank questions; step 2: draft, edit, generate variants, sync to bank).
- **Settings:** Plans & Billing, Account, Sign-in options (PIN and picture), Feedback.

### Live-site anomalies. Fix these in our build; do not copy them.

1. The Children nav item doesn't open a children page; it stays on Settings.
2. One account holds 9 grade subscriptions. This conflicts with the rule 1 kid = 1 class = 1 subscription.
3. PIN sign-in gives full account access with email + PIN, which bypasses 2FA. **PIN and picture sign-in are child-only.**
4. Stats disagree across pages: Mastery is 3% on Dashboard and 4% on Progress; streak is 0 on Dashboard and 1 day on Progress. **Use one selector per metric.**
5. Drill score shows "0 / 220" on a 10-question drill. **Always show `correct / total questions`, with points shown separately.**
6. Challenges shows "0 tests" next to endless skeletons. **Every list needs real empty, loading and error states.**
7. Breadcrumb formats differ between pages. **Use one format: `Child › Section › Sub`.**
8. Account shows a JEE segment. **Remove JEE entirely.**
9. Progress takes about 10s to load. In our build, skeletons must resolve; seeded data loads in under 300ms.
10. The Platform AI section shows null data. This is work in progress on the live site; ignore it. In our build, **AI is included in the child's subscription** and there is no separate AI plan.

---

## 3. Product rules (decided; do not re-ask)

**Accounts**
- One parent account can have many children.
- A child is in exactly **one class at a time**.
- The class switcher in the sidebar becomes a **child switcher** showing avatar, name and class, e.g. "Aarav · Grade 6".

**Curricula**
- India: CBSE/NCERT, "Grade 1–10", priced in **INR**.
- Germany: Berlin Grundschule/Sekundarstufe, "Klasse 1–10", priced in **EUR**.
- The interface language is EN or DE, and follows the parent's setting.

**Pricing** (1 kid = 1 class = 1 subscription)

| Curriculum | Yearly (school year) | Monthly |
|---|---|---|
| India | ₹999 | ₹99 |
| Germany | **€ TBD**. Placeholder €11.99, held in `config/pricing.ts`, marked TODO. | **€ TBD**. Placeholder €1.19. |

- Yearly plans **auto-renew yearly**; monthly plans auto-renew monthly.
- A failed renewal starts a **1-month grace period**. When it ends, the child's access is paused (data is kept).

**Checkout**
- The cart is grouped by currency.
- Children in India and Germany mean **two separate checkouts**.
- Same-currency children can share one checkout.

**Class change mid-subscription:** the **plan transfers** to the new class. The end date stays the same and there is no new charge. Old-class progress is archived and can still be viewed.

**2FA**
- Parent only.
- Email OTP is required at sign-up and at **every** login.
- No PIN or picture sign-in for parents.

**AI**
- **Only parents** use AI (question generation, import, variants).
- **Children get no AI access**: no hints, no AI explanations, no AI buttons on child-facing screens.

**Parent review:** AI drafts go into a review queue. The parent approves, edits or discards each one before it enters the question bank or reaches the child.

**Points**

| Activity | Points |
|---|---|
| Drill | Drill rate (base) |
| **Parent-made test** | **Same as drills** |
| Platform challenge | Higher rate |
| **Parent-made challenge** | **Same as platform challenges** |

Parent tests may include **locked chapters**. Show a "Locked for Aarav" tag on those chapters, but allow them.

**Unlock rules on the child side** (shown read-only to the parent): challenges unlock after the drills are finished; passing a chapter unlocks the next one.

**Removed**
- "Topics improving" is removed everywhere.
- "Build a custom test" is removed from Settings; it now lives in the Tests tab.
- The JEE segment is removed.

**Buddy requests:** a friend code plus parent approval. They are managed in Settings › Buddy requests. A **notification badge** sits on the bell and on the Settings nav item, and an **Overview alert** links to them.

---

## 4. Information architecture (parent)

```
Auth (logged out)
  /welcome            choose: Parent / Child login
  /signup             parent account
  /verify             email OTP (2FA)
  /login              email + password, then /verify
  /forgot, /reset     password reset (mock email link)

Onboarding (logged in, no child yet, or "Add child")
  /onboarding/child          1 child details
  /onboarding/class          2 class content breakdown (subjects, chapters)
  /onboarding/plan           3 course details + price (yearly/monthly)
  /onboarding/login-setup    4 child login (picture for Grade/Klasse 1–4, username+PIN for 5–10)
  /onboarding/review         5 cart, grouped by currency
  /checkout/redirect         "Redirecting to Stripe…"
  /checkout/stripe-mock      Stripe-style mock checkout
  /checkout/success | /checkout/cancelled | /checkout/failed

App shell (sidebar + top bar)
  Sidebar:  [Logo] [Child switcher]
            Overview · Weak topics · Tests · New questions · Settings
  Top bar:  breadcrumb · notification bell (badge) · theme toggle · avatar menu
            + "Create more questions" button (top right, ONLY on Tests and New questions)

  /overview
  /overview/subject/[id]
  /overview/subject/[id]/chapter/[id]
  /overview/subject/[id]/chapter/[id]/topic/[id]
  /weak-topics
  /weak-topics/[topicId]
  /tests                      list: Assigned · In progress · Completed
  /tests/new                  test builder
  /tests/[id]                 test detail + results
  /questions                  tabs: My content · Platform content · AI generated
  /questions/import           PDF / photo / text, then AI drafts, then review
  /questions/generate         AI generator (questions or challenge)
  /questions/review           review queue
  /settings/account
  /settings/security          password, 2FA (always on), active sessions
  /settings/children          list · edit · reset child login · change class · remove
  /settings/children/[id]/change-class
  /settings/buddy-requests
  /settings/billing           custom Stripe management
  /settings/billing/invoices
  /settings/appearance        Light · Dark · System
  /settings/feedback
  /access-paused              shown for a child past the grace period
```

---

## 5. Screens: requirements per screen

### 5.1 Auth + 2FA (real)

- **Sign-up**
  - Fields: full name, email, password (min 10 chars, strength meter), country (India/Germany), interface language (EN/DE).
  - The parent must accept the terms.
  - Submitting sends an OTP and goes to /verify.
- **Verify OTP**
  - 6 separate digit boxes; paste fills all.
  - The code expires in 10 minutes and allows 5 attempts.
  - "Resend code" has a 60-second cooldown with a visible countdown.
  - Show the masked email ("m•••@gmail.com") and a "Change email" link.
  - Errors: wrong code (show attempts left), expired, too many attempts (locked 15 minutes).
- **Login:** email + password, then OTP, every time.
  - Use a generic error: "Email or password is incorrect."
- **Forgot password:** a mock email link that leads to the reset screen.
- **Backend**
  - Store OTPs hashed, never in plain text.
  - Rate-limit by email and IP.
  - Session is an httpOnly signed cookie.
  - Email service: **Resend**, using env `RESEND_API_KEY`.
  - **Dev fallback:** if the key is missing, print the OTP to the server console and show a small "Dev mode: code in terminal" note on /verify.

### 5.2 Onboarding: add child (5 steps with a stepper)

1. **Child details:** first name, last name, date of birth, avatar picker, country (pre-filled from parent), language.
2. **Class content breakdown.** This replaces the old "different subjects" picker.
   - Select the class (Grade 1–10 or Klasse 1–10).
   - Show **every subject in that class** with its chapter count. Each subject expands to list its chapter names. Use the real data from §8.
   - This screen is read-only content; there is no subject picking.
3. **Course details + price**
   - What's included: drills, challenges, mock tests, progress analytics, parent AI question generator.
   - Yearly/Monthly toggle. Yearly is the default and shows its saving.
   - Show the price in the curriculum's currency, the renewal note, and the grace-period note.
4. **Child login setup**
   - Grades 1–4: choose **at least 3 animals in order** (picture sign-in).
   - Grades 5–10: username + 4-digit PIN.
   - Then "Download login card", a printable card with Nivo, the child's name and the login.
5. **Review**
   - The cart lists all children added in this session.
   - **If currencies are mixed, show two checkout blocks.** Each has its own "Pay with Stripe" button and a note: "India and Germany plans are paid separately."

### 5.3 Stripe mock (frontend only)

- **/checkout/redirect:** a spinner, "Redirecting to Stripe…", then auto-continue after 1.5s.
- **/checkout/stripe-mock:** a Stripe-like layout, with the order summary on the left and the payment form on the right.
  - A banner reads **"Test mode — no real payment"**.
  - The card form is pre-filled with test card `4242 4242 4242 4242`.
  - Buttons: Pay · Cancel · "Simulate failed payment".
- **Success:** the child is activated. Offer "Go to overview" and "Add another child".
- **Cancelled / failed:** say what happened and offer to retry. The cart is preserved.

### 5.4 Overview (per selected child)

- **Card row** (Progress-page style):
  1. Mastery
  2. Accuracy
  3. Coverage
  4. **Subject drill-down card.** A subject dropdown plus that subject's mastery ring, with "Open subject".
- **Recent sessions**
  - A 14-day strip; days with activity get a dot sized by minutes.
  - **Clicking a day changes the session details panel.** It shows the drills, tests and challenges done that day, time spent, score (`correct / total`), points earned, and a mistakes list.
  - Days with no activity show an empty state.
- **Subjects grid:** one card per subject (mastery, accuracy, coverage, attempts, "View chapters").
- **Drill-down:** Subject, then Chapter, then Topic, with a breadcrumb at every level.
  - **Subject detail:** the subject overview (trend line over 8 weeks, chapter table, locked/unlocked state, weak topics in this subject).
  - **Chapter:** a topics table.
  - **Topic:** accuracy trend, attempts, the question types missed, and recent wrong answers.
- **Alerts row** (only when relevant):
  - Pending buddy requests ("Aarav has 1 buddy request waiting" → Settings › Buddy requests)
  - Renewal or grace period
  - Tests due
- **"Topics improving" must not appear.**

### 5.5 Weak topics

- A list sorted by severity: accuracy, number of attempts, and how recently the topic was wrong.
- Filters: subject and chapter.
- **"Worth a look" card** at the top: one topic the child gets wrong **repeatedly**.
  - Show the example question, the child's wrong answers with a count ("Chose 1/3 four times"), the correct answer, and the likely misconception in plain words.
  - Actions: **Set a practice test on this** (opens the test builder pre-filled) · **Generate similar questions** (opens the AI generator pre-filled).
- **Topic detail** works like the overview topic page, with the same actions.

### 5.6 Tests

- **List:** tabs for Assigned, In progress and Completed.
  - Each row shows: title, subjects, number of questions, due date, status, and score `correct / total`.
- **Test builder** (port the "Build my drill" flow; this replaces "Build a custom test" from Settings):
  1. Setup: title, child, due date, timing (untimed / X min), difficulty.
  2. Topics: a tree of Subject › Chapter › Topic with multi-select. **Locked chapters are selectable and tagged "Locked for [child]".**
  3. Questions: add from **My content / Platform content / AI generated**. Show the list, allow reordering by drag, and include a **"+ Add more questions"** button.
  4. Review: question count, estimated time, and **points (drill rate)**. Then **Assign test**.
- **Test detail:** the questions, the child's answers after completion, per-topic accuracy, and "Retest weak topics".
- The **top-right button "Create more questions"** opens /questions/generate.

### 5.7 New questions (needs the most work)

- **Tabs**
  - **My content:** the parent's private question bank. Port the existing bank: filters for subject, chapter, topic and text; add; edit; delete; bulk select.
  - **Platform content:** PrepNivo's questions, read-only, with "Add to test".
  - **AI generated:** approved AI questions, plus a badge showing how many are waiting in the review queue.
- **Add question (manual):** types are multiple choice, true/false, short answer, fill in the blank and number answer. Each question has an explanation field, difficulty, and a subject/chapter/topic tag.
- **Import with AI** (/questions/import) takes three input types:
  - **PDF upload**
  - **Photo** of a textbook page (png/jpg; show a crop preview)
  - **Pasted text**

  Then: a "Reading your material…" state (simulated), then drafts with suggested syllabus placement, then the review queue.
- **AI generator** (/questions/generate)
  - Choose: child, subject, chapter, topic, difficulty, count (5/10/20), question types, and optionally "match this weak topic".
  - Output toggle: **Questions** (into the bank) or **Challenge** (a bundle the child plays as a challenge; **same points as platform challenges**).
- **Review queue:** one card per draft with Approve · Edit · Discard, plus bulk approve. **Nothing reaches the child without approval.**
- The **top-right button "Create more questions"** opens the generator.
- No AI entry points appear on any child-facing preview.

### 5.8 Settings

- **Account:** name, email (read-only; changing it requires OTP), country, language. **No segment and no JEE.**
- **Security:** change password (requires OTP); 2FA shown as "Always on for parent accounts"; a list of active sessions with "Sign out".
- **Children:** child cards (avatar, name, class, plan, renewal date), plus these actions:
  - Edit
  - **Reset child login** (picture or PIN)
  - **Change class**
  - Remove child (type the child's name to confirm; mention the effect on the subscription)
  - Add child (starts onboarding)
- **Change class flow:**
  1. Pick the new class.
  2. Preview: subjects gained and lost, and "Progress in Grade 6 is archived and still viewable".
  3. "Your plan transfers. Renewal date unchanged."
  4. Confirm, then a toast.
- **Buddy requests:** Pending (Approve / Decline), Approved (Remove), Declined. Each request shows the friend's display name, class, how they connected (friend code), and the date. This is where the badge's count comes from.
- **Billing** (custom Stripe-style management):
  - Per child: a subscription card with plan, price, status (Active / Renewing / Grace period / Paused), next renewal date, and an auto-renew toggle.
  - Actions: switch yearly/monthly (effective from the next renewal), cancel (confirm, then access continues to the end of the period).
  - Payment methods: one card per currency (e.g. "Visa ending 4242"), with "Update card" (mock form).
  - **Invoices:** date, child, period, amount, currency, status, and download (mock PDF).
  - **Grace-period state:** a red banner: "Payment for Anaya failed on 1 Oct. Update your card by 1 Nov to keep access." With an "Update card" button.
- **Appearance:** Light / Dark / System.
- **Feedback:** keep the existing form.

### 5.9 System states (every screen)

Every screen has loading (skeletons that resolve), empty states that tell the parent what to do next, and errors that say what went wrong and how to fix it. Toasts use the same verb as the button that triggered them ("Assign test" → "Test assigned").

---

## 6. Design system

### Type
- **Desktop-first, and every text must be clearly readable.** The **minimum size anywhere is 14px**, including captions, table meta and chart labels. Nothing is smaller.

| Role | Size / line height | Weight |
|---|---|---|
| Caption / meta | 14 / 20 | 500 |
| Small UI (buttons, tabs, table body) | 15 / 22 | 600 |
| Body | 17 / 26 | 400–500 |
| H4 (card titles) | 18 / 26 | 700 |
| H3 | 21 / 28 | 700 |
| H2 (page titles) | 28 / 34 | 800 |
| H1 / big stat numbers | 36 / 40 | 800 |

- Use the same rounded geometric sans as the live site; inspect prepnivo.com to find it. If it can't be identified, use **Nunito** for headings and **Nunito Sans** for body.
- Don't use all-caps labels for anything longer than two words. Use sentence case everywhere.

### Layout
- Desktop frame: 1440×900.
- Sidebar: 264px. Content max-width: 1200px. 8px spacing grid.
- Card radius: 16px. Input and button radius: 12px.
- Charts must stay legible: axis labels at least 14px, never rely on colour alone, and include a legend.

### Colour tokens (CSS variables; Tailwind maps to these)

| Token | Light | Dark |
|---|---|---|
| `--bg` | `#EAF4EE` | `#111A16` |
| `--surface` | `#FFFFFF` | `#18231E` |
| `--surface-2` | `#F4F9F6` | `#203029` |
| `--border` | `#D6E5DC` | `#2F4239` |
| `--text` | `#1F2A24` | `#E8F1EC` |
| `--text-muted` | `#55665C` | `#A7B9AF` |
| `--primary` | `#2E9B63` | `#4CC48A` |
| `--on-primary` | `#FFFFFF` | `#0C1F15` |
| `--primary-soft` | `#DDF1E5` | `#1E3A2C` |
| `--warning` | `#B26A00` | `#F2B24C` |
| `--danger` | `#C2362B` | `#FF8A80` |
| `--success` | `#2E7D4F` | `#6FD39E` |
| `--focus` | `#1B6FD8` | `#7FB2FF` |
| `--nivo-disc` | `#FBF6EE` | `#FBF6EE` (same in both modes) |

### Dark mode rules
- Dark mode is **parent side only**.
- Contrast:
  - Body text: WCAG AA at least 4.5:1, and **aim for 7:1**.
  - Muted text: at least 4.5:1 on both `--surface` and `--surface-2`.
  - UI borders, icons and focus rings: at least 3:1.
  - **Add an automated check** (`scripts/contrast-check.ts`) that verifies every text/background token pair against these thresholds and fails CI if any pair is below.
- Don't use pure black and don't use pure-white body text.
- Elevation in dark mode comes from lighter surfaces, not shadows.
- Charts use a separate dark palette. Gridlines use `--border`.
- Status colours always come with an icon and a text label.
- Toggle: Settings › Appearance plus a top-bar toggle. The default is System. Persist the choice.

### Nivo and logo (`/assets`)
- **Nivo (`assets/nivo.svg`) must look identical in light and dark modes.**
  - Don't recolour it or invert its outline.
  - Always place it on a `--nivo-disc` circle (the same cream in both modes), so its dark outline stays visible on dark backgrounds.
  - Refine the SVG to match the reference image in `assets/` if needed. Keep the colours: body `#F29C4F`, wings `#FFB878`, belly `#FFD6A0`, horns `#FFD166`, outline `#1F2430`.
- Nivo appears on: welcome/auth, onboarding steps, empty states, checkout success and the login card. Do not put Nivo on data-dense screens.
- **Logo (`assets/prepnivo-logo.svg`):** a four-circle mark plus the "Prep" (text colour) "Nivo" (primary) wordmark. Its colours come from CSS variables, so it adapts to dark mode. Nivo does not.

---

## 7. Tech stack (standalone)

- **Next.js (App Router) + TypeScript + Tailwind CSS.** Use shadcn/ui primitives, restyled to the tokens above.
- **Prisma + SQLite** for parent accounts, sessions and OTPs only.
- **Resend** for OTP emails, with the dev fallback described in §5.1.
- Session: iron-session (or equivalent) with a signed httpOnly cookie.
- Mock data: typed seed files in `/data`. Client state: Zustand.
  - Mutations (assigning tests, approving questions, billing changes) persist in client state for the session.
- Charts: Recharts, themed with the tokens.
- Theme: `next-themes` with the `class` strategy.
- Testing:
  - Playwright for these flows: sign-up → OTP → add 2 children (INR + EUR) → 2 checkouts → overview; create test; AI generate → review → approve; change class; grace-period banner.
  - Vitest for OTP logic and the price/cart grouping.
- `.env.example` must contain `DATABASE_URL`, `SESSION_SECRET` and `RESEND_API_KEY` (optional), plus `EMAIL_FROM`.
- `README` must give one-command setup: `pnpm i && pnpm db:push && pnpm seed && pnpm dev`.

---

## 8. Seed data (realistic; populate every field)

**Parent:** Meera Kapoor, meera.kapoor@example.com, lives in Berlin, interface language EN. In the seed, the password is `DemoParent!2026`. 2FA is still required, and the OTP goes to the console in dev.

**Child 1: Aarav Kapoor**, age 11, **Grade 6, India (CBSE/NCERT)**, INR, yearly plan ₹999, active, renews 13 Jul 2027.

| Subject | Chapters |
|---|---|
| Maths (NCERT *Ganita Prakash*) | Patterns in Mathematics; Lines and Angles; Number Play; Data Handling and Presentation; Prime Time; Perimeter and Area; Fractions; Playing with Constructions; Symmetry; The Other Side of Zero |
| Science (NCERT *Curiosity*) | The Wonderful World of Science; Diversity in the Living World; Mindful Eating; Exploring Magnets; Measurement of Length and Motion; Materials Around Us; Temperature and its Measurement; A Journey through States of Water; Methods of Separation in Everyday Life; Living Creatures; Nature's Treasures; Beyond Earth |
| English (*Poorvi*) | 5 units |
| Hindi (*Malhar*) | — |
| Social Science (*Exploring Society: India and Beyond*) | — |

- Weak topics, in order:
  1. Fractions › Comparing unlike fractions. **This is the "Worth a look" example.** Aarav chose 1/3 > 1/2 four times; the misconception is "bigger denominator = bigger fraction".
  2. Prime Time › Prime factorisation.
  3. Measurement of Length and Motion › Converting units.
- 14 days of sessions with varied activity.
- 1 pending buddy request, from "Kabir S., Grade 6, friend code".
- 1 test assigned, due Fri.
- 2 completed tests.

**Child 2: Anaya Kapoor**, age 8, **Klasse 3, Germany (Berlin)**, EUR, monthly plan (placeholder price), **status: grace period**. The payment failed on 1 Oct 2026 and the grace period ends on 1 Nov 2026.
- Subjects: Deutsch, Mathematik, Sachunterricht, Englisch, Kunst, Musik.
- Mathematik chapters: Zahlen bis 1000; Addition und Subtraktion bis 1000; Das Einmaleins; Division mit Rest; Geometrie: Flächen und Körper; Größen: Geld, Längen, Zeit, Gewicht; Sachaufgaben.
- Picture login: 3 animals (fox, owl, turtle).
- Weak topic: Das Einmaleins › 7er- und 8er-Reihe.

The team should verify curriculum details; keep them in `/data/curricula/*.ts` so they're easy to edit.

---

## 9. Acceptance checklist

- [ ] Sign-up and login both require a working email OTP. Expiry, resend cooldown, attempt limits and lockout all work.
- [ ] There is no PIN or picture sign-in for parents.
- [ ] Onboarding has all 5 steps, and the class breakdown shows real subjects and chapters.
- [ ] A mixed INR/EUR cart produces 2 separate checkouts. Mock Stripe success, cancel and fail all work.
- [ ] The sidebar is exactly: Overview · Weak topics · Tests · New questions · Settings, with the child switcher above it.
- [ ] "Create more questions" appears only on Tests and New questions.
- [ ] Overview has 4 cards (the 4th is the subject drill-down), a clickable day strip that updates session details, and a Subject › Chapter › Topic drill-down.
- [ ] "Topics improving" does not appear anywhere.
- [ ] The Weak topics "Worth a look" card shows the repeated wrong answer, with both actions working.
- [ ] The test builder allows locked chapters (tagged) and "+ Add more questions". Parent test points equal drill points.
- [ ] AI import accepts PDF, photo and text. Every AI draft goes through the review queue. A parent-made challenge earns challenge points.
- [ ] No AI is available anywhere a child would see.
- [ ] Settings has no "Build a custom test" and no JEE, and includes Buddy requests. The badge appears on the bell and the Settings nav, and the Overview alert links to Buddy requests.
- [ ] Change class transfers the plan, keeps the renewal date and archives old progress.
- [ ] The billing screen shows Anaya's grace-period banner, plan switching, auto-renew, cards and invoices.
- [ ] Dark mode passes the contrast script. Nivo looks identical in both modes.
- [ ] No text anywhere is under 14px.
- [ ] Every list has loading, empty and error states.
- [ ] Playwright flows pass.
