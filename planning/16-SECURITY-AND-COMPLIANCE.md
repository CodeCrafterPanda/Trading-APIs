# Security, market rules and launch evidence

This document separates researched requirements from account certification. Regulatory/NSE research: 21 October 2026; Dhan/Upstox docs: 21–22 October 2026; Zerodha docs: 22 October 2026. No account has been approved, connected or traded by this pack. Initial assumption: an investor's own self-developed platform on that investor's Dhan, Upstox and Zerodha accounts; provider/commercial use needs a different review.

## Retail algo framework

SEBI's September 2025 extension applies the framework to all stockbrokers from **1 April 2026**. NSE standards and later clarifications distinguish below-threshold self-developed algorithms (generic tagging, no individual registration), above-threshold registration, and provider algorithms (empanelment). All client API orders are algo orders for tagging purposes; staying below the threshold removes neither identity, tagging, static-IP, 2FA nor broker risk requirements. The initial threshold is not exceeding 10 orders/second per client per exchange/segment, with scope/counting clarified through exchange/broker evidence. [SEBI extension](https://www.sebi.gov.in/sebi_data/attachdocs/sep-2025/1759232056254.pdf), [NSE standards](https://nsearchives.nseindia.com/content/circulars/INVG67858.pdf), [NSE FAQ](https://nsearchives.nseindia.com/web/sites/default/files/inline-files/FAQ_Retail%20Algo_03112025_NSE.pdf), [NSE audit terms, clause 27](https://nsearchives.nseindia.com/content/circulars/INSP73850.pdf).

Broker implementations of the same framework, all effective 1 April 2026:

| Broker | Static IP | Rate/threshold enforcement | Algo identity | Market orders |
|---|---|---|---|---|
| Dhan | Required for all order APIs (normal, super, forever); primary/secondary IPv4 or IPv6; changes lock 7 days; `/ip/*` endpoints | 10/s, 250/min, 1,000/h, 7,000/day (D-ORDER) | Per-account; registration above threshold via broker | Enum present; policy gated per segment/phase |
| Upstox | User-level primary + optional secondary via `/v2/user/ip`; one active API app per user | Regular 10/s, registered 50/s, 500/min, 2,000/30 min (U-ORDER) | `X-Algo-Name` only for exchange-approved configured strategies; UDAPI1154/1156 errors | Unprotected MARKET/SL-M rejected; `market_protection` −1 or 1–25 % |
| Zerodha | Required for order endpoints only; 1–2 IPs in developer console; **one change per calendar week**; data/read endpoints from any IP | 10/s per client ID → HTTP 429; 400/min; 5,000/day (Z-ORDER) | Algo ID tied to API key ("plan for now", staff statement) | `market_protection=0` rejected; −1 automatic |

[Dhan auth](https://dhanhq.co/docs/v2/authentication/), [Upstox deployed changes](https://community.upstox.com/t/important-update-regulatory-changes-for-api-and-algo-trading-are-now-live/14874), [Upstox market protection](https://upstox.com/developer/api-documentation/announcements/market-protection/), [Zerodha static IP](https://support.zerodha.com/category/trading-and-markets/general-kite/kite-api/articles/static-ip), [Zerodha algo rules (staff)](https://kite.trade/forum/discussion/comment/52171).

ComplianceProfile (per account) stores account category, broker app type/tier (Upstox app type; Zerodha Personal vs Connect), exchanges/segments, approved primary/secondary static egress, algo tag/name/registration, order-message counting policy, allowed session/order types, effective dates, evidence references, reviewer and review expiry. Unknown required evidence blocks mutations on that account. Provider/black-box/third-party operation cannot inherit the private-client profile.

## Effective-dated sessions

| Rule set | Status at research date | Implementation |
|---|---|---|
| Current CAS-related changes | Live from 3 August 2026 | Instrument eligibility and phase-specific rules (`CURRENT_2026_08_03_CAS`) |
| NSE normal derivatives close | Timings page lists 15:40 | No universal 15:30 assumption |
| Upstox CAS sequence | Cash continuous close 15:15; auction phases; random close 15:28–15:30; SL/SL-M, IOC and slicing disallowed in CAS; GTT only until 15:15 | Consume `cas_eligible`, exchange status `cas_eligible_status` and stream phases |
| SEBI 15 October circular | Effective 7 December 2026 (`PENDING_2026_12_07_CAS`) | Stored, not activated early |
| Paragraph 4.8 carryover | Effective 8 February 2027 (`PENDING_2027_02_08_CARRYOVER`) | Separate later activation |

Current evidence: [NSE timings](https://www.nseindia.com/static/market-data/market-timings), [NSE July circular](https://nsearchives.nseindia.com/content/circulars/FAOP75472.pdf), [CAS SOP](https://nsearchives.nseindia.com/content/circulars/CMTR73362.pdf), [Upstox CAS](https://upstox.com/developer/api-documentation/announcements/closing-auction-session/), [SEBI October circular](https://www.sebi.gov.in/sebi_data/attachdocs/oct-2026/1792079571420.pdf).

The October circular schedules CAS 15:30–15:40, stock derivatives to 15:45, index derivatives to 15:45 except expiring contracts at 15:30, expiry settlement on 30-minute VWAP and post-close cash trading 15:50–16:00. Preserve future rules behind activation gates requiring operative exchange notices and broker support. NSE's derivatives pre-open permits algo market orders in its first five minutes, so order-type policy is effective-dated and phase-specific: LIMIT/SL-LIMIT initial automation, separately certified exceptions. [NSE derivatives pre-open](https://www.nseindia.com/static/products-services/equity-derivatives-pre-open-session).

Calendar inputs: the platform calendar is authoritative and effective-dated; Upstox `GET /v2/market/holidays[/date]`, `/v2/market/timings/{date}` and `/v2/market/status/{exchange}` are validated inputs and intraday phase signals ([01](01-API-RESEARCH.md)); Dhan and Zerodha publish no equivalent endpoint. BSE's primary notice fetch was blocked during research; BSE-specific circulars, session behavior and approval route remain an external gate (G07). Holidays, special sessions, halts and broker square-off deadlines require daily validated refresh.

## Credential and application security

Secrets use encrypted server-side storage with envelope encryption and a managed key service; token metadata is stored separately from secret values. Each adapter receives only its account's secret handle; UI and analytical workers receive no broker credentials. Egress is restricted to required broker endpoints from approved static addresses. One credential coordinator per account rotates keys/tokens.

| Broker | Default flow | Lifetime | Renewal | Notes |
|---|---|---|---|---|
| Dhan | Web/consent login (API key + secret, 12-month credential); TOTP `generateAccessToken` behind capability flag | 24 h | `GET /RenewToken` only for active Web-origin tokens | 25 consent IDs/day; never log token/PIN/TOTP query strings |
| Upstox | OAuth authorization code, server-side exchange | Next 03:30 boundary | Re-authorize; TOTP endpoint gated (1/s, 10/min, 60/30 min) | Analytics token: GET-only, one year, rejected by execution routes |
| Zerodha | Redirect login → `request_token` (minutes) → `POST /session/token` with SHA-256(api_key+request_token+api_secret) | "6 AM" next day (no timezone label; treat as 06:00 IST) | Interactive re-login; `refresh_token` only for approved platforms | Login elsewhere invalidates (TokenException 403): pause submissions, re-login, reconnect, reconcile |

Unattended PIN/TOTP seed storage is disabled by default; operator-supplied secret inputs are ephemeral and redacted. No invented refresh grants ([Dhan auth](https://dhanhq.co/docs/v2/authentication/), [Upstox token](https://upstox.com/developer/api-documentation/get-token/), [Kite user](https://kite.trade/docs/connect/v3/user/)).

Platform login uses strong authentication, short-lived secure HttpOnly/SameSite sessions, CSRF protection for cookie-authenticated mutations and roles VIEW, OPERATE, CONFIGURE_RISK, ADMINISTER_CREDENTIALS. Risk-limit increases and strategy activation are auditable owner actions. Validate OAuth state/nonce and registered redirects; never log authorization URLs; redact headers, bodies, crash dumps and diagnostics. Encrypted backups never store plaintext keys alongside ciphertext. SBOM review, signed release artifacts and least-privilege service identities are release gates.

## Postbacks, webhooks and untrusted input

| Source | Authentication | Scope | Treatment |
|---|---|---|---|
| Dhan postback | None documented | Orders from the originating access token | Untrusted redundant input |
| Upstox webhook | None; URL filtered at app creation (dynamic DNS/proxy domains blocked), SSL recommended | App's orders; GTT updates need app configuration | Untrusted redundant input |
| Zerodha postback | `checksum` = SHA-256(order_id + order_timestamp + api_secret), verified with the server-held secret | Orders placed with that api_key; sent even when logged out | Authenticated after verification; socket text frames remain primary |

All ingress: TLS, controlled ingress process without broker credentials, random endpoint path secret where the broker allows, strict body/schema/account validation, replay-deduplication hashes and rate controls. Unsigned payloads never release risk on their own; corroborate through the authenticated stream or REST. Webhook fields never trigger URLs, commands or new orders. [Upstox webhook policy](https://upstox.com/developer/api-documentation/announcements/webhook-url-security-policy/), [Dhan postback](https://dhanhq.co/docs/v2/postback/), [Kite postbacks](https://kite.trade/docs/connect/v3/postbacks/).

## Audit, retention and data rights

Audit intent, strategy/config versions, data quality, risk checks, command payload hash, broker results, fills, modifications, cancels, protection, operator actions and reconciliation. Append-only exports include UTC time, actor and trace; secrets are excluded. Five-year application retention is a proposed policy pending legal review; broker audit duties are not automatically identical to private application duties.

Broker data access does not establish public redistribution, commercial dashboards, index constituent/weight licensing or perpetual storage rights. Zerodha's Connect tier and Dhan's Data API are paid entitlements for the account holder's use. Keep license evidence per dataset and use case; public/multi-user exposure is blocked until cleared. [NSE index licensing](https://www.niftyindices.com/offerings/index-licensing).

## Launch gate register

| Gate | Evidence required | Blocks |
|---|---|---|
| G01 Account route | Owner/broker confirm self-developed vs provider/business category, per broker | All execution |
| G02 Auth and IP | Account identity, token flow, approved egress (Dhan 7-day lock, Zerodha weekly change), daily logout behavior | Account mutations |
| G03 Algo controls | Tag/name/registration and message-count scope confirmed per broker | Automated/API execution |
| G04 Segment/products | Enabled NSE/BSE products, shorts, MTF, delivery authorization (DDPI/TPIN/Kite authorise) | Affected routes |
| G05 Data entitlement | Dhan Data API, Upstox tier/Plus, Zerodha Connect vs Personal; mode/connection quotas; deep data | Affected feed/scanners |
| G06 Units/schema | Actual tick/price/volume/Greek/time fixtures per broker, Zerodha divisors by segment | Affected calculations/orders |
| G07 Session/BSE | Operative calendars/circulars and square-off deadlines | Affected venue/session |
| G08 Protections | Dhan Super/Forever, Upstox GTT/trailing, Zerodha GTT OCO/CO partial-fill and modify behavior | Strategies requiring them |
| G09 Data rights | Storage/display/derived/index licensing evidence | Disallowed use/distribution |
| G10 Durability/recovery | Fencing, synchronous ledger, restore/reconciliation tests | Live mutations |
| G11 Live conformance | Separately authorized commissioning evidence per broker/product | Production enablement |
| G12 Risk values | Owner-approved account/global limits and emergency policy | All execution |
| G13 Zerodha app tier | Connect app active for streaming/history; billing/waiver state | Zerodha market data |
| G14 Zerodha update channel | Confirmation that the app type receives socket order frames and postbacks | Zerodha execution (reconciliation-only until verified) |

Each gate has status OPEN|VERIFIED|EXPIRED|REVOKED, evidence URL/artifact hash, reviewer, verified_at and next_review_at. No gate is marked verified by this pack. Read-only functionality proceeds where its own permissions are satisfied; live routes remain fail-closed.

## Acceptance

Security tests: token leakage, role/CSRF boundaries, webhook spoofing/replay on all three ingress paths (including a Zerodha payload with a wrong checksum), invalid OAuth state, secret rotation, revoked credentials, Zerodha login-elsewhere invalidation. Operational tests: future calendar activation, random auction close, special holidays, static-IP mismatch per broker, weekly IP-change lock, expired compliance evidence, 06:00/03:30/24-hour token boundaries. Restore/fencing drills prove an old gateway cannot continue sending after takeover. An audit reviewer must reconstruct any order from its recorded evidence without accessing a live secret.
