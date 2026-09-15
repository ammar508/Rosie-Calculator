# Findings: Rule-Based Pricing for an Instant Quote Calculator

## 1. How tiered/conditional pricing is usually structured

- **Rules as data, not code.** Represent each discount/surcharge as a small declarative object — `{ condition, effect, priority, appliesToBasis }` — evaluated by one small engine, rather than a hardcoded `if/else` chain. Rule engines separate rules from application code so they can be defined, tested, and changed independently (Wikipedia, "Business rules engine").
- **A deterministic mini-engine beats forward/reactive chaining.** Isolated `IF condition THEN effect` production rules and event-driven reaction rules suit stateful enterprise systems. A small quote calculator wants a deterministic, ordered pipeline (`deterministic engine` / DSL approach) — simpler and fully reproducible.
- **Two distinct tier models — don't mix them up.** *Threshold-tier* ("2–4 walks/day = $18/walk") grants one rate for the whole order based on a step table. *Progressive/sliding-scale* ("first 2 walks at $20, each additional at $15") prices marginal units in each bracket separately. The same step table can mean either; you must pick one per rule. Distinguish cumulative (per-period volume) vs non-cumulative (per-quote quantity) quantity discounts (Wikipedia, "Discounts and allowances").
- **Waterfall / price fences.** Stacked rules are conventionally a fixed sequence of layers — list, then volume, then promo, then negotiated — each typically calculated on the net of the previous (or on a fixed base). Express every rule's *basis* (pre- or post-discount) and order explicitly (UsagePricing, "Discount Waterfall"; Umbrex "List-Discount-Rebate Architecture").

## 2. Evaluating multiple rules without silent conflicts

- **Assign an explicit priority, then sort.** Rule engines natively carry priority/score; output must be order-independent of insertion order, so rules evaluate in deterministic priority order. First-match-wins (ordered rules) or best-match-wins (match on all, then pick highest priority) are the two common strategies — pick one.
- **Decide the stacking policy per pair, not globally.** Allow/disallow matrices ("which two rules may combine") prevent most conflicts at compose time. Gate eligibility (who qualifies) separately from combinability (which rules stack).
- **Resolve tie-breaks by a single stated convention**: most specific rule wins, best-for-customer wins, or best-for-business wins — whichever you choose, make it one line and test it.
- **Cap the combined depth.** A max-discount ceiling (e.g., "never more than 30% off the base quote") or a floor on the result prevents stacked rules compounding past the intended saving. This is a hard guardrail, not a preference.
- **Commit once to the basis** — thresholds like "free shipping over $X" are evaluated pre-discount OR post-discount, never left implicit; the mismatch between them is a classic silent bug.

## 3. Typical edge cases

- **Rounding.** Work in cents (integers); do percentage math as integer cents, not floating point. Decide once: round each line, or round only the final total — rounding per step lets errors accumulate. Pick a mode (half-up for money is the common convention) and apply it consistently.
- **Order of operations.** Whether a percentage applies to the base before or after a fixed surcharge, and whether surcharges apply to discounted or undiscounted amounts, changes the result materially. The waterfall order *is* the specification.
- **Which rule wins when two apply.** Threshold boundary inclusivity (`>=` vs `>`) and tier boundaries are off-by-one factories. Two rules matching at the same priority level must have a deterministic tie-break, or the result depends on object/array ordering.
- **Basis ambiguity on second-order features.** Free-gift/free-service unlocks, free shipping, and loyalty perks all have the pre- vs post-discount basis problem; the "free" line must also be marked ineligible for other percentage discounts.
- **Degenerate inputs.** Zero/negative quantities, coupons that push a line to zero, discounts exceeding the value of the walk itself, and multiple rules producing the same tier must all resolve deterministically.

## 4. Failure modes to watch for

- **Composition bugs over logic bugs** — the dominant failure. Every stacking failure reported in production was two individually-correct rules whose *interaction* broke a margin corner on a minority of carts (RdyGo, "How to Stack Shopify Discounts Without Breaking Margin"). Each rule passes review; the combination doesn't.
- **No precedence = nondeterministic quotes.** For an instant quote calculator, this is the worst failure: the same inputs return different prices depending on internal ordering. Exactly what a calculator must *never* do.
- **Double-dipping / discount on discount** — e.g., a tier rate applied on a base that already contains a bundle saving, compounding past the intended ceiling and producing a negative or near-zero price. Bounded by a floor.
- **Hidden basis mismatch** (pattern 2 and 5 in the RdyGo source): thresholds evaluated on a different base than the one marketing described leads to confusing, support-ticket-generating quotes.
- **Floating-point drift** on repeated percentage stacking (10% of 10% of 10%…) producing prices like `$23.59999999`.
- **Tightly coupled if/else chains** — the opposite failure: correct today, brittle to tomorrow's "one more discount" because the sequencing logic is smeared across the app rather than stored with the rules.
- **Unlabelled basis flip during refactors** — someone rewrites "apply discount to base" to "apply to net" to fix one rule and silently changes every stacked rule downstream.

**Bottom line for a small calculator:** one ordered list of declarative rules with explicit priority, an allow-list of stackable pairs, a single pre/post basis committed globally, integer-cents math, and a final cap/floor. That small harness eliminates nearly every failure mode above.