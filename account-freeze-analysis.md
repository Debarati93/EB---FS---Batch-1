# Account Freeze Request Schema Design

## Normalized JSON Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Account Freeze Request Resolution",
  "type": "object",
  "required": ["ticket_id", "action", "affected_accounts", "reason_code", "evidence_summary"],
  "additionalProperties": false,
  "properties": {
    "ticket_id": {
      "type": "string",
      "description": "Unique ticket identifier (e.g., T-4471)",
      "pattern": "^[A-Z]+-[0-9]+$"
    },
    "action": {
      "type": "string",
      "enum": ["freeze", "partial_freeze", "deny"],
      "description": "Resolution action: freeze all linked accounts, freeze subset, or deny the request"
    },
    "affected_accounts": {
      "type": "array",
      "items": {
        "type": "string",
        "pattern": "^[A-Z]+-[0-9]+$"
      },
      "minItems": 0,
      "description": "Array of account IDs affected by this action (empty if action=deny)"
    },
    "reason_code": {
      "type": "string",
      "enum": ["fraud_suspected", "customer_request", "court_order", "insufficient_evidence"],
      "description": "Structured reason category for the action"
    },
    "evidence_summary": {
      "type": "string",
      "maxLength": 300,
      "description": "Concise narrative of supporting evidence; max 300 chars"
    }
  }
}
```

---

## Field-by-Field Mapping from Five Shapes

| Schema Field | dev_a | dev_b | dev_c | dev_d | dev_e | Status |
|---|---|---|---|---|---|---|
| ticket_id | `ticketId` | `ticket.id` | `id` | `ticket_id` | `ticketRef` | **NORMALIZED** |
| action | `decision` (FREEZE) | `outcome.action` (freeze_all) | `result` (frozen) | `action_taken` (true) | `recommendation.type` (FREEZE) | **NORMALIZED** |
| affected_accounts | `accounts` (array) | `affected` (CSV string) | `account_list` (array of objects) | *(implicit)* | *(implicit)* | **NORMALIZED** |
| reason_code | — | — | `reason_code` (numeric: 3) | — | `reasonCode` (text: FRAUD_SUSPECTED) | **PARTIALLY EXISTED** |
| evidence_summary | `why` (text) | `reason` (text) | `notes` (text) | `summary` (text) | `evidence` (array of strings) | **NORMALIZED** |

---

## Fields Invented (Not in Original Five Samples)

### 1. **`action` enum: `["freeze", "partial_freeze", "deny"]`**
   - **What was in the samples:**
     - dev_a: `"FREEZE"` (boolean decision)
     - dev_b: `"freeze_all"` (string)
     - dev_c: `"frozen"` (status, not action)
     - dev_d: `true` (boolean flag)
     - dev_e: `"FREEZE"` (recommendation type)
   - **Why invented:** The samples only showed variations of *freeze* outcomes. The `deny` action (rejecting a freeze request) and `partial_freeze` (freezing a subset of accounts) were not represented in any sample. These are business-level decisions the schema must support.
   - **Lowercase normalization:** Converted all to lowercase enum for consistency (all samples used either uppercase or mixed case).

### 2. **`reason_code` enum: `["fraud_suspected", "customer_request", "court_order", "insufficient_evidence"]`**
   - **What was in the samples:**
     - dev_a: `"customer reports unauthorised transactions"` (free text)
     - dev_b: `"fraud"` (single word)
     - dev_c: `3` (numeric code with no legend)
     - dev_d: *(none; implicit via freeze_type: "full")*
     - dev_e: `"FRAUD_SUSPECTED"` (structured string)
   - **Why invented:** 
     - Samples had inconsistent representations: free text, single keywords, opaque numeric codes, and unstructured strings.
     - None had `court_order` or `customer_request` explicitly (dev_a has "customer reports" but not as a formal reason_code).
     - `insufficient_evidence` is invented to support the `deny` action (when freeze is rejected).
   - **Rationale:** Compliance routing and audit trails require a fixed, machine-readable set of reason categories. The enum enforces consistency across all ticket types.

### 3. **`evidence_summary` field with 300-char max**
   - **What was in the samples:**
     - dev_a: `why` (free text)
     - dev_b: `reason` (free text)
     - dev_c: `notes` (free text)
     - dev_d: `summary` (free text)
     - dev_e: `evidence` (array of strings)
   - **Why invented:**
     - All samples had narrative/justification fields, but under different names (`why`, `reason`, `notes`, `summary`, `evidence`).
     - None had an explicit field named `evidence_summary`.
     - The 300-char max is new—enforces conciseness and prevents verbose justifications that become unscalable in audit logs.
   - **Rationale:** Consolidates heterogeneous narrative fields into one canonical field with explicit length constraints.

### 4. **`affected_accounts` as a simple array of strings**
   - **What was in the samples:**
     - dev_a: `accounts` (array of strings) ✓
     - dev_b: `affected` (comma-separated string)
     - dev_c: `account_list` (array of objects with `.number` property)
     - dev_d: *(no explicit field)*
     - dev_e: *(no explicit field; implied by scope: "ALL_LINKED")*
   - **Why normalized:** 
     - Samples used three different representations: array of strings, CSV string, and array of objects.
     - The `pattern` regex `^[A-Z]+-[0-9]+$` is new (not in samples) to enforce account ID format.
   - **Rationale:** Downstream systems need a consistent, array-based format for iteration and validation. Objects with `.number` properties are unnecessarily nested.

---

## Summary

| Invented Field | Reason | Impact |
|---|---|---|
| `action` enum (especially `deny`, `partial_freeze`) | Samples only showed freezes, no rejections or partial actions | Enables full resolution workflow (approve / partial / reject) |
| `reason_code` enum | Samples had free text, numeric, unstructured; no audit categories | Compliance-grade reason categorization; enables filtering and reporting |
| `evidence_summary` field | Samples had different narrative field names; no length constraint | Single canonical field; enforces brevity for audit scalability |
| Account ID pattern validation | Samples had inconsistent account formats | Prevents typos/malformed IDs in downstream systems |

All other fields (`ticket_id`, `affected_accounts` array format, `evidence_summary` as narrative) normalize existing concepts without inventing new ones.
