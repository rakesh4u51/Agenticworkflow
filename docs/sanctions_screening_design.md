# Sanctions Screening — Insertion Point & Design

## 1. Pipeline Position

Current pipeline: `kyc_check → aml_analysis → [NEW: sanctions_screening] → approval`

The `sanctions_screening` step is inserted **between `aml_analysis` and `approval`**. This position is grounded in the live `get_compliance_pipeline` MCP call, which returns the sequence `kyc_check → aml_analysis → approval`. Sanctions screening must run after AML analysis (so the risk profile is already established) and before the approval decision (so a confirmed match is a hard stop before any funds move).

## 2. Data Inputs Required

| Field | Type | Source | Notes |
|---|---|---|---|
| `customer_id` | string | Transaction payload / banking-api MCP | Non-empty; used for audit correlation, never logged plaintext |
| `counterparty_details` | object | Transaction payload | Must include `name` field; `country` and account identifiers optional |
| `transaction_amount` | float | Transaction payload | Non-negative; used for risk-weighting, not match logic |

## 3. Output Schema

```json
{
  "screened": true,
  "match_list": [],
  "confidence": 0.0
}
```

- `screened` (bool) — whether the screening step executed successfully. `false` on API failure or malformed input.
- `match_list` (list) — zero or more OFAC/UN matches, each with `list_source`, `matched_name`, `score`.
- `confidence` (float, 0–1) — highest match score across all lists; `0.0` when no matches.

## 4. Integration Contract with the AML Module

- **Input from AML module:** `aml_result` (status, risk_score) passes through unchanged; sanctions screening executes independently of the AML verdict.
- **Output to Approval step:** any non-empty `match_list` with `confidence >= 0.75` is a hard stop requiring manual compliance review before approval proceeds.
- **Failure mode:** if the sanctions API is unreachable, `screened` must be `false` and the transaction routed to manual review — never silently passed. This satisfies PCI DSS Req 6 (secure applications) and Req 10 (audit logging).
- **PII handling:** `customer_id` and counterparty name are masked in all log output per PCI DSS Req 3. Results are stored only via the encrypted-storage call site.

## 5. Open Questions / Risks

- **False-positive rate:** fuzzy name matching may flag legitimate counterparties. Threshold `confidence >= 0.75` balances precision vs. recall; adjust post-pilot.
- **List refresh cadence:** OFAC/UN lists update asynchronously; the production implementation must subscribe to delta feeds to avoid stale denials.
- **Dual-list precedence:** when a name appears on both OFAC and UN lists, `confidence` is the maximum score across both. Both entries appear in `match_list`.
