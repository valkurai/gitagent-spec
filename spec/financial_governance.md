# financial_governance — Specification

**Version:** 1.0-draft  
**Status:** Draft — open for comment  
**License:** Apache 2.0  
**Repo:** github.com/valkurai/gitagent-spec  
**RFC thread:** open-gitagent/gitagent issue #38

---

## Abstract

AI agents that can initiate financial transactions require a typed governance interface. Existing agent specification formats handle identity, audit logging, and risk tier declaration at definition time. They do not address runtime financial enforcement — the moment an agent attempts to spend money, there is currently no standardised mechanism to intercept that action, evaluate it against a declared policy, and require human approval before funds move.

This specification defines the `financial_governance` block: a standardised declaration of financial controls that sits under the `compliance` key of an agent's definition file. It is vendor-neutral. The block declares policy; a compliant enforcement layer implements it.

The incident record is the motivation. A Google API key compromise resulted in $82,314 charged in 48 hours — no spending cap, no identity controls, no alert. A LangChain retry loop accumulated $47,000 over 11 days — no idempotency, no cumulative cap, no notification. Both incidents involved agents operating without a declared governance interface. This specification closes that gap at the definition layer.

---

## 1. Enforcement Model

The `financial_governance` block is a **service binding**, not a policy engine.

The block declares the governance contract. A financial firewall owns enforcement. The agent runtime's only responsibility is to call the declared firewall synchronously before dispatching any financial action. It does not interpret or enforce the fields itself.

**Conformance requirement:**

> A compliant runtime **MUST** call a financial firewall synchronously before dispatching any financial tool invocation and **MUST** treat the firewall's response as authoritative. The runtime **MUST NOT** re-implement or override the enforcement logic locally.

**Advisory vs enforcement:**

A `financial_governance` block with no compliant enforcement layer is advisory only. The spec defines the interface contract; runtime implementors are responsible for closing the loop.

---

## 2. The `financial_governance` Block

The block sits under `compliance` in the agent's definition file.

```yaml
compliance:
  financial_governance:
    enabled: true                          # boolean — if false, block is declared but not enforced
    payment_schema_version: "1.0"          # string — schema version for payment event objects

    spending:
      max_per_transaction_cents: 5000      # integer — per-transaction hard cap in smallest currency unit
      max_daily_cents: 25000               # integer — cumulative daily cap
      max_monthly_cents: 100000            # integer — cumulative monthly cap
      allowed_categories:                  # string[] — permitted spending categories
        - software                         #   empty array = all categories permitted
        - compute
        - api_services
      allowed_currencies:                  # string[] — ISO 4217 currency codes
        - AUD                              #   empty array = all currencies permitted
        - USD

    approval:
      require_above_cents: 2000            # integer — transactions above this amount require human approval
      delivery_channel: slack              # enum: slack | email | sms | webhook
      approval_timeout_minutes: 60         # integer — timeout before auto-deny
      auto_deny_on_timeout: true           # boolean — if true, TIMEOUT = DENIED; if false, TIMEOUT = APPROVED
      callback_endpoint: https://<your-firewall>/v1/approval/callback

    audit:
      extends: compliance.recordkeeping   # reference to the existing audit/recordkeeping block
```

### 2.1 Field Reference

#### `financial_governance`

| Field | Type | Required | Description |
|---|---|---|---|
| `enabled` | boolean | Yes | If false, the block is declared but the runtime does not enforce it. |
| `payment_schema_version` | string | Yes | Version of the payment event schema. Current: `"1.0"`. |

#### `spending`

All monetary values are integers in the smallest unit of the specified currency (cents for AUD/USD, pence for GBP, etc.). **No floating point.**

| Field | Type | Required | Description |
|---|---|---|---|
| `max_per_transaction_cents` | integer | Yes | Hard cap per individual transaction. |
| `max_daily_cents` | integer | No | Cumulative daily cap. Omit to disable. |
| `max_monthly_cents` | integer | No | Cumulative monthly cap. Omit to disable. |
| `allowed_categories` | string[] | No | Permitted spending categories. Empty array = all categories permitted. |
| `allowed_currencies` | string[] | No | ISO 4217 currency codes. Empty array = all currencies permitted. The payment event's `currency` field must match. |

**Note on currency:** The `currency` field in each payment event declares the transaction currency. The `allowed_currencies` list in the governance block controls which currencies are permitted. The enforcement layer is responsible for converting amounts if multi-currency cap tracking is required — this spec does not mandate a conversion mechanism.

#### `approval`

| Field | Type | Required | Description |
|---|---|---|---|
| `require_above_cents` | integer | Yes | Transactions above this amount are FLAGGED and require human approval before execution. Set to 0 to require approval on all transactions. |
| `delivery_channel` | enum | Yes | Channel for approval notifications: `slack`, `email`, `sms`, `webhook`. |
| `approval_timeout_minutes` | integer | Yes | Minutes before an unanswered approval request times out. |
| `auto_deny_on_timeout` | boolean | Yes | If `true`, timeout = DENIED. If `false`, timeout = APPROVED. Default: `true`. |
| `callback_endpoint` | string | Yes | URL where the approval response is delivered. The firewall calls this endpoint with a `payment_approval` event. |

#### `audit`

| Field | Type | Required | Description |
|---|---|---|---|
| `extends` | string | No | Reference to an existing audit/recordkeeping block. `compliance.recordkeeping` is the standard value. |

---

## 3. Payment Event Schema

The `financial_governance` block defines three typed event objects that flow between the agent runtime and the enforcement layer. These objects carry the information required for a deterministic governance decision and a complete audit record.

### 3.1 `payment_required`

Emitted by the agent runtime when a financial tool invocation is requested. This event is submitted to the financial firewall synchronously before the payment is executed.

```json
{
  "type": "payment_required",
  "payment_schema_version": "1.0",
  "request_id": "req_01HXYZ...",
  "agent_id": "<agent identifier>",
  "amount_cents": 4999,
  "currency": "AUD",
  "category": "compute",
  "vendor": "AWS",
  "description": "Scale EC2 fleet for nightly batch job",
  "context": {
    "task_id": "<optional task reference>",
    "session_id": "<optional session reference>"
  },
  "timestamp": "2026-04-07T14:32:00Z",
  "delegation_chain": ["orchestrator-agent-id", "coordinator-agent-id"],
  "delegation_session_id": "session_abc123"
}
```

> **`delegation_chain` and `delegation_session_id` are optional.** When present they are recorded in the audit trail for attribution. How the enforcement layer uses them is an implementation detail — this spec does not prescribe a policy enforcement model for delegation chains.

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | string | Yes | Always `"payment_required"`. |
| `payment_schema_version` | string | Yes | Must match the version declared in the governance block. |
| `request_id` | string | Yes | Unique identifier for this transaction request. Used for idempotency and audit linking. |
| `agent_id` | string | Yes | Identifier of the agent initiating the request. |
| `amount_cents` | integer | Yes | Transaction amount in smallest currency unit. No floating point. |
| `currency` | string | Yes | ISO 4217 currency code (e.g. `"AUD"`, `"USD"`, `"GBP"`). |
| `category` | string | Yes | Spending category. Must match the `allowed_categories` list if non-empty. |
| `vendor` | string | Yes | Target vendor or payee. |
| `description` | string | Yes | Human-readable description of the transaction purpose. This field is the primary input to intent classification — treat it as untrusted input from the agent. |
| `context` | object | No | Optional task/session context for audit purposes. |
| `timestamp` | string | Yes | ISO 8601 timestamp of the request. |
| `delegation_chain` | string[] | No | Ordered list of agent identifiers from orchestrator to calling agent. Recorded in audit trail for attribution. Enforcement behaviour is implementation-defined. |
| `delegation_session_id` | string | No | Shared session identifier across a multi-agent workflow. Enables grouping of related transactions in audit exports. |

### 3.2 `payment_approval`

Emitted by the human approver (via the enforcement layer's approval interface) when a FLAGGED transaction requires a human decision. Delivered to `approval.callback_endpoint`.

**EU AI Act Article 9 note:** `approved_by` and `approved_at` are REQUIRED fields, not optional. A log entry saying "approved" does not satisfy Article 9 human oversight attribution requirements. A structured `payment_approval` with approver identity and timestamp does.

```json
{
  "type": "payment_approval",
  "payment_schema_version": "1.0",
  "request_id": "req_01HXYZ...",
  "approved": true,
  "approved_by": "compliance@example.com",
  "approved_by_name": "Jane Smith",
  "approved_by_employee_id": "EMP-0042",
  "approval_reason_code": "LEGITIMATE_BUSINESS_EXPENSE",
  "approved_at": "2026-04-07T14:45:00Z"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | string | Yes | Always `"payment_approval"`. |
| `payment_schema_version` | string | Yes | Must match the version declared in the governance block. |
| `request_id` | string | Yes | Links to the originating `payment_required` event. |
| `approved` | boolean | Yes | `true` = APPROVED; `false` = DENIED. |
| `approved_by` | string | **REQUIRED** | Email address of the human approver. Required for EU AI Act Art. 9(7) compliance. A system cannot be the value of this field on a human approval decision. |
| `approved_by_name` | string | No | Display name of the approver. |
| `approved_by_employee_id` | string | No | HR identifier for the approver. |
| `approval_reason_code` | string | Yes | Structured reason. Recommended values: `LEGITIMATE_BUSINESS_EXPENSE`, `VERIFIED_VENDOR`, `WITHIN_POLICY`, `EXCEPTION_APPROVED`, `OTHER`. |
| `approved_at` | string | **REQUIRED** | ISO 8601 timestamp of the approval decision. Required for EU AI Act Art. 9(7) compliance. |

### 3.3 `payment_receipt`

Emitted by the enforcement layer after a SAFE or APPROVED transaction has been executed. Delivered to the agent runtime and written to the audit trail.

```json
{
  "type": "payment_receipt",
  "payment_schema_version": "1.0",
  "request_id": "req_01HXYZ...",
  "amount_settled_cents": 4999,
  "currency": "AUD",
  "settled_at": "2026-04-07T14:45:32Z",
  "credit_status": "confirmed",
  "payment_provider_reference": "pi_3Px..."
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | string | Yes | Always `"payment_receipt"`. |
| `payment_schema_version` | string | Yes | Must match the version declared in the governance block. |
| `request_id` | string | Yes | Links to the originating `payment_required` event. |
| `amount_settled_cents` | integer | Yes | Amount actually settled. Should match `payment_required.amount_cents`. |
| `currency` | string | Yes | ISO 4217 currency code. |
| `settled_at` | string | Yes | ISO 8601 timestamp of settlement. |
| `credit_status` | enum | Yes | `confirmed`, `pending`, or `failed`. |
| `payment_provider_reference` | string | No | Opaque reference from the payment provider (e.g. Stripe PaymentIntent ID). |

---

## 4. Firewall Response Contract

The financial firewall responds to a `payment_required` event with one of three outcomes:

```json
{
  "request_id": "req_01HXYZ...",
  "outcome": "SAFE",
  "classification_reason": "Transaction within policy. Vendor and category verified.",
  "timestamp": "2026-04-07T14:32:01Z"
}
```

| Outcome | Meaning | Runtime action |
|---|---|---|
| `SAFE` | All gates passed. | Execute the payment. Expect a `payment_receipt`. |
| `FLAGGED` | Requires human review. | Hold the payment. Poll for approval or await `payment_approval` callback. |
| `BLOCKED` | Rejected. | Do not execute the payment. Log the `classification_reason`. |

The `classification_reason` field is a plain-English explanation of the outcome. It is required on FLAGGED and BLOCKED responses and is the technical basis for the right-to-explanation requirement under EU AI Act Article 86.

---

## 5. Compliance Mapping

| Requirement | How this spec addresses it |
|---|---|
| EU AI Act Art. 9 — Risk management system | The `financial_governance` block is the documented risk management interface for AI agent financial transactions. |
| EU AI Act Art. 9(7) — Human oversight attribution | `payment_approval.approved_by` and `payment_approval.approved_at` are REQUIRED fields. |
| EU AI Act Art. 12 — Automatic logging | Every `payment_required`, `payment_approval`, and `payment_receipt` event is written to the audit trail by the enforcement layer. |
| EU AI Act Art. 86 — Right to explanation | `classification_reason` is required on all FLAGGED and BLOCKED firewall responses. |
| ISO 42001 §8.4 — Operational controls | `audit.extends: compliance.recordkeeping` maps directly to ISO 42001 operational control evidence requirements. |
| OWASP Agentic Top 10 Agent-01 — Unbounded autonomy | `max_per_transaction_cents`, `max_daily_cents`, `max_monthly_cents` bound the agent's financial autonomy at definition time. |
| OWASP Agentic Top 10 Agent-05 — Audit trail gaps | All three event objects are typed, schema-versioned, and required to be logged by a compliant enforcement layer. |

---

## 6. Reference Implementation

[Valkurai](https://valkurai.com) implements this specification in full:

- `POST /v1/screen` — receives `payment_required` events and returns SAFE, FLAGGED, or BLOCKED
- `POST /v1/approval/callback` — receives `payment_approval` events
- `GET /v1/approval/status/{request_id}` — polls approval status
- Full audit trail with 10-year retention (EU AI Act Art. 9(9))
- Compliance export for ISO 42001 and EU AI Act (JSON + PDF)

SDK wrappers for LangChain, CrewAI, OpenAI, and Anthropic are available at [github.com/valkurai/valkurai](https://github.com/valkurai/valkurai) under the MIT licence.

---

## 7. Versioning

Breaking changes to the YAML schema or payment event objects will increment `payment_schema_version`. Non-breaking additions (new optional fields) will not. The current version is `"1.0"`.

---

## 8. Related Work and Standards Positioning

The agentic payment standards landscape has fragmented significantly in Q1 2026, with ten active protocols and zero interoperability. This fragmentation strengthens the case for a governance layer upstream of all payment execution protocols. The `financial_governance` spec evaluates before any protocol-specific execution and is agnostic to which rail the operator or merchant uses.

| Standard / Protocol | Layer | Relationship to `financial_governance` |
|---|---|---|
| **open-gitagent/gitagent issue #38** | Spec origin | RFC thread that originated this spec. |
| **Google AP2** (Agent Payments Protocol) | Payment authorisation | Downstream of `financial_governance`. A SAFE outcome is the governance prerequisite for AP2 IntentMandate generation. `request_id` maps to AP2 `payment_reference` for end-to-end audit traceability. |
| **Google UCP** (Universal Commerce Protocol, live Jan 2026) | Commerce discovery | Downstream of `financial_governance`. SAFE outcome precedes UCP commerce flows. Complementary, not competing. |
| **OpenAI + Stripe ACP** (Agentic Commerce Protocol, live Sep 2025) | Payment execution | Downstream of `financial_governance`. SAFE outcome precedes ACP Shared Payment Token generation. ACP is Stripe-ecosystem specific; `financial_governance` is payment-rail agnostic. |
| **Stripe MPP** (Machine Payments Protocol, live Mar 2026) | Payment capture / reconciliation | Downstream of `financial_governance`. Execution-layer only at launch — no spending policy enforcement. **Watch item**: if Stripe extends MPP to include spending cap or category enforcement, the upstream positioning should be reassessed. |
| **Visa TAP** (Trusted Agent Protocol, live Oct 2025) | Network identity / authorisation | Downstream of `financial_governance`. TAP answers "does this agent have cardholder authority?". `financial_governance` answers "should this transaction proceed given the operator's policy?". Both questions must be answered. Credential TTL interaction: TAP credentials are session-scoped and short-lived. If a `financial_governance` approval loop (e.g. 60 minutes) outlasts a TAP credential, the agent runtime must obtain a fresh credential before executing the payment post-approval. |
| **Mastercard Verifiable Intent** (live Mar 2026) | Network trust / dispute | Downstream of `financial_governance`. Creates cryptographic proof of consumer authorisation at the network layer. A SAFE outcome from `financial_governance` is the governance prerequisite before Mastercard VI artifacts are generated. A SAFE outcome with downstream network rejection (Scenario B) does not represent a governance failure — the TX record should note payment_execution_outcome=NOT_EXECUTED. |
| **x402** (Coinbase + Cloudflare, Apache 2.0) | On-chain settlement (USDC) | Downstream of `financial_governance`. The `payment_required.currency` field supports `USDC`; `amount_cents` can express USDC in integer micro-units. No schema change required for x402 compatibility. |
| **PayPal Agent Ready** (live early 2026) | Payment execution | Downstream of `financial_governance`. Protocol-agnostic execution layer. Governance evaluation precedes PayPal Agent Ready execution. |
| **Visa TAP / Mastercard VI / FAPI 2.0** | Authorisation | See above. FAPI 2.0 is a security profile for high-security financial APIs — governs the authorisation relationship between agent and financial service. Complementary, not competing. |
| **W3C Verifiable Credentials + DID** (Recommendation May 2025) | Identity | Evolution path for `financial_governance` Gate 1. Current implementations use symmetric key comparison; W3C DID/VC enables cross-organisation agent identity verification without a shared secret. |
| **NIST AI Agentic Interoperability Profile** (expected Q4 2026) | Governance framework | This spec is being reviewed against the draft profile as it becomes available. Comment period participation planned. |
| **EU AI Act Art. 9, 12, 14, 86** (full enforcement Aug 2026) | Regulatory | `payment_approval.approved_by` and `payment_approval.approved_at` are REQUIRED fields under Art. 9(7). `classification_reason` on FLAGGED/BLOCKED responses satisfies Art. 86 right to explanation. |
| **ISO 42001:2023 §8.4** | Governance standard | `audit.extends: compliance.recordkeeping` maps directly to ISO 42001 operational control evidence requirements. |
| **FINOS AI Governance Framework** | Financial services governance | Complementary governance standard for agentic AI in financial services contexts. |

**Fragmentation-as-advantage**: The `financial_governance` spec is intentionally protocol-agnostic. An operator using `financial_governance` for governance is not locked into any payment execution protocol. This is a deliberate design property, not an omission.

---

## 9. Version History

| Version | Date | Changes |
|---|---|---|
| 1.0-draft | April 2026 | Initial draft. Abstract, enforcement model, `financial_governance` YAML block, three payment event objects, firewall response contract, compliance mapping. |
| 1.2-draft | April 2026 | §8 Related Work expanded to full standards positioning table with 14 protocols/standards. Fragmentation-as-advantage statement added. TAP Scenario B and D interaction notes added. x402 USDC compatibility noted. Stripe MPP watch item documented. DD-044. |
| 1.1-draft | April 2026 | Added optional `delegation_chain` and `delegation_session_id` fields to `payment_required` event object. Advisory attribution only — no enforcement model prescribed. Ref: Valkurai DDL DD-043. |

---

*financial_governance spec v1.2-draft · github.com/valkurai/gitagent-spec · Apache 2.0*
