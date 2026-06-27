# Muves

> Three private repositories, one working music streaming system — built, hosted, and operated without touching a third-party platform for any core function.

![Status](https://img.shields.io/badge/Status-Active-brightgreen)
![Android](https://img.shields.io/badge/Android%20App-React%20Native-61DAFB?logo=react)
![Web](https://img.shields.io/badge/Web-React%2019-61DAFB?logo=react)
![Backend](https://img.shields.io/badge/Backend-Node.js%20%2B%20PostgreSQL-339933?logo=node.js&logoColor=white)
![Infra](https://img.shields.io/badge/Infra-Self--Hosted%20VPS-informational)
![Auth](https://img.shields.io/badge/Auth-JWT-orange)

---

## What Is Muves

Muves is a fully working, self-hosted music streaming system. It streams real audio, handles real authentication, runs on a real VPS, and has been used in production. This is not a clone, not a tutorial walkthrough — it's an end-to-end product built across three repositories, each with a single responsibility, designed to work together as one system. It's still in active development with known rough edges, and some parts are incomplete by design, not by accident.

The reason it exists is straightforward: the goal was to have a streaming setup that is entirely owned. No upload limits enforced by a third party. No API keys that can be revoked. No streaming service terms that prohibit self-hosting. The backend resolves audio from storage, proxies it through an authenticated endpoint, and the clients never see the source URL — the storage layer is an implementation detail, not a constraint on what the clients can do.

---

## The Ecosystem — Three Repositories

Each repository has a single responsibility. They are independently deployable but designed to consume the same backend contract.

| Repository | Role | Stack | Status |
|---|---|---|---|
| `muvesmobile` | Android client | React Native 0.81.5, Expo 54, TypeScript (strict) | Active |
| `kk-lisn` | Web client | React 19, Vite 6, Tailwind CSS 4 | Active |
| `kkaudiobk` | Backend API + DB | Node.js 18, Express 4, PostgreSQL, Redis | Active — Production |

---

## Deep Dive — `kkaudiobk` (Backend)

This is the most complex of the three. It handles auth, streaming, queue, favourites, playlists, admin operations, session tracking, and listening stats. The parts worth explaining are the ones where the obvious solution was not the right one.

---

### Authentication Architecture

The auth system uses two token types with distinct lifecycles rather than a single long-lived JWT. On login, the server issues an access token (2-day, signed JWT) and a refresh token (30-day) as separate HttpOnly cookies — `token` and `refresh_token`.

The refresh token is never stored plaintext. The server generates 48 random bytes, stores a SHA-256 hash of them in the database, and sends the raw bytes to the client. On refresh, the server hashes the incoming token and looks it up — the raw value never touches the database. When a refresh succeeds, the old token is immediately marked `is_revoked = TRUE` and a new one is inserted: proper rotation, not extension.

Admin sessions are a completely separate session type. Admin tokens carry an `isAdmin: true` claim and are verified by a different middleware chain that checks for that claim explicitly — a user JWT with no admin claim fails at the guard, not at the role check inside the handler. Admin login has its own rate limit tier (8 requests per 15 minutes, the tightest in the system) because it is the highest-value target in the API.

Password reset uses a three-step flow: request OTP → verify OTP → exchange for a 15-minute signed reset token. The OTP is consumed immediately on verification so it cannot be reused, and the forgot-password endpoint always returns the same response regardless of whether the email exists, preventing email enumeration. The reset token has a `purpose: "password-reset"` claim that is verified before the password update is allowed.

Session context is recorded on login: device type (detected from User-Agent), IP address (from `X-Forwarded-For`), and raw User-Agent string are written to a `user_sessions` table. This is not used for enforcement — it's there for visibility.

---

### Rate Limiting — 6-Tier System

Most applications implement one or two levels of rate limiting: a global catch-all and maybe a tighter limit on auth endpoints. Muves uses six named tiers, registered in order so Express applies the tightest match first:

| Tier | Limit | Window | Covers |
|---|---|---|---|
| `admin-auth` | 8 requests | 15 min | Admin login endpoint only |
| `auth` | 15 requests | 15 min | `/auth/login`, `/auth/register` |
| `contact` | 5 requests | 15 min | Contact form |
| `admin` | 120 requests | 15 min | All admin panel routes |
| `api` | 60 requests | 1 min | Authenticated music/queue/favourites |
| `global` | 300 requests | 15 min | Every other route (catch-all) |

All six are backed by Redis using a fixed-window counter. If Redis is unavailable, each limiter falls back silently to an in-process `Map` — the API keeps working, it just loses cross-process consistency.

When any limiter fires, it sends a security alert email — but only once per IP per limiter per 10-minute window, throttled via a Redis key with a short TTL. The alert includes IP, endpoint, hit count vs limit, and User-Agent. This is not alerting for alerting's sake: a self-hosted VPS has fixed resources, and knowing which IP is hammering which endpoint matters when you can't just scale horizontally.

The global limiter skips `/health` and `/` so uptime monitors never trip it.

---

### Audio Streaming

Clients request audio via `/auth/music/stream/:id`. The endpoint validates the UUID, fetches the source URL from the database, and proxies the audio — the source URL is never sent to the client.

The primary storage backend is Google Drive, accessed via the googleapis SDK with an OAuth2 client that holds a long-lived refresh token. The Drive client is a singleton — one OAuth2 instance for the process lifetime. File metadata (MIME type and size) is cached in-process for 55 minutes, which matches the OAuth access token lifetime, so the Drive API is not hit on every request.

HTTP Range requests are supported: the server reads the `Range` header, calculates byte offsets, passes the range header through to the Drive API, and responds with `206 Partial Content`. This makes audio seeking work correctly — without it, every seek would restart the stream from byte zero. For non-Drive URLs there is an axios-based fallback proxy that applies the same range logic.

Cover images have a separate proxy endpoint for a specific reason: Android's MediaSession (which populates the lock screen and notification player) loads cover art from a URL that must be publicly accessible without auth. The solution is a `/auth/music/cover/:id` endpoint that the app provides as the MediaSession artwork URI — the backend fetches the image from Drive or the local filesystem and pipes it through, so the lock screen gets the correct artwork without exposing the storage layer.

---

### Database

PostgreSQL on Neon. The schema is built around multi-client access — the same queries serve both the Android app and the web client. The core tables: `users`, `user_profiles`, `refresh_tokens`, `user_sessions`, `songs`, `artists`, `albums`, `genres`, `play_history`, and `user_listening_stats`.

Listening stats are upserted per user per day: each play increments `minutes_listened` and `songs_played` for `CURRENT_DATE` in a single `INSERT ... ON CONFLICT DO UPDATE`. The daily granularity is intentional — per-song play history exists separately in `play_history` for detailed lookup, but the stats table is designed for aggregate queries without scanning the full history table.

Lyrics are stored in LRC format in a `lyrics TEXT` column on `songs`. The server parses LRC on read — extracting timestamps and text into `[{time: seconds, text}]` — which the clients use for synchronized display. If the stored text is not valid LRC, the parser returns it as plain lines so the endpoint never returns nothing when lyrics exist.

---

### MailEngine Integration

`kkaudiobk` does not manage email infrastructure directly. All outbound email — welcome messages, password reset OTPs, rate limit security alerts — is delegated to MailEngine, a separate internal SMTP microservice deployed independently on the same VPS.

The backend calls MailEngine over its internal API. From `kkaudiobk`'s perspective, email is a service call, not a library — the SMTP configuration, template rendering, and delivery logic live entirely outside the backend codebase. This means email behavior can be changed, debugged, or redeployed without touching the backend, and MailEngine can be reused across other projects without copying code. MailEngine is its own catalogue entry.

---

## Deep Dive — `muvesmobile` (Android App)

---

### In-App APK Update Flow

The app ships its own update mechanism rather than going through the Play Store. This was a deliberate choice for a private distribution model: no store approval delays, no review process, full control over the release cycle.

The flow: on startup, the app calls the backend's `/version` endpoint, which returns `{ version, versionCode, apkUrl, message, forceUpdate }`. The app compares `versionCode` against the installed version. If an update is available — or if `forceUpdate` is true — it downloads the APK to the device filesystem using `expo-file-system`, then passes the file to the OS install intent via `expo-intent-launcher`.

The URI scheme matters here. Android 7 (API 24) blocked `file://` URIs in install intents for security reasons — attempting to pass a `file://` path to an install intent crashes with a `FileUriExposedException`. The correct approach is a `content://` URI, which requires a `FileProvider` declared in the Android manifest. The download writes the APK to the cache directory and the install intent receives a `content://` URI that Android's package installer can safely read. The non-obvious part is that `expo-intent-launcher` needs the correct MIME type (`application/vnd.android.package-archive`) alongside the URI, or the installer won't recognize the file type.

If `forceUpdate` is set, the update modal is non-dismissable — the user cannot navigate past it until the install completes.

---

### Streaming on Mobile

The app sends authenticated requests to `/auth/music/stream/:id`. The JWT is included via cookie or `Authorization: Bearer` header — the backend accepts both so the mobile client does not have to manage cookies directly. Token state is managed in `AsyncStorage`, scoped per user ID so multi-account scenarios don't cross-contaminate stored tokens. All API calls are routed through a shared `apiClient` utility that centralises base URL configuration and auth header injection.

---

## Deep Dive — `kk-lisn` (Web App)

---

### Shared Backend, Different Client Context

`kk-lisn` and `muvesmobile` consume the same API. What differs is the browser execution context. Cookie-based auth works natively in a browser — the `token` and `refresh_token` cookies are sent automatically on every same-origin (or CORS-credentialed) request, which simplifies the auth flow compared to the mobile client where tokens are managed manually.

Audio playback in the browser uses Howler.js rather than the native `<audio>` element directly. Howler handles the Web Audio API abstraction, cross-browser compatibility, and the playback state machine. The streaming endpoint returns `Accept-Ranges: bytes` so the browser's HTTP layer handles seeking natively — Howler issues range requests without the app needing to manage byte offsets.

The web client includes an admin panel that the mobile app does not. Admin routes are guarded by a separate auth check against the `admin_token` cookie, mirroring the backend's dual-middleware architecture.

---

### React 19

The web client is on React 19 — the release that promoted concurrent rendering and the new `use` hook to stable, and introduced the Actions API for async state transitions. In practice this means form submissions and data mutations use the Actions pattern rather than `useEffect` + manual loading state, which eliminates a class of race conditions around optimistic updates. The concurrent scheduler also means that expensive re-renders (search, library view with large song lists) don't block the UI thread — the browser stays responsive while the render works.

---

## Infrastructure

All three services run on a single VPS. This is a deliberate constraint, not a limitation — a single machine is operationally simpler, easier to reason about, and sufficient for the current load. The constraint is what makes the rate limiting design relevant: there is no load balancer to distribute burst traffic, so the backend itself has to be the first line of defense.

Nginx sits at the edge as the reverse proxy. It handles SSL termination (via certificates managed on the server), routes traffic to the correct service by subdomain, and serves static files for `kk-lisn` directly from the build output directory without hitting Node.js. PM2 manages the `kkaudiobk` process — it restarts on crash, captures logs, and keeps the service alive across server reboots. MailEngine runs as a separate PM2-managed process on the same host, accessible to the backend over localhost.

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

## Tech Stack

| Layer | Technology | Repository |
|---|---|---|
| Android App | React Native 0.81.5, Expo 54, TypeScript (strict) | `muvesmobile` |
| Web App | React 19, Vite 6, Tailwind CSS 4 | `kk-lisn` |
| Audio Playback (web) | Howler.js | `kk-lisn` |
| Animations (web) | Framer Motion | `kk-lisn` |
| Backend API | Node.js 18, Express 4 | `kkaudiobk` |
| Database | PostgreSQL (Neon) | `kkaudiobk` |
| Cache / Rate-limit store | Redis (in-memory fallback) | `kkaudiobk` |
| Auth | Dual-token JWT (access + refresh, HttpOnly cookies) | `kkaudiobk` |
| Rate Limiting | 6-tier, Redis-backed, breach alerting | `kkaudiobk` |
| Email | MailEngine (internal SMTP microservice) | Separate service |
| Infrastructure | VPS, Nginx, PM2, SSL | — |

---

## Known Rough Edges

These are real, known issues — not disclaimers:

- **Token refresh under poor network conditions** — the mobile client does not yet handle the case where the refresh request itself fails mid-flight. A timed-out refresh during playback will drop the session without a clean retry path.
- **APK download failure handling** — the in-app update flow does not yet gracefully handle a partial download or a connection drop mid-file. A failed download leaves an incomplete APK in the cache directory with no automatic cleanup or retry.
- **No offline playback** — the mobile app has no caching layer. If the backend is unreachable, nothing plays. This is the next meaningful feature to add.
- **Web player responsiveness** — the player UI is designed for desktop-width viewports. Below ~768px some controls overlap or lose touch target size. The mobile web experience works but is not finished.
- **Rate limiter config is hardcoded** — the six tier limits (300/15min, 60/1min, etc.) are constants in source. There is no environment variable override, which means changing limits requires a redeployment.

---

## Source Access

Muves spans three private repositories — `muvesmobile`, `kk-lisn`, and `kkaudiobk`.

If you've read this far and want to see how any of it is actually implemented, reach out.  
I'm happy to walk through the code or grant repository access for review.

**Contact:** [lijishwilson@gmail.com](mailto:lijishwilson@gmail.com)
