# Stripe

Welcome to Aaron's declassified stripe survival guide

Based on lessons learned from countless painful setups, open-source docs, and real production battle scars.

## Prerequisites

You'll need:

- TypeScript (or at least typed JS)
- JS backend (Next.js, Node, etc.)
- Working Auth (validated on backend)
- A KV store (Redis, Upstash, Cloudflare KV… any works)
- Patience (lots of it)

## The Core Problem: Split Brain

Stripe introduces a split brain between:

- Stripe's API (source of truth for payments)
- Your database (source of truth for app state)

Webhooks are supposed to sync the two, but:

- There are 250+ event types (all different shapes).
- Event order is not guaranteed.
- Events can be partial or even wrong.
- Stripe's API is slow (3–10s latency) and rate-limited (100 reads/s).

### Relying on Stripe events directly will inevitably desync your system.

## Philosophy

Avoid partial updates entirely.

Instead:

- Use a single function: `syncStripeDataToKV(customerId: string)`
- Call it everywhere (on success redirect, on webhooks, on-demand).
- Keep all subscription/payment state in your KV, not in Stripe.

This avoids inconsistent states and gives you a fast local cache of billing info.

## The Flow

1. **Frontend**: User clicks "Subscribe" → call backend `generate-stripe-checkout`
2. **Backend**:
   - Create Stripe customer (if not exists)
   - Store `userId → customerId` mapping in KV
   - Create Checkout Session with that `customerId`
3. **User**: Completes payment → redirected to `/success`
4. **Backend (Success Route)**:
   - Call `syncStripeDataToKV(customerId)`
   - Redirect user to app
5. **Backend (Webhook)**:
   - On relevant events → call `syncStripeDataToKV(customerId)`

This is verbose, but it works reliably.

## Checkout Flow Example

```typescript
export async function GET(req: Request) {
  const user = auth(req);

  // Get Stripe customerId from KV
  let stripeCustomerId = await kv.get(`stripe:user:${user.id}`);

  if (!stripeCustomerId) {
    const newCustomer = await stripe.customers.create({
      email: user.email,
      metadata: { userId: user.id }, // IMPORTANT
    });

    await kv.set(`stripe:user:${user.id}`, newCustomer.id);
    stripeCustomerId = newCustomer.id;
  }

  const checkout = await stripe.checkout.sessions.create({
    customer: stripeCustomerId,
    success_url: "https://yourapp.com/success",
    mode: "subscription",
    line_items: [{ price: process.env.STRIPE_PRICE_ID, quantity: 1 }],
  });

  return redirect(checkout.url!);
}
```

### ALWAYS CREATE A CUSTOMER BEFORE CHECKOUT.

Stripe's "ephemeral customer" design is a trap.

## syncStripeDataToKV

This function fetches latest subscription state and caches it.

```typescript
export async function syncStripeDataToKV(customerId: string) {
  const subscriptions = await stripe.subscriptions.list({
    customer: customerId,
    limit: 1,
    status: "all",
    expand: ["data.default_payment_method"],
  });

  if (subscriptions.data.length === 0) {
    const subData = { status: "none" };
    await kv.set(`stripe:customer:${customerId}`, subData);
    return subData;
  }

  const subscription = subscriptions.data[0];
  const subData = {
    subscriptionId: subscription.id,
    status: subscription.status,
    priceId: subscription.items.data[0].price.id,
    currentPeriodEnd: subscription.current_period_end,
    cancelAtPeriodEnd: subscription.cancel_at_period_end,
    paymentMethod:
      subscription.default_payment_method &&
      typeof subscription.default_payment_method !== "string"
        ? {
            brand: subscription.default_payment_method.card?.brand ?? null,
            last4: subscription.default_payment_method.card?.last4 ?? null,
          }
        : null,
  };

  await kv.set(`stripe:customer:${customerId}`, subData);
  return subData;
}
```

## Success Route

```typescript
export async function GET(req: Request) {
  const user = auth(req);
  const stripeCustomerId = await kv.get(`stripe:user:${user.id}`);
  if (!stripeCustomerId) return redirect("/");

  await syncStripeDataToKV(stripeCustomerId);
  return redirect("/dashboard");
}
```

**Reason**: Users often return before webhooks arrive.

This prevents "ghost free-tier" state after checkout.

## Webhooks

```typescript
export async function POST(req: Request) {
  const body = await req.text();
  const signature = (await headers()).get("Stripe-Signature");

  if (!signature) return NextResponse.json({}, { status: 400 });

  const event = stripe.webhooks.constructEvent(
    body,
    signature,
    process.env.STRIPE_WEBHOOK_SECRET!
  );

  if (allowedEvents.includes(event.type)) {
    const customerId = (event.data.object as any).customer;
    if (typeof customerId === "string") {
      await syncStripeDataToKV(customerId);
    }
  }

  return NextResponse.json({ received: true });
}
```

**Tracked Events**:

```typescript
const allowedEvents: Stripe.Event.Type[] = [
  "checkout.session.completed",
  "customer.subscription.created",
  "customer.subscription.updated",
  "customer.subscription.deleted",
  "invoice.paid",
  "invoice.payment_failed",
  "payment_intent.succeeded",
  "payment_intent.payment_failed",
];
```

## Custom Subscription Type

```typescript
export type STRIPE_SUB_CACHE =
  | {
      subscriptionId: string | null;
      status: Stripe.Subscription.Status;
      priceId: string | null;
      currentPeriodStart: number | null;
      currentPeriodEnd: number | null;
      cancelAtPeriodEnd: boolean;
      paymentMethod: {
        brand: string | null;
        last4: string | null;
      } | null;
    }
  | {
      status: "none";
    };
```

## Tips

- **Disable Cash App Pay**
  - 90%+ of canceled/failed payments come from this. It's scam-prone.
- **Enable "Limit customers to one subscription"**
  - Hidden setting under Settings → Checkout and Payment Links → Subscriptions.
  - Prevents users from double-subscribing in multiple tabs.
- **Log aggressively**
  - Stripe events suck. Logs will save you.

## Things Still Your Problem

Stripe won't solve these for you:

- Managing `STRIPE_SECRET_KEY` / `STRIPE_PUBLISHABLE_KEY` across dev & prod
- Managing Price IDs (they differ between dev & prod)
- Tracking usage-based billing (Stripe's API has bizarre behavior here)
- Handling free trials
- Exposing sub data from KV to your UI (just make a `/billing` endpoint)

## Alternatives Worth Considering

If Stripe drives you insane, these are worth a look:

- **Lemon Squeezy** → Merchant of Record, handles taxes and compliance for you.
- **Polar** → Open source wrapper, much simpler API, built on Stripe under the hood.
