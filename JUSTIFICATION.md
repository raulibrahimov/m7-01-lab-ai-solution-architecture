# Justification — Scenario A: Real-time Fraud Scoring

## Serving pattern: online (synchronous)

I picked an **online, synchronous request/response** pattern. The scenario forces
it directly: the app must "score every transaction for fraud risk **before the user
sees confirmation**," and the downstream actions are block / allow / step-up-auth.
The score is on the critical path of a single user action, so it cannot be a batch
job (a precomputed table can't score a transaction that doesn't exist yet) and it
can't be fire-and-forget streaming (the user is blocked waiting for the decision).
The decision must be returned in-band.

Streaming still appears, but **off the request path**: transaction and auth events
flow through an event bus into a feature pipeline that keeps the online feature
store fresh. This is what lets "account history and device fingerprint" be available
at scoring time without recomputing them inside the 80 ms budget. Training is
**batch**, scheduled and drift-triggered, because labels (chargebacks, confirmed
fraud) arrive days later and have no real-time value.

## Where inference runs: cloud

Inference runs **cloud-side**, co-located with the online feature store. The model
needs "account history and device fingerprint" — server-side state that an edge
client doesn't hold and shouldn't (sending full account history to the device is a
data-exposure and tamper risk). A round trip to a regional endpoint fits comfortably
in 80 ms when the feature lookup and model are co-located. Edge would only help if
features were purely on-device; they aren't.

## Targets: optimize latency + correctness, budget throughput

- **Optimize — latency.** p95 must stay under **80 ms end-to-end** because the user
  is staring at a payment-confirmation screen; every added millisecond is abandoned
  checkouts. This is the hard SLO the gateway enforces with a timeout.
- **Optimize — correctness (false-positive rate).** Blocking a legitimate payment is
  a lost sale and an angry customer; the step-up-auth path exists precisely to avoid
  hard-blocking ambiguous cases. I tune the decision thresholds to keep the
  false-positive (false-block) rate low while catching fraud.
- **Budget — throughput.** ~300 tps peak is a known, bounded number. I size for it
  (with headroom) and treat it as a capacity constraint to provision against, not a
  metric to push. The stateless model service scales horizontally if peaks grow.

Cost is the explicit trade-off I accept: keeping a warm, low-latency online stack
sized for peak is more expensive than a batch job, but the latency SLO is
non-negotiable for this use case.

## Fallback when the model is unavailable or wrong

The gateway enforces the 80 ms timeout. If the model service is **slow or down**, it
fails to a **conservative rules-based path**: high-value or high-velocity
transactions route to **step-up-auth** (challenge the user) rather than silently
allowing or hard-blocking everyone. Step-up is the safe middle action — it neither
loses the sale outright nor lets unscored fraud through. If the **feature store**
is unavailable, the model scores on the partial vector and the decision engine
biases toward step-up for anything it can't fully evaluate. Every fallback decision
is emitted to the event bus and flagged in monitoring, so a sustained degradation
raises an alert rather than quietly eroding accuracy.
