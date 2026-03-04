# Stripe Order Recovery: Retries, Refunds, and Not Lying to Users

If you charge money up front and fulfill asynchronously, you will eventually ship the worst user experience in SaaS:

> “Thanks for your payment. Something went wrong. Please contact support.”

Usually it’s not fraud or maliciousness. It’s just reality:

- webhooks arrive late (or twice)
- background jobs crash
- timeouts happen mid-fulfillment
- external APIs rate-limit you
- “success” pages get refreshed 20 times

This is a practical design for “order recovery”: how to detect stuck orders and resolve them (retry, refund, or restore credits) without double-charging, double-fulfilling, or lying to the user.

## TL;DR

- Separate **payment truth** (Stripe) from **fulfillment truth** (your system).
- Model an explicit **order state machine** and treat it like an API contract.
- Make fulfillment **idempotent**, because your triggers will repeat.
- Implement at least one recovery loop:
  - a scheduled job/cron command (recommended)
  - or a queue worker that can reprocess stale jobs
- Decide your policy up front: when to retry vs refund.
- Write the runbook. You’re going to need it at 2am.

## The core problem

The hard part is not taking payments. Stripe is good at that.

The hard part is what happens after payment:

1) user pays
2) you start work (generate a report, run a job, create a dataset, provision infra)
3) you deliver the artifact

Steps 2–3 can fail, and you must handle it in a way that keeps trust intact.

## Model it: an order state machine you can explain

You want states that mean something operationally. Here’s a common baseline:

- `pending_payment`: order exists, payment not confirmed
- `paid`: Stripe says paid, fulfillment not started (or not confirmed started)
- `processing`: fulfillment has started
- `completed`: artifact delivered / user can access it
- `failed`: fulfillment failed and will not auto-retry (without human action)
- `refunded`: payment was refunded (optional separate state; some teams keep “failed + refunded_at”)

The important part is not the names. It’s the invariants.

Example invariants:

- You must never transition *out* of `completed` unless you have a clear “revocation” policy.
- `refunded` must be terminal (or extremely controlled).
- `processing` must have a timeout policy (“stuck” definition).

## Separate payment truth from fulfillment truth

Stripe knows whether a charge succeeded. Your system knows whether the user got what they paid for.

Do not collapse these into one boolean like `paid=true`.

What you want stored for each order:

- internal `order_id`
- `stripe_session_id` (Checkout) and/or `payment_intent_id`
- order timestamps (`created_at`, `updated_at`, `paid_at`, etc.)
- `status`
- an error message field (for debugging + support)

## Idempotency: assume every trigger repeats

Here are the triggers that will happen multiple times:

- Stripe webhook delivery (retries, duplicates, test events)
- user refreshes the success page
- your worker retries the job
- a recovery script runs twice

So your fulfillment function must be safe to call repeatedly.

In pseudocode:

```ts
async function processOrder(orderId: string) {
  const order = await db.getOrderForUpdate(orderId); // lock row

  if (order.status === "completed") return; // idempotent “already done”
  if (order.status === "refunded") return;  // terminal

  if (order.status === "pending_payment") {
    throw new Error("cannot process unpaid order");
  }

  // transition to processing once
  if (order.status !== "processing") {
    await db.updateOrder(orderId, { status: "processing", startedAt: now() });
  }

  // do the actual fulfillment work
  const artifact = await fulfill(order);

  await db.updateOrder(orderId, { status: "completed", artifactRef: artifact.ref });
}
```

Notes:

- “row lock” is important: without it, two workers can both fulfill.
- treat `completed` as a short-circuit.

## Define “stuck” clearly (and make it measurable)

“Stuck” usually means:

- `status=processing` and `created_at < now - threshold`

…or:

- `status=paid` and the processing never started within a threshold

Choose a threshold that matches your workload. If fulfillment usually takes 30 seconds, “stuck after 24 hours” is not a recovery loop—it’s an apology loop.

Also: record *why* you think it’s stuck. That matters when you debug patterns later.

## Recovery mechanisms (pick at least one)

### 1) Scheduled recovery job (recommended)

This is the simplest operationally:

- every N minutes
- find stuck orders
- re-run `processOrder(orderId)`
- if it fails again, decide to refund/credit and mark failed

This can be:

- a cron job
- a platform scheduler (Render, Fly, ECS scheduled task, etc.)
- a “management command” if you’re in Django

Why I like it:

- easy to reason about
- easy to run manually
- easy to add logging + alerting

### 2) Queue worker with retry/backoff

If you already have a queue:

- enqueue fulfillment jobs when payment is confirmed
- auto-retry on transient errors (rate limits, timeouts)
- dead-letter hard failures for manual review

Even then, you still want a “sweeper” that detects stuck items (jobs can be dropped, workers can die, etc.).

### 3) Login-time recovery (nice extra, not a replacement)

When a user logs in, you can:

- query for that user’s recent `paid/processing` orders
- try a bounded retry

This is helpful as a “self-healing” path, but do not rely on it as your only recovery. Users shouldn’t have to log in to get what they paid for.

## Refund vs retry: decide your policy before you need it

When recovery fails, you have two options:

- Retry later (and keep the order alive)
- Refund/credit and stop

My rule of thumb:

- Retry on likely-transient faults (timeouts, rate limits, temporary upstream failure).
- Refund on repeated failures or on faults that indicate the payload can’t be processed.

And say it clearly to users:

- “We’re retrying your order. No action needed.” (and mean it)
- “We refunded you because we couldn’t complete processing.” (and mean it)

## Practical refund handling (Stripe Checkout)

For Checkout-based flows, the common path is:

- order stores `stripe_session_id`
- recovery script loads the session
- reads `payment_intent`
- creates a refund

That’s not “Stripe magic”; it’s just mapping your internal order to the Stripe payment object you need to refund.

Also: make refunds idempotent too. Store `refunded_at` (and optionally `stripe_refund_id`) so a second recovery attempt doesn’t refund twice.

## Observability: treat recovery as a product feature

You want:

- logs that show: “stuck order found → retry attempt → success/failure → refunded”
- counters for how often it happens
- alerting when stuck orders exceed a threshold

If this is a paid product, “silent failure” is not acceptable. Recovery is part of your customer experience.

## Failure modes you should assume will happen

- Webhook delivered twice → fulfillment triggered twice
- User refreshes success page 10 times → fulfillment triggered 10 times
- Worker crashes after generating artifact but before setting `completed` → “phantom failure”
- Refund API call fails (Stripe outage or misconfig) → order is failed but money not returned
- Partial fulfillment (one file generated, another missing) → “completed” must reflect reality

If you can’t explain how your system behaves for each of these, you don’t have an order recovery design yet.

## Concrete reference (code + docs)

I built and documented an order recovery system (Django + Stripe) here:

- README: https://github.com/tuckerjensendev/LinqCV
- Order Recovery doc: https://github.com/tuckerjensendev/LinqCV/blob/main/ORDER_RECOVERY_README.md

## SEO notes (optional)

If I were publishing this as a blog post, I’d ship with:

- Suggested title: “Stripe Order Recovery: How to Fix Stuck Checkouts (Retries, Refunds, Idempotency)”
- Meta description: “A practical backend guide for Stripe Checkout flows: order state machines, idempotent fulfillment, scheduled recovery jobs, and refund/credit policies that keep user trust intact.”
- Internal links:
  - To a post on idempotency keys and webhook validation
  - To a post on background jobs and retry/backoff strategies
