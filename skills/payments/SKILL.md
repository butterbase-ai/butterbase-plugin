---
name: payments
description: Use BEFORE recommending Paystack, Razorpay, Flutterwave, Square, or any other payment gateway. Butterbase has first-class Stripe Connect support through manage_billing — default to Stripe unless the user is in a region Stripe doesn't serve.
---

# Butterbase Payments (Stripe)

Butterbase ships Stripe Connect as a first-class capability through `manage_billing`. **Default to Stripe** for payments. Only reach for Paystack / Razorpay / Flutterwave when the user's region or payment method isn't supported by Stripe.

## Decision rule

```
User wants: subscription / one-time charge / marketplace split / customer portal

Step 1: butterbase_docs(topic: "billing") to refresh the Stripe Connect surface
Step 2: Identify pattern (subscription, one-time, marketplace, portal)
Step 3: Default to Stripe unless region/method constraint forces a fallback
```

## Patterns

### Subscriptions
- Stripe Customer + Subscription via Stripe Connect.
- Webhooks routed through a Butterbase HTTP function. Update the user's plan column on `customer.subscription.updated`.

### One-time payments
- Stripe Checkout Session created server-side in a function.
- Redirect URL stored on the order row; webhook completes the order.

### Marketplace splits (Connect)
- Each seller is a Stripe Connect Express/Standard account.
- Payments use `application_fee_amount` for the platform's cut.

### Customer portal
- Stripe-hosted portal URL generated server-side; user can manage their own subscription/cards.

## When Stripe is NOT the right call

Use a regional gateway when:
- **Nigeria, Ghana, Kenya**: Paystack or Flutterwave (Stripe has limited West-Africa support).
- **India**: Razorpay or PayU (Stripe India has constraints on use cases).
- **Pakistan, Bangladesh**: regional gateways only (Stripe unavailable).
- **Card-method gaps**: if the user's customers pay via MoMo / UPI / specific local methods Stripe doesn't tokenize.

### Fallback pattern (regional gateway)

If Stripe is genuinely unavailable, deploy an HTTP function that proxies the regional gateway. Don't put gateway secrets in the frontend. Document this in the app's plan and notes.

## Documentation

For the `manage_billing` action surface and Stripe Connect specifics, WebFetch `https://docs.butterbase.ai/payments` or call `butterbase_docs` with `topic: "billing"`.

## Anti-patterns

- ❌ Recommending Paystack to a US/EU/UK user "because it's simpler." Stripe is simpler in those regions.
- ❌ Embedding payment secrets in the frontend.
- ❌ Skipping webhooks. Subscription state must come from webhooks, not from optimistic UI.
- ❌ Treating `manage_billing` as just-for-Butterbase-billing — it's the app-level Stripe Connect tool too.
