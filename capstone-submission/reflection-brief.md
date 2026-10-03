# Reflection Brief — Evaluation and Observability Capstone

**Name:** Neeharika Nanjarapalli
**Date:** 2026-10-03

> Ground every answer in your own run. When a question asks for a number, file name, or line, paste
> it from your artifacts — a reviewer should be able to find it. Answers that are correct in the
> abstract but cite nothing do not meet the bar. Keep it short and specific.

---

## 0. Environment

| Field | Value |
|---|---|
| OS & version | Linux 6.6.97+ x86_64 (Vocareum workspace container) |
| Python version | 3.13.0 |
| Date run | 2026-10-03 |
| Ran any system live? (which) | Yes — System 1 (policy pipeline): full `pipeline` run, live test suite, and perturbation run all hit the real API via `claude.vocareum.com`. System 2 (mortgage extraction): the perturbation run used `--mode record` against the live API. System 3 (supply chain) ran entirely offline (`--offline`) per the course's offline-first design for that project. |

---

## 1. Validated, routed pipeline

| Evidence | Value |
|---|---|
| Passing test count | 48 passed, 0 failed (`01-policy-pipeline/tests.txt`) |
| Routing output file | `01-policy-pipeline/routing_decisions.json` |
| auto_approve / human_review / spot_check counts | 0 / 8 / 1 |

**1a. Retry boundary.** From your perturbation run (a required field removed), paste the escalation
record. How many API calls did the system make, and why is retrying a futile case worse than
escalating it?

> From `01-policy-pipeline/perturbation-run.txt`, blanking the `PREMIUM SUMMARY` section of a copied
> policy (`POL-2025-999`) produced exactly **one** API call (`HTTP/1.1 200 OK`, logged once), followed
> immediately by escalation:
> ```json
> {
>   "category": "missing_source",
>   "detected_pattern": "premium_amount_absent",
>   "field": "premium_amount",
>   "kind": "escalation",
>   "reason": "Field 'premium_amount' returned null — the source document does not contain this information. Retry is futile; escalate to human review."
> }
> ```
> Retrying a `missing_source` failure is worse than escalating because the defect isn't in the
> model's reasoning — it's in the source document. No amount of re-prompting conjures a premium figure
> that was never written down. Retrying anyway would burn additional API calls and, worse, delay the
> human reviewer who can actually resolve it (e.g., contacting the insured for the missing figure),
> manufacturing an illusion that "the system tried hard" while making the real fix slower to reach.

**1b. Reading the router.** Pick one `human_review` record from your routing output. Which of the
three signals (confidence, reviewer, integration) sent it to a human? If you had trusted the model's
confidence alone, what would have happened?

> `POL-2025-002` in `01-policy-pipeline/routing_decisions.json` has `confidence_summary` pinned at the
> ceiling across every field (`coverage_limit: 1.0`, `deductible: 0.95`, `endorsements: 1.0`,
> `exclusions: 1.0`, `policy_type: 1.0`, `premium_amount: 1.0`) and an empty `fields_below_threshold`.
> By confidence alone this looks like a textbook auto-approve. But `reviewer_disagreements` lists
> `["coverage_limit", "deductible"]`, and the record's `reason` field reads
> `"reviewer_disagreement=['coverage_limit', 'deductible']"` — **reviewer disagreement**, not
> confidence, is what sent it to `human_review`. Had the system trusted confidence alone, `POL-2025-002`
> would have been auto-approved with two disputed fields nobody ever checked. This pattern repeats
> across the batch: `POL-2025-003`, `004`, `006`, `008`, and `010` all show confidence ≥ 0.9 everywhere
> with empty `fields_below_threshold`, yet every one was still caught by reviewer disagreement —
> confidence was essentially uninformative in this run; the independent review pass did the real work.

**1c. Where the aggregate lies.** Run the calibration snippet. Quote the one cell whose accuracy lags
its confidence, plus the overall figure. What does slicing by `policy_type × field` catch that a
single number hides?

> From `01-policy-pipeline/calibration-report.txt`:
> ```
> umbrella   exclusions      n=2 conf=0.93 acc=0.00 brier=0.865
> OVERALL brier=0.291
> ```
> The `umbrella / exclusions` cell is confidently wrong on both of its two samples — `conf=0.93` but
> `acc=0.00` — while `OVERALL brier=0.291` reads as merely moderate and gives no hint that one cell is
> completely broken. Slicing by `policy_type × field` catches a systematic failure mode tied to a
> specific combination (this model type, this field) that a single aggregate number averages away
> entirely; an operator reading only the overall figure would never know to distrust umbrella
> exclusions specifically.

---

## 2. Schema-enforced two-pass extraction

| Evidence | Value |
|---|---|
| Passing test count | 25 passed, 0 failed (`02-mortgage-extraction/tests.txt`) |
| Document run | `fixtures/documents/income_sum_mismatch.txt` |
| Classified type | `income_verification` |

**2a. Two guarantees.** Paste your discrepancy-run output. Tool use already forces valid JSON, yet the
validator still catches a bad sum. Why are these two different guarantees? Name one error each cannot
catch.

> From `02-mortgage-extraction/discrepancy-run.txt` (`income_sum_mismatch.txt`, classified
> `income_verification`):
> ```json
> "income": { "base_monthly": 5416.67, "bonus_monthly": 1250.0, "commission_monthly": 2140.0,
>             "overtime_monthly": 385.5, "other_monthly": 450.0, "stated_monthly_total": 10892.17 },
> "validation": {
>   "consistent": false,
>   "discrepancies": [ { "field": "total_monthly_income", "calculated": 9642.17,
>                         "stated": 10892.17, "delta": -1250.0 } ]
> }
> ```
> The extraction itself is perfectly well-formed JSON — forced tool-use guarantees *shape*: every
> field present, correctly typed, enum values valid. It cannot know whether `5,416.67 + 2,140.00 +
> 1,250.00 + 385.50 + 450.00` actually equals the stated `10,892.17` (it sums to `9,642.17`) — that's
> an *arithmetic* guarantee, which only the separate consistency validator checks. An error the schema
> alone can't catch: a plausible-but-wrong single field (e.g., a misread `base_monthly` that still
> happens to sum correctly with the rest). An error the validator alone can't catch: a value that sums
> correctly but is still substantively wrong — e.g., the model misreads `commission_monthly` as
> `1,140.00` instead of `2,140.00` and adjusts `other_monthly` to keep the total consistent; the
> arithmetic checks out, and nothing catches that this read the paystub wrong.

**2b. Refusing to fabricate.** Run on a document missing a field. Paste that field's output. Why null
instead of an invented value? Point to the schema choice that allows it.

> From `02-mortgage-extraction/extract-run.txt` (`income_missing_bonus.txt`, classified
> `income_verification`):
> ```json
> "income": { "base_monthly": 5673.08, "bonus_monthly": null, "bonus_ytd": null, ... }
> ```
> `base_monthly` extracted cleanly while `bonus_monthly` came back `null` rather than some plausible
> guessed figure. This is possible because the extraction schema defines income fields as nullable
> unions (exercise 01's "resilient extraction schema" design) rather than requiring every field to
> carry a value — the model has an explicit, valid "this isn't stated" option instead of being forced
> to produce a number it cannot support. Returning `null` here is the honest signal that underwriting
> needs: a fabricated bonus figure would silently corrupt a debt-to-income calculation with no way to
> tell it apart from a real one.

**2c. Normalization.** Quote one field where the source text and extracted value differ in format
("about 2,400 sq ft" → `2400`). Why normalize at extraction time rather than downstream?

> From `02-mortgage-extraction/extract-run.txt` (`appraisal_informal_sqft.txt`, classified
> `appraisal`): the source document describes the living area informally, and the extraction returns
> `"gross_living_area_sqft": 2400` — a clean integer. Normalizing at extraction time means every
> downstream consumer (underwriting calculations, comparisons across documents, consistency checks)
> can treat the field as a typed number immediately, with one normalization rule enforced once at the
> point closest to the source text (where the model still has the original informal phrasing in view
> to interpret correctly). Deferring normalization downstream would mean every consumer re-implements
> its own parsing of "about 2,400 sq ft"-style strings, multiplying the chance of inconsistent or
> wrong parses across the system.

---

## 3. Multi-source synthesis

| Evidence | Value |
|---|---|
| Passing test count | 34 passed, 0 failed (`03-supply-chain/tests.txt`) |
| Briefing file | `03-supply-chain/briefing.md` |
| Section the conflict landed in | Contested |

**3a. Annotate, don't arbitrate.** Quote one conflicting-metric pair from your briefing — both values,
sources, dates. Give one way a reader is better served by the preserved conflict than by a single
reconciled number.

> From `03-supply-chain/briefing.md`, under **Contested**:
> ```
> ### on_time_delivery_rate  _[2 sources, conflicting]_  ⚠️ ESCALATE
> - 95.0 percent — supplier_audit (as of 2026-04-10)
> - 78.0 percent — logistics (as of 2026-04-05)
> ```
> A reader deciding whether Meridian can reliably hit a committed ship date is far better served
> seeing both 95% and 78% side by side than a single blended "~86.5%" figure. The 17-point gap itself
> is the signal — it suggests `supplier_audit` and `logistics` may be measuring genuinely different
> things (e.g., contractual terms vs. actual tracked deliveries), which is exactly the kind of
> discrepancy a procurement team needs to investigate before trusting either number, not something a
> false average should paper over.

**3b. Source goes dark.** Run with `--simulate-timeout`. Paste the part of the briefing showing the
failed source. How is "unreachable" handled differently from "nothing to report," and why does the run
still finish?

> From `03-supply-chain/timeout-run.txt`:
> ```
> > Sources unavailable: logistics unavailable (timeout)
> ...
> ### late_shipment_count  _[missing source: timeout reading logistics]_
> - missing source: timeout reading logistics
> ```
> This is explicitly labeled a *timeout* — a source that was expected to answer and didn't — versus
> `production_capacity_utilization`, which appears in both the clean and timeout runs labeled
> `[missing source: no source reported this metric]` — a metric no source ever covers at all. The
> briefing distinguishes "unreachable right now" from "nothing exists here," which matters operationally:
> the first is worth a retry or an escalation to check the source's health; the second means the data
> was simply never collected. The run still finishes because the coordinator treats a single source
> failure as a bounded, annotated gap rather than a fatal error — the other three sources (supplier_audit,
> internal_quality, industry_news) still have their findings synthesized into a complete briefing.

**3c. Dates as a guardrail.** Quote two claims about the same supplier with different dates. How does
requiring a date stop a time difference from reading as a contradiction?

> `supplier_financial_distress` is dated `2026-03-09` and `port_disruption` is dated `2026-03-17` —
> eight days apart, clearly two distinct events in sequence rather than a disagreement. Contrast that
> with the `on_time_delivery_rate` pair from 3a: `supplier_audit`'s 95.0% is dated `2026-04-10` and
> `logistics`'s 78.0% is dated `2026-04-05` — only five days apart, which rules out "the delivery rate
> simply changed in the interim" as an explanation and confirms this is a genuine same-period
> disagreement between two sources, not a stale-vs-fresh timing artifact. Without a date on every
> claim, a reader couldn't tell "these two sources disagree about the same moment" apart from "the
> world changed between these two reports" — the date is what lets a real conflict be told apart from
> an ordinary sequence of events.

---

## 4. Synthesis

**4a. One principle.** Name the single moment in your runs (system + artifact) where *evaluate the
output, don't trust the model's word* most clearly caught something a trusting design would have
shipped.

> System 1, `01-policy-pipeline/routing_decisions.json`, record `POL-2025-002`: every confidence score
> the extractor reported was at or near 1.0, with zero `fields_below_threshold`. A design that trusted
> the model's own stated confidence would have auto-approved this policy outright. Instead, the
> independent reviewer — a separate model call with no visibility into the extractor's reasoning —
> disagreed on `coverage_limit` and `deductible`, and that disagreement alone routed the record to
> `human_review`. The model's self-reported confidence was not just unhelpful here, it was actively
> misleading, and only an independent evaluation pass caught it.

**4b. Confidence ≠ correctness.** Pick the system where this mattered most, and explain why using
something you observed.

> System 1. Across the nine surviving records in `routing_decisions.json`, eight of nine show
> confidence ≥ 0.9 on every field with an empty `fields_below_threshold`, yet eight of nine were still
> routed to `human_review` by reviewer disagreement. If confidence were a reliable proxy for
> correctness, this batch would have produced far more `auto_approve` decisions than it did (it
> produced zero — the one clean record, `POL-2025-007`, was still pulled into `spot_check` by the
> stratified sampler rather than silently auto-approved). The calibration report
> (`01-policy-pipeline/calibration-report.txt`) sharpens this further: `umbrella/exclusions` sits at
> `conf=0.93, acc=0.00` — confidence and correctness aren't just imperfectly correlated in this system,
> they're inverted in that specific slice.

**4c. Apply it.** Describe a real workflow where an LLM pulls structured results from messy input.
Which pattern — validated retry with escalation, independent review with deterministic routing, or
provenance-preserving conflict annotation — would you reach for first, and what would you instrument
to know when it broke?

> Consider an accounts-payable workflow that extracts invoice line items (vendor, amount, due date,
> PO number) from scanned vendor invoices to feed an ERP system. I'd reach for **validated retry with
> escalation** first: invoices have hard arithmetic relationships (line items must sum to the invoice
> total, same as System 2's `total_monthly_income` check) and a clear missing-vs-malformed distinction
> (a PO number that's genuinely absent from the invoice vs. one the model misread). I'd instrument the
> same signals this capstone surfaced: the `category` breakdown of failures (format/consistency vs.
> missing_source) to track whether failures are recoverable-and-improving or genuinely-absent-and-
> stable over time, the discrepancy rate from the consistency validator as a leading indicator of
> extraction quality drift, and — borrowing from System 1's calibration report — a sliced calibration
> view by vendor/field to catch whether the model is confidently wrong on a specific vendor's invoice
> format before that silently corrupts a batch of payments.
