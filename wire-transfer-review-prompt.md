# Wire Transfer Review Extraction Prompt

## 1. JSON Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "transfer_id": {
      "type": "string",
      "description": "Unique wire transfer identifier"
    },
    "amount_usd": {
      "type": "number",
      "description": "Transfer amount in USD"
    },
    "risk_flags": {
      "type": "array",
      "items": {
        "type": "string",
        "enum": [
          "sanctions_list_match",
          "unusual_amount",
          "new_beneficiary",
          "velocity"
        ]
      },
      "description": "Risk indicators identified per policy KB-209 §2"
    },
    "recommended_action": {
      "type": "string",
      "enum": ["approve", "hold", "escalate"],
      "description": "Recommended compliance action"
    },
    "rationale": {
      "type": "string",
      "maxLength": 400,
      "description": "Compliance justification; must cite at least one risk_flag if any are present"
    }
  },
  "required": [
    "transfer_id",
    "amount_usd",
    "risk_flags",
    "recommended_action",
    "rationale"
  ]
}
```

---

## 2. System + Task Prompt

### System Prompt

You are a compliance analyst extracting structured review decisions for wire transfers. Your judgment **must be grounded exclusively in the CONTEXT block** (transaction data + policy excerpt provided). 

**Decision rules:**
- Identify all applicable risk_flags according to policy KB-209 §2.
- If two or more flags are present, recommend "escalate" (§2.3).
- If the sanctions screen result is inconclusive (not a clear "no match" or definitive list hit), recommend "escalate" regardless of flag count (§2.4).
- If only one flag is present and sanctions screening is conclusive, recommend "hold".
- If no flags and sanctions screening is conclusive "no match", recommend "approve".
- **Never guess or apply external knowledge.** If evidence is ambiguous or missing, escalate.

Rationale must:
- Cite the specific policy section(s) applied.
- Reference evidence from the transaction data.
- If multiple flags, list all that triggered escalation.

---

### Task Prompt

**CONTEXT**

```
<transaction>
transfer_id: {{TRANSFER_ID}}
amount_usd: {{AMOUNT}}
originator: {{ORIGINATOR_ID}}
beneficiary: {{BENEFICIARY_ID}} (first seen {{FIRST_SEEN_DATE}})
corridor: {{CORRIDOR}}
prior_30d_transfers: {{PRIOR_30D_COUNT}}
prior_30d_max_usd: {{PRIOR_30D_MAX}}
sanctions_screen: {{SANCTIONS_RESULT}}
</transaction>

<policy KB-209 §2 — wire transfer review>
2.1 A beneficiary first seen within 14 days is a new_beneficiary flag.
2.2 An amount exceeding 10x the originator's prior 30-day maximum is an unusual_amount flag.
2.3 Two or more flags on a single transfer require recommended_action "escalate".
2.4 Where the sanctions screen is inconclusive rather than clear, escalate regardless of flag count.
</policy>

<today_date>{{TODAY_DATE}}</today_date>
```

Extract a compliance decision in JSON. Apply the schema exactly. If any data is missing or ambiguous, flag it in your rationale and recommend "escalate".

---

## 3. Few-Shot Examples

### Example 1: Clear Approve

**Input:**
```json
{
  "transaction": {
    "transfer_id": "WT-20260920-0112",
    "amount_usd": 5200,
    "originator": "C-44891",
    "beneficiary": "BEN-3001 (first seen 2026-03-15)",
    "corridor": "US->GB",
    "prior_30d_transfers": 12,
    "prior_30d_max_usd": 8500,
    "sanctions_screen": "no match"
  },
  "today_date": "2026-09-30"
}
```

**Output:**
```json
{
  "transfer_id": "WT-20260920-0112",
  "amount_usd": 5200,
  "risk_flags": [],
  "recommended_action": "approve",
  "rationale": "Beneficiary BEN-3001 established >14 days ago (no new_beneficiary flag). Amount $5,200 is 0.61× prior 30-day max of $8,500, well below 10× threshold (no unusual_amount flag). Sanctions screen: clear no match. No flags present and sanctions conclusive. Policy KB-209 §2 criteria met for approve."
}
```

---

### Example 2: Clear Escalate

**Input:**
```json
{
  "transaction": {
    "transfer_id": "WT-20260929-0447",
    "amount_usd": 184000,
    "originator": "C-31955",
    "beneficiary": "BEN-7741 (first seen 2026-09-27)",
    "corridor": "IN->AE",
    "prior_30d_transfers": 1,
    "prior_30d_max_usd": 9500,
    "sanctions_screen": "no match"
  },
  "today_date": "2026-09-30"
}
```

**Output:**
```json
{
  "transfer_id": "WT-20260929-0447",
  "amount_usd": 184000,
  "risk_flags": ["new_beneficiary", "unusual_amount"],
  "recommended_action": "escalate",
  "rationale": "Two risk flags triggered: (1) new_beneficiary — BEN-7741 first seen 2026-09-27, within 14 days per KB-209 §2.1; (2) unusual_amount — $184,000 is 19.4× the prior 30-day max of $9,500, exceeding 10× threshold per §2.2. Policy §2.3 mandates escalate when two or more flags present. Escalation required."
}
```

---

## Integration Notes

- **Templating:** Replace `{{VARIABLE}}` with actual transaction values when invoking the prompt.
- **Validation:** Post-process JSON output against the schema above.
- **Ambiguity handling:** If the sanctions_screen shows "inconclusive", "pending", or any value other than "clear match" or "no match", apply KB-209 §2.4 and recommend "escalate" regardless of flag count.
- **Audit trail:** Include the extracted decision, rationale, and transaction reference in the compliance ticket record.
