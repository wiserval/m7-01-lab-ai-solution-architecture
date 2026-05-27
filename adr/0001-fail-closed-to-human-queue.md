# ADR 0001: Fail Closed to Human Queue

**Context**

The ML model runs on a cloud endpoint. Cloud services go down sometimes, not often but they do. When that happens at 2am, the system still has uploads coming in. It needs to know what to do with them. And because this platform has legal exposure if the wrong content gets through, that's not a question you want to answer in the moment.

**Decision**

When the model endpoint is unavailable, everything goes to the human review queue. Nothing gets auto-approved. The queue grows, reviewers have more work, but no image slips through unreviewed. The UI tells the uploader their content is under review, same as always - they don't see a failure message.

**Alternatives rejected**

**Fail open - auto-approve everything during outages.** User experience stays smooth, uploads go through. But you're potentially publishing violating content with zero review. Given legal exposure, this is the one outcome you can't defend.

**Fail closed - auto-reject everything during outages.** Nothing bad gets through, but you're also blocking legitimate uploads with no path to appeal. A long outage would auto-reject a huge volume of clean content. That's operationally indefensible too.

**Serve from a cached fallback model.** Images are unique - there's no prior score to fall back on. A stale model serving fresh images isn't a fallback, it's just a degraded model making decisions it shouldn't.

**Consequences**

The human review queue is now a critical component, not a secondary channel. It has to stay available even when the ML layer doesn't. That changes how you think about its infrastructure.

Reviewer capacity planning has to account for outage scenarios. If the model goes down for two hours at peak upload time, how many items pile up? That math needs to be done before deployment.

Uploaders experience longer waits during outages. The UI needs to communicate that clearly without revealing system state.

**Revisit if**

The ML serving layer reaches a 99.99%+ SLA with sub-second failover, making outages rare enough that queue growth during them is operationally negligible.