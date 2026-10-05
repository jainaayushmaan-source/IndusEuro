# DECISIONS.md

Assumptions and rulings not covered (or contradicted) by SPEC.md.

## Confirmed by Aayushmaan (5 Oct 2026)

1. **Grace-period banner names Anaya.** SPEC §5.8 said "Aarav"; §8 and CLAUDE.md Phase 7 say Anaya is in grace period. SPEC §5.8 corrected to Anaya.
2. **`--primary` and `--warning` are non-text colours.** In light mode, `--primary` (#2E9B63) fails 4.5:1 with white text and on `--bg`, and `--warning` (#B26A00) fails 4.5:1 on white. Both are used only for UI elements (borders, icons, focus rings, chart marks, fills without text), which need 3:1. They are never used for text.
   - Text that needs a green or amber tone (links, filled-button labels, warning labels) uses separate text-safe tokens defined in Phase 0 and checked by `pnpm check:contrast`.
   - The "Nivo" part of the logo wordmark keeps `--primary`: logos are exempt from WCAG contrast requirements.
