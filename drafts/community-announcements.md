# Community announcement drafts — DO NOT POST without review

These are draft announcements for community channels. Review before posting.
All three reference open-gitagent/gitagent issue #38 as evidence of community validation.
None should be framed as a Valkurai product announcement — frame as an open standard
with Valkurai as the reference implementation.

---

## LangChain Discord — #integrations

> **financial_governance — open spec for AI agent financial constraints**
>
> Following the discussion in [open-gitagent/gitagent issue #38](https://github.com/open-gitagent/gitagent/issues/38), where the absence of a runtime financial enforcement interface in agent specs was independently identified by contributors, we've published a vendor-neutral spec for declarative financial governance in agent systems: **[github.com/Danbi58/gitagent-spec](https://github.com/Danbi58/gitagent-spec)**.
>
> The `financial_governance` YAML block lets you declare spending limits, allowed spend categories, approval thresholds, and audit requirements alongside your agent definition — without hardcoding any runtime config. A payment event schema with three typed objects (`payment_required`, `payment_approval`, `payment_receipt`) gives you a structured audit trail that satisfies EU AI Act Article 9 requirements. Valkurai is the reference implementation, but the `firewall` field is a string identifier — any compliant implementation works.
>
> The Python SDK (`pip install valkurai-sdk`) has a LangChain integration: `create_valkurai_tool(wallet)` drops a `@tool`-decorated function into any agent that submits payments through the firewall before they reach Stripe. Feedback and alternative implementation reports welcome.

---

## CrewAI Discord

> **financial_governance spec — typed financial constraints for autonomous agents**
>
> After [community discussion](https://github.com/open-gitagent/gitagent/issues/38) identified the gap between compliance declarations and runtime financial enforcement in agent specs, we've published a vendor-neutral open standard: **[github.com/Danbi58/gitagent-spec](https://github.com/Danbi58/gitagent-spec)**.
>
> The spec adds a `financial_governance` block to agent YAML definitions — spend caps, category whitelists/blocklists, approval thresholds, and a structured payment event schema that closes the audit loop. The standard is intentionally not tied to any single firewall vendor. Valkurai is the reference implementation; the `firewall` field accepts any string identifier. For CrewAI specifically, `ValkuraiPayTool` is a `BaseTool` subclass with a Pydantic v2 input schema — three lines to add financial guardrails to any agent (`pip install 'valkurai-sdk[crewai]'`). Full spec, examples, and conformance requirements in the repo.

---

## Hacker News — Show HN

**Title:** Show HN: financial_governance — open spec for declarative AI agent financial constraints

> After noticing that AI agent specs (GitAgent, AutoGPT, etc.) have compliance blocks for identity and audit logging but nothing for runtime financial enforcement, I opened [a discussion in the open-gitagent repo](https://github.com/open-gitagent/gitagent/issues/38). Contributors independently confirmed the gap and endorsed the approach, so I've written it up as a vendor-neutral open standard: https://github.com/Danbi58/gitagent-spec.
>
> The core idea: a `financial_governance` YAML block that declares the policy contract — spend caps, allowed categories, approval thresholds — separately from the runtime wiring (endpoint URL, credentials). The block is a service binding, not a policy engine. A conformant runtime calls the declared firewall synchronously before any financial action and treats its response as authoritative. It doesn't re-implement the logic locally.
>
> The spec includes a three-object payment event schema (`payment_required`, `payment_approval`, `payment_receipt`) designed specifically to satisfy EU AI Act Article 9 requirements. The `payment_approval` object requires `approved_by` (email) and `approved_at` (ISO 8601 timestamp) as non-nullable fields — a log entry that says "approved: true" without a named human and timestamp doesn't satisfy Article 9. The `payment_schema_version` field makes this auditable over time as the schema evolves.
>
> Valkurai (https://valkurai.com) is the reference implementation — three-gate architecture (identity, policy, LLM-enriched rule engine) all running synchronously before Stripe is called. But the `firewall` field is a plain string identifier, not an enum, so any compliant implementation works. Apache 2.0. Feedback and alternative implementations welcome.
