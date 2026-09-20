🛒 AI-Powered Cart Recovery & Payment Recovery Automation

A production-ready n8n automation architecture designed to solve a major D2C and retail problem:

High cart abandonment across UPI, online payments, and COD checkout flows.

The system automatically detects abandoned Shopify checkouts, waits before contacting the customer, verifies whether the purchase happened, checks messaging eligibility, sends a personalized WhatsApp recovery message, and follows up with a real Shopify discount when appropriate.

The architecture is built with reliability, idempotency, consent checks, payment synchronization, and failure handling rather than being a simple reminder workflow.

🎯 Business Pain Point

D2C and retail brands lose potential revenue when customers:

Add products to cart but don't complete checkout
Abandon UPI / online payment
Choose COD but don't finish checkout
Leave the store after showing purchase intent

A manual follow-up process is slow and difficult to scale.

This automation turns abandoned checkout events into an automated recovery journey.

🚀 Solution

The customer journey is:
Shopify Abandoned Checkout
          ↓
Webhook Verification
          ↓
Validation + Deduplication
          ↓
Wait 15 Minutes
          ↓
Check Shopify Order Status
          ↓
Already Purchased?
       ↙       ↘
     YES        NO
      ↓          ↓
   Recovered   Consent Check
                   ↓
             Payment Method
               ↙       ↘
           COD          Online
            ↓              ↓
      COD Message    Razorpay Link
               ↘      ↙
             WhatsApp
                 ↓
             Wait 4 Hours
                 ↓
         Re-check Purchase
             ↙        ↘
          YES          NO
           ↓            ↓
       Recovered     Coupon Check
                         ↓
                 Create/Reuse 5%
                    Discount
                         ↓
                    WhatsApp

The first recovery workflow intentionally contains the 15-minute and 4-hour waits, while the separate Razorpay workflow synchronizes payment-link payments back into the Shopify order lifecycle.       

🧩 Workflow 1 — Cart Recovery
1. Intake, Verification & Deduplication

The process begins with a Shopify abandoned-checkout webhook.

Cart Abandoned Webhook
        ↓
Verify Shopify Signature
        ↓
Log Webhook Event
        ↓
Signature Valid?
        ↓
Normalize Cart + Customer Data
        ↓
Validate Required Fields
        ↓
Check Cart Idempotency

Shopify's raw webhook body is used for HMAC SHA-256 verification, instead of re-serializing the parsed JSON.

The workflow also uses an idempotency key based on the checkout token so duplicate webhook events can be ignored safely.

⏱️ 2. Wait 15 Minutes & Re-check

The automation does not immediately send a recovery message.

It waits 15 minutes, then checks the live Shopify checkout again.

Wait 15 Minutes
       ↓
Check Order Status
       ↓
Already Purchased?

This prevents the automation from messaging customers who completed their purchase after the original abandoned-checkout event.

The source explicitly uses the live Shopify checkout rather than relying only on the original webhook payload.

🔐 3. WhatsApp Eligibility

Before contacting the customer, the workflow verifies messaging eligibility.

Phone exists
     AND
WhatsApp opt-in = TRUE
     AND
WhatsApp opt-out = FALSE

Customers without valid consent are suppressed instead of being contacted simply because a phone number exists.

Consent note: the Shopify checkout field buyer_accepts_marketing is treated as a marketing-consent signal. A business using dedicated WhatsApp consent should maintain that consent separately rather than assuming Shopify marketing opt-in automatically equals WhatsApp opt-in.

💳 4. Payment Method Routing

The workflow determines whether the abandoned checkout is:

COD
or
Online / Prepaid

The routing is configuration-driven using a COD gateway list rather than relying on one hard-coded payment provider.

Online / UPI-style recovery

For online payment customers:

Generate Razorpay Payment Link
        ↓
Record Payment Attempt
        ↓
Fetch Product Image
        ↓
Build Recovery Message
        ↓
WhatsApp

The workflow creates a payment link through Razorpay's Payment Links API.

COD recovery

For COD customers:

Build COD Recovery Message
        ↓
Fetch Product Image
        ↓
WhatsApp

Product information is enriched from Shopify because the abandoned-checkout payload does not necessarily contain the required product image information.

📲 5. First-Touch WhatsApp Recovery

The customer receives the first recovery message after the 15-minute check.

The workflow records whether the message was successfully sent and records a failure state when the send operation fails.

Example concept:

Hi 👋

You left something in your cart.

Your [Product Name] is still waiting for you.

Complete your purchase here:
[Payment / Checkout Link]

For COD, the message follows the COD-specific recovery path.
⏰ 6. Second Check After 4 Hours

After the first-touch message:

Wait 4 Hours
      ↓
Re-check Shopify Order
      ↓
Purchased?

If the customer purchased:

Mark Recovered

No further recovery message is sent.

🎟️ 7. Real Shopify Coupon Recovery

When the customer still hasn't purchased after the second check, the workflow can create a real Shopify discount code.

Coupon Enabled?
       ↓
Check Existing Coupon
       ↓
Coupon Exists?
    ↙       ↘
  YES        NO
   ↓          ↓
Reuse      Shopify GraphQL
Coupon     Discount Creation
             ↓
         Store Coupon
             ↓
       Build Message
             ↓
         WhatsApp

The coupon is structurally unreachable from the first-touch branch. It is only created in the second-touch recovery path, and the implementation uses Shopify's discountCodeBasicCreate rather than generating a random string locally.

The default discount represented in the workflow is 5%.

💰 Workflow 2 — Razorpay Payment Synchronization

A separate Razorpay webhook workflow is required because a Razorpay Payment Link payment does not automatically mean that the Shopify checkout has become a completed Shopify order.

Razorpay Payment Webhook
          ↓
Verify Razorpay Signature
          ↓
Extract Event ID
          ↓
Persist Event / Idempotency
          ↓
Validate Payment Event
          ↓
Find Recovery Attempt
          ↓
Mark Payment Attempt Paid
          ↓
Check Shopify Order
       ↙         ↘
Existing       Not Existing
   ↓                ↓
Link Order      Create Shopify
& Recover       Order as Paid
                      ↓
                  Mark Recovered

This workflow prevents a customer who successfully pays through a Razorpay Payment Link from incorrectly receiving a later coupon simply because Shopify had not yet reflected that payment.

It also handles the case where the Shopify order already exists versus needing to create the order after successful payment.

🔒 Reliability & Production Design

A major focus of this project was making the workflow reliable under real-world conditions.

HMAC Security

Shopify webhook requests are verified against the raw request body using HMAC-SHA256.

Razorpay webhooks are also verified using HMAC-SHA256 with the configured webhook secret.

Idempotency

The system uses persistent database records to prevent duplicate:

Shopify checkout processing
Razorpay webhook processing
Recovery attempts
Payment attempts
Coupon creation

Re-check Before Messaging

The workflow does not blindly trust the original abandoned-cart event.

It checks Shopify again before the first message and again before the second-touch coupon.

Failure Handling

External API operations are configured with retries and error outputs that route into explicit failure-recording nodes rather than silently ending the workflow.

🗄️ PostgreSQL State Management

The automation uses PostgreSQL to persist operational state.

Key tables represented in the workflow include:

cartrecovery.webhook_events
cartrecovery.idempotency_keys
cartrecovery.customers
cartrecovery.carts
cartrecovery.recovery_attempts
cartrecovery.payment_attempts
cartrecovery.recovery_messages
cartrecovery.coupon_offers
cartrecovery.razorpay_webhook_events

This allows the automation to maintain a durable history of recovery attempts, payment events, messages, coupons, and recovery outcomes.

⚙️ Environment Variables

The workflow expects configuration such as:

SHOPIFY_STORE_URL=
SHOPIFY_WEBHOOK_SECRET=

COD_GATEWAY_NAMES=

COUPON_ENABLED_FOR_COD=true
COUPON_ENABLED_FOR_ONLINE=true
COUPON_VALIDITY_HOURS=48

RAZORPAY_WEBHOOK_SECRET=

Sensitive API credentials should be configured through n8n credentials/environment variables, not hard-coded into the workflow.

🛠️ Tech Stack
Technology	Purpose
n8n	Workflow orchestration
Shopify	Checkout, products, orders & discount management
PostgreSQL	Persistent state & idempotency
WhatsApp	Customer recovery messaging
Razorpay	Online payment links & payment events
JavaScript	Validation, routing & business logic
Shopify GraphQL API	Discount-code creation
📊 Recovery State Model

The workflow maintains recovery states such as:

pending
first_touch_sent
recovered
closed_no_purchase
coupon_sent
failed
payment_received_order_sync_pending
payment_attempt_mismatch

This gives the system a durable state machine instead of relying only on transient n8n execution data.

📈 Business Value

This automation is designed to help D2C and retail brands:

Recover abandoned checkouts
Reduce repetitive manual follow-ups
Give online customers a direct payment option
Handle COD and prepaid customers differently
Avoid messaging customers who already purchased
Automatically follow up after 4 hours
Issue unique recovery discounts
Synchronize successful Razorpay payments with Shopify
Maintain a persistent recovery history
🧠 Key Engineering Decisions

This project was built around a few important principles:

Verify before processing
        ↓
Validate before waiting
        ↓
Deduplicate before acting
        ↓
Re-check before messaging
        ↓
Check consent before messaging
        ↓
Use deterministic payment routing
        ↓
Create coupons only when eligible
        ↓
Persist every important state change
        ↓
Handle failures explicitly

The architecture deliberately keeps the coupon creation path isolated to the second-touch branch and uses real Shopify discount infrastructure rather than simulated coupon codes.

⚠️ Deployment Notes

The exported workflow contains placeholders such as:

YOUR_WHATSAPP_PHONE_NUMBER_ID

and requires the relevant Shopify, Razorpay, WhatsApp, and PostgreSQL credentials/configuration to be supplied before live deployment.

The workflow should be tested with real Shopify webhook payloads, Razorpay events, WhatsApp templates, API credentials, and database records before enabling it for live customers.

📁 Project Structure
D2C Cart Recovery Automation
│
├── Workflow 1
│   ├── Shopify Abandoned Checkout
│   ├── HMAC Verification
│   ├── Validation
│   ├── Idempotency
│   ├── 15-Minute Delay
│   ├── Purchase Re-check
│   ├── WhatsApp Eligibility
│   ├── COD / Online Routing
│   ├── Razorpay Payment Link
│   ├── WhatsApp First Touch
│   ├── 4-Hour Delay
│   ├── Purchase Re-check
│   ├── Shopify Coupon Creation
│   └── WhatsApp Second Touch
│
└── Workflow 2
    ├── Razorpay Webhook
    ├── HMAC Verification
    ├── Event Idempotency
    ├── Payment Reference Lookup
    ├── Payment State Update
    ├── Shopify Order Check
    ├── Shopify Order Creation
    └── Recovery State Update
🚀 Project Summary

Project: AI-Powered Cart Recovery & Payment Recovery Automation
Industry: D2C & Retail
Primary Problem: High cart abandonment through online/UPI and COD checkout
Platform: n8n
Commerce Platform: Shopify
Payment Gateway: Razorpay
Messaging: WhatsApp
Database: PostgreSQL
Automation Design: 2-workflow recovery architecture

This project was built to go beyond a simple “send a reminder” automation — the focus was on building a reliable recovery system that verifies events, respects consent, re-checks purchase state, synchronizes payments, creates real Shopify discounts, and records failures instead of silently losing them.
