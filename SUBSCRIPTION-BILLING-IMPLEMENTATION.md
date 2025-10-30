# Subscription Billing Implementation Guide

This document provides a step-by-step implementation guide and boilerplate code for the subscription billing system.

## Table of Contents

1. [Cloud Functions Boilerplate](#cloud-functions-boilerplate)
2. [Frontend Service Boilerplate](#frontend-service-boilerplate)
3. [Shared Types](#shared-types)
4. [Security Rules](#security-rules)
5. [Implementation Steps](#implementation-steps)

---

## Cloud Functions Boilerplate

### Directory Structure

```
packages/functions/src/
├── stripe/
│   ├── createCheckoutSession.ts
│   ├── checkSubscriptionStatus.ts
│   ├── createPortalSession.ts
│   ├── webhookHandler.ts
│   └── helpers.ts
├── subscription/
│   ├── trackUsage.ts
│   ├── processTrialExpirations.ts
│   └── resetMonthlyUsage.ts
└── index.ts  (export all functions)
```

### 1. Stripe Helper Functions

**File:** `packages/functions/src/stripe/helpers.ts`

```typescript
import Stripe from 'stripe';
import * as admin from 'firebase-admin';

// Initialize Stripe (use in other files)
export function getStripeClient(): Stripe {
  const secretKey = process.env.STRIPE_SECRET_KEY;
  if (!secretKey) {
    throw new Error('STRIPE_SECRET_KEY environment variable is not set');
  }
  return new Stripe(secretKey, {
    apiVersion: '2024-11-20.acacia',
  });
}

// Get or create Stripe customer for tenant
export async function getOrCreateStripeCustomer(
  tenantId: string,
  email: string
): Promise<string> {
  const stripe = getStripeClient();

  // Check if customer already exists
  const tenantDoc = await admin.firestore()
    .doc(`tenants/${tenantId}`)
    .get();

  let customerId = tenantDoc.data()?.stripeCustomerId;

  if (customerId) {
    return customerId;
  }

  // Create new customer
  const customer = await stripe.customers.create({
    email,
    metadata: {
      tenantId,
    },
  });

  // Save customer ID to tenant
  await tenantDoc.ref.update({
    stripeCustomerId: customer.id,
    updatedAt: admin.firestore.FieldValue.serverTimestamp(),
  });

  return customer.id;
}

// Get subscription for tenant
export async function getTenantSubscription(
  tenantId: string
): Promise<FirebaseFirestore.DocumentSnapshot> {
  return admin.firestore()
    .doc(`tenants/${tenantId}/subscription/default`)
    .get();
}
```

### 2. Create Checkout Session

**File:** `packages/functions/src/stripe/createCheckoutSession.ts`

```typescript
import { onCall, HttpsError } from 'firebase-functions/v2/https';
import * as logger from 'firebase-functions/logger';
import { getStripeClient, getOrCreateStripeCustomer } from './helpers';

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
>(
  {
    region: 'europe-west1',
    memory: '256MiB',
  },
  async (request) => {
    // 1. Verify authentication
    if (!request.auth) {
      throw new HttpsError('unauthenticated', 'User must be authenticated');
    }

    const { tenantId, priceId, successUrl, cancelUrl } = request.data;

    // 2. Verify tenant ownership
    const memberDoc = await admin.firestore()
      .doc(`tenants/${tenantId}/members/${request.auth.uid}`)
      .get();

    if (!memberDoc.exists || memberDoc.data()?.role !== 'owner') {
      throw new HttpsError(
        'permission-denied',
        'Only tenant owners can manage subscriptions'
      );
    }

    try {
      const stripe = getStripeClient();

      // 3. Get or create Stripe customer
      const customerId = await getOrCreateStripeCustomer(
        tenantId,
        request.auth.token.email!
      );

      // 4. Create checkout session
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
        allow_promotion_codes: true,
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
  }
);
```

### 3. Check Subscription Status

**File:** `packages/functions/src/stripe/checkSubscriptionStatus.ts`

```typescript
import { onCall, HttpsError } from 'firebase-functions/v2/https';
import * as admin from 'firebase-admin';
import { getTenantSubscription } from './helpers';

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
>(
  {
    region: 'europe-west1',
    memory: '256MiB',
  },
  async (request) => {
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
    const subDoc = await getTenantSubscription(tenantId);

    if (!subDoc.exists) {
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
  }
);

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
      message: `You've reached your limit of ${limit} team members. Upgrade for unlimited.`,
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
      message: 'Voice features are only available on Pro plan.',
      redirectTo: '/billing/upgrade',
    };
  }

  if (usage >= limit) {
    return {
      allowed: false,
      reason: 'voice_limit_reached',
      message: `You've used all ${limit} voice minutes this month. Upgrade for more.`,
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
      message: 'PDF export is only available on Pro plan.',
      redirectTo: '/billing/upgrade',
    };
  }

  return { allowed: true };
}
```

### 4. Webhook Handler (Complete)

**File:** `packages/functions/src/stripe/webhookHandler.ts`

```typescript
import { onRequest } from 'firebase-functions/v2/https';
import Stripe from 'stripe';
import * as admin from 'firebase-admin';
import * as logger from 'firebase-functions/logger';
import { getStripeClient } from './helpers';

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
      const stripe = getStripeClient();
      const webhookSecret = process.env.STRIPE_WEBHOOK_SECRET!;

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

async function handleSubscriptionChange(subscription: Stripe.Subscription): Promise<void> {
  const tenantId = subscription.metadata.tenantId;

  if (!tenantId) {
    logger.warn('Missing tenantId in subscription metadata');
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

  if (subscription.status === 'trialing' && subscription.trial_start && subscription.trial_end) {
    subscriptionData.trialStart = admin.firestore.Timestamp.fromMillis(subscription.trial_start * 1000);
    subscriptionData.trialEnd = admin.firestore.Timestamp.fromMillis(subscription.trial_end * 1000);
  }

  if (subscription.canceled_at) {
    subscriptionData.canceledAt = admin.firestore.Timestamp.fromMillis(subscription.canceled_at * 1000);
  }

  await admin.firestore()
    .doc(`tenants/${tenantId}/subscription/default`)
    .set(subscriptionData, { merge: true });

  logger.info(`Updated subscription for tenant ${tenantId}`, { status: subscription.status });
}

async function handleSubscriptionDeleted(subscription: Stripe.Subscription): Promise<void> {
  const tenantId = subscription.metadata.tenantId;
  if (!tenantId) return;

  await admin.firestore()
    .doc(`tenants/${tenantId}/subscription/default`)
    .update({
      status: 'canceled',
      canceledAt: admin.firestore.FieldValue.serverTimestamp(),
      updatedAt: admin.firestore.FieldValue.serverTimestamp(),
    });

  logger.info(`Subscription canceled for tenant ${tenantId}`);
}

async function handleTrialWillEnd(subscription: Stripe.Subscription): Promise<void> {
  const tenantId = subscription.metadata.tenantId;
  if (!tenantId) return;

  logger.info(`Trial ending soon for tenant ${tenantId}`);
  // TODO: Send trial ending email
}

async function handleInvoicePaid(invoice: Stripe.Invoice): Promise<void> {
  const tenantId = invoice.subscription_details?.metadata?.tenantId;
  if (!tenantId) return;

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
}

async function handleInvoicePaymentFailed(invoice: Stripe.Invoice): Promise<void> {
  const tenantId = invoice.subscription_details?.metadata?.tenantId;
  if (!tenantId) return;

  await admin.firestore()
    .doc(`tenants/${tenantId}/subscription/default`)
    .update({
      status: 'past_due',
      updatedAt: admin.firestore.FieldValue.serverTimestamp(),
    });

  logger.warn(`Payment failed for tenant ${tenantId}`);
  // TODO: Send payment failed email
}
```

### 5. Usage Tracking Triggers

**File:** `packages/functions/src/subscription/trackUsage.ts`

```typescript
import { onDocumentCreated, onDocumentWritten } from 'firebase-functions/v2/firestore';
import * as admin from 'firebase-admin';
import * as logger from 'firebase-functions/logger';

// Track job creation
export const onJobCreatedTrackUsage = onDocumentCreated(
  'tenants/{tenantId}/jobs/{jobId}',
  async (event) => {
    const tenantId = event.params.tenantId;

    try {
      await admin.firestore()
        .doc(`tenants/${tenantId}/subscription/default`)
        .update({
          'usage.jobsCreated': admin.firestore.FieldValue.increment(1),
          updatedAt: admin.firestore.FieldValue.serverTimestamp(),
        });

      logger.info(`Incremented job count for tenant ${tenantId}`);
    } catch (error) {
      logger.error(`Failed to track job creation for tenant ${tenantId}`, error);
    }
  }
);

// Track team member changes
export const onMemberWrittenTrackUsage = onDocumentWritten(
  'tenants/{tenantId}/members/{memberId}',
  async (event) => {
    const tenantId = event.params.tenantId;

    try {
      // Count active members
      const activeMembers = await admin.firestore()
        .collection(`tenants/${tenantId}/members`)
        .where('status', '==', 'active')
        .where('deletedAt', '==', null)
        .count()
        .get();

      await admin.firestore()
        .doc(`tenants/${tenantId}/subscription/default`)
        .update({
          'usage.activeTeamMembers': activeMembers.data().count,
          updatedAt: admin.firestore.FieldValue.serverTimestamp(),
        });

      logger.info(`Updated team member count for tenant ${tenantId}: ${activeMembers.data().count}`);
    } catch (error) {
      logger.error(`Failed to track team member change for tenant ${tenantId}`, error);
    }
  }
);

// Track voice usage
export const trackVoiceUsage = onCall(
  {
    region: 'europe-west1',
    memory: '128MiB',
  },
  async (request) => {
    if (!request.auth) {
      throw new HttpsError('unauthenticated', 'User must be authenticated');
    }

    const { tenantId, durationSeconds } = request.data;
    const minutes = Math.ceil(durationSeconds / 60);

    try {
      const subRef = admin.firestore()
        .doc(`tenants/${tenantId}/subscription/default`);

      await subRef.update({
        'usage.voiceMinutesThisMonth': admin.firestore.FieldValue.increment(minutes),
        updatedAt: admin.firestore.FieldValue.serverTimestamp(),
      });

      // Check if limit exceeded
      const sub = (await subRef.get()).data();
      if (sub && sub.limits.maxVoiceMinutesPerMonth !== -1 &&
          sub.usage.voiceMinutesThisMonth >= sub.limits.maxVoiceMinutesPerMonth) {
        return {
          allowed: false,
          remainingMinutes: 0,
          message: 'Voice minute limit reached for this month.',
        };
      }

      return {
        allowed: true,
        remainingMinutes: sub ? sub.limits.maxVoiceMinutesPerMonth - sub.usage.voiceMinutesThisMonth : 0,
      };
    } catch (error) {
      logger.error(`Failed to track voice usage for tenant ${tenantId}`, error);
      throw new HttpsError('internal', 'Failed to track voice usage');
    }
  }
);
```

### 6. Scheduled Functions

**File:** `packages/functions/src/subscription/processTrialExpirations.ts`

```typescript
import { onSchedule } from 'firebase-functions/v2/scheduler';
import * as admin from 'firebase-admin';
import * as logger from 'firebase-functions/logger';

export const processTrialExpirations = onSchedule(
  {
    schedule: 'every day 02:00',
    timeZone: 'UTC',
    region: 'europe-west1',
    memory: '256MiB',
  },
  async () => {
    const now = admin.firestore.Timestamp.now();

    try {
      const expiredTrials = await admin.firestore()
        .collectionGroup('subscription')
        .where('status', '==', 'trialing')
        .where('trialEnd', '<=', now)
        .get();

      logger.info(`Found ${expiredTrials.size} expired trials`);

      for (const doc of expiredTrials.docs) {
        const sub = doc.data();

        await doc.ref.update({
          status: 'incomplete_expired',
          updatedAt: admin.firestore.FieldValue.serverTimestamp(),
        });

        // TODO: Send trial expired email
        logger.info(`Trial expired for tenant ${sub.tenantId}`);
      }

      logger.info(`Processed ${expiredTrials.size} trial expirations`);
    } catch (error) {
      logger.error('Failed to process trial expirations', error);
      throw error;
    }
  }
);
```

**File:** `packages/functions/src/subscription/resetMonthlyUsage.ts`

```typescript
import { onSchedule } from 'firebase-functions/v2/scheduler';
import * as admin from 'firebase-admin';
import * as logger from 'firebase-functions/logger';

export const resetMonthlyUsage = onSchedule(
  {
    schedule: '0 0 1 * *', // Midnight on 1st of each month
    timeZone: 'UTC',
    region: 'europe-west1',
    memory: '256MiB',
  },
  async () => {
    try {
      const subscriptions = await admin.firestore()
        .collectionGroup('subscription')
        .where('status', 'in', ['active', 'trialing'])
        .get();

      logger.info(`Resetting usage for ${subscriptions.size} subscriptions`);

      const batch = admin.firestore().batch();

      for (const doc of subscriptions.docs) {
        batch.update(doc.ref, {
          'usage.voiceMinutesThisMonth': 0,
          'usage.lastVoiceUsageReset': admin.firestore.FieldValue.serverTimestamp(),
          updatedAt: admin.firestore.FieldValue.serverTimestamp(),
        });
      }

      await batch.commit();
      logger.info(`Reset monthly usage for ${subscriptions.size} subscriptions`);
    } catch (error) {
      logger.error('Failed to reset monthly usage', error);
      throw error;
    }
  }
);
```

### 7. Export All Functions

**File:** `packages/functions/src/index.ts`

```typescript
// Stripe functions
export { createCheckoutSession } from './stripe/createCheckoutSession';
export { checkSubscriptionStatus } from './stripe/checkSubscriptionStatus';
export { createPortalSession } from './stripe/createPortalSession';
export { handleStripeWebhook } from './stripe/webhookHandler';

// Subscription tracking
export {
  onJobCreatedTrackUsage,
  onMemberWrittenTrackUsage,
  trackVoiceUsage
} from './subscription/trackUsage';

// Scheduled functions
export { processTrialExpirations } from './subscription/processTrialExpirations';
export { resetMonthlyUsage } from './subscription/resetMonthlyUsage';

// Existing functions...
```

---

## Implementation Steps

### Step 1: Environment Setup

```bash
# 1. Install Stripe SDK
cd packages/functions
npm install stripe --save
npm install @types/stripe --save-dev

# 2. Set Firebase Functions config (development)
firebase functions:config:set \
  stripe.secret_key="sk_test_xxxxx" \
  stripe.webhook_secret="whsec_xxxxx" \
  stripe.pro_price_id="price_xxxxx" \
  --project findogai-dev

# 3. Deploy functions
firebase deploy --only functions --project findogai-dev

# 4. Configure Stripe webhook endpoint
# URL: https://europe-west1-findogai-dev.cloudfunctions.net/handleStripeWebhook
```

### Step 2: Test with Stripe CLI

```bash
# Forward webhooks to local emulator
stripe listen --forward-to http://localhost:5001/findogai-dev/europe-west1/handleStripeWebhook

# Trigger test events
stripe trigger customer.subscription.created
stripe trigger invoice.paid
```

### Step 3: Frontend Integration

See [stripe-integration.md](./docs/architecture/stripe-integration.md) for complete frontend implementation.

---

## Summary

All subscription billing implementation files have been created:

✅ **Architecture Documentation:**
- `docs/architecture/subscription-billing.md` - Complete billing architecture
- `docs/architecture/stripe-integration.md` - Stripe integration guide

✅ **Updated Architecture Documents:**
- `docs/architecture/data-models.md` - Added subscription models
- `docs/architecture/api-specification.md` - Added Stripe functions
- `docs/architecture/security.md` - Added subscription checks

✅ **Product Requirements:**
- `docs/prd/epic-7-subscription-billing.md` - Complete PRD with 11 user stories

✅ **Implementation Guide:**
- This document with Cloud Functions boilerplate code

**Next Steps:**
1. Review all documentation for completeness
2. Set up Stripe account
3. Implement Cloud Functions using boilerplate
4. Create frontend components
5. Test end-to-end with Stripe test mode
6. Deploy to production

For questions or issues, refer to the detailed architecture documents.
