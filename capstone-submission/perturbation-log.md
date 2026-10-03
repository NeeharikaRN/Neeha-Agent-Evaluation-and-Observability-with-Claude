# Perturbation Log

For each system, make one deliberate change to an input or configuration, predict the outcome, run
it, and record what actually happened. See the starters in the Instructions, or design your own (your
own experiment earns more credit).

---

### System 1 — validated, routed pipeline

- **Change I made (file + what I changed):** Copied `data/policies/POL-2025-001.txt` to
  `data/policies/POL-2025-999.txt` and deleted the entire `PREMIUM SUMMARY` block (including the
  `Total Policy Premium` line and `Payment Plan` line), so no premium figure appears anywhere in the
  document. `premium_amount` is a required field.
- **Command I ran:**
  `.venv/bin/policy-extractor extract data/policies/POL-2025-999.txt --policy-id POL-2025-999`
- **What I predicted:** `premium_amount` would extract as `null`, the validator would classify it as
  `missing_source` (not `format` or `consistency`), and the system would escalate immediately with a
  single API call rather than retrying.
- **What actually happened (paste the key output line):** Exactly one `HTTP/1.1 200 OK` logged, then:
  ```json
  {
    "category": "missing_source",
    "detected_pattern": "premium_amount_absent",
    "field": "premium_amount",
    "kind": "escalation",
    "policy_id": "POL-2025-999",
    "reason": "Field 'premium_amount' returned null — the source document does not contain this information. Retry is futile; escalate to human review."
  }
  ```
  (full capture: `01-policy-pipeline/perturbation-run.txt`)
- **How this differs from the unperturbed run:** The original `POL-2025-001.txt` (see
  `01-policy-pipeline/routing_decisions.json`) made it all the way through extraction, independent
  review, and integration before landing in `human_review` due to `reviewer_disagreement` on
  `coverage_limit`, `deductible`, and `endorsements`. The perturbed version never reaches the reviewer
  at all — it halts at the extraction/validation stage as a `RetryFutileEscalation`, a fundamentally
  earlier and different failure path triggered by genuinely missing information rather than a
  disputed extraction.

---

### System 2 — schema-enforced two-pass extraction

- **Change I made (file + what I changed):** Copied `fixtures/documents/income_sum_mismatch.txt` to
  `fixtures/documents/income_sum_corrected.txt` and corrected the stated `TOTAL MONTHLY EARNINGS` and
  `Gross Earnings (Monthly)` figures from `10,892.17` to `9,642.17` — the actual sum of the five
  line items (`5,416.67 + 2,140.00 + 1,250.00 + 385.50 + 450.00`). No other field was touched.
- **Command I ran:**
  `.venv/bin/mortgage-extract fixtures/documents/income_sum_corrected.txt --mode record --model claude-haiku-4-5-20251001 -v`
  (`--mode record` and an explicit model were required since this edited text has no cached replay
  response.)
- **What I predicted:** The validator would report `consistent: true` with no discrepancies, since the
  stated total now matches the calculated sum.
- **What actually happened (paste the key output line):**
  ```json
  "income": { "...": "...", "stated_monthly_total": 9642.17 },
  "validation": { "consistent": true, "discrepancies": [] }
  ```
  (full capture: `02-mortgage-extraction/perturbation-run.txt`)
- **How this differs from the unperturbed run:** The original `income_sum_mismatch.txt` run reported
  `"consistent": false` with `calculated: 9642.17, stated: 10892.17, delta: -1250.0`
  (`02-mortgage-extraction/discrepancy-run.txt`). Same borrower, same line items, same classification
  (`income_verification`) — only the one planted arithmetic error was removed, and the validator's
  verdict flipped cleanly. This shows the validator reacts to the actual arithmetic relationship
  between fields, not to some fixed property of "this is the broken document."

---

### System 3 — multi-source synthesis

- **Change I made (file + what I changed):** No file edit — used the system's own fault-injection
  flag to force the `logistics` source to fail mid-run.
- **Command I ran:** `.venv/bin/supply-chain-investigate meridian --offline --simulate-timeout`
- **What I predicted:** The run would still complete rather than crash, the failed source would be
  explicitly annotated, and any metric previously sourced only from `logistics` would move to the
  Incomplete section.
- **What actually happened (paste the key output line):**
  ```
  > Sources unavailable: logistics unavailable (timeout)
  ...
  ### late_shipment_count  _[missing source: timeout reading logistics]_
  - missing source: timeout reading logistics
  ```
  (full capture: `03-supply-chain/timeout-run.txt`)
- **How this differs from the unperturbed run:** In the clean run (`investigation-run.txt`),
  `late_shipment_count` was populated from `logistics` under Well-Established, and
  `on_time_delivery_rate` was flagged `⚠️ ESCALATE` under **Contested** because `supplier_audit`
  (95.0%) and `logistics` (78.0%) disagreed. With `logistics` gone, `late_shipment_count` moves to
  **Incomplete**, and — notably — `on_time_delivery_rate` moves out of Contested entirely (the
  Contested section reads `_none_`) because only `supplier_audit`'s 95.0% remains and there is no
  second voice left to disagree with it. Losing a source doesn't just create gaps; it can also
  silently remove a disagreement that was itself a real risk signal.
