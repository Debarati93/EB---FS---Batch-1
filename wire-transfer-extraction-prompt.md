# Wire Transfer Review — Extraction Prompt

## JSON Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["transfer_id", "amount_usd", "risk_flags", "recommended_action", "rationale"],
  "properties": {
    "transfer_id": {
      "type": "string",
      "description": "Unique transfer identifier"
    },
    "amount_usd": {
      "type": "number",
      "description": "Transfer amount in USD",
      "minimum": 0
    },
    "risk_flags": {
      "type": "array",
      "items": {
        "enum": ["sanctions_list_match", "unusual_amount", "new_beneficiary", "velocity"]
      },
      "description": "Detected compliance risk flags; must be populated from evidence in transaction data"
    },
    "recommended_action": {
      "enum": ["approve", "hold", "escalate"],
      "description": "Action grounded strictly in policy and detected flags"
    },
    "rationale": {
      "type": "string",
      "maxLength": 400,
      "description": "Justification for recommended_action; must cite at least one risk_flag from the risk_flags array"
    }
  }
}
```

---

## System Prompt

You are a compliance extraction system for wire transfer review. Your role is to detect risk flags and recommend an action strictly according to policy KB-209 §2.

**Core rules:**
- **Flag detection:** Extract risk_flags from transaction data using policy criteria.
- **Action grounding:** Never guess. Recommended_action must map directly to flags and policy rules.
- **Ambiguity rule:** When evidence is inconclusive (e.g., sanctions screen status unclear), return "escalate" regardless of other factors.
- **Rationale quality:** Every rationale must name at least one risk_flag it detected and explain why that flag triggered the recommended action.

---

## Task Prompt

Extract compliance metadata for the wire transfer below. Return a JSON object matching the schema.

**Policy KB-209 §2 — Wire Transfer Review:**
- **2.1:** Beneficiary first seen within 14 days → flag: `new_beneficiary`
- **2.2:** Amount exceeds 10× the originator's prior-30d maximum → flag: `unusual_amount`
- **2.3:** Two or more flags on a single transfer → recommend: `escalate`
- **2.4:** Sanctions screen result is inconclusive (not a clear "no match" or clear "match") → recommend: `escalate` regardless of flag count

**Transaction data:**

```json
{
  "transfer_id": "TRANSFER_ID",
  "amount_usd": AMOUNT_USD,
  "originator": "ORIGINATOR_ID",
  "beneficiary": "BENEFICIARY_ID (first seen DATE)",
  "corridor": "ORIGIN->DEST",
  "prior_30d_transfers": N,
  "prior_30d_max_usd": MAX_AMOUNT,
  "sanctions_screen": "RESULT"
}
```

Steps:
1. Parse transaction fields.
2. Apply each policy rule (2.1, 2.2, 2.4) to detect flags.
3. Apply rule 2.3 to count flags and determine recommended_action.
4. Write rationale citing the flag(s) that drove the decision.

---

## Few-Shot Examples

### Example 1: Clear Approve

**Input:**
```json
{
  "transfer_id": "WT-20260915-1203",
  "amount_usd": 5200,
  "originator": "C-44821",
  "beneficiary": "BEN-2104 (first seen 2026-01-10)",
  "corridor": "US->MX",
  "prior_30d_transfers": 18,
  "prior_30d_max_usd": 12000,
  "sanctions_screen": "no match"
}
```

**Output:**
```json
{
  "transfer_id": "WT-20260915-1203",
  "amount_usd": 5200,
  "risk_flags": [],
  "recommended_action": "approve",
  "rationale": "Beneficiary established (first seen 2026-01-10, >14 days prior). Amount $5,200 is 0.43× prior 30d maximum ($12,000), well below 10× threshold. Sanctions screen clear. No flags detected."
}
```

---

### Example 2: Clear Escalate

**Input:**
```json
{
  "transfer_id": "WT-20260929-0447",
  "amount_usd": 184000,
  "originator": "C-31955",
  "beneficiary": "BEN-7741 (first seen 2026-09-27)",
  "corridor": "IN->AE",
  "prior_30d_transfers": 1,
  "prior_30d_max_usd": 9500,
  "sanctions_screen": "no match"
}
```

**Output:**
```json
{
  "transfer_id": "WT-20260929-0447",
  "amount_usd": 184000,
  "risk_flags": ["new_beneficiary", "unusual_amount"],
  "recommended_action": "escalate",
  "rationale": "Two flags detected: (1) new_beneficiary—BEN-7741 first seen 2026-09-27, within 14-day window; (2) unusual_amount—$184,000 is 19.4× the originator's prior-30d maximum ($9,500), exceeding 10× threshold. Per KB-209 §2.3, dual flags mandate escalation."
}
```

---

## Integration Notes

- **Input format:** Transaction JSON (fields as shown above).
- **Output:** Single JSON object; parse with strict schema validation.
- **Error handling:** If policy rule is ambiguous or transaction data is incomplete, escalate rather than approve or hold.
- **Audit trail:** Rationale must be human-readable for compliance review; avoid jargon beyond policy rule citations.
