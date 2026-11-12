# QeakFlow Architecture

Design notes for **QeakFlow**, a managed, privacy-first email marketing platform for the DACH market, built as a multi-tenant service on top of [Listmonk](https://listmonk.app).

This repository contains **documentation only**: diagrams, design decisions and lessons learned. The application code is proprietary and not published here.

> Built and operated by a single engineer: backend, infrastructure, billing and provisioning.

---

## 1. Problem and constraints

Small businesses in Germany want newsletters without Mailchimp's pricing jumps and without sending subscriber data to third parties (GDPR).

Design constraints that shaped the architecture:

- **Strong tenant isolation:** one customer's data and failures must not affect another's.
- **Fully automated onboarding:** payment to a working workspace with no manual steps.
- **Solo operation:** every component must be simple to run, observable and recoverable by one person.
- **EU-hosted, minimal tracking by default.**

## 2. System overview

```mermaid
flowchart LR
  U[Customer] --> API[FastAPI: auth, plans, routing]
  API --> PG[(PostgreSQL)]
  API <-->|checkout, webhooks| S[Stripe]
  API -->|enqueue job| R[(Redis)]
  R --> W[arq worker: provisioning]
  W --> D[Docker host]
  D --> L1[Listmonk: tenant A]
  D --> L2[Listmonk: tenant B]
  L1 & L2 --> SMTP[SMTP: Amazon SES or customer-provided]
```

| Layer | Technology |
|---|---|
| API, auth, billing logic | FastAPI (Python) |
| Data | PostgreSQL |
| Background jobs | arq + Redis |
| Tenant runtime | Docker / Docker Compose, one Listmonk instance per tenant |
| Payments | Stripe (webhook-driven) |
| Sending | Amazon SES or customer-provided SMTP |

## 3. Tenant provisioning flow

Payment triggers provisioning automatically; the customer never waits on a human.

```mermaid
sequenceDiagram
  participant C as Customer
  participant S as Stripe
  participant A as FastAPI
  participant R as Redis / arq
  participant W as Worker
  participant D as Docker
  C->>S: Subscribe
  S->>A: Webhook (checkout completed)
  A->>A: Verify signature, record event once
  A->>R: Enqueue provisioning job
  R->>W: Deliver job
  W->>W: Create tenant database and credentials
  W->>D: Start isolated Listmonk container
  W->>D: Configure instance (token-based auth)
  W->>A: Mark tenant provisioned
  A-->>C: Workspace ready
```

## 4. Onboarding state machine


```mermaid
stateDiagram-v2
  [*] --> PendingPayment
  PendingPayment --> Provisioning: payment confirmed
  Provisioning --> DomainSetup: instance ready
  Provisioning --> Failed: error
  Failed --> Provisioning: retry
  DomainSetup --> DnsVerification: records published
  DnsVerification --> Active: records verified
  Active --> Deactivated: subscription ended
  Deactivated --> CleanedUp: retention period passed
  CleanedUp --> [*]
```

## 5. **Sender-domain authentication**

- A DKIM keypair is generated per tenant during onboarding; the customer receives the DNS records to publish.
- A DNS verification service (Python, `dnspython`) checks the published records before sending is enabled.

**SPF.** A domain may publish only one SPF record, and most customers already have one. QeakFlow therefore shows the `include:` mechanism for the tenant's SMTP provider to merge into the existing record, never a second record. Verification parses the record instead of comparing strings. It checks that the provider's include is present and the record ends in `~all` or `-all`. It also flags multiple SPF records and records over the 10-DNS-lookup limit, both of which cause a permanent SPF error.

**DKIM.** Verification resolves `<selector>._domainkey.<domain>`, joins multi-string TXT values, normalizes whitespace, and compares the `p=` key with the stored public key.

**DMARC.** DMARC is advisory and does not block sending. Verification only checks that a valid `v=DMARC1` record with a `p=` tag exists, and accepts any existing policy. For new setups we suggest `p=none` with an `rua` address, then tightening to `quarantine` once reports look clean.

**Gating.** Sending is enabled when SPF and DKIM pass. A missing DMARC record shows a warning.

**Re-check strategy.**
- A manual "Verify" button, rate-limited per domain.
- After a failed check, automatic background re-checks with backoff: 1 min, 5 min, 15 min, 1 h, then every 6 h for up to 72 h to cover DNS propagation. After that the status becomes `expired` and the user can restart.
- Queries go to public resolvers rather than the host's resolver.
- Timeouts and SERVFAIL are treated as `unknown` and retried. Only a definitive "no record" counts as `missing`, and a single timeout never flips a verified domain to failed.
- Verified domains are re-checked daily. If records disappear, the domain becomes `degraded`, the customer is notified, and sending is paused only after a grace period (e.g. 72 h).

**Failure display.**
- Each record (SPF, DKIM, DMARC) shows its own status: `valid`, `missing`, `mismatch`, `conflict` or `unknown`.
- Each record shows the exact host and value with a copy button, the last-checked time, and the next automatic check.
- Each failure comes with a specific hint, never a generic "verification failed":
  - "Many registrars append the domain automatically, so enter `qeakflow._domainkey`, not the full name."
  - "Remove surrounding quotes."
  - "You have two SPF records, merge them into one."
- The last result is persisted, so the page loads without a live DNS lookup.
## 6. Key design decisions

### 6.1 One Listmonk instance per tenant
- **Why:** hard isolation of data and failures, simple per-tenant deletion for GDPR, no changes needed to Listmonk's single-tenant design.
- **Trade-off:** higher resource cost per tenant and more orchestration work.
- **Mitigation:** automated provisioning and cleanup, so operational cost stays low for one person.

### 6.2 Provisioning as background jobs
- **Why:** provisioning is slow and can fail halfway; it must not block webhooks or HTTP requests.
- **How:** arq workers with Redis, jobs designed to be safely retried.
- **Idempotent provisioning & crash recovery** (State reconciliation, dangling container cleanup, and an automated rollback stack to safely handle workers dying mid-way)

### 6.3 Webhook-driven billing
- **Why:** Stripe is the source of truth for subscription state; the API reacts to events instead of polling.
- **How:** verifies each Stripe webhook signature, stores the Stripe event ID with a unique constraint to prevent duplicate processing, and processes events idempotently. For out-of-order events, it retrieves the latest subscription state from Stripe before updating the local subscription record and synchronizing the tenant's Listmonk instance.

### 6.4 Bring-your-own SMTP
- **Why:** lets customers use their own sending reputation and keeps deliverability responsibility transparent.

## 7. Lifecycle and cleanup

Tenant states: `active` → `past_due` → `frozen` → `deactivated`.

- **Payment failure.** The subscription goes `past_due` and Stripe retries the charge. The tenant keeps working during the grace period (7 days, configurable).
- **Freeze.** When the grace period expires, the tenant's container is stopped and the customer is emailed. Data is kept, but sending and API access stop.
- **Reactivation.** If the customer pays within the retention period, the container restarts with all data intact.
- **Purge.** After the retention period (12 months, configurable), the container and its volumes are removed. The tenant database and DB user are dropped, config files are wiped, stored SMTP credentials are deleted, and the customer gets a confirmation email.
- **Voluntary cancellation.** The subscription is set to cancel at period end. Access continues until then, and the tenant follows the same freeze and purge path.
- **Execution.** A daily scheduled job processes each tenant in its own transaction, so one failure doesn't block the batch. Purging runs as a background job with a per-tenant lock and can safely be re-run.
- **Before deletion.** Customers get a reminder email before the purge date so they can export their contacts. Backups keep deleted data until they rotate out.

## 8. Engineering Challenges & Solutions

**1. Provisioning is a distributed workflow, not a function call.**
The first version created the database, started the container, and configured the instance inside one HTTP request. It took 30-60 seconds and had no retries. A failure halfway left orphaned resources, for example a dropped database with a container still running and holding its port. I moved it to a background worker with a per-tenant lock, a rollback stack, and an explicit status stored in the database (`provisioning`, `active`, `error`). I should have built it that way from day one, because the happy path is the easy 20% of the work and the failure paths are the rest.

**2. Verify upstream behavior with a real instance before building on it.**
I lost time to assumptions about Listmonk's API. v6 rejects Basic Auth and needs token auth. The API user's token comes back in the `password` field. The health endpoint returns 403 before the instance is initialized. I now treat each provisioning step as a contract to check against a real container, and I'd run that as an integration test in CI from the start instead of finding the differences one failed provision at a time.

**3. Isolation has a per-tenant cost and a full lifecycle, so measure and design both early.**
A dedicated container and database per tenant gives strong data separation, but every idle tenant has a memory baseline. In my setup, 10 idle tenants take roughly 1.5 GB. That is a rough measurement, not a benchmark. I'd record the per-tenant footprint from the first prototype so pricing and server sizing rest on numbers. I also designed teardown late. Freezing after failed payments, purging after a retention period, and deleting SMTP credentials all came after provisioning, and they are as important as creating tenants.

## 9. Roadmap

Planned, in rough priority order. No dates; the order follows beta feedback.

**Next**
- Usage dashboard: contacts and sends against plan limits.
- Preflight check: test send to your own inbox with SPF/DKIM results before the first campaign.
- Secondary SMTP slot: automatic failover when the primary provider rate-limits.

**Later**
- Webhooks and REST API access for custom integrations.
- Multiple workspaces per account, for agencies managing several clients.

**Exploring**
- White-label (custom app domain, branding) for agencies.
- Zapier integration.
- Template builder, if Listmonk's editor proves insufficient.

---

**Author:** Andrii Mahalias · [qeakflow.com](https://www.qeakflow.com)
