<div align="center">

# 📧 MailEngine

### One service. Every project's email goes through here.
#### Built once, deployed independently, consumed by any project over HTTP — so SMTP never lives inside application code again.

<br/>

[![Status](https://img.shields.io/badge/Status-Production-brightgreen?style=for-the-badge)](https://github.com/next-coder21)
[![Runtime](https://img.shields.io/badge/Runtime-Node.js-339933?style=for-the-badge&logo=nodedotjs&logoColor=white)](https://github.com/next-coder21)
[![Framework](https://img.shields.io/badge/Framework-Express-000000?style=for-the-badge&logo=express&logoColor=white)](https://github.com/next-coder21)
[![Transport](https://img.shields.io/badge/Transport-Nodemailer-22B573?style=for-the-badge)](https://github.com/next-coder21)
[![Type](https://img.shields.io/badge/Type-Microservice-6366f1?style=for-the-badge)](https://github.com/next-coder21)
[![Consumed By](https://img.shields.io/badge/Consumed%20By-Muves-F7931A?style=for-the-badge)](https://github.com/next-coder21)

</div>

---

## 📖 What Is MailEngine

MailEngine is a self-contained HTTP service with one job: send email. It exposes a minimal REST API — any project that needs to deliver a transactional email makes a POST request to MailEngine and gets on with its work. SMTP credentials, provider configuration, and transport mechanics stay entirely inside the service. No consuming project ever sees them.

The case for building this as a standalone service rather than embedding email logic inside `kkaudiobk` or any other application is about where configuration actually belongs. SMTP credentials and provider details are infrastructure concerns — they have no business sitting inside application code. When email logic lives inside a project, that project has accepted an infrastructure responsibility it shouldn't own.

The problem compounds across projects. If every service embeds its own email logic, you now have multiple places to rotate credentials when a provider changes, multiple failure modes to debug independently, and no single place to observe delivery behaviour. A standalone service solves this cleanly: email can be monitored, updated, and redeployed on its own without touching any consumer. It is the microservice principle applied at the smallest useful scale — not over-engineering, just the right cut at a boundary that genuinely warrants it.

---

## ⚙️ How It Works

When a consuming project needs to send an email — an account verification message, a password reset OTP, a rate-limit security alert — it makes a POST request to MailEngine's `/send` endpoint with the payload: recipient, subject, and body. That is the full extent of what the consumer needs to know.

MailEngine receives the request through its Express layer, validates that the required fields are present and well-formed, then hands the payload to Nodemailer. Nodemailer constructs the message and dispatches it through the configured SMTP provider. Once the provider responds, MailEngine translates the outcome into a structured success or failure response and returns it to the caller. The consuming project handles that response and moves on. At no point does it have visibility into which provider is in use, how credentials are stored, or how the transport is configured. That boundary is intentional and firm.

```
  Consuming Project (e.g. kkaudiobk)
           │
           │  POST /send
           │  { to, subject, html, ... }
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

## 🔌 API Design

MailEngine keeps its interface minimal by design. Consuming projects should not need to know anything about SMTP internals to send an email — they need to know the endpoint, the fields, and what a response looks like. Nothing more.

<details>
<summary><strong>📤 POST /send</strong></summary>

<br/>

| Field | Type | Required | Description |
|:---|:---|:---:|:---|
| `to` | `string` | ✅ | Recipient email address |
| `subject` | `string` | ✅ | Email subject line |
| `html` | `string` | ✅ | HTML email body |
| `text` | `string` | — | Plain text fallback |

MailEngine returns a success or failure status with a message. No provider names, no transport details, no SMTP internals leak into the response.

> [!NOTE]
> The minimal surface is a deliberate trade-off. If the SMTP provider changes, only MailEngine changes. No consuming project needs to be updated, reconfigured, or redeployed. The transport layer is an implementation detail of MailEngine, not a contract it shares outward.

</details>

---

## ⚡ Tech Stack

| Layer | Technology | Purpose |
|:---|:---|:---|
| 🟢 Runtime | Node.js | Service runtime |
| 🔀 Framework | Express | HTTP server and routing |
| 📨 Email Transport | Nodemailer | SMTP abstraction and message construction |
| 🔗 Protocol | SMTP | Email delivery |
| 🏗️ Deployment | VPS · PM2 | Process management and uptime |

---

## 🧠 Design Decisions

<details>
<summary><strong>⚡ Why Express and not a bare Node.js HTTP server?</strong></summary>

<br/>

Express adds minimal overhead but gives clean routing, middleware support for request validation, and structured error handling — all useful even for a single-endpoint service. A bare HTTP server would save nothing meaningful while making the code harder to reason about as the service evolves. For something meant to be long-lived and consumed by multiple projects, the marginal cost of Express is worth the clarity it buys.

</details>

<details>
<summary><strong>📦 Why Nodemailer and not a third-party email API (SendGrid, Resend, etc.)?</strong></summary>

<br/>

Third-party email APIs introduce an external dependency and typically require per-project API key management, provider-specific SDKs, and abstraction layers that ultimately sit on top of SMTP anyway. Nodemailer communicates directly via SMTP — MailEngine owns the transport layer entirely. Switching providers is a configuration update, not a code change across multiple integrated projects. That control matters more than the convenience a managed API offers.

> [!IMPORTANT]
> The key constraint: if provider credentials or endpoints change, the fix happens in exactly one place — MailEngine's config. No consumer is touched. A third-party SDK embedded across multiple projects would require coordinated updates across all of them.

</details>

<details>
<summary><strong>🚀 Why a deployed service and not a shared npm module?</strong></summary>

<br/>

A shared module means every consuming project bundles email logic, manages its own SMTP configuration, and handles its own delivery failures independently. A deployed service means one place to update, one process to monitor, and one set of credentials to manage. When a delivery bug appears, there is exactly one place to look. For a multi-project setup, this trade-off favours the deployed service without ambiguity — the operational overhead of a single small service is far lower than the maintenance cost of scattered email logic across codebases.

</details>

---

## 🔗 Current Usage

MailEngine is in production, currently consumed by:

| Project | Usage |
|:---|:---|
| Muves (`kkaudiobk`) | Transactional emails — account verification, password reset OTPs, rate-limit security alerts |

The service is designed to be consumed by any future project without modification. The API contract is stable, and the transport layer is entirely internal to MailEngine. A new consumer needs only the endpoint and the field spec.

---

## ⚠️ Known Rough Edges

> [!WARNING]
> These are tracked gaps, not disclaimers.

- **No retry queue for failed deliveries** — if SMTP dispatch fails, the error is returned to the caller immediately. Retry logic with backoff is currently the consumer's responsibility. A built-in retry queue is the most significant missing piece.
- **No email templating engine** — HTML is constructed by the consuming project and passed in the payload; MailEngine renders what it receives. A template layer inside MailEngine would eliminate duplication once multiple consumers start sending structurally similar messages.
- **No delivery status tracking or webhooks** — MailEngine is fire-and-forget. There is no mechanism to confirm whether an email was delivered, bounced, or opened. Adding provider webhook handling would close that gap.
- **No service-to-service authentication** — requests are currently trusted at the network level within the VPS internal network. Explicit auth — an API key or signed request header — is the next hardening step before any public-facing exposure.

---

## 📬 Source Access

<div align="center">

MailEngine is a private repository.

If you've read this far and want to see how it's actually wired up — the Express routing, Nodemailer configuration, or how the API contract is enforced — reach out.

<br/>

[![Email](https://img.shields.io/badge/Email-lijishwilson%40gmail.com-D14836?style=for-the-badge&logo=gmail&logoColor=white)](mailto:lijishwilson@gmail.com)

</div>
