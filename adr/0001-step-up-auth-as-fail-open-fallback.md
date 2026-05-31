# ADR 0001: Fail to step-up-auth, not allow-or-block, when scoring is unavailable

## Context
Fraud scoring sits synchronously on the checkout path with a hard p95 ≤ 80 ms budget,
so the gateway must decide what to do when the model service or feature store breaches
its timeout. The fallback choice fires precisely during incidents and peaks — the worst
moment to get it wrong — yet it's a design decision, not a runtime one.

## Decision
When scoring is unavailable, slow, or returns on a partial feature vector, the decision
engine routes the transaction to **step-up-auth** (challenge the user) instead of
auto-allowing or hard-blocking. Step-up is treated as the default safe action under
uncertainty. Plain low-risk transactions can still allow on cached/rules signals, but
anything the system can't confidently evaluate gets challenged. Every fallback decision
is tagged and emitted to monitoring.

## Alternatives rejected
- **Fail-open (allow all):** keeps checkout smooth but lets unscored fraud straight
  through during exactly the outages an attacker would probe for. Unacceptable risk.
- **Fail-closed (block all):** safe against fraud but converts every model blip into a
  revenue outage and a flood of angry legitimate customers. Over-corrects.
- **Hold/queue the transaction:** breaks the synchronous UX — the user is waiting on the
  confirmation screen and can't be parked.

## Consequences
- Outages degrade to extra auth friction, not fraud loss or lost sales — the failure
  mode is bounded and recoverable.
- We must keep the step-up (challenge) path independently reliable; it becomes
  load-bearing during incidents and needs its own capacity and testing.
- Attackers could try to *induce* degradation to force mass step-up; monitoring must
  alert on fallback-rate spikes, not just latency.

## Revisit if
- The step-up channel's friction starts measurably driving cart abandonment, or fallback
  events become frequent enough (not rare) that step-up stops being an exceptional path.
