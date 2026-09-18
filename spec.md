# Spec: Rosie's Dog Walking — Quote Calculator Page

## Goal

A single public web page for Rosie's Dog Walking whose only job is to produce an instant, correct, self-explanatory quote for a dog-walking visit. Customers use it themselves; Rosie opens the exact same page when a customer texts her and reads the answer back. No calculation is done by hand again.

## Pricing model (the rules the page encodes)

- **Base rate**: one standard per-walk rate for the first dog. Stored as a single constant at the top of the file, clearly labelled and easily editable (initial value $25 per walk is a placeholder to be confirmed with Rosie).
- **Additional dogs**: the 2nd and each further dog cost 50% of the base rate. The 2nd dog is discounted, not free.
- **Same-day booking**: a flat surcharge added once per quote. Stored as a single editable constant (initial $5 is a placeholder to be confirmed).
- **Holiday**: the total for the dogs is multiplied by 2 exactly ("double").
- **Aggressive or reactive dog**: a flat surcharge added once per booking when at least one dog in the booking is marked aggressive or reactive. Stored as a single editable constant (initial $10 per booking). It is not charged per dog and is not doubled on holidays; it is added after the holiday doubling.
- **No stacking**: at most one day-based modifier (same-day surcharge OR holiday double) ever applies to a quote. They never combine. Priority: **holiday double wins over same-day surcharge**. The multi-dog half-price pricing is treated as the base rate structure, not a stacking discount, so it always applies. The aggressive/reactive surcharge is not a day-based modifier and may apply alongside either day-based modifier.
  - *Note on interpretation*: "no stacking" is read as "the two day-based modifiers never compound." This and the two placeholder amounts must be confirmed with Rosie before build.

Quote formula (in words): `(base rate + 50% of base rate for each dog beyond the first)` then apply exactly one of {holiday ×2, same-day surcharge, nothing}, then add the aggressive/reactive surcharge once if any dog is marked aggressive or reactive.

## Client-request: #1

> "Can we add a $10 surcharge whenever at least one dog in the booking is marked aggressive or reactive? It's a flat $10 per booking no matter how many aggressive dogs are in it, not per dog — and it doesn't double on holidays, it just gets added on top after the holiday doubling. Had a bad one last week."

This issue adds a single booking-level surcharge for any booking where at least one dog is marked aggressive or reactive. The surcharge is flat once per booking, is not charged per aggressive dog, and is added after holiday doubling so it is not doubled on holidays.

## User scenarios

1. **Customer, on their phone** — taps a link, sees short friendly copy and a small form: number of dogs, walk date, and whether it's today (same-day). As they change any choice the price updates immediately. They read the number, understand how it was reached from the breakdown, and text or call Rosie. No login, no account.
2. **Customer, on desktop** — same page, same behaviour; uses it to check a future date (e.g., a holiday week) without needing to ask.
3. **Rosie, during a text conversation** — opens the same page, enters the dog count and date the customer described, reads the quote back into her reply. She trusts it because the breakdown shows the same-dog discount, the same-day surcharge, or the holiday doubling explicitly.
4. **Rosie, sanity check** — 2 dogs on a normal day: confirms the breakdown shows "second dog 50% off" and a total of 1.5 × base, so no hand-math is needed.

## Functional requirements

Quote correctness and the calculation pipeline:

- FR-1 The quote equals `base` for 1 dog on a normal, booked-ahead day.
- FR-2 For N dogs, the quote is `base + (0.5 × base) × (N − 1)` before any modifier.
- FR-3 A same-day booking adds the same-day surcharge exactly once per quote, regardless of dog count.
- FR-4 A holiday date multiplies the dog subtotal by exactly 2.
- FR-5 Same-day surcharge and holiday double never combine. If a date is same-day AND a holiday, only the holiday double applies and the surcharge is omitted.
- FR-6 All money is handled in cents; the displayed total is always formatted to 2 decimal places. Every intermediate percentage (e.g., 50%) is computed as integer cents. Round increments adopt the standard half-up convention; no step may show more than 2 decimals or produce floating-point artifacts.
- FR-7 The dog subtotal, any modifier, and the final total are computed from a single set of constants (base rate, second-dog discount, surcharge) defined once at the top of the file, so no number is duplicated.

Instant, self-explanatory behaviour:

- FR-8 Editing any input recomputes the total immediately, with no submit/refresh. Default state is 1 dog, a normal (non-holiday), booked-ahead date, showing the base quote.
- FR-9 The page always shows a line-item breakdown: first dog, additional dogs (with "$12.50 2nd dog (50% off)" style labels), and the applicable modifier line (same-day surcharge or holiday double) — or an explicit "no extra fees" line when neither applies.
- FR-10 When a day-based modifier is skipped because another one takes priority (FR-5), the page shows a one-line note, e.g. "Holiday walk — price doubled; same-day surcharge not added."
- FR-11 A date is a holiday if and only if it appears in a small local list of holiday dates kept as a constant in the file (no internet, per the constitution). A date not in the list is a normal day. The same-day condition is: chosen date equals today's date.
- FR-12 Dog count is a numeric input with a minimum of 1. Inputs of 0 or negative are rejected (clamped to 1); no quote below the single-dog base is reachable.
- FR-13 The page is one self-contained HTML file (embedded CSS/JS) that renders and recalculates correctly on phones and desktops, and makes no network requests.

Aggressive/reactive dog surcharge (FR-7 constant source):

- FR-14 When at least one dog in the booking is marked aggressive or reactive, a flat surcharge is added exactly once to the quote, regardless of how many dogs are marked aggressive or reactive.
- FR-15 The aggressive/reactive surcharge is added after the holiday doubling, so it is not itself doubled on holidays.
- FR-16 The page exposes a single boolean input ("Any dog aggressive or reactive?") defaulting to off; changing it recomputes the total instantly with no submit/refresh.
- FR-17 The breakdown lists the aggressive/reactive surcharge as its own line when it applies, or shows no extra line for it when it does not.

## Edge cases & rules resolution

- 1 dog, normal day → base only.
- 2 dogs, normal day → base × 1.5; 3 dogs → base × 2.0 (each extra dog half-price).
- Same-day, 1 dog → base + surcharge; same-day, 2 dogs → base × 1.5 + surcharge (surcharge flat, not per dog).
- Holiday, any dogs → dog subtotal × 2; the second-dog discount is computed first, then doubled.
- **Conflict** — holiday AND same-day → holiday wins, surcharge dropped, FR-10 note shown. This is the single tie-break rule and the only conflict the page must resolve (per the findings: one explicit precedence, committed once).
- Rounding — half-priced amounts are exactly representable in cents (e.g., $12.50 from $25); the total never displays more than 2 decimals. A percentage like 50% of an odd cent is rounded half-up.
- No zero-price/free outcomes — a quote can never be $0 or negative; the lowest reachable total is the base rate for 1 normal dog.
- Choice of day in the past is allowed and priced the same as a future date, unless it is today (same-day) or in the holiday list.

## Out of scope

- Any backend, database, login/account, booking, payment, or scheduling.
- Editing/creating the pricing or holiday list from the UI — those are constants at the top of the file that Rosie edits in the code.
- Rates varying by dog size, breed, walk length, mileage, or multiple walks per visit.
- Discount codes, promos, first-visit offers, referral rewards, subscriptions, or packages.
- Contact forms, analytics, SEO, multi-language, or anything requiring external calls.
- A separate "owner view" — Rosie and customers share the same page and the same numbers (the constitution principle).

## Acceptance criteria

Against the initial constants (base $25, surcharge $5):

1. Opening the file from disk in a browser with no build step renders the calculator; DevTools console shows no errors, and the Network tab shows zero requests. (FR-8, FR-13)
2. Default quote = $25.00. (FR-1, FR-8)
3. 2 dogs, normal day → $37.50; 3 dogs → $50.00; 4 dogs → $62.50. (FR-2)
4. Same-day 1 dog → $30.00; same-day 2 dogs → $42.50. (FR-3)
5. Holiday, 1 dog → $50.00; holiday, 2 dogs → $75.00. (FR-4, FR-7)
6. Holiday AND same-day, 1 dog → $50.00 (NOT $55.00), and the FR-10 note appears. (FR-5, FR-10)
7. Changing any input instantly updates both the breakdown and total, on phone-sized and desktop viewport, with no refresh. (FR-8)
8. Every displayed price has exactly 2 decimals (e.g., $25.00, never $25 or $25.0001). (FR-6)
9. Setting dog count to 0 or -1 yields the $25.00 single-dog quote, never an error or a negative price. (FR-12)
10. A date picked from the calendar that is in the local holiday list quotes double; the same date removed from the list quotes normal — proving the page has no hidden rules. (FR-7, FR-11)
11. The breakdown always sums exactly to the shown total, modifier lines included. (FR-9, FR-7)

Aggressive/reactive surcharge acceptance criteria:

- 12. 1 dog, normal day, no aggression → $25.00 (unchanged).
- 13. 2 dogs, normal day, no aggression → $37.50 (unchanged).
- 14. 1 dog marked aggressive/reactive, normal day → $35.00 ($25 base + $10 surcharge).
- 15. 2 dogs marked aggressive/reactive, normal day → $47.50 ($37.50 base + $10 surcharge).
- 16. Same-day, 1 dog, aggression → $40.00 ($30 same-day surcharge + $10 aggression).
- 17. Same-day, 2 dogs, aggression → $52.50 ($42.50 same-day + base 1.5 + $10 aggression).
- 18. Holiday, 1 dog, aggression → $60.00 ($50 holiday double + $10 aggression; surcharge not doubled).
- 19. Holiday, 2 dogs, aggression → $85.00 ($75 holiday double + $10 aggression).
- 20. Holiday AND same-day, 1 dog, aggression → $60.00 ($50 holiday double only; same-day omitted; + $10 aggression added after).