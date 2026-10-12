# DECISIONS.md

Assumptions and rulings not covered (or contradicted) by SPEC.md.

## Confirmed by Aayushmaan (5 Oct 2026)

1. **Grace-period banner names Anaya.** SPEC §5.8 said "Aarav"; §8 and CLAUDE.md Phase 7 say Anaya is in grace period. SPEC §5.8 corrected to Anaya.
2. **`--primary` and `--warning` are non-text colours.** In light mode, `--primary` (#2E9B63) fails 4.5:1 with white text and on `--bg`, and `--warning` (#B26A00) fails 4.5:1 on white. Both are used only for UI elements (borders, icons, focus rings, chart marks, fills without text), which need 3:1. They are never used for text.
   - Text that needs a green or amber tone (links, filled-button labels, warning labels) uses separate text-safe tokens defined in Phase 0 and checked by `pnpm check:contrast`.
   - The "Nivo" part of the logo wordmark keeps `--primary`: logos are exempt from WCAG contrast requirements.

## Prototype rounds 1–2 (5–6 Oct 2026)

These were agreed in the prototype chat but never committed. On 11 Oct 2026 they were rebuilt from `prototype/prepnivo-prototype.html`, which is the source for the exact values.

3. **Added colour tokens.**
   - Text and controls: `--primary-strong` (light #1F7046 / dark #4CC48A) for green text, links and filled-button backgrounds; `--border-strong` (#7D9286 / #6E857A) for input and control borders at 3:1.
   - Status and overlays: `--warning-soft`, `--danger-soft`, `--on-danger`, `--overlay`, `--shadow` (none in dark mode).
   - Charts and logo: `--chart-1…4` (a separate dark palette); `--logo-mark-1/2`, `--logo-text` and `--logo-accent`.
   - All of them are checked by the contrast script.
4. **Curriculum gaps.** Only Grade 6 (IN) and Klasse 3 (DE) have full chapter and topic data.
   - Topic names inside chapters were filled in by the prototype and are marked `verify: true` for the curriculum team.
   - Other classes show a standard subject list with "Chapters to be confirmed". AI generation and tests are unavailable for subjects with no chapters.
5. **Point values.** Points per correct answer: drill 10, parent test 10, Olympiad paper 10, challenge (platform or parent-made) 20. These are placeholders in one config, `RATES`.
6. **Metric formulas.** Every screen uses these single selectors:
   - Topic mastery = accuracy × min(1, attempts / 10).
   - A topic is **weak** with ≥ 5 attempts and < 60% accuracy. Severity = (100 − accuracy) + min(attempts, 20) + 15 if wrong in the last 3 days (+8 if within 7).
   - A chapter is **passed** when every topic has been practised and accuracy is ≥ 60%.
7. **Security limits.**
   - Parent login: 5 wrong passwords lock that email for 15 minutes.
   - OTP: at most 5 sends per email per hour.
   - Idle logout after 30 minutes.
   - Reset links are single-use and valid for 1 hour, and a reset signs out every device.
   - A password can't contain the email's local part (when that part has 4+ characters).
8. **Onboarding limits.**
   - At most 6 children per account.
   - The child's age must be 4–17. An age/class mismatch over 1 year is a warning only.
   - No duplicate first name + date of birth.
   - Picture code: 3–6 animals.
   - Username: `[a-z0-9.]{3,20}`, unique in the family.
   - Reject easy PINs (repeated digits, 1234, 4321, 1212, 0123, 2580, 9876) and the birth year.
9. **Unfinished sign-ups and carts.**
   - Logging in with an unconfirmed sign-up resends the sign-up code.
   - "I'll do this later" on `/onboarding/start` logs the parent out with a notice.
   - Unpaid cart items are kept and shown as an alert on Overview.
10. **Checkout outcomes.** The mock covers success, 3-D Secure, declined, insufficient funds, expired card, authentication failed and connection lost (`/checkout/pending`). An already-paid checkout can't be paid twice. The "Test scenario" picker on the mock is for prototype review only.
11. **One API layer for network behaviour.**
    - Every server-backed action goes through one function, which handles offline, slow and server errors.
    - Actions are disabled while running and keep their input on failure.
    - Lists have offline, error and "Try again" states.
    - The server error page shows a reference code (`PN-XXXXXX`).
12. **Delete account.** Lives in a danger zone in Settings › Account.
    - The parent types `DELETE`, then enters an email OTP.
    - All subscriptions are cancelled with no refund for the current period, and all child data is deleted.
    - The flow ends on `/goodbye`.
13. **Prototype tools are not product.** None of these is built into the app:
    - the prototype's "Prototype tools" panel (simulate offline, slow network, failing requests, expire session, AI usage at 95%, end grace period, reset data)
    - the demo-account "Fill in" note on login
    - the visible OTP in the dev note

    The real dev fallback is still "Dev mode: code in terminal" (SPEC §5.1).

## Prototype round 3 (9 Oct 2026), SPEC §5.11

14. **The child app is in scope.**
    - SPEC §1 used to say the child side was out of scope. The child app at `/kid` is now part of the build, frontend-only, in the same codebase.
    - The hard rules still hold: no AI for children, no PIN/picture sign-in for parents, and the child app is always light theme.
    - The "parent side only" lines in CLAUDE.md and AGENTS.md mean "parents' features"; they don't exclude `/kid`.
15. **§5.11 replaces earlier task text.**
    - The topics tree, the three question sources and "+ Add more questions" in SPEC §5.6, CLAUDE.md Phase 5 and AGENTS.md Phase 5 are replaced by §5.11.5.
    - The tabs and review page in §5.7 and Phase 6 are replaced by §5.11.2–4.
    - SPEC wins.
16. **Parents never browse the PrepNivo question bank.**
    - Parents see PrepNivo questions only inside a test they are building (after auto-pick, replace or retest) and in completed tests.
    - Olympiad paper questions stay hidden until the child finishes.
17. **How topics are auto-picked.**
    - Candidates come in this order: weak topics in the subject → the newest unlocked chapter's topics → topics practised in the last 14 days → other unlocked chapters.
    - De-duplicate them, and keep only topics that have PrepNivo or parent questions. Stop at 4.
    - If nothing qualifies, use the current chapter's topics.
    - Topics the parent adds (in the tree, or by adding or replacing a question) get the reason "Chosen by you".
18. **How questions are filled.**
    - Go round-robin across the topics, preferring the chosen difficulty (mixed = no preference).
    - Then fill from the parent's own questions in the same subject, newest first.
    - Removed or replaced PrepNivo questions aren't picked again for that draft.
    - A retest skips the questions from the original test.
19. **Olympiad paper details.**
    - The catalogue is placeholder content (SPEC §8).
    - Uploaded papers take 10–180 minutes and are split into sections by subject.
    - Retest isn't offered for papers.
    - A parent can remove a test or paper that hasn't been started.
20. **Rules for AI tokens.**
    - **Estimates:** 450 tokens per question (×1.2 when matching a weak topic), 2,600 per photo, 6,000 per PDF, 12,000 per complete paper, and about 4 tokens per word of pasted text (minimum 800). The recorded charge is the estimate ±10%.
    - **Allowance:** the month runs from the 1st, and only active and grace-period children count.
    - **Top-ups:** the currency follows the parent's country, and a card for that currency is required.
    - **Automatic top-up:** buys the smallest pack.
    - This is not a separate AI plan, so it is consistent with SPEC §2 anomaly 10.
21. **Child sign-in.**
    - The device's family is the last parent account that logged in on it.
    - The PIN stays 4 digits (SPEC §5.2).
    - The 5-tries / 5-minute lock applies to each child on the device.
22. **Child play rules.**
    - A drill has 10 questions; a challenge has 8 and earns double points.
    - A challenge unlocks at a best drill score of ≥ 60% (or when the chapter is passed).
    - The next chapter unlocks at ≥ 70% on the newest unlocked chapter's drill.
    - Quitting a test sends it back to Assigned and discards its answers.
    - **Answer matching:**
      - Numbers match numerically (a comma counts as a decimal point).
      - Text is matched after normalising case, spaces and ×/x.
      - Short answers without digits pass when ≥ 60% of their keywords are present.
23. **Buddies.**
    - Friend codes look like `ABC-1234` (first 3 letters of the name + 4 digits).
    - Approved buddies can see each other's badges and streaks.
    - Declined requests can be made again.
24. **Profile menu target.** Settings in the avatar menu opens Settings › Children. `/styleguide` stays as a development route (Phase 0) but isn't linked from any product menu.

## Prototype round 4 (11 Oct 2026), SPEC §5.12

25. **Data-art palette.** Six tokens, `--viz-1…6`, one per subject slot in curriculum order.
    - Light uses deep tones: #0F6E56 / #185FA5 / #854F0B / #534AB7 / #993C1D / #993556.
    - Dark uses light tones: #5DCAA5 / #85B7EB / #FAC775 / #AFA9EC / #F0997B / #ED93B1.
    - Measured contrast: 5.5–7.0 (light) and 7.3–11.4 (dark) on `--bg` and `--surface`.
    - Classes with more than 6 subjects reuse colours; the hover card names the subject.
26. **When the bloom appears.** Automatically once per app load, on the first Overview visit, so it never interrupts normal navigation. "Family view" reopens it at any time. It never appears in the child app.
27. **The bloom replaces the earlier in-page hero.** The first draft put a particle cloud inside the Overview page. It was replaced by the full-screen bloom (feedback: "a whole-screen artifact, nothing else on the screen"). Overview keeps the same visual language through particle mastery rings on its subject cards.
28. **Making petals easy to select.**
    - Each subject is a separate petal with its own colour and a gap around it.
    - Hover picks the nearest petal particle within 20 px.
    - The rotation eases to a stop while hovering, and the other petals dim.
29. **The bloom is part of the Overview page, not a pop-up (12 Oct 2026).**
    - The flowers are always one scroll up from anywhere on Overview, and scrolling down morphs them into the 8-week mastery chart.
    - Only the first Overview visit per app load starts at the flowers; later visits start at the chart, so normal navigation isn't slowed down.
    - The sidebar and top bar are hidden while the flowers fill the screen and fade in as the chart lands.
30. **Nothing clickable overlaps (12 Oct 2026).**
    - The 8-week chart uses one lane per subject, so lines never cross and each lane is a full-height target.
    - On the flowers, hover picks the front-most petal near the pointer. Petals respond only while the flowers are on screen, and lanes only once the chart has landed.
    - The prototype tools sit in the sidebar foot (or the bottom-left corner with page padding).
    - The offline banner is in the page flow, not floating.
    - Show password sits beside the field.
    - An automated audit of every parent, child, public and login screen at 1440 and 1280 found no remaining overlaps, apart from the accepted cases in SPEC §6.
