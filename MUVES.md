<div align="center">

<img src="https://res.cloudinary.com/dokomt4m4/image/upload/v1782547480/uploads/hjeqr18ufniynoab00mp.avif" alt="Muves Logo" width="480"/>

<br/>

### Three private repositories. One working music streaming system.
#### Built, hosted, and operated without touching a third-party platform for any core function.

<br/>

[![Status](https://img.shields.io/badge/Status-Active-brightgreen?style=for-the-badge)](https://github.com/next-coder21)
[![Android](https://img.shields.io/badge/Android-React%20Native-61DAFB?style=for-the-badge&logo=react&logoColor=black)](https://github.com/next-coder21)
[![Web](https://img.shields.io/badge/Web-React%2019-61DAFB?style=for-the-badge&logo=react&logoColor=black)](https://github.com/next-coder21)
[![Backend](https://img.shields.io/badge/Backend-Node.js-339933?style=for-the-badge&logo=node.js&logoColor=white)](https://github.com/next-coder21)
[![Database](https://img.shields.io/badge/Database-PostgreSQL-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)](https://github.com/next-coder21)
[![Infra](https://img.shields.io/badge/Infra-Self--Hosted%20VPS-0a0a0a?style=for-the-badge&logo=linux&logoColor=white)](https://github.com/next-coder21)
[![Auth](https://img.shields.io/badge/Auth-Dual%20JWT-F7931A?style=for-the-badge&logo=jsonwebtokens&logoColor=white)](https://github.com/next-coder21)

</div>

---

## 📖 What Is Muves

Muves is a fully working, self-hosted music streaming system. It streams real audio, handles real authentication, runs on a real VPS, and has been used in production. This is not a clone, not a tutorial walkthrough — it's an end-to-end product built across three repositories, each with a single responsibility, designed to work together as one system. It's still in active development with known rough edges, and some parts are incomplete by design, not by accident.

The reason it exists is straightforward: the goal was to have a streaming setup that is entirely owned. No upload limits enforced by a third party. No API keys that can be revoked. No streaming service terms that prohibit self-hosting. The backend resolves audio from storage, proxies it through an authenticated endpoint, and the clients never see the source URL — the storage layer is an implementation detail, not a constraint on what the clients can do.

---

## 🗂️ The Ecosystem — Three Repositories

Each repository has a single responsibility. They are independently deployable but designed to consume the same backend contract.

| Repository | Role | Stack | Status |
|:---|:---|:---|:---:|
| `muvesmobile` | Android client | React Native 0.81.5 · Expo 54 · TypeScript (strict) | 🟢 Active |
| `kk-lisn` | Web client | React 19 · Vite 6 · Tailwind CSS 4 | 🟢 Active |
| `kkaudiobk` | Backend API + DB | Node.js 18 · Express 4 · PostgreSQL · Redis | 🟢 Production |

---

## 🔧 Deep Dive — `kkaudiobk` (Backend)

This is the most complex of the three. It handles auth, streaming, queue, favourites, playlists, admin operations, session tracking, and listening stats. The parts worth explaining are the ones where the obvious solution was not the right one.

<details>
<summary><strong>🔐 Authentication Architecture</strong></summary>

<br/>

The auth system uses two token types with distinct lifecycles rather than a single long-lived JWT. On login, the server issues an access token (2-day, signed JWT) and a refresh token (30-day) as separate HttpOnly cookies — `token` and `refresh_token`.

The refresh token is never stored plaintext. The server generates 48 random bytes, stores a SHA-256 hash of them in the database, and sends the raw bytes to the client. On refresh, the server hashes the incoming token and looks it up — the raw value never touches the database. When a refresh succeeds, the old token is immediately marked `is_revoked = TRUE` and a new one is inserted: proper rotation, not extension.

> [!IMPORTANT]
> Admin sessions are an entirely separate session type — not a role check on a shared token. Admin tokens carry an `isAdmin: true` claim and are verified by a dedicated middleware chain. A user JWT with no admin claim fails at the guard, before any handler code runs. Admin login has its own rate limit tier (8 requests per 15 minutes) because it is the highest-value target in the API.

Password reset uses a three-step flow: request OTP → verify OTP → exchange for a 15-minute signed reset token. The OTP is consumed immediately on verification so it cannot be reused, and the forgot-password endpoint always returns the same response regardless of whether the email exists, preventing email enumeration. The reset token has a `purpose: "password-reset"` claim that is verified before the password update is allowed.

Session context is recorded on login: device type (detected from User-Agent), IP address (from `X-Forwarded-For`), and raw User-Agent string are written to a `user_sessions` table. This is not used for enforcement — it's there for visibility.

</details>

<details>
<summary><strong>🛡️ Rate Limiting — 6-Tier System</strong></summary>

<br/>

Most applications implement one or two levels of rate limiting: a global catch-all and maybe a tighter limit on auth endpoints. Muves uses six named tiers, registered in order so Express applies the tightest match first:

| Tier | Limit | Window | Protects | Severity |
|:---|:---:|:---:|:---|:---:|
| `admin-auth` | 8 req | 15 min | Admin login endpoint only | 🔴 |
| `contact` | 5 req | 15 min | Contact form (spam target) | 🔴 |
| `auth` | 15 req | 15 min | `/auth/login`, `/auth/register` | 🟠 |
| `admin` | 120 req | 15 min | All admin panel routes | 🟡 |
| `api` | 60 req | 1 min | Authenticated music/queue/favourites | 🟡 |
| `global` | 300 req | 15 min | Every other route (catch-all) | 🟢 |

All six are backed by Redis using a fixed-window counter. If Redis is unavailable, each limiter falls back silently to an in-process `Map` — the API keeps working, it just loses cross-process consistency.

> [!NOTE]
> When any limiter fires, it sends a security alert email — but only once per IP per limiter per 10-minute window, throttled via a Redis key with a short TTL. The alert includes IP, endpoint, hit count vs limit, and User-Agent. This is not alerting for alerting's sake: a self-hosted VPS has fixed resources, and knowing which IP is hammering which endpoint matters when you can't just scale horizontally.

The global limiter skips `/health` and `/` so uptime monitors never trip it.

</details>

<details>
<summary><strong>🎧 Audio Streaming</strong></summary>

<br/>

Clients request audio via `/auth/music/stream/:id`. The endpoint validates the UUID, fetches the source URL from the database, and proxies the audio — the source URL is never sent to the client.

The primary storage backend is Google Drive, accessed via the googleapis SDK with an OAuth2 client that holds a long-lived refresh token. The Drive client is a singleton — one OAuth2 instance for the process lifetime. File metadata (MIME type and size) is cached in-process for 55 minutes, which matches the OAuth access token lifetime, so the Drive API is not hit on every request.

> [!NOTE]
> HTTP Range requests are fully supported: the server reads the `Range` header, calculates byte offsets, passes the range header through to the Drive API, and responds with `206 Partial Content`. Without this, every seek would restart the stream from byte zero.

Cover images have a separate proxy endpoint for a specific reason: Android's MediaSession (which populates the lock screen and notification player) loads cover art from a URL that must be publicly accessible without auth. The solution is a `/auth/music/cover/:id` endpoint — the backend fetches the image from Drive or the local filesystem and pipes it through, so the lock screen gets the correct artwork without exposing the storage layer.

</details>

<details>
<summary><strong>🗄️ Database Design</strong></summary>

<br/>

PostgreSQL on Neon. The schema is built around multi-client access — the same queries serve both the Android app and the web client.

**Core tables:**

| Table | Purpose |
|:---|:---|
| `users` + `user_profiles` | Auth identity and profile data (split to keep auth table lean) |
| `refresh_tokens` | Hashed refresh tokens with device context and revocation flag |
| `user_sessions` | Login history — device type, IP, User-Agent |
| `songs` + `artists` + `albums` + `genres` | Catalogue with full relational structure |
| `play_history` | Per-user per-song play log |
| `user_listening_stats` | Daily aggregate: minutes listened + songs played per user |

Listening stats are upserted per user per day: each play increments `minutes_listened` and `songs_played` for `CURRENT_DATE` in a single `INSERT ... ON CONFLICT DO UPDATE`. The daily granularity is intentional — `play_history` exists for detailed per-song lookup, but the stats table handles aggregate queries without scanning the full history table.

Lyrics are stored in LRC format in a `lyrics TEXT` column on `songs`. The server parses LRC on read — extracting timestamps and text into `[{time: seconds, text}]` — which the clients use for synchronized display. If the stored text is not valid LRC, the parser returns it as plain lines so the endpoint never returns nothing when lyrics exist.

</details>

<details>
<summary><strong>📧 MailEngine Integration</strong></summary>

<br/>

`kkaudiobk` does not manage email infrastructure directly. All outbound email — welcome messages, password reset OTPs, rate limit security alerts — is delegated to **MailEngine**, a separate internal SMTP microservice deployed independently on the same VPS.

The backend calls MailEngine over its internal API. From `kkaudiobk`'s perspective, email is a service call, not a library — the SMTP configuration, template rendering, and delivery logic live entirely outside the backend codebase. This means email behavior can be changed, debugged, or redeployed without touching the backend, and MailEngine can be reused across other projects without copying code.

> MailEngine is its own catalogue entry.

</details>

---

## 📱 Deep Dive — `muvesmobile` (Android App)

<details>
<summary><strong>📦 In-App APK Update Flow</strong></summary>

<br/>

The app ships its own update mechanism rather than going through the Play Store. This was a deliberate choice for a private distribution model: no store approval delays, no review process, full control over the release cycle.

**The flow:**

```
Backend /version endpoint
        │
        ▼
App compares versionCode with installed version
        │
        ├── No update → continue
        │
        └── Update available (or forceUpdate = true)
                │
                ▼
        Download APK via expo-file-system → cache directory
                │
                ▼
        Generate content:// URI via FileProvider
                │
                ▼
        expo-intent-launcher → Android package installer
```

> [!IMPORTANT]
> Android 7 (API 24) blocked `file://` URIs in install intents — passing one throws `FileUriExposedException` and crashes. The correct path is a `content://` URI via a declared `FileProvider`. The install intent also needs the explicit MIME type `application/vnd.android.package-archive` or the package installer won't recognize the file.

If `forceUpdate` is set server-side, the update modal is non-dismissable — the user cannot navigate past it until the install completes.

</details>

<details>
<summary><strong>🔁 Streaming & Auth on Mobile</strong></summary>

<br/>

The app sends authenticated requests to `/auth/music/stream/:id`. The JWT is included via cookie or `Authorization: Bearer` header — the backend accepts both so the mobile client does not have to manage cookies directly. Token state is managed in `AsyncStorage`, scoped per user ID so multi-account scenarios don't cross-contaminate stored tokens. All API calls are routed through a shared `apiClient` utility that centralises base URL configuration and auth header injection.

</details>

---

## 🌐 Deep Dive — `kk-lisn` (Web App)

<details>
<summary><strong>🔗 Shared Backend, Different Client Context</strong></summary>

<br/>

`kk-lisn` and `muvesmobile` consume the same API. What differs is the browser execution context. Cookie-based auth works natively in a browser — the `token` and `refresh_token` cookies are sent automatically on every credentialed request, which simplifies the auth flow compared to the mobile client where tokens are managed manually.

Audio playback uses **Howler.js** rather than the native `<audio>` element directly. Howler handles the Web Audio API abstraction, cross-browser compatibility, and the playback state machine. The streaming endpoint returns `Accept-Ranges: bytes` so the browser's HTTP layer handles seeking natively — Howler issues range requests without the app needing to manage byte offsets.

The web client includes an admin panel that the mobile app does not. Admin routes are guarded by a separate auth check against the `admin_token` cookie, mirroring the backend's dual-middleware architecture.

</details>

<details>
<summary><strong>⚛️ React 19</strong></summary>

<br/>

The web client is on React 19 — the release that promoted concurrent rendering and the new `use` hook to stable, and introduced the Actions API for async state transitions. In practice this means form submissions and data mutations use the Actions pattern rather than `useEffect` + manual loading state, which eliminates a class of race conditions around optimistic updates. The concurrent scheduler also means that expensive re-renders (search, library view with large song lists) don't block the UI thread — the browser stays responsive while the render works.

</details>

---

## 🏗️ Infrastructure

All three services run on a single VPS. This is a deliberate constraint, not a limitation — a single machine is operationally simpler, easier to reason about, and sufficient for the current load. The constraint is what makes the rate limiting design load-bearing: there is no load balancer to distribute burst traffic, so the backend has to be the first line of defense.

Nginx sits at the edge as the reverse proxy — SSL termination, subdomain routing, and static file serving for `kk-lisn` directly from the build output directory without hitting Node.js. PM2 manages the backend process and MailEngine as separate processes — restart on crash, log capture, persistence across reboots.

```
┌──────────────────────────────────────────────────────┐
│                        VPS                           │
│                                                      │
│   ┌────────────────────────────────────────────┐    │
│   │                  Nginx                     │    │
│   │         (Reverse Proxy + SSL)              │    │
│   └───────┬─────────────────────┬─────────────┘    │
│           │                     │                    │
│           ▼                     ▼                    │
│   ┌───────────────┐    ┌────────────────────┐       │
│   │  kk-lisn      │    │    kkaudiobk       │       │
│   │  Web App      │    │    Backend API     │       │
│   │  React 19     │    │    Node.js + PG    │◄──┐   │
│   └───────────────┘    └────────────────────┘   │   │
│                                  │               │   │
│                         ┌────────▼──────┐        │   │
│                         │  PostgreSQL   │        │   │
│                         └───────────────┘        │   │
│                                               ┌──┴─┐ │
│                                               │Mail│ │
│                                               │Eng │ │
│                                               └────┘ │
│                     PM2 (Process Manager)            │
└──────────────────────────────────────────────────────┘

         muvesmobile (Android) ──► kkaudiobk API
```

---

## ⚡ Tech Stack

| Layer | Technology | Repository |
|:---|:---|:---:|
| 📱 Android App | React Native 0.81.5 · Expo 54 · TypeScript (strict) | `muvesmobile` |
| 🌐 Web App | React 19 · Vite 6 · Tailwind CSS 4 | `kk-lisn` |
| 🔊 Audio Playback (web) | Howler.js | `kk-lisn` |
| ✨ Animations (web) | Framer Motion | `kk-lisn` |
| ⚙️ Backend API | Node.js 18 · Express 4 | `kkaudiobk` |
| 🗄️ Database | PostgreSQL (Neon) | `kkaudiobk` |
| ⚡ Cache / Rate-limit store | Redis · in-memory fallback | `kkaudiobk` |
| 🔐 Auth | Dual-token JWT · HttpOnly cookies | `kkaudiobk` |
| 🛡️ Rate Limiting | 6-tier · Redis-backed · breach alerting | `kkaudiobk` |
| 📧 Email | MailEngine (internal SMTP microservice) | Separate service |
| 🏗️ Infrastructure | VPS · Nginx · PM2 · SSL | — |

---

## ⚠️ Known Rough Edges

> [!WARNING]
> These are real, tracked issues — not disclaimers.

- **Token refresh under poor network** — the mobile client does not yet handle the case where the refresh request itself fails mid-flight. A timed-out refresh during playback drops the session without a clean retry path.
- **APK download failure handling** — the in-app update flow does not gracefully handle a partial download or connection drop. A failed download leaves an incomplete APK in the cache directory with no automatic cleanup or retry.
- **No offline playback** — the mobile app has no caching layer. If the backend is unreachable, nothing plays. This is the next meaningful feature to add.
- **Web player responsiveness** — the player UI is designed for desktop-width viewports. Below ~768px some controls overlap or lose touch target size. The mobile web experience works but is not finished.
- **Rate limiter config is hardcoded** — tier limits are constants in source. There is no environment variable override, which means adjusting limits requires a redeployment.

---

## 📬 Source Access

<div align="center">

Muves spans three private repositories — `muvesmobile`, `kk-lisn`, and `kkaudiobk`.

If you've read this far and want to see how any of it is actually implemented, reach out.
I'm happy to walk through the code or grant repository access for review.

<br/>

[![Email](https://img.shields.io/badge/Email-lijishwilson%40gmail.com-D14836?style=for-the-badge&logo=gmail&logoColor=white)](mailto:lijishwilson@gmail.com)

</div>
