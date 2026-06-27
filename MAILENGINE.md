# MailEngine

**A dedicated, independently deployable email service — built once, consumed by any project over HTTP, so email infrastructure never lives inside application code again.**

![Status](https://img.shields.io/badge/status-production-brightgreen)
![Runtime](https://img.shields.io/badge/runtime-Node.js-339933?logo=nodedotjs&logoColor=white)
![Framework](https://img.shields.io/badge/framework-Express-000000?logo=express&logoColor=white)
![Transport](https://img.shields.io/badge/transport-Nodemailer-22B573)
![Type](https://img.shields.io/badge/type-microservice-blue)
![Consumed By](https://img.shields.io/badge/consumed%20by-Muves-6366f1)

---

## What Is MailEngine

MailEngine is a self-contained HTTP service that handles all transactional email delivery. It exposes a minimal REST API — any project that needs to send an email makes a POST request to MailEngine and gets on with its work. SMTP configuration, provider credentials, and delivery mechanics stay entirely inside the service.

The reason it exists as a separate service rather than embedded email logic inside `kkaudiobk` or any other application comes down to where email configuration actually belongs. SMTP credentials, provider details, and anything touching the transport layer are infrastructure concerns — they have no business sitting inside application code. When email logic is embedded directly in a project, the project now owns an infrastructure responsibility it shouldn't have.

The problem compounds with multiple projects. If `kkaudiobk` and any future project each carry their own email logic, you now have multiple places to update when the SMTP provider changes, multiple sets of credentials to rotate, and multiple failure modes to debug independently. A standalone service solves this cleanly: email can be monitored, updated, and redeployed on its own without touching any consuming application. It is the microservice principle applied at the smallest useful scale — not over-engineering, just the right separation at a boundary that genuinely warrants it.

---

## How It Works

When a consuming project needs to send an email — an account verification message, a notification, anything — it makes a POST request to MailEngine's `/send` endpoint with the email payload: recipient address, subject, and body. That is the full extent of what the consuming project needs to know.

MailEngine receives the request through its Express layer, validates that the required fields are present and well-formed, then passes the payload to Nodemailer. Nodemailer constructs the email and dispatches it via the configured SMTP provider. Once the provider responds, MailEngine translates that outcome into a structured response — success or failure — and returns it to the caller. The consuming project handles the response and proceeds accordingly. At no point does it have visibility into which SMTP provider is in use, how the credentials are stored, or how the transport is configured. That boundary is intentional and firm.

```
  Consuming Project (e.g. kkaudiobk)
           │
           │  POST /send
           │  { to, subject, body, ... }
           ▼
    ┌─────────────────┐
    │   MailEngine    │
    │   Express API   │
    │                 │
    │   Nodemailer    │
    │   SMTP Client   │
    └────────┬────────┘
             │
             ▼
      SMTP Provider
      (Email Delivery)
```

---

## API Design

MailEngine keeps its interface minimal by design. Consuming projects should not need to know anything about SMTP internals to send an email — they need to know the endpoint, the fields, and what a response looks like. Nothing more.

### `POST /send`

| Field | Type | Required | Description |
|---|---|---|---|
| `to` | `string` | Yes | Recipient email address |
| `subject` | `string` | Yes | Email subject line |
| `html` | `string` | Yes | HTML email body |
| `text` | `string` | No | Plain text fallback |

MailEngine returns a success or failure status with a message. No SMTP details, no provider names, no transport-layer information leaks into the response.

The minimal surface is a deliberate trade-off. Consuming projects are shielded from all transport-layer concerns — if the SMTP provider changes, only MailEngine changes. No consuming project needs to be updated, redeployed, or even notified.

---

## Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| Runtime | Node.js | Service runtime |
| Framework | Express | HTTP server and routing |
| Email Transport | Nodemailer | SMTP abstraction and email construction |
| Protocol | SMTP | Email delivery |
| Deployment | VPS, PM2 | Process management and uptime |

---

## Design Decisions

### Why Express and not a bare Node.js HTTP server?

Express adds minimal overhead but gives clean routing, middleware support for request validation, and structured error handling — all useful even for a single-endpoint service. A bare HTTP server would save nothing meaningful while making the code harder to reason about if the API surface grows. For a service that is meant to be long-lived and consumed by multiple projects, the marginal cost of Express is worth the clarity it provides.

### Why Nodemailer and not a third-party email API (SendGrid, Resend, etc.)?

Third-party email APIs introduce an external dependency and often require per-project API key management, provider-specific SDKs, and abstraction layers that ultimately sit on top of SMTP anyway. Nodemailer communicates directly via SMTP — MailEngine owns the transport layer entirely. Switching providers is a configuration update, not a code change across multiple integrated projects. That control matters more than the convenience a third-party API offers.

### Why a deployed service and not a shared npm module?

A shared module means every consuming project bundles email logic, manages its own SMTP configuration, and handles its own failures independently. A deployed service means: one place to update, one process to monitor, one set of credentials to manage. When a bug in email delivery appears, there is exactly one place to look. For a multi-project setup, this trade-off favors the deployed service unambiguously — the operational overhead of a single small service is far lower than the maintenance cost of scattered email logic across projects.

---

## Current Usage

MailEngine is in production and currently consumed by:

| Project | Usage |
|---|---|
| Muves (`kkaudiobk`) | Transactional emails — account verification, notifications |

The service is designed to be consumed by any future project without modification. The API contract is stable and the transport layer is entirely internal to MailEngine — a new consumer needs only the endpoint and the field spec.

---

## Known Rough Edges

These are known gaps being evaluated, not afterthoughts:

- **No retry queue for failed deliveries.** If SMTP dispatch fails, the error is returned to the caller. Retry logic is currently the consumer's responsibility. A built-in retry queue with exponential backoff is the most significant missing piece.
- **No email templating engine.** HTML is constructed by the consuming project and passed in the payload; MailEngine renders what it receives. A template layer inside MailEngine would remove duplication if multiple consumers end up sending structurally similar emails.
- **No delivery status tracking or webhooks.** MailEngine is fire-and-forget — there is no mechanism to confirm whether an email was delivered, bounced, or opened. Adding provider webhook handling would close that gap.
- **No service-to-service authentication.** Requests are currently trusted at the network level within the VPS internal network. Explicit auth — an API key or signed request header — is the next hardening step before any public-facing exposure.

---

## Source Access

---

MailEngine is a private repository.

If you want to see how the service is structured — the Express routing, Nodemailer configuration, or how the API contract is enforced — reach out.

**Contact:** [lijishwilson@gmail.com](mailto:lijishwilson@gmail.com)  
**Portfolio:** [OLAI](https://olai.yourdomain.com)
