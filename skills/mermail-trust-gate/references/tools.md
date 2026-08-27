# Tools

**This skill owns no MCP tools.** It claims no entry in `tool-coverage.json` and must not
be added to `domains` or `walletScopedDomains`. It reads one third-party HTTP endpoint and
emits a verdict; every wallet and payment tool stays with its existing owner.

## Ownership boundary

| Concern | Owner |
| --- | --- |
| Pre-spend verdict on a counterparty | this skill (HTTP only, no MCP tool) |
| Paying an x402 resource, proofs, redemption | `mermail-x402-agent` |
| Wallet inspect, fund, transfer, swap | `mermail-agent-wallet` |
| Reading the mail a payment request arrived in | `mermail-agent-inbox` |

A verdict is not an authorization. This skill never creates a proof, never signs, and never
calls a `paybox_*` tool. On `allow` it hands off; it does not execute.

## HTTP surface

| Call | Auth | Purpose | Risk |
| --- | --- | --- | --- |
| `POST /v1/intel/preflight` | none | free pre-spend verdict | read |
| `GET /v1/intel/trust/{pubkey}` | x402 (HTTP 402) | paid signed receipt | external-effect (spends) |

Base origin `https://intel.twzrd.xyz`. Rate limited per anon caller (`x-twzrd-door-limit`,
120/min at time of writing); honour `x-twzrd-door-remaining` and back off rather than retry
in a loop.

## Request

```json
{
  "seller_wallet": "CnTmHDXVEafkc8sFSzNky9w5zwk63Bk2mHZZodorjhvR",
  "price_usdc": 0.01,
  "agent_intent": "preflight"
}
```

Pass `price_usdc` as a **native JSON number**, not a string. Send the price actually about
to be authorized — the returned cap is amount-sensitive and a cap fetched for a different
amount is not comparable.

## Response fields that drive the decision

| Field | Meaning |
| --- | --- |
| `decision` | `allow` \| `warn` \| `block`. `null` means the gate did not evaluate |
| `trust_score` | 0–100. Meaningless when `null_reason` is set |
| `null_reason` | `insufficient_signal` means unmeasured, not clean |
| `can_spend` | false blocks the amount that was sent, not all amounts |
| `recommended_cap_usdc` | ceiling, never a quote |
| `basis` | what the score was computed from |
| `caveats` | human-readable reasons; quote these, do not paraphrase into a grade |
| `expires_at` | re-request rather than reuse a stale card |

The paid receipt is escalation, not the default. Never call it without the spend having
been authorized under the normal `mermail-x402-agent` approval contract.
