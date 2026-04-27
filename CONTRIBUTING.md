# Contributing to financial_governance

This standard emerged from open community discussion — specifically [open-gitagent/gitagent-protocol #38](https://github.com/open-gitagent/gitagent-protocol/issues/38) — and welcomes contributions from any implementor, researcher, or practitioner working on AI agent financial governance.

## What kinds of contributions are welcome

- **New patterns** — real-world configurations that the current spec does not cover well
- **Clarifications** — places where the spec is ambiguous or underspecified
- **Implementation reports** — notes from implementing a conformant firewall or runtime integration
- **Corrections** — factual errors, broken examples, outdated references
- **Additional event schema fields** — proposed additions to `payment_required`, `payment_approval`, or `payment_receipt`
- **New compliance notes** — regulatory frameworks beyond EU AI Act Article 9 that have structured requirements for AI-initiated payments

Contributions from any compliant firewall vendor are welcome. The `firewall` field is intentionally a string identifier, not an enum — no changes to this spec are required to reference a new implementation.

## Process

### For small changes (typos, clarifications, broken examples)

Open a pull request directly. No issue required.

### For substantive changes (new fields, modified conformance requirements, new sections)

1. **Open an issue first** using the template below
2. Allow at least 2 weeks for community discussion
3. Open a pull request that references the issue

### Issue template for new patterns

```markdown
## Pattern proposal

**What problem does this address?**
<!-- Describe the real-world scenario that the current spec doesn't handle well -->

**Proposed addition**
<!-- YAML or schema fragment showing the proposed change -->

**Conformance impact**
<!-- Does this change the conformance requirement? Would existing conformant implementations need to update? -->

**Regulatory relevance**
<!-- Does this have EU AI Act, PCI DSS, or other regulatory implications? -->

**Implementation notes**
<!-- Have you implemented this pattern? What did you learn? -->
```

## What this spec does not own

The following are out of scope for this specification:

- **Transport and authentication** — how the agent authenticates to the firewall, how the request is signed, and what transport layer is used. These belong in runtime configuration or a separate `integrations` block, never in `agent.yaml`.
- **Firewall implementation details** — how a specific firewall evaluates the policy. The spec defines the interface; the firewall owns the implementation.
- **Orchestration platform internals** — how a specific agent framework invokes the firewall. The conformance requirement is that it is called synchronously before any financial action.

## Code of conduct

This project follows the [Contributor Covenant v2.1](https://www.contributor-covenant.org/version/2/1/code_of_conduct/) code of conduct.

## Licence

By submitting a contribution, you agree that your contribution will be licensed under the Apache 2.0 licence that covers this repository.
