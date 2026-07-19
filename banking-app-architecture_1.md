# Banking Application — Architecture & Design Document
**Scope:** NEFT, RTGS, IMPS, UPI, International Payments, Insurance Payments
**Stage:** Startup MVP

---

## 1. Strategic Foundation

As a startup, you will **not** build direct connections to NEFT/RTGS/UPI rails yourself — that requires being a licensed bank or NPCI member. Instead, you orchestrate payments through licensed partners.

| Rail | Who you actually need |
|---|---|
| NEFT / RTGS / IMPS | A sponsor/partner bank's API Banking product (ICICI, Yes Bank, Axis) or a BaaS provider (Setu, Decentro, M2P, Cashfree Payouts, RazorpayX) |
| UPI | Either become an NPCI-certified TPAP (slow, heavy compliance) or integrate via a UPI-enabled PG/PSP partner (faster for MVP) |
| International payments | An Authorized Dealer (AD) bank tie-up, or cross-border specialists like Nium, Thunes, Wise Platform, Cashfree's cross-border product — plus RBI LRS/FEMA compliance |
| Insurance payments | Usually just a payment gateway integration on top of insurer/insurtech APIs — unless you also want to *distribute* insurance, which needs an IRDAI Web Aggregator/Corporate Agent license |

You will almost certainly need a **Payment Aggregator (PA) license from RBI** if you collect and route funds on behalf of merchants/users — mandatory, not optional.

---

## 2. High-Level Architecture

```
                        ┌─────────────────────┐
                        │   Client Apps        │
                        │ (Web / Mobile / API)  │
                        └──────────┬───────────┘
                                   │
                        ┌──────────▼───────────┐
                        │   API Gateway          │
                        │ (auth, rate-limit,     │
                        │  idempotency keys)     │
                        └──────────┬───────────┘
                                   │
        ┌──────────────┬──────────┼──────────┬───────────────┐
        │              │          │          │               │
   ┌────▼────┐   ┌─────▼────┐ ┌──▼─────┐ ┌──▼──────┐  ┌──────▼──────┐
   │ KYC/    │   │ Payment   │ │ Ledger │ │ Risk &  │  │ Notification │
   │ Onboard │   │ Orchestr. │ │ Service│ │ Fraud   │  │ Service      │
   │ Service │   │ Engine    │ │(double │ │ Engine  │  │ (SMS/Email/  │
   └─────────┘   └─────┬─────┘ │ entry) │ └─────────┘  │  Push)       │
                        │       └────────┘              └─────────────┘
        ┌───────────────┼────────────────────────┐
        │               │                         │
   ┌────▼────┐    ┌─────▼─────┐            ┌──────▼──────┐
   │ NEFT/   │    │ UPI PSP   │            │ Cross-border │
   │ RTGS/   │    │ Connector │            │ / Insurance  │
   │ IMPS    │    │           │            │ Connectors   │
   │ Connector│   └───────────┘            └──────────────┘
   └─────────┘
        │
   ┌────▼─────────────┐
   │ Sponsor Bank /    │
   │ BaaS Partner APIs │
   └───────────────────┘
```

### Core services

- **Payment Orchestration Engine** — picks the right rail based on amount, urgency, beneficiary bank, cut-off times. Handles retries, fallback rails, and partner failover.
- **Ledger Service** — true double-entry ledger (not just a balance field), for auditability and reconciliation. The single most safety-critical piece.
- **Reconciliation & Settlement Engine** — matches internal ledger against partner bank statements daily.
- **Risk & Fraud Engine** — velocity checks, AML screening, transaction limits, OFAC/sanctions screening (critical for international payments).
- **KYC/Onboarding** — Aadhaar e-KYC / Video KYC / CKYC integration, mandatory before any wallet/account creation.

---

## 3. Compliance Checklist

- **RBI Payment Aggregator (PA) authorization** — needed before going live with real money flows
- **PCI-DSS** — if touching card data at all
- **Data localization** — RBI mandates all payment data stored only in India
- **PMLA/AML KYC** norms
- **IRDAI license** — only if distributing insurance, not just collecting premium payments

---

## 4. Ledger & Database Schema Design

Core principle: **never store a single "balance" field as the source of truth.** Use double-entry bookkeeping where every transaction creates at least two balanced entries (debit + credit).

### `accounts`
Represents any holder of value (user wallet, merchant account, partner bank suspense account, fee/revenue account).

```sql
CREATE TABLE accounts (
    id              UUID PRIMARY KEY,
    account_type    VARCHAR(30) NOT NULL, -- 'user_wallet', 'merchant', 'suspense', 'fee_revenue', 'partner_nostro'
    owner_id        UUID,                  -- FK to users/merchants, nullable for system accounts
    currency        CHAR(3) NOT NULL DEFAULT 'INR',
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### `ledger_entries`
Immutable, append-only. Single source of truth.

```sql
CREATE TABLE ledger_entries (
    id              UUID PRIMARY KEY,
    transaction_id  UUID NOT NULL,         -- groups entries belonging to one transaction
    account_id      UUID NOT NULL REFERENCES accounts(id),
    entry_type      VARCHAR(6) NOT NULL CHECK (entry_type IN ('DEBIT','CREDIT')),
    amount          NUMERIC(20,4) NOT NULL CHECK (amount > 0),
    currency        CHAR(3) NOT NULL,
    balance_after   NUMERIC(20,4) NOT NULL, -- snapshot for fast lookups/audits
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    metadata        JSONB                    -- rail used, UTR, narration etc.
);
CREATE INDEX idx_ledger_account ON ledger_entries(account_id, created_at);
CREATE INDEX idx_ledger_txn ON ledger_entries(transaction_id);
```

Critical rule: for every `transaction_id`, `SUM(debits) = SUM(credits)`. Enforce this in application logic with a DB transaction wrapping all entries, and add a nightly batch job that verifies this invariant across the whole ledger — any imbalance is a P0 alert.

### `transactions`
Business-level record (one row per user-initiated payment), separate from ledger entries.

```sql
CREATE TABLE transactions (
    id                  UUID PRIMARY KEY,
    type                VARCHAR(30) NOT NULL, -- 'NEFT','RTGS','IMPS','UPI','INTL','INSURANCE_PREMIUM'
    status              VARCHAR(20) NOT NULL, -- 'INITIATED','PENDING','SUCCESS','FAILED','REVERSED'
    source_account_id   UUID NOT NULL REFERENCES accounts(id),
    dest_account_id     UUID,                  -- nullable if external (e.g. NEFT to other bank)
    dest_bank_ifsc      VARCHAR(11),
    dest_account_number VARCHAR(34),
    amount              NUMERIC(20,4) NOT NULL,
    currency            CHAR(3) NOT NULL,
    rail_reference      VARCHAR(64),          -- UTR/RRN from NPCI/bank
    idempotency_key     VARCHAR(64) UNIQUE NOT NULL,
    partner_provider    VARCHAR(30),           -- 'setu','decentro', etc.
    initiated_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at        TIMESTAMPTZ,
    failure_reason      TEXT
);
```

### `transaction_state_log`
Every status transition, for audit trail.

```sql
CREATE TABLE transaction_state_log (
    id              BIGSERIAL PRIMARY KEY,
    transaction_id  UUID NOT NULL REFERENCES transactions(id),
    old_status      VARCHAR(20),
    new_status      VARCHAR(20) NOT NULL,
    changed_by      VARCHAR(50), -- 'system','webhook','admin','reconciliation_job'
    changed_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    notes           TEXT
);
```

### `reconciliation_records`
Matches your ledger against partner/bank statements.

```sql
CREATE TABLE reconciliation_records (
    id                  UUID PRIMARY KEY,
    transaction_id      UUID REFERENCES transactions(id),
    partner_statement_ref VARCHAR(64),
    match_status        VARCHAR(20), -- 'MATCHED','UNMATCHED','MISMATCH_AMOUNT','ORPHAN'
    reconciled_at        TIMESTAMPTZ,
    raw_partner_record   JSONB
);
```

### Non-negotiable design rules

1. **Idempotency everywhere.** Every payment API call must carry a client-generated `idempotency_key`. Network retries are common with bank rails — without this you'll double-debit users.
2. **Never UPDATE a balance directly.** Balances are always derived (`SUM` of ledger entries) or cached with a `balance_after` snapshot updated only by appending new entries.
3. **Use DB-level row locking** (`SELECT ... FOR UPDATE`) on the account row during transaction creation to prevent race conditions (two simultaneous debits exceeding balance).
4. **Separate "authorized/held" from "settled" state** — many rails (especially UPI, IMPS) have a pending window where you shouldn't show funds as fully available.
5. **Use NUMERIC, never FLOAT**, for all money fields — floating point rounding errors are unacceptable in finance.
6. **Partition `ledger_entries` and `transaction_state_log`** by month once you have volume — these tables grow forever and are append-only, perfect for partitioning.

---

## 5. Payment Orchestration Logic (Rail Selection)

### Rail selection pseudocode

```
function selectRail(amount, urgency, destBank, time, isInternational):

    if isInternational:
        return INTERNATIONAL_RAIL  # SWIFT via AD bank partner

    if urgency == "INSTANT":
        if amount <= 100000:           # UPI per-txn limit (typically ~1L, higher for select categories)
            return UPI
        else:
            return IMPS              # IMPS supports up to ~5L typically, 24x7 instant

    if urgency == "SAME_DAY" and amount >= 200000:
        if isWithinRTGSWindow(time):  # RTGS: 24x7 now in India, but check partner cutoffs
            return RTGS
        else:
            return IMPS  # fallback if RTGS batch window closed for partner

    if urgency == "STANDARD":
        return NEFT  # batch settlement, half-hourly windows, now 24x7 too

    # Default fallback chain
    return tryInOrder([UPI, IMPS, NEFT])
```

### Rail constraints reference table

| Rail | Min/Max amount | Settlement | Availability |
|---|---|---|---|
| UPI | No min, ~1L per txn (higher for select categories) | Real-time | 24x7 |
| IMPS | No min, ~5L max (bank-dependent) | Real-time | 24x7 |
| NEFT | No min/max | Batched (near real-time) | 24x7 since Dec 2019 |
| RTGS | Min 2L, no max | Real-time gross settlement | 24x7 since Dec 2020 |

*These limits change periodically via RBI circulars — pull limits from a config table, not hardcoded values. Your BaaS partner's API usually enforces/returns current limits too.*

### Failover & retry strategy

```
1. Attempt rail via Partner A
2. If timeout/5xx → retry with exponential backoff (max 3 attempts, idempotency key reused)
3. If rail itself fails (e.g. beneficiary bank down) →
     - For IMPS/UPI failures → auto-fallback to NEFT (slower but more reliable)
     - For RTGS failures → fallback to NEFT if amount allows, else queue and alert
4. If all rails fail → mark transaction PENDING_MANUAL_REVIEW, alert ops team
5. Always reconcile within T+1 day even for "successful" transactions
```

### Transaction state machine

```
INITIATED → VALIDATED → RAIL_SELECTED → SENT_TO_PARTNER →
   ├─→ PARTNER_ACK → PROCESSING → SUCCESS
   ├─→ PARTNER_ACK → PROCESSING → FAILED → (refund flow) → REVERSED
   └─→ TIMEOUT → RECONCILIATION_PENDING → (resolved via statement matching) → SUCCESS/FAILED
```

Webhooks from your BaaS partner update this state — but **never trust a webhook alone**. Always run a polling/reconciliation job that double-checks the partner's transaction status API every few minutes for anything not in a terminal state, since webhooks can be missed or delayed.

### Idempotency at the orchestration layer

Every external call to a bank/UPI partner should use a **deterministic reference ID** derived from your `idempotency_key`, so retries are recognized as the same request rather than creating a duplicate payment.

---

## 6. Vendor / BaaS Comparison

| Provider | Strengths | Rails Covered | Best For | Watch-outs |
|---|---|---|---|---|
| **Setu** | Strong API design, good docs, backed by Pine Labs, account aggregator + payments combo | UPI, NEFT/RTGS/IMPS payouts, AA, BBPS | Startups wanting clean modern APIs and fast integration | Smaller partner bank network than M2P |
| **Decentro** | Wide product suite (payments, lending, KYC) in one platform | UPI, NEFT/RTGS/IMPS, KYC, escrow accounts | Startups wanting one vendor for KYC + payments + lending infra | Pricing can stack up across modules |
| **Cashfree Payments** | Mature, high reliability, strong PG + payout combo | UPI, NEFT/RTGS/IMPS, cross-border (limited), PG | Startups prioritizing reliability/scale over cutting-edge API design | Less flexible for deep banking-as-a-service needs |
| **M2P Fintech** | Deepest banking infra (card issuance, core banking, lending stack) | Full BaaS — accounts, cards, payments, lending | Startups planning to issue cards/full neobank accounts | Heavier onboarding, more enterprise-sales-driven |
| **RazorpayX** | Easiest to start if already using Razorpay PG, good payout APIs | UPI, NEFT/RTGS/IMPS payouts | Startups already in Razorpay ecosystem | Less suited for complex BaaS/neobank features |

### International payments specifically
- **Nium** — strong cross-border payout network, good for B2B/B2C remittance
- **Wise Platform** — reliable, transparent FX, but less India-specific BaaS integration
- **Thunes** — broad emerging-market payout coverage

### Recommendation for MVP stage
Start with **Setu or Cashfree** for domestic rails (NEFT/RTGS/IMPS/UPI) — fastest to integrate, good sandbox environments, reasonable compliance support. Add **Nium** separately for international payments once domestic flow is stable. For insurance premium collection, route through the same payment gateway as a use-case/category — unless also building insurance distribution (separate IRDAI licensing track).

---

*Document prepared as architecture reference for MVP planning. Regulatory limits (UPI/IMPS/RTGS thresholds, licensing requirements) should be verified against current RBI/NPCI circulars before implementation, as these are revised periodically.*
