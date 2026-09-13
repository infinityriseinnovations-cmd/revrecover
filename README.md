# RevRecover (Dunnify) — Complete Architecture, Multi-Gateway Dunning Engine & Scale Plan

> **The Next-Generation Automated Revenue Recovery & Multi-Gateway Dunning SaaS Platform**
> *Built to capture lost subscription MRR across Stripe, Razorpay, PayU India, Cashfree Payments, PhonePe PG, Paytm PG, and CCAvenue using WhatsApp-first recovery sequences, zero-friction smart payment links, and native DPDP Act (2023) data protection compliance.*

---

## 1. Executive Summary & Market Position

Subscription-based platforms in emerging market hubs (SaaS, D2C Subscriptions, EdTech, OTT, and FinTech) lose between **9% and 18% of their Monthly Recurring Revenue (MRR)** every month due to **Involuntary Churn**—failed recurring card tokens, UPI Autopay mandate execution failures, bank clearance timeouts, and insufficient wallet/account balances.

While legacy dunning tools like ProfitWell Retain, Baremetrics, and Butter Payments focus strictly on Western enterprise markets tied exclusively to Stripe and email flows, **RevRecover** provides a localized, **WhatsApp-First, AI-Dynamic, Multi-Gateway Architecture** supporting both international gateways and the complete suite of Indian payment stack providers.

---

## 2. DPDP Act (2023) Data Protection & Compliance Architecture

Operating subscription dunning workflows in India requires strict adherence to the **Digital Personal Data Protection (DPDP) Act, 2023**. RevRecover is architected ground-up as a compliant Data Processor.

```
                  +──────────────────────────────────────────────────+
                  |    Data Principal (End-User Subscriber)           |
                  +─────────────────────────┬────────────────────────+
                                            │
                                            ▼
                  +──────────────────────────────────────────────────+
                  |    Consent & Tokenized Scope Verifier            |
                  |  - Explicit Transaction Consent Verification     |
                  |  - Temporary Scoped Token Access (72h TTL)        |
                  +─────────────────────────┬────────────────────────+
                                            │
                                            ▼
                  +──────────────────────────────────────────────────+
                  |    DPDP Vault & Encrypted Processing Node       |
                  |  - AWS ap-south-1 (Mumbai Region) Residency       |
                  |  - AES-256 Field-Level PII Encryption            |
                  |  - Anonymized Analytical Logs Storage             |
                  +──────────────────────────────────────────────────+
```

### Compliance Controls Checklist

| Compliance Vector | DPDP Requirement | RevRecover Technical Implementation |
| :--- | :--- | :--- |
| **Data Localization** | Personal data stored within Indian territory | Hosted exclusively in AWS `ap-south-1` (Mumbai) & GCP `asia-south1` |
| **Right to Erasure** | Immediate deletion upon request / mandate withdrawal | Automated Redis & PostgreSQL cascade purge via `DELETE /v1/privacy/purge` |
| **Notice & Purpose** | Specific, clear notice for processing payment recovery | Consent token embedded in WhatsApp/SMS links with explicit transaction scope |
| **Data Minimization** | Collect only data necessary for processing | Zero storage of raw PAN/CVV; store only gateway tokens & masked phone numbers |
| **Audit Trails** | Immutable logs of consent and data access | SHA-256 hashed audit log table with zero PII exposure |

---

## 3. End-to-End System Architecture & Data Flow

RevRecover acts as a high-throughput, real-time event listener, unified payload translator, AI scheduling engine, and multi-channel communication orchestra.

```
                  +──────────────────────────────────────────────────+
                  |            Payment Gateway Webhooks              |
                  | (Stripe / Razorpay / Cashfree / PayU / PhonePe / |
                  |               Paytm / CCAvenue)                  |
                  +─────────────────────────┬────────────────────────+
                                            │
                                            ▼
                  +──────────────────────────────────────────────────+
                  |         RevRecover Ingestion & Router            |
                  |    - Dynamic HMAC Signature Verification        |
                  |    - Payload Normalization Adapter Layer         |
                  +─────────────────────────┬────────────────────────+
                                            │
                                            ▼
                  +──────────────────────────────────────────────────+
                  |         AI Failure Classification Engine         |
                  |  - Insufficient Balance? (Trigger Payday Logic) |
                  |  - Expired Mandate / Card Token?                |
                  |  - Bank Clearing / Server Downtime Error?       |
                  +─────────────────────────┬────────────────────────+
                                            │
        ┌───────────────────────────────────┼───────────────────────────────────┐
        ▼                                   ▼                                   ▼
+───────────────────────+       +───────────────────────+       +───────────────────────+
| Channel 1: WhatsApp   |       |   Channel 2: Email    |       |   Channel 3: SMS      |
| Interactive Buttons & |       | Smart Dynamic Link &  |       | Urgent Payment Notice |
| Quick Pay Links       |       | HTML Invoice Summary  |       | & Mandate Warning     |
+───────────┬───────────+       +───────────┬───────────+       +───────────┬───────────+
            │                               │                               │
            └───────────────────────────────┼───────────────────────────────┘
                                            │
                                            ▼
                  +──────────────────────────────────────────────────+
                  |   Tokenized Single-Click Payment Recovery Page     |
                  |   (Zero Password / Scoped Gateway Router Page)    |
                  +─────────────────────────┬────────────────────────+
                                            │
                                            ▼
                  +──────────────────────────────────────────────────+
                  | Payment Success Webhook Fired back to Source PG  |
                  | Mandate Restored & Subscription State Updated    |
                  +──────────────────────────────────────────────────+
```

---

## 4. Multi-Gateway Adapter & Webhook Ingestion Engine

### Standardized Webhook Ingestion Handler (`Node.js / TypeScript`)

```typescript
import { Request, Response } from 'express';
import crypto from 'crypto';

export interface StandardDunningPayload {
  gateway: 'stripe' | 'razorpay' | 'cashfree' | 'payu' | 'phonepe' | 'paytm' | 'ccavenue';
  eventId: string;
  tenantId: string;
  customerId: string;
  customerPhone?: string;
  customerEmail?: string;
  amount: number;
  currency: string;
  failureReason: 'INSUFFICIENT_FUNDS' | 'EXPIRED_CARD' | 'MANDATE_EXPIRED' | 'NETWORK_TIMEOUT' | 'UNKNOWN';
  subscriptionId: string;
  mandateId?: string;
}

export async function webhookIngestionRouter(req: Request, res: Response) {
  const provider = (req.headers['x-provider'] as string) || (req.query.provider as string);
  let normalizedPayload: StandardDunningPayload | null = null;

  try {
    switch (provider) {
      case 'stripe':
        if (!verifyStripeSignature(req)) return res.status(401).send('Invalid Stripe Signature');
        normalizedPayload = parseStripeEvent(req.body);
        break;

      case 'razorpay':
        if (!verifyRazorpaySignature(req)) return res.status(401).send('Invalid Razorpay Signature');
        normalizedPayload = parseRazorpayEvent(req.body);
        break;

      case 'cashfree':
        if (!verifyCashfreeSignature(req)) return res.status(401).send('Invalid Cashfree Signature');
        normalizedPayload = parseCashfreeEvent(req.body);
        break;

      case 'payu':
        if (!verifyPayUSignature(req.body)) return res.status(401).send('Invalid PayU Signature');
        normalizedPayload = parsePayUEvent(req.body);
        break;

      case 'phonepe':
        if (!verifyPhonePeSignature(req)) return res.status(401).send('Invalid PhonePe Signature');
        normalizedPayload = parsePhonePeEvent(req.body);
        break;

      case 'paytm':
        if (!verifyPaytmSignature(req.body)) return res.status(401).send('Invalid Paytm Signature');
        normalizedPayload = parsePaytmEvent(req.body);
        break;

      case 'ccavenue':
        const decryptedBody = decryptCCAvenuePayload(req.body.encResp);
        normalizedPayload = parseCCAvenueEvent(decryptedBody);
        break;

      default:
        return res.status(400).json({ error: 'Unsupported Gateway Provider' });
    }

    if (normalizedPayload) {
      await enqueueDunningWorkflow(normalizedPayload);
    }

    return res.status(200).json({ status: 'success', eventId: normalizedPayload?.eventId });
  } catch (error) {
    console.error('Webhook Ingestion Error:', error);
    return res.status(500).json({ error: 'Internal Ingestion Error' });
  }
}

// Verification Handlers
function verifyRazorpaySignature(req: Request): boolean {
  const secret = process.env.RAZORPAY_WEBHOOK_SECRET!;
  const signature = req.headers['x-razorpay-signature'] as string;
  const expected = crypto.createHmac('sha256', secret).update(JSON.stringify(req.body)).digest('hex');
  return signature === expected;
}

function verifyCashfreeSignature(req: Request): boolean {
  const secret = process.env.CASHFREE_CLIENT_SECRET!;
  const timestamp = req.headers['x-webhook-timestamp'] as string;
  const signature = req.headers['x-webhook-signature'] as string;
  const rawBody = JSON.stringify(req.body);
  const data = timestamp + rawBody;
  const expected = crypto.createHmac('sha256', secret).update(data).digest('base64');
  return signature === expected;
}

function verifyPayUSignature(body: any): boolean {
  const salt = process.env.PAYU_MERCHANT_SALT!;
  const hashString = `${salt}|${body.status}|||||||||||${body.email}|${body.firstname}|${body.productinfo}|${body.amount}|${body.txnid}|${body.key}`;
  const expected = crypto.createHash('sha512').update(hashString).digest('hex');
  return body.hash === expected;
}
```

---

## 5. PostgreSQL Data Architecture (Schema DDL)

```sql
-- Enable UUID extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Tenants (SaaS Clients)
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    company_name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    mrr_tier VARCHAR(50) DEFAULT 'Starter',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Gateways Configuration
CREATE TABLE tenant_gateways (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE,
    gateway_name VARCHAR(50) NOT NULL,
    api_key_encrypted TEXT NOT NULL,
    webhook_secret_encrypted TEXT NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Failed Payment Log Entries
CREATE TABLE failed_invoices (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE,
    gateway VARCHAR(50) NOT NULL,
    external_invoice_id VARCHAR(255) NOT NULL,
    customer_name VARCHAR(255),
    customer_email VARCHAR(255),
    customer_phone VARCHAR(50),
    amount_inr NUMERIC(12, 2) NOT NULL,
    failure_reason VARCHAR(100) NOT NULL,
    status VARCHAR(50) DEFAULT 'RECOVERING', -- RECOVERING, RECOVERED, FAILED
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Dunning Logs (Execution Steps)
CREATE TABLE dunning_logs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    invoice_id UUID REFERENCES failed_invoices(id) ON DELETE CASCADE,
    channel VARCHAR(50) NOT NULL, -- WhatsApp, Email, SMS
    attempt_number INT NOT NULL,
    status VARCHAR(50) NOT NULL, -- SENT, DELIVERED, READ, CLICKED
    executed_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_failed_invoices_tenant ON failed_invoices(tenant_id);
CREATE INDEX idx_failed_invoices_status ON failed_invoices(status);
```

---

## 6. Smart Retry AI & Payday Logic Engine

RevRecover evaluates failure metadata to optimize retry dates and maximize conversion:

1. **Salary Cycle Targeting (1st–5th of Month):** If failure is categorized as `INSUFFICIENT_FUNDS`, retry execution is automatically scheduled for the 1st, 3rd, or 5th day of the month when bank balances refresh.
2. **Bank Clearing Timeout Handler:** If failure occurs due to bank server timeouts (`NETWORK_TIMEOUT`), retries fire at 06:00 AM IST during off-peak clearance hours.
3. **Mandate Expiry Router:** Automatically routes the customer to a passwordless mandate re-authorization page via UPI Autopay.

---

## 7. Scale Plan to ₹10 Lakhs MRR (₹1.2 Cr ARR)

```
                        GROWTH & MONETIZATION ROADMAP
                       
  Phase 1: Founder Outreach       Phase 2: Self-Serve SaaS      Phase 3: Agency & Enterprise
    (Months 1-3: 0 to ₹1.5L)       (Months 4-8: ₹1.5L to ₹6L)       (Months 9-12: ₹6L to ₹10L+)
┌────────────────────────────┐ ┌────────────────────────────┐ ┌────────────────────────────┐
│ • Direct outreach to 30    │ │ • Launch self-serve 1-click│ │ • Launch White-Label Agency│
│   Indian SaaS & D2C brands.│ │   gateway onboarding.      │ │   portal for digital agency│
│ • Zero-Risk Performance:   │ │ • Launch Growth & Pro      │ │   clusters.                │
│   "10% of recovered revenue│ │   plans (₹3,999 - ₹11,999).│ │ • Enterprise SLAs, custom  │
│   in Month 1, zero upfront"│ │ • Targeted content marketing│ │   gateway routing & dynamic│
│                            │ │   ("Razorpay Dunning").    │ │   bank uptime retries.     │
└────────────────────────────┘ └────────────────────────────┘ └────────────────────────────┘
```

### Financial Model (Path to ₹1,000,000 / Month)

| Customer Tier | Monthly Fee | Target Accounts | Revenue Contribution |
| :--- | :--- | :--- | :--- |
| **Starter Tier** | ₹3,999 | 50 Accounts | ₹199,950 |
| **Growth Tier** | ₹11,999 | 50 Accounts | ₹599,950 |
| **Enterprise Tier** | ₹39,999 | 5 Accounts | ₹199,995 |
| **Total** | | **105 Accounts** | **₹999,895 / mo** |

---

## 8. Next Immediate Action Items

1. **Interactive Dashboard App**: Access the interactive prototype in `App.jsx` to test live dunning sequences, customer payment flows, and ROI calculations.
2. **Database Provisioning**: Execute the PostgreSQL schema DDL on Supabase / Managed AWS RDS.
3. **Meta WhatsApp Business Registration**: Submit pre-approved payment recovery templates for WhatsApp transactional API access.
```

### Suggested Next Steps:
* Would you like me to generate the **Meta WhatsApp Template JSON payload** for official Meta API verification?
* Would you like me to build out specific **Express API endpoints** for managing tenant gateway authentication keys?
