# Epic 7: Subscription Billing & Monetization

**Goal:** Implement subscription-based monetization with Stripe integration, trial management, usage limits, and billing workflows to enable sustainable revenue generation.

**Priority:** Phase 2 (Post-MVP)

**Dependencies:** Epics 1-6 (MVP must be complete and validated)

---

## Overview

This epic transforms FinDogAI from a free MVP into a sustainable SaaS business with subscription billing. It implements:
- 14-day free trial with automatic expiration
- Stripe payment processing
- Three subscription tiers (Free, Pro, Enterprise)
- Usage limit enforcement
- Self-service billing management

---

## Story 7.1: Subscription Data Model & Initial Setup

**As a** developer,
**I want** subscription and payment data models implemented,
**so that** the system can track billing status and usage limits.

**Acceptance Criteria:**

1. Firestore collections created:
   - `/tenants/{tenantId}/subscription/default` (single document)
   - `/tenants/{tenantId}/payments/{paymentId}` (historical records)

2. TenantSubscription interface includes:
   - `stripeCustomerId`, `stripeSubscriptionId`, `stripePriceId`
   - `plan`: 'free' | 'trial' | 'pro' | 'enterprise'
   - `status`: subscription state (active, trialing, past_due, etc.)
   - `limits`: {maxJobs, maxTeamMembers, maxVoiceMinutesPerMonth, pdfExport}
   - `usage`: {jobsCreated, activeTeamMembers, voiceMinutesThisMonth}
   - `trialStart`, `trialEnd`, `currentPeriodStart`, `currentPeriodEnd`

3. PaymentRecord interface includes:
   - Stripe references (invoiceId, paymentIntentId)
   - Amount, currency, status
   - Invoice PDF URL
   - Period covered, failure reason (if applicable)

4. Tenant model extended with `stripeCustomerId?: string`

5. Subscription plans defined in code (free, trial, pro, enterprise) with:
   - Plan ID, name, price, currency, interval
   - Stripe Price ID mapping
   - Features list
   - Usage limits

6. Migration function created to add subscription to existing tenants:
   - Default: 14-day trial for all existing tenants
   - Status: 'trialing'
   - Trial ends 14 days from migration date

**Technical Notes:**
- TypeScript interfaces in `/packages/shared-types`
- Plan definitions in backend config file
- Migration callable function for one-time execution

---

## Story 7.2: Stripe Account Setup & Integration

**As a** developer,
**I want** Stripe account configured and integrated with Firebase,
**so that** the system can process payments securely.

**Acceptance Criteria:**

1. Stripe account created and activated:
   - Business verified
   - Bank account connected
   - Tax settings configured (EU VAT)

2. Products and prices created in Stripe Dashboard:
   - **FinDogAI Pro**: €29/month, recurring, 14-day trial
   - **FinDogAI Enterprise**: Custom pricing

3. Webhook endpoint configured:
   - URL: `https://europe-west1-findogai-prod.cloudfunctions.net/handleStripeWebhook`
   - Events: subscription.*, invoice.*, payment_intent.*
   - Webhook signing secret saved securely

4. API keys stored securely:
   - Test keys in Firebase Functions config (dev)
   - Live keys in Secret Manager (production)
   - Publishable keys in frontend environment config

5. Stripe SDK installed:
   - Backend: `stripe` npm package (^14.11.0)
   - Frontend: `@stripe/stripe-js`

6. Environment variables set:
   ```
   STRIPE_SECRET_KEY=sk_live_xxxxx
   STRIPE_PUBLISHABLE_KEY=pk_live_xxxxx
   STRIPE_WEBHOOK_SECRET=whsec_xxxxx
   STRIPE_PRO_PRICE_ID=price_xxxxx
   ```

**Testing:**
- Test mode checkout completes successfully
- Webhook events received and processed
- Signature verification works

---

## Story 7.3: Trial Creation on Signup

**As a** new user,
**I want** to automatically start a 14-day free trial when I create an account,
**so that** I can try all Pro features before committing.

**Acceptance Criteria:**

1. On tenant creation (during signup):
   - Stripe customer created with user email
   - `stripeCustomerId` saved to tenant document
   - Subscription document created with:
     - plan: 'trial'
     - status: 'trialing'
     - trialStart: now()
     - trialEnd: now() + 14 days
     - limits: Pro plan limits (unlimited jobs, 1000 voice minutes, etc.)
     - usage: all zeros

2. Welcome email sent with trial information:
   - "Your 14-day Pro trial has started"
   - Trial expiration date
   - Link to billing page

3. Trial banner shown in app header:
   - "Trial expires in X days - Upgrade to keep access"
   - Only visible when status='trialing' and 7 days remain
   - Click → navigates to billing page

4. User has full Pro access during trial:
   - Unlimited jobs
   - Team members
   - Voice commands
   - PDF export

5. Error handling:
   - If Stripe customer creation fails, retry with exponential backoff
   - If fails after 3 attempts, log error and create tenant anyway (manual fix later)
   - Alert developers via monitoring

**Edge Cases:**
- User signs up with same email twice → reuse existing Stripe customer
- Network failure during creation → idempotent retry logic

---

## Story 7.4: Trial Expiration & Access Restriction

**As a** system admin,
**I want** trials to expire automatically after 14 days,
**so that** users are prompted to upgrade.

**Acceptance Criteria:**

1. Scheduled Cloud Function `processTrialExpirations` runs daily at 02:00 UTC:
   - Query subscriptions where status='trialing' AND trialEnd <= now()
   - For each expired trial:
     - Update status to 'incomplete_expired'
     - Send trial expired email
     - Log event to audit trail

2. When status='incomplete_expired':
   - Security rules block all write operations (jobs, costs, members)
   - Read access still allowed (view-only mode)
   - Upgrade modal shown on app load (cannot be dismissed)

3. Trial expired email sent:
   - Subject: "Your FinDogAI trial has ended"
   - Body: "Upgrade to Pro to continue using all features"
   - CTA button: "Upgrade Now" (links to billing page)
   - Reminder: Trial expires 7 days after inactivity

4. Upgrade modal UI:
   - Heading: "Your trial has ended"
   - Body: "Upgrade to Pro for €29/month to continue"
   - Features list (unlimited jobs, team members, voice, PDF)
   - Buttons: "Upgrade to Pro" (primary), "Downgrade to Free" (secondary)
   - Cannot be dismissed (blocking)

5. Grace period: 7 days read-only access after trial expires:
   - Days 1-7: Can view data, cannot create/edit
   - Day 8: Tenant archived (soft delete), data retained for 30 days
   - Email sent on Day 7: "Last chance to upgrade"

6. Monitoring:
   - Alert if scheduled function fails to run
   - Track trial conversion rate (upgraded / total trials)
   - Log all expiration events

**Technical Notes:**
- Use Firestore collectionGroup query for efficiency
- Batch updates (max 500 per execution)
- Idempotent (safe to run multiple times)

---

## Story 7.5: Upgrade to Pro Plan

**As a** trial user,
**I want** to upgrade to Pro plan with credit card,
**so that** I can continue using FinDogAI after my trial ends.

**Acceptance Criteria:**

1. Billing page route: `/billing`
   - Shows current plan (Trial, Free, Pro)
   - Trial users: "X days remaining" countdown
   - Expired users: "Trial ended" message
   - CTA: "Upgrade to Pro" button

2. Click "Upgrade to Pro" → calls `createCheckoutSession` Cloud Function:
   - Input: {tenantId, priceId='price_xxxxx'}
   - Creates Stripe Checkout session with:
     - Customer ID from tenant
     - Pro plan price (€29/month)
     - 14-day trial (if not already used)
     - Success URL: `/billing/success?session_id={CHECKOUT_SESSION_ID}`
     - Cancel URL: `/billing`
   - Returns: {checkoutUrl}

3. Frontend redirects to Stripe Checkout:
   - User enters payment details
   - Stripe handles 3D Secure / SCA if required
   - Test cards work in test mode

4. On successful payment:
   - Stripe sends `customer.subscription.created` webhook
   - `handleStripeWebhook` function processes event:
     - Updates subscription document:
       - status: 'active'
       - plan: 'pro'
       - stripeSubscriptionId: sub_xxxxx
       - stripePriceId: price_xxxxx
       - currentPeriodStart, currentPeriodEnd
       - Maintains current usage counts
     - Creates payment record in `/tenants/{tenantId}/payments/`
   - Frontend redirected to success URL

5. Success page (`/billing/success`):
   - "Welcome to Pro!" message
   - Confirmation: "Your subscription is now active"
   - Receipt: "Invoice sent to your email"
   - CTA: "Start using FinDogAI" → navigate to dashboard

6. User immediately has Pro access:
   - All limits lifted (or set to Pro limits)
   - Voice commands available
   - PDF export enabled
   - Team invites unlocked

7. Invoice email sent by Stripe:
   - Receipt with payment details
   - Invoice PDF attached
   - Next billing date

**Error Handling:**
- Checkout canceled → redirect to `/billing` with message: "Payment canceled"
- Payment failed → Stripe shows error, user can retry
- Webhook delivery fails → Stripe auto-retries (exponential backoff)

**Testing:**
- Test with card 4242 4242 4242 4242 (success)
- Test with card 4000 0000 0000 0002 (decline)
- Test with card 4000 0027 6000 3184 (requires 3D Secure)

---

## Story 7.6: Usage Limit Enforcement

**As a** Free plan user,
**I want** to be blocked from exceeding my plan limits,
**so that** I'm prompted to upgrade when needed.

**Acceptance Criteria:**

1. Job creation limit (Free plan: 5 jobs):
   - Before creating job, call `checkSubscriptionStatus` function
   - If usage.jobsCreated >= limits.maxJobs:
     - Return {allowed: false, reason: 'job_limit_reached', message: "You've reached your limit of 5 jobs. Upgrade to Pro for unlimited jobs."}
     - Frontend shows upgrade modal (blocking)
   - Security rules also enforce limit (server-side validation)

2. Team member limit (Free plan: 1 member):
   - Before inviting member, check limit
   - If usage.activeTeamMembers >= limits.maxTeamMembers:
     - Block with upgrade prompt

3. Voice command limit (Free plan: 0 minutes):
   - Before processing voice command, check limit
   - If limits.maxVoiceMinutesPerMonth == 0:
     - Block with message: "Voice features are only available on Pro plan"

4. PDF export limit (Free plan: disabled):
   - Export PDF button disabled (grayed out)
   - Tooltip: "Upgrade to Pro to export PDFs"
   - Click → shows upgrade modal

5. Usage tracking triggers:
   - `onJobCreated`: Increment `usage.jobsCreated`
   - `onMemberWritten`: Recalculate `usage.activeTeamMembers`
   - `trackVoiceUsage`: Increment `usage.voiceMinutesThisMonth` (in seconds/60)

6. Monthly usage reset (voice minutes):
   - Scheduled function `resetMonthlyUsage` runs on 1st of month
   - Resets `usage.voiceMinutesThisMonth` to 0
   - Sets `usage.lastVoiceUsageReset` to now()

7. Upgrade modal UI:
   - Heading: "Upgrade to unlock this feature"
   - Body: Reason for block (e.g., "Job limit reached")
   - Current usage vs. limit shown
   - Buttons: "Upgrade to Pro" (primary), "Cancel" (secondary)
   - Can be dismissed (user returns to previous screen)

8. Frontend guards:
   - Before job creation form submit → check limit
   - Before team invite → check limit
   - Before voice command → check limit
   - Show proactive warnings at 80% of limit

**Performance:**
- Cache subscription status in frontend for 5 minutes
- Use Firestore listeners for real-time updates
- Security rules provide final enforcement (cannot be bypassed)

---

## Story 7.7: Billing Management Portal

**As a** Pro user,
**I want** to manage my subscription and payment methods,
**so that** I can update billing details and view invoices.

**Acceptance Criteria:**

1. Billing page (`/billing`) shows for Pro users:
   - Current plan: "Pro - €29/month"
   - Billing cycle: "Next payment on [date]"
   - Payment method: "Visa •••• 4242" (last 4 digits)
   - Usage this month:
     - Jobs created: X (unlimited)
     - Team members: X (unlimited)
     - Voice minutes: X / 1000
   - Buttons: "Manage Billing", "Cancel Subscription"

2. "Manage Billing" button → calls `createPortalSession` function:
   - Creates Stripe Customer Portal session
   - Redirects user to Stripe-hosted portal
   - Portal allows:
     - Update payment method
     - View invoice history
     - Download invoice PDFs
     - Update billing email

3. "Cancel Subscription" button:
   - Shows confirmation modal:
     - "Are you sure you want to cancel?"
     - "You'll lose access to Pro features at the end of your billing period ([date])"
     - Buttons: "Keep Subscription", "Cancel Subscription"
   - On confirm → cancels via Stripe API:
     - Sets `cancel_at_period_end: true`
     - Subscription remains active until period end
     - `cancelAtPeriodEnd` flag set in Firestore

4. Canceled subscription behavior:
   - User retains Pro access until current period ends
   - Banner shown: "Subscription canceled - Access until [date]"
   - Option to reactivate: "Reactivate Subscription" button
   - At period end → webhook updates status to 'canceled'

5. Reactivation:
   - "Reactivate Subscription" → removes cancellation
   - Stripe API: `subscription.update({cancel_at_period_end: false})`
   - Updates Firestore: `cancelAtPeriodEnd: false`
   - Banner removed

6. Invoice history:
   - Table with columns: Date, Amount, Status, Invoice
   - Rows: Payment records from `/tenants/{tenantId}/payments/`
   - "Download PDF" link → opens Stripe-hosted invoice
   - Sorted by date (newest first)

7. Owner-only access:
   - Only tenant owner can access billing page
   - Representatives/team members see: "Contact owner to manage billing"

**Security:**
- Billing portal session expires after 1 hour
- Only owner role can access billing functions
- Payment method details never stored in Firestore (Stripe only)

---

## Story 7.8: Payment Failure & Recovery

**As a** Pro user,
**I want** to be notified if my payment fails,
**so that** I can update my payment method and avoid service interruption.

**Acceptance Criteria:**

1. Payment failure webhook (`invoice.payment_failed`):
   - Updates subscription status to 'past_due'
   - Sends payment failed email:
     - Subject: "Payment failed for your FinDogAI subscription"
     - Body: "We couldn't process your payment. Update your payment method to avoid service interruption."
     - CTA: "Update Payment Method" → Stripe Customer Portal

2. Grace period: 3 days after payment failure:
   - Days 1-3: Full Pro access continues
   - Banner shown: "Payment failed - Update payment method"
   - Stripe automatically retries payment (Smart Retries)

3. Retry schedule:
   - Day 1: Immediate retry
   - Day 3: Second retry
   - Day 5: Third retry
   - Day 7: Final retry → if fails, subscription becomes 'unpaid'

4. Status 'unpaid' (after all retries fail):
   - Access reverted to Free plan limits
   - Modal shown on login: "Subscription unpaid - Update payment to restore access"
   - Cannot dismiss until payment updated

5. Payment recovery:
   - User updates payment method in Stripe Portal
   - Stripe retries payment immediately
   - On success → webhook `invoice.paid`:
     - Status updated to 'active'
     - Access restored
     - Recovery email sent: "Payment successful - Access restored"

6. Dunning emails:
   - Day 1: Payment failed (immediate notification)
   - Day 3: Reminder - "Payment retry scheduled"
   - Day 7: Final notice - "Last chance to update payment"

7. Monitoring:
   - Alert on payment failure spike (>10% of subscriptions)
   - Track payment recovery rate
   - Log all failure reasons (card declined, insufficient funds, etc.)

**Edge Cases:**
- Card expired → Stripe sends advance notice (60 days before)
- 3D Secure required → Stripe sends email with authentication link
- Bank requires action → Webhook `invoice.payment_action_required`

---

## Story 7.9: Free Plan Downgrade

**As a** user,
**I want** to downgrade to Free plan,
**so that** I can continue using FinDogAI with limited features at no cost.

**Acceptance Criteria:**

1. Billing page has "Downgrade to Free" button (for Pro users):
   - Shows confirmation modal:
     - "Downgrade to Free plan?"
     - Limits explained: "You'll be limited to 5 jobs, 1 team member, no voice commands, no PDF export"
     - Warning: "Existing data will be retained but read-only if limits exceeded"
     - Buttons: "Keep Pro", "Downgrade to Free"

2. On confirm → cancels Stripe subscription:
   - Calls Stripe API: `subscription.del()`
   - Stripe sends `customer.subscription.deleted` webhook
   - Webhook handler:
     - Updates status to 'canceled'
     - plan: 'free'
     - Clears stripeSubscriptionId, stripePriceId
     - Sets Free plan limits
     - Retains current usage counts

3. Downgrade effective immediately:
   - User reverted to Free limits
   - If usage exceeds limits:
     - Jobs: Can view all, can only edit first 5 (by creation date)
     - Team members: Can view all, can only add if 1 remaining
     - Voice: Disabled
     - PDF: Disabled

4. Data handling:
   - All jobs retained (no data loss)
   - Team members remain active (if >1, cannot add more)
   - Usage counts maintained
   - User can upgrade again anytime

5. Email sent:
   - Subject: "You've downgraded to Free plan"
   - Body: "Your subscription has been canceled. You can upgrade anytime."
   - Link to pricing page

6. Upgrade path:
   - "Upgrade to Pro" button always visible on billing page
   - Can re-subscribe at any time (no trial if previously trialed)

**Business Logic:**
- Downgrade → immediate cancellation (no period-end grace)
- Re-upgrade → new subscription starts immediately
- No refunds for unused time (Stripe default behavior)

---

## Story 7.10: Admin Subscription Dashboard

**As a** founder/admin,
**I want** to view subscription metrics and user billing status,
**so that** I can monitor revenue and identify issues.

**Acceptance Criteria:**

1. Admin dashboard route: `/admin/subscriptions` (owner-only):
   - Requires Firebase custom claim `admin: true`
   - Lists all tenants with subscription info

2. Subscription metrics displayed:
   - **MRR (Monthly Recurring Revenue):** €X,XXX
   - **Active subscriptions:** X Pro + X Enterprise
   - **Trials:** X active trials
   - **Churn rate:** X% (canceled this month / total at start)
   - **Trial conversion rate:** X% (trials → paid / total trials)
   - **Failed payments:** X tenants in 'past_due'

3. Tenant table with columns:
   - Tenant ID (link to Firestore console)
   - Owner email
   - Plan (Free/Trial/Pro/Enterprise)
   - Status (active/trialing/past_due/canceled)
   - Trial ends (if applicable)
   - Next billing date
   - Usage: Jobs, Members, Voice minutes
   - Actions: "View in Stripe", "Send email"

4. Filters:
   - Plan: All / Free / Trial / Pro / Enterprise
   - Status: All / Active / Past Due / Canceled
   - Trial expiring: <7 days, <3 days, <1 day

5. Actions:
   - "View in Stripe" → Opens Stripe Dashboard customer page
   - "Send email" → Manual email trigger (support contact)
   - "Extend trial" → Callable function to add 7 days (manual intervention)

6. Export:
   - "Export to CSV" button → Downloads tenant list with metrics
   - Includes: tenantId, email, plan, status, MRR, usage

7. Real-time updates:
   - Dashboard updates when subscription changes (Firestore listener)
   - No need to refresh page

**Security:**
- Admin-only route (custom claim required)
- Cloud Functions have admin SDK access for cross-tenant queries
- Audit log all admin actions

**Technical Notes:**
- Use Firestore collectionGroup queries for aggregation
- Cache metrics (update hourly, not real-time)
- Consider BigQuery export for advanced analytics

---

## Story 7.11: Subscription Email Notifications

**As a** user,
**I want** to receive timely emails about my subscription,
**so that** I'm always informed about billing events.

**Acceptance Criteria:**

1. Email service integrated:
   - Use SendGrid, Mailgun, or Firebase Extensions (Email Trigger)
   - Templates created for each email type
   - From: noreply@findogai.app, Reply-to: support@findogai.app

2. Email types implemented:

   **Trial Started:**
   - Trigger: Tenant creation
   - Subject: "Welcome to FinDogAI - Your 14-day trial has started"
   - Body: Trial end date, link to getting started guide
   - CTA: "Get Started"

   **Trial Ending (7 days before):**
   - Trigger: Daily cron, 7 days before trialEnd
   - Subject: "Your FinDogAI trial expires in 7 days"
   - Body: Upgrade to continue, pricing
   - CTA: "Upgrade to Pro"

   **Trial Ending (3 days before):**
   - Trigger: Daily cron, 3 days before trialEnd
   - Subject: "Only 3 days left in your FinDogAI trial"
   - Body: Urgent tone, upgrade reminder
   - CTA: "Upgrade Now"

   **Trial Ending (1 day before):**
   - Trigger: Daily cron, 1 day before trialEnd
   - Subject: "Last day of your FinDogAI trial"
   - Body: Final reminder, what happens after expiration
   - CTA: "Upgrade Now"

   **Trial Expired:**
   - Trigger: processTrialExpirations function
   - Subject: "Your FinDogAI trial has ended"
   - Body: Upgrade to continue, downgrade to Free option
   - CTA: "Upgrade to Pro" / "Downgrade to Free"

   **Payment Succeeded:**
   - Trigger: Stripe webhook `invoice.paid`
   - Subject: "Payment received - Thank you!"
   - Body: Receipt, next billing date
   - CTA: "View Invoice"

   **Payment Failed:**
   - Trigger: Stripe webhook `invoice.payment_failed`
   - Subject: "Payment failed for your FinDogAI subscription"
   - Body: Update payment method, consequences of non-payment
   - CTA: "Update Payment Method"

   **Subscription Canceled:**
   - Trigger: Stripe webhook `customer.subscription.deleted`
   - Subject: "Your FinDogAI subscription has been canceled"
   - Body: Access until period end, re-subscribe option
   - CTA: "Reactivate Subscription"

   **Usage Limit Warning (80%):**
   - Trigger: Usage tracking triggers (when 80% of limit reached)
   - Subject: "You're approaching your plan limit"
   - Body: Current usage, limit, upgrade recommendation
   - CTA: "Upgrade to Pro"

3. Email preferences:
   - User can opt out of marketing emails (not billing emails)
   - Billing emails always sent (required for payment)
   - Link in footer: "Unsubscribe from marketing emails"

4. Email tracking:
   - Log all emails sent to Firestore: `/tenants/{tenantId}/emails/{emailId}`
   - Track: sent date, type, status (sent/failed), opened (if tracking enabled)

5. Testing:
   - Use Mailtrap or similar for dev/staging email testing
   - Verify all email templates render correctly
   - Check spam score (avoid spam folder)

**Technical Notes:**
- Use email templates with variables ({{name}}, {{trialEndDate}}, etc.)
- Queue emails (don't send synchronously in webhooks)
- Retry failed sends (max 3 attempts)

---

## Technical Implementation Checklist

### Backend (Cloud Functions)

- [ ] Install Stripe SDK: `npm install stripe --save`
- [ ] Implement `createCheckoutSession` callable function
- [ ] Implement `checkSubscriptionStatus` callable function
- [ ] Implement `createPortalSession` callable function
- [ ] Implement `trackVoiceUsage` callable function
- [ ] Implement `handleStripeWebhook` HTTP function
- [ ] Implement `onJobCreated` trigger (usage tracking)
- [ ] Implement `onMemberWritten` trigger (usage tracking)
- [ ] Implement `processTrialExpirations` scheduled function
- [ ] Implement `resetMonthlyUsage` scheduled function
- [ ] Implement email notification functions
- [ ] Add subscription checks to existing functions (job creation, etc.)
- [ ] Unit tests for all subscription functions (>80% coverage)

### Frontend (Angular/Ionic)

- [ ] Install Stripe.js: `npm install @stripe/stripe-js`
- [ ] Create `SubscriptionService` with methods:
  - [ ] `getSubscription(tenantId): Observable<TenantSubscription>`
  - [ ] `upgradeToProPlan(tenantId): Promise<void>`
  - [ ] `openBillingPortal(tenantId): Promise<void>`
  - [ ] `checkCanPerformAction(tenantId, action): Promise<boolean>`
  - [ ] `getRemainingTrialDays(subscription): number`
  - [ ] `isFeatureAvailable(subscription, feature): boolean`
- [ ] Create billing page component (`/billing`)
- [ ] Create upgrade modal component
- [ ] Create trial banner component
- [ ] Add subscription checks to:
  - [ ] Job creation form
  - [ ] Team member invite
  - [ ] Voice command pipeline
  - [ ] PDF export button
- [ ] Add usage indicators to UI (jobs created, voice minutes used)
- [ ] Add "Upgrade" prompts throughout app
- [ ] Unit tests for SubscriptionService
- [ ] E2E tests for upgrade flow

### Database (Firestore)

- [ ] Create Firestore indexes:
  - [ ] `subscription.status + subscription.trialEnd` (for cron queries)
  - [ ] `payments.tenantId + payments.createdAt` (for invoice history)
- [ ] Deploy updated security rules with subscription checks
- [ ] Test security rules with Firebase Emulator
- [ ] Run migration function to add subscription to existing tenants

### Stripe Configuration

- [ ] Create Stripe account and verify business
- [ ] Create products and prices in Stripe Dashboard
- [ ] Configure webhook endpoint (dev + prod)
- [ ] Test webhook delivery with Stripe CLI
- [ ] Configure Customer Portal settings
- [ ] Set up billing email notifications in Stripe
- [ ] Enable fraud detection (Stripe Radar)
- [ ] Configure tax settings (EU VAT)

### Testing

- [ ] Test trial creation on signup
- [ ] Test trial expiration (manually adjust trialEnd date)
- [ ] Test upgrade flow with test card 4242 4242 4242 4242
- [ ] Test payment failure with test card 4000 0000 0000 0002
- [ ] Test 3D Secure with test card 4000 0027 6000 3184
- [ ] Test webhook processing (subscription.*, invoice.*)
- [ ] Test usage limit enforcement (jobs, members, voice)
- [ ] Test monthly usage reset
- [ ] Test downgrade to Free
- [ ] Test billing portal (update payment method, view invoices)
- [ ] Test cancellation and reactivation
- [ ] Test all email notifications
- [ ] Load test: 100 concurrent checkout sessions

### Monitoring & Alerts

- [ ] Set up Stripe webhook monitoring (alert on failures)
- [ ] Set up payment failure rate alert (>10% triggers notification)
- [ ] Set up trial conversion tracking
- [ ] Set up MRR dashboard (Stripe + custom analytics)
- [ ] Set up churn rate tracking
- [ ] Log all subscription events to audit trail

### Documentation

- [ ] Update API documentation with subscription functions
- [ ] Create Stripe integration runbook for developers
- [ ] Document trial expiration process
- [ ] Document payment failure recovery process
- [ ] Create user-facing billing FAQ
- [ ] Document admin dashboard usage

---

## Success Metrics

### Business Metrics
- **Trial Conversion Rate:** Target ≥20% (trials → paid subscriptions)
- **Monthly Churn Rate:** Target <5%
- **MRR Growth:** Target +10% month-over-month
- **Payment Success Rate:** Target >95%
- **Average Revenue Per User (ARPU):** Target €29

### Technical Metrics
- **Webhook Delivery Success Rate:** Target >99%
- **Checkout Session Success Rate:** Target >90%
- **API Response Time:** p95 <500ms
- **Trial Expiration Processing Time:** <5 minutes (for all expired trials)

### User Experience Metrics
- **Time to Upgrade:** Target <2 minutes (from click to active)
- **Billing Portal Usage:** Target >50% of Pro users access portal
- **Support Tickets (billing):** Target <5% of users require support

---

## Rollout Plan

### Phase 1: Internal Testing (Week 1-2)
- Deploy to dev environment
- Team testing with test cards
- Verify all webhooks work
- Test trial expiration manually

### Phase 2: Beta Testing (Week 3-4)
- Deploy to staging with real Stripe test mode
- Invite 10-20 beta users
- Monitor closely for issues
- Collect feedback on upgrade flow

### Phase 3: Soft Launch (Week 5)
- Deploy to production with live Stripe
- Enable for new signups only (existing users stay free)
- Monitor payment success rate
- A/B test pricing (€29 vs €39)

### Phase 4: Full Launch (Week 6)
- Announce subscription plans publicly
- Email existing users about trial offer
- Marketing push
- Monitor metrics closely

---

## Risks & Mitigation

| Risk | Impact | Likelihood | Mitigation |
|------|--------|-----------|------------|
| **High payment failure rate** | Lost revenue | Medium | Implement Smart Retries, dunning emails, grace period |
| **Webhook delivery failures** | Subscription status out of sync | Low | Idempotent handlers, monitoring, manual reconciliation script |
| **Users abuse trial (multiple accounts)** | Revenue loss | Medium | Email verification, fraud detection, limit 1 trial per email |
| **Low trial conversion rate** | Revenue below target | High | Optimize onboarding, in-app upgrade prompts, feature gating |
| **Stripe account suspended** | Complete service outage | Very Low | Follow Stripe ToS, respond to disputes quickly, diversify with backup provider |
| **EU VAT compliance issues** | Legal/financial penalties | Low | Use Stripe Tax, consult tax advisor, proper invoicing |

---

## Dependencies & Blockers

**Must Complete Before Starting:**
- Epic 6 (MVP) fully deployed and stable
- At least 50 active users for trial testing
- Legal review of subscription terms and privacy policy

**External Dependencies:**
- Stripe account approval (1-2 business days)
- Bank account verification for payouts (3-5 business days)
- Email service setup (SendGrid/Mailgun approval)

---

## Future Enhancements (Post-Epic 7)

- **Annual Billing:** 20% discount for annual subscriptions
- **Team Plans:** Per-seat pricing for larger teams
- **Usage-Based Pricing:** Pay per voice minute (alternative to flat fee)
- **Referral Program:** Give 1 month free for each referral
- **Custom Enterprise Plans:** Negotiated pricing for large customers
- **Stripe Billing Portal Customization:** Branded portal with custom CSS
- **Subscription Analytics Dashboard:** Advanced metrics for founders

---

*For detailed technical implementation, see [Subscription Billing Architecture](../architecture/subscription-billing.md) and [Stripe Integration Guide](../architecture/stripe-integration.md).*
