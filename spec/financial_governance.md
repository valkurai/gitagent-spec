# financial_governance — RFC v1.0

**Status:** Draft
**License:** Apache 2.0
**Repository:** https://github.com/Danbi58/gitagent-spec
**Origin:** Community discussion in [open-gitagent/gitagent issue #38](https://github.com/open-gitagent/gitagent/issues/38)

---

## 1. Abstract

Autonomous AI agents increasingly initiate financial transactions: purchasing API credits, provisioning cloud infrastructure, paying for SaaS subscriptions, and settling vendor invoices. The existing `compliance` block in agent specifications handles identity, audit logging, and risk tier declaration at definition time. It does not declare runtime enforcement.

This gap creates a structural problem. An agent that carries a compliance declaration but no runtime financial guardrails can exceed its intended spend authority without any enforcement mechanism stopping it. The `financial_governance` block fills that gap by providing a typed, declarative interface for runtime financial constraints.

The block is designed to be:

- **Declarative** — the configuration states what the constraints are, not how they are enforced
- **Vendor-neutral** — any compliant financial firewall implementation can be referenced in the `firewall` field
- **Auditable** — the payment event schema produces structured records that satisfy regulatory requirements
- **Composable** — it extends the existing `compliance` block rather than replacing it

---

## 2. Enforcement Model

> **This section is normative. Implementors must read it before writing a conformance claim.**

### 2.1 What the block declares

The `financial_governance` block is a **service binding**, not a policy engine. The block declares the policy contract; the financial firewall owns enforcement. The agent runtime's only responsibility is to call the firewall synchronously before dispatching any financial action. It does not interpret or enforce the fields itself.

The spending limits, approval thresholds, and category lists in this block are authoritative inputs to the firewall. They are not hints. The agent runtime and orchestrator have no visibility into whether a transaction exceeded a threshold — only into the firewall's response. The orchestrator cannot override these fields at runtime.

### 2.2 Conformance requirement

A **conformant runtime** calls the declared firewall synchronously before any financial action and treats its response as authoritative. It does not re-implement or override the enforcement logic locally.

Specifically:

- The runtime MUST call `POST /v1/request` (or the equivalent endpoint for the declared firewall) before dispatching payment
- The runtime MUST block the financial action if the firewall returns a non-2xx response
- The runtime MUST NOT cache or locally evaluate the policy fields from this block as a substitute for calling the firewall
- The runtime MUST treat a `402 Payment Required` response as a signal to wait for the `payment_approval` event before proceeding
- The runtime MUST NOT retry a flagged or blocked transaction without receiving an explicit `payment_approval` event

### 2.3 Advisory vs enforcement

A `financial_governance` block with no compliant enforcement layer is **advisory only**. The spec defines the interface contract; runtime implementors are responsible for closing the loop. Agent registries and orchestration platforms that wish to carry a conformance badge must demonstrate that they call the declared firewall for all agents with `enabled: true`.

---

## 3. Configuration Block

### 3.1 YAML definition

```yaml
financial_governance:
  enabled: true
  firewall: valkurai              # string identifier — any compliant implementation
  payment_schema_version: "1.0"

  spending:
    max_per_transaction_cents: 5000
    max_monthly_cents: 200000
    currency_default: AUD
    allowed_categories:
      - software
      - api
      - saas
      - cloud
    blocked_categories:
      - gambling
      - crypto
      - unknown

  approval:
    require_above_cents: 2000
    approval_timeout_minutes: 60
    auto_deny_on_timeout: true

  audit:
    extends: compliance.recordkeeping
    transaction_log: true

  event_schema:
    payment_required:
      fields: [type, amount_cents, currency, category, context, request_id]
    payment_approval:
      fields: [type, request_id, approved, approved_by, approved_at]
    payment_receipt:
      fields: [type, request_id, amount_settled_cents, settled_at, credit_status]
```

### 3.2 Field reference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `enabled` | boolean | yes | When `false`, the block is ignored entirely. Default is `false`. Agents without payment capabilities should omit this block or set `enabled: false`. |
| `firewall` | string | yes | Identifier for the financial firewall implementation. Not an enum — any compliant implementation may be referenced here. |
| `payment_schema_version` | string | yes | Version of the payment event schema this agent declares compliance with. Used by auditors to verify which schema governed each transaction. |
| `spending.max_per_transaction_cents` | integer | no | Maximum spend per transaction in minor currency units (cents). Enforced by the firewall. |
| `spending.max_monthly_cents` | integer | no | Maximum cumulative spend per calendar month in minor currency units. Enforced by the firewall. |
| `spending.currency_default` | string | no | ISO 4217 currency code. Default currency when the transaction does not specify one. |
| `spending.allowed_categories` | string[] | no | Whitelist of spend categories the agent may transact in. Transactions in unlisted categories are blocked at Gate 2. |
| `spending.blocked_categories` | string[] | no | Explicit blocklist of spend categories. Takes precedence over `allowed_categories` if both are present. |
| `approval.require_above_cents` | integer | no | Transactions above this threshold require human approval before the firewall authorises payment. |
| `approval.approval_timeout_minutes` | integer | no | Time window in minutes within which an approver must respond. After timeout, `auto_deny_on_timeout` determines outcome. |
| `approval.auto_deny_on_timeout` | boolean | no | If `true`, the firewall automatically denies transactions where approval was not received within the timeout window. |
| `audit.extends` | string | no | Reference to an existing compliance block to inherit from. Use `compliance.recordkeeping` to avoid duplicating retention period declarations. |
| `audit.transaction_log` | boolean | no | If `true`, all transactions (approved, denied, flagged, blocked) are written to the audit log. |

### 3.3 Design notes

**`firewall` is a string, not an enum.** Any compliant implementation can be referenced. New firewall vendors do not require changes to this spec.

**Endpoint, auth credentials, and transport config are runtime configuration**, not spec fields. They belong in environment variables or a separate `integrations` block. They MUST NOT appear in `agent.yaml`. This separates the policy declaration (what constraints apply) from the operational wiring (where the firewall runs and how to authenticate to it).

**`approval.require_above_cents` is enforced by the firewall.** The agent runtime and orchestrator have no visibility into whether a transaction exceeded the threshold — only into the firewall's response. The orchestrator cannot override this field.

**`audit.extends: compliance.recordkeeping`** reuses the existing compliance block rather than redeclaring `retention_period` and similar fields. This prevents drift between the two blocks and avoids conflicting retention declarations.

**Agents without payment capabilities ignore this block entirely.** `enabled: false` is the default. Registries SHOULD validate that agents claiming `enabled: true` have a conformant runtime.

---

## 4. Payment Event Schema

All financial events emitted by a conformant system MUST conform to one of the three typed objects below. The `payment_schema_version` field in the configuration block identifies which version of this schema governs a given agent's transactions.

### 4.1 `payment_required`

Emitted by the agent runtime before any financial action is dispatched. This event represents the agent's intent to spend and is the input to the firewall's evaluation pipeline.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | `"payment_required"` | **REQUIRED** | Discriminator field. |
| `request_id` | string (UUID) | **REQUIRED** | Unique identifier for this payment request. Used as the idempotency key throughout the approval lifecycle. |
| `amount_cents` | integer | **REQUIRED** | Requested amount in minor currency units (cents). Never a float. |
| `currency` | string | **REQUIRED** | ISO 4217 currency code (e.g. `"USD"`, `"AUD"`). |
| `category` | string | **REQUIRED** | Spend category. Must match one of the values in `spending.allowed_categories`. |
| `context` | string | **REQUIRED** | Human-readable description of what the payment is for. Used by the LLM enrichment gate and displayed to human approvers. |
| `agent_key_hash` | string | optional | SHA-256 hash of the agent key. Never the raw key. |
| `expires_at` | string (ISO 8601) | optional | Timestamp after which this request should not be approved. |

**Example:**

```json
{
  "type": "payment_required",
  "request_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "amount_cents": 4999,
  "currency": "USD",
  "category": "api_credits",
  "context": "Purchase 500 Stability AI image generation credits for the current design sprint"
}
```

### 4.2 `payment_approval`

Returned by the firewall's approval surface when a human approver acts on a flagged transaction. This event closes the approval loop and is the authorisation signal the runtime waits for before proceeding.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | `"payment_approval"` | **REQUIRED** | Discriminator field. |
| `request_id` | string (UUID) | **REQUIRED** | The `request_id` from the corresponding `payment_required` event. |
| `approved` | boolean | **REQUIRED** | `true` if the transaction is approved; `false` if denied. |
| `approved_by` | string (email) | **REQUIRED** | Email address of the human who approved or denied the transaction. See EU AI Act note in Section 5. |
| `approved_at` | string (ISO 8601) | **REQUIRED** | Timestamp at which the approval or denial was recorded. See EU AI Act note in Section 5. |

> **`approved_by` and `approved_at` are REQUIRED. They MUST NOT be omitted, nullable, or optional.** A log entry that records only `approved: true` does not satisfy Article 9 of the EU AI Act. See Section 5.

**Example:**

```json
{
  "type": "payment_approval",
  "request_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "approved": true,
  "approved_by": "alice@example.com",
  "approved_at": "2025-04-03T14:22:09Z"
}
```

### 4.3 `payment_receipt`

Emitted after payment settlement. Closes the audit loop. The firewall or payment processor emits this event; the agent runtime records it.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | `"payment_receipt"` | **REQUIRED** | Discriminator field. |
| `request_id` | string (UUID) | **REQUIRED** | The `request_id` from the originating `payment_required` event. |
| `amount_settled_cents` | integer | **REQUIRED** | Amount actually settled in minor currency units. May differ from `amount_cents` if partial settlement occurred. |
| `settled_at` | string (ISO 8601) | **REQUIRED** | Timestamp at which settlement was confirmed. |
| `credit_status` | `"settled"` \| `"pending"` \| `"failed"` | **REQUIRED** | Settlement outcome. |
| `stripe_payment_intent_id` | string | optional | Stripe PaymentIntent ID, if Stripe was the settlement layer. |

**Example:**

```json
{
  "type": "payment_receipt",
  "request_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "amount_settled_cents": 4999,
  "settled_at": "2025-04-03T14:22:15Z",
  "credit_status": "settled",
  "stripe_payment_intent_id": "pi_3OqXyZ2eZvKYlo2C1234abcd"
}
```

---

## 5. EU AI Act Compliance Note

Under EU AI Act Article 9, payment service providers (PSPs) enabling AI agents to initiate transactions are generally held liable for unauthorised transactions they cannot explain. Demonstrating explainability requires a structured audit trail — not just a status flag.

### What does not satisfy Article 9

A log entry stating `"approved": true` does not satisfy Article 9. Neither does a timestamp without an identified approver. Neither does an approver email without a timestamp.

### What does satisfy Article 9

A structured `payment_approval` event containing:

- **`approved_by`** — the email address of the named human who approved or denied the transaction (required)
- **`approved_at`** — an ISO 8601 timestamp recording when the approval was recorded (required)

These two fields together create an auditable record linking each transaction to a named human decision-maker at a specific point in time.

### The role of `payment_schema_version`

The `payment_schema_version` field in the configuration block makes the compliance assertion tractable. Auditors can identify which schema version governed each transaction and verify that the schema in force at the time required `approved_by` and `approved_at`. This is particularly important when the schema evolves — auditors can determine whether a historical transaction was governed by a version of the schema that required these fields.

### System denials

Transactions denied automatically (e.g. by `auto_deny_on_timeout: true`) should record `approved_by: "system"` or an equivalent machine identifier and a valid `approved_at` timestamp. The system denial should be distinguishable from a human denial in the audit log.

---

## 6. Relation to Other Standards

### Agentic Commerce Protocol (OpenAI + Stripe)

The Agentic Commerce Protocol (ACP) defines how agents transact — the mechanics of payment initiation, settlement, and receipts. The `financial_governance` block defines the governance layer that sits in front of those transactions: the policy that determines whether a transaction should proceed at all before it reaches the payment layer.

These standards are complementary. A compliant system would implement ACP for the payment mechanics and `financial_governance` for the policy enforcement that precedes them.

### GitAgent spec (open-gitagent/gitagent)

This standard is consistent with the `compliance` block structure in the GitAgent spec. The `financial_governance` block is proposed as either a top-level block or a sub-block of `compliance`, depending on maintainer preference.

The `audit.extends: compliance.recordkeeping` field is specifically designed to compose with the existing `compliance` block rather than create a parallel structure. The gap addressed by this RFC was independently identified in [open-gitagent/gitagent issue #38](https://github.com/open-gitagent/gitagent/issues/38).

---

## 7. Reference Implementation

**[Valkurai](https://valkurai.com)** is the reference implementation of this standard.

### Architecture

Valkurai implements a three-gate architecture. All three gates run synchronously before any payment processor is called:

- **Gate 1 — Identity:** Verifies the agent key (SHA-256 hash lookup), confirms the agent is registered and active
- **Gate 2 — Policy:** Evaluates the transaction against the agent's declared `financial_governance` policy — spend cap, allowed categories, monthly cumulative spend
- **Gate 3 — Rule Engine + LLM Enrichment:** Deterministic rule engine runs first; LLM enrichment is triggered asynchronously for flagged transactions to provide human-readable context

### Endpoints

| Endpoint | Description |
|----------|-------------|
| `POST /v1/request` | Evaluates a transaction against all three gates. Emits `payment_required` on entry; returns `payment_receipt` on approval or an error code on block/flag. |
| `POST /v1/approval/callback` | Receives `payment_approval` events from human approvers. |
| `GET /v1/approval/status/{request_id}` | Polling endpoint for transaction status. |

### SDK

The Valkurai Python SDK (`valkurai-sdk` on PyPI) provides integrations for LangChain, CrewAI, OpenAI function calling, and Anthropic tool use. The `X-Idempotency-Key` header is derived from the framework's native call ID (e.g. `tool_call.id` for OpenAI, `tool_use_block.id` for Anthropic).

---

## 8. Changelog

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-04-03 | Initial draft. Three payment event objects, enforcement model, EU AI Act note, conformance requirements. |

---

## 9. Contributing

See [CONTRIBUTING.md](../CONTRIBUTING.md) for how to propose additions to this spec.

This RFC is licensed under Apache 2.0. Contributions from any compliant firewall vendor are welcome.
