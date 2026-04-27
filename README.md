# financial_governance — Open standard for AI agent financial governance interfaces

A vendor-neutral specification for declarative financial governance in autonomous AI agent systems.

## Overview

As AI agents gain the ability to initiate financial transactions autonomously, agent specifications need a typed interface for declaring runtime financial constraints. The `financial_governance` block fills this gap.

This standard defines:

- A declarative YAML configuration block for financial governance
- A payment event schema with three typed objects (`payment_required`, `payment_approval`, `payment_receipt`)
- Conformance requirements for compliant agent runtimes
- EU AI Act compliance guidance for payment approval audit trails

## Origin

This standard emerged from community discussion in [open-gitagent/gitagent-protocol issue #38](https://github.com/open-gitagent/gitagent-protocol/issues/38), where the absence of a runtime financial enforcement interface in agent specs was independently identified and verified by contributors. The core proposal was endorsed as technically sound by that community.

It is consistent with the [Agentic Commerce Protocol](https://openai.com/index/introducing-the-model-spec/) (OpenAI + Stripe), which defines how agents transact. The `financial_governance` block defines the governance layer that sits in front of those transactions.

## Specification

See [`spec/financial_governance.md`](spec/financial_governance.md) for the full RFC.

## Example

```yaml
financial_governance:
  enabled: true
  firewall: valkurai
  payment_schema_version: "1.0"

  spending:
    max_per_transaction_cents: 5000
    max_monthly_cents: 200000
    currency_default: USD
    allowed_categories:
      - software
      - api
      - saas
      - cloud
```

See [`examples/purchasing-agent/agent.yaml`](examples/purchasing-agent/agent.yaml) for a complete working example.

## Reference Implementation

[Valkurai](https://valkurai.com) is the reference implementation of this standard. It implements the three-gate architecture (identity, policy, deterministic rule engine with async LLM enrichment) described in the specification.

Any compliant financial firewall implementation may be referenced in the `firewall` field. The standard is intentionally vendor-neutral.

## Status

**v1.0 — Draft**

Community feedback welcome. See [CONTRIBUTING.md](CONTRIBUTING.md) for how to propose additions.

## License

Apache 2.0 — see [LICENSE](LICENSE).
