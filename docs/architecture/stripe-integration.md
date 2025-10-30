[Back to Index](./index.md) | [Previous: Subscription Billing](./subscription-billing.md) | [Next: Monitoring](./monitoring.md)

# Stripe Integration Guide

## Overview

This document provides detailed implementation guidance for integrating Stripe payment processing into FinDogAI. It covers setup, Cloud Functions implementation, webhook handling, security best practices, and testing strategies.

## Stripe Account Setup

### 1. Create Stripe Account

1. Go to https://dashboard.stripe.com/register
2. Sign up with business email
3. Complete business profile:
   - Business name: FinDogAI
   - Country: Czech Republic
   - Business type: Software/SaaS
   - Website: https://findogai.app

### 2. Activate Account

1. Verify email address
2. Complete identity verification (required for payouts)
3. Add bank account for payouts
4. Configure business settings

### 3. API Keys

Navigate to Developers → API Keys:

```bash
# Test Mode Keys (for development)
STRIPE_TEST_PUBLISHABLE_KEY=pk_test_xxxxxxxxxxxxx
STRIPE_TEST_SECRET_KEY=sk_test_xxxxxxxxxxxxx

# Live Mode Keys (for production)
STRIPE_LIVE_PUBLISHABLE_KEY=pk_live_xxxxxxxxxxxxx
STRIPE_LIVE_SECRET_KEY=sk_live_xxxxxxxxxxxxx
```

**Security:**
- **NEVER** commit secret keys to git
- Store in Firebase Functions config or Secret Manager
- Rotate keys if compromised

### 4. Create Products & Prices

**Via Stripe Dashboard:**

Products → Add Product:

```yaml
Product 1:
  Name: FinDogAI Pro
  Description: Professional plan with unlimited features
  Pricing:
    - Recurring: Monthly
    - Price: €29.00 EUR
    - Price ID: price_xxxxxxxxxxxxx (copy this for code)
    - Trial period: 14 days

Product 2:
  Name: FinDogAI Enterprise
  Description: Custom enterprise solution
  Pricing: Custom (handled manually)
```

**Via Stripe API (optional):**
```typescript
const product = await stripe.products.create({
  name: 'FinDogAI Pro',
  description: 'Professional plan with unlimited features',
});

const price = await stripe.prices.create({
  product: product.id,
  unit_amount: 2900, // €29.00 in cents
  currency: 'eur',
  recurring: {
    interval: 'month',
    trial_period_days: 14,
  },
});

console.log('Price ID:', price.id); // Save this in environment config
```

### 5. Configure Webhooks

Developers → Webhooks → Add endpoint:

**Development Endpoint:**
```
URL: https://europe-west1-findogai-dev.cloudfunctions.net/handleStripeWebhook
Description: Dev environment webhook handler
```

**Production Endpoint:**
```
URL: https://europe-west1-findogai-prod.cloudfunctions.net/handleStripeWebhook
Description: Production webhook handler
```

**Events to listen for:**
```
✓ customer.subscription.created
✓ customer.subscription.updated
✓ customer.subscription.deleted
✓ customer.subscription.trial_will_end
✓ invoice.paid
✓ invoice.payment_failed
✓ invoice.payment_action_required
✓ payment_intent.succeeded
✓ payment_intent.payment_failed
```

**Get Webhook Signing Secret:**
```bash
# Webhook signing secret (for verifying webhook authenticity)
STRIPE_WEBHOOK_SECRET=whsec_xxxxxxxxxxxxx
```

## Firebase Functions Configuration

### Environment Variables

**Set via Firebase CLI:**
```bash
# Development
firebase functions:config:set \
  stripe.secret_key="sk_test_xxxxxxxxxxxxx" \
  stripe.publishable_key="pk_test_xxxxxxxxxxxxx" \
  stripe.webhook_secret="whsec_xxxxxxxxxxxxx" \
  stripe.pro_price_id="price_xxxxxxxxxxxxx" \
  --project findogai-dev

# Production
firebase functions:config:set \
  stripe.secret_key="sk_live_xxxxxxxxxxxxx" \
  stripe.publishable_key="pk_live_xxxxxxxxxxxxx" \
  stripe.webhook_secret="whsec_xxxxxxxxxxxxx" \
  stripe.pro_price_id="price_xxxxxxxxxxxxx" \
  --project findogai-prod
```

**Using Secret Manager (recommended for production):**
```bash
# Store in Google Cloud Secret Manager
echo -n "sk_live_xxxxxxxxxxxxx" | gcloud secrets create stripe-secret-key --data-file=-

# Grant access to Cloud Functions
gcloud secrets add-iam-policy-binding stripe-secret-key \
  --member="serviceAccount:findogai-prod@appspot.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

**Access in Cloud Functions:**
```typescript
import { defineSecret } from 'firebase-functions/params';

const stripeSecretKey = defineSecret('STRIPE_SECRET_KEY');

export const myFunction = onCall(
  { secrets: [stripeSecretKey] },
  async (request) => {
    const stripe = new Stripe(stripeSecretKey.value());
    // ...
  }
);
```

### Install Stripe SDK

```bash
cd packages/functions
npm install stripe --save
npm install @types/stripe --save-dev
```

**package.json:**
```json
{
  "dependencies": {
    "stripe": "^14.11.0"
  }
}
```

## Cloud Functions Implementation

### 1. Create Checkout Session

**File:** `packages/functions/src/stripe/createCheckoutSession.ts`

```typescript
import { onCall, HttpsError } from 'firebase-functions/v2/https';
import Stripe from 'stripe';
import * as admin from 'firebase-admin';
import * as logger from 'firebase-functions/logger';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2024-11-20.acacia',
});

interface CreateCheckoutSessionRequest {
  tenantId: string;
  priceId: string;
  successUrl?: string;
  cancelUrl?: string;
}

interface CreateCheckoutSessionResponse {
  checkoutUrl: string;
  sessionId: string;
}

export const createCheckoutSession = onCall<
  CreateCheckoutSessionRequest,
  Promise<CreateCheckoutSessionResponse>
>(async (request) => {
  // 1. Verify authentication
  if (!request.auth) {
    throw new HttpsError('unauthenticated', 'User must be authenticated');
  }

  const { tenantId, priceId, successUrl, cancelUrl } = request.data;

  // 2. Verify tenant membership and owner role
  const memberDoc = await admin.firestore()
    .doc(`tenants/${tenantId}/members/${request.auth.uid}`)
    .get();

  if (!memberDoc.exists || memberDoc.data()?.role !== 'owner') {
    throw new HttpsError(
      'permission-denied',
      'Only tenant owners can manage subscriptions'
    );
  }

  // 3. Get tenant data
  const tenantDoc = await admin.firestore()
    .doc(`tenants/${tenantId}`)
    .get();

  if (!tenantDoc.exists) {
    throw new HttpsError('not-found', 'Tenant not found');
  }

  const tenant = tenantDoc.data()!;

  try {
    // 4. Get or create Stripe customer
    let customerId = tenant.stripeCustomerId;

    if (!customerId) {
      const customer = await stripe.customers.create({
        email: request.auth.token.email,
        metadata: {
          tenantId,
          firebaseUid: request.auth.uid,
        },
      });

      customerId = customer.id;

      // Save customer ID to tenant
      await tenantDoc.ref.update({
        stripeCustomerId: customerId,
        updatedAt: admin.firestore.FieldValue.serverTimestamp(),
      });

      logger.info(`Created Stripe customer ${customerId} for tenant ${tenantId}`);
    }

    // 5. Create checkout session
    const session = await stripe.checkout.sessions.create({
      customer: customerId,
      mode: 'subscription',
      payment_method_types: ['card'],
      line_items: [
        {
          price: priceId,
          quantity: 1,
        },
      ],
      success_url: successUrl || `${process.env.APP_URL}/billing/success?session_id={CHECKOUT_SESSION_ID}`,
      cancel_url: cancelUrl || `${process.env.APP_URL}/billing`,
      subscription_data: {
        trial_period_days: 14,
        metadata: {
          tenantId,
          firebaseUid: request.auth.uid,
        },
      },
      metadata: {
        tenantId,
        firebaseUid: request.auth.uid,
      },
      allow_promotion_codes: true, // Enable promo codes
      billing_address_collection: 'required',
      tax_id_collection: {
        enabled: true, // For EU VAT
      },
    });

    logger.info(`Created checkout session ${session.id} for tenant ${tenantId}`);

    return {
      checkoutUrl: session.url!,
      sessionId: session.id,
    };
  } catch (error: any) {
    logger.error('Failed to create checkout session', error);
    throw new HttpsError('internal', `Failed to create checkout session: ${error.message}`);
  }
});
```

### 2. Webhook Handler

**File:** `packages/functions/src/stripe/webhookHandler.ts`

```typescript
import { onRequest } from 'firebase-functions/v2/https';
import Stripe from 'stripe';
import * as admin from 'firebase-admin';
import * as logger from 'firebase-functions/logger';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);
const webhookSecret = process.env.STRIPE_WEBHOOK_SECRET!;

export const handleStripeWebhook = onRequest(
  {
    region: 'europe-west1',
    memory: '256MiB',
    timeoutSeconds: 60,
  },
  async (req, res) => {
    const sig = req.headers['stripe-signature'];

    if (!sig) {
      logger.warn('Missing stripe-signature header');
      res.status(400).send('Missing signature');
      return;
    }

    let event: Stripe.Event;

    try {
      // Verify webhook signature
      event = stripe.webhooks.constructEvent(
        req.rawBody,
        sig,
        webhookSecret
      );
    } catch (err: any) {
      logger.error('Webhook signature verification failed', err);
      res.status(400).send(`Webhook Error: ${err.message}`);
      return;
    }

    logger.info(`Received Stripe event: ${event.type}`);

    try {
      // Handle event
      switch (event.type) {
        case 'customer.subscription.created':
        case 'customer.subscription.updated':
          await handleSubscriptionChange(event.data.object as Stripe.Subscription);
          break;

        case 'customer.subscription.deleted':
          await handleSubscriptionDeleted(event.data.object as Stripe.Subscription);
          break;

        case 'customer.subscription.trial_will_end':
          await handleTrialWillEnd(event.data.object as Stripe.Subscription);
          break;

        case 'invoice.paid':
          await handleInvoicePaid(event.data.object as Stripe.Invoice);
          break;

        case 'invoice.payment_failed':
          await handleInvoicePaymentFailed(event.data.object as Stripe.Invoice);
          break;

        case 'invoice.payment_action_required':
          await handlePaymentActionRequired(event.data.object as Stripe.Invoice);
          break;

        default:
          logger.info(`Unhandled event type: ${event.type}`);
      }

      res.json({ received: true });
    } catch (error: any) {
      logger.error('Error processing webhook', error);
      res.status(500).send(`Webhook handler failed: ${error.message}`);
    }
  }
);

// Handle subscription created/updated
async function handleSubscriptionChange(subscription: Stripe.Subscription): Promise<void> {
  const tenantId = subscription.metadata.tenantId;

  if (!tenantId) {
    logger.warn('Missing tenantId in subscription metadata', { subscriptionId: subscription.id });
    return;
  }

  const subscriptionData: any = {
    tenantId,
    stripeCustomerId: subscription.customer as string,
    stripeSubscriptionId: subscription.id,
    stripePriceId: subscription.items.data[0].price.id,
    plan: subscription.metadata.plan || 'pro',
    status: subscription.status,
    currentPeriodStart: admin.firestore.Timestamp.fromMillis(subscription.current_period_start * 1000),
    currentPeriodEnd: admin.firestore.Timestamp.fromMillis(subscription.current_period_end * 1000),
    cancelAtPeriodEnd: subscription.cancel_at_period_end,
    updatedAt: admin.firestore.FieldValue.serverTimestamp(),
  };

  // Add trial dates if in trial
  if (subscription.status === 'trialing' && subscription.trial_start && subscription.trial_end) {
    subscriptionData.trialStart = admin.firestore.Timestamp.fromMillis(subscription.trial_start * 1000);
    subscriptionData.trialEnd = admin.firestore.Timestamp.fromMillis(subscription.trial_end * 1000);
  }

  // Add cancellation date if canceled
  if (subscription.canceled_at) {
    subscriptionData.canceledAt = admin.firestore.Timestamp.fromMillis(subscription.canceled_at * 1000);
  }

  await admin.firestore()
    .doc(`tenants/${tenantId}/subscription/default`)
    .set(subscriptionData, { merge: true });

  logger.info(`Updated subscription for tenant ${tenantId}`, { status: subscription.status });
}

// Handle subscription deletion
async function handleSubscriptionDeleted(subscription: Stripe.Subscription): Promise<void> {
  const tenantId = subscription.metadata.tenantId;

  if (!tenantId) {
    logger.warn('Missing tenantId in subscription metadata');
    return;
  }

  await admin.firestore()
    .doc(`tenants/${tenantId}/subscription/default`)
    .update({
      status: 'canceled',
      canceledAt: admin.firestore.FieldValue.serverTimestamp(),
      updatedAt: admin.firestore.FieldValue.serverTimestamp(),
    });

  logger.info(`Subscription canceled for tenant ${tenantId}`);

  // TODO: Send cancellation email
}

// Handle trial ending soon
async function handleTrialWillEnd(subscription: Stripe.Subscription): Promise<void> {
  const tenantId = subscription.metadata.tenantId;

  if (!tenantId) return;

  logger.info(`Trial ending soon for tenant ${tenantId}`);

  // TODO: Send trial ending email (3 days before)
}

// Handle successful payment
async function handleInvoicePaid(invoice: Stripe.Invoice): Promise<void> {
  const tenantId = invoice.subscription_details?.metadata?.tenantId;

  if (!tenantId) {
    logger.warn('Missing tenantId in invoice metadata');
    return;
  }

  // Save payment record
  await admin.firestore()
    .collection(`tenants/${tenantId}/payments`)
    .add({
      tenantId,
      stripeInvoiceId: invoice.id,
      stripePaymentIntentId: invoice.payment_intent as string,
      amount: invoice.amount_paid,
      currency: invoice.currency,
      status: 'paid',
      invoicePdfUrl: invoice.invoice_pdf,
      invoiceNumber: invoice.number,
      periodStart: admin.firestore.Timestamp.fromMillis(invoice.period_start * 1000),
      periodEnd: admin.firestore.Timestamp.fromMillis(invoice.period_end * 1000),
      paidAt: admin.firestore.Timestamp.fromMillis(invoice.status_transitions.paid_at! * 1000),
      createdAt: admin.firestore.FieldValue.serverTimestamp(),
    });

  logger.info(`Payment recorded for tenant ${tenantId}`, { amount: invoice.amount_paid });

  // TODO: Send payment receipt email
}

// Handle failed payment
async function handleInvoicePaymentFailed(invoice: Stripe.Invoice): Promise<void> {
  const tenantId = invoice.subscription_details?.metadata?.tenantId;

  if (!tenantId) return;

  // Update subscription status
  await admin.firestore()
    .doc(`tenants/${tenantId}/subscription/default`)
    .update({
      status: 'past_due',
      updatedAt: admin.firestore.FieldValue.serverTimestamp(),
    });

  logger.warn(`Payment failed for tenant ${tenantId}`);

  // TODO: Send payment failed email
}

// Handle payment requiring action (3D Secure)
async function handlePaymentActionRequired(invoice: Stripe.Invoice): Promise<void> {
  const tenantId = invoice.subscription_details?.metadata?.tenantId;

  if (!tenantId) return;

  logger.info(`Payment requires action for tenant ${tenantId}`);

  // TODO: Send email with payment action link
}
```

### 3. Check Subscription Status

**File:** `packages/functions/src/stripe/checkSubscriptionStatus.ts`

```typescript
import { onCall, HttpsError } from 'firebase-functions/v2/https';
import * as admin from 'firebase-admin';
import * as logger from 'firebase-functions/logger';

interface CheckSubscriptionRequest {
  tenantId: string;
  action: 'create_job' | 'add_team_member' | 'use_voice' | 'export_pdf';
}

interface CheckSubscriptionResponse {
  allowed: boolean;
  reason?: string;
  message?: string;
  redirectTo?: string;
  currentUsage?: any;
  limits?: any;
}

export const checkSubscriptionStatus = onCall<
  CheckSubscriptionRequest,
  Promise<CheckSubscriptionResponse>
>(async (request) => {
  if (!request.auth) {
    throw new HttpsError('unauthenticated', 'User must be authenticated');
  }

  const { tenantId, action } = request.data;

  // Verify tenant membership
  const memberDoc = await admin.firestore()
    .doc(`tenants/${tenantId}/members/${request.auth.uid}`)
    .get();

  if (!memberDoc.exists) {
    throw new HttpsError('permission-denied', 'User is not a member of this tenant');
  }

  // Get subscription
  const subDoc = await admin.firestore()
    .doc(`tenants/${tenantId}/subscription/default`)
    .get();

  if (!subDoc.exists) {
    // No subscription exists - shouldn't happen, but handle gracefully
    return {
      allowed: false,
      reason: 'no_subscription',
      message: 'No subscription found. Please contact support.',
    };
  }

  const subscription = subDoc.data()!;

  // Check subscription status
  const activeStatuses = ['active', 'trialing'];
  if (!activeStatuses.includes(subscription.status)) {
    return {
      allowed: false,
      reason: 'subscription_inactive',
      message: getInactiveMessage(subscription.status),
      redirectTo: '/billing/upgrade',
    };
  }

  // Check action-specific limits
  switch (action) {
    case 'create_job':
      return checkJobLimit(subscription);

    case 'add_team_member':
      return checkTeamMemberLimit(subscription);

    case 'use_voice':
      return checkVoiceLimit(subscription);

    case 'export_pdf':
      return checkPdfExportLimit(subscription);

    default:
      return { allowed: true };
  }
});

function getInactiveMessage(status: string): string {
  const messages: Record<string, string> = {
    past_due: 'Your payment failed. Please update your payment method.',
    canceled: 'Your subscription has been canceled.',
    incomplete_expired: 'Your trial has expired. Upgrade to continue.',
    unpaid: 'Your subscription is unpaid. Please update your payment method.',
  };

  return messages[status] || 'Your subscription is not active.';
}

function checkJobLimit(subscription: any): CheckSubscriptionResponse {
  const limit = subscription.limits.maxJobs;
  const usage = subscription.usage.jobsCreated;

  if (limit === -1) {
    return { allowed: true };
  }

  if (usage >= limit) {
    return {
      allowed: false,
      reason: 'job_limit_reached',
      message: `You've reached your limit of ${limit} jobs. Upgrade to create more.`,
      redirectTo: '/billing/upgrade',
      currentUsage: usage,
      limits: { maxJobs: limit },
    };
  }

  return {
    allowed: true,
    currentUsage: usage,
    limits: { maxJobs: limit },
  };
}

function checkTeamMemberLimit(subscription: any): CheckSubscriptionResponse {
  const limit = subscription.limits.maxTeamMembers;
  const usage = subscription.usage.activeTeamMembers;

  if (limit === -1) {
    return { allowed: true };
  }

  if (usage >= limit) {
    return {
      allowed: false,
      reason: 'team_member_limit_reached',
      message: `You've reached your limit of ${limit} team members. Upgrade for unlimited team members.`,
      redirectTo: '/billing/upgrade',
      currentUsage: usage,
      limits: { maxTeamMembers: limit },
    };
  }

  return {
    allowed: true,
    currentUsage: usage,
    limits: { maxTeamMembers: limit },
  };
}

function checkVoiceLimit(subscription: any): CheckSubscriptionResponse {
  const limit = subscription.limits.maxVoiceMinutesPerMonth;
  const usage = subscription.usage.voiceMinutesThisMonth;

  if (limit === -1) {
    return { allowed: true };
  }

  if (limit === 0) {
    return {
      allowed: false,
      reason: 'voice_not_available',
      message: 'Voice features are not available on your plan. Upgrade to Pro.',
      redirectTo: '/billing/upgrade',
    };
  }

  if (usage >= limit) {
    return {
      allowed: false,
      reason: 'voice_limit_reached',
      message: `You've used all ${limit} voice minutes this month. Upgrade for more minutes.`,
      redirectTo: '/billing/upgrade',
      currentUsage: usage,
      limits: { maxVoiceMinutesPerMonth: limit },
    };
  }

  return {
    allowed: true,
    currentUsage: usage,
    limits: { maxVoiceMinutesPerMonth: limit },
  };
}

function checkPdfExportLimit(subscription: any): CheckSubscriptionResponse {
  if (!subscription.limits.pdfExport) {
    return {
      allowed: false,
      reason: 'pdf_export_not_available',
      message: 'PDF export is not available on your plan. Upgrade to Pro.',
      redirectTo: '/billing/upgrade',
    };
  }

  return { allowed: true };
}
```

### 4. Customer Portal Session

**File:** `packages/functions/src/stripe/createPortalSession.ts`

```typescript
import { onCall, HttpsError } from 'firebase-functions/v2/https';
import Stripe from 'stripe';
import * as admin from 'firebase-admin';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

interface CreatePortalSessionRequest {
  tenantId: string;
  returnUrl?: string;
}

interface CreatePortalSessionResponse {
  portalUrl: string;
}

export const createPortalSession = onCall<
  CreatePortalSessionRequest,
  Promise<CreatePortalSessionResponse>
>(async (request) => {
  if (!request.auth) {
    throw new HttpsError('unauthenticated', 'User must be authenticated');
  }

  const { tenantId, returnUrl } = request.data;

  // Verify owner role
  const memberDoc = await admin.firestore()
    .doc(`tenants/${tenantId}/members/${request.auth.uid}`)
    .get();

  if (!memberDoc.exists || memberDoc.data()?.role !== 'owner') {
    throw new HttpsError('permission-denied', 'Only owners can access billing portal');
  }

  // Get customer ID
  const tenantDoc = await admin.firestore()
    .doc(`tenants/${tenantId}`)
    .get();

  const customerId = tenantDoc.data()?.stripeCustomerId;

  if (!customerId) {
    throw new HttpsError('not-found', 'No Stripe customer found');
  }

  // Create portal session
  const session = await stripe.billingPortal.sessions.create({
    customer: customerId,
    return_url: returnUrl || `${process.env.APP_URL}/billing`,
  });

  return {
    portalUrl: session.url,
  };
});
```

## Frontend Integration

### Install Stripe.js

```bash
cd apps/mobile-app
npm install @stripe/stripe-js
```

### Subscription Service

**File:** `apps/mobile-app/src/app/services/subscription.service.ts`

```typescript
import { Injectable, inject } from '@angular/core';
import { Firestore, doc, docData } from '@angular/fire/firestore';
import { Functions, httpsCallable } from '@angular/fire/functions';
import { loadStripe, Stripe } from '@stripe/stripe-js';
import { Observable, map } from 'rxjs';
import { environment } from '../../environments/environment';

export interface TenantSubscription {
  tenantId: string;
  stripeCustomerId: string;
  stripeSubscriptionId: string | null;
  stripePriceId: string | null;
  plan: 'free' | 'trial' | 'pro' | 'enterprise';
  status: string;
  trialStart: any;
  trialEnd: any;
  currentPeriodStart: any;
  currentPeriodEnd: any;
  limits: {
    maxJobs: number;
    maxTeamMembers: number;
    maxVoiceMinutesPerMonth: number;
    pdfExport: boolean;
  };
  usage: {
    jobsCreated: number;
    activeTeamMembers: number;
    voiceMinutesThisMonth: number;
  };
  cancelAtPeriodEnd: boolean;
  canceledAt: any;
}

@Injectable({ providedIn: 'root' })
export class SubscriptionService {
  private firestore = inject(Firestore);
  private functions = inject(Functions);
  private stripe: Stripe | null = null;

  async initStripe(): Promise<void> {
    if (!this.stripe) {
      this.stripe = await loadStripe(environment.stripe.publishableKey);
    }
  }

  getSubscription(tenantId: string): Observable<TenantSubscription> {
    const subRef = doc(this.firestore, `tenants/${tenantId}/subscription/default`);
    return docData(subRef) as Observable<TenantSubscription>;
  }

  async upgradeToProPlan(tenantId: string): Promise<void> {
    await this.initStripe();

    const createSession = httpsCallable<any, { checkoutUrl: string }>(
      this.functions,
      'createCheckoutSession'
    );

    const result = await createSession({
      tenantId,
      priceId: environment.stripe.proPriceId,
      successUrl: `${window.location.origin}/billing/success`,
      cancelUrl: `${window.location.origin}/billing`,
    });

    // Redirect to Stripe Checkout
    window.location.href = result.data.checkoutUrl;
  }

  async openBillingPortal(tenantId: string): Promise<void> {
    const createPortal = httpsCallable<any, { portalUrl: string }>(
      this.functions,
      'createPortalSession'
    );

    const result = await createPortal({
      tenantId,
      returnUrl: `${window.location.origin}/billing`,
    });

    // Redirect to Stripe Customer Portal
    window.location.href = result.data.portalUrl;
  }

  async checkCanPerformAction(
    tenantId: string,
    action: string
  ): Promise<{ allowed: boolean; reason?: string; message?: string }> {
    const checkStatus = httpsCallable<any, any>(
      this.functions,
      'checkSubscriptionStatus'
    );

    const result = await checkStatus({ tenantId, action });
    return result.data;
  }

  getRemainingTrialDays(subscription: TenantSubscription): number {
    if (subscription.status !== 'trialing' || !subscription.trialEnd) {
      return 0;
    }

    const now = new Date();
    const trialEnd = subscription.trialEnd.toDate();
    const diff = trialEnd.getTime() - now.getTime();

    return Math.ceil(diff / (1000 * 60 * 60 * 24));
  }

  isFeatureAvailable(
    subscription: TenantSubscription,
    feature: string
  ): boolean {
    const activeStatuses = ['active', 'trialing'];
    if (!activeStatuses.includes(subscription.status)) {
      return false;
    }

    switch (feature) {
      case 'voice':
        return subscription.limits.maxVoiceMinutesPerMonth > 0;
      case 'pdf_export':
        return subscription.limits.pdfExport;
      default:
        return true;
    }
  }
}
```

### Environment Configuration

**File:** `apps/mobile-app/src/environments/environment.ts`

```typescript
export const environment = {
  production: false,
  firebase: {
    // ... Firebase config
  },
  stripe: {
    publishableKey: 'pk_test_xxxxxxxxxxxxx',
    proPriceId: 'price_xxxxxxxxxxxxx',
  },
  appUrl: 'http://localhost:4200',
};
```

**File:** `apps/mobile-app/src/environments/environment.prod.ts`

```typescript
export const environment = {
  production: true,
  firebase: {
    // ... Firebase config
  },
  stripe: {
    publishableKey: 'pk_live_xxxxxxxxxxxxx',
    proPriceId: 'price_xxxxxxxxxxxxx',
  },
  appUrl: 'https://findogai.app',
};
```

## Security Best Practices

### 1. Never Expose Secret Keys

```typescript
// ❌ NEVER DO THIS
const stripe = new Stripe('sk_live_xxxxxxxxxxxxx');

// ✅ DO THIS
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);
```

### 2. Always Verify Webhook Signatures

```typescript
// ❌ NEVER DO THIS (accepts any webhook)
const event = JSON.parse(req.body);

// ✅ DO THIS (verifies signature)
const event = stripe.webhooks.constructEvent(
  req.rawBody,
  sig,
  webhookSecret
);
```

### 3. Validate Metadata

```typescript
// Always include and validate tenantId
subscription.metadata = {
  tenantId: tenantId,
  firebaseUid: uid,
};

// In webhook handler, always check:
if (!subscription.metadata.tenantId) {
  logger.error('Missing tenantId in metadata');
  return;
}
```

### 4. Idempotency

```typescript
// Stripe webhooks may be delivered multiple times
// Use Firestore transactions or set with merge
await admin.firestore()
  .doc(`tenants/${tenantId}/subscription/default`)
  .set(data, { merge: true }); // Safe for duplicate events
```

### 5. Rate Limiting

```typescript
// Limit checkout session creation
const rateLimiter = new Map<string, number>();

function checkRateLimit(tenantId: string): boolean {
  const now = Date.now();
  const lastCall = rateLimiter.get(tenantId) || 0;

  if (now - lastCall < 60000) { // 1 minute
    return false;
  }

  rateLimiter.set(tenantId, now);
  return true;
}
```

## Testing

### Local Testing with Stripe CLI

**Install Stripe CLI:**
```bash
# macOS
brew install stripe/stripe-cli/stripe

# Windows
scoop install stripe

# Linux
wget https://github.com/stripe/stripe-cli/releases/download/v1.19.0/stripe_1.19.0_linux_x86_64.tar.gz
```

**Login:**
```bash
stripe login
```

**Forward Webhooks to Local Functions:**
```bash
# Start Firebase emulators
firebase emulators:start

# In another terminal, forward webhooks
stripe listen --forward-to http://localhost:5001/findogai-dev/europe-west1/handleStripeWebhook
```

**Trigger Test Events:**
```bash
# Subscription created
stripe trigger customer.subscription.created

# Payment succeeded
stripe trigger invoice.paid

# Payment failed
stripe trigger invoice.payment_failed

# Trial ending
stripe trigger customer.subscription.trial_will_end
```

### Test Cards

```typescript
// Use these test cards in Stripe Checkout
const TEST_CARDS = {
  success: '4242 4242 4242 4242',
  decline: '4000 0000 0000 0002',
  requiresAuth: '4000 0027 6000 3184', // 3D Secure
  insufficientFunds: '4000 0000 0000 9995',
};
```

### Test Scenarios

```yaml
Scenario 1: Successful Subscription
  1. Create checkout session
  2. Use test card 4242 4242 4242 4242
  3. Verify subscription created in Firestore
  4. Verify status: trialing
  5. Verify trial dates set correctly

Scenario 2: Payment Failure
  1. Create checkout session
  2. Use test card 4000 0000 0000 0002
  3. Verify payment fails
  4. Verify error handling

Scenario 3: Webhook Processing
  1. Trigger customer.subscription.created
  2. Verify Firestore updated
  3. Check audit logs
  4. Verify email sent (if implemented)

Scenario 4: Trial Expiration
  1. Create subscription with trial
  2. Manually set trialEnd to past date
  3. Run processTrialExpirations function
  4. Verify status updated to incomplete_expired
  5. Verify access blocked

Scenario 5: Usage Limits
  1. Create free plan subscription
  2. Create 5 jobs (at limit)
  3. Attempt to create 6th job
  4. Verify blocked with upgrade prompt
```

## Monitoring & Alerts

### Stripe Dashboard

Monitor in Stripe Dashboard:
- Recent payments
- Failed payments
- Subscription churn
- Revenue trends
- Refund requests

### Cloud Functions Logs

```bash
# View webhook logs
firebase functions:log --only handleStripeWebhook

# Filter for errors
firebase functions:log --only handleStripeWebhook | grep ERROR
```

### Alerts

```yaml
Alert: High Payment Failure Rate
  Condition: payment_failure_rate > 10%
  Action: Email founders + check Stripe dashboard

Alert: Webhook Delivery Failing
  Condition: webhook_failure_rate > 5%
  Action: Critical alert + investigate immediately

Alert: Subscription Cancellation Spike
  Condition: cancellations_today > avg * 2
  Action: Email founders + analyze feedback
```

## Common Issues & Troubleshooting

### Issue: Webhook not receiving events

**Solution:**
```bash
# 1. Check webhook endpoint is deployed
curl https://europe-west1-findogai-prod.cloudfunctions.net/handleStripeWebhook

# 2. Verify webhook secret matches
firebase functions:config:get

# 3. Check Stripe Dashboard → Developers → Webhooks
# Look for failed deliveries

# 4. Manually replay webhook
# Stripe Dashboard → Event → "Resend webhook"
```

### Issue: Signature verification fails

**Solution:**
```typescript
// Ensure using raw body, not parsed JSON
const event = stripe.webhooks.constructEvent(
  req.rawBody, // NOT req.body
  sig,
  webhookSecret
);
```

### Issue: Customer ID not found

**Solution:**
```typescript
// Always check if customer exists before creating
const tenantDoc = await admin.firestore()
  .doc(`tenants/${tenantId}`)
  .get();

let customerId = tenantDoc.data()?.stripeCustomerId;

if (!customerId) {
  // Create customer first
  const customer = await stripe.customers.create({...});
  customerId = customer.id;

  // Save to Firestore
  await tenantDoc.ref.update({ stripeCustomerId: customerId });
}
```

### Issue: Duplicate subscriptions

**Solution:**
```typescript
// Check for existing subscription before creating
const existingSub = await admin.firestore()
  .doc(`tenants/${tenantId}/subscription/default`)
  .get();

if (existingSub.exists && existingSub.data()?.stripeSubscriptionId) {
  throw new HttpsError('already-exists', 'Subscription already exists');
}
```

## Production Checklist

```yaml
Before Going Live:
  ✓ Switch to live API keys
  ✓ Update webhook endpoint to production URL
  ✓ Test with real (small amount) payment
  ✓ Verify webhook signature validation
  ✓ Configure Stripe email notifications
  ✓ Set up billing alerts
  ✓ Enable Stripe Radar (fraud detection)
  ✓ Configure tax settings (EU VAT)
  ✓ Add business information to receipts
  ✓ Test cancellation flow
  ✓ Test refund flow
  ✓ Document runbook for common issues
  ✓ Set up monitoring dashboards
```

---

*Next: [Monitoring](./monitoring.md)*
