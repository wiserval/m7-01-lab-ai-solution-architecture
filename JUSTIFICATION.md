**Serving pattern**

The scenario says 600ms p95 and "some content goes straight through." That second part is the tell - there's a fast path and a slow path, and the fast path has a time limit. That's not batch. Batch means you collect a pile of things and process them on a schedule, like Monday morning churn scores. Nobody's waiting. Here, someone just uploaded something and the platform needs to decide what to do with it before showing it. So it has to be online - each upload triggers the pipeline individually. The async part just means the user gets an immediate "we got it" instead of staring at a spinner. The moderation still runs right away, just not while they're blocked waiting.

**Where inference runs**

Cloud, for two reasons. First, a decent image classification model is large - we're talking hundreds of megabytes. You can't ship that to an edge device. Second, the load swings between 2 uploads per second on average and 20 at peak. That's a 10x difference. Cloud auto-scaling absorbs that without us having to provision for worst case 24/7. And 600ms is generous - a cloud inference call for image classification comes back in well under 200ms on a warm endpoint. There's room to spare.

**What we're optimizing for**

Latency and recall. Latency because 600ms is the stated budget and the auto-path needs to stay under it - if every image goes to human review anyway, the platform loses the point of having a model. Recall because the scenario says mistakes have legal weight. A missed violation that slips through is far more damaging than a safe image getting flagged and reviewed by a human. So when in doubt, the system escalates rather than approves. Throughput is the budget constraint - 20 per second peak tells us what to provision for, but it's not the hard problem here. A standard cloud serving setup handles 20 RPS without breaking a sweat.

**Fallback**

Two kinds of things can go wrong. The model can be unavailable, or the model can be confidently wrong. For unavailability - everything routes to the human review queue. Nothing gets auto-approved and nothing gets silently dropped. The queue grows, human reviewers have more work, but nothing slips through. That's the right trade-off given legal exposure. For the confidently-wrong case - the two-threshold design already handles uncertainty. Anything the model isn't sure about goes to humans anyway. The truly dangerous failure is a high-confidence wrong prediction, like flagging clean content as a violation at scale. Monitoring catches this through escalation rate anomalies and reviewer disagreement signals. When those spike, it triggers retraining.