# PrepNivo Parent App — Build Spec

Single source of truth for Claude Code (CLAUDE.md) and Codex (AGENTS.md). Both agents build the same thing from this file. If a task file and this spec disagree, this spec wins.

Last updated: 11 Oct 2026. §5.10 (6 Oct) and §5.11 (9 Oct) were added from review rounds on the prototype. Original version: 5 Oct 2026, from team meeting notes and Q&A with Aayushmaan.

**Reference prototype:** `prototype/prepnivo-prototype.html` is a working single-file prototype of the whole product. It is the agreed reference for screens and behaviour. If this spec and the prototype disagree, this spec wins; log the difference in `DECISIONS.md`. Its "Prototype tools" panel is for review only and is not part of the product.

**Later sections win.** §5.10 and §5.11 replace earlier rules where they conflict. For example, §5.11 replaces the test builder in §5.6 and the tabs in §5.7. Phase text in CLAUDE.md and AGENTS.md that disagrees with them is out of date.

---

## 1. What we are building

A **standalone, runnable desktop web app** for the **parent side** of PrepNivo, based on the live structure at prepnivo.com/dashboard. It also includes a **public home page** (§5.10) and a **child app at `/kid`** (§5.11.10) that plays what parents assign and writes results back to the parent side. The child app is frontend-only with mock data, always uses the light theme, and has no AI.

| Area | Real or mock |
|---|---|
| Parent sign-up, login, **email 2FA (OTP)** | **Real.** Minimal backend: API routes, SQLite, email service. |
| Stripe checkout | **Frontend mock.** "Redirecting to Stripe…" then a Stripe-style mock checkout page, then success, cancel or fail. |
| Payment management | **Custom in-app screen** with mock data. Not the Stripe Customer Portal. |
| Dashboard, progress, tests, questions, AI generation | **Frontend with seeded realistic data.** AI generation is mocked with canned output and a simulated delay. |
| AI usage billing (tokens, top-ups) | **Frontend mock.** Usage is estimated and recorded in client state; top-ups create mock invoices. |
| Child app (`/kid`) | **Frontend mock** on the same seed data. Results update the parent side. |

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
| **Olympiad paper** (§5.11.6) | **Same as drills** |

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
Public (logged out)                                            §5.10
  /home               public home: features, how it works, pricing, safety, FAQ
  /goodbye            after account deletion
  /maintenance        planned maintenance
  /error              server error page with a reference code
  (unknown routes)    "We couldn't find that page" (public frame, or in-app when logged in)

Auth (logged out)
  /welcome            choose: Parent / Child login
  /child-login        redirects to /kid/login
  /signup             parent account
  /verify             email OTP (2FA)
  /login              email + password, then /verify
  /forgot, /reset     password reset (mock email link)

Onboarding (logged in, no child yet, or "Add child")
  /onboarding/start          registration welcome after email is verified (§5.10)
  /onboarding/child          1 child details
  /onboarding/class          2 class content breakdown (subjects, chapters)
  /onboarding/plan           3 course details + price (yearly/monthly)
  /onboarding/login-setup    4 child login (picture for Grade/Klasse 1–4, username+PIN for 5–10)
  /onboarding/review         5 cart, grouped by currency
  /checkout/redirect         "Redirecting to Stripe…"
  /checkout/stripe-mock      Stripe-style mock checkout
  /checkout/success | /checkout/cancelled | /checkout/failed[?reason=declined|funds|expired|auth]
  /checkout/pending          connection lost during payment: "Don't pay again" (§5.10)

App shell (sidebar + top bar)
  Sidebar:  [Logo] [Child switcher]
            Overview · Weak topics · Tests · New questions · Settings
  Top bar:  breadcrumb · notification bell (badge) · theme toggle · avatar menu
            + "Create more questions" button (top right, ONLY on Tests and New questions)
  Avatar menu: Account · Settings (→ /settings/children) · Log out      (§5.11.8)

  /overview
  /overview/subject/[id]
  /overview/subject/[id]/chapter/[id]
  /overview/subject/[id]/chapter/[id]/topic/[id]
  /weak-topics
  /weak-topics/[topicId]
  /tests                      list: Assigned · In progress · Completed; filter by type and subject (§5.11.6)
  /tests/new                  builder: Subject test (4 steps) or Olympiad paper (3 steps) (§5.11.5–6)
  /tests/[id]                 test detail + results (paper questions hidden until the child finishes)
  /questions                  one list of the parent's own questions, with source symbols (§5.11.2–3)
  /questions/import           PDF / photo / text, then AI drafts, then review (?paper=1: complete paper)
  /questions/generate         AI generator for the child selected in the sidebar (questions or challenge)
  /questions/review           opens /questions with the review pop-up open (§5.11.4)
  /settings/account
  /settings/security          password, 2FA (always on), active sessions
  /settings/children          list · edit · reset child login · change class · remove
  /settings/children/[id]/change-class
  /settings/buddy-requests
  /settings/billing           custom Stripe management + AI usage and top-ups (§5.11.7)
  /settings/billing/invoices
  /settings/appearance        Light · Dark · System
  /settings/feedback
  /access-paused              shown for a child past the grace period

Child app (always light theme, no AI)                          §5.11.10
  /kid                        → /kid/home if signed in, else /kid/login
  /kid/login                  "Who's practising today?": the device's family, pick avatar
  /kid/login/[childId]        picture code (Grade/Klasse 1–4) or PIN (5–10)
  /kid/home                   tests from the parent · Keep going · subjects · badges · streak · points
  /kid/subject/[id]           chapters: drill, then challenge; locked chapters
  /kid/play                   question player (drill, challenge, test, paper)
  /kid/result                 score, points, unlocks, tricky ones
  /kid/buddies                friend code, add a buddy, buddies, waiting for a grown-up
  /kid/paused                 the child's access is paused
```

`/styleguide` (Phase 0) stays as a development route. It is not linked from any product menu.

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

### 5.10 Public home, registration, and network / session / payment edge cases (added 6 Oct 2026)

**Public home (`/home`).** The default route when logged out (`/` → `/home`, or `/overview` when logged in).
- Top bar: logo; section links (How it works · Pricing · Safety · FAQ) that scroll; "Log in" (→ /welcome) and "Create account" (→ /signup).
- Hero: headline, one-line pitch, both buttons, "CBSE / NCERT Grade 1–10 · Berlin Klasse 1–10", and Nivo.
- Four feature cards: Overview, Weak topics, Tests, New questions.
- How it works: 3 numbered steps (create account, add each child, follow along and help).
- Pricing: one card per curriculum, from `config/pricing.ts`: yearly price, monthly price, the yearly saving, auto-renew, and the 1-month grace period. EUR shows "Euro prices to be confirmed".
- Safety: email 2FA for parents; no AI for children; you approve every AI question; buddies need your OK.
- FAQ (accordion): more than one child; class change mid-year; failed payment; does my child use AI; how children sign in.
- Footer: Log in · Child sign-in (→ /kid/login) · Terms · Privacy.

**Registration welcome (`/onboarding/start`).** Shown right after the sign-up OTP is verified.
- Nivo, an "Email verified" badge, and "Welcome, <first name>! Your account is ready".
- "You'll need": the child's name and date of birth, their class, and a card. Show the price per child in the parent's currency.
- "Add my first child" starts onboarding. "I'll do this later" logs out and shows the notice "Your account is saved. Log in any time to add your first child."
- A logged-in parent with no children who opens an app route goes to `/onboarding/child`.

**First-run card on Overview.** New accounts get a "Getting started with <child>" card with a progress bar and 4 steps:
1. Set up the child's login on their device (show the login card).
2. The child finishes a first drill.
3. Set a first test.
4. Try the AI question generator.

Each step shows "Done" or an action. The card can be hidden. If any children in the cart are unpaid, an alert says "<names> is waiting for payment" with "Finish checkout".

**Sign-up, login and password edge cases**
- Sign-up keeps everything typed when there's an error.
- If the email already has an account, show an inline message with "Log in" and "reset your password" links.
- Suggest corrections for common email-domain typos ("Did you mean …@gmail.com?").
- The password must not contain the email's local part. Password fields have a Show/Hide toggle.
- Logging in with an email whose sign-up was never confirmed sends a new sign-up code and shows "You started signing up with this email but didn't confirm it."
- Login: 5 wrong passwords lock that email for 15 minutes ("Too many attempts. Try again in N minutes, or reset your password.").
- OTP: at most 5 codes per email per hour. /verify shows "Can't find it? Check your spam folder" and a "Send a new code" button when the code has expired.
- Password reset: the link works once, for 1 hour. An expired or used link shows "This link has expired" with "Send a new link".
  - The new password must differ from the old one.
  - A reset logs the parent out on all devices. The login screen then says "Password changed. You were logged out on all devices."
- Changing the email or password needs an OTP (in a pop-up). The email change sends the code to the new address.

**Session**
- Idle for 30 minutes: logout. On the next action, go to /login with "For your security, you were logged out after 30 minutes without activity." After login, return to the page they were on (but never back into onboarding or checkout).
- Logging out in one tab logs out every tab: "You logged out in another tab or window."

**Network and server errors.** Every server-backed action goes through one API layer.
- **Offline:**
  - Show a sticky banner: "You're offline. Pages you've opened still work. Logging in, paying, saving and AI need a connection." It has a "Try again" button.
  - Lists that haven't loaded yet show "You're offline, so this can't load" with "Try again".
  - Coming back online shows the toast "Back online" and reloads lists.
- **Failed actions:**
  - Buttons are disabled with a spinner while an action runs, which prevents double submits.
  - A failed action shows the toast "Couldn't <action>. <reason>" and keeps what the parent typed.
  - Use these reasons:
    - offline: "No internet connection. Check your Wi-Fi or mobile data, then try again."
    - server: "Something went wrong on our side. Please try again in a minute."
- **Lists that fail to load** show "We couldn't load this list" with how to fix it and "Try again".
- **Browser storage blocked:** show the banner "This browser is blocking storage, so changes won't be kept after you close the tab."
- **Server error page (`/error`):** "Something went wrong", "Your data is safe", an error reference ("PN-XXXXXX") to quote to support, then "Try again" and "Go to overview" (or "Go to home").
- **Maintenance (`/maintenance`):** "PrepNivo is getting an update", the time we'll be back in CET and IST, "Your children's progress is safe".
- **Page not found:** logged in, show it inside the app shell; logged out, show it in the public frame with "Go to home" and "Log in".

**Onboarding validation**
- **Child details**
  - First name is required, 40 characters at most.
  - Date of birth is required and can't be in the future. Ages under 4 or over 17 are rejected with a reason.
  - Don't allow the same first name and date of birth twice in the family or cart.
  - An account can have up to 6 children ("Contact support if you need more").
- **Class:** if the class doesn't match the age (more than 1 year off), show a non-blocking warning: "<child> is 11. Most children this age are in Grade 5–6. Check you've picked the right class."
- **Picture code:** 3–6 animals.
- **Username:** 3–20 letters, numbers or dots, unique within the family.
- **PIN:** exactly 4 digits.
  - Reject easy PINs: repeated digits, 1234, 4321, 1212, 0123, 2580, 9876.
  - Reject the child's birth year.
- Classes without confirmed chapter data show their subjects with "Chapters to be confirmed".

**Checkout edge cases**
- `/checkout/redirect` while offline: "We couldn't reach Stripe. You haven't been charged, and your cart is saved."
- An already-paid checkout (back button or refresh) shows "This checkout is already paid. You won't be charged again."
- The mock checkout handles these outcomes:
  - success
  - 3-D Secure confirmation ("Confirm with your bank")
  - card declined
  - insufficient funds
  - expired card
  - authentication failed
  - connection lost during payment

  The failed page names the reason and the fix. The prototype's "Test scenario" picker is for review only.
- Connection lost while paying → `/checkout/pending`:
  - "We're confirming your payment … **Don't pay again.** We'll show the result here, and email you either way."
  - "Check payment status" resolves it to success or failed.
- Success with another currency still unpaid: "You still have a Germany (EUR) checkout to complete for <names>", with "Continue".

**Delete account.** In Settings › Account, under a "Delete account" danger zone.
- Explain the effect: every child's progress, tests and questions are deleted, and all subscriptions are cancelled with no refund for the current period.
- Type `DELETE` to confirm, then enter an email OTP.
- Then go to `/goodbye` ("Your account is deleted … A confirmation is on its way to your email").

### 5.11 Third round: questions, tests, Olympiad papers, AI billing, child app (added 9 Oct 2026)

This section replaces the conflicting parts of §5.6 (test builder) and §5.7 (tabs, review queue, generator child picker).

#### 5.11.1 AI generator: no child picker
- `/questions/generate` has no child field. It uses the child selected in the sidebar, and says "For <child> · <class>" with "To make questions for another child, switch child in the sidebar."
- The fields are: subject, chapter, topic, difficulty, count (5/10/20), question types, and "Match this weak topic". "Match this weak topic" is enabled only when the topic is weak for this child.
  - Locked chapters are labelled "(locked for <child>)".
- A link with `?topic=` for another child's class switches the sidebar to that child.
- Classes without a syllabus show an empty state, because AI can't write questions for them yet.
- After generating, go to the review pop-up (§5.11.4).

#### 5.11.2 New questions: one list, source symbols
- There is **no Platform content tab anywhere**, including the "Add from my questions" picker in the test builder. Parents never browse PrepNivo's question bank.
- "My content" and "AI generated" are **one list**: the parent's own questions for the selected child's class.
- Each row shows a **source symbol only** (icon with a tooltip and an accessible label):
  - **sparkles** = AI generated
  - **file** = from an uploaded file (tooltip: "From <file name>")
  - **pencil** = added by the parent
- A legend under the page title explains the three symbols.
- Page actions: "Upload a file", "Generate with AI", "Add question".
- Rows have Edit, Delete and Add to test. Bulk select supports "Add to test" and "Delete".
- Deleting a question used by an unfinished test warns that the test keeps a copy.

#### 5.11.3 Organising your own questions
- **Sort:** newest first (default), oldest first, by subject and chapter, or by topic A–Z.
- **Filters:**
  - subject / chapter / topic / text search
  - source (all, added by you, AI generated, from a file)
  - file name
  - date added: any, today, last 7 days, last 30 days, or "Choose dates" (from–to, not after today)
- **"Your uploads" chips:** one per file, showing the file name, number of questions and last date. Clicking a chip filters by that file; clicking again clears it.
- Show "N questions shown of M", "Clear filters", and an empty state for "No questions match".

#### 5.11.4 Review pop-up (replaces the review queue page)
- New and generated questions open in a **review pop-up**, newest on top.
  - Each draft shows its source symbol, its origin (e.g. "Generated · <topic>" or "Uploaded <file>") and its time.
  - Each draft has **Approve · Edit · Discard**.
  - The header has **Approve selected (N)** and **Approve all N**.
- Edit opens the question editor and returns to the pop-up.
- When everything is reviewed, show "All reviewed".
- `/questions/review` opens `/questions` with the same pop-up open.
- While drafts are waiting, `/questions` shows an alert with "Review now".
- Nothing reaches a child until it's approved. A challenge or uploaded paper is assigned only once **every** draft in it is reviewed. The toast says "Challenge assigned" or "Paper assigned".

#### 5.11.5 Tests: one subject, up to 15 questions
- **One subject per test.** A test has at most **15 questions**.
- **Step 1, Setup:**
  - Type toggle: Subject test or Olympiad paper (see §5.11.6).
  - Subject: subjects without questions are marked "(no questions yet)".
  - Title: defaults to "<Subject> test, <date>".
  - Number of questions: 5 / 10 / 15.
  - Difficulty: easy / medium / hard / mixed.
  - Due date: from today onwards.
  - Timing: untimed, or timed with 5–180 minutes.
  - Shows "For <child> · <class>".
- **Step 2, "What this <subject> test covers":** topics are **picked automatically**.
  - **Order:** the child's weak topics in this subject; then the topics of the current (newest unlocked) chapter; then topics practised in the last 14 days; then other unlocked chapters.
  - Pick **up to 4 topics**, from **unlocked chapters only**. Skip topics that have no questions.
  - Each topic shows **why it was picked**: Weak topic · Current chapter · Practised recently · Chosen by you.
  - Summary tiles: questions (N of count), topics, chapters, points.
  - The table shows each topic's chapter, the child's score and its number of questions.
  - **The parent can change topics** under "Change topics". This is a tree for this subject only. Locked chapters are selectable and tagged "Locked for <child>".
  - "Pick topics automatically again" restores the automatic choice.
  - If there aren't enough questions, a warning suggests adding the parent's own or generating some.
- **Step 3, Questions:** PrepNivo questions are **auto-picked** for the chosen topics.
  - The parent sees **only the questions in this test**, never the platform bank.
  - For each question they can **remove** it, **replace** it, reorder it (drag or arrow buttons), or **add** more.
  - Adding uses one **"Add from my questions"** button. It opens a picker of the parent's own questions in this subject (search, "Only this test's topics"), newest first, and stops at 15.
  - **Replace** offers another PrepNivo question on the same topic, or one of the parent's own questions from the same subject (same topic first).
  - A removed PrepNivo question isn't picked again for this test.
- **Step 4, Review:** questions, time (timed, or an estimate), points (drill rate), title, subject, due date, difficulty and topics. Topics from locked chapters get a note. Then **Assign test**.
- **Adding from elsewhere.** "Add to test" from New questions adds to the current draft (creating one if needed). Questions from another subject, or beyond 15, are **skipped with a message**: "N skipped: tests have one subject", "N skipped: max 15 per test".
- **Pre-filled entry points:**
  - "Set a practice test on this" and "Retest weak topics" open step 2 pre-filled.
  - A retest avoids the original test's questions.
- The tests list shows a draft banner: "You have a draft: … Continue building".
- An unstarted test or paper can be removed.

#### 5.11.6 Olympiad papers
- **What they are:** complete, timed papers, possibly multi-subject, split into sections. Examples: Logical reasoning, Mathematical reasoning, Everyday mathematics, Achievers section.
- **Setting one** (the builder's "Olympiad paper" type has 3 steps: Setup → Paper overview → Review):
  - The parent picks a paper from the class's **catalogue**. Each paper shows its question count, minutes and subjects, with a "Multi-subject" badge where relevant.
  - Or they **upload a whole paper**: `/questions/import?paper=1`, with the "This is a complete question paper" option, a paper title, and the time allowed (10–180 minutes).
    - The AI reads it into drafts, grouped into sections by subject.
    - The parent reviews every draft. The paper is assigned once every draft is reviewed.
- **Hidden questions:** the parent **can't see paper questions until the child finishes**. Before that, they see the sections and question counts and "Questions are hidden for now".
  - After completion, they see score by section, accuracy by topic, and the questions with the child's answers.
  - Retest isn't offered for papers.
- **Points:** papers earn points at the **drill rate**.
- **Tests list:** filter by type (All · Subject tests · Olympiad papers · Challenges) and by subject, on top of the Assigned / In progress / Completed tabs.
  - Rows show "Challenge" or "Olympiad paper" badges.
  - Multi-subject tests show "Multi-subject".
  - Points show as earned, or as "up to N".

#### 5.11.7 AI usage billing (Settings › Billing › "AI question tools")
- **Included:** **50,000 tokens a month per active child** (paused children don't count), included in the subscription. This is not a separate AI plan.
- **Top-ups** add tokens to the current month and expire at the monthly reset:

  | Tokens | INR | EUR |
  |---|---|---|
  | 50,000 | ₹79 | €0.99 (placeholder, `// TODO confirm`) |
  | 200,000 | ₹249 | €2.99 (placeholder, `// TODO confirm`) |

  - The currency follows the parent's country.
  - Buying a top-up needs a card for that currency, and opens a confirmation pop-up.
  - **Every top-up creates an invoice** ("AI top-up · 50,000 tokens") in the invoices table.
- **Automatic top-up:** optional and **off by default**. When on and tokens run out, we add the smallest pack automatically, with an invoice.
- **Usage meter:**
  - Shows used / allowance (%), tokens left and the reset date.
  - Turns amber at 80% and red at 100%.
  - Also shows: included tokens (per child × children), top-ups, questions drafted, files and photos read.
- **Usage history:** a table of date and time, action ("Generated questions", "Read a file", "Read a photo", "Read pasted text"), details and tokens.
- **Explainer** ("What the AI does"):
  - What the AI does: writes syllabus-matched questions, reads worksheets, photos, papers and notes into drafts, and targets weak topics.
  - Every draft waits for the parent's approval. Children never use AI.
  - **What a token is:** about 450 per question, about 6,000 per one-page worksheet, and about 2,600 per photo.
- **Every AI action shows its estimated token cost** before it runs. This covers the generator and the upload; the generator panel also shows a compact meter and links to usage.
  - When the estimate is more than what's left (and automatic top-up is off), the action is **blocked**. Show "Not enough tokens left this month" with a **"Buy tokens"** button.
- **At 80% usage**, a notification appears on the bell ("You've used N% of this month's AI tokens"; at 100%, "You have no AI tokens left this month"). It links to Billing.

#### 5.11.8 Profile (avatar) menu
- Shows the parent's name and email, then **only**: Account (→ Settings › Account), **Settings** (→ **Settings › Children**) and Log out.
- **The style guide is removed from all product menus.**

#### 5.11.9 Recent sessions: several per day
- When a day has several sessions, the details panel shows one card per session in time order: **"Session 1 · 4:10 pm"**, "Session 2 · …".
  - Each card has the type badge, title and subject, minutes, score (`correct / total`), points and its mistakes.
  - The panel header totals the day's sessions, minutes and points.
- The 14-day strip shows **×N** under days with more than one session.

#### 5.11.10 Child app (`/kid`)
The child app is **always light theme**, with **no AI anywhere**: no hints, AI explanations, AI buttons or AI wording. It is styled for children, with large type and buttons; the 14px minimum still applies.

- **Sign-in (device-linked family)**
  - The device belongs to the family of the last parent who logged in on it.
  - If no parent has logged in on the device yet, show "This device isn't set up yet" with "Parent log in".
  - "Who's practising today?": the child picks their **avatar**.
  - Then they enter their code:
    - Grade/Klasse 1–4: a **picture code** (tap the animals in order, with Undo).
    - Grade/Klasse 5–10: a **4-digit PIN** on a keypad (keyboard digits work too).
  - **5 wrong tries lock that child's sign-in for 5 minutes** ("Too many tries. Ask a grown-up to help, or wait N minutes"). From 2 tries left, the screen warns how many are left.
  - "That's not me" goes back. "I'm a parent" goes to the parent login.
  - A paused child goes to `/kid/paused` ("PrepNivo is paused for now … all your progress is safe").
- **Home**
  - Greeting with Nivo and the streak.
  - **"From your parent":** tests, challenges and papers that aren't completed, sorted by due date. Each shows "Was due" when late, plus Start or Carry on.
  - **"Keep going":** a drill for the current chapter.
  - Subjects with mastery bars.
  - Badges: First drill · 3-day streak · 500 points · Challenge done · First test · 1,000 points.
  - The header shows points, streak, avatar, Home, Buddies and Log out.
- **Subject page:** a list of chapters.
  - Locked chapters show "Finish chapter N to unlock". Chapters without content show "Coming soon".
  - Unlocked chapters have **Drill** (10 questions) and **Challenge** (8 questions, double points).
  - **A challenge unlocks after the drill** (best drill score ≥ 60%).
  - **Scoring ≥ 70% on the newest unlocked chapter's drill unlocks the next chapter.**
- **Player**
  - Header: the progress bar, "Question N of M", and either a timer or a type chip.
  - **Drills and challenges give feedback after each question:** Check, then "Correct! +N points" or "Not quite. The answer is …". The explanation is shown without any AI wording.
  - **Tests and papers give no feedback.** The child can go Back and Next freely. Paper questions show their section.
  - **Timed** tests and papers show a countdown, and finish automatically at 0:00 ("Time's up!").
  - **Finishing with blank answers asks for confirmation:** "You've answered N of M … Finish now?"
  - **Quit:** a drill's answers are discarded. A test goes back to Assigned, and its answers are not kept.
- **Result page:** an encouraging headline, the score `correct / total`, points, any chapter unlocked, and "Let's look at the tricky ones" with the child's answer and the correct one.
- **Results write back to the parent side:**
  - The test is completed with its answers and score.
  - A session is added for today, with its time, minutes, score, points and mistakes.
  - Topic stats are updated (attempts, correct, last practised, last wrong, missed types, recent wrong answers).
  - Chapter unlocks are recorded.
  - Weak topics, Overview and Tests update from these, using the same selectors.
- **Offline:** results are **queued on the device** and the result page says "You're offline. Your score is saved on this device and will be sent when you're back online." They **sync automatically** when back online ("Your score is saved").
- **Buddies**
  - The child's **friend code** (format `ABC-1234`) with "A grown-up has to say yes before you become buddies".
  - "Add a buddy" takes a friend's code and validates it: wrong format, own code, unknown code, already buddies or already asked.
  - **Adding a buddy creates a pending request for the parent** in Settings › Buddy requests (with badges and the Overview alert). The child sees it under "Waiting for a grown-up".
  - Approved buddies are listed.

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

**Added in §5.11 (match the prototype's seed):**
- Every session has a start time. Some days have 2 or more sessions, e.g. Aarav yesterday and 4 days ago, and Anaya 3 days ago.
- Questions from uploaded files:
  - "Fractions worksheet.pdf" (3 questions, Aarav)
  - "Magnets notes.jpg" (2 questions, Aarav)
  - "Einmaleins Arbeitsblatt.pdf" (1 question, Anaya)
- Every question has a `createdAt` timestamp.
- AI usage history for October 2026: 5 entries, about 19,300 tokens in total. Automatic top-up is off and there are no top-ups.
- Olympiad paper catalogue:
  - Grade 6: Maths Level 1, Science Level 1, Multi-subject.
  - Klasse 3: Mathe-Olympiade Übungsrunde.
- Friend codes the prototype recognises: RIA-5520 (Ria P., Grade 6), LEO-3391 (Leo M., Klasse 3), ZOE-8812 (Zoe K., Grade 5), KAB-6271 (Kabir S., Grade 6).

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

**§5.10: public home, registration, edge cases**
- [ ] `/home` is the logged-out default, with features, how it works, pricing from `config/pricing.ts`, safety and FAQ.
- [ ] After the sign-up OTP, `/onboarding/start` welcomes the parent. "I'll do this later" logs out with a notice. New accounts see the first-run "Getting started" card.
- [ ] Login locks for 15 minutes after 5 wrong passwords. Reset links work once, for 1 hour, and log out all devices. 30 minutes idle logs out and returns to the same page after login. Logging out in one tab logs out the others.
- [ ] Offline shows the banner and offline list states. Failed actions keep the input and show "Couldn't …" toasts. Buttons can't be double-submitted. `/error`, `/maintenance` and the not-found page exist.
- [ ] Onboarding rejects invalid ages, duplicate children, more than 6 children, weak PINs and duplicate usernames, and warns when the age and class don't match.
- [ ] Checkout handles 3-D Secure, the decline reasons, offline redirect, already-paid, and connection lost (`/checkout/pending`, "Don't pay again").
- [ ] Delete account needs `DELETE` plus an OTP, cancels every subscription and ends on `/goodbye`.

**§5.11: third round**
- [ ] The AI generator has no child picker. It says "For <child> · <class>" and follows the sidebar.
- [ ] There is no Platform content tab anywhere. New questions is one list with source symbols (sparkles / file with the file name as a tooltip / pencil).
- [ ] Your own questions are newest first, with every sort (newest, oldest, subject/chapter, topic) and every filter (subject/chapter/topic/search, source, file, date range), plus "Your uploads" chips.
- [ ] New and generated questions open in the review pop-up (newest on top; Approve / Edit / Discard, Approve selected, Approve all). `/questions/review` opens the pop-up.
- [ ] A test has one subject and at most 15 questions. Step 2 auto-picks up to 4 topics from unlocked chapters (weak, then current chapter, then last 14 days), each with its reason, and the parent can change them.
- [ ] Step 3 shows only the test's questions: remove, replace (same-topic PrepNivo question or the parent's own from the same subject), and one "Add from my questions" button. Other-subject and over-15 additions are skipped with a message.
- [ ] Olympiad papers: pick one from the catalogue or upload one (assigned after every draft is reviewed). Questions stay hidden from the parent until the child finishes. Drill-rate points. The tests list filters by type and subject.
- [ ] AI billing shows 50,000 tokens a month per active child, ₹79/₹249 top-ups (EUR placeholders marked TODO), automatic top-up off by default, a meter, history and the explainer. Every AI action shows its estimate and is blocked with "Buy tokens" when tokens run out. A notification appears at 80%. Top-ups create invoices.
- [ ] The avatar menu is exactly Account · Settings (→ Settings › Children) · Log out. The style guide isn't in any product menu.
- [ ] Days with several sessions show numbered "Session N · time" cards, and the day strip shows ×N.
- [ ] Child app: sign-in on a device-linked family (avatar, then picture code or PIN, with a 5-minute lock after 5 wrong tries); home; subject drills; challenge unlocks at ≥60%; next chapter unlocks at ≥70%; feedback only for drills and challenges; timer; confirmation before finishing with blanks; results write back (queued offline, synced online); buddy requests reach the parent; always light; no AI.
